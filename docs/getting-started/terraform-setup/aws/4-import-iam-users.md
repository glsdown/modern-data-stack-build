# Import IAM Users

On this page, you will:

- [x] Import your personal IAM user into Terraform
- [x] Create single-role assume policies for flexible access control
- [x] Set up a pattern for managing team members with configurable role access

## Navigate to Your Terraform Directory

```sh
cd ~/projects/data/data-stack-infrastructure/terraform/aws
```

## Understanding IAM User Management

You created your personal IAM user in [AWS Account Setup](../../account-setup/aws.md). This user has:

- MFA enabled
- Access keys for CLI access
- An `AssumeRolesPolicy` allowing role assumption

We'll import this user into Terraform and set up flexible role-based policies that can be combined as needed.

!!! warning "What We Don't Manage in Terraform"
    Some IAM user resources should **not** be managed in Terraform:

    - **Access keys**: Creating these in Terraform would expose them in state
    - **MFA devices**: These are user-specific and require interactive setup
    - **Login profiles**: Console passwords should be set by users, not in code

    Terraform manages the user resource and policy attachments, whilst users manage their own credentials.

## Single-Role Policy Model

Rather than creating tiered policies that bundle multiple roles together, we'll create one policy per role. Users can then be assigned any combination of policies based on their needs.

| Policy | Role Allowed |
|--------|--------------|
| `AssumeAdminRolePolicy` | AdminRole |
| `AssumeDataEngineerRolePolicy` | DataEngineerRole |
| `AssumeInfrastructureAdminRolePolicy` | InfrastructureAdminRole |

This approach follows the principle of least privilege whilst providing flexibility:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Flexible Role Assignment                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Alice (Platform Engineer)                                      │
│    ├── AssumeDataEngineerRolePolicy      ✓                      │
│    └── AssumeInfrastructureAdminRolePolicy ✓                    │
│                                                                 │
│  Bob (Data Engineer)                                            │
│    └── AssumeDataEngineerRolePolicy      ✓                      │
│                                                                 │
│  Charlie (Account Admin)                                        │
│    ├── AssumeAdminRolePolicy             ✓                      │
│    ├── AssumeDataEngineerRolePolicy      ✓                      │
│    └── AssumeInfrastructureAdminRolePolicy ✓                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Gather User Information

First, retrieve information about your existing IAM user:

```sh
# List IAM users
aws iam list-users --profile infrastructure-admin

# Get your user details (replace YOUR_USERNAME)
aws iam get-user --user-name YOUR_USERNAME --profile infrastructure-admin

# List policies attached to your user
aws iam list-attached-user-policies --user-name YOUR_USERNAME --profile infrastructure-admin
```

Note the username and any attached policies.

## Add Variables

Update `variables.tf`:

```hcl
variable "iam_users" {
  description = "Map of IAM users with their role access"
  type = map(object({
    description = string
    roles       = list(string)  # Valid values: "admin", "data_engineer", "infrastructure_admin"
  }))
  default = {}

  validation {
    condition = alltrue([
      for user, config in var.iam_users : alltrue([
        for role in config.roles : contains(["admin", "data_engineer", "infrastructure_admin"], role)
      ])
    ])
    error_message = "Valid role values are: admin, data_engineer, infrastructure_admin"
  }
}
```

## Configure Users

Create `iam_users.auto.tfvars` for user configuration:

```hcl
iam_users = {
  "your-username" = {  # Replace with your actual username
    description = "Account owner - full access"
    roles       = ["admin", "data_engineer", "infrastructure_admin"]
  }
  "alice@example.com" = {
    description = "Alice - Platform Engineer"
    roles       = ["data_engineer", "infrastructure_admin"]
  }
  "bob@example.com" = {
    description = "Bob - Data Engineer"
    roles       = ["data_engineer"]
  }
}
```

!!! info "Why a Separate File?"
    Using `iam_users.auto.tfvars` follows the same pattern as [GitHub User Management](../github/4-github-user-management.md):

    - **Auto-loaded**: Terraform automatically loads `*.auto.tfvars` files
    - **Separation**: User configuration is separate from general settings
    - **CODEOWNERS**: Can require admin approval for user changes
    - **Clarity**: Easy to see all users in one place

!!! tip "Username Format"
    IAM usernames can be:

    - Simple names: `alice`, `bob-smith`
    - Email addresses: `alice@example.com`

    Using email addresses makes it easier to identify users and aligns with SSO practices if you adopt them later.

!!! tip "Role Combinations"
    Common role combinations:

    - **Account administrators**: `["admin", "data_engineer", "infrastructure_admin"]`
    - **Platform engineers**: `["data_engineer", "infrastructure_admin"]`
    - **Data engineers/analysts**: `["data_engineer"]`

## Create IAM Users Configuration

Create `iam_users.tf`:

```hcl
# =============================================================================
# IAM Users
# =============================================================================

# -----------------------------------------------------------------------------
# Locals - Role references and user-role flattening
# -----------------------------------------------------------------------------

locals {
  # Map role keys to actual IAM role resources from iam_roles.tf
  # This avoids hard-coding role names and keeps everything in sync
  assumable_roles = {
    admin                = aws_iam_role.admin
    data_engineer        = aws_iam_role.data_engineer
    infrastructure_admin = aws_iam_role.infrastructure_admin
  }

  # Flatten user-role assignments into individual attachments
  # Creates entries like: { "alice/data_engineer" = { user = "alice", role = "data_engineer" } }
  user_role_attachments = merge([
    for username, config in var.iam_users : {
      for role in config.roles : "${username}/${role}" => {
        user = username
        role = role
      }
    }
  ]...)
}

# -----------------------------------------------------------------------------
# Assume Role Policy Documents - One per role
# -----------------------------------------------------------------------------

data "aws_iam_policy_document" "assume_role" {
  for_each = local.assumable_roles

  statement {
    actions   = ["sts:AssumeRole"]
    resources = [each.value.arn]
  }
}

# -----------------------------------------------------------------------------
# Assume Role Policies - One per role, created dynamically
# -----------------------------------------------------------------------------

resource "aws_iam_policy" "assume_role" {
  for_each = local.assumable_roles

  name        = "Assume${each.value.name}Policy"
  description = "Allows assuming ${each.value.name}"
  policy      = data.aws_iam_policy_document.assume_role[each.key].json

  tags = {
    Name = "Assume${each.value.name}Policy"
  }
}

# -----------------------------------------------------------------------------
# IAM Users
# -----------------------------------------------------------------------------

resource "aws_iam_user" "users" {
  for_each = var.iam_users

  name          = each.key
  force_destroy = true  # Allow deletion even if user has access keys or MFA devices

  tags = {
    Name        = each.key
    Description = each.value.description
    Roles       = join(", ", each.value.roles)
  }
}

# -----------------------------------------------------------------------------
# Policy Attachments - One per user/role combination
# -----------------------------------------------------------------------------

resource "aws_iam_user_policy_attachment" "user_roles" {
  for_each = local.user_role_attachments

  user       = aws_iam_user.users[each.value.user].name
  policy_arn = aws_iam_policy.assume_role[each.value.role].arn
}
```

!!! info "force_destroy Setting"
    Setting `force_destroy = true` allows Terraform to delete users even if they have:

    - Access keys (managed outside Terraform)
    - MFA devices
    - Login profiles (console passwords)

    Without this, `terraform destroy` or removing a user from `iam_users.auto.tfvars` would fail if they have any of these resources attached.

!!! info "Two Patterns Working Together"
    This configuration uses two powerful Terraform patterns:

    **1. Policy documents**: Using `aws_iam_policy_document` is the Terraform-native way to create IAM policies. It validates at plan time, handles JSON formatting automatically, and can reference other resources directly.

    **2. Flatten pattern**: The `user_role_attachments` local transforms nested user-role lists into a flat map for policy attachments - the same pattern from [GitHub User Management](../github/4-github-user-management.md).

    For the example configuration, `user_role_attachments` creates:
    ```
    {
      "your-username/admin"                        = { user = "your-username", role = "admin" }
      "your-username/data_engineer"                = { user = "your-username", role = "data_engineer" }
      "your-username/infrastructure_admin"         = { user = "your-username", role = "infrastructure_admin" }
      "alice@example.com/data_engineer"            = { user = "alice@example.com", role = "data_engineer" }
      "alice@example.com/infrastructure_admin"     = { user = "alice@example.com", role = "infrastructure_admin" }
      "bob@example.com/data_engineer"              = { user = "bob@example.com", role = "data_engineer" }
    }
    ```

## Handle Existing AssumeRolesPolicy

If you created an `AssumeRolesPolicy` during the AWS account setup, delete it before applying:

```sh
# First, detach the policy from your user
aws iam detach-user-policy \
  --user-name your-username \
  --policy-arn arn:aws:iam::123456789012:policy/AssumeRolesPolicy \
  --profile admin

# Then delete the policy
aws iam delete-policy \
  --policy-arn arn:aws:iam::123456789012:policy/AssumeRolesPolicy \
  --profile admin
```

Replace `your-username` with your actual username and `123456789012` with your account ID.

## Add Import Blocks

Add these import blocks to `imports.tf`:

```hcl
# IAM Users - Import existing users
import {
  to = aws_iam_user.users["your-username"]  # Replace with your actual username
  id = "your-username"
}

# Add import blocks for any other existing users
# import {
#   to = aws_iam_user.users["alice@example.com"]
#   id = "alice@example.com"
# }
```

!!! warning "Update the Values"
    Replace `your-username` with your actual IAM username.

## Plan and Apply

```sh
terraform plan
```

Review the output. You should see:

- Users being imported or created
- Three new policies being created
- Policy attachments being created for each user/role combination

Apply the changes:

```sh
terraform apply
```

## Verify the Setup

```sh
# Check policies in state
terraform state list | grep aws_iam_policy

# Check users in state
terraform state list | grep aws_iam_user
```

Expected output (for the example configuration):

```
aws_iam_policy.assume_role["admin"]
aws_iam_policy.assume_role["data_engineer"]
aws_iam_policy.assume_role["infrastructure_admin"]
aws_iam_user.users["alice@example.com"]
aws_iam_user.users["bob@example.com"]
aws_iam_user.users["your-username"]
aws_iam_user_policy_attachment.user_roles["alice@example.com/data_engineer"]
aws_iam_user_policy_attachment.user_roles["alice@example.com/infrastructure_admin"]
aws_iam_user_policy_attachment.user_roles["bob@example.com/data_engineer"]
aws_iam_user_policy_attachment.user_roles["your-username/admin"]
aws_iam_user_policy_attachment.user_roles["your-username/data_engineer"]
aws_iam_user_policy_attachment.user_roles["your-username/infrastructure_admin"]
```

Verify the policies are attached correctly:

```sh
aws iam list-attached-user-policies --user-name alice@example.com --profile infrastructure-admin
```

## Add Outputs

Update `outputs.tf`:

```hcl
# IAM User outputs
output "iam_user_arns" {
  description = "ARNs of all IAM users"
  value       = { for k, v in aws_iam_user.users : k => v.arn }
}

# Assume role policy ARNs
output "assume_role_policy_arns" {
  description = "ARNs of the assume role policies"
  value       = { for k, v in aws_iam_policy.assume_role : k => v.arn }
}
```

## Commit Your Work

```sh
git add terraform/aws/
git commit -m "Import IAM users with flexible role-based policies"
```

## Managing Users

### Adding a New User

Add them to `iam_users.auto.tfvars`:

```hcl
iam_users = {
  # ... existing users ...
  "david@example.com" = {
    description = "David - Data Analyst"
    roles       = ["data_engineer"]
  }
}
```

Run `terraform apply` - the user and their policy attachments will be created.

### Changing a User's Roles

Update their `roles` list in `iam_users.auto.tfvars`:

```hcl
"alice@example.com" = {
  description = "Alice - Platform Engineer (promoted)"
  roles       = ["admin", "data_engineer", "infrastructure_admin"]  # Added admin
}
```

Run `terraform apply` - Terraform will attach the new policy.

### Removing a Role

Remove the role from the list:

```hcl
"alice@example.com" = {
  description = "Alice - Data Engineer"
  roles       = ["data_engineer"]  # Removed infrastructure_admin
}
```

Run `terraform apply` - Terraform will detach the policy.

### Offboarding a User

Remove them from `iam_users.auto.tfvars` and run `terraform apply`. Terraform will:

1. Detach all policies from the user
2. Delete the user

!!! warning "Before Offboarding"
    Before removing a user:

    1. Revoke any active sessions
    2. Review resources they created
    3. Transfer ownership if needed

## Security Considerations

!!! warning "User Credentials Are Not in Terraform"
    Terraform manages the user resource but does **not** manage:

    - Access keys (would be exposed in state)
    - MFA devices (require user interaction)
    - Console passwords (should be user-managed)

    Users should create their own access keys and set up MFA through the AWS Console or CLI.

!!! tip "Validation Catches Typos"
    The variable validation ensures only valid role names are used. If someone types `"data-engineer"` instead of `"data_engineer"`, Terraform will fail with a clear error message during plan.

## Troubleshooting

### Error: User Already Exists

If Terraform tries to create a user that already exists, add an import block:

```hcl
import {
  to = aws_iam_user.users["alice@example.com"]
  id = "alice@example.com"
}
```

### Error: Invalid Role Name

If you see the validation error, check your `iam_users.auto.tfvars` for typos in role names. Valid values are:

- `admin`
- `data_engineer`
- `infrastructure_admin`

### Migrating from Old AssumeRolesPolicy

If users have the old `AssumeRolesPolicy` attached, list who has it:

```sh
aws iam list-entities-for-policy \
  --policy-arn arn:aws:iam::123456789012:policy/AssumeRolesPolicy \
  --profile admin
```

Detach from each user, then delete the policy before running Terraform.

## What's Next

You've successfully set up flexible IAM user management:

- [x] Personal IAM user managed in code
- [x] Single-role assume policies for granular access control
- [x] Pattern established for managing users with configurable role combinations

Continue to [import budget alerts](5-import-budget-alerts.md) →
