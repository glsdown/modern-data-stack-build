# Create the Terraform Repository

On this page, you will:

- [x] Clone your infrastructure repository
- [x] Set up the Terraform directory structure
- [x] Configure the remote state backend
- [x] Set up pre-commit hooks
- [x] Configure .gitignore for Terraform
- [x] Initialise Terraform and verify the setup

## Clone Your Infrastructure Repository

Navigate to your projects directory and clone the `data-stack-infrastructure` repository you created in the [GitHub setup guide](../initial-setup/1-github-setup.md).

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

As we are doing a new piece of work, let's create a new branch:

```sh
git checkout -b feature/terraform-setup
```

## Set Up `.envrc` for Environment Variables

We'll use [direnv](https://direnv.net/) to automatically load environment variables when you enter the project directory. This is much better than manually exporting variables every time. You should have already installed it during [Local Development Environment](../initial-setup/2-local-environment.md#install-direnv).

### Add `.envrc` to `.gitignore`

direnv is a tool that runs script commands whenever you `cd` into a given directory. Those commands are stored in a `.envrc` file.

The `.envrc` file is already excluded by your global `.gitignore` (you set this up in [Local Development Environment](../initial-setup/2-local-environment.md#set-up-a-global-gitignore)). However, if someone else hasn't added it to their own, they could accidentally end up committing it to git. Therefore, it's good practice to also include it in your project `.gitignore` - you can never be too careful.

Add the following line to the project `.gitignore` file (if not already present):

```gitignore
# Secrets
.envrc
```

### Create .envrc File

In your repository root, create a `.envrc` file:

```sh
touch ~/projects/data/data-stack-infrastructure/.envrc
```

As we are using AWS S3 for the backend for all resources, we need to make sure that we are using the correct AWS profile from those we setup in the [AWS account setup](../account-setup/aws.md). Add this to your `.envrc` file:

```sh
# AWS Profile for Terraform Backend
export AWS_PROFILE="data-engineer"
```

It's also useful to include a template for other developers using the repo to use for reference. This will be held under version control, so should only ever contain placeholders.

```sh
touch ~/projects/data/data-stack-infrastructure/.envrc.example
```

Add the templated profile above - it's useful to include details for other developers about the purpose of the variable:

```sh
# AWS Profile for Terraform Backend
# Update this with the profile to use with AWS
# or choose "default" if you don't have profiles configured
export AWS_PROFILE="data-engineer"
```

!!! tip "Using aws-vault"
    If you configured aws-vault in [AWS Account Setup](../account-setup/aws.md), the `AWS_PROFILE` environment variable won't work the same way. Instead, you'll need to run Terraform commands within an aws-vault session:

    ```sh
    aws-vault exec data-engineer -- terraform init
    ```

    Or start a shell session with `aws-vault exec data-engineer` and run commands from there.

#### Allow direnv

```sh
direnv allow .
```

Now, whenever you `cd` into this directory, direnv will automatically load your environment variables. When you leave, it unloads them.


## Create the Directory Structure

Terraform projects benefit from a clear, consistent structure. With multiple providers (GitHub, AWS, Snowflake), we'll organise code by provider whilst keeping shared configuration at the root level.

### Recommended Structure

Create the following directory structure:

```sh
cd ~/projects/data/data-stack-infrastructure
mkdir -p terraform/{github,aws,snowflake,modules}
```

Add `.gitkeep` files into the folders to ensure that the directory structure is committed to GitHub:

```sh
touch github/.gitkeep
touch aws/.gitkeep
touch snowflake/.gitkeep
touch modules/.gitkeep
```

Your repository should now look like:

```
data-stack-infrastructure/
├── .envrc
├── .github/
│   └── CODEOWNERS
├── terraform/
│   ├── github/           # GitHub provider resources
|   │   └── .gitkeep
│   ├── aws/              # AWS provider resources
|   │   └── .gitkeep
│   ├── snowflake/        # Snowflake provider resources
|   │   └── .gitkeep
│   └── modules/          # Reusable Terraform modules (future)
|   │   └── .gitkeep
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

## Format the Code

Format all Terraform files to the standard style:

```sh
terraform fmt -recursive
```

This ensures consistent formatting across all `.tf` files.

## Commit Your Work

Now commit the initial Terraform setup.

```sh
cd ../..  # Back to repository root
git checkout -b feature/terraform-setup
git add .
git commit -m "Initial Terraform setup

- Create provider-specific directory structure
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

Your `terraform/` directory structure:

```sh
tree terraform/ -L 2 -a
```

Expected structure:

```
terraform/
├── github/
│   └── .gitkeep
├── aws/
│   └── .gitkeep
├── snowflake/
│   └── .gitkeep
└── modules/
    └── .gitkeep
```

Repository root should contain:

```
.checkov.yaml             # Checkov configuration
.github/                  # GitHub configuration
.gitignore                # Git ignore rules
.pre-commit-config.yaml   # Pre-commit hooks
.tflint.hcl               # TFLint configuration
LICENSE                   # Repository license
README.md                 # Repository README
terraform/                # Terraform configuration
```

## Troubleshooting

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

## What's Next

You now have a complete Terraform repository structure:

- ✅ Provider-specific directory structure (github/, aws/, snowflake/)
- ✅ Pre-commit hooks configured
- ✅ Code quality checks in place (TFLint, Checkov)
- ✅ Documentation structure ready

Next, you'll add your first resources by configuring the github provider and importing the GitHub organisation settings and teams you created manually into Terraform.

Continue to [Add GitHub to Terraform](github/index.md) →
