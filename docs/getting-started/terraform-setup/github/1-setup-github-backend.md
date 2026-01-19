# Add GitHub to Terraform

On this page, you will:

- [x] Set up the GitHub backend
- [x] Set up the GitHub provider

## Navigate to Your Terraform Directory

```sh
cd ~/projects/data/data-stack-infrastructure/terraform/github
```

!!! note "Working Directory"
  All files noted below are inside this directory. You should replace `~/projects/data/data-stack-infrastructure` with the path to your project folder.

### Configure the Remote State Backend

The remote state backend is where the current "state" of the infrastructure as terraform sees it is stored. You set it up as part of the [Terraform Remote State setup steps](../1-set-up-terraform-remote-state.md). We need to configure our GH provider to use it.

Create `backend.tf`:

```hcl
terraform {
  backend "s3" {
    bucket         = "terraform-state-123456789012"  # Replace with your bucket name
    key            = "github/terraform.tfstate"       # GitHub-specific state file
    region         = "eu-west-2"                      # Replace with your region
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

Replace:
- `terraform-state-123456789012` with your S3 bucket name (from page 1)
- `eu-west-2` with your AWS region

!!! warning "Bucket Name Must Match"
    Use the exact bucket name you created in [Set Up Terraform Remote State](../1-set-up-terraform-remote-state.md). You can verify it with:

    ```sh
    aws s3 ls --profile data-engineer | grep terraform-state
    ```

!!! tip "Provider-Specific State Keys"
    Notice the `key` is `github/terraform.tfstate`. When you set up AWS and Snowflake later, you'll use `aws/terraform.tfstate` and `snowflake/terraform.tfstate` respectively. This keeps each provider's state isolated whilst using the same S3 bucket.

### Configure Terraform and Provider Versions

Create `main.tf` to specify Terraform version requirements and the GitHub provider:

```hcl
terraform {
  required_version = ">= 1.14.0"

  required_providers {
    github = {
      source  = "integrations/github"
      version = "~> 6.0"
    }
  }
}
```

This configuration:
- Requires Terraform 1.14.0 or later
- Specifies only the GitHub provider (AWS and Snowflake will have their own `main.tf` files in their respective directories)
- Uses version constraints (`~> 6.0` means >= 6.0 and < 7.0)

!!! tip "Provider Versioning"
    Using `~>` (pessimistic constraint) allows patch and minor updates but prevents major version changes that might break your code. This balances stability with security updates.

!!! note "One Provider Per Directory"
    Each provider directory only declares the provider it needs. The `github/` directory only needs the GitHub provider, `aws/` will only need the AWS provider, etc. This keeps configurations focused and reduces unnecessary provider downloads.

### Create Variables

Create `variables.tf` to define the organisation variable needed by the provider:

```hcl
# GitHub Variables
variable "github_organization" {
  description = "GitHub organisation name"
  type        = string
}
```

This is the only variable needed for now - it tells Terraform which GitHub organisation to manage. We'll add more variables in the next page when we start managing actual resources.

## Create Variable Values

Create `terraform.tfvars` to provide the value for your organisation:

```hcl
# GitHub Configuration
github_organization = "your-org-name" # Replace with your GitHub org slug
```

!!! tip "Variables vs tfvars"
    - `variables.tf` defines what inputs are available (the schema)
    - `terraform.tfvars` provides the actual values (the data)
    - This separation means you can have the same configuration work for different organisations just by changing `terraform.tfvars`

## Configure the GitHub Provider

Now that we've defined the variable, we can configure the GitHub provider to use it.

Create `providers.tf`:

```hcl
# GitHub Provider
# This sets the organisation context for all GitHub resources
provider "github" {
  owner = var.github_organization
}
```

This configuration:
- Sets the organisation context - all GitHub resources (teams, repositories, etc.) will be created in this organisation
- References the `github_organization` variable we just defined (keeps config reusable)
- Authentication will use the `GITHUB_TOKEN` environment variable (we'll set this in the next page)

!!! tip "Provider Organisation Context"
    The `owner` parameter in the provider configuration means you don't need to specify the organisation for each individual resource. All `github_team`, `github_repository`, and other resources automatically belong to this organisation.

!!! warning "GitHub Enterprise"
  If you are working on GitHub Enterprise (self hosted), you'll need to ensure that you also set `base_url` alongside the owner. This is so that terraform knows where to send requests to. The value must end with a slash e.g.

  ```hcl
  # providers.tf
  provider "github" {
    owner = var.github_organization
    base_url = var.github_base_url
  }

  # variables.tf
  variable "github_base_url" {
    description = "Target GitHub base API endpoint"
    type        = string
  }

  # terraform.tfvars
  github_base_url = "https://github.yourcompany.com/api/v3/"
  ```

## Create Outputs

Create `outputs.tf` as a placeholder for future outputs:

```hcl
# Outputs for GitHub resources
# Add outputs here as you create resources in the next page
```

This file is empty for now but provides a place to add outputs as we create resources in the next page.

## Initialise Terraform

Now that everything is configured, initialise Terraform for the GitHub provider:

```sh
cd terraform/github
terraform init
```

Expected output:

```
Initializing the backend...

Successfully configured the backend "s3"! Terraform will automatically
use this backend unless the backend configuration changes.

Initializing provider plugins...
- Finding integrations/github versions matching "~> 6.0"...
- Installing integrations/github v6.0.0...
- Installed integrations/github v6.0.0

Terraform has been successfully initialized!
```

This command:
- Configures the S3 backend with key `github/terraform.tfstate`
- Downloads the GitHub provider
- Creates a `.terraform/` directory with provider plugins
- Creates a `.terraform.lock.hcl` file

### Verify Backend Configuration

Check that Terraform created the state file in S3:

```sh
aws s3 ls s3://terraform-state-123456789012/github/ --profile data-engineer
```

You should see an empty state file:

```
2026-01-17 22:00:00         0 terraform.tfstate
```

!!! tip "Empty State is Normal"
    The state file exists but is empty because we haven't created any resources yet. This confirms the backend is working correctly.

## Commit your work

Make sure to commit your work - remember commit frequently. You need to check you are on the correct branch, which you can do at any time by running `gst` if you haven't set up your command prompt to include the current branch.

## What's Next

You've successfully created and configured your GitHub provider and backend to manage the state.

Continue to [import your organisation settings](2-import-github-organisation.md) →
