# Storage Integrations

On this page, you will:

- [x] Understand how Snowflake connects to cloud storage
- [x] Create IAM roles for Snowflake S3 access
- [x] Create storage integrations in Snowflake
- [x] Set up the trust relationship between Snowflake and AWS
- [x] Grant access to the appropriate roles

## What Are Storage Integrations?

Storage integrations allow Snowflake to securely access files in cloud storage. They're essential for:

- **Data loading**: Bulk loading data from files using COPY INTO
- **External tables**: Querying data that lives in cloud storage
- **Data unloading**: Exporting query results to storage

The integration creates a trust relationship between Snowflake and AWS, eliminating the need to manage storage credentials directly.

!!! info "Other Cloud Providers"
    While Snowflake supports GCS and Azure Blob storage, this guide focuses on AWS S3. If your Snowflake account is hosted on AWS (which is recommended), using S3 avoids cross-cloud data egress fees. Using GCS or Azure with an AWS-hosted Snowflake account would incur egress charges on every data transfer.

## How It Works

```
┌─────────────--┐     Trust      ┌─────────────┐
│  Snowflake    │◄──────────────►│    AWS      │
│  Integration  │   Relationship │    IAM      │
└─────────────--┘                └──────┬──────┘
                                        │
                                 ┌──────▼──────┐
                                 │     S3      │
                                 │   Bucket    │
                                 └─────────────┘
```

1. You create a storage integration in Snowflake
2. Snowflake provides an IAM user ARN and external ID
3. You configure an AWS IAM role to trust that identity
4. Snowflake can now access the bucket without credentials

## Why ACCOUNTADMIN?

Storage integrations are account-level objects that require ACCOUNTADMIN privileges to create. This is because they:

- Establish trust relationships with external cloud providers
- Can access any allowed storage location
- Are shared across the entire Snowflake account

The grants to use the integration are managed by SECURITYADMIN (via the module), but the integration itself must be created by ACCOUNTADMIN.

## The Storage Integration Module

This module creates the Snowflake storage integration and outputs the values needed to configure AWS IAM.

Create the module:

```sh
mkdir -p modules/snowflake_storage_integration
```

### main.tf

Create `modules/snowflake_storage_integration/main.tf`:

```hcl
terraform {
  required_providers {
    snowflake = {
      source                = "Snowflake-Labs/snowflake"
      version               = "~> 0.99"
      configuration_aliases = [snowflake.account_admin, snowflake.security_admin]
    }
  }
}

# -----------------------------------------------------------------------------
# Storage Integration
# -----------------------------------------------------------------------------
resource "snowflake_storage_integration" "this" {
  provider = snowflake.account_admin
  name     = upper(var.integration_name)
  comment  = var.integration_comment
  type     = "EXTERNAL_STAGE"
  enabled  = true

  storage_provider          = "S3"
  storage_allowed_locations = var.storage_allowed_locations
  storage_blocked_locations = var.storage_blocked_locations
  storage_aws_role_arn      = var.storage_aws_role_arn
}

# -----------------------------------------------------------------------------
# Grant USAGE to Roles
# -----------------------------------------------------------------------------
resource "snowflake_grant_privileges_to_account_role" "grant_usage" {
  provider = snowflake.security_admin
  for_each = toset(var.grant_usage_to_roles)

  account_role_name = each.value
  privileges        = ["USAGE"]

  on_account_object {
    object_type = "INTEGRATION"
    object_name = snowflake_storage_integration.this.name
  }
}
```

### variables.tf

Create `modules/snowflake_storage_integration/variables.tf`:

```hcl
variable "integration_name" {
  description = "Name of the storage integration (will be uppercased)"
  type        = string
}

variable "integration_comment" {
  description = "Description of the integration's purpose"
  type        = string
  default     = ""
}

variable "storage_allowed_locations" {
  description = "List of allowed S3 locations (e.g., s3://bucket/path/)"
  type        = list(string)
}

variable "storage_blocked_locations" {
  description = "List of blocked S3 locations"
  type        = list(string)
  default     = []
}

variable "storage_aws_role_arn" {
  description = "AWS IAM role ARN for S3 access"
  type        = string
}

variable "grant_usage_to_roles" {
  description = "Account roles to grant USAGE on the integration"
  type        = list(string)
  default     = []
}
```

### outputs.tf

Create `modules/snowflake_storage_integration/outputs.tf`:

```hcl
output "integration_name" {
  description = "Name of the storage integration"
  value       = snowflake_storage_integration.this.name
}

output "storage_aws_iam_user_arn" {
  description = "Snowflake's AWS IAM user ARN (use in IAM role trust policy)"
  value       = snowflake_storage_integration.this.storage_aws_iam_user_arn
}

output "storage_aws_external_id" {
  description = "External ID for AWS IAM role assumption"
  value       = snowflake_storage_integration.this.storage_aws_external_id
}
```

## Prerequisites: S3 Buckets

Before creating storage integrations, ensure you have created the S3 data lake buckets in the [Build Your AWS Infrastructure](../aws/1-s3-data-lake.md) section. You should have:

- `your-project-data-lake-dev` - Development environment bucket
- `your-project-data-lake-staging` - Staging environment bucket
- `your-project-data-lake-prod` - Production environment bucket

If you haven't created these yet, complete the [S3 Data Lake](../aws/1-s3-data-lake.md) page first.

## Create IAM Roles for Snowflake

Create IAM roles that Snowflake will assume to access the buckets. These roles use external IDs from the Snowflake storage integrations for secure trust relationships.

Add to `terraform/aws/iam_snowflake.tf`:

```hcl
# =============================================================================
# Snowflake Storage Access IAM Roles
# =============================================================================
# IAM roles that Snowflake assumes to access S3 buckets.
# The trust policy is updated after the Snowflake integration is created.

# -----------------------------------------------------------------------------
# Variables for Snowflake Trust (set after integration is created)
# -----------------------------------------------------------------------------
variable "snowflake_storage_aws_iam_user_arn" {
  description = "Snowflake's AWS IAM user ARN from the storage integration"
  type        = string
  default     = ""  # Set after first Snowflake deployment
}

variable "snowflake_storage_external_ids" {
  description = "External IDs for each storage integration"
  type = object({
    dev              = string
    staging          = string
    prod             = string
    prod_readonly    = string
  })
  default = {
    dev              = ""
    staging          = ""
    prod             = ""
    prod_readonly    = ""
  }
}

locals {
  # Environments that need read-write access
  snowflake_rw_environments = ["dev", "staging", "prod"]
}

# -----------------------------------------------------------------------------
# Read-Write Roles (dev, staging, prod)
# -----------------------------------------------------------------------------
resource "aws_iam_role" "snowflake_storage" {
  for_each = toset(local.snowflake_rw_environments)

  name = "snowflake-storage-${each.value}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = var.snowflake_storage_aws_iam_user_arn != "" ? [{
      Effect    = "Allow"
      Principal = { AWS = var.snowflake_storage_aws_iam_user_arn }
      Action    = "sts:AssumeRole"
      Condition = {
        StringEquals = {
          "sts:ExternalId" = var.snowflake_storage_external_ids[each.value]
        }
      }
    }] : [{
      # Placeholder until Snowflake integration is created
      Effect    = "Deny"
      Principal = { AWS = "*" }
      Action    = "sts:AssumeRole"
    }]
  })

  tags = {
    Name        = "snowflake-storage-${each.value}"
    Environment = each.value
    Purpose     = "Snowflake S3 access"
    ManagedBy   = "terraform"
  }
}

resource "aws_iam_role_policy" "snowflake_storage" {
  for_each = toset(local.snowflake_rw_environments)

  name = "s3-access"
  role = aws_iam_role.snowflake_storage[each.value].id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:GetObjectVersion",
          "s3:PutObject",
          "s3:DeleteObject"
        ]
        Resource = ["${module.data_lake[each.value].bucket_arn}/*"]
      },
      {
        Effect   = "Allow"
        Action   = ["s3:ListBucket", "s3:GetBucketLocation"]
        Resource = [module.data_lake[each.value].bucket_arn]
      }
    ]
  })
}

# -----------------------------------------------------------------------------
# Prod Read-Only Role
# -----------------------------------------------------------------------------
# Separate read-only role for developers to access prod data without write access
resource "aws_iam_role" "snowflake_storage_prod_readonly" {
  name = "snowflake-storage-prod-readonly"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = var.snowflake_storage_aws_iam_user_arn != "" ? [{
      Effect    = "Allow"
      Principal = { AWS = var.snowflake_storage_aws_iam_user_arn }
      Action    = "sts:AssumeRole"
      Condition = {
        StringEquals = {
          "sts:ExternalId" = var.snowflake_storage_external_ids.prod_readonly
        }
      }
    }] : [{
      Effect    = "Deny"
      Principal = { AWS = "*" }
      Action    = "sts:AssumeRole"
    }]
  })

  tags = {
    Name        = "snowflake-storage-prod-readonly"
    Environment = "prod"
    Purpose     = "Snowflake S3 read-only access"
    ManagedBy   = "terraform"
  }
}

resource "aws_iam_role_policy" "snowflake_storage_prod_readonly" {
  name = "s3-read-access"
  role = aws_iam_role.snowflake_storage_prod_readonly.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:GetObjectVersion"
        ]
        Resource = ["${module.data_lake["prod"].bucket_arn}/*"]
      },
      {
        Effect   = "Allow"
        Action   = ["s3:ListBucket", "s3:GetBucketLocation"]
        Resource = [module.data_lake["prod"].bucket_arn]
      }
    ]
  })
}

# -----------------------------------------------------------------------------
# Outputs
# -----------------------------------------------------------------------------
output "snowflake_storage_role_arns" {
  description = "IAM role ARNs for Snowflake storage integrations"
  value = {
    for env in local.snowflake_rw_environments :
    env => aws_iam_role.snowflake_storage[env].arn
  }
}

output "snowflake_storage_role_arn_prod_readonly" {
  description = "IAM role ARN for Snowflake prod storage integration (read-only)"
  value       = aws_iam_role.snowflake_storage_prod_readonly.arn
}
```

!!! note "Module References"
    The IAM policies reference `module.data_lake[each.value].bucket_arn` from the S3 bucket module created in [S3 Data Lake](../aws/1-s3-data-lake.md). This ensures the bucket ARNs are always correct.

## Deploy AWS Resources

Deploy the AWS IAM roles first:

```sh
cd terraform/aws
git add iam_snowflake.tf
git commit -m "Add IAM roles for Snowflake storage integrations"
git push
```

Create and merge the PR. The IAM roles will be created with placeholder trust policies (deny all) until the Snowflake integration values are added.

Note the IAM role ARNs from the Terraform outputs - you'll need these for the Snowflake integrations.

## Create Snowflake Integrations

Now create the storage integrations in Snowflake. We create separate integrations for each environment and access level.

Add to `terraform/snowflake/integrations.tf`:

```hcl
# =============================================================================
# Storage Integrations
# =============================================================================
# S3 access for data loading and unloading.

# -----------------------------------------------------------------------------
# Variables
# -----------------------------------------------------------------------------
variable "aws_account_id" {
  description = "AWS account ID for IAM role ARNs"
  type        = string
}

variable "project_name" {
  description = "Project name used in S3 bucket names"
  type        = string
}

locals {
  data_lake_environments = ["dev", "staging", "prod"]
}

# -----------------------------------------------------------------------------
# Environment Integrations (dev, staging, prod)
# -----------------------------------------------------------------------------
module "integration_data_lake" {
  source   = "./modules/snowflake_storage_integration"
  for_each = toset(local.data_lake_environments)

  providers = {
    snowflake.account_admin  = snowflake.account_admin
    snowflake.security_admin = snowflake.security_admin
  }

  integration_name    = "DATA_LAKE_${upper(each.value)}"
  integration_comment = "Access to ${each.value} data lake S3 bucket."

  storage_aws_role_arn      = "arn:aws:iam::${var.aws_account_id}:role/snowflake-storage-${each.value}"
  storage_allowed_locations = ["s3://${var.project_name}-data-lake-${each.value}/"]

  # Different role grants per environment
  grant_usage_to_roles = each.value == "prod" ? [
    # Prod: only transformers can write
    module.role_analytics_transformer.role_name,
  ] : [
    # Dev/Staging: developers and transformers can use
    module.role_analytics_developer.role_name,
    module.role_analytics_transformer.role_name,
  ]
}

# -----------------------------------------------------------------------------
# Prod Read-Only Integration
# -----------------------------------------------------------------------------
# Separate integration for developers to read production data without write access
module "integration_data_lake_prod_readonly" {
  source = "./modules/snowflake_storage_integration"

  providers = {
    snowflake.account_admin  = snowflake.account_admin
    snowflake.security_admin = snowflake.security_admin
  }

  integration_name    = "DATA_LAKE_PROD_READONLY"
  integration_comment = "Read-only access to production data lake S3 bucket."

  storage_aws_role_arn      = "arn:aws:iam::${var.aws_account_id}:role/snowflake-storage-prod-readonly"
  storage_allowed_locations = ["s3://${var.project_name}-data-lake-prod/"]

  grant_usage_to_roles = [
    module.role_analytics_developer.role_name,
  ]
}

# -----------------------------------------------------------------------------
# Outputs
# -----------------------------------------------------------------------------
output "storage_integrations" {
  description = "Storage integration details for AWS IAM configuration"
  value = {
    for env, integration in module.integration_data_lake :
    env => {
      name                     = integration.integration_name
      storage_aws_iam_user_arn = integration.storage_aws_iam_user_arn
      storage_aws_external_id  = integration.storage_aws_external_id
    }
  }
}

output "storage_integration_prod_readonly" {
  description = "Prod read-only storage integration details for AWS IAM configuration"
  value = {
    name                     = module.integration_data_lake_prod_readonly.integration_name
    storage_aws_iam_user_arn = module.integration_data_lake_prod_readonly.storage_aws_iam_user_arn
    storage_aws_external_id  = module.integration_data_lake_prod_readonly.storage_aws_external_id
  }
}
```

!!! note "Role Permissions"
    Notice the different role grants:

    - **Dev/Staging integrations**: Both `ANALYTICS_DEVELOPER` and `ANALYTICS_TRANSFORMER` can read and write
    - **Prod read-write**: Only `ANALYTICS_TRANSFORMER` can use it (for dbt to write models)
    - **Prod read-only**: `ANALYTICS_DEVELOPER` can read production data but not write

    This allows developers to analyse production data while preventing accidental writes.

## Deploy and Get Trust Values

This is a multi-step process that requires two rounds of CI/CD deployment.

### Step 1: Deploy Snowflake Integrations

Commit and push your Snowflake changes:

```sh
cd terraform/snowflake
git add modules/snowflake_storage_integration/ integrations.tf variables.tf
git commit -m "Add data lake storage integrations"
git push
```

Create and merge the PR. After CI/CD completes, the integrations exist in Snowflake.

### Step 2: Get Trust Values

After the Snowflake deployment completes, retrieve the trust values. Query Snowflake directly:

```sql
DESCRIBE INTEGRATION DATA_LAKE_DEV;
DESCRIBE INTEGRATION DATA_LAKE_STAGING;
DESCRIBE INTEGRATION DATA_LAKE_PROD;
DESCRIBE INTEGRATION DATA_LAKE_PROD_READONLY;
```

Note the `STORAGE_AWS_IAM_USER_ARN` (same for all integrations) and `STORAGE_AWS_EXTERNAL_ID` (different for each) values.

### Step 3: Update AWS Trust Policies

Now update the AWS IAM roles with the trust values from Snowflake. Add to `terraform/aws/terraform.tfvars`:

```hcl
# Snowflake storage integration trust values
# Get these from DESCRIBE INTEGRATION output
snowflake_storage_aws_iam_user_arn = "arn:aws:iam::123456789012:user/abc1-s-example1"

snowflake_storage_external_ids = {
  dev           = "ABC123_SFCRole=2_xyz123dev="
  staging       = "ABC123_SFCRole=2_xyz123staging="
  prod          = "ABC123_SFCRole=2_xyz123prod="
  prod_readonly = "ABC123_SFCRole=2_xyz123prodro="
}
```

Commit and push the AWS changes:

```sh
cd terraform/aws
git add terraform.tfvars
git commit -m "Add Snowflake trust values for storage integrations"
git push
```

Create and merge the PR. After CI/CD completes, the trust relationship is established.

## Verify the Integrations

Test that the integrations work by creating test stages:

```sql
-- Check integrations exist
SHOW INTEGRATIONS;

-- Describe the integrations
DESCRIBE INTEGRATION DATA_LAKE_DEV;
DESCRIBE INTEGRATION DATA_LAKE_STAGING;
DESCRIBE INTEGRATION DATA_LAKE_PROD;
DESCRIBE INTEGRATION DATA_LAKE_PROD_READONLY;

-- Test dev access (as ANALYTICS_DEVELOPER role)
USE ROLE ANALYTICS_DEVELOPER;

CREATE OR REPLACE STAGE test_stage_dev
  URL = 's3://your-project-data-lake-dev/'
  STORAGE_INTEGRATION = DATA_LAKE_DEV;

LIST @test_stage_dev;
DROP STAGE test_stage_dev;

-- Test staging access (as ANALYTICS_DEVELOPER role)
CREATE OR REPLACE STAGE test_stage_staging
  URL = 's3://your-project-data-lake-staging/'
  STORAGE_INTEGRATION = DATA_LAKE_STAGING;

LIST @test_stage_staging;
DROP STAGE test_stage_staging;

-- Test prod read-only access (as ANALYTICS_DEVELOPER role)
-- Developers can READ from prod but not write
CREATE OR REPLACE STAGE test_stage_prod_ro
  URL = 's3://your-project-data-lake-prod/'
  STORAGE_INTEGRATION = DATA_LAKE_PROD_READONLY;

LIST @test_stage_prod_ro;
DROP STAGE test_stage_prod_ro;

-- Test prod read-write access (as ANALYTICS_TRANSFORMER role)
USE ROLE ANALYTICS_TRANSFORMER;

CREATE OR REPLACE STAGE test_stage_prod
  URL = 's3://your-project-data-lake-prod/'
  STORAGE_INTEGRATION = DATA_LAKE_PROD;

LIST @test_stage_prod;
DROP STAGE test_stage_prod;
```

!!! note "Multi-Step CI/CD Process"
    Storage integrations require coordination between AWS and Snowflake across multiple PRs:

    1. **PR 1 (AWS)**: Create S3 buckets (completed in [S3 Data Lake](../aws/1-s3-data-lake.md))
    2. **PR 2 (AWS)**: Deploy IAM roles with placeholder trust policies
    3. **PR 3 (Snowflake)**: Deploy Snowflake integrations
    4. **Manual**: Get trust values from Snowflake
    5. **PR 4 (AWS)**: Update AWS with the trust values
    6. **Manual**: Verify access works

## Summary

You've set up secure cloud storage access for Snowflake:

- [x] Created IAM roles for dev, staging, prod, and prod read-only access
- [x] Built the `snowflake_storage_integration` module
- [x] Created storage integrations for all environments
- [x] Configured the trust relationship between Snowflake and AWS
- [x] Granted dev/staging access to developers and transformers (read-write)
- [x] Granted prod read-only access to developers
- [x] Restricted prod write access to transformers only

## What's Next

With storage integrations in place, you can load data from S3. The next section covers optional SSO setup for human user authentication.

Continue to [SSO Setup](9-sso-setup.md) →
