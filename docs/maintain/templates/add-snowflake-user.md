---
name: add-snowflake-user
description: Add a new Snowflake user to the Terraform configuration using the snowflake_user module. Handles admin users, developer users, and service accounts with the correct patterns for roles, warehouses, and network policies.
allowed-tools: Read, Write, Glob, Grep, Bash
---

# Add a Snowflake User

This skill adds a new Snowflake user using the `snowflake_user` module, following the established patterns for different user categories.

## When to Use

- Adding a new team member (developer or admin)
- Adding a new service account for a tool (SVC_ prefix)
- Onboarding someone who needs Snowflake access

## Before You Start

Gather the following information:

1. **User name** - e.g. `JBLOGGS` for a human, `SVC_TOOLNAME` for a service account
2. **User type** - admin, developer, or service account
3. **For humans** - email address, first name, last name, display name
4. **Default role** - `ANALYTICS_DEVELOPER` for developers, `SYSADMIN` for admins, or dedicated role for service accounts
5. **Default warehouse** - `DEVELOPER` for humans, `LOADING` / `TRANSFORMING` / `REPORTING` for service accounts
6. **Additional roles** - any extra role grants needed (optional)
7. **Dedicated role** - whether to create one (`user_create_dedicated_role` - always `true` for service accounts)

## Reference: User Categories

Read `snowflake/config/users.tf` and `snowflake/config/*.auto.tfvars` to see existing user definitions. The categories are:

| Category | Example | Default Role | Default Warehouse | Dedicated Role |
|----------|---------|--------------|-------------------|----------------|
| Infrastructure | `SVC_TERRAFORM` | ACCOUNTADMIN | DEVELOPER | Yes |
| Admin | `JBLOGGS_ADMIN` | SYSADMIN | DEVELOPER | No |
| Developer | `JBLOGGS` | ANALYTICS_DEVELOPER | DEVELOPER | No |
| Transformer | `SVC_DBT` | ANALYTICS_TRANSFORMER | TRANSFORMING | Yes |
| Reporter | `SVC_LIGHTDASH` | ANALYTICS_REPORTER | REPORTING | Yes |
| Loader | `SVC_DLT` | Dedicated role | LOADING | Yes |

## Steps

### 1. Determine User Category

Read `snowflake/config/users.tf` to understand the existing module call patterns for each category.

### 2. For Admin or Developer Users

Edit the appropriate variable list in `snowflake/config/users.auto.tfvars`:

- **Admin users** - add to the `admin_user_list` variable
- **Developer users** - add to the `developer_user_list` variable

Each entry needs: `user_name`, `user_display_name`, `user_email`, `user_first_name`, `user_last_name`, and any `user_additional_roles`.

Follow the pattern of existing entries exactly.

### 3. For Service Accounts

Add a new module block to `snowflake/config/users.tf`:

```hcl
module "user_svc_<tool>" {
  source = "./modules/snowflake_user"

  providers = {
    snowflake.security_admin = snowflake.security_admin
    snowflake.user_admin     = snowflake.user_admin
  }

  user_name                  = "SVC_<TOOL>"
  user_display_name          = "<Tool> Service Account"
  user_comment               = "Service account for <tool>."
  user_is_service_account    = true
  user_create_dedicated_role = true
  user_default_warehouse     = module.warehouse_<workload>.warehouse_name

  user_additional_roles = [
    # Add functional roles as needed
  ]
}
```

Replace `<TOOL>` with the tool name in UPPER_CASE and `<workload>` with the appropriate warehouse (`loading`, `transforming`, or `reporting`).

### 4. Add Network Policy Assignment (if Applicable)

If the user needs IP-based access restrictions, update `snowflake/config/network_policies.auto.tfvars` to include the new user in the appropriate network policy.

### 5. Validate

Run from the `snowflake/config/` directory:

```sh
terraform plan
```

Verify:

- User created with correct name, role, and warehouse
- Dedicated role created (if applicable) with correct name
- Role grants are correct
- No unexpected changes to existing resources

### 6. Create Pull Request

Commit changes, push the branch, and create a PR. CI/CD runs `terraform plan` automatically. After approval, CI/CD applies the changes.

## Safety Checks

- **Never** set passwords in Terraform - use key-pair authentication for service accounts
- **Never** hard-code warehouse names - reference `module.warehouse_<name>.warehouse_name`
- The `snowflake_user` module already includes `lifecycle { ignore_changes }` for sensitive fields
- Verify the user's `default_role` exists before applying
- Verify the user's `default_warehouse` exists before applying
- For service accounts, always set `user_create_dedicated_role = true`
- For service accounts, always set `user_is_service_account = true`
