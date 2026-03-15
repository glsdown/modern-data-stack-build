# Finishing Up

On this page, you will:

- [x] Remove import blocks
- [x] Verify configuration matches reality
- [x] Update CI/CD workflows for Snowflake deployment
- [x] Create Snowflake README
- [x] Format and push your code

## Remove Import Blocks

Now that all resources are imported, delete the `imports.tf` file (if you haven't already).

Import blocks are only needed once. After successful import, the entire file can be deleted.

### Delete imports.tf

```sh
rm imports.tf
```

Your configuration now consists of:

- `backend.tf` - S3 backend configuration
- `main.tf` - Terraform and provider version requirements
- `providers.tf` - Snowflake provider configuration
- `variables.tf` - Input variable definitions
- `terraform.tfvars` - Configuration values (no secrets!)
- `outputs.tf` - Output definitions
- `users.tf` - User resources (admin, service account)

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

Now you need to update the GitHub Actions workflows to plan and apply Snowflake infrastructure alongside GitHub and AWS resources.

### Update CI Workflow

Edit `.github/workflows/terraform_ci.yml` to add a Snowflake plan job:

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

  plan-snowflake:
    name: plan-snowflake
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

      - name: Get Snowflake credentials from AWS Secrets Manager
        uses: aws-actions/aws-secretsmanager-get-secrets@v2
        with:
          secret-ids: |
            SNOWFLAKE_CREDS, terraform/snowflake-credentials
          parse-json-secrets: true

      - name: Set Snowflake environment variables
        run: |
          echo "TF_VAR_SNOWFLAKE_PRIVATE_KEY<<EOF" >> $GITHUB_ENV
          echo "${{ env.SNOWFLAKE_CREDS_PRIVATE_KEY }}" >> $GITHUB_ENV
          echo "EOF" >> $GITHUB_ENV

      - name: Terraform Plan
        uses: ./.github/actions/terraform_plan_comment
        with:
          prefix: terraform/snowflake
          github_token: ${{ secrets.GITHUB_TOKEN }}
```

!!! info "Snowflake Credentials from Secrets Manager"
    The Snowflake job retrieves credentials from AWS Secrets Manager and sets them as environment variables. The `parse-json-secrets: true` option automatically creates separate environment variables for each key in the JSON secret.

    The secret structure:
    ```json
    {
      "organization_name": "MYORG",
      "account_name": "MYACCOUNT",
      "user": "SVC_TERRAFORM",
      "private_key": "-----BEGIN PRIVATE KEY-----\n..."
    }
    ```

    Becomes:
    - `SNOWFLAKE_CREDS_ORGANIZATION_NAME`
    - `SNOWFLAKE_CREDS_ACCOUNT_NAME`
    - `SNOWFLAKE_CREDS_USER`
    - `SNOWFLAKE_CREDS_PRIVATE_KEY`

    Note: The organization and account names are also in `terraform.tfvars`, so the secret primarily provides the private key for authentication.

### Update Apply Workflow

Edit `.github/workflows/terraform_apply.yml` to add a Snowflake apply job:

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

  apply-snowflake:
    name: apply-snowflake
    runs-on: ubuntu-latest
    concurrency:
      group: terraform-apply-snowflake
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

      - name: Get Snowflake credentials from AWS Secrets Manager
        uses: aws-actions/aws-secretsmanager-get-secrets@v2
        with:
          secret-ids: |
            SNOWFLAKE_CREDS, terraform/snowflake-credentials
          parse-json-secrets: true

      - name: Set Snowflake environment variables
        run: |
          echo "TF_VAR_SNOWFLAKE_PRIVATE_KEY<<EOF" >> $GITHUB_ENV
          echo "${{ env.SNOWFLAKE_CREDS_PRIVATE_KEY }}" >> $GITHUB_ENV
          echo "EOF" >> $GITHUB_ENV

      - name: Terraform Apply
        uses: ./.github/actions/terraform_apply
        with:
          prefix: terraform/snowflake
```

!!! warning "Separate Concurrency Groups"
    Each Terraform configuration uses its own concurrency group (`terraform-apply-github`, `terraform-apply-aws`, `terraform-apply-snowflake`). This allows all three to run in parallel whilst preventing concurrent applies within the same configuration.

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

## Create Snowflake README

Document your Snowflake Terraform configuration.

Create `terraform/snowflake/README.md`:

```markdown
# Snowflake Infrastructure

This directory manages Snowflake account infrastructure via Terraform.

## Resources Managed

### Users
- **Admin user**: Your personal admin account (with ACCOUNTADMIN/ORGADMIN roles)
- **SVC_TERRAFORM**: Service account for Terraform operations

## What We Don't Manage Here

- **Password/MFA**: Admin credentials managed manually, stored in 1Password
- **RSA public keys**: Set via SQL, keys stored in 1Password/Secrets Manager
- **Warehouses/Databases**: Added in the Data Warehouse section
- **Roles**: Added in the Data Warehouse section
- **Grants**: Added in the Data Warehouse section

## File Structure

```
terraform/snowflake/
├── backend.tf          # S3 backend configuration
├── main.tf             # Terraform and provider versions
├── providers.tf        # Snowflake provider configuration
├── variables.tf        # Input variable definitions
├── terraform.tfvars    # Configuration values (no secrets!)
├── users.auto.tfvars   # User configuration
├── outputs.tf          # Output definitions
└── users.tf            # User resources
```

## Authentication

### Local Development
Set the private key as an environment variable:
```sh
export TF_VAR_SNOWFLAKE_PRIVATE_KEY=$(cat ~/.snowflake/svc_terraform_key.p8)
```

Or use 1Password CLI:
```sh
export TF_VAR_SNOWFLAKE_PRIVATE_KEY=$(op read "op://Vault/Snowflake SVC_TERRAFORM/Private Key")
```

### CI/CD
Credentials are stored in AWS Secrets Manager (`terraform/snowflake-credentials`) and retrieved by GitHub Actions during workflow execution.

## Making Changes

### Adding a New User

1. Add the user resource to `users.tf`
2. Run `terraform plan` to preview
3. Create a PR for review
4. After approval, merge and CI/CD will apply
5. Set the user's password/key-pair manually in Snowflake

### Modifying an Existing User

1. Update the user resource in `users.tf`
2. Run `terraform plan` to preview
3. Create a PR for review
4. After approval, merge and CI/CD will apply

Note: Sensitive fields (password, RSA keys) are ignored by Terraform and must be managed manually.

## Future Additions

This configuration is the foundation. The Data Warehouse section will add:

- Warehouses and resource monitors
- Databases for each environment
- Functional roles (ANALYTICS_DEVELOPER, etc.)
- Service accounts for dbt, Airbyte, etc.
- Network policies
- SSO integrations
- Storage integrations for S3
```

## Commit Your Work

```sh
git add .
git commit -m "Complete Snowflake Terraform configuration

- Remove import blocks after successful import
- Update CI/CD workflows for Snowflake deployment
- Add README documentation
- Configure SVC_TERRAFORM for CI/CD access"
```

Push to GitHub:

```sh
git push
```

Create the PR, get the relevant approvals, and merge to `main`.

!!! tip "First CI/CD Run"
    This will be the first time the Snowflake Terraform configuration runs through CI/CD. Watch the workflow runs to verify everything works correctly.

## Verify CI/CD Works

After merging, check the GitHub Actions workflow:

1. Go to your repository's Actions tab
2. Find the "Apply Infrastructure" workflow run
3. Verify the `apply-snowflake` job completed successfully

If it fails, check:

- The Snowflake credentials secret exists and contains valid JSON
- The TerraformGitHubActionsRole has permission to read `terraform/*` secrets
- The private key format is correct (including newlines)

## Understanding What You've Done

You've now brought your Snowflake infrastructure under Terraform management:

### Created Resources
- **SVC_TERRAFORM**: Dedicated service account with key-pair authentication
- **AWS Secret**: Snowflake credentials stored securely in Secrets Manager

### Imported Resources
- **Admin user**: Your existing admin account

### Established Patterns
- Key-pair authentication for service accounts
- Credentials stored in AWS Secrets Manager
- Lifecycle blocks to prevent Terraform from managing sensitive fields
- Import workflow for bringing existing resources under management

### CI/CD Integration
- Plan on PR for all three configurations (GitHub, AWS, Snowflake)
- Apply on merge with separate concurrency groups
- Snowflake credentials retrieved from Secrets Manager

### Key Benefits
- **Version controlled**: Snowflake user management is now in code
- **Auditable**: Changes tracked in Git history with approval requirements
- **Consistent**: CI/CD ensures uniform application of changes
- **Secure**: Private keys never in Terraform state or code
- **Foundation**: Ready for Data Warehouse configuration

## Current File Structure

```
terraform/
├── github/
│   ├── backend.tf
│   ├── main.tf
│   ├── providers.tf
│   ├── variables.tf
│   ├── terraform.tfvars
│   ├── outputs.tf
│   ├── organisation.tf
│   ├── teams.tf
│   ├── users.tf
│   └── README.md
├── aws/
│   ├── backend.tf
│   ├── main.tf
│   ├── providers.tf
│   ├── variables.tf
│   ├── terraform.tfvars
│   ├── iam_users.auto.tfvars
│   ├── outputs.tf
│   ├── iam_roles.tf
│   ├── iam_users.tf
│   ├── oidc.tf
│   ├── state.tf
│   ├── budgets.tf
│   ├── secrets.tf
│   └── README.md
└── snowflake/
    ├── backend.tf
    ├── main.tf
    ├── providers.tf
    ├── variables.tf
    ├── terraform.tfvars
    ├── users.auto.tfvars
    ├── outputs.tf
    ├── users.tf
    └── README.md
```

## What's Next

You've successfully configured Snowflake infrastructure with Terraform:

- [x] Service account created with key-pair authentication
- [x] Admin user imported
- [x] CI/CD workflows updated
- [x] Credentials stored securely in Secrets Manager
- [x] Documentation complete

!!! success "Terraform Setup Complete!"
    You now have a complete Terraform setup managing:

    - **GitHub**: Organisation, teams, users, repository settings
    - **AWS**: IAM roles, users, state infrastructure, budgets, secrets
    - **Snowflake**: Admin user, service account

    All changes go through pull requests with automated planning and applying.

Your next step is to build out your data warehouse in Snowflake. You'll create warehouses, databases, roles, and additional users using the patterns established here.

Continue to [Build Your Data Warehouse](../../../build/data-warehouse/index.md) →
