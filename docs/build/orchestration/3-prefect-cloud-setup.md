# Prefect Cloud Setup

On this page, you will:

- [x] Add the Prefect API key to AWS Secrets Manager
- [x] Configure the Prefect Terraform provider
- [x] Create work pools via Terraform
- [x] Set up the local development environment

## Prerequisites

Before starting, ensure you have:

- [x] [Prefect Cloud account](../../getting-started/account-setup/prefect.md) with workspace and API key
- [x] Terraform configured with remote state
- [x] AWS Terraform configuration from the [Getting Started](../../getting-started/terraform-setup/aws/index.md) section

## Architecture Overview

With Prefect Cloud, Prefect manages the control plane. You manage:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          PREFECT CLOUD                                      │
│                     (Managed by Prefect)                                    │
│  ┌─────────────────────────────────────────────────────────────────┐        │
│  │  API  │  Database  │  UI  │  Scheduling  │  State Management    │        │
│  └─────────────────────────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ API
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        YOUR INFRASTRUCTURE                                  │
│                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐              │
│  │    Workers      │  │   Flow Code     │  │    Secrets      │              │
│  │  (EC2 / ECS)    │  │  (Git / S3)     │  │  (AWS SM)       │              │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Step 1: Add Prefect Secret to AWS Terraform

First, add the Prefect API key secret container to your **AWS Terraform configuration**. This follows the same pattern as other secrets.

Navigate to your AWS Terraform directory and update `secrets.tf`:

```hcl
# Add to terraform/aws/secrets.tf

# -----------------------------------------------------------------------------
# Prefect API Key
# -----------------------------------------------------------------------------
resource "aws_secretsmanager_secret" "prefect_api_key" {
  name        = "prefect/api-key"
  description = "Prefect Cloud API key for Terraform and CI/CD"

  tags = {
    Name      = "prefect/api-key"
    ManagedBy = "terraform"
  }
}
```

### Update CI/CD Permissions

The CI/CD role needs access to Prefect secrets. Update the IAM policy in `oidc.tf` to include the `prefect/*` prefix:

```hcl
# In terraform/aws/oidc.tf - update the SecretsManagerAccess statement

statement {
  sid     = "SecretsManagerAccess"
  actions = ["secretsmanager:GetSecretValue"]
  resources = [
    "arn:aws:secretsmanager:${var.aws_region}:${var.aws_account_id}:secret:terraform/*",
    "arn:aws:secretsmanager:${var.aws_region}:${var.aws_account_id}:secret:prefect/*"
  ]
}
```

### Deploy and Set the Secret Value

Commit, push, and merge the changes to deploy via CI/CD:

```sh
git add terraform/aws/secrets.tf terraform/aws/oidc.tf
git commit -m "Add Prefect API key secret and CI/CD permissions"
git push
```

After the CI/CD pipeline completes, set the secret value using your API key from [Prefect Cloud account setup](../../getting-started/account-setup/prefect.md):

```sh
aws secretsmanager put-secret-value \
    --secret-id "prefect/api-key" \
    --secret-string "YOUR_API_KEY_HERE" \
    --profile infrastructure-admin
```

!!! tip "Avoid Shell History"
    To keep the API key out of your shell history, pipe from 1Password:

    ```sh
    aws secretsmanager put-secret-value \
        --secret-id "prefect/api-key" \
        --secret-string "$(op item get 'Prefect Cloud' --field 'API Key')" \
        --profile infrastructure-admin
    ```

## Step 2: Configure Local Development

Set up your local environment to work with Prefect.

### Install Prefect

Using `uv` (installed in [Local Environment Setup](../../getting-started/initial-setup/2-local-environment.md)):

```sh
# Create a new project directory
mkdir -p ~/projects/data/data-pipelines
cd ~/projects/data/data-pipelines

# Create virtual environment and install Prefect
uv venv
source .venv/bin/activate
uv pip install prefect prefect-aws prefect-snowflake
```

### Authenticate with Prefect Cloud

```sh
# Login to Prefect Cloud (opens browser)
prefect cloud login

# Or use API key directly
prefect cloud login --key YOUR_API_KEY
```

### Verify Connection and Get IDs

Once logged in, retrieve your account and workspace IDs:

```sh
# Verify connection
prefect version
prefect cloud workspace ls

# Get account and workspace IDs from the API URL
prefect config view --show-sources | grep PREFECT_API_URL
# The URL format is: .../accounts/{account_id}/workspaces/{workspace_id}
```

Store these IDs in 1Password alongside your API key - you'll need them for Terraform configuration.

## Step 3: Create Prefect Terraform Configuration

Create a new Terraform configuration for Prefect Cloud resources.

### Project Structure

```
terraform/
└── prefect/
    └── config/
        ├── backend.tf
        ├── main.tf
        ├── providers.tf
        ├── variables.tf
        └── work_pools.tf
```

### Backend Configuration

Create `terraform/prefect/config/backend.tf`:

```hcl
terraform {
  backend "s3" {
    bucket         = "your-terraform-state-bucket"
    key            = "prefect/terraform.tfstate"
    region         = "eu-west-2"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

### Provider Configuration

Create `terraform/prefect/config/main.tf`:

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    prefect = {
      source  = "PrefectHQ/prefect"
      version = "~> 2.0"
    }
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

Create `terraform/prefect/config/providers.tf`:

```hcl
# -----------------------------------------------------------------------------
# AWS Provider (for Secrets Manager access)
# -----------------------------------------------------------------------------
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = "data-platform"
      ManagedBy   = "terraform"
      Environment = var.environment
    }
  }
}

# -----------------------------------------------------------------------------
# Retrieve Prefect API Key from Secrets Manager
# -----------------------------------------------------------------------------
data "aws_secretsmanager_secret_version" "prefect_api_key" {
  secret_id = "prefect/api-key"
}

# -----------------------------------------------------------------------------
# Prefect Provider
# -----------------------------------------------------------------------------
provider "prefect" {
  api_key      = data.aws_secretsmanager_secret_version.prefect_api_key.secret_string
  account_id   = var.prefect_account_id
  workspace_id = var.prefect_workspace_id
}
```

### Variables

Create `terraform/prefect/config/variables.tf`:

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "eu-west-2"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "production"
}

variable "prefect_account_id" {
  description = "Prefect Cloud account ID"
  type        = string
}

variable "prefect_workspace_id" {
  description = "Prefect Cloud workspace ID"
  type        = string
}
```

!!! warning "Sensitive Values in CI/CD"
    The account and workspace IDs should be set as environment variables in CI/CD, not committed to `terraform.tfvars`. See the CI/CD section below.

### Work Pools

Create `terraform/prefect/config/work_pools.tf`:

```hcl
# -----------------------------------------------------------------------------
# Work Pools
# -----------------------------------------------------------------------------
# Work pools define the infrastructure configuration for flow execution.

# -----------------------------------------------------------------------------
# Development Work Pool (Process-based)
# -----------------------------------------------------------------------------
resource "prefect_work_pool" "development" {
  name        = "development"
  type        = "process"
  description = "Development work pool for local testing"
  paused      = false
}

# -----------------------------------------------------------------------------
# Production Work Pool (ECS-based)
# -----------------------------------------------------------------------------
resource "prefect_work_pool" "production" {
  name        = "production"
  type        = "ecs"
  description = "Production work pool for ECS Fargate execution"
  paused      = false

  base_job_template = jsonencode({
    variables = {
      type = "object"
      properties = {
        cpu = {
          type    = "integer"
          title   = "CPU Units"
          default = 256
        }
        memory = {
          type    = "integer"
          title   = "Memory (MB)"
          default = 512
        }
      }
    }
  })
}

# -----------------------------------------------------------------------------
# Outputs
# -----------------------------------------------------------------------------
output "work_pool_development" {
  description = "Development work pool name"
  value       = prefect_work_pool.development.name
}

output "work_pool_production" {
  description = "Production work pool name"
  value       = prefect_work_pool.production.name
}
```

## Step 4: Set Up CI/CD for Prefect Terraform

Create a GitHub Actions workflow for the Prefect Terraform configuration.

### Add Repository Secrets

In your GitHub repository settings, add these secrets:

| Secret | Description |
|--------|-------------|
| `PREFECT_ACCOUNT_ID` | Your Prefect Cloud account ID |
| `PREFECT_WORKSPACE_ID` | Your Prefect Cloud workspace ID |

These are passed as environment variables to Terraform.

### Create Workflow

Create `.github/workflows/terraform-prefect.yml`:

```yaml
name: Terraform - Prefect

on:
  push:
    branches: [main]
    paths:
      - 'terraform/prefect/**'
  pull_request:
    branches: [main]
    paths:
      - 'terraform/prefect/**'

env:
  AWS_REGION: eu-west-2
  TF_VAR_prefect_account_id: ${{ secrets.PREFECT_ACCOUNT_ID }}
  TF_VAR_prefect_workspace_id: ${{ secrets.PREFECT_WORKSPACE_ID }}

jobs:
  terraform:
    name: Terraform
    runs-on: ubuntu-latest

    permissions:
      id-token: write
      contents: read
      pull-requests: write

    defaults:
      run:
        working-directory: terraform/prefect/config

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.TERRAFORM_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.14.3"

      - name: Terraform Init
        run: terraform init

      - name: Terraform Format Check
        run: terraform fmt -check

      - name: Terraform Plan
        id: plan
        run: terraform plan -no-color -out=tfplan
        continue-on-error: true

      - name: Comment PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const output = `#### Terraform Plan 📖
            \`\`\`
            ${{ steps.plan.outputs.stdout }}
            \`\`\`
            `;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            })

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve tfplan
```

!!! note "Repository Variable"
    Add `TERRAFORM_ROLE_ARN` as a repository variable (not secret) containing your `TerraformGitHubActionsRole` ARN.

### Deploy the Configuration

Commit and push to deploy:

```sh
git add terraform/prefect/ .github/workflows/terraform-prefect.yml
git commit -m "Add Prefect Cloud Terraform configuration"
git push
```

Create a PR to review the plan, then merge to apply.

## Step 5: Verify the Setup

After CI/CD completes, verify the work pools were created:

```sh
# List work pools
prefect work-pool ls
```

Expected output:

```
                            Work Pools
┏━━━━━━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━┓
┃ Name         ┃ Type     ┃ Status   ┃ Description        ┃
┡━━━━━━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━┩
│ development  │ process  │ Ready    │ Development work...│
│ production   │ ecs      │ Ready    │ Production work... │
└──────────────┴──────────┴──────────┴────────────────────┘
```

## Step 6: Start a Development Worker

Start a worker locally to test flow execution:

```sh
# Activate your virtual environment
cd ~/projects/data/data-pipelines
source .venv/bin/activate

# Start a worker for the development pool
prefect worker start --pool development
```

Keep this running in a terminal while developing flows.

## Summary

You've set up Prefect Cloud with Terraform:

- [x] Added Prefect API key to AWS Secrets Manager
- [x] Configured the Prefect Terraform provider
- [x] Created development and production work pools
- [x] Set up CI/CD for Prefect Terraform
- [x] Started a development worker

## What's Next

With work pools configured, you're ready to deploy your first flow.

Continue to [Your First Flow](7-first-flow.md) →

Or, if you need self-hosted options for data sovereignty requirements:

- [Docker Compose Setup](4-self-hosted-docker-compose.md) - Simple self-hosted (~$17/month)
- [ECS Production Setup](5-self-hosted-ecs-production.md) - Production self-hosted (~$67/month)
