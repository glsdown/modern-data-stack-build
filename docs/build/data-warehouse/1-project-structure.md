# Project Structure

On this page, you will:

- [x] Understand the module-based approach for Snowflake Terraform
- [x] Create the directory structure for modules
- [x] Learn how to pass providers to modules

## Why Modules?

In the [Getting Started](../../getting-started/terraform-setup/snowflake/index.md) section, you created a Terraform configuration with multiple aliased providers and basic user management. That flat structure works for simple setups, but as your data warehouse grows, you'll need a more scalable approach.

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

## Create the Modules Directory

Navigate to your Snowflake Terraform directory:

```sh
cd ~/projects/data/data-stack-infrastructure/terraform/snowflake
```

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

## Using Provider Aliases in Modules

You set up four aliased providers in the [Getting Started](../../getting-started/terraform-setup/snowflake/1-setup-snowflake-backend.md) section:

| Provider | Role | Purpose |
|----------|------|---------|
| `snowflake.account_admin` | ACCOUNTADMIN | Network policies, storage integrations |
| `snowflake.sys_admin` | SYSADMIN | Warehouses, databases, schemas |
| `snowflake.security_admin` | SECURITYADMIN | Grants and privileges |
| `snowflake.user_admin` | USERADMIN | Users and roles |

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

## Commit Your Work

```sh
git add terraform/snowflake/modules/
git commit -m "Create modules directory structure"
```

## Summary

You've set up the foundation for a module-based Snowflake configuration:

- [x] Modules directory created
- [x] Understanding of how to pass providers to modules
- [x] Reference for which provider to use for each operation

## What's Next

With the project structure in place, you're ready to start building modules. In the next section, you'll create the `snowflake_warehouse` module and set up warehouses for different workloads.

Continue to [Warehouses](2-warehouses.md) →
