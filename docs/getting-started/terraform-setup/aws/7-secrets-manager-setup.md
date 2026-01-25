# Set Up Secrets Manager

On this page, you will:

- [x] Understand the pattern for managing secrets with Terraform
- [x] Import the existing GitHub token secret
- [x] Create additional secret containers for Snowflake
- [x] Update IAM policies for CI/CD secret access

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

!!! note "New vs Imported Secrets"
    The `github_token` secret already exists and will be imported. The `snowflake_credentials` secret is new and will be created by Terraform.

## Plan and Apply

```sh
terraform plan
```

Review the output. You should see:

```
Plan: 1 to import, 1 to add, 0 to change, 0 to destroy.
```

- **1 to import**: The existing `terraform/github-token` secret
- **1 to add**: The new `terraform/snowflake-credentials` secret

Apply the changes:

```sh
terraform apply
```

## Set Secret Values

After Terraform creates the secret containers, set the values using the CLI.

### Set Snowflake Credentials

You'll set these credentials after creating the Snowflake service account in the [Snowflake Terraform setup](../../terraform-setup/snowflake/2-create-terraform-service-account.md). For now, you can create a placeholder:

```sh
aws secretsmanager put-secret-value \
  --secret-id "terraform/snowflake-credentials" \
  --secret-string '{"account": "placeholder", "user": "placeholder", "private_key": "placeholder"}' \
  --profile infrastructure-admin
```

!!! warning "Update After Snowflake Setup"
    Remember to update this secret with real credentials after completing the Snowflake service account setup. The placeholder values will cause Terraform to fail when configuring the Snowflake provider.

!!! tip "Avoid Shell History"
    To keep secrets out of your shell history, pipe the value from 1Password or use a heredoc with environment variables:

    ```sh
    aws secretsmanager put-secret-value \
      --secret-id "terraform/snowflake-credentials" \
      --secret-string "$(op item get 'Snowflake Terraform Service Account' --format json | jq -c '{account: .fields[0].value, user: .fields[1].value, private_key: .fields[2].value}')" \
      --profile infrastructure-admin
    ```

## Update CI/CD Permissions

The `TerraformGitHubActionsRole` needs permission to read the new Snowflake credentials secret. Update `policies/permissions/terraform-github-actions.json.tpl`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "TerraformStateAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::terraform-state-${account_id}",
        "arn:aws:s3:::terraform-state-${account_id}/*"
      ]
    },
    {
      "Sid": "TerraformStateLocking",
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:DeleteItem"
      ],
      "Resource": "arn:aws:dynamodb:${region}:${account_id}:table/terraform-state-lock"
    },
    {
      "Sid": "SecretsManagerAccess",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": [
        "arn:aws:secretsmanager:${region}:${account_id}:secret:terraform/*"
      ]
    }
  ]
}
```

The wildcard `terraform/*` in the `SecretsManagerAccess` statement already covers both secrets. If you add secrets with different prefixes, you'll need to update this policy.

## Secret Naming Convention

Organise secrets using a prefix-based naming convention:

| Prefix | Purpose | Example |
|--------|---------|---------|
| `terraform/` | Secrets used by Terraform providers | `terraform/github-token`, `terraform/snowflake-credentials` |
| `applications/` | Secrets used by applications | `applications/api-keys`, `applications/database-urls` |
| `cicd/` | Secrets specific to CI/CD | `cicd/deploy-keys`, `cicd/registry-credentials` |

This structure makes it straightforward to apply IAM policies to groups of secrets. For example, the CI/CD role only needs access to `terraform/*` secrets, not application secrets.

## Adding More Secrets

To add a new secret:

1. **Add the resource to `secrets.tf`**:

    ```hcl
    resource "aws_secretsmanager_secret" "new_secret" {
      name        = "terraform/new-secret"
      description = "Description of the secret"

      tags = {
        Name        = "terraform/new-secret"
        ManagedBy   = "terraform"
        Environment = "all"
      }
    }
    ```

2. **Apply the Terraform change** (via PR and CI/CD)

3. **Set the secret value via CLI**:

    ```sh
    aws secretsmanager put-secret-value \
      --secret-id "terraform/new-secret" \
      --secret-string "your-secret-value" \
      --profile infrastructure-admin
    ```

4. **Update IAM policies** if the secret uses a new prefix

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

output "snowflake_credentials_secret_arn" {
  description = "ARN of the Snowflake credentials secret"
  value       = aws_secretsmanager_secret.snowflake_credentials.arn
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
aws_secretsmanager_secret.snowflake_credentials
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

## What's Next

You've successfully set up Secrets Manager with Terraform:

- [x] Secret containers managed as Infrastructure as Code
- [x] Secret values kept out of Terraform state
- [x] CI/CD permissions configured for secret access
- [x] Pattern established for adding new secrets

Continue to [finishing up](8-finishing-up.md) →
