# Users

On this page, you will:

- [x] Build the `snowflake_user` module
- [x] Create service accounts for dbt and BI tools
- [x] Create developer and admin users

## User Categories

Our Snowflake setup has several categories of users:

| Category | Example | Default Role | Default Warehouse |
|----------|---------|--------------|-------------------|
| **Admin** | `JBLOGGS_ADMIN` | SYSADMIN | DEVELOPER |
| **Developer** | `JBLOGGS` | ANALYTICS_DEVELOPER | DEVELOPER |
| **Transformer** | `SVC_DBT` | ANALYTICS_TRANSFORMER | TRANSFORMING |
| **Reporter** | `SVC_METABASE` | ANALYTICS_REPORTER | REPORTING |
| **Loader** | `SVC_AIRBYTE` | Dedicated role | LOADING |

Users share the workload-specific warehouses we created earlier. This approach:

- **Reduces cost**: Fewer warehouse resumes (60-second minimum billing each)
- **Simplifies management**: Four warehouses instead of dozens
- **Tracks costs by workload**: See spend by loading, transforming, reporting, development

## The User Module

The user module creates a user with role grants and a default warehouse.

```sh
mkdir -p modules/snowflake_user
```

### main.tf

Create `modules/snowflake_user/main.tf`:

```hcl
terraform {
  required_providers {
    snowflake = {
      source                = "Snowflake-Labs/snowflake"
      version               = "~> 0.99"
      configuration_aliases = [snowflake.security_admin, snowflake.user_admin]
    }
  }
}

locals {
  # Determine the default role
  default_role = var.user_create_dedicated_role ? module.dedicated_role[0].role_name : upper(var.user_default_role)

  # Combine default role with additional roles
  all_roles = concat([local.default_role], [for r in var.user_additional_roles : upper(r)])
}

# -----------------------------------------------------------------------------
# User
# -----------------------------------------------------------------------------
resource "snowflake_user" "this" {
  provider = snowflake.user_admin

  name              = upper(var.user_name)
  login_name        = upper(var.user_name)
  display_name      = var.user_display_name
  comment           = var.user_comment
  email             = var.user_email
  first_name        = var.user_first_name
  last_name         = var.user_last_name
  default_role      = local.default_role
  default_warehouse = upper(var.user_default_warehouse)

  # Validation for human users
  lifecycle {
    precondition {
      condition     = var.user_is_service_account || var.user_email != null
      error_message = "Email is required for human users."
    }
  }
}

# -----------------------------------------------------------------------------
# Dedicated Role (Optional)
# -----------------------------------------------------------------------------
# Useful for data loaders that need their own role for database writes
module "dedicated_role" {
  source = "../snowflake_role"
  count  = var.user_create_dedicated_role ? 1 : 0

  providers = {
    snowflake.user_admin = snowflake.user_admin
  }

  role_name    = var.user_name
  role_comment = "Dedicated role for ${var.user_name}."
}

# -----------------------------------------------------------------------------
# Role Grants
# -----------------------------------------------------------------------------
resource "snowflake_grant_account_role" "roles_to_user" {
  provider  = snowflake.security_admin
  for_each  = toset(local.all_roles)
  role_name = each.value
  user_name = snowflake_user.this.name
}
```

### variables.tf

Create `modules/snowflake_user/variables.tf`:

```hcl
variable "user_name" {
  description = "The username (will be uppercased)"
  type        = string
}

variable "user_comment" {
  description = "Description of the user"
  type        = string
}

variable "user_display_name" {
  description = "Display name shown in the UI"
  type        = string
  default     = ""
}

variable "user_email" {
  description = "Email address (required for human users)"
  type        = string
  default     = null

  validation {
    condition     = var.user_email == null || can(regex("^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$", var.user_email))
    error_message = "Invalid email address format."
  }
}

variable "user_first_name" {
  description = "First name"
  type        = string
  default     = ""
}

variable "user_last_name" {
  description = "Last name"
  type        = string
  default     = ""
}

variable "user_default_role" {
  description = "Default role for the user (ignored if user_create_dedicated_role is true)"
  type        = string
  default     = "PUBLIC"

  validation {
    condition     = upper(var.user_default_role) != "ACCOUNTADMIN"
    error_message = "Default role cannot be ACCOUNTADMIN. Use SYSADMIN as the highest default."
  }
}

variable "user_default_warehouse" {
  description = "Default warehouse for the user"
  type        = string
}

variable "user_additional_roles" {
  description = "Additional roles to grant to the user"
  type        = list(string)
  default     = []
}

variable "user_create_dedicated_role" {
  description = "Create a dedicated role for this user (useful for data loaders)"
  type        = bool
  default     = false
}

variable "user_is_service_account" {
  description = "Is this a service account? Relaxes email validation"
  type        = bool
  default     = false
}
```

### outputs.tf

Create `modules/snowflake_user/outputs.tf`:

```hcl
output "user_name" {
  description = "The username"
  value       = snowflake_user.this.name
}

output "user_default_role" {
  description = "The user's default role"
  value       = local.default_role
}

output "user_default_warehouse" {
  description = "The user's default warehouse"
  value       = snowflake_user.this.default_warehouse
}
```

## Create Service Accounts

Create `users.tf` in your root Snowflake directory:

```hcl
# =============================================================================
# Service Accounts
# =============================================================================

# -----------------------------------------------------------------------------
# dbt Transformer
# -----------------------------------------------------------------------------
module "user_svc_dbt" {
  source = "./modules/snowflake_user"

  providers = {
    snowflake.security_admin = snowflake.security_admin
    snowflake.user_admin     = snowflake.user_admin
  }

  user_name               = "SVC_DBT"
  user_comment            = "Service account for dbt transformations."
  user_display_name       = "dbt Service Account"
  user_is_service_account = true

  user_default_warehouse = module.warehouse_transforming.warehouse_name
  user_default_role      = module.role_analytics_transformer.role_name
}

# -----------------------------------------------------------------------------
# Metabase Reporter
# -----------------------------------------------------------------------------
module "user_svc_metabase" {
  source = "./modules/snowflake_user"

  providers = {
    snowflake.security_admin = snowflake.security_admin
    snowflake.user_admin     = snowflake.user_admin
  }

  user_name               = "SVC_METABASE"
  user_comment            = "Service account for Metabase BI tool."
  user_display_name       = "Metabase Service Account"
  user_is_service_account = true

  user_default_warehouse = module.warehouse_reporting.warehouse_name
  user_default_role      = module.role_analytics_reporter.role_name
}
```

## Create Human Users

For human users, use a variable-driven approach. Add to `variables.tf`:

```hcl
variable "developer_users" {
  description = "Map of developer users to create"
  type = map(object({
    display_name = string
    first_name   = string
    last_name    = string
    email        = string
  }))
  default = {}
}

variable "admin_users" {
  description = "Map of admin users (creates both _ADMIN and regular accounts)"
  type = map(object({
    display_name = string
    first_name   = string
    last_name    = string
    email        = string
  }))
  default = {}
}
```

Add to `users.tf`:

```hcl
# =============================================================================
# Developer Users
# =============================================================================
module "user_developers" {
  source   = "./modules/snowflake_user"
  for_each = var.developer_users

  providers = {
    snowflake.security_admin = snowflake.security_admin
    snowflake.user_admin     = snowflake.user_admin
  }

  user_name         = each.key
  user_comment      = "${each.value.display_name} - Analytics Developer"
  user_display_name = each.value.display_name
  user_first_name   = each.value.first_name
  user_last_name    = each.value.last_name
  user_email        = each.value.email

  user_default_warehouse = module.warehouse_developer.warehouse_name
  user_default_role      = module.role_analytics_developer.role_name
}

# =============================================================================
# Admin Users
# =============================================================================
# Each admin gets two accounts:
# - USERNAME_ADMIN: For administrative tasks (SYSADMIN default)
# - USERNAME: For daily development work (ANALYTICS_DEVELOPER default)

# Admin account
module "user_admins_admin" {
  source   = "./modules/snowflake_user"
  for_each = var.admin_users

  providers = {
    snowflake.security_admin = snowflake.security_admin
    snowflake.user_admin     = snowflake.user_admin
  }

  user_name         = "${each.key}_ADMIN"
  user_comment      = "${each.value.display_name} - Admin Account"
  user_display_name = "${each.value.display_name} (Admin)"
  user_first_name   = each.value.first_name
  user_last_name    = each.value.last_name
  user_email        = each.value.email

  user_default_warehouse = module.warehouse_developer.warehouse_name
  user_default_role      = "SYSADMIN"
  user_additional_roles  = ["ACCOUNTADMIN", "USERADMIN", "SECURITYADMIN"]
}

# Developer account
module "user_admins_developer" {
  source   = "./modules/snowflake_user"
  for_each = var.admin_users

  providers = {
    snowflake.security_admin = snowflake.security_admin
    snowflake.user_admin     = snowflake.user_admin
  }

  user_name         = each.key
  user_comment      = "${each.value.display_name} - Developer Account"
  user_display_name = each.value.display_name
  user_first_name   = each.value.first_name
  user_last_name    = each.value.last_name
  user_email        = each.value.email

  user_default_warehouse = module.warehouse_developer.warehouse_name
  user_default_role      = module.role_analytics_developer.role_name
}
```

Create `users.auto.tfvars`:

```hcl
# Non-admin developers - creates a single account
developer_users = {
  # "ASMITH" = {
  #   display_name = "Alice Smith"
  #   first_name   = "Alice"
  #   last_name    = "Smith"
  #   email        = "alice.smith@company.com"
  # }
}

# Admins - creates TWO accounts: USERNAME_ADMIN and USERNAME
# Don't add admins to developer_users - they already get a developer account
admin_users = {
  # "JBLOGGS" = {
  #   display_name = "Joe Bloggs"
  #   first_name   = "Joe"
  #   last_name    = "Bloggs"
  #   email        = "joe.bloggs@company.com"
  # }
}
```

!!! warning "Don't Duplicate Users"
    Each person should be in only ONE of these maps. Admins automatically get a developer account (`JBLOGGS`) alongside their admin account (`JBLOGGS_ADMIN`), so they don't need to be in `developer_users`.

## Why Dual Admin Accounts?

Admins get two accounts to enforce privilege separation:

| Account | Default Role | When to Use |
|---------|--------------|-------------|
| `JBLOGGS_ADMIN` | SYSADMIN | Creating objects, managing grants, Terraform changes |
| `JBLOGGS` | ANALYTICS_DEVELOPER | Daily queries, dbt development, exploring data |

This prevents accidentally running queries with elevated privileges and makes audit logs clearer about when admin access was actually needed.

## Authentication

Users created by Terraform don't have passwords set. Authentication options:

1. **SSO/SAML** (recommended for human users) - covered in [SSO Setup](9-sso-setup.md)
2. **Key-pair authentication** (recommended for service accounts)
3. **Password** (set manually in Snowflake UI if needed)

For service accounts, generate key pairs and store in 1Password or AWS Secrets Manager.

## Commit and Deploy

Commit your changes and push to trigger the CI/CD pipeline:

```sh
git add terraform/snowflake/
git commit -m "Add snowflake_user module and service accounts"
git push
```

## Verify in Snowflake

After the pipeline completes, verify the users:

```sql
-- Check users exist
SHOW USERS LIKE 'SVC_%';

-- Check user configuration
DESCRIBE USER SVC_DBT;

-- Check role grants
SHOW GRANTS TO USER SVC_DBT;
```

## Summary

You've created the user infrastructure for your data platform:

- [x] Built the `snowflake_user` module
- [x] Created `SVC_DBT` and `SVC_METABASE` service accounts
- [x] Set up patterns for developer and admin users with shared warehouses

## What's Next

With users in place, you can organise data within databases using schemas. In the next section, you'll create the schema module and set up the standard schema structure.

Continue to [Schemas](6-schemas.md) →
