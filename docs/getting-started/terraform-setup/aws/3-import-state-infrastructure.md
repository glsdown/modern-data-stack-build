# Import State Infrastructure

On this page, you will:

- [x] Import the S3 bucket used for Terraform state
- [x] Import the DynamoDB table used for state locking
- [x] Configure lifecycle rules and encryption settings in code

## Understanding the Challenge

Importing Terraform's own state infrastructure creates a unique situation: you're using Terraform to manage the very resources that store Terraform's state. This circular dependency requires careful handling.

The good news is that Terraform handles this gracefully. Once the resources are imported, Terraform can manage them like any other resource. The key is ensuring your configuration exactly matches what exists.

## Gather State Infrastructure Information

First, retrieve details about your existing state resources:

```sh
# Get S3 bucket details
aws s3api get-bucket-versioning --bucket terraform-state-YOUR_ACCOUNT_ID --profile infrastructure-admin

# Get bucket encryption
aws s3api get-bucket-encryption --bucket terraform-state-YOUR_ACCOUNT_ID --profile infrastructure-admin

# Get bucket lifecycle configuration
aws s3api get-bucket-lifecycle-configuration --bucket terraform-state-YOUR_ACCOUNT_ID --profile infrastructure-admin

# Get DynamoDB table details
aws dynamodb describe-table --table-name terraform-state-lock --profile infrastructure-admin
```

Replace `YOUR_ACCOUNT_ID` with your actual AWS account ID.

## Add Variables for State Infrastructure

Update `variables.tf` to include state-related variables:

```hcl
# State infrastructure naming
variable "state_bucket_name" {
  description = "Name of the S3 bucket for Terraform state"
  type        = string
  default     = null  # Will default to terraform-state-{account_id}
}

variable "state_lock_table_name" {
  description = "Name of the DynamoDB table for state locking"
  type        = string
  default     = "terraform-state-lock"
}
```

Update `terraform.tfvars` if you used a custom bucket name:

```hcl
# State Infrastructure (only needed if you used non-default names)
# state_bucket_name     = "my-custom-bucket-name"
# state_lock_table_name = "my-custom-lock-table"
```

## Create State Infrastructure Configuration

Create `state_infrastructure.tf`:

```hcl
# =============================================================================
# Terraform State Infrastructure
# =============================================================================
# These resources store Terraform's own state. Import them carefully.

locals {
  state_bucket_name = coalesce(var.state_bucket_name, "terraform-state-${var.aws_account_id}")
}

# -----------------------------------------------------------------------------
# S3 Bucket for Terraform State
# -----------------------------------------------------------------------------
resource "aws_s3_bucket" "terraform_state" {
  bucket = local.state_bucket_name

  # Prevent accidental deletion of state bucket
  lifecycle {
    prevent_destroy = true
  }

  tags = {
    Name        = local.state_bucket_name
    Purpose     = "TerraformState"
    Environment = "Shared"
  }
}

# Bucket versioning - enables rollback for state files
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}

# Server-side encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
    bucket_key_enabled = true
  }
}

# Block all public access
resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Lifecycle rules for cost optimisation
resource "aws_s3_bucket_lifecycle_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    id     = "MoveOldVersionsToIA"
    status = "Enabled"

    noncurrent_version_transition {
      noncurrent_days = 30
      storage_class   = "STANDARD_IA"
    }

    noncurrent_version_expiration {
      noncurrent_days = 90
    }
  }
}

# -----------------------------------------------------------------------------
# DynamoDB Table for State Locking
# -----------------------------------------------------------------------------
resource "aws_dynamodb_table" "terraform_state_lock" {
  name         = var.state_lock_table_name
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  point_in_time_recovery {
    enabled = true
  }

  # Prevent accidental deletion of lock table
  lifecycle {
    prevent_destroy = true
  }

  tags = {
    Name        = var.state_lock_table_name
    Purpose     = "TerraformStateLock"
    Environment = "Shared"
  }
}
```

!!! warning "Prevent Destroy"
    The `prevent_destroy` lifecycle rule stops accidental deletion of state infrastructure. If you ever need to remove these resources, you must first remove or set this to `false` in your configuration.

## Create Import Blocks

Add to `imports.tf`:

```hcl
# =============================================================================
# State Infrastructure Imports
# =============================================================================

# S3 Bucket
import {
  to = aws_s3_bucket.terraform_state
  id = "terraform-state-123456789012"  # Replace with your bucket name
}

# S3 Bucket Versioning
import {
  to = aws_s3_bucket_versioning.terraform_state
  id = "terraform-state-123456789012"  # Same bucket name
}

# S3 Bucket Encryption
import {
  to = aws_s3_bucket_server_side_encryption_configuration.terraform_state
  id = "terraform-state-123456789012"  # Same bucket name
}

# S3 Bucket Public Access Block
import {
  to = aws_s3_bucket_public_access_block.terraform_state
  id = "terraform-state-123456789012"  # Same bucket name
}

# S3 Bucket Lifecycle Configuration
import {
  to = aws_s3_bucket_lifecycle_configuration.terraform_state
  id = "terraform-state-123456789012"  # Same bucket name
}

# DynamoDB Table
import {
  to = aws_dynamodb_table.terraform_state_lock
  id = "terraform-state-lock"
}
```

!!! note "Update Bucket Name"
    Replace `terraform-state-123456789012` with your actual bucket name.

## Plan the Import

```sh
terraform plan
```

Review the output. You should see resources to be imported with no changes:

```
Plan: 6 to import, 0 to add, 0 to change, 0 to destroy.
```

### Common Configuration Differences

If Terraform shows planned changes, check these common issues:

**Lifecycle Rule ID Mismatch**

The lifecycle rule ID must match exactly. Check your current configuration:

```sh
aws s3api get-bucket-lifecycle-configuration \
  --bucket terraform-state-YOUR_ACCOUNT_ID \
  --profile infrastructure-admin
```

Update the `id` in your lifecycle rule to match.

**Missing Tags**

If you didn't add tags when creating the resources manually, Terraform will want to add them. This is safe to apply.

**Encryption Configuration**

Verify the encryption algorithm matches:

```sh
aws s3api get-bucket-encryption \
  --bucket terraform-state-YOUR_ACCOUNT_ID \
  --profile infrastructure-admin
```

## Apply the Import

```sh
terraform apply
```

Type `yes` when prompted.

Expected output:

```
aws_s3_bucket.terraform_state: Importing... [id=terraform-state-123456789012]
aws_s3_bucket.terraform_state: Import complete [id=terraform-state-123456789012]
aws_s3_bucket_versioning.terraform_state: Importing...
aws_s3_bucket_versioning.terraform_state: Import complete
...

Apply complete! Resources: 6 imported, 0 added, 0 changed, 0 destroyed.
```

## Verify the Import

Check the imported resources in state:

```sh
terraform state list | grep -E "(s3_bucket|dynamodb_table)"
```

Expected output:

```
aws_dynamodb_table.terraform_state_lock
aws_s3_bucket.terraform_state
aws_s3_bucket_lifecycle_configuration.terraform_state
aws_s3_bucket_public_access_block.terraform_state
aws_s3_bucket_server_side_encryption_configuration.terraform_state
aws_s3_bucket_versioning.terraform_state
```

## Add Outputs

Update `outputs.tf`:

```hcl
# State Infrastructure
output "state_bucket_name" {
  description = "Name of the Terraform state S3 bucket"
  value       = aws_s3_bucket.terraform_state.id
}

output "state_bucket_arn" {
  description = "ARN of the Terraform state S3 bucket"
  value       = aws_s3_bucket.terraform_state.arn
}

output "state_lock_table_name" {
  description = "Name of the Terraform state lock DynamoDB table"
  value       = aws_dynamodb_table.terraform_state_lock.name
}

output "state_lock_table_arn" {
  description = "ARN of the Terraform state lock DynamoDB table"
  value       = aws_dynamodb_table.terraform_state_lock.arn
}
```

## Clean Up Import Blocks

After successful import, you can optionally remove the import blocks from `imports.tf`. They're only needed for the initial import. However, keeping them doesn't cause issues - Terraform will simply skip resources already in state.

!!! tip "Keep or Remove?"
    Some teams prefer to keep import blocks as documentation of what was imported. Others remove them to keep the configuration clean. Choose what works best for your team.

## Commit Your Work

```sh
git add terraform/aws/
git commit -m "Import Terraform state infrastructure"
```

## Troubleshooting

### Error: Bucket Does Not Exist

If you see:

```
Error: error reading S3 Bucket: NoSuchBucket
```

Verify:

1. The bucket name is correct (including your account ID)
2. You're using the `infrastructure-admin` profile
3. The profile has access to the correct AWS account

```sh
aws s3 ls --profile infrastructure-admin | grep terraform
```

### Error: Lifecycle Configuration Not Found

If you see:

```
Error: error getting S3 Bucket Lifecycle Configuration
```

The bucket might not have a lifecycle configuration. Either:

1. Create one manually (see [Remote State Setup](../1-set-up-terraform-remote-state.md))
2. Or remove the lifecycle import block and let Terraform create it

### Error: Table Does Not Exist

If you see:

```
Error: ResourceNotFoundException: Requested resource not found
```

Verify:

1. The table name is correct (`terraform-state-lock` by default)
2. You're in the correct region
3. Check the table exists:

```sh
aws dynamodb list-tables --profile infrastructure-admin
```

## What's Next

You've successfully imported the state infrastructure:

- [x] S3 bucket managed in code
- [x] Bucket versioning configured
- [x] Encryption enabled
- [x] Public access blocked
- [x] Lifecycle rules active
- [x] DynamoDB table managed in code

Continue to [import IAM users](5-import-iam-users.md) →
