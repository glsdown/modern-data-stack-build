# Functional Roles

On this page, you will:

- [x] Build the `snowflake_role` module
- [x] Create functional roles for different job functions
- [x] Wire up the role hierarchy with database and warehouse access

## What Are Functional Roles?

Rather than granting permissions directly to users, we create **functional roles** that represent job functions in your organisation. Users are then assigned the appropriate functional role as their default.

This approach provides:

- **Clarity**: Role names describe what the user does, not who they are
- **Consistency**: Everyone with the same job function has the same access
- **Maintainability**: Change permissions in one place, affects all users with that role
- **Auditability**: Easy to see what access a job function has

## Role Hierarchy

Here's how our functional roles relate to each other and to database roles:

```
SYSADMIN (built-in)
├── ANALYTICS_DEVELOPER
│   ├── ANALYTICS_SOURCES_READER
│   │   └── <LOADER>_DB_READER (for each data source)
│   ├── ANALYTICS_DB_READER
│   └── ANALYTICS_DEV_DB_WRITER
│
├── ANALYTICS_TRANSFORMER
│   ├── ANALYTICS_SOURCES_READER
│   │   └── <LOADER>_DB_READER (for each data source)
│   └── ANALYTICS_DB_WRITER
│
└── ANALYTICS_REPORTER
    └── ANALYTICS_DB_READER (reporting schemas only)
```

| Role | Purpose | Access |
|------|---------|--------|
| `ANALYTICS_DEVELOPER` | Data team members | Read sources, read prod, write dev |
| `ANALYTICS_TRANSFORMER` | dbt service account | Read sources, write prod |
| `ANALYTICS_REPORTER` | BI tools | Read reporting schemas only |
| `ANALYTICS_SOURCES_READER` | Shared source access | Read all loader databases |

## The Role Module

Create the account role module:

```sh
mkdir -p modules/snowflake_role
```

### main.tf

Create `modules/snowflake_role/main.tf`:

```hcl
terraform {
  required_providers {
    snowflake = {
      source                = "Snowflake-Labs/snowflake"
      version               = "~> 0.99"
      configuration_aliases = [snowflake.user_admin]
    }
  }
}

# -----------------------------------------------------------------------------
# Account Role
# -----------------------------------------------------------------------------
resource "snowflake_account_role" "this" {
  provider = snowflake.user_admin
  name     = upper(var.role_name)
  comment  = var.role_comment
}

# Grant the role to SYSADMIN so admins can manage it
resource "snowflake_grant_account_role" "to_sysadmin" {
  provider         = snowflake.user_admin
  role_name        = snowflake_account_role.this.name
  parent_role_name = "SYSADMIN"
}
```

### variables.tf

Create `modules/snowflake_role/variables.tf`:

```hcl
variable "role_name" {
  description = "The name of the role (will be uppercased)"
  type        = string
}

variable "role_comment" {
  description = "Description of the role's purpose"
  type        = string
}
```

### outputs.tf

Create `modules/snowflake_role/outputs.tf`:

```hcl
output "role_name" {
  description = "The name of the role"
  value       = snowflake_account_role.this.name
}
```

## Create Functional Roles

Create `functional_roles.tf` in your root Snowflake directory:

```hcl
# =============================================================================
# Functional Roles
# =============================================================================
# Account roles that represent job functions. Users are assigned these roles
# as their default_role.

# -----------------------------------------------------------------------------
# Analytics Sources Reader
# -----------------------------------------------------------------------------
# Shared role for reading all data sources. Granted to developers and transformers.
module "role_analytics_sources_reader" {
  source = "./modules/snowflake_role"

  providers = {
    snowflake.user_admin = snowflake.user_admin
  }

  role_name    = "ANALYTICS_SOURCES_READER"
  role_comment = "Read access to all analytics data sources."
}

# -----------------------------------------------------------------------------
# Analytics Developer
# -----------------------------------------------------------------------------
# For data team members developing models
module "role_analytics_developer" {
  source = "./modules/snowflake_role"

  providers = {
    snowflake.user_admin = snowflake.user_admin
  }

  role_name = "ANALYTICS_DEVELOPER"
  role_comment = join(" ", [
    "For data team members.",
    "Read access to sources and production.",
    "Write access to development database.",
  ])
}

# Grant ANALYTICS_SOURCES_READER to ANALYTICS_DEVELOPER
resource "snowflake_grant_account_role" "sources_reader_to_developer" {
  provider         = snowflake.security_admin
  role_name        = module.role_analytics_sources_reader.role_name
  parent_role_name = module.role_analytics_developer.role_name
}

# -----------------------------------------------------------------------------
# Analytics Transformer
# -----------------------------------------------------------------------------
# For dbt and transformation tools
module "role_analytics_transformer" {
  source = "./modules/snowflake_role"

  providers = {
    snowflake.user_admin = snowflake.user_admin
  }

  role_name    = "ANALYTICS_TRANSFORMER"
  role_comment = "For dbt. Read access to sources. Write access to production analytics."
}

# Grant ANALYTICS_SOURCES_READER to ANALYTICS_TRANSFORMER
resource "snowflake_grant_account_role" "sources_reader_to_transformer" {
  provider         = snowflake.security_admin
  role_name        = module.role_analytics_sources_reader.role_name
  parent_role_name = module.role_analytics_transformer.role_name
}

# -----------------------------------------------------------------------------
# Analytics Reporter
# -----------------------------------------------------------------------------
# For BI tools like Metabase
module "role_analytics_reporter" {
  source = "./modules/snowflake_role"

  providers = {
    snowflake.user_admin = snowflake.user_admin
  }

  role_name    = "ANALYTICS_REPORTER"
  role_comment = "For BI tools. Read access to reporting schemas only."
}
```

## Update Database Grants

Now update `databases.tf` to grant the database roles to functional roles:

```hcl
# =============================================================================
# Databases
# =============================================================================

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
    module.role_analytics_reporter.role_name,
  ]
  grant_writer_to_account_roles = [
    module.role_analytics_transformer.role_name,
  ]
}

# -----------------------------------------------------------------------------
# Analytics Dev (Development)
# -----------------------------------------------------------------------------
module "database_analytics_dev" {
  source = "./modules/snowflake_database"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  database_name    = "ANALYTICS_DEV"
  database_comment = "Development environment for analytics developers."

  grant_reader_to_account_roles = []
  grant_writer_to_account_roles = [
    module.role_analytics_developer.role_name,
  ]
}

# -----------------------------------------------------------------------------
# Admin
# -----------------------------------------------------------------------------
module "database_admin" {
  source = "./modules/snowflake_database"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  database_name    = "ADMIN"
  database_comment = "Administrative objects including masking policies and tags."

  grant_reader_to_account_roles = []
  grant_writer_to_account_roles = []
}
```

## Update Warehouse Grants

Update `warehouses.tf` to grant warehouse usage to functional roles:

```hcl
# =============================================================================
# Shared Warehouses
# =============================================================================

# -----------------------------------------------------------------------------
# Loading Warehouse
# -----------------------------------------------------------------------------
module "warehouse_loading" {
  source = "./modules/snowflake_warehouse"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  warehouse_name     = "LOADING"
  warehouse_comment  = "Data ingestion from external sources."
  warehouse_size     = "X-Small"
  warehouse_auto_suspend = 60

  warehouse_usage_roles = [
    # Add loader service accounts here as you create them
  ]
}

# -----------------------------------------------------------------------------
# Transforming Warehouse
# -----------------------------------------------------------------------------
module "warehouse_transforming" {
  source = "./modules/snowflake_warehouse"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  warehouse_name     = "TRANSFORMING"
  warehouse_comment  = "dbt transformations and data modelling."
  warehouse_size     = "Small"
  warehouse_auto_suspend = 60

  warehouse_usage_roles = [
    module.role_analytics_transformer.role_name,
  ]
}

# -----------------------------------------------------------------------------
# Reporting Warehouse
# -----------------------------------------------------------------------------
module "warehouse_reporting" {
  source = "./modules/snowflake_warehouse"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  warehouse_name     = "REPORTING"
  warehouse_comment  = "Business intelligence and reporting queries."
  warehouse_size     = "Small"
  warehouse_auto_suspend = 300

  warehouse_usage_roles = [
    module.role_analytics_reporter.role_name,
  ]
}

# -----------------------------------------------------------------------------
# Developer Warehouse
# -----------------------------------------------------------------------------
module "warehouse_developer" {
  source = "./modules/snowflake_warehouse"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  warehouse_name     = "DEVELOPER"
  warehouse_comment  = "Ad-hoc queries and development work."
  warehouse_size     = "X-Small"
  warehouse_auto_suspend = 120

  warehouse_usage_roles = [
    module.role_analytics_developer.role_name,
  ]
}
```

## Understanding the Access Matrix

Here's what each role can do after wiring everything up:

| Role | ANALYTICS | ANALYTICS_DEV | Sources | Warehouse |
|------|-----------|---------------|---------|-----------|
| ANALYTICS_DEVELOPER | Read | Read/Write | Read | DEVELOPER |
| ANALYTICS_TRANSFORMER | Read/Write | - | Read | TRANSFORMING |
| ANALYTICS_REPORTER | Read | - | - | REPORTING |

## Commit and Deploy

Commit your changes and push to trigger the CI/CD pipeline:

```sh
git add terraform/snowflake/
git commit -m "Add functional roles and wire up database/warehouse access"
git push
```

Review the plan carefully - you should see:

- 4 new roles being created
- Role grants between functional roles
- Database role grants to functional roles
- Warehouse grants to functional roles

## Verify in Snowflake

After the pipeline completes, verify the roles and their grants:

```sql
-- Check roles exist
SHOW ROLES LIKE 'ANALYTICS%';

-- Check role hierarchy
SHOW GRANTS TO ROLE ANALYTICS_DEVELOPER;
SHOW GRANTS TO ROLE ANALYTICS_TRANSFORMER;
SHOW GRANTS TO ROLE ANALYTICS_REPORTER;

-- Verify database access
SHOW GRANTS TO ROLE ANALYTICS_DEVELOPER;
```

## Summary

You've created the access control structure for your data platform:

- [x] Built the `snowflake_role` module
- [x] Created ANALYTICS_DEVELOPER, ANALYTICS_TRANSFORMER, ANALYTICS_REPORTER, and ANALYTICS_SOURCES_READER roles
- [x] Wired up database and warehouse access for each role

## What's Next

With roles in place, you can now create users and assign them appropriate functional roles. In the next section, you'll build the user module and create admin users, developers, and service accounts.

Continue to [Users](5-users.md) →
