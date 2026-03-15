# Terraform Deployment with CI/CD

On this page, you will:

- [x] Understand why Terraform should only run through CI/CD
- [x] Set up GitHub Actions workflows for plan and apply
- [x] Configure OIDC authentication between GitHub and AWS
- [x] Integrate AWS Secrets Manager for secure credential management
- [x] Lock down your Terraform repository to enforce CD-only deployment

## Why CI/CD for Terraform?

Running Terraform locally works for learning and development, but production infrastructure requires a more controlled approach. When multiple team members run Terraform from their laptops, you encounter several problems:

- **State conflicts**: Two people running `terraform apply` simultaneously can corrupt state
- **Inconsistent environments**: Different Terraform versions, provider versions, or credentials
- **No audit trail**: No record of who changed what and when
- **Credential sprawl**: Every developer needs access to production credentials

By moving Terraform execution to CI/CD, you centralise infrastructure changes through a single, controlled process. Every change goes through code review, runs in a consistent environment, and creates an auditable history.

## The Deployment Model

The workflow follows a standard pull request model:

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Create Branch  │────▶│   Open PR       │────▶│  terraform plan │
│  Make Changes   │     │   (triggers CI) │     │  (comment on PR)│
└─────────────────┘     └─────────────────┘     └─────────────────┘
                                                        │
                                                        ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Infrastructure │◀────│ terraform apply │◀────│   Merge to main │
│    Updated      │     │  (CD pipeline)  │     │   (approved PR) │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

1. **Developer creates a branch** and makes Terraform changes
2. **Pull request triggers CI** which runs `terraform plan`
3. **Plan output appears as a PR comment** for review
4. **After approval and merge**, CD runs `terraform apply`
5. **Infrastructure is updated** with full audit trail

!!! info "Plan on PR, Apply on Merge"
    This pattern ensures that every infrastructure change is reviewed before application. The plan output shows exactly what will change, and the apply only runs after human approval.

## Setting Up GitHub Actions

You'll create two workflows: one for planning (CI) and one for applying (CD).

### Repository Structure

We will be adding the following structure to your Terraform repository:

```
.github/
├── actions/
│   ├── terraform_plan_comment/
│   │   └── action.yml
│   └── terraform_apply/
│       └── action.yml
└── workflows/
    ├── terraform_ci.yml
    └── terraform_apply.yml
```

### Prerequisites: GitHub Personal Access Token

Before setting up the workflows, you need a GitHub Personal Access Token (PAT) for the Terraform GitHub provider. This PAT allows Terraform to manage your GitHub organisation's resources.

!!! tip "Use a Service / Bot Account"
    Rather than using your personal GitHub account, consider creating a dedicated "service" account for your organisation (e.g., `svc)-acme-data`). This provides:

    - **Continuity**: The PAT doesn't break when team members leave
    - **Auditing**: Actions appear as the bot user, not personal accounts
    - **Security**: Limited blast radius if credentials are compromised

    Make this bot account an owner of your organisation, then create the PAT from that account.

    Remember to add the login details to 1Password.

#### Create the PAT

1. Login as the GH service account. Go to **Settings > Developer settings > Personal access tokens > Fine-grained tokens**
2. Click **Generate new token**
3. Set a descriptive name: `Terraform - data-stack-infrastructure`
4. Set expiration (recommend 90 days, with a reminder to rotate)
5. Select your organisation under **Resource owner**
6. Under **Repository access**, select **All repositories** (or specific repos)
7. Under **Permissions**, grant:
   - **Repository permissions**: Administration (Read and write), Contents (Read and write), Metadata (Read)
   - **Organization permissions**: Members (Read and write), Administration (Read and write)
8. Click **Generate token** and copy the value - store in 1Password.

#### Store the PAT in AWS Secrets Manager

Rather than storing the PAT in GitHub Secrets, store it in AWS Secrets Manager. This centralises your secrets and provides better auditing via CloudTrail.

```sh
aws secretsmanager create-secret \
    --name "terraform/github-token" \
    --description "GitHub PAT for Terraform provider" \
    --secret-string "ghp_xxxxxxxxxxxxxxxxxxxx" \
    --profile admin
```

Replace `ghp_xxxxxxxxxxxxxxxxxxxx` with your actual PAT value.

!!! tip "Avoid Shell History"
    To keep the PAT out of your shell history, you can pipe it from 1Password:

    ```sh
    aws secretsmanager create-secret \
        --name "terraform/github-token" \
        --description "GitHub PAT for Terraform provider" \
        --secret-string "$(op item get 'GitHub PAT - Terraform' --fields credential)" \
        --profile admin
    ```


### Version Pinning

It's always a good idea to pin the version of terraform that you are using to ensure consistency between local dev and CI/CD. The workflows you create will read the Terraform version from a `.terraform-version` file. You need to create this file manually in your repository root.

First, check which version you've been using locally:

```sh
terraform version
```

Then create the version file with just the version number (no "v" prefix):

```sh
echo "1.10.3" > .terraform-version
```

Commit this file to your repository:

```sh
git add .terraform-version
git commit -m "Add Terraform version file for CI/CD"
```

This ensures CI/CD uses the same Terraform version as your local development, preventing version mismatch issues.

### The CI Workflow (Plan on PR)

Create `.github/workflows/terraform_ci.yml`:

```yaml
name: CI - fmt and plan

on:
  workflow_dispatch:
  pull_request:

jobs:
  # Check that the formatting etc. all works as expected
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

  # Run terraform plan to see what will change
  plan-github:
    name: plan-github
    runs-on: ubuntu-latest
    needs: check-pre-commit
    permissions:
      pull-requests: write
      id-token: write   # Required for OIDC authentication
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
          prefix: github
          github_token: ${{ secrets.GITHUB_TOKEN }}
```

!!! info "Two Different GitHub Tokens"
    This workflow uses two different tokens:

    - **`TF_VAR_GITHUB_TOKEN`**: Your service account PAT, retrieved from AWS Secrets Manager and used by the Terraform GitHub provider to manage organisation resources
    - **`secrets.GITHUB_TOKEN`**: The built-in Actions token, used only for posting PR comments (limited permissions)

    The built-in `GITHUB_TOKEN` cannot manage organisation resources, which is why you need the PAT stored in Secrets Manager.

!!! tip "Workflow Dispatch"
    Including `workflow_dispatch` allows you to manually trigger the workflow from the GitHub UI, useful for debugging or re-running failed plans.

### The CD Workflow (Apply on Merge)

Create `.github/workflows/terraform_apply.yml`:

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
    permissions:
      id-token: write   # Required for OIDC authentication
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
          prefix: github
```

Both workflows retrieve the PAT from AWS Secrets Manager using the `aws-secretsmanager-get-secrets` action. The action sets `TF_VAR_GITHUB_TOKEN` as an environment variable, which Terraform automatically uses.

### Custom Actions

These reusable actions keep your workflows DRY and consistent.

#### Terraform Plan Action

Create `.github/actions/terraform_plan_comment/action.yml`:

```yaml
name: Terraform Plan
description: Runs Terraform plan and comments on PR
inputs:
  prefix:
    description: "The root module directory"
    required: true
  github_token:
    description: "GitHub token for PR comments"
    required: true

runs:
  using: "composite"
  steps:
    - name: Get Terraform Version
      id: tf_version
      run: |
        echo "terraform_version=$(cat .terraform-version)" >> $GITHUB_OUTPUT
      shell: bash

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: ${{ steps.tf_version.outputs.terraform_version }}

    - name: Terraform fmt
      id: fmt
      run: terraform -chdir=${{ inputs.prefix }} fmt -check -recursive
      continue-on-error: true
      shell: bash

    - name: Terraform Init
      id: init
      run: terraform -chdir=${{ inputs.prefix }} init
      shell: bash

    - name: Terraform Validate
      id: validate
      run: terraform -chdir=${{ inputs.prefix }} validate -no-color
      shell: bash

    - name: Terraform Plan
      id: plan
      run: |
        terraform -chdir=${{ inputs.prefix }} plan -out=plan.tmp
        terraform -chdir=${{ inputs.prefix }} show -no-color plan.tmp >${GITHUB_WORKSPACE}/plan.out
      continue-on-error: true
      shell: bash

    - name: Comment on PR
      uses: actions/github-script@v6
      if: github.event_name == 'pull_request'
      with:
        github-token: ${{ inputs.github_token }}
        script: |
          const { data: comments } = await github.rest.issues.listComments({
            owner: context.repo.owner,
            repo: context.repo.repo,
            issue_number: context.issue.number,
          })
          const botComment = comments.find(comment => {
            return comment.user.type === 'Bot' && comment.body.includes('Terraform Summary: `${{ inputs.prefix }}`')
          })

          const fs = require('fs')
          const planFile = fs.readFileSync('plan.out', 'utf8');
          const runUrl = process.env.GITHUB_SERVER_URL + '/' + process.env.GITHUB_REPOSITORY + '/actions/runs/' + process.env.GITHUB_RUN_ID

          var planOutput = '';
          var planSummary = '';
          if (planFile.length > 50000) {
            planSummary = '> Plan output too long - see [Actions run](' + runUrl + ') for details.\n\n';
            planOutput = planFile.toString().substring(planFile.length - 500, planFile.length);
          } else {
            planOutput = planFile;
          };

          const output = `## Terraform Summary: \`${{ inputs.prefix }}\`
          ### ${ '${{ steps.fmt.outcome }}' == 'success' ? ':white_check_mark:' : ':x:' } Format \`${{ steps.fmt.outcome }}\`
          ### ${ '${{ steps.init.outcome }}' == 'success' ? ':white_check_mark:' : ':x:' } Init \`${{ steps.init.outcome }}\`
          ### ${ '${{ steps.validate.outcome }}' == 'success' ? ':white_check_mark:' : ':x:' } Validate \`${{ steps.validate.outcome }}\`
          ### ${ '${{ steps.plan.outcome }}' == 'success' ? ':white_check_mark:' : ':x:' } Plan \`${{ steps.plan.outcome }}\`

          <details><summary>Show Plan</summary>

          ${planSummary}\`\`\`
          ${planOutput}
          \`\`\`
          </details>

          *Pusher: @${{ github.actor }}, Workflow: \`${{ github.workflow }}\`*`;

          if (botComment) {
            github.rest.issues.updateComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              comment_id: botComment.id,
              body: output
            })
          } else {
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            })
          }

    - name: Terraform Plan Status
      if: steps.plan.outcome == 'failure' || steps.fmt.outcome == 'failure'
      run: exit 1
      shell: bash
```

#### Terraform Apply Action

Create `.github/actions/terraform_apply/action.yml`:

```yaml
name: Terraform Apply
description: Applies Terraform configuration
inputs:
  prefix:
    description: 'The root module directory'
    required: true

runs:
  using: "composite"
  steps:
    - name: Get Terraform Version
      id: tf_version
      run: |
        echo "terraform_version=$(cat .terraform-version)" >> $GITHUB_OUTPUT
      shell: bash

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: ${{ steps.tf_version.outputs.terraform_version }}

    - name: Terraform Init
      run: terraform -chdir=${{ inputs.prefix }} init -input=false
      shell: bash

    - name: Terraform Apply
      run: terraform -chdir=${{ inputs.prefix }} apply -auto-approve -input=false
      shell: bash
```

## OIDC Authentication with AWS

Rather than storing long-lived AWS credentials as GitHub secrets, use OpenID Connect (OIDC) to establish trust between GitHub and AWS. This approach:

- **Eliminates credential rotation**: No access keys to manage or rotate
- **Provides short-lived credentials**: Tokens expire after the workflow completes
- **Enables fine-grained access**: Control which repositories and branches can assume roles

### How OIDC Works

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│ GitHub Actions  │────▶│  AWS IAM        │────▶│  Assume Role    │
│ requests token  │     │  validates OIDC │     │  (temporary)    │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

1. GitHub Actions requests an OIDC token from GitHub's identity provider
2. AWS IAM validates the token against the configured trust policy
3. If valid, AWS issues temporary credentials for the specified role

### Create the OIDC Identity Provider

In your AWS account, create the GitHub OIDC provider. You only need to do this once per account. These IAM operations require elevated permissions, so use the `admin` profile.

First, check if the provider already exists:

```sh
aws iam list-open-id-connect-providers --profile admin
```

If no GitHub provider is listed, create one:

```sh
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1 \
  --profile admin
```

!!! info "About the Thumbprint"
    The thumbprint (`6938fd4d98bab03faadb97b34396831e3780aea1`) is the certificate fingerprint for GitHub's OIDC provider. This value is consistent for all GitHub users - you don't need to generate your own. AWS uses it to verify that tokens genuinely come from GitHub.

    Note: AWS now automatically fetches and validates the thumbprint for well-known providers like GitHub, so you may see this value updated automatically in the console.

### Create the Terraform Role

Create an IAM role that GitHub Actions can assume. This involves creating a trust policy and then creating the role.

#### Step 1: Get Your Account ID

First, get your AWS account ID:

```sh
aws sts get-caller-identity --query Account --output text --profile admin
```

Note this value - you'll use it in the trust policy below.

#### Step 2: Create the Trust Policy

Create a file called `trust-policy.json` with the following content. Replace `YOUR_ACCOUNT_ID` with your actual account ID and `YOUR_ORG/YOUR_REPO` with your GitHub organisation and repository name (e.g., `acme-data/data-stack-infrastructure`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG/YOUR_REPO:*"
        }
      }
    }
  ]
}
```

!!! warning "Restrict Repository Access"
    The `sub` condition is critical. It restricts which repositories can assume this role. Without it, any GitHub repository could potentially assume your role. Always specify your organisation and repository name.

#### Step 3: Create the Role

Create the IAM role using the trust policy:

```sh
aws iam create-role \
  --role-name TerraformGitHubActionsRole \
  --assume-role-policy-document file://trust-policy.json \
  --description "Role for GitHub Actions to run Terraform" \
  --profile admin
```

You should see output confirming the role was created. Note the `Arn` value - you'll need this for your workflows.

### Role Permissions

Now attach a policy that grants the permissions Terraform needs. For managing GitHub resources with remote state in S3, the role needs access to the state bucket and lock table.

#### Step 4: Create the Permissions Policy

Create a file called `terraform-policy.json`. Replace the placeholder values with your actual bucket name, account ID, and region.

!!! tip "Finding Your Bucket and Table Names"
    If you followed the [Remote State Setup](1-set-up-terraform-remote-state.md), your bucket and table names will match what you created there (`terraform-state-YOUR_ACCOUNT_ID` is the bucket name, and `terraform-state-lock` the table). You can check this with:

    ```sh
    # List S3 buckets
    aws s3 ls --profile infrastructure-admin | grep terraform

    # List DynamoDB tables
    aws dynamodb list-tables --profile infrastructure-admin
    ```

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
        "arn:aws:s3:::terraform-state-YOUR_ACCOUNT_ID",
        "arn:aws:s3:::terraform-state-YOUR_ACCOUNT_ID/*"
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
      "Resource": "arn:aws:dynamodb:eu-west-2:YOUR_ACCOUNT_ID:table/terraform-state-lock"
    },
    {
      "Sid": "SecretsManagerAccess",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": [
        "arn:aws:secretsmanager:eu-west-2:YOUR_ACCOUNT_ID:secret:terraform/*"
      ]
    }
  ]
}
```

#### Step 5: Create and Attach the Policy

Create the policy and attach it to the role:

```sh
# Create the policy
aws iam create-policy \
  --policy-name TerraformGitHubActionsPolicy \
  --policy-document file://terraform-policy.json \
  --description "Permissions for Terraform GitHub Actions" \
  --profile admin

# Get your account ID for the policy ARN
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text --profile admin)

# Attach the policy to the role
aws iam attach-role-policy \
  --role-name TerraformGitHubActionsRole \
  --policy-arn "arn:aws:iam::${ACCOUNT_ID}:policy/TerraformGitHubActionsPolicy" \
  --profile admin
```

#### Step 6: Verify the Role

Confirm the role is set up correctly:

```sh
# View the role
aws iam get-role --role-name TerraformGitHubActionsRole --profile admin

# List attached policies
aws iam list-attached-role-policies --role-name TerraformGitHubActionsRole --profile admin
```

You can now clean up the temporary policy files:

```sh
rm trust-policy.json terraform-policy.json
```

## Managing Repository Secrets and Variables

GitHub Actions needs certain values to authenticate with AWS and other providers. These fall into two categories:

### Secrets vs Variables

| Type | Use For | Example |
|------|---------|---------|
| **Secrets** | Sensitive values that must never be exposed | `AWS_ACCOUNT_ID`, API keys |
| **Variables** | Non-sensitive configuration | `AWS_REGION`, environment names |

### Configuring Repository Settings

Navigate to your repository's **Settings > Secrets and variables > Actions**.

Add the following:

**Secrets:**
- `AWS_ACCOUNT_ID` - Your AWS account number

**Variables:**
- `AWS_REGION` - Your primary AWS region (e.g., `eu-west-2`)

!!! tip "Account Region"
  You can retrieve your account region by running the following in your terminal:

  ```sh
  aws configure get region --profile data-engineer
  ```

!!! tip "Environment-Specific Secrets"
    For multi-environment setups, use GitHub Environments to scope secrets. Create environments like `production` and `staging`, each with their own `AWS_ACCOUNT_ID`.

### Should Secrets Be in Terraform?

A common question: should you manage GitHub repository secrets through Terraform?

**Arguments against (recommended approach):**

- **Circular dependency**: Terraform needs secrets to run, but Terraform would manage those secrets
- **Bootstrap problem**: You need secrets configured before CI/CD can run
- **Security exposure**: Secret values would appear in Terraform state
- **Blast radius**: A Terraform misconfiguration could delete critical secrets

**Arguments for:**

- **Consistency**: All repository configuration in one place
- **Auditability**: Changes to secrets go through code review

!!! warning "Recommendation: Keep CI/CD Secrets Out of Terraform"
    The bootstrap and circular dependency problems make Terraform-managed CI/CD secrets impractical. Configure these secrets manually through the GitHub UI or CLI, and document the required secrets in your repository README.

For application secrets (database passwords, API keys used by your applications), the approach differs. These go in AWS Secrets Manager, which you've already configured for the GitHub PAT.

## Locking Down Terraform Execution

With CI/CD in place, the final step is preventing local `terraform apply` commands entirely. This ensures all changes flow through the reviewed, auditable pipeline.

### Branch Protection Rules

If you followed the [GitHub Organisation Setup](../initial-setup/1-github-setup.md#configure-branch-protection), you already have branch protection configured on `main`. The key settings for Terraform CI/CD are:

- **Require status checks to pass before merging** - ensures the plan workflow runs successfully
- **Require branches to be up to date before merging** - prevents stale plans from being applied

To add the plan workflow as a required status check:

1. Go to **Settings > Branches** and edit your `main` protection rule
2. Under **Require status checks to pass**, search for `plan-github`
3. Select it to make it required

!!! info "Status Check Names"
    The status check name matches the job name in your workflow. If you have multiple plan jobs (e.g., `plan-github`, `plan-aws`), add each one as a required check.

### CODEOWNERS for Sensitive Files

If you set up CODEOWNERS in the [GitHub Organisation Setup](../initial-setup/1-github-setup.md#create-a-codeowners-file), you can extend it with more specific rules for infrastructure files. Update your `.github/CODEOWNERS` file:

```
# Global owners - all files
*       @your-org/data-platform-admins   @your-org/data-engineers

# Terraform configuration requires platform admin approval
*.tf @your-org/data-platform-admins
*.tfvars @your-org/data-platform-admins

# Workflow changes require platform admin approval
.github/workflows/*.yml @your-org/data-platform-admins
.github/actions/**/*.yml @your-org/data-platform-admins
```

This means any changes to Terraform or workflow files require approval from a data platform admin, even if a data engineer creates the PR.

### Preventing Local Apply

To prevent accidental local applies, add a `role_arn` to your backend configuration. This role can only be assumed from GitHub Actions (via OIDC), so local Terraform commands will fail.

First, get your role ARN:

```sh
aws iam get-role --role-name TerraformGitHubActionsRole --query 'Role.Arn' --output text --profile admin
```

Then update your backend configuration. In your Terraform configuration directory (e.g., `github/`), edit the `backend.tf` file:

```hcl
# github/backend.tf
terraform {
  backend "s3" {
    bucket         = "your-company-terraform-state"
    key            = "github/terraform.tfstate"
    region         = "eu-west-2"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true

    # This role can only be assumed from GitHub Actions via OIDC
    role_arn = "arn:aws:iam::YOUR_ACCOUNT_ID:role/TerraformGitHubActionsRole"
  }
}
```

After adding the `role_arn`, you'll need to reinitialise Terraform. Commit and push the change, then let CI/CD handle it.

!!! warning "What Happens Locally"
    With this configuration, running `terraform init` locally will fail with an access denied error because your local credentials cannot assume the OIDC-only role. This is intentional - it prevents accidental local applies.

!!! tip "Can I Still Run Plan Locally?"
    No - with the `role_arn` in the backend, even `terraform plan` requires access to the state file, which means assuming the role. You'll need to review plan output in PR comments instead.

    If you need local plan capability for debugging, you have two options:

    1. **Temporarily comment out the `role_arn`** - only do this for debugging, never commit
    2. **Use a separate workspace with local state** - create a `terraform.tfvars.local` for testing

    For most teams, relying on CI/CD plan output is the recommended approach.

### Concurrency Control

Terraform apply operations must run one at a time to prevent state conflicts. Add concurrency controls to your apply workflow.

Update `.github/workflows/terraform_apply.yml` to add the `concurrency` block at the job level:

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
    # Prevent parallel applies - only one can run at a time
    concurrency:
      group: terraform-apply-github
      cancel-in-progress: false
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: 'arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/TerraformGitHubActionsRole'
          aws-region: ${{ vars.AWS_REGION }}

      - name: Get secrets from AWS Secrets Manager
        uses: aws-actions/aws-secretsmanager-get-secrets@v2
        with:
          secret-ids: |
            TF_VAR_GITHUB_TOKEN, terraform/github-token

      # ... rest of the job (terraform apply)
```

Key settings:

- **`group`**: A unique identifier for this concurrency group. Use a descriptive name like `terraform-apply-github` or `terraform-apply-aws`.
- **`cancel-in-progress: false`**: If a second workflow is triggered while one is running, the second will **wait** rather than cancelling the first. This is critical for Terraform - you never want to cancel an in-progress apply.

!!! info "Multiple Terraform Configurations"
    If you have multiple Terraform configurations (github, aws, snowflake), use different concurrency groups for each. This allows parallel applies across different configurations while preventing conflicts within the same one.

## Testing Your CI/CD Pipeline

Verify your setup with a low-risk change:

1. Create a new branch:
   ```sh
   git checkout -b test-cicd
   ```

2. Make a minor change (e.g., update a team description in `terraform.tfvars`)

3. Push and create a pull request:
   ```sh
   git push -u origin test-cicd
   ```

4. Verify the CI workflow runs and comments the plan on your PR

5. After approval, merge to main

6. Verify the CD workflow applies the change

7. Check your GitHub organisation to confirm the change applied

!!! success "What You've Accomplished"
    - [x] GitHub Actions workflows for terraform plan and apply
    - [x] OIDC authentication eliminating long-lived credentials
    - [x] AWS Secrets Manager integration for secure credential management
    - [x] Branch protection enforcing code review for all changes
    - [x] CODEOWNERS requiring admin approval for sensitive files
    - [x] Concurrency controls preventing conflicting applies

## Summary

Your Terraform deployment pipeline now enforces:

| Control | How It's Enforced |
|---------|-------------------|
| All changes reviewed | Branch protection requires PR approval |
| Plan before apply | CI runs plan, CD runs apply |
| Consistent environment | GitHub Actions with pinned versions |
| Secure authentication | OIDC with short-lived credentials |
| Centralised secrets | AWS Secrets Manager integration |
| Audit trail | Git history + GitHub Actions logs |
| No local applies | Role only assumable from GitHub Actions |

## What's Next

!!! success
    You've established a production-grade CI/CD pipeline for Terraform. All infrastructure changes now flow through code review with full auditability.

Your next step is to apply this pattern to AWS infrastructure. You'll create a separate Terraform configuration for AWS resources, using the same CI/CD pipeline with additional IAM permissions.

Continue to [AWS Infrastructure with Terraform](aws/index.md) →
