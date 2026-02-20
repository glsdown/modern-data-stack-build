# dbt Cloud Setup

On this page, you will:

- [x] Create a dbt Cloud account and connect it to your Snowflake and GitHub repositories
- [x] (Optional) Provision dbt Cloud infrastructure with Terraform
- [x] Configure development and production environments
- [x] Set up production and CI jobs
- [x] Install the dbt Cloud CLI for local development with native deferral
- [x] Understand how dbt Cloud handles deferral and docs automatically

## Overview

dbt Cloud provides a managed platform on top of dbt Core. It handles CI/CD, state management, deferral, and docs hosting without the infrastructure work covered in the [dbt Core Deployment](9-dbt-core-deployment.md) page.

The project files (`models/`, `dbt_project.yml`, `packages.yml`) are identical to dbt Core — only the execution environment changes.

## Create a dbt Cloud Account

1. Go to [cloud.getdbt.com](https://cloud.getdbt.com) and sign up
2. Create an organisation (your company name)
3. On the Developer (free) tier, you get:
   - 1 developer seat
   - 1 project
   - Unlimited jobs
   - Browser IDE
   - Hosted docs site

Store the account credentials in 1Password and AWS Secrets Manager following the same pattern as other service accounts:

```sh
aws secretsmanager put-secret-value \
    --secret-id "dbt-cloud/api-credentials" \
    --secret-string '{
        "api_token": "YOUR_SERVICE_TOKEN",
        "account_id": "YOUR_ACCOUNT_ID",
        "project_id": "YOUR_PROJECT_ID",
        "job_id_production": "YOUR_PRODUCTION_JOB_ID"
    }' \
    --profile infrastructure-admin
```

The `api_token` is used by Prefect to trigger jobs. Generate a **service token** (not a personal token) under Account Settings → Service Tokens.

## Terraform Setup (Alternative)

Instead of manually configuring dbt Cloud via the UI, you can manage it with Terraform using the `dbt-labs/dbtcloud` provider. This approach treats your dbt Cloud configuration as code, versioned alongside your infrastructure.

### Configure the Provider

Add to your Terraform AWS configuration (or create a separate `terraform/dbt-cloud/` directory):

```hcl
# terraform/dbt-cloud/provider.tf
terraform {
  required_providers {
    dbtcloud = {
      source  = "dbt-labs/dbtcloud"
      version = "~> 0.3"
    }
  }
}

provider "dbtcloud" {
  account_id = var.dbt_cloud_account_id
  token      = var.dbt_cloud_api_token
}
```

Store the credentials in Terraform variables:

```hcl
# terraform/dbt-cloud/variables.tf
variable "dbt_cloud_account_id" {
  description = "dbt Cloud account ID"
  type        = string
  sensitive   = true
}

variable "dbt_cloud_api_token" {
  description = "dbt Cloud service token"
  type        = string
  sensitive   = true
}

variable "snowflake_account" {
  description = "Snowflake account identifier"
  type        = string
}

variable "dbt_private_key" {
  description = "SVC_DBT private key (PEM format)"
  type        = string
  sensitive   = true
}
```

Pass these from environment variables in CI/CD:

```sh
export TF_VAR_dbt_cloud_account_id="123456"
export TF_VAR_dbt_cloud_api_token="dbtc_..."
export TF_VAR_snowflake_account="your-account.snowflakecomputing.com"
export TF_VAR_dbt_private_key="$(cat svc_dbt_rsa_key.pem)"
```

### Create the Project

```hcl
# terraform/dbt-cloud/project.tf
resource "dbtcloud_project" "dbt_transform" {
  name = "dbt-transform"
}

resource "dbtcloud_repository" "github" {
  project_id          = dbtcloud_project.dbt_transform.id
  remote_url          = "https://github.com/your-org/dbt-transform.git"
  github_installation_id = var.github_installation_id  # From GitHub App
}

resource "dbtcloud_connection" "snowflake" {
  project_id  = dbtcloud_project.dbt_transform.id
  name        = "Snowflake"
  type        = "snowflake"
  account     = var.snowflake_account
  database    = "ANALYTICS"
  warehouse   = "TRANSFORMING"
  role        = "SVC_DBT"
}

resource "dbtcloud_credential" "svc_dbt" {
  project_id = dbtcloud_project.dbt_transform.id
  auth_type  = "keypair"
  schema     = "ANALYTICS"
  user       = "SVC_DBT"

  num_threads = 8

  snowflake_keypair = {
    private_key = var.dbt_private_key
  }
}
```

### Create Environments

```hcl
# terraform/dbt-cloud/environments.tf
resource "dbtcloud_environment" "development" {
  project_id     = dbtcloud_project.dbt_transform.id
  name           = "Development"
  dbt_version    = "1.8-latest"
  type           = "development"
  credential_id  = dbtcloud_credential.svc_dbt.id

  deployment_type = "production"  # Use production credentials for dev
}

resource "dbtcloud_environment" "production" {
  project_id     = dbtcloud_project.dbt_transform.id
  name           = "Production"
  dbt_version    = "1.8-latest"
  type           = "deployment"
  credential_id  = dbtcloud_credential.svc_dbt.id

  deployment_type = "production"

  custom_branch = "main"
}
```

### Create Jobs

```hcl
# terraform/dbt-cloud/jobs.tf
resource "dbtcloud_job" "production_daily" {
  project_id     = dbtcloud_project.dbt_transform.id
  environment_id = dbtcloud_environment.production.id
  name           = "Production - Daily"

  execute_steps = [
    "dbt source freshness",
    "dbt build"
  ]

  triggers = {
    schedule = true
  }

  schedule = {
    cron = "0 8 * * *"  # Daily at 08:00 UTC
  }

  generate_docs = true
}

resource "dbtcloud_job" "production_weekly_full_refresh" {
  project_id     = dbtcloud_project.dbt_transform.id
  environment_id = dbtcloud_environment.production.id
  name           = "Production - Weekly Full Refresh"

  execute_steps = [
    "dbt build --full-refresh"
  ]

  triggers = {
    schedule = true
  }

  schedule = {
    cron = "0 2 * * 0"  # Weekly on Sunday at 02:00 UTC
  }

  generate_docs = true
}

resource "dbtcloud_job" "ci_pull_request" {
  project_id     = dbtcloud_project.dbt_transform.id
  environment_id = dbtcloud_environment.production.id
  name           = "CI - Pull Request"

  execute_steps = [
    "dbt build --select state:modified+ --defer --state artifact"
  ]

  triggers = {
    github_webhook = true
  }

  deferring_job_id = dbtcloud_job.production_daily.id

  settings = {
    target_name = "ci"
  }
}

output "production_job_id" {
  description = "ID for the production daily job (for Prefect)"
  value       = dbtcloud_job.production_daily.id
}
```

### Apply the Configuration

```sh
cd terraform/dbt-cloud
terraform init
terraform plan
terraform apply
```

Store the output `production_job_id` in AWS Secrets Manager:

```sh
aws secretsmanager put-secret-value \
    --secret-id "dbt-cloud/api-credentials" \
    --secret-string "$(jq -n \
        --arg token "$TF_VAR_dbt_cloud_api_token" \
        --arg account "$TF_VAR_dbt_cloud_account_id" \
        --arg project "$(terraform output -raw project_id)" \
        --arg job "$(terraform output -raw production_job_id)" \
        '{api_token: $token, account_id: $account, project_id: $project, job_id_production: $job}')" \
    --profile infrastructure-admin
```

!!! tip "Manual vs Terraform"
    The Terraform approach is recommended for production setups where you want configuration versioned and reproducible. For learning or small teams, the manual UI approach (documented in the sections below) is faster to get started.

## Connect GitHub Repository

In dbt Cloud:

1. Navigate to **Account Settings** → **Integrations** → **GitHub**
2. Authorise the dbt Cloud GitHub app on your organisation
3. Select the `dbt-transform` repository

This allows dbt Cloud to:
- Clone the repository for each run
- Post CI check results back to pull requests
- Read repository contents in the browser IDE

## Connect Snowflake

Navigate to **Account Settings** → **Connections** → **New Connection**:

| Field | Value |
|-------|-------|
| Connection type | Snowflake |
| Account | `your-account.snowflakecomputing.com` |
| Database | `ANALYTICS` |
| Warehouse | `TRANSFORMING` |
| Role | `SVC_DBT` |

For authentication, use the `SVC_DBT` key-pair credentials:

| Field | Value |
|-------|-------|
| Auth method | Key Pair |
| User | `SVC_DBT` |
| Private key | Paste contents of `svc_dbt_rsa_key.pem` from 1Password |

!!! note "Connection vs Environment credentials"
    dbt Cloud separates the connection (Snowflake account details) from the credentials (user + auth). The connection is shared; credentials are set per environment. This lets you use different service accounts or roles per environment if needed.

## Create Environments

dbt Cloud environments map to your Snowflake databases. Create two environments:

### Development Environment

Navigate to **Environments** → **New Environment**:

| Field | Value |
|-------|-------|
| Name | `Development` |
| Type | Development |
| dbt version | 1.8 (latest) |
| Connection | Snowflake (your connection) |
| Database | `ANALYTICS_DEV` |
| Schema | `ANALYTICS_DEV` |
| Custom branch | (leave blank — each developer uses their own branch) |

The **Schema** field sets `target.schema` in dbt, which the custom `generate_schema_name` macro uses as a prefix for all schemas in non-production environments. With `ANALYTICS_DEV` as the prefix, staging models land in `ANALYTICS_DEV.ANALYTICS_DEV_STAGING`, mart models in `ANALYTICS_DEV.ANALYTICS_DEV_MARTS`, and so on.

Developers connect to this environment using their personal credentials (from their dbt Cloud profile, configured under their account settings with SSO or username+password).

### Production Environment

| Field | Value |
|-------|-------|
| Name | `Production` |
| Type | Deployment |
| dbt version | 1.8 (latest) |
| Connection | Snowflake (your connection) |
| Database | `ANALYTICS` |
| Schema | `ANALYTICS` |
| Credentials | `SVC_DBT` key-pair (the service token, not personal) |
| Custom branch | `main` |

In production, the `generate_schema_name` macro ignores the schema prefix and uses the custom schema name directly — staging models land in `ANALYTICS.STAGING`, mart models in `ANALYTICS.MARTS`, and so on.

## Configure Jobs

### Production Daily Run

Navigate to **Jobs** → **Create Job** → **Deploy Job**:

| Field | Value |
|-------|-------|
| Job name | `Production - Daily` |
| Environment | Production |
| Commands | `dbt source freshness` then `dbt build` |
| Schedule | On a schedule: Daily at 08:00 UTC |
| Generate docs | ✅ Enabled |
| Run on source freshness failure | ❌ (fail the run if sources are stale) |

Setting **Generate docs** means dbt Cloud automatically updates the hosted docs site after each successful run.

!!! tip "Scheduling with Prefect"
    If you want Prefect to orchestrate the timing (so dbt runs after ingestion completes), disable the schedule here and trigger the job via the dbt Cloud API from Prefect instead. See [Prefect Orchestration](11-prefect-orchestration.md) for this pattern.

### Production Weekly Full Refresh

Some models use incremental materialisation and need a periodic full rebuild:

| Field | Value |
|-------|-------|
| Job name | `Production - Weekly Full Refresh` |
| Environment | Production |
| Commands | `dbt build --full-refresh` |
| Schedule | Weekly on Sunday at 02:00 UTC |
| Generate docs | ✅ Enabled |

### Slim CI Job

The slim CI job runs on every pull request, using dbt Cloud's built-in deferral to only build changed models:

Navigate to **Jobs** → **Create Job** → **CI Job**:

| Field | Value |
|-------|-------|
| Job name | `CI - Pull Request` |
| Environment | Production (defer to this environment) |
| Commands | `dbt build --select state:modified+ --defer --state artifact` |
| Triggered by | Pull requests (dbt Cloud triggers this automatically via GitHub integration) |
| Target name | `ci` |

dbt Cloud manages the production state artifact automatically — no S3 bucket required. The CI job defers unchanged upstream models to the production environment.

!!! note "CI environment schema"
    dbt Cloud creates each PR run in a schema named after the pull request (e.g. `DBT_CLOUD_PR_123_456`). The schema is dropped automatically when the PR is merged or closed.

## Using the Browser IDE

The dbt Cloud IDE lets you develop, test, and preview models in the browser without local setup:

1. Navigate to **Develop** in dbt Cloud
2. Select your branch (or create a new one)
3. Write or edit SQL in the editor
4. Click **Preview** to run the query against your dev environment
5. Click **Compile** to see the compiled SQL (with `ref()` and `source()` resolved)
6. Run `dbt build --select your_model` in the terminal at the bottom

The IDE shows:
- **Lineage graph**: visual DAG of your model and its dependencies
- **File tree**: all models, macros, tests, and seeds
- **Compiled SQL**: the resolved SQL that will run in Snowflake

## Native Deferral with dbt Cloud CLI

If developers prefer a local CLI experience, the dbt Cloud CLI provides native deferral without managing S3 artifacts.

### Uninstall dbt Core (if installed)

The dbt Cloud CLI is a separate binary from dbt Core. If you have dbt Core installed (via `uv`, `pip`, or `brew`), uninstall it first to avoid conflicts:

```sh
# If installed via uv (in the dbt-transform project)
cd ~/projects/dbt/dbt-transform
uv remove dbt-core dbt-snowflake

# If installed via Homebrew
brew uninstall dbt-core dbt-snowflake

# If installed globally via pip
pip uninstall dbt-core dbt-snowflake
```

Check that dbt is fully removed:

```sh
which dbt
# Should return nothing

dbt --version
# Should return "command not found"
```

### Install dbt Cloud CLI

```sh
# Install via Homebrew
brew tap dbt-labs/dbt-cli && brew install dbt
```

Then configure:

1. In dbt Cloud, navigate to **Account Settings** → **CLI**
2. Click **Download CLI configuration file** — this gives you a `dbt_cloud.yml` containing your account ID and API token
3. Save it to `~/.dbt/dbt_cloud.yml`
4. Add your project ID to `dbt_project.yml`:

```yaml
dbt-cloud:
  project-id: YOUR_PROJECT_ID
```

Then run dbt as normal — deferral to production is automatic:

```sh
dbt run --select stg_airbyte__contacts+
```

The dbt Cloud CLI automatically:
1. Fetches the latest production `manifest.json` from dbt Cloud
2. Runs changed models in your local dev environment
3. Defers unchanged models to production

No `--defer` or `--state` flags needed — this is handled transparently.

## View Docs and Lineage

After a job run completes:

1. Navigate to **Explore** in dbt Cloud (or the **Docs** link in the job run)
2. Browse the data catalogue — all models, sources, and descriptions
3. Click any model to see its columns, tests, lineage, and recent run history

The docs URL is shareable with analysts and stakeholders. Access is controlled by dbt Cloud account membership.

## Semantic Layer (Team/Enterprise)

On Team and Enterprise plans, the dbt Cloud semantic layer lets you define metrics in dbt and query them from supported BI tools:

```yaml
# models/marts/core/metrics.yml
semantic_models:
  - name: exchange_rates
    description: Exchange rates semantic model
    model: ref('fct_exchange_rates')
    entities:
      - name: exchange_rate
        type: primary
        expr: exchange_rate_id
    dimensions:
      - name: rate_date
        type: time
        type_params:
          time_granularity: day
      - name: target_currency
        type: categorical
    measures:
      - name: average_rate
        agg: average
        expr: exchange_rate
```

!!! warning "BI tool compatibility"
    The semantic layer requires a supported BI tool (Tableau, Power BI, Omni, Mode, etc.). See [dbt Core vs dbt Cloud](2-deployment-options.md) for the full list.

## Summary

You've configured dbt Cloud:

- [x] Created dbt Cloud account and connected GitHub and Snowflake
- [x] (Optional) Provisioned dbt Cloud resources using Terraform for infrastructure-as-code
- [x] Configured Development and Production environments
- [x] Set up Daily and Weekly Full Refresh production jobs
- [x] Set up the Slim CI job for pull request validation
- [x] Installed the dbt Cloud CLI (after uninstalling dbt Core) for local development
- [x] Understood the browser IDE and native deferral with the dbt Cloud CLI
- [x] Stored the dbt Cloud API token in AWS Secrets Manager for Prefect

## What's Next

Connect Prefect to dbt — either triggering dbt Core via CLI or dbt Cloud via API.

Continue to [Prefect Orchestration](11-prefect-orchestration.md) →
