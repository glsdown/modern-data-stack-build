# Network Policies

On this page, you will:

- [x] Understand Snowflake network policies and network rules
- [x] Create network rules in the ADMIN database
- [x] Create network policies that reference network rules
- [x] Attach policies to users

## What Are Network Policies and Rules?

Network policies restrict which IP addresses can connect to your Snowflake account. They provide an additional security layer beyond username/password or key-pair authentication.

Snowflake's network access control uses two components:

- **Network rules**: Schema-level objects that define lists of IP addresses or ranges. These are reusable building blocks.
- **Network policies**: Account-level objects that reference network rules to allow or block access. Policies are attached to users or the account.

Common use cases:

- **Office access only**: Restrict human users to your office IP range
- **VPN access only**: Restrict human users to a company VPN
- **Service account lockdown**: Limit service accounts to specific cloud provider IP ranges
- **CI/CD access**: Allow GitHub Actions or other CI/CD platforms

## When to Use Network Policies

Network policies are most valuable when:

| Scenario | Recommendation |
|----------|----------------|
| Service accounts (dbt, Fivetran, Metabase) | Recommended - lock to known IPs |
| Admin accounts | Recommended - restrict to office/VPN |
| Developer accounts | Optional - depends on remote work policy |
| CI/CD service accounts | Recommended - use platform IP ranges |

!!! warning "Plan Before Implementing"
    A misconfigured network policy can lock you out of your account. Always ensure at least one admin account can connect before applying restrictive policies.

## Add the GitHub Provider

To dynamically fetch GitHub Actions IP ranges, add the GitHub provider to your `providers.tf`:

```hcl
# -----------------------------------------------------------------------------
# GitHub Provider (for IP ranges)
# -----------------------------------------------------------------------------
provider "github" {}
```

Add the provider to your `versions.tf` or `main.tf`:

```hcl
terraform {
  required_providers {
    # ... existing providers ...
    github = {
      source  = "integrations/github"
      version = "~> 6.0"
    }
  }
}
```

## Fetch GitHub IP Ranges

Create a data source to fetch the current GitHub Actions IP ranges. Add to `network_policies.tf`:

```hcl
# =============================================================================
# GitHub IP Ranges
# =============================================================================
# Dynamically fetch GitHub Actions IP ranges from the GitHub API.
# This ensures IP ranges are always current without manual maintenance.

data "github_ip_ranges" "latest" {}
```

This data source provides `data.github_ip_ranges.latest.actions_ipv4` - a list of all IPv4 CIDR ranges used by GitHub Actions runners.

!!! tip "Keep lists up to date"
  Due to how many IP addresses there are for GitHub actions, the GitHub provider makes a dynamic reference available within terraform. However, many companies do not do this, and you will need to hard-code the values. Make sure to set regular reminders to check the lists are still up-to-date.

## Create the Network Policies Schema

Network rules are schema-level objects, so we need a schema to store them. Add a network policies schema to the ADMIN database in `schemas.tf`:

```hcl
module "schema_admin_network_policies" {
  source = "./modules/snowflake_schema"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  schema_name          = "NETWORK_POLICIES"
  schema_database_name = module.database_admin.database_name
  schema_comment       = "Network rules for access control policies."
}
```

## Create Network Rules

Network rules define the IP addresses that can be referenced by policies. Create `network_policies.tf`:

```hcl
# =============================================================================
# Network Rules
# =============================================================================
# Reusable IP address lists stored in the ADMIN database.

# -----------------------------------------------------------------------------
# GitHub Actions Network Rule
# -----------------------------------------------------------------------------
resource "snowflake_network_rule" "github_actions" {
  provider = snowflake.account_admin

  name       = "GITHUB_ACTIONS"
  database   = module.database_admin.database_name
  schema     = module.schema_admin_network_policies.schema_name
  comment    = "GitHub Actions runner IP ranges."
  type       = "IPV4"
  mode       = "INGRESS"
  value_list = data.github_ip_ranges.latest.actions_ipv4
}

# -----------------------------------------------------------------------------
# Office Network Rule
# -----------------------------------------------------------------------------
resource "snowflake_network_rule" "office" {
  provider = snowflake.account_admin

  name       = "OFFICE_ACCESS"
  database   = module.database_admin.database_name
  schema     = module.schema_admin_network_policies.schema_name
  comment    = "Office and VPN IP ranges."
  type       = "IPV4"
  mode       = "INGRESS"
  value_list = var.office_ip_ranges
}
```

## Add Variables

Add the office IP ranges variable to `variables.tf`:

```hcl
variable "office_ip_ranges" {
  description = "Office and VPN IP ranges for human user access."
  type        = list(string)
  default     = ["0.0.0.0/0"]  # Allow all by default - restrict in tfvars
}
```

Configure your office IP ranges in `network_policies.auto.tfvars`:

```hcl
office_ip_ranges = [
  "203.0.113.0/24",  # Office IP range
  "198.51.100.0/24", # VPN IP range
]
```

!!! warning "Update these"
  Make sure that you update these with your actual IP addresses or you will be locked out. The CIDR notation `0.0.0.0/0` means "any IP address". Use this initially if you don't want to restrict access yet, then update with specific ranges when ready.

## Create Network Policies

Network policies reference network rules to control access. Add to `network_policies.tf`:

```hcl
# =============================================================================
# Network Policies
# =============================================================================
# Policies that reference network rules and are attached to users.

# -----------------------------------------------------------------------------
# GitHub Actions Policy
# -----------------------------------------------------------------------------
resource "snowflake_network_policy" "github_actions" {
  provider = snowflake.account_admin

  name    = "GITHUB_ACTIONS_POLICY"
  comment = "Access from GitHub Actions runners."

  allowed_network_rule_list = [
    snowflake_network_rule.github_actions.fully_qualified_name
  ]
}

# -----------------------------------------------------------------------------
# Office Access Policy
# -----------------------------------------------------------------------------
resource "snowflake_network_policy" "office" {
  provider = snowflake.account_admin

  name    = "OFFICE_ACCESS_POLICY"
  comment = "Access from office network and VPN."

  allowed_network_rule_list = [
    snowflake_network_rule.office.fully_qualified_name
  ]
}
```

## Attach Policies to Users

Network policies are attached to individual users. This restricts where they can connect from.

### Terraform Service Account

The `SVC_TERRAFORM` service account runs in GitHub Actions, so it should be restricted to GitHub's IP ranges. Update the user resource to attach the policy.

First, add the `network_policy` attribute to the user module. Update `modules/snowflake_user/variables.tf`:

```hcl
variable "user_network_policy" {
  description = "Network policy to attach to the user"
  type        = string
  default     = null
}
```

Update `modules/snowflake_user/main.tf` to apply the policy:

```hcl
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

  # Lifecycle as before...
}

# -----------------------------------------------------------------------------
# Network Policy Attachment
# -----------------------------------------------------------------------------
resource "snowflake_network_policy_attachment" "this" {
  count = var.user_network_policy != null ? 1 : 0

  provider       = snowflake.security_admin
  network_policy_name = var.user_network_policy
  users          = [snowflake_user.this.name]
}
```

Now attach the GitHub Actions policy to `SVC_TERRAFORM` in `users.tf`:

```hcl
module "user_svc_terraform" {
  source = "./modules/snowflake_user"

  providers = {
    snowflake.security_admin = snowflake.security_admin
    snowflake.user_admin     = snowflake.user_admin
  }

  user_name               = "SVC_TERRAFORM"
  user_comment            = "Service account for Terraform infrastructure management."
  user_display_name       = "Terraform Service Account"
  user_is_service_account = true

  user_default_warehouse = module.warehouse_developer.warehouse_name
  user_default_role      = "ACCOUNTADMIN"

  # Restrict to GitHub Actions IP ranges
  user_network_policy = snowflake_network_policy.github_actions.name
}
```

### Human Users

For human users defined via tfvars, we need a different approach. Since tfvars files can only contain literal values (not Terraform resource references), we use a lookup map to translate string keys to actual policy names.

First, update the user variable types to include network policy:

```hcl
variable "admin_users" {
  description = "Admin users who get both admin and developer accounts."
  type = map(object({
    display_name   = string
    first_name     = string
    last_name      = string
    email          = string
    network_policy = optional(string)  # Add this
  }))
  default = {}
}

variable "developer_users" {
  description = "Non-admin developers."
  type = map(object({
    display_name   = string
    first_name     = string
    last_name      = string
    email          = string
    network_policy = optional(string)  # Add this
  }))
  default = {}
}
```

Then reference policies in `users.auto.tfvars`:

```hcl
admin_users = {
  "JBLOGGS" = {
    display_name   = "Joe Bloggs"
    first_name     = "Joe"
    last_name      = "Bloggs"
    email          = "joe.bloggs@company.com"
    network_policy = "office"  # Apply office access policy
  }
}

developer_users = {
  "ASMITH" = {
    display_name   = "Alice Smith"
    first_name     = "Alice"
    last_name      = "Smith"
    email          = "alice.smith@company.com"
    network_policy = "office"
  }
}
```

Create a local map to look up policies by name in `users.tf`:

```hcl
locals {
  network_policies = {
    "office" = snowflake_network_policy.office.name
    "github" = snowflake_network_policy.github_actions.name
  }
}
```

Update the user module calls to apply the policy:

```hcl
module "user_developers" {
  source = "./modules/snowflake_user"
  # ... other config ...

  for_each = var.developer_users

  # Lookup the network policy by key, if specified
  user_network_policy = each.value.network_policy != null ? local.network_policies[each.value.network_policy] : null
}
```

## Account-Level Policies

You can also set a default network policy for the entire account. This applies to any user without an explicit policy:

```hcl
resource "snowflake_account_parameter" "network_policy" {
  provider = snowflake.account_admin
  key      = "NETWORK_POLICY"
  value    = snowflake_network_policy.office.name
}
```

!!! danger "Use With Caution"
    Account-level policies affect all users. Ensure you have at least one admin account that can connect before setting this, or you risk locking yourself out.

## Commit and Deploy

Commit your changes and push to trigger the CI/CD pipeline:

```sh
git add terraform/snowflake/
git commit -m "Add network policies for service accounts and users"
git push
```

Review the plan carefully - verify that:

- Network rules are created with correct IP ranges
- Network policies reference the correct rules
- Policies are attached to the intended users
- No existing access is accidentally blocked

## Verify in Snowflake

After deployment, verify the rules and policies:

```sql
-- List all network rules
SHOW NETWORK RULES IN SCHEMA ADMIN.NETWORK_POLICIES;

-- Check rule details
DESCRIBE NETWORK RULE ADMIN.NETWORK_POLICIES.GITHUB_ACTIONS;

-- List all network policies
SHOW NETWORK POLICIES;

-- Check policy details
DESCRIBE NETWORK POLICY GITHUB_ACTIONS_POLICY;

-- Check which users have policies attached
SELECT name, network_policy
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE network_policy IS NOT NULL;
```

## Troubleshooting

If a user gets locked out:

1. **Use an admin account without a network policy** to modify or remove the problematic policy
2. **Connect from an allowed IP** if possible
3. **Contact Snowflake Support** as a last resort - they can help with account recovery

To temporarily disable a policy for a user:

```sql
-- Run as SECURITYADMIN
ALTER USER JBLOGGS UNSET NETWORK_POLICY;
```

## Summary

You've implemented network-based access control:

- [x] Created network rules in the ADMIN database for reusable IP lists
- [x] Used the GitHub provider to dynamically fetch GitHub Actions IP ranges
- [x] Created network policies that reference network rules
- [x] Attached the GitHub Actions policy to the SVC_TERRAFORM service account
- [x] Understood how to apply policies to human users

## What's Next

With network security in place, you're ready to set up storage integrations for loading data from cloud storage. In the next section, you'll create S3 or GCS integrations for your data loaders.

Continue to [Storage Integrations](8-storage-integrations.md) →
