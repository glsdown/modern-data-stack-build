# Create the Terraform Repository

On this page, you will:

- [x] Clone your infrastructure repository
- [x] Set up the Terraform directory structure
- [x] Configure the remote state backend
- [x] Set up provider configurations for GitHub, AWS, and Snowflake
- [x] Create variables and outputs structure
- [x] Set up pre-commit hooks
- [x] Configure .gitignore for Terraform
- [x] Initialise Terraform and verify the setup

## Clone Your Infrastructure Repository

Navigate to your projects directory and clone the `data-stack-infrastructure` repository you created in the [GitHub setup guide](../1-github-setup.md).

```sh
cd ~/projects/data  # or wherever you store your projects
git clone git@github.com:YOUR-ORG/data-stack-infrastructure.git
cd data-stack-infrastructure
```

Replace `YOUR-ORG` with your GitHub organisation name.

Verify you're in the correct repository:

```sh
git remote -v
```

Expected output:

```
origin  git@github.com:YOUR-ORG/data-stack-infrastructure.git (fetch)
origin  git@github.com:YOUR-ORG/data-stack-infrastructure.git (push)
```

## Create the Directory Structure

Terraform projects benefit from a clear, consistent structure. With multiple providers (GitHub, AWS, Snowflake), we'll organise code by provider whilst keeping shared configuration at the root level.

### Recommended Structure

Create the following directory structure:

```sh
cd ~/projects/data/data-stack-infrastructure
mkdir -p terraform/{github,aws,snowflake,modules}
```

Your repository should now look like:

```
data-stack-infrastructure/
├── .github/
│   └── CODEOWNERS
├── terraform/
│   ├── github/           # GitHub provider resources
│   ├── aws/              # AWS provider resources
│   ├── snowflake/        # Snowflake provider resources
│   └── modules/          # Reusable Terraform modules (future)
├── .gitignore
├── LICENSE
└── README.md
```

### Why Split by Provider?

This structure provides several benefits:

- **Clarity**: Easy to find resources by provider
- **Independence**: Each provider has its own backend.tf with a unique state key
- **Team workflow**: Different teams can own different providers
- **Blast radius**: Changes to GitHub resources won't affect AWS or Snowflake

All providers use the same S3 bucket and DynamoDB table, but each provider directory has its own `backend.tf` file with a provider-specific state key.

!!! tip "Provider-Specific State Keys"
    Each provider directory will use a different state file key in S3:

    - `github/terraform.tfstate` - GitHub resources
    - `aws/terraform.tfstate` - AWS resources
    - `snowflake/terraform.tfstate` - Snowflake resources

    This isolates changes and reduces the risk of conflicts.

## Configure the Remote State Backend

We'll start with the GitHub provider. Each provider directory will have its own backend configuration pointing to the same S3 bucket but with different state file keys.

```sh
cd terraform/github
```

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
    Use the exact bucket name you created in [Set Up Terraform Remote State](1-set-up-terraform-remote-state.md). You can verify it with:

    ```sh
    aws s3 ls --profile data-engineer | grep terraform-state
    ```

!!! tip "Provider-Specific State Keys"
    Notice the `key` is `github/terraform.tfstate`. When you set up AWS and Snowflake later, you'll use `aws/terraform.tfstate` and `snowflake/terraform.tfstate` respectively. This keeps each provider's state isolated whilst using the same S3 bucket.

## Configure Terraform and Provider Versions

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

## Configure the GitHub Provider

Create `providers.tf` to configure the GitHub provider:

```hcl
# GitHub Provider
provider "github" {
  owner = var.github_organization
}
```

This configuration:
- References a variable for the organisation name (keeps config reusable)
- Authentication will use the `GITHUB_TOKEN` environment variable (we'll set this in the next page)

## Create Variables

Create `variables.tf` to define input variables for GitHub:

```hcl
# GitHub Variables
variable "github_organization" {
  description = "GitHub organization name"
  type        = string
}
```

## Create Variable Values

Create `terraform.tfvars` to provide values for your variables:

```hcl
# GitHub Configuration
github_organization = "your-org-name"  # Replace with your GitHub org
```

!!! warning "Never Commit Secrets"
    This `terraform.tfvars` file doesn't contain secrets (yet), so it's safe to commit. Later, when you add sensitive values like passwords, you'll use a `.tfvars` file that's gitignored or use environment variables instead.

## Create Outputs

Create `outputs.tf` for values you want to display after Terraform runs:

```hcl
# Future outputs will be added here as we create resources
# For example:
# output "github_teams" {
#   description = "List of GitHub teams"
#   value       = { for team in github_team.teams : team.name => team.id }
# }
```

This file is empty for now but provides a place to add outputs as we create resources.

## Update .gitignore for Terraform-Specific Files

When you created the repository in the [GitHub setup guide](../initial-setup/1-github-setup.md), you selected the Terraform `.gitignore` template. This gives you basic Terraform exclusions at the repository root.

However, we need to add a few custom rules to the `.gitignore` to handle our specific setup:

```sh
cd ..  # Back to repository root if you're still in terraform/
```

Add these lines to the existing `.gitignore`:

```gitignore
# Exclude all tfvars files which may contain sensitive data
*.tfvars
*.tfvars.json

# BUT include terraform.tfvars since it only has non-sensitive config
!terraform.tfvars

# Ignore lock files for now (we'll commit them later when stable)
.terraform.lock.hcl
```

!!! tip "Why Ignore .terraform.lock.hcl Initially"
    We're ignoring the lock file initially to avoid conflicts whilst the team is setting up. Once your provider versions are stable, you'll commit the lock file to ensure everyone uses the same provider versions.

!!! note "GitHub's Terraform Template"
    GitHub's Terraform `.gitignore` template already covers the basics like `.terraform/` directories, `*.tfstate` files, crash logs, and override files. We're just adding our specific rules for tfvars files and the lock file.

## Set Up Pre-commit Hooks

Create `.pre-commit-config.yaml` in the repository root (not in `terraform/` directory):

```sh
cd ..  # Back to repository root
```

Create `.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
      - id: check-merge-conflict

  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.88.0
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_docs
        args:
          - --hook-config=--path-to-file=README.md
          - --hook-config=--add-to-existing-file=true
          - --hook-config=--create-file-if-not-exists=true
      - id: terraform_tflint
        args:
          - --args=--config=__GIT_WORKING_DIR__/.tflint.hcl

  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.88.0
    hooks:
      - id: terraform_checkov
        args:
          - --args=--config-file=__GIT_WORKING_DIR__/.checkov.yaml

default_stages: [push]
```

This configuration:
- Runs basic checks (whitespace, YAML syntax, large files)
- Formats Terraform code automatically
- Validates Terraform syntax
- Generates documentation
- Runs security checks with Checkov
- Runs linting with TFLint
- Only runs on `push` (not every commit)

## Create Pre-commit Configuration Files

### TFLint Configuration

Create `.tflint.hcl` in the repository root:

```hcl
config {
  module = true
  force   = false
}

plugin "terraform" {
  enabled = true
  preset  = "recommended"
}

plugin "aws" {
  enabled = true
  version = "0.30.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}
```

### Checkov Configuration

Create `.checkov.yaml` in the repository root:

```yaml
framework:
  - terraform

download-external-modules: true

soft-fail: false

quiet: false

compact: true

skip-check:
  # Add any checks you want to skip here
  # Example: CKV_AWS_18 - S3 bucket logging
```

## Install Pre-commit Hooks

Install the pre-commit hooks for this repository:

```sh
pre-commit install --hook-type pre-push
```

Expected output:

```
pre-commit installed at .git/hooks/pre-push
```

This installs the hooks to run before you push code to the remote repository.

!!! tip "Test Pre-commit"
    You can test pre-commit hooks manually at any time:

    ```sh
    pre-commit run --all-files
    ```

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

## Validate the Configuration

Run Terraform's built-in validation:

```sh
terraform validate
```

Expected output:

```
Success! The configuration is valid.
```

This checks:
- Syntax is correct
- Required variables are defined
- Provider configurations are valid

## Format the Code

Format all Terraform files to the standard style:

```sh
terraform fmt -recursive
```

This ensures consistent formatting across all `.tf` files.

## Create Initial README

Create `terraform/github/README.md` documenting the GitHub configuration:

```markdown
# GitHub Infrastructure

This directory manages GitHub organisation settings, teams, repositories, and permissions via Terraform.

## Prerequisites

- Terraform >= 1.14.0
- AWS CLI configured with appropriate credentials (for S3 backend)
- GitHub personal access token with `repo` and `admin:org` scopes

## Structure

- `backend.tf` - S3 backend configuration for remote state
- `main.tf` - Terraform and GitHub provider version requirements
- `providers.tf` - GitHub provider configuration
- `variables.tf` - Input variable definitions
- `terraform.tfvars` - Variable values (non-sensitive)
- `outputs.tf` - Output definitions

## Usage

### Initialize

\`\`\`sh
cd terraform/github
terraform init
\`\`\`

### Plan Changes

\`\`\`sh
terraform plan
\`\`\`

### Apply Changes (CI/CD only)

Terraform apply should only be run by CI/CD, never locally.

## Managed Resources

Currently managing:
- (None yet - will be added in the next page)

Future resources:
- GitHub organisation settings
- Teams (data-platform-admins, data-engineers, data-analysts)
- Repository configuration
- Team permissions
- Branch protection rules
```

## Commit Your Work

Now commit the initial Terraform setup.

```sh
cd ../..  # Back to repository root
git checkout -b feature/terraform-setup
git add .
git commit -m "Initial Terraform setup with GitHub provider configured

- Create provider-specific directory structure
- Configure S3 backend for GitHub resources
- Set up GitHub provider in terraform/github/
- Add pre-commit hooks for code quality
- Create TFLint and Checkov configurations"
```

Finally push to GitHub, and create the PR.

```sh
git push -u origin feature/terraform-setup
```

!!! tip "Pre-commit Will Run"
    When you push, pre-commit hooks will automatically:

    - Format your Terraform code
    - Validate syntax
    - Generate documentation
    - Run security checks

    If any checks fail, fix the issues and push again.

## Verify Your Setup

Your `terraform/github/` directory should now contain:

```sh
ls -la terraform/github/
```

Expected files:

```
.terraform/           # Provider plugins (gitignored)
backend.tf           # Remote state configuration
main.tf              # Terraform and provider versions
outputs.tf           # Output definitions (empty for now)
providers.tf         # GitHub provider configuration
README.md            # Documentation
terraform.tfvars     # Variable values
variables.tf         # Variable definitions
```

Your `terraform/` directory structure:

```sh
tree terraform/ -L 2 -a
```

Expected structure:

```
terraform/
├── github/
│   ├── .terraform/
│   ├── backend.tf
│   ├── main.tf
│   ├── outputs.tf
│   ├── providers.tf
│   ├── README.md
│   ├── terraform.tfvars
│   └── variables.tf
├── aws/              # Empty for now
├── snowflake/        # Empty for now
└── modules/          # Empty for now
```

Repository root should contain:

```
.checkov.yaml        # Checkov configuration
.github/             # GitHub configuration
.gitignore          # Git ignore rules
.pre-commit-config.yaml  # Pre-commit hooks
.tflint.hcl         # TFLint configuration
LICENSE             # Repository license
README.md           # Repository README
terraform/          # Terraform configuration
```

## Troubleshooting

### Error: Failed to get existing workspaces

If you see:

```
Error: Failed to get existing workspaces: AccessDenied
```

Check your AWS credentials:

```sh
aws sts get-caller-identity --profile data-engineer
```

Ensure you're using the correct profile with permissions for S3 and DynamoDB.

### Error: No valid credential sources found

For the GitHub provider, set your token:

```sh
export GITHUB_TOKEN="ghp_your_token_here"
```

Generate a token at [github.com/settings/tokens](https://github.com/settings/tokens) with `repo` and `admin:org` scopes.

### Pre-commit Hook Failures

If pre-commit hooks fail:

```sh
# See what failed
git push

# Run pre-commit manually to see details
pre-commit run --all-files

# Fix issues, then try again
git add .
git commit --amend --no-edit
git push
```

### Terraform Init Fails

If `terraform init` fails, check:

1. Backend configuration matches your S3 bucket and region
2. AWS credentials are valid
3. S3 bucket and DynamoDB table exist
4. Network connectivity to AWS

## What's Next

You now have a complete Terraform repository structure:

- ✅ Provider-specific directory structure (github/, aws/, snowflake/)
- ✅ Remote state configured in S3 with GitHub-specific state key
- ✅ GitHub provider configured in terraform/github/
- ✅ Pre-commit hooks configured
- ✅ Code quality checks in place (TFLint, Checkov)
- ✅ Documentation structure ready

Next, you'll add your first resources by importing the GitHub organisation and teams you created manually into Terraform.

!!! note "AWS and Snowflake Later"
    We've created the `aws/` and `snowflake/` directories but they're empty for now. You'll configure these providers in later pages after you've learned the import workflow with GitHub first.

Continue to [Add GitHub to Terraform](4-add-github-to-terraform.md) →
