# Set Up Snowflake Backend

On this page, you will:

- [x] Set up the Snowflake backend configuration
- [x] Configure the Snowflake provider
- [x] Initialise Terraform for Snowflake resources

## Navigate to Your Terraform Directory

First, create the Snowflake directory if it doesn't exist:

```sh
take ~/projects/data/data-stack-infrastructure/terraform/snowflake
```

!!! note "Working Directory"
    All files noted below are inside this directory. Replace `~/projects/data/data-stack-infrastructure` with the path to your project folder.

## Configure the Remote State Backend

The remote state backend stores Terraform's view of your Snowflake infrastructure. You set it up as part of the [Terraform Remote State setup steps](../1-set-up-terraform-remote-state.md). Now configure Terraform to use it for Snowflake resources.

Create `backend.tf`:

```hcl
terraform {
  backend "s3" {
    bucket         = "terraform-state-123456789012"  # Replace with your bucket name
    key            = "snowflake/terraform.tfstate"   # Snowflake-specific state file
    region         = "eu-west-2"                     # Replace with your region
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

Replace:

- `terraform-state-123456789012` with your S3 bucket name (from [Remote State Setup](../1-set-up-terraform-remote-state.md))
- `eu-west-2` with your AWS region

!!! warning "Bucket Name Must Match"
    Use the exact bucket name you created in [Set Up Terraform Remote State](../1-set-up-terraform-remote-state.md). You can verify it with:

    ```sh
    aws s3 ls --profile infrastructure-admin | grep terraform-state
    ```

!!! tip "Provider-Specific State Keys"
    Notice the `key` is `snowflake/terraform.tfstate`. This keeps Snowflake state separate from GitHub (`github/terraform.tfstate`) and AWS (`aws/terraform.tfstate`) whilst using the same S3 bucket.

## Configure Terraform and Provider Versions

Create `main.tf` to specify Terraform version requirements and the Snowflake provider:

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

This configuration:

- Requires Terraform 1.14.0 or later
- Specifies the Snowflake provider from Snowflake-Labs (not Hashicorp)
- Uses version constraints (`~> 0.99` means >= 0.99 and < 1.0)

!!! info "Provider Source"
    Unlike AWS (from `hashicorp/aws`), the Snowflake provider is maintained by Snowflake Labs. The source `Snowflake-Labs/snowflake` tells Terraform where to download it from.

!!! warning "Provider Versions"
    The Snowflake provider has undergone significant changes. Version 0.87.0 introduced breaking changes to authentication and resource names. Make sure you're using a recent version (0.99.0 or later is recommended) for the latest features and stability.

## Create Variables

Create `variables.tf` to define the variables needed for Snowflake:

```hcl
# Snowflake Variables
variable "snowflake_organization_name" {
  description = "Snowflake organization name"
  type        = string
}

variable "snowflake_account_name" {
  description = "Snowflake account name"
  type        = string
}
```

## Create Variable Values

Create `terraform.tfvars` to provide your specific values:

```hcl
# Snowflake Configuration
snowflake_organization_name = "MYORG"     # Replace with your organization name
snowflake_account_name      = "MYACCOUNT" # Replace with your account name
```

!!! info "Finding Your Organization and Account Names"
    You can find these values in several ways:

    **From the URL**: Your login URL follows `https://ORG-ACCOUNT.snowflakecomputing.com`

    **From SQL**:
    ```sql
    SELECT CURRENT_ORGANIZATION_NAME();  -- Returns: MYORG
    SELECT CURRENT_ACCOUNT_NAME();       -- Returns: MYACCOUNT
    ```

    **From organization accounts** (if you have ORGADMIN):
    ```sql
    USE ROLE ORGADMIN;
    SHOW ORGANIZATION ACCOUNTS;
    ```

## Configure the Snowflake Providers

Snowflake uses role-based access control (RBAC) where different operations require different roles. Rather than running Terraform with `ACCOUNTADMIN` for everything (which would work but violates least privilege), we configure separate providers for each role:

| Role | Purpose |
|------|---------|
| **ACCOUNTADMIN** | Account-level settings (network policies, storage integrations) |
| **SYSADMIN** | Object management (warehouses, databases, schemas) |
| **SECURITYADMIN** | Access control (grants, privileges) |
| **USERADMIN** | User management (users, roles) |

Create `providers.tf`:

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
  user              = "your_admin_username"  # Your admin user from account setup
  password          = var.SNOWFLAKE_PASSWORD
  role              = "ACCOUNTADMIN"
}

# System Admin - for creating warehouses, databases, schemas
provider "snowflake" {
  alias             = "sys_admin"
  organization_name = var.snowflake_organization_name
  account_name      = var.snowflake_account_name
  user              = "your_admin_username"
  password          = var.SNOWFLAKE_PASSWORD
  role              = "SYSADMIN"
}

# Security Admin - for managing grants and privileges
provider "snowflake" {
  alias             = "security_admin"
  organization_name = var.snowflake_organization_name
  account_name      = var.snowflake_account_name
  user              = "your_admin_username"
  password          = var.SNOWFLAKE_PASSWORD
  role              = "SECURITYADMIN"
}

# User Admin - for creating users and roles
provider "snowflake" {
  alias             = "user_admin"
  organization_name = var.snowflake_organization_name
  account_name      = var.snowflake_account_name
  user              = "your_admin_username"
  password          = var.SNOWFLAKE_PASSWORD
  role              = "USERADMIN"
}
```

!!! warning "Replace Username"
    Replace `your_admin_username` with your actual admin username from Snowflake account setup.

!!! info "Authentication Methods"
    The Snowflake provider supports several authentication methods:

    - **Key-pair authentication** (recommended for automation)
    - Password authentication
    - OAuth
    - Browser-based SSO

    We'll use password authentication temporarily for bootstrapping. In the next section, you'll create a service account with key-pair authentication and update these providers.

## Set Up Authentication

Before you can initialise Terraform, you need to set up authentication. For the initial setup, you'll use your admin user with password authentication.

Add the password variable to `variables.tf`:

```hcl
variable "SNOWFLAKE_PASSWORD" {
  description = "Snowflake password for initial setup (temporary)"
  type        = string
  sensitive   = true
  default     = null
}
```

!!! warning "Temporary Only"
    Password authentication is only for initial bootstrapping. In the next section, you'll:

    1. Create the `SVC_TERRAFORM` service account with key-pair authentication
    2. Update the provider to use key-pair authentication
    3. Remove the password variable

    Never commit passwords to version control. We'll use environment variables.

### Set the Password Environment Variable

Export the password as an environment variable:

```sh
export TF_VAR_SNOWFLAKE_PASSWORD="your-admin-password"
```

!!! tip "1Password CLI"
    If you use 1Password, you can retrieve the password directly:

    ```sh
    export TF_VAR_SNOWFLAKE_PASSWORD=$(op read "op://Vault/Snowflake Admin User/password")
    ```

    This avoids typing or storing the password in your shell history.

## Create Outputs

Create `outputs.tf` as a placeholder for future outputs:

```hcl
# Outputs for Snowflake resources
# Add outputs here as you create resources

output "snowflake_account_identifier" {
  description = "Snowflake account identifier (ORG-ACCOUNT format)"
  value       = "${var.snowflake_organization_name}-${var.snowflake_account_name}"
}

output "snowflake_organization_name" {
  description = "Snowflake organization name"
  value       = var.snowflake_organization_name
}

output "snowflake_account_name" {
  description = "Snowflake account name"
  value       = var.snowflake_account_name
}
```

## Initialise Terraform

Now that everything is configured, initialise Terraform for the Snowflake provider:

```sh
cd terraform/snowflake
terraform init
```

Expected output:

```
Initializing the backend...

Successfully configured the backend "s3"! Terraform will automatically
use this backend unless the backend configuration changes.

Initializing provider plugins...
- Finding snowflake-labs/snowflake versions matching "~> 0.99"...
- Installing snowflake-labs/snowflake v0.99.x...
- Installed snowflake-labs/snowflake v0.99.x

Terraform has been successfully initialized!
```

This command:

- Configures the S3 backend with key `snowflake/terraform.tfstate`
- Downloads the Snowflake provider
- Creates a `.terraform/` directory with provider plugins
- Creates a `.terraform.lock.hcl` file

## Verify Backend Configuration

Check that Terraform created the state file in S3:

```sh
aws s3 ls s3://terraform-state-123456789012/snowflake/ --profile infrastructure-admin
```

You should see an empty state file:

```
2026-01-30 10:00:00         0 terraform.tfstate
```

!!! tip "Empty State is Normal"
    The state file exists but is empty because you haven't imported or created any resources yet. This confirms the backend is working correctly.

## Test the Provider

Verify the provider can communicate with Snowflake:

```sh
terraform plan
```

Expected output:

```
No changes. Your infrastructure matches the configuration.

Terraform has compared your real infrastructure against your configuration
and found no differences, so no changes are needed.
```

This confirms:

- Snowflake credentials are working
- The provider is configured correctly
- Terraform can read from and write to the state backend

!!! warning "Authentication Errors"
    If you see authentication errors, check:

    - Your account identifier format (should be `ORG-ACCOUNT`)
    - Your username and password are correct
    - Your user has the necessary permissions

## Commit Your Work

Commit your progress, but be careful not to commit sensitive information:

```sh
git add terraform/snowflake/
git commit -m "Add Snowflake Terraform provider configuration"
```

!!! warning "Don't Commit Passwords"
    The `terraform.tfvars` file should NOT contain passwords. Authentication should use environment variables. Double-check before committing.

## Current File Structure

You should now have:

```
terraform/snowflake/
├── backend.tf        # S3 backend configuration
├── main.tf           # Terraform and provider versions
├── providers.tf      # Snowflake provider configuration
├── variables.tf      # Variable definitions
├── terraform.tfvars  # Variable values (no secrets!)
└── outputs.tf        # Output definitions
```

## What's Next

You've successfully configured the Snowflake provider and backend. The current authentication uses your admin user with password - this is temporary.

Next, you'll create a dedicated service account for Terraform with key-pair authentication. This is more secure and works in CI/CD pipelines.

Continue to [create the Terraform service account](2-create-terraform-service-account.md) →
