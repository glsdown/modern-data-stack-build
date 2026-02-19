# Snowflake Infrastructure

On this page, you will:

- [x] Create the AIRBYTE database using the existing database module
- [x] Create the SVC_AIRBYTE service account with a dedicated role
- [x] Configure role grants for writer and reader access
- [x] Add the HUBSPOT schema

## Overview

Airbyte needs a Snowflake database to load data into, and a service account to authenticate with. This follows the same Terraform module patterns used in the [Data Warehouse](../data-warehouse/index.md) section.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SNOWFLAKE INFRASTRUCTURE FOR AIRBYTE                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Database                    Roles                    Access                │
│  ────────                    ─────                    ──────                │
│                                                                             │
│  ┌─────────────────┐         ┌──────────────────┐                           │
│  │ AIRBYTE         │         │ SVC_AIRBYTE      │  (dedicated role)         │
│  │ ├── HUBSPOT     │◀────────│ (service account)│                           │
│  │ │   └─ CONTACTS │  write  └──────────────────┘                           │
│  │ └── (future)    │                │                                       │
│  └─────────────────┘                │ granted                               │
│         │                           ▼                                       │
│         │              ┌──────────────────────┐                             │
│         │              │ AIRBYTE_DB_WRITER    │  (database role)            │
│         │              └──────────────────────┘                             │
│         │                                                                   │
│         │              ┌──────────────────────┐                             │
│         └──────────────│ AIRBYTE_DB_READER    │  (database role)            │
│              read      └──────────┬───────────┘                             │
│                                   │ granted to                              │
│                                   ▼                                         │
│                        ┌──────────────────────┐                             │
│                        │ ANALYTICS_SOURCES_   │  (account role)             │
│                        │ READER               │                             │
│                        └──────────────────────┘                             │
│                                   │                                         │
│                                   ▼                                         │
│                        All analytics roles can read Airbyte data            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Prerequisites

Ensure you have:

- [x] [Snowflake Terraform Setup](../../getting-started/terraform-setup/snowflake/index.md) - Snowflake managed by Terraform
- [x] [Data Warehouse](../data-warehouse/index.md) - Database, schema, and user modules available
- [x] `ANALYTICS_SOURCES_READER` role exists (from [Functional Roles](../data-warehouse/4-functional-roles.md))

## Create the AIRBYTE Database

Use the existing database module to create the AIRBYTE database. The module automatically creates `AIRBYTE_DB_READER` and `AIRBYTE_DB_WRITER` database roles.

```hcl
# terraform/snowflake/databases.tf

module "database_airbyte" {
  source = "./modules/snowflake_database"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  database_name    = "AIRBYTE"
  database_comment = "Raw data loaded by Airbyte connectors."

  # Grant AIRBYTE_DB_READER to ANALYTICS_SOURCES_READER
  # so all analytics roles can read Airbyte-loaded data
  grant_reader_to_account_roles = [
    module.role_analytics_sources_reader.role_name,
  ]

  # Grant AIRBYTE_DB_WRITER to SVC_AIRBYTE's dedicated role
  grant_writer_to_account_roles = [
    module.user_svc_airbyte.user_default_role,
  ]
}
```

This creates:

| Object | Type | Purpose |
|--------|------|---------|
| `AIRBYTE` | Database | Container for all Airbyte-loaded data |
| `AIRBYTE_DB_READER` | Database role | Read access to all schemas/tables |
| `AIRBYTE_DB_WRITER` | Database role | Write access to all schemas/tables |

The `grant_reader_to_account_roles` parameter grants `AIRBYTE_DB_READER` to `ANALYTICS_SOURCES_READER`. This means all roles in the analytics hierarchy (e.g., `ANALYTICS_DEVELOPER`, `ANALYTICS_ANALYST`) can read Airbyte data without any additional grants.

## Create the SVC_AIRBYTE Service Account

Use the user module with `user_create_dedicated_role = true` to create a service account with its own role.

```hcl
# terraform/snowflake/users.tf

module "user_svc_airbyte" {
  source = "./modules/snowflake_user"

  providers = {
    snowflake.security_admin = snowflake.security_admin
    snowflake.user_admin     = snowflake.user_admin
  }

  user_name                  = "SVC_AIRBYTE"
  user_comment               = "Service account for Airbyte data loading"
  user_display_name          = "SVC_AIRBYTE"
  user_is_service_account    = true
  user_create_dedicated_role = true  # Creates SVC_AIRBYTE role automatically

  user_default_warehouse = module.warehouse_loading.warehouse_name
  # user_default_role is set automatically to the dedicated role (SVC_AIRBYTE)
}
```

The `user_create_dedicated_role = true` flag:

1. Creates a `SVC_AIRBYTE` account role
2. Grants the role to the `SVC_AIRBYTE` user
3. Sets it as the user's default role

### Grant Warehouse Usage

The service account needs warehouse access to execute queries:

```hcl
# terraform/snowflake/users.tf

resource "snowflake_grant_privileges_to_account_role" "svc_airbyte_warehouse" {
  provider          = snowflake.security_admin
  account_role_name = module.user_svc_airbyte.user_default_role
  privileges        = ["USAGE"]

  on_account_object {
    object_type = "WAREHOUSE"
    object_name = module.warehouse_loading.warehouse_name
  }
}
```

## Add the HUBSPOT Schema

Create the HUBSPOT schema in the AIRBYTE database:

```hcl
# terraform/snowflake/schemas.tf

module "schema_airbyte_hubspot" {
  source = "./modules/snowflake_schema"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  schema_name          = "HUBSPOT"
  schema_database_name = "AIRBYTE"
  schema_comment       = "HubSpot CRM data loaded via Airbyte."
}
```

!!! note "Schema Pre-creation"
    Airbyte can create schemas automatically when syncing. Pre-creating them in Terraform ensures they are managed as infrastructure and follow your naming conventions.

## Generate Key Pair for SVC_AIRBYTE

Generate an RSA key pair for the service account, following the same pattern as SVC_DLT from [Snowflake Infrastructure](../batch-data-ingestion/3-snowflake-infrastructure.md).

```sh
# Generate private key
openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out svc_airbyte_rsa_key.p8 -nocrypt

# Generate public key
openssl rsa -in svc_airbyte_rsa_key.p8 -pubout -out svc_airbyte_rsa_key.pub
```

Set the public key on the Snowflake user:

```sql
ALTER USER SVC_AIRBYTE SET RSA_PUBLIC_KEY = 'MIIBIjANBgkqhki...';
```

!!! warning "Remove Headers"
    When setting the RSA public key, strip the `-----BEGIN PUBLIC KEY-----` and `-----END PUBLIC KEY-----` headers and any newlines.

### Store Credentials in AWS Secrets Manager

The `airbyte/snowflake-credentials` secret container was created in [Airbyte Cloud Setup](3-airbyte-cloud-setup.md). Set the value with the generated private key:

```sh
aws secretsmanager put-secret-value \
    --secret-id "airbyte/snowflake-credentials" \
    --secret-string '{
        "account": "orgname-accountname",
        "username": "SVC_AIRBYTE",
        "private_key": "'"$(cat svc_airbyte_rsa_key.p8 | base64)"'",
        "database": "AIRBYTE",
        "warehouse": "LOADING",
        "role": "SVC_AIRBYTE"
    }' \
    --profile infrastructure-admin
```

Delete local key files after storing:

```sh
rm svc_airbyte_rsa_key.p8 svc_airbyte_rsa_key.pub
```

## Deploy

=== "CI/CD"

    Commit and push the changes to trigger the Snowflake Terraform CI/CD pipeline:

    ```sh
    git add terraform/snowflake/
    git commit -m "Add AIRBYTE database, SVC_AIRBYTE service account, and HUBSPOT schema"
    git push
    ```

    Create a PR to review the plan. After merging, Terraform applies the changes automatically.

=== "Local"

    ```sh
    cd terraform/snowflake
    terraform plan
    terraform apply
    ```

## Verify

### Check Database and Schema

```sql
-- Database exists
SHOW DATABASES LIKE 'AIRBYTE';

-- Schema exists
SHOW SCHEMAS IN DATABASE AIRBYTE;
-- Should show: HUBSPOT

-- Database roles exist
SHOW DATABASE ROLES IN DATABASE AIRBYTE;
-- Should show: AIRBYTE_DB_READER, AIRBYTE_DB_WRITER
```

### Check Service Account

```sql
-- User exists
SHOW USERS LIKE 'SVC_AIRBYTE';
DESCRIBE USER SVC_AIRBYTE;
-- default_role should be SVC_AIRBYTE
-- default_warehouse should be LOADING

-- Role exists
SHOW ROLES LIKE 'SVC_AIRBYTE';

-- Role grants
SHOW GRANTS TO ROLE SVC_AIRBYTE;
-- Should include AIRBYTE_DB_WRITER and LOADING warehouse
```

### Check Reader Access Chain

```sql
-- AIRBYTE_DB_READER is granted to ANALYTICS_SOURCES_READER
SHOW GRANTS OF DATABASE ROLE AIRBYTE.AIRBYTE_DB_READER;
-- Should show: ANALYTICS_SOURCES_READER

-- Verify analyst can read (once data is loaded)
USE ROLE ANALYTICS_DEVELOPER;
SELECT CURRENT_ROLE();  -- Should show ANALYTICS_DEVELOPER
-- The following will work once Airbyte loads data:
-- SELECT * FROM AIRBYTE.HUBSPOT.CONTACTS LIMIT 5;
```

## Role Hierarchy Summary

```
ACCOUNTADMIN
└── SYSADMIN
    └── SVC_AIRBYTE (dedicated role, created by user module)
        └── AIRBYTE_DB_WRITER (database role, created by database module)
            └── Write access to AIRBYTE database

ANALYTICS_SOURCES_READER (account role)
└── AIRBYTE_DB_READER (database role, granted by database module)
    └── Read access to AIRBYTE database
        └── ANALYTICS_DEVELOPER, ANALYTICS_ANALYST inherit this
```

This mirrors the pattern used for the DLT database:

| Loader | Database | Writer Role | Reader Chain |
|--------|----------|------------|-------------|
| dlt | `DLT` | `SVC_DLT` via `DLT_DB_WRITER` | `DLT_DB_READER` → `ANALYTICS_SOURCES_READER` |
| Airbyte | `AIRBYTE` | `SVC_AIRBYTE` via `AIRBYTE_DB_WRITER` | `AIRBYTE_DB_READER` → `ANALYTICS_SOURCES_READER` |

## Summary

You've set up the Snowflake infrastructure for Airbyte:

- [x] Created the AIRBYTE database with reader/writer database roles
- [x] Created SVC_AIRBYTE service account with dedicated role
- [x] Granted AIRBYTE_DB_WRITER to SVC_AIRBYTE for loading
- [x] Granted AIRBYTE_DB_READER to ANALYTICS_SOURCES_READER for analyst access
- [x] Added HUBSPOT schema
- [x] Generated and stored key-pair credentials

## What's Next

With Snowflake ready, configure the HubSpot source and Snowflake destination in Airbyte.

Continue to [HubSpot Connection](6-hubspot-connection.md) →
