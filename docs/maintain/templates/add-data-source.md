---
name: add-data-source
description: Add a new data source to the Snowflake infrastructure, including database, service account, schemas, role grants, and AWS Secrets Manager container. Follows the established pattern used for DLT, AIRBYTE, SNOWPIPE, and STREAMING sources.
allowed-tools: Read, Write, Glob, Grep, Bash
---

# Add a New Data Source

This skill adds complete Snowflake infrastructure for a new data source, following the pattern used for existing loader databases.

## When to Use

- Adding a new data loader tool (e.g. Fivetran, Stitch, a custom loader)
- Adding a new category of data that needs its own database
- Setting up infrastructure before building ingestion pipelines

## Before You Start

Gather the following information:

1. **Loader tool name** - used for database and service account naming (e.g. `FIVETRAN`)
2. **Schema names** - one per data source (e.g. `HUBSPOT`, `STRIPE`, `SALESFORCE`)
3. **Whether the loader needs a storage integration** - S3 access for file-based loading
4. **Whether to use Snowpipe** - auto-ingest from S3 vs direct loading
5. **Whether the loader needs an AWS secret** - credentials for the service account

## Reference: Existing Data Sources

Read `snowflake/config/databases.tf` and `snowflake/config/users.tf` for existing patterns:

| Database | Service Account | Schemas | Pattern |
|----------|----------------|---------|---------|
| `DLT` | `SVC_DLT` | OPEN_EXCHANGE_RATES, APPLICATION_DATA, HUBSPOT | Direct load via dlt |
| `SNOWPIPE` | (none - auto-ingest) | OPEN_EXCHANGE_RATES | S3 auto-ingest |
| `AIRBYTE` | `SVC_AIRBYTE` | HUBSPOT | Connector load via Airbyte |
| `STREAMING` | `SVC_KAFKA_CONNECTOR` | ORDER_EVENTS | Kafka sink connector |

## Steps

### 1. Create the Database

Add to `snowflake/config/databases.tf`:

```hcl
module "database_<loader>" {
  source = "./modules/snowflake_database"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  database_name    = "<LOADER>"
  database_comment = "Raw data loaded by <loader>."

  grant_reader_to_account_roles = [
    module.role_analytics_sources_reader.role_name,
  ]
  grant_writer_to_account_roles = [
    module.user_svc_<loader>.user_default_role,
  ]
}
```

Replace `<LOADER>` with the tool name in UPPER_CASE and `<loader>` in lowercase.

The database module automatically creates `<LOADER>_DB_READER` and `<LOADER>_DB_WRITER` database roles. Granting the reader to `ANALYTICS_SOURCES_READER` maintains the reader access chain so downstream developers and transformers can query the data.

### 2. Create the Service Account

Add to `snowflake/config/users.tf`:

```hcl
module "user_svc_<loader>" {
  source = "./modules/snowflake_user"

  providers = {
    snowflake.security_admin = snowflake.security_admin
    snowflake.user_admin     = snowflake.user_admin
  }

  user_name                  = "SVC_<LOADER>"
  user_display_name          = "<Loader> Service Account"
  user_comment               = "Service account for <loader> data loading."
  user_is_service_account    = true
  user_create_dedicated_role = true
  user_default_warehouse     = module.warehouse_loading.warehouse_name

  user_additional_roles = []
}
```

Setting `user_create_dedicated_role = true` creates a role named `SVC_<LOADER>` that the database module can reference for writer grants.

### 3. Create Schemas

Add to `snowflake/config/schemas.tf` (or the relevant file where schemas are defined):

```hcl
module "schema_<loader>_<source>" {
  source = "./modules/snowflake_schema"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  database_name  = module.database_<loader>.database_name
  schema_name    = "<SOURCE>"
  schema_comment = "Data from <source> loaded by <loader>."
}
```

Repeat for each schema (data source) within the database.

### 4. Add AWS Secrets Manager Container (if Needed)

If the service account needs credentials stored for CI/CD, add to `aws/config/secrets.tf`:

```hcl
resource "aws_secretsmanager_secret" "<loader>_snowflake_credentials" {
  name        = "<loader>/snowflake-credentials"
  description = "Snowflake credentials for SVC_<LOADER>."
}
```

The actual secret value (account, user, private key) is set manually via the AWS CLI after Terraform creates the container.

### 5. Add Storage Integration (if Needed)

If the loader reads from S3, add a storage integration in `snowflake/config/storage_integrations.tf` using the `snowflake_storage_integration` module. Reference the existing S3 data lake bucket patterns.

### 6. Add Snowpipe (if Needed)

If using auto-ingest from S3, add a Snowpipe definition using the `snowflake_snowpipe` module. This creates the stage, file format, pipe, and SQS event notification. See the SNOWPIPE database setup for a working example.

### 7. Validate

Run from both directories:

```sh
cd snowflake/config && terraform plan
cd ../../aws/config && terraform plan
```

Verify:

- Database created with correct name
- `DB_READER` granted to `ANALYTICS_SOURCES_READER`
- `DB_WRITER` granted to the service account's dedicated role
- Service account created with dedicated role and `LOADING` warehouse
- Schema(s) created in the correct database
- AWS secret container created (if applicable)
- No unexpected changes to existing resources

### 8. Create Pull Request

Commit, push, and create a PR. CI/CD validates with `terraform plan` and applies after approval.

## Safety Checks

- Database names must be UPPER_CASE
- Service account names must start with `SVC_`
- **Always** grant `DB_READER` to `ANALYTICS_SOURCES_READER` (maintains the reader access chain)
- **Never** hard-code Snowflake account IDs or ARNs
- **Never** hard-code secret values in Terraform - use containers and set values via CLI
- Run `terraform plan` in **both** `snowflake/config/` and `aws/config/` before creating a PR
- The service account module call must come before the database module call if the database references `user_svc_<loader>.user_default_role` - or use `depends_on` as needed
