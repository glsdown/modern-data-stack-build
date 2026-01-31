# Project Structure

On this page, you will:

- [x] Understand the module-based approach for Snowflake Terraform
- [x] Configure multiple Snowflake providers for different admin roles
- [x] Create the directory structure for modules
- [x] Learn when to use each provider

## Why Modules?

In the [Getting Started](../../getting-started/terraform-setup/snowflake/index.md) section, you created a simple Terraform configuration to manage Snowflake users. That flat structure works for basic setups, but as your data warehouse grows, you'll need a more scalable approach.

Terraform modules provide:

- **Reusability**: Define a resource pattern once, use it many times
- **Encapsulation**: Hide complexity behind a simple interface
- **Consistency**: Every database, warehouse, and user follows the same patterns
- **Maintainability**: Update the module, and all instances benefit

For example, the `snowflake_database` module we'll build creates not just the database, but also:

- A `DB_READER` database role with read-only access
- A `DB_WRITER` database role with full access
- All necessary grants for both roles
- Future grants for objects that don't exist yet

This means creating a new database is as simple as:

```hcl
module "database_analytics" {
  source = "./modules/snowflake_database"
  # ... providers ...

  database_name    = "ANALYTICS"
  database_comment = "Production analytics models."

  grant_database_reader_to_account_roles = ["ANALYTICS_DEVELOPER"]
  grant_database_writer_to_account_roles = ["ANALYTICS_TRANSFORMER"]
}
```

## Why Multiple Providers?

Snowflake uses role-based access control (RBAC) where different operations require different roles:

| Role | Purpose | Operations |
|------|---------|------------|
| **ACCOUNTADMIN** | Account-level settings | Network policies, storage integrations, account parameters |
| **SYSADMIN** | Object management | Create warehouses, databases, schemas |
| **SECURITYADMIN** | Access control | Grant privileges, manage roles |
| **USERADMIN** | User management | Create users, create roles |

Rather than running Terraform with `ACCOUNTADMIN` for everything (which would work but violates least privilege), we configure separate providers for each role. This ensures:

- Operations use the minimum required privileges
- Audit logs clearly show which role performed each action
- Modules can be explicit about which role they need

## Update Your Provider Configuration

Navigate to your Snowflake Terraform directory:

```sh
cd ~/projects/data/data-stack-infrastructure/terraform/snowflake
```

### Update providers.tf

Replace your current `providers.tf` with multiple provider configurations:

```hcl
# =============================================================================
# Snowflake Providers
# =============================================================================
# Each provider uses a different Snowflake role for least-privilege access.
# Modules specify which provider they need via configuration_aliases.

# Account Admin - for account-level settings (rarely needed)
provider "snowflake" {
  alias             = "account_admin"
  organization_name = var.snowflake_organization_name
  account_name      = var.snowflake_account_name
  user              = "SVC_TERRAFORM"
  authenticator     = "JWT"
  private_key       = var.SNOWFLAKE_PRIVATE_KEY
  role              = "ACCOUNTADMIN"
}

# System Admin - for creating warehouses, databases, schemas
provider "snowflake" {
  alias             = "sys_admin"
  organization_name = var.snowflake_organization_name
  account_name      = var.snowflake_account_name
  user              = "SVC_TERRAFORM"
  authenticator     = "JWT"
  private_key       = var.SNOWFLAKE_PRIVATE_KEY
  role              = "SYSADMIN"
}

# Security Admin - for managing grants and privileges
provider "snowflake" {
  alias             = "security_admin"
  organization_name = var.snowflake_organization_name
  account_name      = var.snowflake_account_name
  user              = "SVC_TERRAFORM"
  authenticator     = "JWT"
  private_key       = var.SNOWFLAKE_PRIVATE_KEY
  role              = "SECURITYADMIN"
}

# User Admin - for creating users and roles
provider "snowflake" {
  alias             = "user_admin"
  organization_name = var.snowflake_organization_name
  account_name      = var.snowflake_account_name
  user              = "SVC_TERRAFORM"
  authenticator     = "JWT"
  private_key       = var.SNOWFLAKE_PRIVATE_KEY
  role              = "USERADMIN"
}
```

!!! info "Same User, Different Roles"
    All providers use the same `SVC_TERRAFORM` service account but with different roles. Snowflake allows a single user to assume different roles, and the provider's `role` parameter specifies which role to use for that provider instance.

!!! warning "SVC_TERRAFORM Needs All Roles"
    The `SVC_TERRAFORM` user must be granted all four admin roles. You created it with `ACCOUNTADMIN` in the Getting Started section, which inherits from all other admin roles. If you restricted its access, ensure it has `SYSADMIN`, `SECURITYADMIN`, and `USERADMIN` granted.

### Migrate Existing Resources to Aliased Providers

In the [Getting Started](../../getting-started/terraform-setup/snowflake/3-import-admin-user.md) section, you created `snowflake_user.users` resources using an implicit default provider. Now you need to migrate these to use the explicit `user_admin` provider.

#### Step 1: Keep a Default Provider Temporarily

First, add your aliased providers while keeping a default provider for backward compatibility. Your `providers.tf` should have both:

```hcl
# =============================================================================
# Snowflake Providers
# =============================================================================

# Default provider - temporary, for existing resources
provider "snowflake" {
  organization_name = var.snowflake_organization_name
  account_name      = var.snowflake_account_name
  user              = "SVC_TERRAFORM"
  authenticator     = "JWT"
  private_key       = var.SNOWFLAKE_PRIVATE_KEY
  role              = "USERADMIN"  # Match the role your existing resources need
}

# Account Admin - for account-level settings (rarely needed)
provider "snowflake" {
  alias             = "account_admin"
  organization_name = var.snowflake_organization_name
  account_name      = var.snowflake_account_name
  user              = "SVC_TERRAFORM"
  authenticator     = "JWT"
  private_key       = var.SNOWFLAKE_PRIVATE_KEY
  role              = "ACCOUNTADMIN"
}

# ... other aliased providers ...
```

Reinitialise Terraform:

```sh
terraform init
```

Run a plan to verify existing resources are still working:

```sh
terraform plan
```

You should see "No changes" - the existing resources continue using the default provider.

#### Step 2: Update users.tf

Open `users.tf` and add the `provider` argument to your existing resources:

```hcl
resource "snowflake_user" "users" {
  provider = snowflake.user_admin  # Add this line
  for_each = var.snowflake_users

  name         = upper(each.key)
  login_name   = upper(each.key)
  display_name = each.value.display_name
  email        = each.value.email
  default_role = upper(each.value.default_role)
  comment      = each.value.comment

  lifecycle {
    ignore_changes = [
      password,
      rsa_public_key,
      rsa_public_key_2,
      must_change_password,
    ]
  }
}
```

#### Step 3: Plan and Check for Recreation

Run a plan to see what Terraform wants to do:

```sh
terraform plan
```

**If you see "No changes"** - Terraform recognised that the aliased provider uses the same role as the default. You can skip to Step 5.

**If you see resources being destroyed and recreated** - Terraform thinks these are different resources. You need to use state manipulation to tell Terraform they're the same.

#### Step 4: Move State (If Needed)

If Terraform wants to recreate resources, you need to move them in state. First, list your current resources:

```sh
terraform state list | grep snowflake_user
```

For each user, remove and re-import to associate with the new provider:

```sh
# Remove from state (doesn't delete from Snowflake)
terraform state rm 'snowflake_user.users["YOUR_ADMIN_USERNAME"]'
terraform state rm 'snowflake_user.users["SVC_TERRAFORM"]'

# Re-import with the new provider
terraform import 'snowflake_user.users["YOUR_ADMIN_USERNAME"]' YOUR_ADMIN_USERNAME
terraform import 'snowflake_user.users["SVC_TERRAFORM"]' SVC_TERRAFORM
```

!!! warning "Replace Usernames"
    Replace `YOUR_ADMIN_USERNAME` with your actual admin username (uppercase).

Run a plan again:

```sh
terraform plan
```

You should now see "No changes" or only minor attribute updates (not recreation).

#### Step 5: Remove the Default Provider

Once all resources use aliased providers, remove the default provider from `providers.tf`:

```hcl
# =============================================================================
# Snowflake Providers
# =============================================================================
# Each provider uses a different Snowflake role for least-privilege access.
# Modules specify which provider they need via configuration_aliases.

# Account Admin - for account-level settings (rarely needed)
provider "snowflake" {
  alias             = "account_admin"
  # ...
}

# System Admin - for creating warehouses, databases, schemas
provider "snowflake" {
  alias             = "sys_admin"
  # ...
}

# Security Admin - for managing grants and privileges
provider "snowflake" {
  alias             = "security_admin"
  # ...
}

# User Admin - for creating users and roles
provider "snowflake" {
  alias             = "user_admin"
  # ...
}
```

#### Step 6: Final Verification

Reinitialise and verify:

```sh
terraform init
terraform plan
```

Expected output:

```
No changes. Your infrastructure matches the configuration.
```

!!! success "Migration Complete"
    Your existing resources now use explicit aliased providers. All new resources will also use aliased providers, ensuring least-privilege access throughout your configuration.

## Create the Modules Directory

Create the directory structure for your modules:

```sh
mkdir -p modules
```

Your directory structure will look like this:

```
terraform/snowflake/
├── modules/
│   ├── snowflake_warehouse/
│   ├── snowflake_database/
│   ├── snowflake_database_role/
│   ├── snowflake_role/
│   ├── snowflake_schema/
│   ├── snowflake_user/
│   ├── snowflake_storage_integration/
│   └── snowflake_saml2_integration/
├── backend.tf
├── main.tf
├── providers.tf
├── variables.tf
├── terraform.tfvars
├── users.auto.tfvars
├── outputs.tf
├── users.tf
├── warehouses.tf          # Coming soon
├── databases.tf           # Coming soon
├── functional_roles.tf    # Coming soon
└── README.md
```

## Understanding Provider Aliases in Modules

When you create a module, you specify which providers it needs using `configuration_aliases`. Here's how the `snowflake_warehouse` module declares its provider requirements:

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
```

When calling the module, you pass the providers from your root configuration:

```hcl
module "warehouse_loading" {
  source = "./modules/snowflake_warehouse"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  warehouse_name    = "LOADING"
  warehouse_comment = "Warehouse for data loading operations."
  # ...
}
```

Inside the module, resources specify which provider to use:

```hcl
# Create the warehouse using SYSADMIN
resource "snowflake_warehouse" "this" {
  provider = snowflake.sys_admin
  name     = var.warehouse_name
  # ...
}

# Grant usage using SECURITYADMIN
resource "snowflake_grant_privileges_to_account_role" "usage" {
  provider          = snowflake.security_admin
  privileges        = ["USAGE"]
  account_role_name = each.value
  # ...
}
```

## Which Provider for Which Operation?

Here's a quick reference for which provider to use:

| Operation | Provider | Why |
|-----------|----------|-----|
| Create warehouse | `sys_admin` | SYSADMIN owns object creation |
| Create database | `sys_admin` | SYSADMIN owns object creation |
| Create schema | `sys_admin` | SYSADMIN owns object creation |
| Grant privileges | `security_admin` | SECURITYADMIN manages grants |
| Grant role to user | `security_admin` | SECURITYADMIN manages grants |
| Create role | `user_admin` | USERADMIN creates roles |
| Create user | `user_admin` | USERADMIN creates users |
| Create network policy | `account_admin` | Account-level security |
| Create storage integration | `account_admin` | Account-level integration |

## Update main.tf

Ensure your `main.tf` has the correct Terraform and provider version requirements:

```hcl
terraform {
  required_version = ">= 1.14.0"

  required_providers {
    snowflake = {
      source  = "Snowflake-Labs/snowflake"
      version = "~> 0.99"
    }
  }
}
```

!!! warning "Provider Version"
    The Snowflake provider has undergone significant changes. Ensure you're using version 0.99.0 or later for compatibility with the module patterns in this guide.

## Verify the Configuration

Reinitialise Terraform to pick up the provider changes:

```sh
terraform init
```

Run a plan to verify everything works:

```sh
terraform plan
```

You should see no changes if you haven't modified any resources. If you see errors about missing providers, ensure your `providers.tf` is correct.

## Commit Your Work

```sh
git add terraform/snowflake/
git commit -m "Configure multiple Snowflake providers for module-based architecture"
```

## Summary

You've set up the foundation for a module-based Snowflake configuration:

- [x] Multiple providers configured for least-privilege access
- [x] Provider aliases for each admin role (ACCOUNTADMIN, SYSADMIN, SECURITYADMIN, USERADMIN)
- [x] Modules directory created
- [x] Understanding of which provider to use for which operation

## What's Next

With the provider structure in place, you're ready to start building modules. In the next section, you'll create the `snowflake_warehouse` module and set up warehouses for different workloads.

Continue to [Warehouses](2-warehouses.md) →
