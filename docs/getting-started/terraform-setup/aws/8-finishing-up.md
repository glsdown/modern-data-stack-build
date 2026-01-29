# Finishing Up

On this page, you will:

- [x] Remove import blocks
- [x] Verify configuration matches reality
- [x] Update CI/CD workflows for AWS deployment
- [x] Create AWS README
- [x] Format and push your code

## Remove Import Blocks

Now that all resources are imported, delete the `imports.tf` file.

Import blocks are only needed once. After successful import, the entire file can be deleted.

### Delete imports.tf

```sh
rm imports.tf
```

Your configuration now consists of:

- `backend.tf` - S3 backend configuration
- `main.tf` - Terraform and provider version requirements
- `providers.tf` - AWS provider configuration
- `variables.tf` - Input variable definitions
- `terraform.tfvars` - Configuration values
- `outputs.tf` - Output definitions
- `iam_roles.tf` - IAM role definitions
- `iam_users.tf` - IAM user definitions
- `oidc.tf` - GitHub OIDC provider and TerraformGitHubActionsRole
- `state.tf` - S3 bucket and DynamoDB table for Terraform state
- `budgets.tf` - AWS budget configuration
- `secrets.tf` - Secrets Manager configuration

### Verify No Changes

```sh
terraform plan
```

Expected output:

```
No changes. Your infrastructure matches the configuration.
```

If Terraform shows planned changes, something is misconfigured. Review the diff and adjust your resource definitions to match reality.

## Update CI/CD Workflows

Now you need to update the GitHub Actions workflows to plan and apply AWS infrastructure alongside GitHub resources.

### Update CI Workflow

Edit `.github/workflows/terraform_ci.yml` to add an AWS plan job:

```yaml
name: CI - fmt and plan

on:
  workflow_dispatch:
  pull_request:

jobs:
  check-pre-commit:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up Python
        uses: actions/setup-python@v3

      - name: Run Pre-Commit
        uses: pre-commit/action@v3.0.1

  plan-github:
    name: plan-github
    runs-on: ubuntu-latest
    needs: check-pre-commit
    permissions:
      pull-requests: write
      id-token: write
      contents: read
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: 'arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/TerraformGitHubActionsRole'
          role-session-name: github-actions-terraform
          aws-region: ${{ vars.AWS_REGION }}

      - name: Get secrets from AWS Secrets Manager
        uses: aws-actions/aws-secretsmanager-get-secrets@v2
        with:
          secret-ids: |
            TF_VAR_GITHUB_TOKEN, terraform/github-token
          parse-json-secrets: false

      - name: Terraform Plan
        uses: ./.github/actions/terraform_plan_comment
        with:
          prefix: terraform/github
          github_token: ${{ secrets.GITHUB_TOKEN }}

  plan-aws:
    name: plan-aws
    runs-on: ubuntu-latest
    needs: check-pre-commit
    permissions:
      pull-requests: write
      id-token: write
      contents: read
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: 'arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/TerraformGitHubActionsRole'
          role-session-name: github-actions-terraform
          aws-region: ${{ vars.AWS_REGION }}

      - name: Terraform Plan
        uses: ./.github/actions/terraform_plan_comment
        with:
          prefix: terraform/aws
          github_token: ${{ secrets.GITHUB_TOKEN }}
```

!!! info "No Secrets Manager for AWS Plan"
    The AWS plan job doesn't need to retrieve secrets from Secrets Manager because the AWS provider uses the OIDC credentials directly. Only the GitHub provider needs the PAT from Secrets Manager.

### Update Apply Workflow

Edit `.github/workflows/terraform_apply.yml` to add an AWS apply job:

```yaml
name: Apply Infrastructure

on:
  workflow_dispatch:
  push:
    branches:
      - main

jobs:
  apply-github:
    name: apply-github
    runs-on: ubuntu-latest
    concurrency:
      group: terraform-apply-github
      cancel-in-progress: false
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: 'arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/TerraformGitHubActionsRole'
          role-session-name: github-actions-terraform
          aws-region: ${{ vars.AWS_REGION }}

      - name: Get secrets from AWS Secrets Manager
        uses: aws-actions/aws-secretsmanager-get-secrets@v2
        with:
          secret-ids: |
            TF_VAR_GITHUB_TOKEN, terraform/github-token
          parse-json-secrets: false

      - name: Terraform Apply
        uses: ./.github/actions/terraform_apply
        with:
          prefix: terraform/github

  apply-aws:
    name: apply-aws
    runs-on: ubuntu-latest
    concurrency:
      group: terraform-apply-aws
      cancel-in-progress: false
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: 'arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/TerraformGitHubActionsRole'
          role-session-name: github-actions-terraform
          aws-region: ${{ vars.AWS_REGION }}

      - name: Terraform Apply
        uses: ./.github/actions/terraform_apply
        with:
          prefix: terraform/aws
```

!!! warning "Separate Concurrency Groups"
    Each Terraform configuration uses its own concurrency group (`terraform-apply-github`, `terraform-apply-aws`). This allows GitHub and AWS applies to run in parallel whilst preventing concurrent applies within the same configuration.

## Format and Validate

Format your code:

```sh
terraform fmt -recursive
```

Validate syntax:

```sh
terraform validate
```

Expected output:

```
Success! The configuration is valid.
```

## Create AWS README

Document your AWS Terraform configuration.

Create `terraform/aws/README.md`:

```markdown
# AWS Infrastructure

This directory manages AWS account infrastructure via Terraform.

## Resources Managed

### IAM Roles
- **AdminRole**: Full administrative access for account administrators
- **DataEngineerRole**: Data platform development with state file restrictions
- **InfrastructureAdminRole**: Terraform infrastructure management
- **TerraformGitHubActionsRole**: CI/CD pipeline access via OIDC

### IAM Users
- Personal IAM user with AssumeRolesPolicy
- Pattern for team member management with `for_each`

### OIDC Provider
- GitHub Actions OIDC provider for secure, credential-less authentication

### State Infrastructure
- S3 bucket for Terraform state storage
- DynamoDB table for state locking

### Budgets
- Monthly cost budget with configurable alerts
- Pattern for service-specific budgets

### Secrets Manager
- Secret containers for Terraform provider credentials
- Pattern for managing secrets without exposing values in state

## What We Don't Manage Here

- **Secret values**: Set via CLI to keep out of Terraform state
- **Access keys**: Created by users, not in code
- **MFA devices**: User-specific, require interactive setup
- **Console passwords**: User-managed

## File Structure

```
terraform/aws/
├── backend.tf           # S3 backend configuration
├── main.tf              # Terraform and provider versions
├── providers.tf         # AWS provider configuration
├── variables.tf         # Input variable definitions
├── terraform.tfvars     # Configuration values
├── iam_users.auto.tfvars # IAM user configuration (auto-loaded)
├── outputs.tf           # Output definitions
├── iam_roles.tf         # IAM role definitions and policy documents
├── iam_users.tf         # IAM user definitions
├── oidc.tf              # GitHub OIDC provider and CI/CD role
├── state.tf             # State infrastructure (S3, DynamoDB)
├── budgets.tf           # AWS budget configuration
└── secrets.tf           # Secrets Manager resources
```

## Configuration Approach

### Policy Documents
Policies use `aws_iam_policy_document` data sources:
- Validates at plan time, catching errors early
- References variables and resources directly
- No separate JSON files to manage
- Type-safe Terraform syntax

### Environment Variables
Shared values (like GitHub organisation name) use `TF_VAR_*` environment variables:
- Set in `.envrc` at repository root
- Shared between `terraform/github/` and `terraform/aws/`
- No duplication in multiple `tfvars` files

## Making Changes

### Adding a New IAM Role

1. Add a `data "aws_iam_policy_document"` for the trust policy (if needed)
2. Add a `data "aws_iam_policy_document"` for permissions
3. Add the `aws_iam_role` resource
4. Add the `aws_iam_policy` resource using the policy document
5. Add `aws_iam_role_policy_attachment` to link them
6. Run `terraform plan` to preview
7. Create a PR for review
8. After approval, merge and CI/CD will apply

### Adding a New Secret

1. Add the secret resource to `secrets.tf`
2. Run `terraform apply` (via CI/CD) to create the container
3. Set the secret value via CLI:
   ```sh
   aws secretsmanager put-secret-value \
     --secret-id "terraform/new-secret" \
     --secret-string "value" \
     --profile infrastructure-admin
   ```
4. Update IAM policies if using a new prefix

### Updating Budget Alerts

1. Modify `budgets.tf` or `terraform.tfvars`
2. Run `terraform plan` to preview
3. Create a PR for review
4. After approval, merge and CI/CD will apply

## AWS CLI Profiles

This configuration expects these AWS CLI profiles:

| Profile | Role | Use Case |
|---------|------|----------|
| `data-engineer` | DataEngineerRole | Day-to-day data work |
| `infrastructure-admin` | InfrastructureAdminRole | Local Terraform operations (initial import only) |
| `admin` | AdminRole | Account administration |

After initial setup, all Terraform operations should run through CI/CD, not locally.
```

## Commit Your Work

```sh
git add .
git commit -m "Complete AWS Terraform configuration

- Remove import blocks after successful import
- Update CI/CD workflows for AWS deployment
- Add README documentation
- Configure TerraformGitHubActionsRole for AWS management"
```

Push to GitHub:

```sh
git push
```

Create the PR, get the relevant approvals, and merge to `main`.

!!! tip "First CI/CD Run"
    This will be the first time the AWS Terraform configuration runs through CI/CD. Watch the workflow runs to verify everything works correctly.

## Understanding What You've Done

You've now brought your AWS infrastructure under Terraform management:

### Imported Existing Resources
- IAM roles (AdminRole, DataEngineerRole, InfrastructureAdminRole)
- TerraformGitHubActionsRole and policy
- GitHub OIDC provider
- S3 state bucket and DynamoDB lock table
- IAM users and policies
- Budget alerts
- Secrets Manager secrets

### Established Patterns
- `aws_iam_policy_document` for type-safe, validated policies
- Secrets management without exposing values in state
- Environment variables for shared configuration
- Import workflow for bringing existing resources under management

### CI/CD Integration
- Plan on PR for both GitHub and AWS configurations
- Apply on merge with separate concurrency groups
- OIDC authentication eliminating long-lived credentials

### Key Benefits
- **Version controlled**: All AWS infrastructure is now in code
- **Auditable**: Changes tracked in Git history with approval requirements
- **Consistent**: CI/CD ensures uniform application of changes
- **Secure**: Secrets values never in Terraform state
- **Documented**: Configuration serves as documentation
- **Scalable**: Patterns established for adding resources

## What's Next

You've successfully configured AWS infrastructure with Terraform:

- [x] IAM roles managed in code
- [x] State infrastructure managed in code
- [x] Budget alerts managed in code
- [x] Secrets Manager configured
- [x] CI/CD workflows updated
- [x] Documentation complete

Your next step is to configure Snowflake with Terraform. You'll create a service account for Terraform, import your admin user, and establish patterns for managing Snowflake resources.

Continue to [Snowflake Infrastructure with Terraform](../snowflake/index.md) →
