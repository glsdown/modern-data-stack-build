# Set Up Secrets Manager

On this page, you will:

- [x] Understand the pattern for managing secrets with Terraform
- [x] Import the existing GitHub token secret
- [x] Establish a pattern for adding new secrets

## Navigate to Your Terraform Directory

```sh
cd ~/projects/data/data-stack-infrastructure/terraform/aws
```

## Understanding Secrets in Terraform

AWS Secrets Manager stores sensitive values like API keys, database passwords, and service credentials. When managing secrets with Terraform, you face an important decision: should Terraform manage the secret values, or just the secret containers?

### The Problem with Secret Values in Terraform

If Terraform manages secret values directly, those values appear in:

- **Terraform state**: The state file contains the full secret value in plaintext
- **Plan output**: Secret values may appear in plan diffs
- **Version control**: If anyone commits `terraform.tfvars` with secrets

This creates security risks, especially when state is stored in S3 (even with encryption).

### The Solution: Containers vs Values

The recommended pattern separates concerns:

| Managed by Terraform | Managed by CLI/Console |
|---------------------|------------------------|
| Secret resource (container) | Secret value |
| Secret description | Secret rotation |
| IAM access policies | Initial value setting |
| Tags and metadata | Value updates |

Terraform creates and manages the secret "container" - the AWS resource itself. The actual secret value is set separately using the AWS CLI or Console, keeping it out of Terraform state entirely.

```
┌─────────────────────────────────────────────────────────────────┐
│                     Secret Management                           |
├─────────────────────────────────┬───────────────────────────────┤
│         Terraform               │           AWS CLI             │
├─────────────────────────────────┼───────────────────────────────┤
│ • Creates secret resource       │ • Sets initial secret value   │
│ • Manages resource policies     │ • Updates secret values       │
│ • Configures rotation schedule  │ • Rotates values manually     │
│ • Sets tags and metadata        │ • Retrieves values for use    │
└─────────────────────────────────┴───────────────────────────────┘
```

## Review Existing Secrets

You created the `terraform/github-token` secret in [Terraform Deployment](../4-terraform-deployment.md). Let's check what secrets currently exist:

```sh
aws secretsmanager list-secrets --profile infrastructure-admin
```

Expected output:

```json
{
    "SecretList": [
        {
            "Name": "terraform/github-token",
            "Description": "GitHub PAT for Terraform provider",
            ...
        }
    ]
}
```

## Create Secrets Configuration

Create `secrets.tf`:

```hcl
# =============================================================================
# AWS Secrets Manager
# =============================================================================
#
# This file manages secret containers only. Secret values are set via CLI
# to keep them out of Terraform state.
#
# To set a secret value:
#   aws secretsmanager put-secret-value \
#     --secret-id "secret-name" \
#     --secret-string "value" \
#     --profile infrastructure-admin
# =============================================================================

# -----------------------------------------------------------------------------
# Terraform Secrets - Used by CI/CD pipelines
# -----------------------------------------------------------------------------

resource "aws_secretsmanager_secret" "github_token" {
  name        = "terraform/github-token"
  description = "GitHub PAT for Terraform provider"

  tags = {
    Name        = "terraform/github-token"
    ManagedBy   = "terraform"
    Environment = "all"
  }
}
```

!!! info "Why No Secret Values?"
    Notice that we don't use `aws_secretsmanager_secret_version` resources. This is intentional - secret values are managed outside Terraform to keep them secure.

## Add Import Block for Existing Secret

Add to `imports.tf`:

```hcl
# Secrets Manager
import {
  to = aws_secretsmanager_secret.github_token
  id = "terraform/github-token"
}
```

## Plan and Apply

```sh
terraform plan
```

Review the output. You should see:

```
Plan: 1 to import, 0 to add, 0 to change, 0 to destroy.
```

The existing `terraform/github-token` secret will be imported into Terraform state.

Apply the changes:

```sh
terraform apply
```

## CI/CD Permissions

The `TerraformGitHubActionsRole` already has permission to read secrets with the `terraform/` prefix. The policy document in `oidc.tf` uses a wildcard pattern:

```hcl
statement {
  sid       = "SecretsManagerAccess"
  actions   = ["secretsmanager:GetSecretValue"]
  resources = ["arn:aws:secretsmanager:${var.aws_region}:${var.aws_account_id}:secret:terraform/*"]
}
```

This pattern covers any secret with the `terraform/` prefix. When you add new secrets with this prefix, no policy changes are needed.

## Secret Naming Convention

Organise secrets using a prefix-based naming convention:

| Prefix | Purpose | Example |
|--------|---------|---------|
| `terraform/` | Secrets used by Terraform providers | `terraform/github-token` |
| `applications/` | Secrets used by applications | `applications/api-keys`, `applications/database-urls` |
| `cicd/` | Secrets specific to CI/CD | `cicd/deploy-keys`, `cicd/registry-credentials` |

This structure makes it straightforward to apply IAM policies to groups of secrets. For example, the CI/CD role only needs access to `terraform/*` secrets, not application secrets.

## Adding More Secrets

When you need additional secrets (for example, Snowflake credentials for the Terraform provider), follow this pattern:

1. **Add the resource to `secrets.tf`**:

    ```hcl
    resource "aws_secretsmanager_secret" "snowflake_credentials" {
      name        = "terraform/snowflake-credentials"
      description = "Snowflake service account credentials for Terraform provider"

      tags = {
        Name        = "terraform/snowflake-credentials"
        ManagedBy   = "terraform"
        Environment = "all"
      }
    }
    ```

2. **Apply the Terraform change** (via PR and CI/CD)

3. **Set the secret value via CLI**:

    ```sh
    aws secretsmanager put-secret-value \
      --secret-id "terraform/snowflake-credentials" \
      --secret-string '{"account": "your-account", "user": "TERRAFORM_SVC", "private_key": "..."}' \
      --profile infrastructure-admin
    ```

4. **Update IAM policies** if the secret uses a new prefix

!!! tip "Avoid Shell History"
    To keep secrets out of your shell history, pipe the value from a password manager:

    ```sh
    aws secretsmanager put-secret-value \
      --secret-id "terraform/snowflake-credentials" \
      --secret-string "$(op item get 'Snowflake Terraform' --format json | jq -c '...')" \
      --profile infrastructure-admin
    ```

## Secret Rotation

AWS Secrets Manager supports automatic rotation for certain secret types (RDS credentials, for example). For API tokens like the GitHub PAT, you'll rotate manually:

1. **Generate a new token** in GitHub
2. **Update the secret value**:

    ```sh
    aws secretsmanager put-secret-value \
      --secret-id "terraform/github-token" \
      --secret-string "ghp_new_token_value" \
      --profile infrastructure-admin
    ```

3. **Revoke the old token** in GitHub

!!! tip "Set Rotation Reminders"
    When you create tokens with expiration dates (GitHub recommends 90 days), add reminders to your calendar to rotate before they expire. A failed CI/CD pipeline due to an expired token is disruptive.

## Add Outputs

Update `outputs.tf`:

```hcl
# Secrets Manager outputs
output "github_token_secret_arn" {
  description = "ARN of the GitHub token secret"
  value       = aws_secretsmanager_secret.github_token.arn
}
```

## Verify the Setup

Check that secrets are properly configured:

```sh
# List all secrets
aws secretsmanager list-secrets --profile infrastructure-admin

# Verify Terraform state
terraform state list | grep aws_secretsmanager
```

Expected state output:

```
aws_secretsmanager_secret.github_token
```

## Commit Your Work

```sh
git add terraform/aws/
git commit -m "Add Secrets Manager configuration for Terraform secrets"
```

## Troubleshooting

### Error: Secret Already Exists

If you get an error when creating a secret that already exists:

```
Error: creating Secrets Manager Secret: ResourceExistsException
```

Add an import block instead of letting Terraform create it:

```hcl
import {
  to = aws_secretsmanager_secret.existing_secret
  id = "secret-name"
}
```

### Error: Access Denied When Setting Value

If the CLI command to set a secret value fails with access denied:

1. Verify you're using the correct profile (`--profile infrastructure-admin`)
2. Check the profile has `secretsmanager:PutSecretValue` permission
3. Verify the secret exists (it must be created by Terraform first)

### Secret Value Not Available in CI/CD

If GitHub Actions can't retrieve a secret:

1. Verify the secret name matches exactly (case-sensitive)
2. Check the `TerraformGitHubActionsRole` policy includes the secret ARN pattern
3. Ensure the secret has a value set (empty secrets fail to retrieve)

## Security Considerations

As you add more secrets for data pipelines (`dlt/*`, `airbyte/*`, `prefect/*`), it's important to control who can read and write secret values.

### Separate Read and Write Access

Not everyone who creates secrets needs to read them. Use separate IAM policies:

**Write-only policy** — for administrators who set secret values:

```hcl
resource "aws_iam_policy" "secrets_writer" {
  name        = "SecretsManagerWriter"
  description = "Allow creating and updating secret values"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "WriteSecrets"
        Effect = "Allow"
        Action = [
          "secretsmanager:PutSecretValue",
          "secretsmanager:UpdateSecret",
          "secretsmanager:CreateSecret",
          "secretsmanager:TagResource",
        ]
        Resource = "arn:aws:secretsmanager:${var.aws_region}:${var.aws_account_id}:secret:*"
      }
    ]
  })
}
```

**Read-only policy** — scoped by prefix for services that consume secrets:

```hcl
resource "aws_iam_policy" "dlt_secrets_reader" {
  name        = "DltSecretsReader"
  description = "Allow reading dlt pipeline secrets"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "ReadDltSecrets"
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue",
          "secretsmanager:DescribeSecret",
        ]
        Resource = "arn:aws:secretsmanager:${var.aws_region}:${var.aws_account_id}:secret:dlt/*"
      }
    ]
  })
}
```

### Prefix-Based Access Control

Use secret name prefixes to scope access:

| Prefix | Who Reads | Who Writes |
|--------|-----------|------------|
| `terraform/*` | CI/CD role (`TerraformGitHubActionsRole`) | Admins via CLI |
| `dlt/*` | Prefect worker IAM role | Admins via CLI |
| `airbyte/*` | Prefect worker IAM role, Airbyte ECS task role | Admins via CLI |
| `prefect/*` | CI/CD role, Prefect worker IAM role | Admins via CLI |

This ensures that, for example, the Prefect worker can read `dlt/*` secrets to run pipelines but cannot read `terraform/*` secrets that control infrastructure.

### Resource Policies

For additional protection, attach resource policies to individual secrets to restrict access even if an IAM policy would otherwise allow it:

```hcl
resource "aws_secretsmanager_secret_policy" "restrict_dlt_secrets" {
  secret_arn = aws_secretsmanager_secret.dlt_snowflake_credentials.arn

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "RestrictToWorkerRole"
        Effect    = "Allow"
        Principal = {
          AWS = var.prefect_worker_role_arn
        }
        Action    = "secretsmanager:GetSecretValue"
        Resource  = "*"
      }
    ]
  })
}
```

### Encryption

AWS Secrets Manager encrypts all secrets at rest using AWS KMS. By default, this uses the AWS-managed key (`aws/secretsmanager`). For stricter control, use a customer-managed KMS key:

```hcl
resource "aws_secretsmanager_secret" "sensitive_secret" {
  name       = "dlt/snowflake-credentials"
  kms_key_id = aws_kms_key.secrets_key.arn
}
```

This lets you control who can decrypt secrets independently of who can call the Secrets Manager API.

## What's Next

You've successfully set up Secrets Manager with Terraform:

- [x] Secret containers managed as Infrastructure as Code
- [x] Secret values kept out of Terraform state
- [x] CI/CD permissions configured for secret access
- [x] Pattern established for adding new secrets
- [x] Security best practices for read/write separation

Continue to [finishing up](8-finishing-up.md) →
