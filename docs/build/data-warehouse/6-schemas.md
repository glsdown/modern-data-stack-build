# Schemas

On this page, you will:

- [x] Build the `snowflake_schema` module with schema-level reader roles
- [x] Create the REPORTING schema for BI tool access
- [x] Update ANALYTICS_REPORTER to use schema-level access

## Why Manage Schemas in Terraform?

Most schemas in your analytics database are created dynamically by dbt - staging, intermediate, marts, and so on. These don't need Terraform management because:

- dbt creates them automatically when models run
- Database-level future grants provide the necessary permissions
- The transformer role has write access to the entire database

However, some schemas benefit from Terraform management:

- **Fine-grained access control**: Schema-level reader roles let you restrict access to specific schemas
- **Stable infrastructure**: Schemas that exist independently of dbt runs
- **BI tool access**: Reporting schemas where you want to limit what BI tools can see

## The Schema Module

The schema module creates a schema with an associated reader database role, enabling fine-grained access control.

Create the schema module:

```sh
mkdir -p modules/snowflake_schema
```

### main.tf

Create `modules/snowflake_schema/main.tf`:

```hcl
terraform {
  required_providers {
    snowflake = {
      source                = "Snowflake-Labs/snowflake"
      version               = "~> 0.99"
      configuration_aliases = [snowflake.sys_admin, snowflake.security_admin]
    }
  }
}

# -----------------------------------------------------------------------------
# Schema
# -----------------------------------------------------------------------------
resource "snowflake_schema" "this" {
  provider = snowflake.sys_admin
  database = upper(var.schema_database)
  name     = upper(var.schema_name)
  comment  = var.schema_comment

  is_managed         = var.schema_is_managed
  is_transient       = var.schema_is_transient
  data_retention_days = var.schema_data_retention_days
}

# -----------------------------------------------------------------------------
# Schema Reader Role
# -----------------------------------------------------------------------------
# Database role for read access to this specific schema
module "schema_reader_role" {
  source = "../snowflake_database_role"

  providers = {
    snowflake.security_admin = snowflake.security_admin
  }

  role_name          = "${var.schema_database}_${var.schema_name}_SCHEMA_READER"
  role_comment       = "Read access to ${var.schema_database}.${var.schema_name} schema."
  role_database_name = upper(var.schema_database)
}

# Grant USAGE on the schema to the reader role
resource "snowflake_grant_privileges_to_database_role" "schema_usage" {
  provider           = snowflake.security_admin
  database_role_name = "\"${upper(var.schema_database)}\".\"${module.schema_reader_role.role_name}\""
  privileges         = ["USAGE"]

  on_schema {
    schema_name = "\"${upper(var.schema_database)}\".\"${snowflake_schema.this.name}\""
  }
}

# Grant SELECT on all tables in the schema
resource "snowflake_grant_privileges_to_database_role" "table_select" {
  provider           = snowflake.security_admin
  database_role_name = "\"${upper(var.schema_database)}\".\"${module.schema_reader_role.role_name}\""
  privileges         = ["SELECT"]

  on_schema_object {
    all {
      object_type_plural = "TABLES"
      in_schema          = "\"${upper(var.schema_database)}\".\"${snowflake_schema.this.name}\""
    }
  }
}

# Grant SELECT on future tables in the schema
resource "snowflake_grant_privileges_to_database_role" "future_table_select" {
  provider           = snowflake.security_admin
  database_role_name = "\"${upper(var.schema_database)}\".\"${module.schema_reader_role.role_name}\""
  privileges         = ["SELECT"]

  on_schema_object {
    future {
      object_type_plural = "TABLES"
      in_schema          = "\"${upper(var.schema_database)}\".\"${snowflake_schema.this.name}\""
    }
  }
}

# Grant SELECT on all views in the schema
resource "snowflake_grant_privileges_to_database_role" "view_select" {
  provider           = snowflake.security_admin
  database_role_name = "\"${upper(var.schema_database)}\".\"${module.schema_reader_role.role_name}\""
  privileges         = ["SELECT"]

  on_schema_object {
    all {
      object_type_plural = "VIEWS"
      in_schema          = "\"${upper(var.schema_database)}\".\"${snowflake_schema.this.name}\""
    }
  }
}

# Grant SELECT on future views in the schema
resource "snowflake_grant_privileges_to_database_role" "future_view_select" {
  provider           = snowflake.security_admin
  database_role_name = "\"${upper(var.schema_database)}\".\"${module.schema_reader_role.role_name}\""
  privileges         = ["SELECT"]

  on_schema_object {
    future {
      object_type_plural = "VIEWS"
      in_schema          = "\"${upper(var.schema_database)}\".\"${snowflake_schema.this.name}\""
    }
  }
}

# -----------------------------------------------------------------------------
# Grant Schema Reader to Account Roles
# -----------------------------------------------------------------------------
resource "snowflake_grant_database_role" "grant_schema_reader_to_account_roles" {
  provider           = snowflake.security_admin
  for_each           = toset(var.grant_reader_to_account_roles)
  database_role_name = "\"${upper(var.schema_database)}\".\"${module.schema_reader_role.role_name}\""
  parent_role_name   = each.value
}
```

### variables.tf

Create `modules/snowflake_schema/variables.tf`:

```hcl
variable "schema_name" {
  description = "The name of the schema (will be uppercased)"
  type        = string
}

variable "schema_database" {
  description = "The database this schema belongs to"
  type        = string
}

variable "schema_comment" {
  description = "Description of the schema's purpose"
  type        = string
  default     = ""
}

variable "schema_is_managed" {
  description = "Whether the schema is managed (prevents DDL outside Terraform)"
  type        = bool
  default     = false
}

variable "schema_is_transient" {
  description = "Whether the schema is transient (no Fail-safe)"
  type        = bool
  default     = false
}

variable "schema_data_retention_days" {
  description = "Number of days for Time Travel (0-90)"
  type        = number
  default     = 1
}

variable "grant_reader_to_account_roles" {
  description = "Account roles to grant the schema reader database role to"
  type        = list(string)
  default     = []
}
```

### outputs.tf

Create `modules/snowflake_schema/outputs.tf`:

```hcl
output "schema_name" {
  description = "The fully qualified schema name"
  value       = snowflake_schema.this.name
}

output "schema_reader_role_name" {
  description = "The name of the schema reader database role"
  value       = module.schema_reader_role.role_name
}
```

## Create the Reporting Schema

The REPORTING schema is where dbt publishes models specifically for BI tools. By managing it in Terraform, we can grant ANALYTICS_REPORTER access to just this schema rather than the entire database.

Create `schemas.tf` in your root Snowflake directory:

```hcl
# =============================================================================
# Schemas
# =============================================================================
# Terraform-managed schemas for fine-grained access control.
# Note: Most schemas (staging, intermediate, marts) are created by dbt
# and don't need Terraform management.

# -----------------------------------------------------------------------------
# ANALYTICS.REPORTING
# -----------------------------------------------------------------------------
# The reporting schema contains models specifically for BI tools.
# ANALYTICS_REPORTER gets access to this schema only, not the entire database.
module "schema_analytics_reporting" {
  source = "./modules/snowflake_schema"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  schema_name     = "REPORTING"
  schema_database = module.database_analytics.database_name
  schema_comment  = "Published models for BI tools and reporting."

  grant_reader_to_account_roles = [
    module.role_analytics_reporter.role_name,
  ]
}
```

## Update Database Grants

Now update `databases.tf` to remove ANALYTICS_REPORTER from the database-level reader role. The reporter will get access via the schema-level role instead:

```hcl
# -----------------------------------------------------------------------------
# Analytics (Production)
# -----------------------------------------------------------------------------
module "database_analytics" {
  source = "./modules/snowflake_database"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  database_name    = "ANALYTICS"
  database_comment = "Production analytics models created by dbt."

  grant_reader_to_account_roles = [
    module.role_analytics_developer.role_name,
    # Note: ANALYTICS_REPORTER gets schema-level access via schemas.tf
  ]
  grant_writer_to_account_roles = [
    module.role_analytics_transformer.role_name,
  ]
}
```

## Understanding the Access Model

With schema-level access, here's what ANALYTICS_REPORTER can see:

| Database | Schema | Access |
|----------|--------|--------|
| ANALYTICS | REPORTING | ✅ Read (via schema reader role) |
| ANALYTICS | staging | ❌ No access |
| ANALYTICS | intermediate | ❌ No access |
| ANALYTICS | marts | ❌ No access |

This is more secure than database-level access because:

- BI tools only see curated, published models
- Internal staging and intermediate models are hidden
- You control exactly what gets exposed for reporting

## Commit and Deploy

Commit your changes and push to trigger the CI/CD pipeline:

```sh
git add terraform/snowflake/
git commit -m "Add reporting schema with schema-level reader role"
git push
```

Review the plan - you should see:

- The REPORTING schema being created
- A schema reader database role
- Grants from the schema reader role to ANALYTICS_REPORTER

## Verify in Snowflake

After the pipeline completes, verify the schema and access:

```sql
-- Check schema exists
SHOW SCHEMAS IN DATABASE ANALYTICS;

-- Check schema reader role
SHOW DATABASE ROLES IN DATABASE ANALYTICS;

-- Verify grants
SHOW GRANTS TO DATABASE ROLE ANALYTICS.ANALYTICS_REPORTING_SCHEMA_READER;

-- Test as reporter (should only see REPORTING schema)
USE ROLE ANALYTICS_REPORTER;
SHOW SCHEMAS IN DATABASE ANALYTICS;
```

## Summary

You've implemented fine-grained schema-level access control:

- [x] Built the `snowflake_schema` module with automatic reader roles
- [x] Created the REPORTING schema in the ANALYTICS database
- [x] Updated ANALYTICS_REPORTER to use schema-level access instead of database-level

## What's Next

With access control configured down to the schema level, you're ready to secure your Snowflake account with network policies. In the next section, you'll restrict access based on IP addresses and configure policies for different user types.

Continue to [Network Policies](7-network-policies.md) →
