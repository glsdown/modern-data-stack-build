# Snowflake Infrastructure

On this page, you will:

- [x] Create the DLT and SNOWPIPE loader databases
- [x] Create the SVC_DLT service account with a dedicated role
- [x] Set up schemas for each data source
- [x] Build a reusable Snowpipe Terraform module
- [x] Create secret containers in AWS Secrets Manager

## Overview

Before running dlt pipelines, you need Snowflake infrastructure to receive the data. This follows the same Terraform patterns established in the [Data Warehouse](../data-warehouse/index.md) section.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         LOADER INFRASTRUCTURE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  DLT Database                         SNOWPIPE Database                     │
│  ┌─────────────────────────────┐      ┌─────────────────────────────┐       │
│  │  OPEN_EXCHANGE_RATES schema │      │  OPEN_EXCHANGE_RATES schema │       │
│  │  └── CURRENCIES table       │      │  └── EXCHANGE_RATES table   │       │
│  │                             │      │                             │       │
│  │  APPLICATION_DATA schema    │      │  (External stage + pipe)    │       │
│  │  └── PRODUCTS table         │      │                             │       │
│  │                             │      └─────────────────────────────┘       │
│  │  HUBSPOT schema (optional)  │                  │                         │
│  │  └── CONTACTS table etc.    │                  │                         │
│  └─────────────────────────────┘                  │                         │
│              │                                    │                         │
│              └──────────┬─────────────────────────┘                         │
│                         ▼                                                   │
│              ┌─────────────────────┐                                        │
│              │ ANALYTICS_SOURCES_  │                                        │
│              │ READER role         │                                        │
│              └─────────────────────┘                                        │
│                         │                                                   │
│                         ▼                                                   │
│              ┌───────────────────────┐                                      │
│              │ ANALYTICS_DEVELOPER   │                                      │
│              │ ANALYTICS_TRANSFORMER │                                      │
│              └───────────────────────┘                                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

Data flows into two databases depending on the loading mechanism:

- **DLT database** — Data loaded directly by dlt pipelines (API → Snowflake). Currencies, products, and HubSpot data land here.
- **SNOWPIPE database** — Data loaded via Snowpipe auto-ingestion from S3. Exchange rates are written to S3 by dlt and auto-ingested by Snowpipe.

## Prerequisites

Ensure you have completed:

- [x] [Data Warehouse Setup](../data-warehouse/index.md) — Terraform modules for databases, roles, users
- [x] [S3 Data Lake](../aws/1-s3-data-lake.md) — S3 buckets for staging data
- [x] [Storage Integrations](../data-warehouse/8-storage-integrations.md) — Snowflake ↔ S3 trust relationship

## Navigate to Your Terraform Directory

```sh
cd ~/projects/data/data-stack-infrastructure/terraform/snowflake
```

## DLT Database

The DLT database stores data loaded directly by dlt pipelines. Create the database using the existing `snowflake_database` module.

Add to `databases.tf`:

```hcl
# -----------------------------------------------------------------------------
# DLT (dlt loader database)
# -----------------------------------------------------------------------------
# Raw data loaded by dlt pipelines from APIs and databases
module "database_dlt" {
  source = "./modules/snowflake_database"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  database_name    = "DLT"
  database_comment = "Raw data loaded by dlt pipelines."

  grant_reader_to_account_roles = [
    module.role_analytics_sources_reader.role_name,
  ]
  grant_writer_to_account_roles = [
    module.user_svc_dlt.user_default_role,  # SVC_DLT dedicated role
  ]
}
```

This creates:

- The `DLT` database
- `DLT_DB_READER` database role (granted to `ANALYTICS_SOURCES_READER`)
- `DLT_DB_WRITER` database role (granted to the `SVC_DLT` dedicated role)

## SNOWPIPE Database

The SNOWPIPE database stores data loaded via Snowpipe auto-ingestion from S3. This is separate from DLT because the loader mechanism is different — Snowpipe uses the storage integration (not a user role) to write data.

Add to `databases.tf`:

```hcl
# -----------------------------------------------------------------------------
# SNOWPIPE (Snowpipe loader database)
# -----------------------------------------------------------------------------
# Raw data loaded via Snowpipe auto-ingestion from S3
module "database_snowpipe" {
  source = "./modules/snowflake_database"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  database_name    = "SNOWPIPE"
  database_comment = "Raw data loaded via Snowpipe auto-ingestion from S3."

  grant_reader_to_account_roles = [
    module.role_analytics_sources_reader.role_name,
  ]
  # Snowpipe uses the storage integration, not a role for loading
  grant_writer_to_account_roles = []
}
```

## SVC_DLT Service Account

Create the service account that dlt pipelines use to connect to Snowflake. The `user_create_dedicated_role = true` setting creates a role named `SVC_DLT` that matches the username — this is the same pattern described in [Users](../data-warehouse/5-users.md) for service accounts that need their own role for database writes.

Add to `users.tf`:

```hcl
# -----------------------------------------------------------------------------
# dlt Service Account
# -----------------------------------------------------------------------------
module "user_svc_dlt" {
  source = "./modules/snowflake_user"

  providers = {
    snowflake.security_admin = snowflake.security_admin
    snowflake.user_admin     = snowflake.user_admin
  }

  user_name               = "SVC_DLT"
  user_comment            = "Service account for dlt data pipelines."
  user_display_name       = "dlt Service Account"
  user_is_service_account = true

  user_default_warehouse      = module.warehouse_loading.warehouse_name
  user_create_dedicated_role  = true
}
```

Grant the `SVC_DLT` role usage on the LOADING warehouse:

```hcl
# Grant warehouse usage to SVC_DLT
resource "snowflake_grant_privileges_to_account_role" "svc_dlt_warehouse" {
  provider          = snowflake.security_admin
  account_role_name = module.user_svc_dlt.user_default_role
  privileges        = ["USAGE"]

  on_account_object {
    object_type = "WAREHOUSE"
    object_name = module.warehouse_loading.warehouse_name
  }
}
```

!!! info "Why a Dedicated Role?"
    A dedicated role (instead of a shared functional role like `DLT_LOADER`) gives you precise control. If you later add another dlt service account for a different team, each gets its own role with independent database grants. The [Users](../data-warehouse/5-users.md) page explains this pattern in detail.

## Source Schemas

dlt can create schemas automatically, but defining them in Terraform ensures they exist before the first pipeline run and maintains consistency.

!!! tip "Managing Schemas in Terraform"
    Read the guidance on managing schemas in the [Data Warehouse Schemas](../data-warehouse/6-schemas.md) page.

Add to `schemas.tf`:

```hcl
# =============================================================================
# DLT Database Schemas
# =============================================================================
# Source-based schemas for dlt-loaded data

# -----------------------------------------------------------------------------
# DLT.OPEN_EXCHANGE_RATES
# -----------------------------------------------------------------------------
# Currencies data from Open Exchange Rates API
module "schema_dlt_open_exchange_rates" {
  source = "./modules/snowflake_schema"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  schema_name     = "OPEN_EXCHANGE_RATES"
  schema_database = module.database_dlt.database_name
  schema_comment  = "Currencies data from Open Exchange Rates API."

  # Access via DLT_DB_READER, no additional schema-level grants needed
  grant_reader_to_account_roles = []
}

# -----------------------------------------------------------------------------
# DLT.APPLICATION_DATA
# -----------------------------------------------------------------------------
# Application data from external databases (PostgreSQL, etc.)
module "schema_dlt_application_data" {
  source = "./modules/snowflake_schema"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  schema_name     = "APPLICATION_DATA"
  schema_database = module.database_dlt.database_name
  schema_comment  = "Application data from external databases."

  grant_reader_to_account_roles = []
}

# -----------------------------------------------------------------------------
# DLT.HUBSPOT (Optional)
# -----------------------------------------------------------------------------
# HubSpot CRM data loaded by dlt. Skip this if you plan to use Airbyte
# for HubSpot instead (see the SaaS Ingestion section).
module "schema_dlt_hubspot" {
  source = "./modules/snowflake_schema"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  schema_name     = "HUBSPOT"
  schema_database = module.database_dlt.database_name
  schema_comment  = "HubSpot CRM data loaded by dlt pipelines."

  grant_reader_to_account_roles = []
}

# =============================================================================
# SNOWPIPE Database Schemas
# =============================================================================
# Source-based schemas for Snowpipe-loaded data

# -----------------------------------------------------------------------------
# SNOWPIPE.OPEN_EXCHANGE_RATES
# -----------------------------------------------------------------------------
# Exchange rates auto-ingested via Snowpipe from S3
module "schema_snowpipe_open_exchange_rates" {
  source = "./modules/snowflake_schema"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  schema_name     = "OPEN_EXCHANGE_RATES"
  schema_database = module.database_snowpipe.database_name
  schema_comment  = "Exchange rates data loaded via Snowpipe from S3."

  grant_reader_to_account_roles = []
}
```

## Snowpipe Module

Snowpipe auto-ingestion involves three closely related resources: an external stage (pointing to S3), a file format (for parsing the data), and a pipe (the auto-ingest definition). Bundling these into a reusable module keeps things consistent when you add more Snowpipes later.

```sh
mkdir -p modules/snowflake_snowpipe
```

### main.tf

Create `modules/snowflake_snowpipe/main.tf`:

```hcl
terraform {
  required_providers {
    snowflake = {
      source                = "Snowflake-Labs/snowflake"
      version               = "~> 0.99"
      configuration_aliases = [snowflake.sys_admin]
    }
  }
}

# -----------------------------------------------------------------------------
# External Stage
# -----------------------------------------------------------------------------
# Points to the S3 location where dlt writes files
resource "snowflake_stage" "this" {
  provider = snowflake.sys_admin

  database = var.database
  schema   = var.schema
  name     = "${var.name}_STAGE"
  comment  = "S3 stage for ${lower(var.name)} data from dlt pipeline."

  url                 = var.stage_url
  storage_integration = var.storage_integration
}

# -----------------------------------------------------------------------------
# File Format
# -----------------------------------------------------------------------------
# Defines how Snowpipe parses the incoming files
resource "snowflake_file_format" "this" {
  provider = snowflake.sys_admin

  database = var.database
  schema   = var.schema
  name     = "${var.name}_FORMAT"
  comment  = "File format for ${lower(var.name)} Snowpipe ingestion."

  format_type = var.format_type

  # JSON-specific options
  strip_outer_array = var.format_type == "JSON" ? true : null
  date_format       = "AUTO"
  timestamp_format  = "AUTO"
}

# -----------------------------------------------------------------------------
# Snowpipe
# -----------------------------------------------------------------------------
# Auto-ingests data from S3 when new files arrive
resource "snowflake_pipe" "this" {
  provider = snowflake.sys_admin

  database = var.database
  schema   = var.schema
  name     = "${var.name}_PIPE"
  comment  = "Auto-ingest ${lower(var.name)} data from S3."

  copy_statement = <<-SQL
    COPY INTO ${var.database}.${var.schema}.${var.target_table}
    FROM @${var.database}.${var.schema}.${var.name}_STAGE
    FILE_FORMAT = (FORMAT_NAME = '${var.database}.${var.schema}.${var.name}_FORMAT')
  SQL

  auto_ingest = true
}
```

### variables.tf

Create `modules/snowflake_snowpipe/variables.tf`:

```hcl
variable "database" {
  description = "Database where the Snowpipe resources are created"
  type        = string
}

variable "schema" {
  description = "Schema where the Snowpipe resources are created"
  type        = string
}

variable "name" {
  description = "Name prefix for stage, format, and pipe (e.g. EXCHANGE_RATES)"
  type        = string
}

variable "stage_url" {
  description = "S3 URL for the external stage (e.g. s3://bucket/prefix/)"
  type        = string
}

variable "storage_integration" {
  description = "Name of the Snowflake storage integration for S3 access"
  type        = string
}

variable "target_table" {
  description = "Target table name for the COPY INTO statement"
  type        = string
}

variable "format_type" {
  description = "File format type (JSON, CSV, PARQUET, etc.)"
  type        = string
  default     = "JSON"
}
```

### outputs.tf

Create `modules/snowflake_snowpipe/outputs.tf`:

```hcl
output "pipe_name" {
  description = "The Snowpipe name"
  value       = snowflake_pipe.this.name
}

output "notification_channel" {
  description = "SQS queue ARN for S3 event notifications"
  value       = snowflake_pipe.this.notification_channel
}

output "stage_name" {
  description = "The external stage name"
  value       = snowflake_stage.this.name
}
```

!!! note "Using the Module"
    You've defined the module here but won't use it until [Exchange Rates to S3](6-exchange-rates-to-s3.md), where you'll create the actual Snowpipe alongside the pipeline that writes data to S3. This keeps the Snowpipe configuration in context with the pipeline that feeds it.

## Secret Containers

The pipeline credentials need to be stored in AWS Secrets Manager. Following the pattern from [Secrets Manager Setup](../../getting-started/terraform-setup/aws/7-secrets-manager-setup.md), you create the secret containers in Terraform and set values via the CLI.

Navigate to your AWS Terraform directory:

```sh
cd ~/projects/data/data-stack-infrastructure/terraform/aws
```

Add to `secrets.tf`:

```hcl
# -----------------------------------------------------------------------------
# dlt Pipeline Secrets
# -----------------------------------------------------------------------------

resource "aws_secretsmanager_secret" "dlt_snowflake_credentials" {
  name        = "dlt/snowflake-credentials"
  description = "Snowflake service account credentials for dlt pipelines"

  tags = {
    Name        = "dlt/snowflake-credentials"
    ManagedBy   = "terraform"
    Environment = "all"
  }
}

resource "aws_secretsmanager_secret" "dlt_open_exchange_rates" {
  name        = "dlt/open-exchange-rates"
  description = "Open Exchange Rates API key for dlt pipelines"

  tags = {
    Name        = "dlt/open-exchange-rates"
    ManagedBy   = "terraform"
    Environment = "all"
  }
}

resource "aws_secretsmanager_secret" "dlt_clever_cloud_postgres" {
  name        = "dlt/clever-cloud-postgres"
  description = "Clever Cloud PostgreSQL credentials for dlt pipelines"

  tags = {
    Name        = "dlt/clever-cloud-postgres"
    ManagedBy   = "terraform"
    Environment = "all"
  }
}

resource "aws_secretsmanager_secret" "dlt_hubspot_api_key" {
  name        = "dlt/hubspot-api-key"
  description = "HubSpot API key for dlt pipelines"

  tags = {
    Name        = "dlt/hubspot-api-key"
    ManagedBy   = "terraform"
    Environment = "all"
  }
}
```

!!! info "Why No Secret Values?"
    Secret containers are managed by Terraform, but the values are set via CLI to keep them out of Terraform state. See [Secrets Manager Setup](../../getting-started/terraform-setup/aws/7-secrets-manager-setup.md) for the full explanation.

## Generate SVC_DLT Key Pair

The SVC_DLT user uses key-pair authentication rather than a password. This is more secure and avoids password rotation concerns.

Generate the key pair:

```sh
# Generate private key (PKCS#8 format, unencrypted)
openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out svc_dlt_rsa_key.p8 -nocrypt

# Generate public key
openssl rsa -in svc_dlt_rsa_key.p8 -pubout -out svc_dlt_rsa_key.pub

# Display public key for Snowflake (remove header/footer)
grep -v "PUBLIC KEY" svc_dlt_rsa_key.pub | tr -d '\n'
```

Set the public key in Snowflake:

```sql
ALTER USER SVC_DLT SET RSA_PUBLIC_KEY = 'MIIBIjANBgkqhki...';
```

!!! warning "Remove Headers"
    When setting the RSA public key, strip the `-----BEGIN PUBLIC KEY-----` and `-----END PUBLIC KEY-----` headers and any newlines.

Store the credentials in AWS Secrets Manager:

```sh
aws secretsmanager put-secret-value \
  --secret-id "dlt/snowflake-credentials" \
  --secret-string '{
    "database": "DLT",
    "warehouse": "LOADING",
    "role": "SVC_DLT",
    "username": "SVC_DLT",
    "host": "orgname-accountname.snowflakecomputing.com",
    "private_key": "'"$(base64 < svc_dlt_rsa_key.p8)"'"
  }' \
  --profile infrastructure-admin
```

!!! tip "Avoid Shell History"
    To keep secrets out of your shell history, pipe the value from a password manager:

    ```sh
    aws secretsmanager put-secret-value \
      --secret-id "dlt/snowflake-credentials" \
      --secret-string "$(op item get 'dlt Snowflake Credentials' --format json | jq -c '...')" \
      --profile infrastructure-admin
    ```

Clean up the key files from your local machine:

```sh
rm svc_dlt_rsa_key.p8 svc_dlt_rsa_key.pub
```

## Deployment

Deploy the infrastructure in two stages. The Snowflake and AWS changes go through separate CI/CD pipelines.

### Snowflake Changes

=== "CI/CD"

    Create a PR with the Snowflake changes:

    ```sh
    cd ~/projects/data/data-stack-infrastructure
    git checkout -b feat/dlt-snowflake-infrastructure
    git add terraform/snowflake/databases.tf terraform/snowflake/users.tf \
            terraform/snowflake/schemas.tf terraform/snowflake/modules/snowflake_snowpipe/
    git commit -m "Add DLT and SNOWPIPE databases with loader infrastructure"
    git push -u origin feat/dlt-snowflake-infrastructure
    ```

    Open a PR. The CI/CD pipeline runs `terraform plan` automatically. Review the plan, then merge to apply.

=== "Local"

    ```sh
    cd ~/projects/data/data-stack-infrastructure/terraform/snowflake
    terraform plan
    terraform apply
    ```

### AWS Changes

=== "CI/CD"

    Create a PR with the AWS secret containers:

    ```sh
    cd ~/projects/data/data-stack-infrastructure
    git checkout -b feat/dlt-secret-containers
    git add terraform/aws/secrets.tf
    git commit -m "Add secret containers for dlt pipeline credentials"
    git push -u origin feat/dlt-secret-containers
    ```

    Open a PR, review the plan, then merge to apply.

=== "Local"

    ```sh
    cd ~/projects/data/data-stack-infrastructure/terraform/aws
    terraform plan
    terraform apply
    ```

After the AWS changes are deployed, set the secret values using the CLI commands shown above.

## Verify

After deploying, verify the infrastructure:

```sql
-- Check databases
SHOW DATABASES LIKE 'DLT';
SHOW DATABASES LIKE 'SNOWPIPE';

-- Check schemas
SHOW SCHEMAS IN DATABASE DLT;
-- Should show: OPEN_EXCHANGE_RATES, APPLICATION_DATA, HUBSPOT
SHOW SCHEMAS IN DATABASE SNOWPIPE;
-- Should show: OPEN_EXCHANGE_RATES

-- Check user and dedicated role
SHOW USERS LIKE 'SVC_DLT';
DESCRIBE USER SVC_DLT;
-- default_role should be SVC_DLT

SHOW ROLES LIKE 'SVC_DLT';
SHOW GRANTS TO ROLE SVC_DLT;
-- Should include: DLT_DB_WRITER, LOADING warehouse USAGE

-- Check role grants chain
SHOW GRANTS OF DATABASE ROLE DLT.DLT_DB_READER;
-- Should show: ANALYTICS_SOURCES_READER

-- Check reader access (as ANALYTICS_DEVELOPER)
USE ROLE ANALYTICS_DEVELOPER;
SELECT CURRENT_ROLE();
SHOW SCHEMAS IN DATABASE DLT;  -- Should work
```

Verify the AWS secrets exist:

```sh
aws secretsmanager list-secrets \
  --filter Key=name,Values=dlt/ \
  --profile infrastructure-admin
```

## Access Model Summary

Here's how access flows through the role hierarchy:

| User/Role | DLT Database | SNOWPIPE Database | LOADING Warehouse |
|-----------|--------------|-------------------|-------------------|
| `SVC_DLT` (via dedicated role) | Read/Write | Read only | Usage |
| `ANALYTICS_DEVELOPER` | Read (via `ANALYTICS_SOURCES_READER`) | Read | - |
| `ANALYTICS_TRANSFORMER` | Read (via `ANALYTICS_SOURCES_READER`) | Read | - |

The `SVC_DLT` dedicated role has write access to the DLT database. Snowpipe uses the storage integration (not a user role) to write to the SNOWPIPE database.

## Summary

You've created the Snowflake infrastructure for dlt data loading:

- [x] DLT database for dlt-loaded data (currencies, products, HubSpot)
- [x] SNOWPIPE database for auto-ingested data (exchange rates)
- [x] SVC_DLT service account with dedicated role
- [x] Source-based schemas (OPEN_EXCHANGE_RATES, APPLICATION_DATA, HUBSPOT)
- [x] Reusable Snowpipe Terraform module
- [x] Secret containers in AWS Secrets Manager
- [x] Key-pair authentication for SVC_DLT

## What's Next

With infrastructure in place, you need to configure the credentials that dlt will use to connect to data sources and Snowflake.

Continue to [Credentials Setup](4-credentials-setup.md) →
