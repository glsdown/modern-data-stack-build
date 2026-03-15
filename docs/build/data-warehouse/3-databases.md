# Databases

On this page, you will:

- [x] Build the `snowflake_database_role` module
- [x] Build the `snowflake_database` module with automatic reader/writer roles
- [x] Create ANALYTICS, ANALYTICS_DEV, and ADMIN databases

## Why Database Roles?

Snowflake supports two types of roles:

- **Account roles**: Traditional roles that exist at the account level
- **Database roles**: Roles scoped to a specific database (introduced in Snowflake)

Database roles simplify access management by keeping permissions close to the data they protect. Our pattern creates two database roles for each database:

- `<DATABASE>_DB_READER`: Read-only access to all schemas and objects
- `<DATABASE>_DB_WRITER`: Full access to create, modify, and delete

These database roles are then granted to account-level functional roles, creating a clean hierarchy:

```
ANALYTICS_DEVELOPER (account role)
  └── ANALYTICS_DEV_DB_WRITER (database role)
  └── ANALYTICS_DB_READER (database role)
  └── ANALYTICS_SOURCES_READER (account role)
        └── AIRBYTE_DB_READER (database role)
        └── DLT_DB_READER (database role)
```

## The Database Role Module

First, create the database role module that the database module will use.

```sh
mkdir -p modules/snowflake_database_role
```

### main.tf

Create `modules/snowflake_database_role/main.tf`:

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
# Database Role
# -----------------------------------------------------------------------------
resource "snowflake_database_role" "this" {
  provider = snowflake.sys_admin
  database = upper(var.role_database_name)
  name     = upper(var.role_name)
  comment  = var.role_comment
}

# Grant the role to SYSADMIN so it can manage objects
resource "snowflake_grant_database_role" "to_sysadmin" {
  provider           = snowflake.security_admin
  database_role_name = "\"${snowflake_database_role.this.database}\".\"${snowflake_database_role.this.name}\""
  parent_role_name   = "SYSADMIN"
}
```

### variables.tf

Create `modules/snowflake_database_role/variables.tf`:

```hcl
variable "role_name" {
  description = "The name of the database role (will be uppercased)"
  type        = string
}

variable "role_database_name" {
  description = "The database this role belongs to"
  type        = string
}

variable "role_comment" {
  description = "Description of the role's purpose"
  type        = string
}
```

### outputs.tf

Create `modules/snowflake_database_role/outputs.tf`:

```hcl
output "role_name" {
  description = "The name of the database role"
  value       = snowflake_database_role.this.name
}

output "role_database_name" {
  description = "The database this role belongs to"
  value       = snowflake_database_role.this.database
}

output "role_fq_name" {
  description = "The fully-qualified name (DATABASE.ROLE) for use in grants"
  value       = "\"${snowflake_database_role.this.database}\".\"${snowflake_database_role.this.name}\""
}
```

## The Database Module

Now create the database module that automatically sets up reader and writer roles.

```sh
mkdir -p modules/snowflake_database
```

### main.tf

Create `modules/snowflake_database/main.tf`:

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
# Database
# -----------------------------------------------------------------------------
resource "snowflake_database" "this" {
  provider                    = snowflake.sys_admin
  name                        = upper(var.database_name)
  comment                     = var.database_comment
  data_retention_time_in_days = var.database_data_retention_time_in_days
}

# -----------------------------------------------------------------------------
# Database Roles
# -----------------------------------------------------------------------------
module "reader_role" {
  source = "../snowflake_database_role"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  role_name          = "${snowflake_database.this.name}_DB_READER"
  role_database_name = snowflake_database.this.name
  role_comment       = "Read-only access to ${snowflake_database.this.name} database."
}

module "writer_role" {
  source = "../snowflake_database_role"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  role_name          = "${snowflake_database.this.name}_DB_WRITER"
  role_database_name = snowflake_database.this.name
  role_comment       = "Full access to ${snowflake_database.this.name} database."
}

# -----------------------------------------------------------------------------
# Reader Role Grants
# -----------------------------------------------------------------------------
# Usage on all current and future schemas
resource "snowflake_grant_privileges_to_database_role" "reader_usage_schemas" {
  provider           = snowflake.security_admin
  privileges         = ["USAGE"]
  database_role_name = module.reader_role.role_fq_name

  on_schema {
    all_schemas_in_database = snowflake_database.this.name
  }
}

resource "snowflake_grant_privileges_to_database_role" "reader_usage_future_schemas" {
  provider           = snowflake.security_admin
  privileges         = ["USAGE"]
  database_role_name = module.reader_role.role_fq_name

  on_schema {
    future_schemas_in_database = snowflake_database.this.name
  }
}

# Select on all current and future objects
resource "snowflake_grant_privileges_to_database_role" "reader_select_objects" {
  provider           = snowflake.security_admin
  for_each           = toset(["TABLES", "VIEWS", "MATERIALIZED VIEWS"])
  privileges         = ["SELECT"]
  database_role_name = module.reader_role.role_fq_name

  on_schema_object {
    all {
      object_type_plural = each.value
      in_database        = snowflake_database.this.name
    }
  }
}

resource "snowflake_grant_privileges_to_database_role" "reader_select_future_objects" {
  provider           = snowflake.security_admin
  for_each           = toset(["TABLES", "VIEWS", "MATERIALIZED VIEWS"])
  privileges         = ["SELECT"]
  database_role_name = module.reader_role.role_fq_name

  on_schema_object {
    future {
      object_type_plural = each.value
      in_database        = snowflake_database.this.name
    }
  }
}

# -----------------------------------------------------------------------------
# Writer Role Grants
# -----------------------------------------------------------------------------
# Create schema privilege on database
resource "snowflake_grant_privileges_to_database_role" "writer_create_schema" {
  provider           = snowflake.security_admin
  privileges         = ["CREATE SCHEMA"]
  database_role_name = module.writer_role.role_fq_name

  on_database = snowflake_database.this.name
}

# All privileges on all current and future schemas
resource "snowflake_grant_privileges_to_database_role" "writer_all_schemas" {
  provider           = snowflake.security_admin
  all_privileges     = true
  database_role_name = module.writer_role.role_fq_name

  on_schema {
    all_schemas_in_database = snowflake_database.this.name
  }
}

resource "snowflake_grant_privileges_to_database_role" "writer_all_future_schemas" {
  provider           = snowflake.security_admin
  all_privileges     = true
  database_role_name = module.writer_role.role_fq_name

  on_schema {
    future_schemas_in_database = snowflake_database.this.name
  }
}

# All privileges on all current and future objects
resource "snowflake_grant_privileges_to_database_role" "writer_all_objects" {
  provider           = snowflake.security_admin
  for_each           = toset(["TABLES", "VIEWS", "MATERIALIZED VIEWS", "STAGES", "FILE FORMATS", "STREAMS"])
  all_privileges     = true
  database_role_name = module.writer_role.role_fq_name

  on_schema_object {
    all {
      object_type_plural = each.value
      in_database        = snowflake_database.this.name
    }
  }
}

resource "snowflake_grant_privileges_to_database_role" "writer_all_future_objects" {
  provider           = snowflake.security_admin
  for_each           = toset(["TABLES", "VIEWS", "MATERIALIZED VIEWS", "STAGES", "FILE FORMATS", "STREAMS"])
  all_privileges     = true
  database_role_name = module.writer_role.role_fq_name

  on_schema_object {
    future {
      object_type_plural = each.value
      in_database        = snowflake_database.this.name
    }
  }
}

# -----------------------------------------------------------------------------
# Grant Database Roles to Account Roles
# -----------------------------------------------------------------------------
resource "snowflake_grant_database_role" "reader_to_account_roles" {
  provider = snowflake.security_admin
  for_each = toset(var.grant_reader_to_account_roles)

  database_role_name = module.reader_role.role_fq_name
  parent_role_name   = upper(each.value)
}

resource "snowflake_grant_database_role" "writer_to_account_roles" {
  provider = snowflake.security_admin
  for_each = toset(var.grant_writer_to_account_roles)

  database_role_name = module.writer_role.role_fq_name
  parent_role_name   = upper(each.value)
}
```

### variables.tf

Create `modules/snowflake_database/variables.tf`:

```hcl
variable "database_name" {
  description = "The name of the database (will be uppercased)"
  type        = string
}

variable "database_comment" {
  description = "Description of the database's purpose"
  type        = string
}

variable "database_data_retention_time_in_days" {
  description = "Days to retain data for Time Travel (1-90, Enterprise edition for >1)"
  type        = number
  default     = 1

  validation {
    condition     = var.database_data_retention_time_in_days >= 0 && var.database_data_retention_time_in_days <= 90
    error_message = "Data retention must be between 0 and 90 days."
  }
}

variable "grant_reader_to_account_roles" {
  description = "Account roles to grant the DB_READER database role to"
  type        = list(string)
  default     = []
}

variable "grant_writer_to_account_roles" {
  description = "Account roles to grant the DB_WRITER database role to"
  type        = list(string)
  default     = []
}
```

### outputs.tf

Create `modules/snowflake_database/outputs.tf`:

```hcl
output "database_name" {
  description = "The name of the database"
  value       = snowflake_database.this.name
}

output "reader_role_name" {
  description = "The name of the DB_READER database role"
  value       = module.reader_role.role_name
}

output "reader_role_fq_name" {
  description = "The fully-qualified name of the DB_READER role"
  value       = module.reader_role.role_fq_name
}

output "writer_role_name" {
  description = "The name of the DB_WRITER database role"
  value       = module.writer_role.role_name
}

output "writer_role_fq_name" {
  description = "The fully-qualified name of the DB_WRITER role"
  value       = module.writer_role.role_fq_name
}
```

## Create Databases

Now use the module to create your databases. Create `databases.tf` in your root Snowflake directory:

```hcl
# =============================================================================
# Databases
# =============================================================================
# Each database automatically gets DB_READER and DB_WRITER database roles.

# -----------------------------------------------------------------------------
# Analytics (Production)
# -----------------------------------------------------------------------------
# Production analytics models created by dbt
module "database_analytics" {
  source = "./modules/snowflake_database"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  database_name    = "ANALYTICS"
  database_comment = "Production analytics models created by dbt."

  grant_reader_to_account_roles = [
    # Add roles that need read access
    # "ANALYTICS_DEVELOPER",
    # "ANALYTICS_REPORTER",
  ]
  grant_writer_to_account_roles = [
    # Add roles that need write access
    # "ANALYTICS_TRANSFORMER",
  ]
}

# -----------------------------------------------------------------------------
# Analytics Dev (Development)
# -----------------------------------------------------------------------------
# Development environment for analytics developers
module "database_analytics_dev" {
  source = "./modules/snowflake_database"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  database_name    = "ANALYTICS_DEV"
  database_comment = "Development environment for analytics developers."

  grant_reader_to_account_roles = [
    # Developers can read their own dev work
  ]
  grant_writer_to_account_roles = [
    # Developers can write to dev
    # "ANALYTICS_DEVELOPER",
  ]
}

# -----------------------------------------------------------------------------
# Admin
# -----------------------------------------------------------------------------
# Administrative objects like masking policies and tags
module "database_admin" {
  source = "./modules/snowflake_database"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  database_name    = "ADMIN"
  database_comment = "Administrative objects including masking policies and tags."

  # Admin database is managed by SYSADMIN only
  grant_reader_to_account_roles = []
  grant_writer_to_account_roles = []
}
```

!!! info "Roles Coming Later"
    The role grants are commented out because we haven't created the functional roles yet. You'll uncomment these in the [Functional Roles](4-functional-roles.md) section.

## Understanding the Grant Hierarchy

When you create a database with this module, here's what gets created:

```
DATABASE: ANALYTICS
├── Database Role: ANALYTICS_DB_READER
│   ├── USAGE on all schemas (current and future)
│   └── SELECT on all tables, views, materialized views (current and future)
│
└── Database Role: ANALYTICS_DB_WRITER
    ├── CREATE SCHEMA on database
    ├── ALL PRIVILEGES on all schemas (current and future)
    └── ALL PRIVILEGES on all objects (current and future)
```

Both roles are automatically granted to SYSADMIN, ensuring administrators can always manage objects.

## Data Retention and Time Travel

The `database_data_retention_time_in_days` setting controls Snowflake's Time Travel feature:

```hcl
database_data_retention_time_in_days = 1  # Default
```

| Days | Use Case | Edition |
|------|----------|---------|
| 0 | No Time Travel (not recommended) | All |
| 1 | Default, minimal storage overhead | All |
| 7-90 | Extended recovery window | Enterprise+ |

!!! tip "Start with 1 Day"
    Time Travel storage adds cost. Start with 1 day and increase only if you need longer recovery windows for critical data.

## Commit and Deploy

Commit your changes and push to trigger the CI/CD pipeline:

```sh
git add terraform/snowflake/
git commit -m "Add snowflake_database and snowflake_database_role modules"
git push
```

Review the plan in your pull request. You should see the three databases being created along with their database roles and grants.

## Verify in Snowflake

After the pipeline completes, verify the databases and roles:

```sql
-- Check databases
SHOW DATABASES;

-- Check database roles for ANALYTICS
USE DATABASE ANALYTICS;
SHOW DATABASE ROLES;

-- Verify role grants
SHOW GRANTS TO DATABASE ROLE ANALYTICS.ANALYTICS_DB_READER;
SHOW GRANTS TO DATABASE ROLE ANALYTICS.ANALYTICS_DB_WRITER;
```

## Summary

You've created the storage infrastructure for your data platform:

- [x] Built the `snowflake_database_role` module for scoped access control
- [x] Built the `snowflake_database` module with automatic reader/writer roles
- [x] Created ANALYTICS, ANALYTICS_DEV, and ADMIN databases

## What's Next

With warehouses (compute) and databases (storage) in place, you need roles to control who can access what. In the next section, you'll create functional roles that represent job functions in your organisation.

Continue to [Functional Roles](4-functional-roles.md) →
