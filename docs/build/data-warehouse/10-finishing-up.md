# Finishing Up

On this page, you will:

- [x] Verify your complete Snowflake infrastructure
- [x] Review the final project structure
- [x] Understand what you've built
- [x] Plan next steps

## Verification Checklist

Run through these checks to verify your Snowflake infrastructure is correctly configured.

### Databases

```sql
-- List all databases
SHOW DATABASES;

-- Expected: ANALYTICS, ANALYTICS_DEV, ADMIN (plus system databases)
```

### Warehouses

```sql
-- List warehouses
SHOW WAREHOUSES;

-- Expected: LOADING, TRANSFORMING, REPORTING, DEVELOPER
-- Check they auto-suspend correctly
SELECT name, auto_suspend, size
FROM TABLE(INFORMATION_SCHEMA.WAREHOUSES());
```

### Roles

```sql
-- List custom roles
SHOW ROLES LIKE 'ANALYTICS%';

-- Check role hierarchy
SHOW GRANTS TO ROLE ANALYTICS_DEVELOPER;
SHOW GRANTS TO ROLE ANALYTICS_TRANSFORMER;
SHOW GRANTS TO ROLE ANALYTICS_REPORTER;
SHOW GRANTS TO ROLE ANALYTICS_SOURCES_READER;
```

### Users

```sql
-- List users
SHOW USERS;

-- Check your admin and developer users
SHOW USERS LIKE '%ADMIN%';

-- Verify SVC_TERRAFORM service account
DESCRIBE USER SVC_TERRAFORM;

-- Verify default roles and warehouses
SELECT name, default_role, default_warehouse
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE deleted_on IS NULL;
```

### Schemas

```sql
-- Check ANALYTICS database schemas
SHOW SCHEMAS IN DATABASE ANALYTICS;

-- Verify REPORTING schema exists
DESCRIBE SCHEMA ANALYTICS.REPORTING;
```

### Network Policies

```sql
-- List network policies
SHOW NETWORK POLICIES;

-- Check which users have policies
SELECT name, network_policy
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE network_policy IS NOT NULL;
```

### Storage Integrations

```sql
-- List integrations
SHOW INTEGRATIONS;

-- Test storage access
DESCRIBE INTEGRATION DATA_LAKE_INTEGRATION;
```

## Final Project Structure

Your Terraform project should now look like this:

```
terraform/snowflake/
├── modules/
│   ├── snowflake_database/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── snowflake_database_role/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── snowflake_role/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── snowflake_schema/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── snowflake_storage_integration/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── snowflake_user/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── snowflake_warehouse/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
├── databases.tf
├── functional_roles.tf
├── integrations.tf
├── network_policies.tf
├── network_policies.auto.tfvars
├── providers.tf
├── schemas.tf
├── users.tf
├── users.auto.tfvars
├── variables.tf
└── warehouses.tf
```

## What You've Built

Here's a summary of your Snowflake infrastructure:

### Access Control Model

```
ACCOUNTADMIN
└── SYSADMIN
    ├── ANALYTICS_DEVELOPER
    │   ├── ANALYTICS_SOURCES_READER
    │   │   └── (Loader DB roles)
    │   ├── ANALYTICS_DB_READER
    │   └── ANALYTICS_DEV_DB_WRITER
    │
    ├── ANALYTICS_TRANSFORMER
    │   ├── ANALYTICS_SOURCES_READER
    │   └── ANALYTICS_DB_WRITER
    │
    └── ANALYTICS_REPORTER
        └── ANALYTICS_REPORTING_SCHEMA_READER
```

### Databases and Access

| Database | DEVELOPER | TRANSFORMER | REPORTER |
|----------|-----------|-------------|----------|
| ANALYTICS | Read | Read/Write | Read (REPORTING schema only) |
| ANALYTICS_DEV | Read/Write | - | - |
| ADMIN | - | - | - |

### Warehouses

| Warehouse | Purpose | Roles with Access |
|-----------|---------|-------------------|
| LOADING | Data ingestion | Loader service accounts |
| TRANSFORMING | dbt jobs | ANALYTICS_TRANSFORMER |
| REPORTING | BI queries | ANALYTICS_REPORTER |
| DEVELOPER | Ad-hoc queries | ANALYTICS_DEVELOPER |

### Users

| User Type | Default Role | Authentication |
|-----------|--------------|----------------|
| Admins | SYSADMIN | SSO (or password during initial setup) |
| Developers | ANALYTICS_DEVELOPER | SSO |
| SVC_TERRAFORM | ACCOUNTADMIN | Key pair |

!!! note "Service Accounts Added Later"
    Additional service accounts (SVC_DBT, SVC_METABASE, etc.) are created when you add those tools to your stack. Each follows the same pattern established for SVC_TERRAFORM.

## Common Next Steps

### Add Data Sources

When you add a new data source (e.g., Stripe, Salesforce via Airbyte or dlt):

1. Create a loader database:
   ```hcl
   module "database_stripe" {
     source = "./modules/snowflake_database"

     providers = {
       snowflake.sys_admin      = snowflake.sys_admin
       snowflake.security_admin = snowflake.security_admin
     }

     database_name    = "STRIPE"
     database_comment = "Raw data from Stripe loaded via Airbyte."

     grant_reader_to_account_roles = [
       module.role_analytics_sources_reader.role_name,
     ]
   }
   ```

2. Create a service account for the loader tool (when setting up Airbyte, dlt, etc.)
3. Configure the loader tool with the service account credentials

### Add More Schemas

For additional Terraform-managed schemas:

```hcl
module "schema_analytics_sensitive" {
  source = "./modules/snowflake_schema"
  # ...
  schema_name     = "SENSITIVE"
  schema_database = module.database_analytics.database_name
  grant_reader_to_account_roles = [
    # Only specific roles
  ]
}
```

### Add Resource Monitors

To add cost controls with resource monitors:

```hcl
resource "snowflake_resource_monitor" "monthly_limit" {
  provider      = snowflake.account_admin
  name          = "MONTHLY_CREDIT_LIMIT"
  credit_quota  = 100
  frequency     = "MONTHLY"
  start_timestamp = "2024-01-01 00:00"

  notify_triggers = [75, 90, 100]
  suspend_trigger = 100
  suspend_immediate_trigger = 110
}
```

### Add Tags for Cost Allocation

Snowflake tags help with governance and cost tracking. Create tags in the ADMIN database:

```hcl
# Create tags in ADMIN database
resource "snowflake_tag" "cost_center" {
  provider = snowflake.account_admin
  name     = "COST_CENTER"
  database = module.database_admin.database_name
  schema   = "PUBLIC"
  comment  = "Cost center for billing allocation."

  allowed_values = ["engineering", "analytics", "marketing", "operations"]
}

resource "snowflake_tag" "environment" {
  provider = snowflake.account_admin
  name     = "ENVIRONMENT"
  database = module.database_admin.database_name
  schema   = "PUBLIC"
  comment  = "Environment classification."

  allowed_values = ["dev", "staging", "prod"]
}

resource "snowflake_tag" "data_classification" {
  provider = snowflake.account_admin
  name     = "DATA_CLASSIFICATION"
  database = module.database_admin.database_name
  schema   = "PUBLIC"
  comment  = "Data sensitivity classification."

  allowed_values = ["public", "internal", "confidential", "restricted"]
}
```

Apply tags to resources:

```hcl
# Tag a warehouse
resource "snowflake_tag_association" "warehouse_transforming" {
  provider  = snowflake.account_admin
  tag_id    = snowflake_tag.cost_center.fully_qualified_name
  tag_value = "analytics"

  object_identifiers {
    name = module.warehouse_transforming.warehouse_name
  }
  object_type = "WAREHOUSE"
}

# Tag a database
resource "snowflake_tag_association" "database_analytics" {
  provider  = snowflake.account_admin
  tag_id    = snowflake_tag.environment.fully_qualified_name
  tag_value = "prod"

  object_identifiers {
    name = module.database_analytics.database_name
  }
  object_type = "DATABASE"
}
```

!!! tip "Tag-Based Cost Allocation"
    Use the `COST_CENTER` tag with Snowflake's cost attribution features to track spending by team or project. Query `SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES` to see tag assignments.

## Troubleshooting

### "Insufficient privileges" errors

Check the role hierarchy:

```sql
SHOW GRANTS TO ROLE <role_name>;
SHOW GRANTS OF ROLE <role_name>;
```

Ensure the role has been granted to the user and the necessary database/schema/object grants exist.

### Users can't see schemas

Verify database role grants:

```sql
SHOW GRANTS TO DATABASE ROLE ANALYTICS.ANALYTICS_DB_READER;
```

And that the database role is granted to the account role:

```sql
SHOW GRANTS OF DATABASE ROLE ANALYTICS.ANALYTICS_DB_READER;
```

### Terraform state drift

If manual changes were made in Snowflake:

```sh
terraform plan  # Review drift
terraform apply # Reconcile to desired state
```

Or import manual resources:

```sh
terraform import snowflake_warehouse.example WAREHOUSE_NAME
```

## Summary

You've completed the Snowflake data warehouse setup:

- [x] Verified all infrastructure components
- [x] Reviewed the final project structure
- [x] Understood the access control model
- [x] Identified common next steps

## What's Next

With your data warehouse infrastructure in place, you're ready to:

1. **[Set up data loading](../data-loaders/index.md)** - Configure Fivetran or Airbyte to load data
2. **[Build transformations](../data-transformations/index.md)** - Set up dbt for data modelling
3. **[Connect BI tools](../data-reporting/index.md)** - Configure Metabase or your preferred tool

Your Snowflake account is now production-ready with proper access controls, cost-efficient shared warehouses, and secure authentication.
