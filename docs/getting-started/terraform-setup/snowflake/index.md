# Add Snowflake to Terraform

In this section, you will:

- [x] Understand the Snowflake provider and what to manage
- [x] Set up the Snowflake backend configuration
- [x] Create a service account for Terraform with key-pair authentication
- [x] Store credentials securely in AWS Secrets Manager
- [x] Import your existing admin user
- [x] Update CI/CD workflows for Snowflake deployment

## The Snowflake Provider

The Snowflake provider allows Terraform to manage Snowflake resources - users, roles, warehouses, databases, and more.

You can find more about the Snowflake provider in the [registry documentation](https://registry.terraform.io/providers/Snowflake-Labs/snowflake/latest/docs), including the full list of supported resources.

!!! info "Provider Versions"
    The Snowflake provider has undergone significant changes. Make sure you're using a recent version (0.87.0 or later) as older versions have different resource names and authentication methods.

## What Should We Manage in Terraform?

For this initial setup, we'll manage:

- **Admin user**: Your existing admin user (the one you created during account setup)
- **Service account**: The `SVC_TERRAFORM` user for Terraform operations

In later sections (when building the data warehouse), you'll add:

- **Warehouses**: Compute resources for queries
- **Databases**: Storage for your data
- **Roles**: RBAC for access control
- **Users**: Team members and additional service accounts
- **Grants**: Permissions connecting roles to resources

For now, we won't manage:

- **Organisation settings**: These require ORGADMIN and are rarely changed
- **Account parameters**: Most defaults are fine for initial setup
- **Network policies**: Add these when you have specific security requirements

!!! note "Incremental Approach"
    We're starting with the minimum needed to get Terraform working with Snowflake. You'll add warehouses, databases, and roles in the [Data Warehouse](../../../build/data-warehouse/) section.

## Authentication Approach

Unlike AWS (which uses OIDC for GitHub Actions), Snowflake requires a service account with credentials. We'll use key-pair authentication because:

- **More secure**: No password to manage or rotate
- **Required for automation**: Key-pair auth works in CI/CD pipelines
- **Industry standard**: Recommended by Snowflake for programmatic access

```
┌─────────────────────────────────────────────────────────────────┐
│                  Snowflake Authentication Flow                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Local Development              CI/CD Pipeline                  │
│  ─────────────────              ────────────────                │
│                                                                 │
│  1. Generate key pair           1. GitHub Actions starts        │
│  2. Store in 1Password          2. Assumes AWS role (OIDC)      │
│  3. Export as env vars          3. Reads from Secrets Manager   │
│  4. Run terraform               4. Run terraform                │
│                                                                 │
│  Both use the same SVC_TERRAFORM account with key-pair auth     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### The SVC_TERRAFORM Service Account

We'll create a dedicated service account for Terraform:

| Attribute | Value |
|-----------|-------|
| **Username** | `SVC_TERRAFORM` |
| **Default role** | `ACCOUNTADMIN` |
| **Authentication** | RSA key-pair (2048-bit minimum) |
| **Password** | None (key-pair only) |

!!! warning "ACCOUNTADMIN for Terraform"
    The service account needs `ACCOUNTADMIN` to create and manage all resource types. This is similar to how the AWS TerraformGitHubActionsRole needs broad permissions.

    In a more mature setup, you could create a custom role with only the permissions Terraform needs, but `ACCOUNTADMIN` is pragmatic for initial setup.

### Credentials Storage

Service account credentials will be stored in AWS Secrets Manager:

| Secret | Contents |
|--------|----------|
| `terraform/snowflake-credentials` | Account identifier, username, private key |

This follows the pattern established in [Secrets Manager Setup](../aws/7-secrets-manager-setup.md) - Terraform creates the secret container, you set the value via CLI.

## Running Terraform Locally

As with AWS, you'll run Terraform locally during initial setup:

1. **Create the service account** manually in Snowflake (needs ACCOUNTADMIN)
2. **Generate key pair** and store in 1Password
3. **Run terraform import** to bring existing resources under management
4. **Configure CI/CD** to use Secrets Manager for credentials

After setup, all Snowflake infrastructure changes go through pull requests.

!!! warning "Local Execution is Temporary"
    Once CI/CD is configured:

    - All Snowflake changes must go through pull requests
    - Local `terraform plan` is fine for debugging
    - Local `terraform apply` should never be run

## File Organisation

For Snowflake resources, we'll use a similar structure to AWS:

```
terraform/snowflake/
├── backend.tf           # S3 backend configuration
├── main.tf              # Terraform and provider versions
├── providers.tf         # Snowflake provider configuration
├── variables.tf         # Variable definitions
├── terraform.tfvars     # Variable values (account, region)
├── outputs.tf           # Output definitions
├── imports.tf           # Import blocks (temporary)
└── users.tf             # User resources (admin, service account)
```

As you add more resources in later sections, you'll add files like:

- `warehouses.tf` - Compute resources
- `databases.tf` - Storage resources
- `roles.tf` - RBAC configuration
- `grants.tf` - Permission assignments

## Prerequisites

Before starting this section, ensure you have:

- [x] A Snowflake account with an admin user ([Snowflake Account Setup](../../account-setup/snowflake.md))
- [x] AWS Terraform setup complete ([AWS Section](../aws/index.md))
- [x] CI/CD workflows configured ([Terraform Deployment](../4-terraform-deployment.md))
- [x] 1Password for storing the service account key pair

## What's Next

Start by [setting up the Snowflake backend](1-setup-snowflake-backend.md) for Terraform state management.
