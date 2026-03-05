# Add AWS to Terraform

On this page, you will:

- [x] Import the AdminRole into Terraform
- [x] Import the DataEngineerRole into Terraform
- [x] Import the InfrastructureAdminRole into Terraform
- [x] Import the TerraformGitHubActionsRole and its policy
- [x] Import the OIDC provider for GitHub Actions

## Navigate to Your Terraform Directory

```sh
cd ~/projects/data/data-stack-infrastructure/terraform/aws
```

!!! note "Working Directory"
    All files noted below are inside this directory.

## Understanding the Existing Roles

You've created several IAM roles manually:

| Role | Created In | Purpose |
|------|-----------|---------|
| AdminRole | [AWS Account Setup](../../account-setup/aws.md) | Full administrative access |
| DataEngineerRole | [AWS Account Setup](../../account-setup/aws.md) | Data platform development |
| InfrastructureAdminRole | [AWS Account Setup](../../account-setup/aws.md) | Terraform infrastructure management |
| TerraformGitHubActionsRole | [Terraform Deployment](../4-terraform-deployment.md) | CI/CD pipeline access |

You've also created an OIDC provider for GitHub Actions authentication. We'll import all of these into Terraform.

## Gather Role Information

First, retrieve information about your existing roles:

```sh
# Get AdminRole details
aws iam get-role --role-name AdminRole --profile infrastructure-admin

# Get DataEngineerRole details
aws iam get-role --role-name DataEngineerRole --profile infrastructure-admin

# Get InfrastructureAdminRole details
aws iam get-role --role-name InfrastructureAdminRole --profile infrastructure-admin

# Get TerraformGitHubActionsRole details
aws iam get-role --role-name TerraformGitHubActionsRole --profile infrastructure-admin

# List attached policies for each role
aws iam list-attached-role-policies --role-name AdminRole --profile infrastructure-admin
aws iam list-attached-role-policies --role-name DataEngineerRole --profile infrastructure-admin
aws iam list-attached-role-policies --role-name InfrastructureAdminRole --profile infrastructure-admin
aws iam list-attached-role-policies --role-name TerraformGitHubActionsRole --profile infrastructure-admin
```

Note the ARNs and attached policies - you'll need these for your Terraform configuration.

## Add Variables for Roles

The OIDC trust policy needs your GitHub organisation name. Since this same value is used in the `terraform/github/` configuration, you can share it via environment variables rather than duplicating values in multiple `terraform.tfvars` files.

### Option 1: Environment Variables (Recommended)

Add these to your `.envrc` file in the repository root:

```sh
# Shared Terraform variables
export TF_VAR_github_organization="your-org-name"
export TF_VAR_github_infrastructure_repo="data-stack-infrastructure"
```

Then run `direnv allow` to reload the environment.

!!! tip "Why Environment Variables?"
    Using `TF_VAR_*` environment variables means:

    - **Shared values**: Both `terraform/github/` and `terraform/aws/` pick up the same values
    - **No duplication**: You don't need to maintain the same values in multiple `tfvars` files
    - **Secure**: Sensitive values stay out of version control

Update `variables.tf` to include role-related variables:

```hcl
# GitHub organisation and repository for OIDC
variable "github_organization" {
  description = "GitHub organisation name"
  type        = string
}

variable "github_infrastructure_repo" {
  description = "GitHub repository name for infrastructure"
  type        = string
}
```

### Option 2: terraform.tfvars

If you prefer to keep values in `terraform.tfvars`, add:

```hcl
# GitHub Configuration (for OIDC)
github_organization        = "your-org-name"              # Replace with your GitHub org
github_infrastructure_repo = "data-stack-infrastructure"  # Replace with your repo name
```

!!! note "Variable Naming"
    We use `github_organization` (not `github_org`) to match the variable name used in `terraform/github/`. This makes environment variable sharing seamless.

## Create IAM Roles Configuration

Create `iam_roles.tf` using `aws_iam_policy_document` for all policies. This is the Terraform-native approach - it validates at plan time, references resources directly, and handles JSON formatting automatically.

```hcl
# =============================================================================
# IAM Roles
# =============================================================================

# -----------------------------------------------------------------------------
# Trust Policy Documents
# -----------------------------------------------------------------------------

# Trust policy allowing users in the same account to assume roles
data "aws_iam_policy_document" "assume_role_same_account" {
  statement {
    actions = ["sts:AssumeRole"]

    principals {
      type        = "AWS"
      identifiers = ["arn:aws:iam::${var.aws_account_id}:root"]
    }
  }
}

# -----------------------------------------------------------------------------
# AdminRole - Full administrative access
# -----------------------------------------------------------------------------
resource "aws_iam_role" "admin" {
  name        = "AdminRole"
  description = "Full administrative access for account administrators"

  assume_role_policy = data.aws_iam_policy_document.assume_role_same_account.json

  tags = {
    Name = "AdminRole"
  }
}

resource "aws_iam_role_policy_attachment" "admin_administrator_access" {
  role       = aws_iam_role.admin.name
  policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"
}

# -----------------------------------------------------------------------------
# DataEngineerRole - Data platform development permissions
# -----------------------------------------------------------------------------
resource "aws_iam_role" "data_engineer" {
  name        = "DataEngineerRole"
  description = "Permissions for data platform development and operations"

  assume_role_policy = data.aws_iam_policy_document.assume_role_same_account.json

  tags = {
    Name = "DataEngineerRole"
  }
}

# S3 policy with state file write restrictions
data "aws_iam_policy_document" "data_engineer_s3" {
  statement {
    sid       = "S3FullAccessExceptState"
    actions   = ["s3:*"]
    resources = ["*"]
  }

  statement {
    sid    = "DenyTerraformStateWrite"
    effect = "Deny"
    actions = [
      "s3:PutObject",
      "s3:DeleteObject"
    ]
    resources = ["arn:aws:s3:::terraform-state-${var.aws_account_id}/*"]
  }
}

resource "aws_iam_policy" "data_engineer_s3" {
  name        = "DataEngineerS3Policy"
  description = "S3 access for data engineers with state file write restrictions"
  policy      = data.aws_iam_policy_document.data_engineer_s3.json
}

resource "aws_iam_role_policy_attachment" "data_engineer_s3" {
  role       = aws_iam_role.data_engineer.name
  policy_arn = aws_iam_policy.data_engineer_s3.arn
}

# DynamoDB policy with state lock restrictions
data "aws_iam_policy_document" "data_engineer_dynamodb" {
  statement {
    sid       = "DynamoDBFullAccess"
    actions   = ["dynamodb:*"]
    resources = ["*"]
  }

  statement {
    sid    = "DenyTerraformStateLockWrite"
    effect = "Deny"
    actions = [
      "dynamodb:PutItem",
      "dynamodb:DeleteItem",
      "dynamodb:UpdateItem"
    ]
    resources = ["arn:aws:dynamodb:${var.aws_region}:${var.aws_account_id}:table/terraform-state-lock"]
  }
}

resource "aws_iam_policy" "data_engineer_dynamodb" {
  name        = "DataEngineerDynamoDBPolicy"
  description = "DynamoDB access for data engineers with state lock restrictions"
  policy      = data.aws_iam_policy_document.data_engineer_dynamodb.json
}

resource "aws_iam_role_policy_attachment" "data_engineer_dynamodb" {
  role       = aws_iam_role.data_engineer.name
  policy_arn = aws_iam_policy.data_engineer_dynamodb.arn
}

# AWS managed policies for DataEngineerRole
resource "aws_iam_role_policy_attachment" "data_engineer_secrets_manager" {
  role       = aws_iam_role.data_engineer.name
  policy_arn = "arn:aws:iam::aws:policy/SecretsManagerReadWrite"
}

resource "aws_iam_role_policy_attachment" "data_engineer_cloudwatch" {
  role       = aws_iam_role.data_engineer.name
  policy_arn = "arn:aws:iam::aws:policy/CloudWatchFullAccess"
}

resource "aws_iam_role_policy_attachment" "data_engineer_iam_readonly" {
  role       = aws_iam_role.data_engineer.name
  policy_arn = "arn:aws:iam::aws:policy/IAMReadOnlyAccess"
}

# -----------------------------------------------------------------------------
# InfrastructureAdminRole - Terraform operations
# -----------------------------------------------------------------------------
resource "aws_iam_role" "infrastructure_admin" {
  name        = "InfrastructureAdminRole"
  description = "Role for Terraform infrastructure management"

  assume_role_policy = data.aws_iam_policy_document.assume_role_same_account.json

  tags = {
    Name = "InfrastructureAdminRole"
  }
}

resource "aws_iam_role_policy_attachment" "infrastructure_admin" {
  role       = aws_iam_role.infrastructure_admin.name
  policy_arn = aws_iam_policy.infrastructure_admin.arn
}
```

## Create OIDC Provider Configuration

Create `oidc.tf`:

```hcl
# =============================================================================
# GitHub OIDC Provider and Role
# =============================================================================

# -----------------------------------------------------------------------------
# OIDC Identity Provider for GitHub Actions
# -----------------------------------------------------------------------------
resource "aws_iam_openid_connect_provider" "github" {
  url             = "https://token.actions.githubusercontent.com"
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]

  tags = {
    Name = "GitHubActions"
  }
}

# -----------------------------------------------------------------------------
# Trust policy for GitHub Actions OIDC
# -----------------------------------------------------------------------------
data "aws_iam_policy_document" "github_oidc_trust" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]

    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github.arn]
    }

    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }

    condition {
      test     = "StringLike"
      variable = "token.actions.githubusercontent.com:sub"
      values   = ["repo:${var.github_organization}/${var.github_infrastructure_repo}:*"]
    }
  }
}

# -----------------------------------------------------------------------------
# TerraformGitHubActionsRole - Role for CI/CD pipelines
# -----------------------------------------------------------------------------
resource "aws_iam_role" "terraform_github_actions" {
  name        = "TerraformGitHubActionsRole"
  description = "Role for GitHub Actions to run Terraform"

  assume_role_policy = data.aws_iam_policy_document.github_oidc_trust.json

  tags = {
    Name = "TerraformGitHubActionsRole"
  }
}

# -----------------------------------------------------------------------------
# Policy for Terraform GitHub Actions Role
# -----------------------------------------------------------------------------
data "aws_iam_policy_document" "terraform_github_actions" {
  # State management
  statement {
    sid = "TerraformStateAccess"
    actions = [
      "s3:GetObject",
      "s3:PutObject",
      "s3:DeleteObject",
      "s3:ListBucket"
    ]
    resources = [
      "arn:aws:s3:::terraform-state-${var.aws_account_id}",
      "arn:aws:s3:::terraform-state-${var.aws_account_id}/*"
    ]
  }

  statement {
    sid = "TerraformStateLocking"
    actions = [
      "dynamodb:GetItem",
      "dynamodb:PutItem",
      "dynamodb:DeleteItem"
    ]
    resources = ["arn:aws:dynamodb:${var.aws_region}:${var.aws_account_id}:table/terraform-state-lock"]
  }

  # Secrets Manager - read for provider credentials, manage for terraform/* secrets
  statement {
    sid       = "SecretsManagerAccess"
    actions   = ["secretsmanager:GetSecretValue"]
    resources = ["arn:aws:secretsmanager:${var.aws_region}:${var.aws_account_id}:secret:terraform/*"]
  }

  statement {
    sid = "SecretsManagerManagement"
    actions = [
      "secretsmanager:DescribeSecret",
      "secretsmanager:ListSecrets",
      "secretsmanager:TagResource",
      "secretsmanager:CreateSecret"
    ]
    resources = ["arn:aws:secretsmanager:${var.aws_region}:${var.aws_account_id}:secret:terraform/*"]
  }

  # IAM - read access for planning, tag access for applying
  statement {
    sid = "IAMReadAccess"
    actions = [
      "iam:GetRole",
      "iam:GetPolicy",
      "iam:GetPolicyVersion",
      "iam:ListRolePolicies",
      "iam:ListAttachedRolePolicies",
      "iam:GetUser",
      "iam:ListUserPolicies",
      "iam:ListAttachedUserPolicies",
      "iam:ListOpenIDConnectProviders",
      "iam:GetOpenIDConnectProvider",
      "iam:TagRole",
      "iam:TagPolicy",
      "iam:TagUser",
      "iam:TagOpenIDConnectProvider"
    ]
    resources = ["*"]
  }

  # Budgets - for managing AWS budget alerts
  statement {
    sid = "BudgetsManagement"
    actions = [
      "budgets:ViewBudget",
      "budgets:ModifyBudget",
      "budgets:DescribeBudget",
      "budgets:DescribeBudgets"
    ]
    resources = ["*"]
  }
}

resource "aws_iam_policy" "terraform_github_actions" {
  name        = "TerraformGitHubActionsPolicy"
  description = "Permissions for Terraform GitHub Actions"
  policy      = data.aws_iam_policy_document.terraform_github_actions.json

  tags = {
    Name = "TerraformGitHubActionsPolicy"
  }
}

resource "aws_iam_role_policy_attachment" "terraform_github_actions" {
  role       = aws_iam_role.terraform_github_actions.name
  policy_arn = aws_iam_policy.terraform_github_actions.arn
}
```

!!! info "CI/CD Permissions for AWS Management"
    This policy includes permissions for IAM, Budgets, and Secrets Manager so that GitHub Actions can run `terraform plan` and `terraform apply` on the AWS configuration. Without these, CI/CD would fail when you add AWS plan/apply jobs to the workflows.

    The policy follows least privilege - read access for most resources, with limited write access for tags and secrets management.

## Create Infrastructure Admin Policy

Add the policy document and resource to `iam_roles.tf` (before the InfrastructureAdminRole attachment):

```hcl
data "aws_iam_policy_document" "infrastructure_admin" {
  statement {
    sid = "TerraformStateFullAccess"
    actions = [
      "s3:GetObject",
      "s3:PutObject",
      "s3:DeleteObject",
      "s3:ListBucket"
    ]
    resources = [
      "arn:aws:s3:::terraform-state-${var.aws_account_id}",
      "arn:aws:s3:::terraform-state-${var.aws_account_id}/*"
    ]
  }

  statement {
    sid = "TerraformStateLocking"
    actions = [
      "dynamodb:GetItem",
      "dynamodb:PutItem",
      "dynamodb:DeleteItem"
    ]
    resources = ["arn:aws:dynamodb:${var.aws_region}:${var.aws_account_id}:table/terraform-state-lock"]
  }

  statement {
    sid       = "IAMFullAccess"
    actions   = ["iam:*"]
    resources = ["*"]
  }

  statement {
    sid       = "S3FullAccess"
    actions   = ["s3:*"]
    resources = ["*"]
  }

  statement {
    sid       = "DynamoDBFullAccess"
    actions   = ["dynamodb:*"]
    resources = ["*"]
  }

  statement {
    sid       = "SecretsManagerFullAccess"
    actions   = ["secretsmanager:*"]
    resources = ["*"]
  }

  statement {
    sid       = "BudgetsFullAccess"
    actions   = ["budgets:*"]
    resources = ["*"]
  }

  statement {
    sid       = "CloudWatchFullAccess"
    actions   = ["cloudwatch:*"]
    resources = ["*"]
  }
}

resource "aws_iam_policy" "infrastructure_admin" {
  name        = "InfrastructureAdminPolicy"
  description = "Permissions for infrastructure management via Terraform"
  policy      = data.aws_iam_policy_document.infrastructure_admin.json

  tags = {
    Name = "InfrastructureAdminPolicy"
  }
}
```

!!! tip "Benefits of aws_iam_policy_document"
    - **Validation**: Errors caught at plan time, not apply time
    - **Direct references**: Use variables and resource attributes directly
    - **No separate files**: Everything in one place, easier to review
    - **Type safety**: Terraform validates the structure
    - **Reusability**: Policy documents can be combined with `source_policy_documents`

## Create Import Configuration

Create `imports.tf` with import blocks for all existing resources:

```hcl
# =============================================================================
# Import Blocks - Remove after successful import
# =============================================================================

# IAM Roles
import {
  to = aws_iam_role.admin
  id = "AdminRole"
}

import {
  to = aws_iam_role.data_engineer
  id = "DataEngineerRole"
}

import {
  to = aws_iam_role.terraform_github_actions
  id = "TerraformGitHubActionsRole"
}

import {
  to = aws_iam_role.infrastructure_admin
  id = "InfrastructureAdminRole"
}

# IAM Role Policy Attachments - AdminRole
import {
  to = aws_iam_role_policy_attachment.admin_administrator_access
  id = "AdminRole/arn:aws:iam::aws:policy/AdministratorAccess"
}

# IAM Role Policy Attachments - DataEngineerRole
import {
  to = aws_iam_role_policy_attachment.data_engineer_s3
  id = "DataEngineerRole/arn:aws:iam::aws:policy/AmazonS3FullAccess"
}

import {
  to = aws_iam_role_policy_attachment.data_engineer_dynamodb
  id = "DataEngineerRole/arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess"
}

import {
  to = aws_iam_role_policy_attachment.data_engineer_secrets_manager
  id = "DataEngineerRole/arn:aws:iam::aws:policy/SecretsManagerReadWrite"
}

import {
  to = aws_iam_role_policy_attachment.data_engineer_cloudwatch
  id = "DataEngineerRole/arn:aws:iam::aws:policy/CloudWatchFullAccess"
}

import {
  to = aws_iam_role_policy_attachment.data_engineer_iam_readonly
  id = "DataEngineerRole/arn:aws:iam::aws:policy/IAMReadOnlyAccess"
}

# OIDC Provider - Use the provider ARN
# Get this with: aws iam list-open-id-connect-providers --profile infrastructure-admin
import {
  to = aws_iam_openid_connect_provider.github
  id = "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"  # Replace with your ARN
}

# TerraformGitHubActionsPolicy
import {
  to = aws_iam_policy.terraform_github_actions
  id = "arn:aws:iam::123456789012:policy/TerraformGitHubActionsPolicy"  # Replace with your ARN
}

# TerraformGitHubActionsRole policy attachment
import {
  to = aws_iam_role_policy_attachment.terraform_github_actions
  id = "TerraformGitHubActionsRole/arn:aws:iam::123456789012:policy/TerraformGitHubActionsPolicy"  # Replace with your ARN
}

# InfrastructureAdminRole policy and attachment
# Note: These are new resources - they won't exist yet if you created InfrastructureAdminRole
# with managed policies. The apply will create these new custom policies.
```

!!! warning "Update the ARNs"
    Replace `123456789012` with your actual AWS account ID in the import blocks.

## Get OIDC Provider ARN

You need the OIDC provider ARN for the import block:

```sh
aws iam list-open-id-connect-providers --profile infrastructure-admin
```

Example output:

```json
{
    "OpenIDConnectProviderList": [
        {
            "Arn": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
        }
    ]
}
```

Update the import block with this ARN.

## Plan the Import

```sh
terraform plan
```

Review the output carefully. You should see resources marked for import:

```
Plan: 11 to import, 0 to add, 0 to change, 0 to destroy.
```

If you see changes (indicated by `~` symbols), your Terraform configuration doesn't match the actual resource. Common differences:

- **Trust policy differences**: Check the `assume_role_policy` matches exactly
- **Tag differences**: Add any missing tags to your configuration
- **Description differences**: Update descriptions to match

## Apply the Import

!!! warning "Importing Locally"
  For these initial imports, we are running terraform locally with the infrastructure admin role. This is not something you should do once the initial setup has been complete. All changes should only be applied via the defined GitHub action.

```sh
terraform apply
```

Type `yes` when prompted.

Expected output:

```
aws_iam_role.admin: Importing... [id=AdminRole]
aws_iam_role.admin: Import complete [id=AdminRole]
aws_iam_role.data_engineer: Importing... [id=DataEngineerRole]
aws_iam_role.data_engineer: Import complete [id=DataEngineerRole]
...

Apply complete! Resources: 11 imported, 0 added, 0 changed, 0 destroyed.
```

## Verify the Import

Check that the roles are now in Terraform state:

```sh
terraform state list | grep aws_iam_role
```

Expected output:

```
aws_iam_role.admin
aws_iam_role.data_engineer
aws_iam_role.infrastructure_admin
aws_iam_role.terraform_github_actions
```

## Add Outputs

Update `outputs.tf` to export role ARNs:

```hcl
# Role ARNs
output "admin_role_arn" {
  description = "ARN of the AdminRole"
  value       = aws_iam_role.admin.arn
}

output "data_engineer_role_arn" {
  description = "ARN of the DataEngineerRole"
  value       = aws_iam_role.data_engineer.arn
}

output "infrastructure_admin_role_arn" {
  description = "ARN of the InfrastructureAdminRole"
  value       = aws_iam_role.infrastructure_admin.arn
}

output "terraform_github_actions_role_arn" {
  description = "ARN of the TerraformGitHubActionsRole"
  value       = aws_iam_role.terraform_github_actions.arn
}

output "github_oidc_provider_arn" {
  description = "ARN of the GitHub OIDC provider"
  value       = aws_iam_openid_connect_provider.github.arn
}
```

## Commit Your Work

```sh
git add terraform/aws/
git commit -m "Import IAM roles and OIDC provider into Terraform"
```

## Troubleshooting

### Error: Resource already managed

If you see:

```
Error: Resource already managed by Terraform
```

The resource is already in your state file. Remove the import block for that resource.

### Error: Cannot import non-existent resource

If you see:

```
Error: Cannot import non-existent remote object
```

Check:

1. Role name is correct (case-sensitive)
2. You're authenticated to the correct AWS account
3. Your profile has permission to read IAM resources

### Trust Policy Mismatch

If the plan shows changes to the trust policy, compare your configuration with the actual policy:

```sh
aws iam get-role --role-name RoleName --query 'Role.AssumeRolePolicyDocument' --profile infrastructure-admin
```

Update your Terraform configuration to match exactly.

## What's Next

You've successfully imported IAM roles into Terraform:

- [x] AdminRole managed in code
- [x] DataEngineerRole managed in code
- [x] InfrastructureAdminRole managed in code
- [x] TerraformGitHubActionsRole managed in code
- [x] GitHub OIDC provider managed in code

Continue to [import state infrastructure](3-import-state-infrastructure.md) →
