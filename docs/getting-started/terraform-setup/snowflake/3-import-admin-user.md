# Import Existing Users

On this page, you will:

- [x] Create a flexible user configuration pattern
- [x] Import your admin user into Terraform state
- [x] Import the `SVC_TERRAFORM` service account
- [x] Plan and apply changes

## What We're Importing

You currently have two users that exist in Snowflake but aren't managed by Terraform:

1. **Your admin user** - Created during [Snowflake Account Setup](../../account-setup/snowflake.md)
2. **SVC_TERRAFORM** - Created manually in the [previous page](2-create-terraform-service-account.md)

By importing these users into Terraform, future changes go through version control and code review.

!!! note "Import vs Create"
    When you import a resource:

    1. Terraform reads the current configuration from Snowflake
    2. You write matching Terraform code that describes the resource
    3. Terraform adds the resource to its state
    4. Future changes go through the normal plan/apply workflow

    This is different from creating new resources where Terraform provisions them from scratch.

## Gather User Information

First, get the details of your existing users from Snowflake:

```sql
USE ROLE ACCOUNTADMIN;

-- List all users
SHOW USERS;

-- Get details of your admin user
DESC USER your_admin_username;

-- Get details of the service account
DESC USER SVC_TERRAFORM;
```

Note down:

- Your admin username
- The default role assigned to each user
- Any custom settings

## Add Variables

Update `variables.tf` to add user configuration. This follows the same pattern as [AWS IAM User Management](../aws/4-import-iam-users.md) and [GitHub User Management](../github/4-github-user-management.md):

```hcl
variable "snowflake_users" {
  description = "Map of Snowflake users to manage"
  type = map(object({
    display_name = string
    email        = optional(string)
    default_role = string
    comment      = optional(string)
    type         = optional(string, "PERSON")  # PERSON or SERVICE
  }))
  default = {}

  validation {
    condition = alltrue([
      for user, config in var.snowflake_users : contains(["PERSON", "SERVICE"], upper(config.type))
    ])
    error_message = "Valid type values are: PERSON, SERVICE"
  }
}
```

!!! info "User Types"
    Snowflake supports different user types:

    - **PERSON**: Standard users who can log in via the web UI
    - **SERVICE**: Service accounts that cannot use the web UI and are designed for automation

## Configure Users

Create `users.auto.tfvars` for user configuration:

```hcl
snowflake_users = {
  "YOUR_ADMIN_USERNAME" = {  # Replace with your actual username (uppercase)
    display_name = "Your Name (Admin)"
    email        = "your.email@example.com"
    default_role = "PUBLIC"
    comment      = "Account administrator"
    type         = "PERSON"
  }
  "SVC_TERRAFORM" = {
    display_name = "Terraform Service Account"
    default_role = "ACCOUNTADMIN"
    comment      = "Service account for Terraform infrastructure management"
    type         = "SERVICE"
  }
}
```

!!! warning "Username Format"
    Snowflake usernames are case-insensitive but stored in uppercase. Use uppercase in your configuration to match what Snowflake stores.

!!! info "Why a Separate File?"
    Using `users.auto.tfvars` follows the same pattern as [GitHub User Management](../github/4-github-user-management.md) and [AWS IAM Users](../aws/4-import-iam-users.md):

    - **Auto-loaded**: Terraform automatically loads `*.auto.tfvars` files
    - **Separation**: User configuration is separate from general settings
    - **CODEOWNERS**: Can require admin approval for user changes
    - **Clarity**: Easy to see all users in one place

## Create Users Configuration

Create `users.tf`:

```hcl
# =============================================================================
# Snowflake Users
# =============================================================================
# This file manages Snowflake user accounts using a for_each pattern.
# Users are configured in users.auto.tfvars.

# -----------------------------------------------------------------------------
# Snowflake Users
# -----------------------------------------------------------------------------

resource "snowflake_user" "users" {
  for_each = var.snowflake_users

  name         = upper(each.key)
  login_name   = upper(each.key)
  display_name = each.value.display_name
  email        = each.value.email
  default_role = upper(each.value.default_role)
  comment      = each.value.comment

  # Sensitive fields are managed outside Terraform:
  # - Passwords: Set via UI, stored in 1Password
  # - RSA keys: Set via SQL, stored in 1Password/Secrets Manager
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

!!! info "Lifecycle Ignore Changes"
    The `lifecycle { ignore_changes = [...] }` block tells Terraform to ignore changes to sensitive fields like passwords and RSA keys. This prevents Terraform from trying to "fix" these values on each apply.

    These fields are managed manually:

    - **Passwords**: Set in Snowflake UI, stored in 1Password
    - **RSA keys**: Set via SQL, stored in 1Password/Secrets Manager

!!! info "Why `for_each` Instead of Separate Resources?"
    Using `for_each` with a map variable provides several benefits:

    - **DRY**: No duplicate resource blocks for each user
    - **Consistent**: All users are created with the same configuration pattern
    - **Easy to manage**: Add/remove users by editing `users.auto.tfvars`
    - **Clear diffs**: Changes to users show as map updates, not resource replacements

## Create Import Blocks

Create `imports.tf` to import the existing users:

```hcl
# =============================================================================
# Import Blocks
# =============================================================================
# These blocks import existing Snowflake resources into Terraform state.
# Remove these blocks after the initial import is complete.

# Import admin user
import {
  to = snowflake_user.users["YOUR_ADMIN_USERNAME"]  # Replace with your actual username
  id = "YOUR_ADMIN_USERNAME"
}

# Import Terraform service account
import {
  to = snowflake_user.users["SVC_TERRAFORM"]
  id = "SVC_TERRAFORM"
}
```

!!! warning "Update the Username"
    Replace `YOUR_ADMIN_USERNAME` with your actual admin username (uppercase). The key in the `to` attribute must match the key in `users.auto.tfvars`.

## Plan the Import

Run terraform plan to preview what will be imported:

```sh
terraform plan
```

Expected output shows the import actions:

```
snowflake_user.users["YOUR_ADMIN_USERNAME"]: Preparing import... [id=YOUR_ADMIN_USERNAME]
snowflake_user.users["SVC_TERRAFORM"]: Preparing import... [id=SVC_TERRAFORM]

Terraform will perform the following actions:

  # snowflake_user.users["YOUR_ADMIN_USERNAME"] will be imported
    resource "snowflake_user" "users" {
        + comment                      = "Account administrator"
        + default_role                 = "PUBLIC"
        + display_name                 = "Your Name (Admin)"
        + email                        = "your.email@example.com"
        + id                           = "YOUR_ADMIN_USERNAME"
        + login_name                   = "YOUR_ADMIN_USERNAME"
        + name                         = "YOUR_ADMIN_USERNAME"
        ...
    }

  # snowflake_user.users["SVC_TERRAFORM"] will be imported
    resource "snowflake_user" "users" {
        + comment                      = "Service account for Terraform infrastructure management"
        + default_role                 = "ACCOUNTADMIN"
        + display_name                 = "Terraform Service Account"
        + id                           = "SVC_TERRAFORM"
        + login_name                   = "SVC_TERRAFORM"
        + name                         = "SVC_TERRAFORM"
        ...
    }

Plan: 2 to import, 0 to add, 0 to change, 0 to destroy.
```

!!! warning "Review Carefully"
    Check that the planned values match your expectations. If Terraform shows changes to existing values, your configuration may not match the actual Snowflake state.

## Apply the Import

Apply the changes to import the users:

```sh
terraform apply
```

Type `yes` to confirm.

Expected output:

```
snowflake_user.users["YOUR_ADMIN_USERNAME"]: Importing... [id=YOUR_ADMIN_USERNAME]
snowflake_user.users["YOUR_ADMIN_USERNAME"]: Import complete [id=YOUR_ADMIN_USERNAME]
snowflake_user.users["SVC_TERRAFORM"]: Importing... [id=SVC_TERRAFORM]
snowflake_user.users["SVC_TERRAFORM"]: Import complete [id=SVC_TERRAFORM]

Apply complete! Resources: 2 imported, 0 added, 0 changed, 0 destroyed.
```

## Verify the Import

Check that the users are now in Terraform state:

```sh
terraform state list
```

Expected output:

```
snowflake_user.users["SVC_TERRAFORM"]
snowflake_user.users["YOUR_ADMIN_USERNAME"]
```

View the details of an imported user:

```sh
terraform state show 'snowflake_user.users["YOUR_ADMIN_USERNAME"]'
```

## Clean Up Import Blocks

After successful import, delete the `imports.tf` file:

```sh
rm imports.tf
```

Run `terraform plan` to verify no changes are planned:

```sh
terraform plan
```

Expected output:

```
No changes. Your infrastructure matches the configuration.
```

## Add Outputs

Update `outputs.tf` to include user information:

```hcl
# User outputs
output "snowflake_user_names" {
  description = "Names of all Snowflake users"
  value       = { for k, v in snowflake_user.users : k => v.name }
}
```

## Commit Your Work

Commit your progress:

```sh
git add terraform/snowflake/
git commit -m "Import Snowflake users with for_each pattern"
```

## Managing Users

### Adding a New User

Add them to `users.auto.tfvars`:

```hcl
snowflake_users = {
  # ... existing users ...
  "ALICE_ADMIN" = {
    display_name = "Alice Smith (Admin)"
    email        = "alice@example.com"
    default_role = "PUBLIC"
    comment      = "Platform engineer admin account"
    type         = "PERSON"
  }
}
```

Run `terraform apply` - the user will be created.

!!! note "New Users Need Credentials"
    After Terraform creates a user, you'll need to:

    1. Set an initial password via SQL or Snowflake UI
    2. Have the user set up MFA
    3. Store credentials in 1Password

### Changing a User's Configuration

Update their entry in `users.auto.tfvars`:

```hcl
"ALICE_ADMIN" = {
  display_name = "Alice Smith (Admin)"
  email        = "alice.smith@example.com"  # Updated email
  default_role = "SYSADMIN"                 # Changed default role
  comment      = "Platform engineer admin account"
  type         = "PERSON"
}
```

Run `terraform apply` - Terraform will update the user.

### Offboarding a User

Remove them from `users.auto.tfvars` and run `terraform apply`. Terraform will delete the user.

!!! warning "Before Offboarding"
    Before removing a user:

    1. Revoke any active sessions
    2. Review objects they own (transfer ownership if needed)
    3. Remove from any roles they're granted

## Current State

You now have the following in Terraform state:

| Resource | Key | Notes |
|----------|-----|-------|
| Admin user | `snowflake_user.users["YOUR_ADMIN_USERNAME"]` | Your personal admin account |
| SVC_TERRAFORM | `snowflake_user.users["SVC_TERRAFORM"]` | Terraform service account |

## What About Other Users?

!!! info "Regular Users"
    In this section, we only manage the admin user and service account - the minimum needed for Terraform to work. You'll create additional users (developers, service accounts for dbt, Airbyte, etc.) in the [Data Warehouse](../../../build/data-warehouse/index.md) section.

    The guidance about having separate admin and regular accounts for the same person (e.g. `JBLOGGS_ADMIN` and `JBLOGGS`) will be addressed when setting up the full user management in the Data Warehouse section.

## What's Next

You've imported your existing users into Terraform. Next, you'll finish up by updating CI/CD workflows and verifying everything works end-to-end.

Continue to [finishing up](4-finishing-up.md) →
