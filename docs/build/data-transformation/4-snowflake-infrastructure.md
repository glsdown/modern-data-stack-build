# Snowflake Infrastructure

On this page, you will:

- [x] Create the `SVC_DBT` service account in Terraform
- [x] Store dbt credentials in AWS Secrets Manager
- [x] Understand the existing access grants that cover dbt-created schemas

## Overview

dbt needs a Snowflake service account to run transformations in production and CI/CD. The required database, warehouse, and role hierarchy are already in place from the [Data Warehouse](../data-warehouse/index.md) section — this page adds only the service account.

```
Existing Snowflake Infrastructure (already built)
├── TRANSFORMING warehouse       ← dbt will use this
├── ANALYTICS database           ← dbt writes here (production)
│   ├── REPORTING schema         ← Terraform-managed; BI tool access
│   └── (staging, intermediate, marts created by dbt)
├── ANALYTICS_DEV database       ← dbt writes here (development)
├── ANALYTICS_TRANSFORMER role   ← has write access to ANALYTICS
└── ANALYTICS_DEVELOPER role     ← has read access to ANALYTICS

New (this page)
└── SVC_DBT service account
    └── SVC_DBT role (dedicated, auto-created)
        └── Granted ANALYTICS_TRANSFORMER
```

## How Existing Grants Cover dbt Schemas

dbt creates schemas dynamically when it runs (`STAGING`, `INTERMEDIATE`, `MARTS`). These schemas do not need to be pre-created in Terraform.

The `ANALYTICS` database module already configures two grant types that cover dbt-created schemas:

1. **`ANALYTICS_TRANSFORMER`** has write access at the database level via future grants — when dbt creates a new schema and writes tables to it, the transformer role has access automatically

2. **`ANALYTICS_DEVELOPER`** has read access at the database level — when dbt creates tables in `STAGING`, `INTERMEDIATE`, or `MARTS`, developer role members can query them immediately

The `REPORTING` schema is the exception — it was created explicitly in Terraform with schema-level reader grants. dbt writes to it using the transformer role, and `ANALYTICS_REPORTER` reads from it via the Terraform-configured schema reader role.

No new Terraform schema resources are needed. No dbt grants configuration is needed either.

## Create the Service Account

Open the `terraform/snowflake/` repository and add `SVC_DBT` to `users.tf`.

Following the same pattern as `SVC_DLT` and `SVC_AIRBYTE`, use `user_create_dedicated_role = true`:

```hcl
# users.tf — add to data_loaders or create a data_transformers section

# =============================================================================
# Data Transformers
# =============================================================================

module "user_svc_dbt" {
  source = "./modules/snowflake_user"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
    snowflake.user_admin     = snowflake.user_admin
  }

  user_name    = "SVC_DBT"
  user_comment = "Service account for dbt transformation runs."

  default_warehouse = module.warehouse_transforming.warehouse_name
  default_role      = "SVC_DBT"  # set after role is created by module

  # Creates a SVC_DBT role and grants it to this user
  user_create_dedicated_role = true

  # Key-pair authentication — set manually after Terraform apply
  lifecycle {
    ignore_changes = [rsa_public_key, rsa_public_key_2]
  }
}
```

### Grant ANALYTICS_TRANSFORMER to SVC_DBT

The dedicated `SVC_DBT` role needs the `ANALYTICS_TRANSFORMER` functional role so it inherits all transformation permissions:

```hcl
# users.tf — below the user module

resource "snowflake_grant_account_role" "svc_dbt_gets_analytics_transformer" {
  provider  = snowflake.security_admin
  role_name = module.role_analytics_transformer.role_name
  user_name = module.user_svc_dbt.user_name
}
```

!!! note "Why grant to the user, not the role?"
    The `SVC_DBT` dedicated role is used as the `default_role` so dbt connects with a predictable role. Granting `ANALYTICS_TRANSFORMER` directly to the `SVC_DBT` user means the user inherits those permissions regardless of which role is active. Either approach works — granting to the role is equally valid if you prefer the role inheritance model.

### Deploy the Changes

```sh
cd terraform/snowflake
git checkout -b add-svc-dbt-user
git add users.tf
git commit -m "Add SVC_DBT service account for dbt transformations"
git push
```

Open a pull request, review the plan, and merge to trigger the CI/CD pipeline.

After the pipeline completes, verify the user was created:

```sql
SHOW USERS LIKE 'SVC_DBT';
DESCRIBE USER SVC_DBT;
-- default_warehouse should be TRANSFORMING
-- default_role should be SVC_DBT
```

## Set Up Key-Pair Authentication

dbt uses key-pair authentication for `SVC_DBT`. Follow the same process as for other service accounts:

### Generate the Key Pair

```sh
# Generate private key
openssl genrsa -out svc_dbt_rsa_key.pem 2048

# Generate public key
openssl rsa -in svc_dbt_rsa_key.pem -pubout -out svc_dbt_rsa_key.pub

# Display public key (without header/footer lines)
grep -v "PUBLIC KEY" svc_dbt_rsa_key.pub | tr -d '\n'
```

### Assign the Public Key in Snowflake

```sql
USE ROLE USERADMIN;

ALTER USER SVC_DBT SET RSA_PUBLIC_KEY='<paste-public-key-here>';
```

Verify:

```sql
DESCRIBE USER SVC_DBT;
-- RSA_PUBLIC_KEY_FP should now be populated
```

### Store the Private Key in 1Password

Store the private key file in the `Service Accounts` vault in 1Password:

- **Title**: `SVC_DBT Snowflake Key Pair`
- **Username**: `SVC_DBT`
- **Private key**: contents of `svc_dbt_rsa_key.pem`

Delete the local key files:

```sh
rm svc_dbt_rsa_key.pem svc_dbt_rsa_key.pub
```

## Store Credentials in AWS Secrets Manager

dbt CI/CD retrieves credentials from AWS Secrets Manager at runtime. Add the dbt credentials to Terraform:

```hcl
# terraform/aws/secrets.tf — add alongside existing secrets

resource "aws_secretsmanager_secret" "dbt_snowflake_credentials" {
  name        = "dbt/snowflake-credentials"
  description = "Snowflake credentials for SVC_DBT (dbt transformation service account)"

  tags = {
    Project     = "data-platform"
    ManagedBy   = "terraform"
    ServiceName = "dbt"
  }
}
```

!!! note "Secret value set manually"
    The secret resource is created by Terraform but the value is set manually (following the same pattern as other service account credentials). This avoids storing the private key in Terraform state.

After `terraform apply`, set the secret value in the AWS console or CLI:

```sh
aws secretsmanager put-secret-value \
    --secret-id "dbt/snowflake-credentials" \
    --secret-string '{
        "account": "your-account.snowflakecomputing.com",
        "user": "SVC_DBT",
        "private_key": "-----BEGIN RSA PRIVATE KEY-----\n...\n-----END RSA PRIVATE KEY-----",
        "role": "SVC_DBT",
        "warehouse": "TRANSFORMING",
        "database": "ANALYTICS",
        "schema": "ANALYTICS"
    }' \
    --profile infrastructure-admin
```

Retrieve the private key from 1Password when running this command, then clear your terminal history.

## Update `profiles.yml` for Local Development

With the service account created, update the `prod` target in your local `profiles.yml` (not committed):

```yaml
# ~/.dbt/profiles.yml
dbt_transform:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: YOUR_PERSONAL_USERNAME
      authenticator: externalbrowser
      role: ANALYTICS_DEVELOPER
      warehouse: DEVELOPER
      database: ANALYTICS_DEV
      schema: ANALYTICS_DEV
      threads: 4

    prod:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: SVC_DBT
      private_key_path: "{{ env_var('SNOWFLAKE_PRIVATE_KEY_PATH') }}"
      private_key_passphrase: "{{ env_var('SNOWFLAKE_PRIVATE_KEY_PASSPHRASE', '') }}"
      role: SVC_DBT
      warehouse: TRANSFORMING
      database: ANALYTICS
      schema: ANALYTICS
      threads: 8
```

Test the production profile locally (optional — only needed for debugging):

```sh
export SNOWFLAKE_ACCOUNT="your-account.snowflakecomputing.com"
export SNOWFLAKE_PRIVATE_KEY_PATH="/path/to/svc_dbt_rsa_key.pem"

dbt debug --target prod
```

## Verify Access

Confirm `SVC_DBT` has the correct permissions:

```sql
-- Connect as SVC_DBT
USE ROLE SVC_DBT;

-- Can use the TRANSFORMING warehouse
USE WAREHOUSE TRANSFORMING;

-- Can create schemas in ANALYTICS (needed for dbt to create STAGING etc.)
USE DATABASE ANALYTICS;
CREATE SCHEMA IF NOT EXISTS STAGING;  -- should succeed
DROP SCHEMA STAGING;

-- Can write to the REPORTING schema (Terraform-managed)
USE SCHEMA REPORTING;
CREATE TABLE IF NOT EXISTS test_table (id int);  -- should succeed
DROP TABLE test_table;

-- Can read from raw source databases
USE DATABASE DLT;
SHOW SCHEMAS;  -- should show OPEN_EXCHANGE_RATES

USE DATABASE AIRBYTE;
SHOW SCHEMAS;  -- should show HUBSPOT
```

## Summary

You've set up the Snowflake infrastructure for dbt:

- [x] Created `SVC_DBT` service account with `user_create_dedicated_role = true`
- [x] Granted `ANALYTICS_TRANSFORMER` to `SVC_DBT` for write access to ANALYTICS
- [x] Configured key-pair authentication
- [x] Stored credentials in AWS Secrets Manager
- [x] Confirmed existing database-level grants cover dbt-created schemas

## What's Next

Define your raw data sources and build the first staging models.

Continue to [Sources and Staging](5-sources-and-staging.md) →
