# S3 Data Lake

On this page, you will:

- [x] Build the `s3_bucket` module with versioning, lifecycle, and IAM policies
- [x] Create data lake buckets for dev, staging, and prod environments
- [x] Understand the bucket configuration and access patterns

## What is a Data Lake?

A data lake is a centralised storage repository that holds raw data in its native format until needed. Unlike a data warehouse (which stores processed, structured data), a data lake stores:

- **Raw source data**: Extracts from APIs, databases, and files
- **Intermediate files**: Staging data during transformations
- **Exports**: Query results and reports exported from Snowflake

For our data platform, S3 serves as the data lake, with Snowflake storage integrations providing secure access.

## The S3 Bucket Module

This module creates S3 buckets with production-ready configuration including versioning, encryption, lifecycle policies, and pre-built IAM policies.

Create the module directory:

```sh
mkdir -p modules/s3_bucket
```

### main.tf

Create `modules/s3_bucket/main.tf`:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# -----------------------------------------------------------------------------
# S3 Bucket
# -----------------------------------------------------------------------------
resource "aws_s3_bucket" "this" {
  bucket = var.bucket_name

  tags = merge(var.tags, {
    Name        = var.bucket_name
    Environment = var.environment
    ManagedBy   = "terraform"
  })
}

# -----------------------------------------------------------------------------
# Versioning
# -----------------------------------------------------------------------------
resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id

  versioning_configuration {
    status = var.versioning_enabled ? "Enabled" : "Suspended"
  }
}

# -----------------------------------------------------------------------------
# Encryption
# -----------------------------------------------------------------------------
resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
    bucket_key_enabled = true
  }
}

# -----------------------------------------------------------------------------
# Public Access Block
# -----------------------------------------------------------------------------
resource "aws_s3_bucket_public_access_block" "this" {
  bucket = aws_s3_bucket.this.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# -----------------------------------------------------------------------------
# Lifecycle Rules
# -----------------------------------------------------------------------------
resource "aws_s3_bucket_lifecycle_configuration" "this" {
  bucket = aws_s3_bucket.this.id

  # Clean up incomplete multipart uploads
  rule {
    id     = "abort-incomplete-multipart-uploads"
    status = "Enabled"

    abort_incomplete_multipart_upload {
      days_after_initiation = 7
    }
  }

  # Move old versions to cheaper storage, then delete
  dynamic "rule" {
    for_each = var.versioning_enabled ? [1] : []
    content {
      id     = "noncurrent-version-management"
      status = "Enabled"

      noncurrent_version_transition {
        noncurrent_days = var.noncurrent_version_transition_days
        storage_class   = "STANDARD_IA"
      }

      noncurrent_version_expiration {
        noncurrent_days = var.noncurrent_version_expiration_days
      }
    }
  }

  # Optional: expire objects with specific prefix
  dynamic "rule" {
    for_each = var.temp_prefix_expiration_days != null ? [1] : []
    content {
      id     = "temp-prefix-expiration"
      status = "Enabled"

      filter {
        prefix = "temp/"
      }

      expiration {
        days = var.temp_prefix_expiration_days
      }
    }
  }
}

# -----------------------------------------------------------------------------
# IAM Policy Documents
# -----------------------------------------------------------------------------

# Read-only policy - for Snowflake read-only integrations
data "aws_iam_policy_document" "read" {
  statement {
    sid    = "AllowListBucket"
    effect = "Allow"
    actions = [
      "s3:ListBucket",
      "s3:GetBucketLocation"
    ]
    resources = [aws_s3_bucket.this.arn]
  }

  statement {
    sid    = "AllowReadObjects"
    effect = "Allow"
    actions = [
      "s3:GetObject",
      "s3:GetObjectVersion"
    ]
    resources = ["${aws_s3_bucket.this.arn}/*"]
  }
}

# Read-write policy - for Snowflake full integrations
data "aws_iam_policy_document" "write" {
  statement {
    sid    = "AllowListBucket"
    effect = "Allow"
    actions = [
      "s3:ListBucket",
      "s3:GetBucketLocation"
    ]
    resources = [aws_s3_bucket.this.arn]
  }

  statement {
    sid    = "AllowReadWriteObjects"
    effect = "Allow"
    actions = [
      "s3:GetObject",
      "s3:GetObjectVersion",
      "s3:PutObject",
      "s3:DeleteObject"
    ]
    resources = ["${aws_s3_bucket.this.arn}/*"]
  }
}

# -----------------------------------------------------------------------------
# IAM Policies (for attachment to roles)
# -----------------------------------------------------------------------------
resource "aws_iam_policy" "read" {
  count = var.create_iam_policies ? 1 : 0

  name        = "${var.bucket_name}-read"
  description = "Read-only access to ${var.bucket_name} S3 bucket"
  policy      = data.aws_iam_policy_document.read.json

  tags = var.tags
}

resource "aws_iam_policy" "write" {
  count = var.create_iam_policies ? 1 : 0

  name        = "${var.bucket_name}-write"
  description = "Read-write access to ${var.bucket_name} S3 bucket"
  policy      = data.aws_iam_policy_document.write.json

  tags = var.tags
}
```

### variables.tf

Create `modules/s3_bucket/variables.tf`:

```hcl
variable "bucket_name" {
  description = "Name of the S3 bucket (must be globally unique)"
  type        = string
}

variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "versioning_enabled" {
  description = "Enable versioning on the bucket"
  type        = bool
  default     = true
}

variable "noncurrent_version_transition_days" {
  description = "Days before moving noncurrent versions to STANDARD_IA"
  type        = number
  default     = 30
}

variable "noncurrent_version_expiration_days" {
  description = "Days before deleting noncurrent versions"
  type        = number
  default     = 90
}

variable "temp_prefix_expiration_days" {
  description = "Days before expiring objects with temp/ prefix (null to disable)"
  type        = number
  default     = 7
}

variable "create_iam_policies" {
  description = "Create IAM policies for read and write access"
  type        = bool
  default     = true
}

variable "tags" {
  description = "Additional tags for the bucket"
  type        = map(string)
  default     = {}
}
```

### outputs.tf

Create `modules/s3_bucket/outputs.tf`:

```hcl
output "bucket_id" {
  description = "The name of the bucket"
  value       = aws_s3_bucket.this.id
}

output "bucket_arn" {
  description = "The ARN of the bucket"
  value       = aws_s3_bucket.this.arn
}

output "bucket_domain_name" {
  description = "The bucket domain name"
  value       = aws_s3_bucket.this.bucket_domain_name
}

output "bucket_regional_domain_name" {
  description = "The bucket region-specific domain name"
  value       = aws_s3_bucket.this.bucket_regional_domain_name
}

# IAM policy outputs
output "read_policy_arn" {
  description = "ARN of the read-only IAM policy"
  value       = var.create_iam_policies ? aws_iam_policy.read[0].arn : null
}

output "read_policy_json" {
  description = "JSON of the read-only IAM policy document"
  value       = data.aws_iam_policy_document.read.json
}

output "write_policy_arn" {
  description = "ARN of the read-write IAM policy"
  value       = var.create_iam_policies ? aws_iam_policy.write[0].arn : null
}

output "write_policy_json" {
  description = "JSON of the read-write IAM policy document"
  value       = data.aws_iam_policy_document.write.json
}
```

## Create Data Lake Buckets

Now use the module to create data lake buckets for each environment.

### Add Variables

Add to `variables.tf`:

```hcl
variable "project_name" {
  description = "Project name used for resource naming"
  type        = string
}

variable "data_lake_environments" {
  description = "Environments to create data lake buckets for"
  type        = list(string)
  default     = ["dev", "staging", "prod"]
}
```

Configure in `terraform.tfvars`:

```hcl
project_name = "mycompany"  # Replace with your project/company name
```

### Create the Buckets

Create `s3_data_lake.tf`:

```hcl
# =============================================================================
# Data Lake S3 Buckets
# =============================================================================
# S3 buckets for Snowflake data loading and unloading.
# One bucket per environment (dev, staging, prod).

module "data_lake" {
  source   = "./modules/s3_bucket"
  for_each = toset(var.data_lake_environments)

  bucket_name = "${var.project_name}-data-lake-${each.value}"
  environment = each.value

  # Versioning protects against accidental deletions
  versioning_enabled = true

  # Lifecycle settings
  noncurrent_version_transition_days = 30  # Move old versions to IA after 30 days
  noncurrent_version_expiration_days = 90  # Delete old versions after 90 days
  temp_prefix_expiration_days        = 7   # Clean up temp/ files after 7 days

  # Create IAM policies for Snowflake integration roles
  create_iam_policies = true

  tags = {
    Purpose = "Snowflake data lake storage"
  }
}

# -----------------------------------------------------------------------------
# Outputs
# -----------------------------------------------------------------------------
output "data_lake_buckets" {
  description = "Data lake bucket details by environment"
  value = {
    for env, bucket in module.data_lake : env => {
      id               = bucket.bucket_id
      arn              = bucket.bucket_arn
      read_policy_arn  = bucket.read_policy_arn
      write_policy_arn = bucket.write_policy_arn
    }
  }
}
```

## Understanding the Configuration

### Versioning

Versioning is enabled by default. This means:

- Every overwrite creates a new version (original is preserved)
- Deleted objects can be recovered
- Lifecycle rules clean up old versions to control costs

### Encryption

All objects are encrypted at rest using AES-256 (SSE-S3). This is AWS's default encryption and has no additional cost. For sensitive data, you can switch to SSE-KMS with a customer-managed key.

### Public Access Block

All four public access settings are blocked:

- `block_public_acls` - Blocks public ACLs on objects
- `block_public_policy` - Blocks public bucket policies
- `ignore_public_acls` - Ignores existing public ACLs
- `restrict_public_buckets` - Restricts access to AWS principals only

### Lifecycle Rules

The module includes three lifecycle rules:

1. **Abort incomplete uploads**: Cleans up failed multipart uploads after 7 days
2. **Noncurrent version management**: Moves old versions to cheaper storage, then deletes
3. **Temp prefix expiration**: Automatically deletes files in `temp/` after 7 days

### IAM Policies

The module creates two IAM policies:

| Policy | Permissions | Use Case |
|--------|-------------|----------|
| `*-read` | ListBucket, GetObject, GetObjectVersion | Snowflake read-only integrations |
| `*-write` | ListBucket, GetObject, PutObject, DeleteObject | Snowflake full integrations |

These policies are attached to Snowflake IAM roles in the [Storage Integrations](../data-warehouse/8-storage-integrations.md) section.

## Commit and Deploy

Commit your changes and push:

```sh
git add modules/s3_bucket/ s3_data_lake.tf variables.tf terraform.tfvars
git commit -m "Add S3 bucket module and data lake buckets"
git push
```

## Verify in AWS

After deployment, verify the buckets:

```sh
# List buckets
aws s3 ls --profile data-engineer | grep data-lake

# Check bucket configuration
aws s3api get-bucket-versioning --profile data-engineer --bucket mycompany-data-lake-dev
aws s3api get-bucket-encryption --profile data-engineer --bucket mycompany-data-lake-dev
aws s3api get-public-access-block --profile data-engineer --bucket mycompany-data-lake-dev

# List IAM policies
aws iam list-policies --profile data-engineer --scope Local | grep data-lake
```

## Summary

You've created the S3 infrastructure for your data platform:

- [x] Built the `s3_bucket` module with versioning, encryption, and lifecycle policies
- [x] Created data lake buckets for dev, staging, and prod environments
- [x] Generated IAM policies for Snowflake integration roles

## What's Next

The S3 buckets are ready for Snowflake to access. Next, create the storage integrations that connect Snowflake to these buckets.

Continue to [Storage Integrations](../data-warehouse/8-storage-integrations.md) →
