# CI/CD Deployment

On this page, you will:

- [x] Create a GitHub Actions workflow for deploying Prefect flows
- [x] Configure environment variables from AWS Secrets Manager
- [x] Understand environment-specific deployment overrides
- [x] Set up automated deployment on push to main

## Overview

With flows built and tested locally, you need a way to deploy them automatically when code merges to `main`. This page sets up a GitHub Actions workflow that runs `prefect deploy --all` whenever flow code changes.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       CI/CD FLOW DEPLOYMENT                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐   │
│  │   Push to main  │       │  GitHub Actions  │       │  Prefect Cloud  │   │
│  │                 │ ────▶ │                  │ ────▶ │                 │   │
│  │  (flows/ or     │       │  uv sync         │       │  Deployments    │   │
│  │   prefect.yaml) │       │  prefect deploy  │       │  updated        │   │
│  └─────────────────┘       └─────────────────┘       └─────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│                            ┌─────────────────┐                              │
│                            │ AWS Secrets Mgr  │                              │
│                            │ prefect/api-key  │                              │
│                            └─────────────────┘                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Prerequisites

Ensure you have completed:

- [x] [Prefect Cloud Setup](3-prefect-cloud-setup.md) — Work pools and API key in Secrets Manager
- [x] [Your First Flow](7-first-flow.md) — At least one flow deployed
- [x] GitHub OIDC configured for AWS access (from [AWS Terraform Setup](../../getting-started/terraform-setup/aws/index.md))

## Why Automate Deployment?

Without CI/CD, deploying flows requires a developer to run `prefect deploy --all` from their local machine. This creates several problems:

- Deployments can drift from `main` if someone forgets to deploy after merging
- Local environment differences may produce different deployment configurations
- No audit trail of who deployed what and when
- Blocked if the developer who usually deploys is unavailable

Automating deployment ensures every merge to `main` immediately updates Prefect Cloud with the latest flow definitions, schedules, and configuration.

## GitHub Actions Workflow

### Add Repository Secrets

The workflow needs the Prefect API URL to authenticate with Prefect Cloud. In your GitHub repository settings (**Settings** > **Secrets and variables** > **Actions**), add:

| Type | Name | Value |
|------|------|-------|
| Variable | `TERRAFORM_ROLE_ARN` | Your `TerraformGitHubActionsRole` ARN |
| Secret | `PREFECT_API_URL` | `https://api.prefect.cloud/api/accounts/{ACCOUNT_ID}/workspaces/{WORKSPACE_ID}` |

The Prefect API key is fetched from AWS Secrets Manager at runtime (not stored as a GitHub secret), so it can be rotated without updating GitHub.

### Create the Workflow

=== "CI/CD"

    Create `.github/workflows/prefect-deploy.yml` in your `data-pipelines` repository:

    ```yaml
    name: Deploy Prefect Flows

    on:
      push:
        branches: [main]
        paths:
          - 'flows/**'
          - 'pipelines/**'
          - 'sources/**'
          - 'utils/**'
          - 'prefect.yaml'
      workflow_dispatch:  # Allow manual trigger

    env:
      AWS_REGION: eu-west-2
      PYTHON_VERSION: "3.12"

    jobs:
      deploy:
        name: Deploy Flows
        runs-on: ubuntu-latest

        permissions:
          id-token: write    # OIDC authentication
          contents: read

        steps:
          - name: Checkout
            uses: actions/checkout@v4

          - name: Configure AWS credentials
            uses: aws-actions/configure-aws-credentials@v4
            with:
              role-to-assume: ${{ vars.TERRAFORM_ROLE_ARN }}
              aws-region: ${{ env.AWS_REGION }}

          - name: Fetch Prefect API key from Secrets Manager
            id: prefect-key
            run: |
              PREFECT_API_KEY=$(aws secretsmanager get-secret-value \
                --secret-id "prefect/api-key" \
                --query SecretString \
                --output text)
              echo "::add-mask::$PREFECT_API_KEY"
              echo "key=$PREFECT_API_KEY" >> "$GITHUB_OUTPUT"

          - name: Install uv
            uses: astral-sh/setup-uv@v4
            with:
              version: "latest"

          - name: Set up Python
            run: uv python install ${{ env.PYTHON_VERSION }}

          - name: Install dependencies
            run: uv sync

          - name: Deploy all flows
            env:
              PREFECT_API_KEY: ${{ steps.prefect-key.outputs.key }}
              PREFECT_API_URL: ${{ secrets.PREFECT_API_URL }}
            run: uv run prefect deploy --all
    ```

=== "Local"

    To deploy manually from your local machine:

    ```sh
    cd ~/projects/data/data-pipelines

    # Ensure you're authenticated
    uv run prefect cloud login

    # Deploy all flows
    uv run prefect deploy --all
    ```

    Or deploy a specific flow:

    ```sh
    uv run prefect deploy --name exchange-rates-daily
    ```

### How It Works

1. **Trigger**: The workflow runs when files in `flows/`, `pipelines/`, `sources/`, `utils/`, or `prefect.yaml` change on `main`
2. **AWS Authentication**: Uses OIDC to assume the Terraform GitHub Actions role (no long-lived credentials)
3. **Prefect API Key**: Fetched from AWS Secrets Manager at runtime, masked in logs
4. **Dependencies**: Installed via `uv sync` from `pyproject.toml` and `uv.lock`
5. **Deployment**: `prefect deploy --all` reads `prefect.yaml` and creates/updates all deployments in Prefect Cloud

The `workflow_dispatch` trigger also allows manual deployment from the GitHub Actions UI, useful for re-deploying after a Prefect Cloud configuration change.

## Environment-Specific Overrides

If you need different configurations per environment (e.g., different work pools for staging vs production), use Prefect's variable substitution in `prefect.yaml`:

```yaml
deployments:
  - name: exchange-rates-daily
    entrypoint: flows/exchange_rates.py:exchange_rates_daily_flow
    work_pool:
      name: "{{ $PREFECT_WORK_POOL | default('production') }}"
    schedules:
      - cron: "0 9 * * *"
        timezone: "UTC"
        active: true
```

Then set the environment variable in the workflow:

```yaml
- name: Deploy all flows
  env:
    PREFECT_API_KEY: ${{ steps.prefect-key.outputs.key }}
    PREFECT_API_URL: ${{ secrets.PREFECT_API_URL }}
    PREFECT_WORK_POOL: production
  run: uv run prefect deploy --all
```

This allows the same `prefect.yaml` to target different work pools depending on the deployment context.

## API Key Rotation

Because the Prefect API key lives in AWS Secrets Manager (not as a GitHub secret), rotating it requires a single command:

```sh
aws secretsmanager put-secret-value \
    --secret-id "prefect/api-key" \
    --secret-string "NEW_API_KEY_HERE" \
    --profile infrastructure-admin
```

The next workflow run automatically picks up the new key. No GitHub repository settings need to change.

## Troubleshooting

### Workflow Fails at "Fetch Prefect API key"

```
An error occurred (AccessDeniedException) when calling the GetSecretValue operation
```

**Solution**: The GitHub Actions IAM role needs `secretsmanager:GetSecretValue` permission for `prefect/*` secrets. Check the IAM policy in `terraform/aws/oidc.tf` includes the `prefect/*` prefix (configured in [Prefect Cloud Setup](3-prefect-cloud-setup.md)).

### Deployment Succeeds but Flows Don't Run

**Solution**: Check the work pool name in `prefect.yaml` matches an existing work pool, and that a worker is connected to that pool:

```sh
uv run prefect work-pool ls
uv run prefect worker ls --pool production
```

### "No module named" Errors

**Solution**: Ensure all dependencies are in `pyproject.toml` and the `uv sync` step succeeds. Check the workflow logs for installation errors.

## Summary

You've set up automated flow deployment:

- [x] Created a GitHub Actions workflow triggered on push to `main`
- [x] Configured AWS OIDC authentication for Secrets Manager access
- [x] Fetched the Prefect API key securely at runtime
- [x] Deployed flows using `uv run prefect deploy --all`

## What's Next

With CI/CD deployment in place, set up alerting to know when flows fail.

Continue to [Alerting and Notifications](10-alerting.md) →
