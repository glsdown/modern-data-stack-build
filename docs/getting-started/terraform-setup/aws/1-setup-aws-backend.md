# Add AWS to Terraform

On this page, you will:

- [x] Set up the AWS backend configuration
- [x] Configure the AWS provider
- [x] Initialise Terraform for AWS resources

## Navigate to Your Terraform Directory

First, create the AWS directory if it doesn't exist:

```sh
take ~/projects/data/data-stack-infrastructure/terraform/aws
```

!!! note "Working Directory"
    All files noted below are inside this directory. Replace `~/projects/data/data-stack-infrastructure` with the path to your project folder.

## Configure the Remote State Backend

The remote state backend stores Terraform's view of your AWS infrastructure. You set it up as part of the [Terraform Remote State setup steps](../1-set-up-terraform-remote-state.md). Now configure the AWS provider to use it.

Create `backend.tf`:

```hcl
terraform {
  backend "s3" {
    bucket         = "terraform-state-123456789012"  # Replace with your bucket name
    key            = "aws/terraform.tfstate"         # AWS-specific state file
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
    Notice the `key` is `aws/terraform.tfstate`. This keeps AWS state separate from GitHub (`github/terraform.tfstate`) and Snowflake (`snowflake/terraform.tfstate`) whilst using the same S3 bucket.

## Configure Terraform and Provider Versions

Create `main.tf` to specify Terraform version requirements and the AWS provider:

```hcl
terraform {
  required_version = ">= 1.14.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

This configuration:

- Requires Terraform 1.14.0 or later
- Specifies only the AWS provider (GitHub and Snowflake have their own directories)
- Uses version constraints (`~> 5.0` means >= 5.0 and < 6.0)

!!! tip "Provider Versioning"
    Using `~>` (pessimistic constraint) allows patch and minor updates but prevents major version changes that might break your code. This balances stability with security updates.

## Create Variables

Create `variables.tf` to define the variables needed by the provider:

```hcl
# AWS Variables
variable "aws_region" {
  description = "AWS region for resources"
  type        = string
  default     = "eu-west-2"
}

variable "aws_account_id" {
  description = "AWS account ID"
  type        = string
}
```

## Create Variable Values

Create `terraform.tfvars` to provide your specific values:

```hcl
# AWS Configuration
aws_region     = "eu-west-2"        # Replace with your region
aws_account_id = "123456789012"     # Replace with your account ID
```

Get your AWS account ID:

```sh
aws sts get-caller-identity --query Account --output text --profile infrastructure-admin
```

!!! tip "Variables vs tfvars"
    - `variables.tf` defines what inputs are available (the schema)
    - `terraform.tfvars` provides the actual values (the data)
    - This separation means the same configuration can work across different AWS accounts

## Configure the AWS Provider

Create `providers.tf`:

```hcl
# AWS Provider
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      ManagedBy   = "Terraform"
      Environment = "Production"
      Project     = "DataStack"
    }
  }
}
```

This configuration:

- Sets the region from the variable
- Applies default tags to all resources created by Terraform
- Uses credentials from the `AWS_PROFILE` environment variable (set in `.envrc`)

!!! info "Default Tags"
    Default tags are automatically applied to every resource Terraform creates. This makes it easy to identify Terraform-managed resources and filter costs by project. You can override or add to these tags on individual resources.

!!! warning "Assuming Roles"
    The provider uses your default AWS credentials (from `AWS_PROFILE`). Ensure your `infrastructure-admin` profile is configured correctly in `~/.aws/config` to assume the InfrastructureAdminRole.

!!! tip "Using aws-vault"
    If you're using aws-vault instead of standard AWS CLI credentials, you'll need to run Terraform commands within an aws-vault session:

    ```sh
    aws-vault exec infrastructure-admin -- terraform init
    aws-vault exec infrastructure-admin -- terraform plan
    aws-vault exec infrastructure-admin -- terraform apply
    ```

    Alternatively, start a shell session with `aws-vault exec infrastructure-admin` and run Terraform commands from there.

## Create Outputs

Create `outputs.tf` as a placeholder for future outputs:

```hcl
# Outputs for AWS resources
# Add outputs here as you create resources

output "aws_account_id" {
  description = "AWS account ID"
  value       = var.aws_account_id
}

output "aws_region" {
  description = "AWS region"
  value       = var.aws_region
}
```

## Initialise Terraform

Now that everything is configured, initialise Terraform for the AWS provider:

```sh
cd terraform/aws
terraform init
```

Expected output:

```
Initializing the backend...

Successfully configured the backend "s3"! Terraform will automatically
use this backend unless the backend configuration changes.

Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.x.x...
- Installed hashicorp/aws v5.x.x

Terraform has been successfully initialized!
```

This command:

- Configures the S3 backend with key `aws/terraform.tfstate`
- Downloads the AWS provider
- Creates a `.terraform/` directory with provider plugins
- Creates a `.terraform.lock.hcl` file

## Verify Backend Configuration

Check that Terraform created the state file in S3:

```sh
aws s3 ls s3://terraform-state-123456789012/aws/ --profile infrastructure-admin
```

You should see an empty state file:

```
2026-01-24 10:00:00         0 terraform.tfstate
```

!!! tip "Empty State is Normal"
    The state file exists but is empty because you haven't imported or created any resources yet. This confirms the backend is working correctly.

## Test the Provider

Verify the provider can communicate with AWS:

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

- AWS credentials are working
- The provider is configured correctly
- Terraform can read from and write to the state backend

## Commit Your Work

Make sure to commit your work:

```sh
git add terraform/aws/
git commit -m "Add AWS Terraform provider configuration"
```

## What's Next

You've successfully configured the AWS provider and backend. Before importing resources, you need to understand how the existing TerraformGitHubActionsRole fits in.

Continue to [import your IAM roles](3-import-iam-roles.md) →
