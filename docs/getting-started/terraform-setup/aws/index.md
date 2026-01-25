# Add AWS to Terraform

In this section, you will:

- [x] Understand the AWS provider and what to manage
- [x] Create an InfrastructureAdminRole for Terraform operations
- [x] Set up the AWS backend and provider configuration
- [x] Import existing IAM roles (AdminRole, DataEngineerRole)
- [x] Import the Terraform state infrastructure (S3 bucket, DynamoDB table)
- [x] Import IAM users and policies
- [x] Import budget alerts
- [x] Configure Secrets Manager for application secrets
- [x] Plan and verify changes

## Running Terraform Locally for Imports

During this section, you'll run Terraform commands locally to import your existing AWS resources. This is a one-time setup process - once CI/CD is fully configured, all Terraform operations should go through GitHub Actions.

!!! warning "Local Execution is Temporary"
    Running `terraform apply` locally is only for the initial import of existing resources. After completing this section:

    - All infrastructure changes must go through pull requests
    - The CI/CD pipeline handles plan and apply operations
    - Local `terraform plan` may be used for debugging, but `terraform apply` should never be run locally

    We'll configure permissions to enforce this at the end of the setup.

## The InfrastructureAdminRole

Before starting, you'll create a new role specifically for Terraform operations. This role - not DataEngineerRole - will have write access to the Terraform state files.

**Why a separate role?**

- **Principle of least privilege**: DataEngineerRole is for day-to-day data work, not infrastructure changes
- **Prevent accidents**: Data engineers can't accidentally run `terraform apply` locally
- **Clear separation**: Infrastructure changes require explicitly assuming the InfrastructureAdminRole
- **Audit trail**: Easy to see who made infrastructure changes vs data changes

After CI/CD is configured:

| Role | State File Access | When to Use |
|------|------------------|-------------|
| DataEngineerRole | Read-only | Day-to-day data platform work |
| InfrastructureAdminRole | Read/Write | Local debugging (rarely needed) |
| TerraformGitHubActionsRole | Read/Write | CI/CD pipeline (primary method) |

!!! note "Post-Setup Security Hardening"
    At the end of this section, we'll update IAM policies to ensure DataEngineerRole cannot write to state files. This prevents accidental local applies whilst still allowing engineers to read state for debugging.

## The AWS Provider

The AWS provider allows Terraform to interact with Amazon Web Services resources.

> The AWS provider is used to interact with the many resources supported by AWS. The provider needs to be configured with the proper credentials before it can be used.

You can find out more about the AWS provider in their [docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs), along with the extensive range of resources that can be managed with it.

## What Should We Manage in Terraform?

For AWS, we'll focus on managing:

- **IAM roles**: AdminRole, DataEngineerRole, TerraformGitHubActionsRole
- **IAM users**: Your personal IAM user and any team members
- **IAM policies**: Role permissions and user policies
- **State infrastructure**: The S3 bucket and DynamoDB table used for Terraform state
- **Budget alerts**: Cost monitoring and notifications
- **Secrets Manager**: Secret containers (not the secret values themselves)
- **OIDC provider**: GitHub Actions authentication

For now, we'll not manage in Terraform:

- **Root account settings**: These require root user access and are rarely changed
- **Secret values**: Only the secret containers are managed; values are set via CLI
- **Billing preferences**: These are account-level settings that rarely change

!!! note "Pragmatic Approach"
    The goal is to manage infrastructure that benefits from version control and code review whilst keeping sensitive operations (like secret values) out of state files. This strikes a balance between infrastructure-as-code benefits and security.

## Why Import Existing Resources?

You've already created several AWS resources manually:

- **AdminRole** and **DataEngineerRole** (from [AWS Account Setup](../../account-setup/aws.md))
- **S3 bucket** and **DynamoDB table** (from [Terraform Remote State](../1-set-up-terraform-remote-state.md))
- **IAM user** with MFA and access keys
- **Budget alerts** for cost monitoring
- **TerraformGitHubActionsRole** and **OIDC provider** (from [Terraform Deployment](../4-terraform-deployment.md))

Rather than recreating these resources (which would require deleting and recreating them), you'll import them into Terraform state. This allows Terraform to manage them going forward without disruption.

## Import vs Create

When you import a resource:

1. **Terraform reads the current configuration** from AWS
2. **You write matching Terraform code** that describes the resource
3. **Terraform adds the resource to its state** file
4. **Future changes** go through the normal plan/apply workflow

This is different from creating new resources where Terraform provisions them from scratch.

!!! warning "Import Requires Matching Configuration"
    When importing, your Terraform code must match the actual resource configuration. If there are differences, Terraform will show a plan to modify the resource. Review carefully before applying.

## Authentication Setup

Before working with the AWS provider, ensure your AWS CLI is configured correctly.

### Verify Your Credentials

For Terraform operations, use the InfrastructureAdminRole which has write access to state files:

```sh
aws sts get-caller-identity --profile infrastructure-admin
```

Expected output showing the InfrastructureAdminRole:

```json
{
    "UserId": "AROAEXAMPLE:aws-cli-session",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/InfrastructureAdminRole/aws-cli-session"
}
```

### Update the .envrc File

In your repository root, update the `.envrc` file to include AWS configuration:

```sh
# AWS Configuration - use infrastructure-admin for Terraform operations
export AWS_PROFILE="infrastructure-admin"
export AWS_REGION="eu-west-2"
```

Verify it's working:

```sh
cd ~/projects/data/data-stack-infrastructure
echo $AWS_PROFILE  # Should show: infrastructure-admin
echo $AWS_REGION   # Should show: eu-west-2
```

Update the `.envrc.example` to document the required variables:

```sh
# AWS Configuration - use infrastructure-admin for Terraform operations
export AWS_PROFILE="infrastructure-admin"
export AWS_REGION="eu-west-2"
```

!!! tip "Using aws-vault"
    If you configured aws-vault in the [AWS Account Setup](../../account-setup/aws.md), you'll need to run Terraform commands within an aws-vault session instead of relying on `AWS_PROFILE`:

    ```sh
    # Option 1: Prefix each command
    aws-vault exec infrastructure-admin -- terraform plan

    # Option 2: Start a shell session
    aws-vault exec infrastructure-admin
    terraform plan
    terraform apply
    exit
    ```

    The `AWS_PROFILE` environment variable in `.envrc` is for users of the standard AWS CLI configuration.

## File Organisation

For AWS resources, we'll organise files by resource type:

```
terraform/aws/
├── backend.tf           # S3 backend configuration
├── main.tf              # Terraform and provider versions
├── providers.tf         # AWS provider configuration
├── variables.tf         # Variable definitions
├── terraform.tfvars     # Variable values
├── outputs.tf           # Output definitions
├── imports.tf           # Import blocks (temporary)
├── iam_roles.tf         # IAM role resources
├── iam_users.tf         # IAM user resources
├── iam_policies.tf      # IAM policy resources
├── state_infrastructure.tf  # S3 bucket and DynamoDB table
├── budgets.tf           # Budget and alert resources
├── secrets_manager.tf   # Secrets Manager resources
└── oidc.tf              # GitHub OIDC provider and role
```

This organisation makes it easy to find and manage resources as your infrastructure grows. IAM resources are grouped by type, whilst infrastructure resources each have their own file.

!!! tip "Alternative Organisation Approaches"
    Some teams prefer different organisation strategies:

    - **Single file** (`aws.tf`): Simple for very small configurations
    - **By resource type** (what we're using): `iam_roles.tf`, `iam_users.tf`, etc.
    - **By domain** (`security.tf`, `storage.tf`): Group resources by function

    Choose what works best for your team. The important thing is consistency.

## What's Next

You've already created the InfrastructureAdminRole in [AWS Account Setup](../../account-setup/aws.md). Now you're ready to [set up the AWS backend](1-setup-aws-backend.md) for Terraform.
