# SSO Setup (Optional)

On this page, you will:

- [x] Understand SSO options for Snowflake
- [x] Create the SAML2 security integration in Terraform
- [x] Configure your identity provider
- [x] Update user configuration for SSO authentication
- [x] Understand the hybrid user management approach

## Why SSO?

Single Sign-On provides several benefits:

- **Security**: Centralised authentication with your identity provider's policies (MFA, session management)
- **User experience**: One set of credentials for all tools
- **Compliance**: Audit logs in one place, automatic deprovisioning when users leave
- **Reduced management**: No Snowflake-specific passwords to manage

## SSO Options

Snowflake supports multiple authentication methods:

| Method | Use Case | Terraform Managed? |
|--------|----------|-------------------|
| SAML 2.0 | Enterprise SSO (Okta, Azure AD, etc.) | Yes |
| OAuth | Programmatic access, BI tools | Yes |
| Key Pair | Service accounts, CI/CD | Yes |
| Password | Fallback, initial setup | Yes |

For human users, SAML SSO is recommended. For service accounts, key pair authentication is more secure than passwords.

## Create SAML2 Security Integration

The SAML2 security integration tells Snowflake how to authenticate users via your identity provider. This is managed in Terraform for version control and consistency.

### Add Variables

Add to `variables.tf`:

```hcl
variable "sso_enabled" {
  description = "Whether SSO is enabled"
  type        = bool
  default     = false
}

variable "sso_issuer" {
  description = "SAML2 issuer (from your IdP)"
  type        = string
  default     = ""
}

variable "sso_sso_url" {
  description = "SAML2 SSO URL (from your IdP)"
  type        = string
  default     = ""
}

variable "sso_certificate" {
  description = "SAML2 X509 certificate (from your IdP)"
  type        = string
  default     = ""
  sensitive   = true
}
```

### Create the Security Integration

Create `sso.tf`:

```hcl
# =============================================================================
# SSO Security Integration
# =============================================================================
# SAML2 integration for human user authentication.
# Enable by setting sso_enabled = true and providing IdP values.

resource "snowflake_saml2_integration" "sso" {
  count    = var.sso_enabled ? 1 : 0
  provider = snowflake.account_admin

  name                        = "SSO_INTEGRATION"
  enabled                     = true
  saml2_issuer                = var.sso_issuer
  saml2_sso_url               = var.sso_sso_url
  saml2_provider              = "CUSTOM"  # Or "OKTA", "ADFS"
  saml2_x509_cert             = var.sso_certificate
  saml2_enable_sp_initiated   = true
  saml2_sp_initiated_login_page_label = "SSO Login"
}
```

### Configure in terraform.tfvars

```hcl
# SSO Configuration
sso_enabled = true
sso_issuer  = "http://www.okta.com/exk123456"
sso_sso_url = "https://your-org.okta.com/app/snowflake/exk123456/sso/saml"
# sso_certificate is sensitive - set via environment variable:
# export TF_VAR_sso_certificate="MIIDp..."
```

!!! tip "Certificate Management"
    Store the X509 certificate in AWS Secrets Manager and retrieve it as an environment variable for Terraform, keeping it out of version control.

## Configure Your Identity Provider

Each provider has specific SAML configuration:

=== "Okta"

    1. Create a new **Snowflake** application in Okta
    2. Configure SAML settings:
        - **Single sign on URL**: `https://<account>.snowflakecomputing.com/fed/login`
        - **Audience URI**: `https://<account>.snowflakecomputing.com`
    3. Attribute mappings:
        - `email` → `user.email`
        - `firstName` → `user.firstName`
        - `lastName` → `user.lastName`
    4. Copy the **Identity Provider metadata** values to your Terraform variables

=== "Azure AD"

    1. Create a new **Snowflake** enterprise application in Azure AD
    2. Configure SAML settings:
        - **Identifier**: `https://<account>.snowflakecomputing.com`
        - **Reply URL**: `https://<account>.snowflakecomputing.com/fed/login`
    3. Configure claims:
        - Required claim: `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress`
    4. Download the **Certificate (Base64)** and copy the **Login URL** and **Azure AD Identifier**

## Optional: SCIM User Provisioning

SCIM (System for Cross-domain Identity Management) automatically syncs users from your identity provider to Snowflake. This is optional - you can manage users in Terraform instead.

### Create SCIM Integration

Add to `sso.tf`:

```hcl
resource "snowflake_scim_integration" "scim" {
  count    = var.sso_enabled && var.scim_enabled ? 1 : 0
  provider = snowflake.account_admin

  name           = "SCIM_INTEGRATION"
  enabled        = true
  scim_client    = "GENERIC"  # Or "OKTA", "AZURE"
  run_as_role    = "SCIM_PROVISIONER"  # Create this role first
}
```

### Generate SCIM Token

After applying, generate a SCIM token:

```sql
-- As ACCOUNTADMIN
SELECT SYSTEM$GENERATE_SCIM_ACCESS_TOKEN('SCIM_INTEGRATION');
```

!!! warning "Token Security"
    The SCIM token provides significant access. Store it securely in your identity provider and rotate it periodically.

## User Configuration for SSO

When SSO is enabled, Terraform continues to manage users (creating them, assigning roles, setting defaults), but authentication happens via SSO instead of passwords.

### Update the User Module

The `snowflake_user` module already supports SSO users. The key configuration points:

1. **`login_name`** must match what your IdP sends (usually email address)
2. **No password** is set - authentication happens via SSO
3. **User must exist** in Snowflake before they can authenticate via SSO

Update your `users.auto.tfvars` to ensure `login_name` matches IdP usernames:

```hcl
developer_users = {
  "JSMITH" = {
    email        = "jsmith@company.com"
    display_name = "Jane Smith"
    first_name   = "Jane"
    last_name    = "Smith"
    login_name   = "jsmith@company.com"  # Must match IdP username
  }
  "BJONES" = {
    email        = "bjones@company.com"
    display_name = "Bob Jones"
    first_name   = "Bob"
    last_name    = "Jones"
    login_name   = "bjones@company.com"  # Must match IdP username
  }
}
```

### Update the User Module for SSO

Ensure your `snowflake_user` module handles SSO users correctly. The key changes:

1. Set `login_name` explicitly (don't default to `user_name`)
2. Don't set passwords for SSO users
3. Keep `must_change_password = false`

Update `modules/snowflake_user/variables.tf` to add a `login_name` variable if not present:

```hcl
variable "user_login_name" {
  description = "Login name for SSO authentication (usually email). Defaults to user_name if not set."
  type        = string
  default     = null
}
```

Update `modules/snowflake_user/main.tf`:

```hcl
resource "snowflake_user" "this" {
  provider = snowflake.user_admin
  name     = upper(var.user_name)

  # SSO login name - must match IdP username
  login_name = var.user_login_name != null ? var.user_login_name : upper(var.user_name)

  email        = var.user_email
  display_name = var.user_display_name
  first_name   = var.user_first_name
  last_name    = var.user_last_name
  comment      = var.user_comment

  default_warehouse       = var.user_default_warehouse
  default_role            = var.user_default_role
  default_namespace       = var.user_default_namespace
  default_secondary_roles = var.user_default_secondary_roles

  # SSO users: no password management
  # Password is only set for service accounts or fallback access
  must_change_password = false
}
```

### Hybrid Approach: Terraform Users with SSO Auth

This is the recommended approach:

| Managed by Terraform | Managed by SSO |
|---------------------|----------------|
| User creation | Authentication |
| Role assignments | Password/MFA policies |
| Default warehouse/role | Session management |
| User attributes (name, email) | Login credentials |

Benefits of this approach:

- **Version control**: User access is reviewed in PRs
- **Consistency**: All users follow the same pattern
- **Auditability**: Git history shows who was granted access and when
- **No sync issues**: No race conditions between SCIM and Terraform

```hcl
# users.tf - Terraform manages the user lifecycle
module "user_developers" {
  source   = "./modules/snowflake_user"
  for_each = var.developer_users

  providers = {
    snowflake.security_admin = snowflake.security_admin
    snowflake.user_admin     = snowflake.user_admin
  }

  user_name         = each.key
  user_email        = each.value.email
  user_login_name   = each.value.login_name  # For SSO matching
  user_display_name = each.value.display_name
  user_first_name   = each.value.first_name
  user_last_name    = each.value.last_name

  user_default_warehouse = module.warehouse_developer.warehouse_name
  user_default_role      = module.role_analytics_developer.role_name

  grant_roles = [module.role_analytics_developer.role_name]
}
```

### Onboarding New Users

When a new team member joins:

1. **Add to `users.auto.tfvars`** with their IdP email as `login_name`
2. **Create PR** for review
3. **Merge** - CI/CD creates the Snowflake user
4. **Add to IdP group** - user can now authenticate via SSO

When a team member leaves:

1. **Remove from IdP** - immediately blocks authentication
2. **Remove from `users.auto.tfvars`** - creates PR
3. **Merge** - CI/CD removes the Snowflake user

## Service Account Authentication

Service accounts use key pair authentication, not SSO. You already set this up for `SVC_TERRAFORM` in [Add Snowflake to Terraform](../../getting-started/terraform-setup/snowflake/index.md).

### Review: Key Pair Authentication

Service accounts authenticate using RSA key pairs:

1. **Private key**: Stored securely (AWS Secrets Manager, CI/CD secrets)
2. **Public key**: Assigned to the Snowflake user

When you add new service accounts (e.g., `SVC_DBT`, `SVC_AIRBYTE`), follow the same pattern:

```hcl
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

  grant_roles = [module.role_analytics_transformer.role_name]
}
```

### Generate Keys for New Service Accounts

Follow the same process used for `SVC_TERRAFORM`:

```sh
# Generate private key (PKCS8 format, no encryption)
openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out svc_dbt_key.p8 -nocrypt

# Generate public key
openssl rsa -in svc_dbt_key.p8 -pubout -out svc_dbt_key.pub

# Get the public key without headers (for Snowflake)
grep -v "BEGIN\|END" svc_dbt_key.pub | tr -d '\n'
```

### Assign Public Key

Assign the public key to the user in Snowflake. You can either:

**Option 1: SQL command**

```sql
-- As SECURITYADMIN
ALTER USER SVC_DBT SET RSA_PUBLIC_KEY = 'MIIBIjANBgkqh...';
```

**Option 2: Add to Terraform module**

Update the user module to accept an optional public key:

```hcl
variable "user_rsa_public_key" {
  description = "RSA public key for key pair authentication"
  type        = string
  default     = null
}

# In the user resource
resource "snowflake_user" "this" {
  # ... existing config ...
  rsa_public_key = var.user_rsa_public_key
}
```

!!! tip "Key Storage"
    Store private keys in AWS Secrets Manager and retrieve them in CI/CD pipelines. Never commit private keys to version control.

## Verify SSO Configuration

Test the complete setup:

```sql
-- Check SSO integration
SHOW SECURITY INTEGRATIONS;

-- Check a user's authentication methods
DESCRIBE USER JBLOGGS;

-- Check SSO login history
SELECT *
FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE FIRST_AUTHENTICATION_FACTOR = 'SAML'
ORDER BY EVENT_TIMESTAMP DESC
LIMIT 10;
```

## Summary

You've configured enterprise authentication for Snowflake:

- [x] Understood SSO options (SAML, OAuth, key pair)
- [x] Created the SAML2 security integration in Terraform
- [x] Configured your identity provider
- [x] Updated user configuration for SSO authentication
- [x] Understood the hybrid approach (Terraform users + SSO auth)
- [x] Reviewed service account key pair authentication

## What's Next

With authentication sorted, you've completed the core Snowflake infrastructure. The final section wraps up with verification steps and next actions.

Continue to [Finishing Up](10-finishing-up.md) →
