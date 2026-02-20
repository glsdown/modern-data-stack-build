# Lightdash Cloud Setup

On this page, you will:

- [x] Create a Lightdash Cloud account and project
- [x] Connect Lightdash to your GitHub `dbt-transform` repository
- [x] Connect Lightdash to Snowflake using the `SVC_LIGHTDASH` service account
- [x] Understand how Lightdash reads your dbt project and discovers models
- [x] Configure deployment and development environments
- [x] Verify Lightdash can query your dbt models

## Overview

Lightdash Cloud is the managed SaaS version of Lightdash. It provides:
- Zero infrastructure management (no ECS, RDS, or Docker to configure)
- Automatic updates and security patches
- Hosted at `https://your-org.lightdash.cloud`
- $2400/month flat rate (unlimited users)

This page covers the quick-start path using Lightdash Cloud. If you prefer self-hosting, skip to [Self-Hosted Lightdash](7-self-hosted-lightdash.md).

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     LIGHTDASH CLOUD ARCHITECTURE                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  GitHub                  Lightdash Cloud              Snowflake         │
│  ──────                  ────────────────             ─────────         │
│                                                                         │
│  ┌──────────────┐        ┌──────────────┐            ┌──────────────┐  │
│  │ dbt-transform│───────▶│ Lightdash    │───────────▶│ ANALYTICS    │  │
│  │ repository   │        │ • Reads dbt  │            │ • REPORTING  │  │
│  │              │        │ • Compiles   │            │   schema     │  │
│  │ • models/    │        │ • Syncs      │            └──────────────┘  │
│  │ • metrics    │        │   metrics    │                             │
│  └──────────────┘        └──────────────┘            SVC_LIGHTDASH     │
│                                                       (read-only)       │
│  Lightdash clones your dbt repo, reads YAML,                           │
│  and queries Snowflake to build dashboards.                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Sign Up for Lightdash Cloud

1. Navigate to [Lightdash Cloud](https://www.lightdash.com/)
2. Click **Start for free** or **Get started**
3. Sign up with:
   - **Email** (or Google/GitHub SSO)
   - **Organisation name** (your company name, e.g., "Acme Data")
   - **Project name** (e.g., "Analytics")

4. Complete the onboarding wizard

You'll land on the Lightdash home page with a prompt to connect your dbt project.

!!! info "Pricing Confirmation"
    Lightdash Cloud is $2400/month (unlimited users). A 14-day free trial is available. No credit card required to start the trial.

## Connect Your GitHub Repository

Lightdash reads your dbt project directly from GitHub. It clones the repository, parses `dbt_project.yml` and model YAML files, and discovers metrics and models.

### Step 1: Authorise GitHub Access

1. In Lightdash Cloud, click **Connect project**
2. Select **GitHub** as your repository provider
3. Click **Authorise Lightdash** to install the Lightdash GitHub App
4. Select your organisation (the one containing the `dbt-transform` repo)
5. Choose **Only select repositories** and select `dbt-transform`
6. Click **Install & Authorise**

### Step 2: Configure the Project

After authorising GitHub, Lightdash prompts you to configure the project:

| Field | Value |
|-------|-------|
| **Repository** | `your-org/dbt-transform` |
| **Branch** | `main` (production branch) |
| **Project subdirectory** | Leave blank (unless dbt project is in a subfolder) |
| **Target** | `prod` (from `profiles.yml`) |

Lightdash will clone the repository and attempt to parse the dbt project.

!!! tip "Multiple Environments"
    You can create multiple Lightdash projects for different branches (e.g., `main` for production, `develop` for staging). This mirrors dbt Cloud's environment model.

## Connect to Snowflake

Lightdash needs Snowflake credentials to query your data. Use the `SVC_LIGHTDASH` service account created in the previous step.

### Step 1: Retrieve Credentials from AWS Secrets Manager

Fetch the credentials you stored earlier:

```sh
aws secretsmanager get-secret-value \
    --secret-id "lightdash/snowflake-credentials" \
    --query SecretString \
    --output text \
    --profile infrastructure-admin | jq -r .
```

Copy the values for:
- `account`
- `user`
- `role`
- `warehouse`
- `database`
- `schema`
- `private_key`

### Step 2: Configure Snowflake Connection in Lightdash

In Lightdash Cloud, navigate to **Settings** → **Project settings** → **Data connection**:

| Field | Value |
|-------|-------|
| **Warehouse type** | Snowflake |
| **Account** | `your-account.snowflakecomputing.com` (from secrets) |
| **User** | `SVC_LIGHTDASH` |
| **Role** | `SVC_LIGHTDASH` |
| **Database** | `ANALYTICS` |
| **Warehouse** | `REPORTING` |
| **Schema** | `REPORTING` |

For authentication, choose **Key pair**:

| Field | Value |
|-------|-------|
| **Authentication method** | Key pair |
| **Private key** | Paste the entire private key from AWS Secrets Manager (including `-----BEGIN RSA PRIVATE KEY-----` and `-----END RSA PRIVATE KEY-----`) |

Click **Test connection**.

Expected result:
```
✅ Connection successful!
Connected to ANALYTICS.REPORTING as SVC_LIGHTDASH.
```

Click **Save**.

!!! warning "Private Key Security"
    The private key is stored encrypted in Lightdash Cloud. Lightdash uses it to authenticate to Snowflake but never exposes it to users.

### Troubleshooting Connection Issues

**Error: "Authentication failed"**
- Verify the private key is correct (copy entire key including headers)
- Ensure public key was added to Snowflake via Terraform
- Check the user exists: `SHOW USERS LIKE 'SVC_LIGHTDASH';`

**Error: "Insufficient privileges"**
- Verify `SVC_LIGHTDASH` has `ANALYTICS_REPORTER` granted: `SHOW GRANTS TO ROLE SVC_LIGHTDASH;`
- Test access manually: `USE ROLE SVC_LIGHTDASH; SELECT COUNT(*) FROM analytics.reporting.fct_exchange_rates;`

**Error: "Database not found"**
- Ensure database is `ANALYTICS` (not `ANALYTICS_DEV`)
- Verify schema is `REPORTING` (not `MARTS` or `STAGING`)

## Compile and Sync the dbt Project

After connecting Snowflake, Lightdash compiles your dbt project to discover models and metrics.

### Step 1: Trigger Initial Compile

In Lightdash, navigate to **Settings** → **Project settings** → **dbt**:

1. Click **Compile project**
2. Lightdash will:
   - Clone the `dbt-transform` repository
   - Run `dbt deps` to install packages
   - Run `dbt compile` to generate the `manifest.json`
   - Parse model YAML files to discover metrics

This takes 30-90 seconds.

Expected result:
```
✅ Compilation successful!
Discovered 5 models, 12 metrics, 8 dimensions.
```

### Step 2: Review Discovered Models

Navigate to **Explore** in the left sidebar. You should see:

- `fct_exchange_rates` (from `models/marts/core/fct_exchange_rates.sql`)
- `dim_products` (from `models/marts/core/dim_products.sql`)
- Other models you've published to the `REPORTING` schema

Click a model to see:
- **Dimensions** (columns from the model)
- **Metrics** (defined in the model's YAML file, e.g., `fct_exchange_rates.yml`)
- **Joins** (if you've defined relationships in dbt)

!!! info "How Lightdash Discovers Models"
    Lightdash reads your dbt project and looks for:
    1. Models in `models/` directory
    2. YAML files with `meta` tags for metrics and dimensions
    3. Relationships defined in `sources.yml` and model YAML

    It does **not** create any tables or views in Snowflake — it only reads existing dbt models.

## Understanding Lightdash Environments

Lightdash has two environment concepts:

### Production Environment (Main Branch)

- Reads dbt models from the `main` branch
- Queries the production Snowflake database (`ANALYTICS.REPORTING`)
- Used by end users for dashboards and reports
- Syncs automatically when you merge PRs to `main`

### Development Environment (Feature Branches)

- Developers can create "developer previews" from feature branches
- Queries the development Snowflake database (`ANALYTICS_DEV.ANALYTICS_DEV_REPORTING`)
- Used for testing new metrics before merging to `main`
- Isolated from production

To create a development environment:

1. Navigate to **Settings** → **Environments**
2. Click **+ Add environment**
3. Select branch (e.g., `feature/new-metrics`)
4. Configure Snowflake to use `ANALYTICS_DEV` database
5. Compile the project

!!! tip "Developer Workflow"
    - Create a branch in `dbt-transform` with new metrics
    - Create a Lightdash dev environment for that branch
    - Test metrics in Lightdash before merging
    - Merge to `main` → Lightdash production auto-syncs

## Auto-Sync on Git Push

Lightdash can automatically recompile when you push to GitHub:

1. Navigate to **Settings** → **Project settings** → **Git integration**
2. Enable **Auto-sync on push**
3. Select branches to sync: `main` (and optionally `develop`)

Now when you:
- Merge a PR to `main` → Lightdash automatically recompiles and syncs new metrics
- Push to `develop` → Lightdash updates the development environment

This means your BI tool stays in sync with your dbt project automatically.

## Test Querying a Model

### Step 1: Navigate to Explore

1. Click **Explore** in the left sidebar
2. Select **fct_exchange_rates** (or another model)

### Step 2: Build a Query

Lightdash provides a drag-and-drop interface to build queries:

1. **Dimensions** (left sidebar): Select `rate_date`, `target_currency`
2. **Metrics** (left sidebar): Select a metric (if defined in YAML, e.g., `average_exchange_rate`)
3. **Filters**: Add filter `base_currency` = `GBP`
4. Click **Run query**

Lightdash generates SQL and queries Snowflake:

```sql
-- Generated by Lightdash
SELECT
    rate_date,
    target_currency,
    AVG(exchange_rate) AS average_exchange_rate
FROM analytics.reporting.fct_exchange_rates
WHERE base_currency = 'GBP'
GROUP BY rate_date, target_currency
ORDER BY rate_date DESC;
```

Results appear in a table. Click **Chart** to visualise.

### Step 3: Create a Chart

1. Click **Chart** (top-right)
2. Select chart type: **Line chart**
3. Configure:
   - **X-axis**: `rate_date`
   - **Y-axis**: `average_exchange_rate`
   - **Group by**: `target_currency`

You now have a line chart showing exchange rate trends by currency.

### Step 4: Save the Chart

1. Click **Save chart**
2. Name it: "GBP Exchange Rates Trend"
3. Choose a space (or create one): "Exchange Rates"
4. Click **Save**

The chart is now saved and can be added to dashboards or shared with teammates.

!!! success "First Chart Created"
    You've successfully queried a dbt model and created a chart in Lightdash. This chart is dynamic — as your dbt models update, the chart updates automatically.

## Verify the Setup

Checklist to confirm Lightdash is working:

- [ ] GitHub repository connected (Lightdash can clone `dbt-transform`)
- [ ] Snowflake connection successful (Lightdash can query `ANALYTICS.REPORTING`)
- [ ] dbt project compiled (Lightdash discovered your models and metrics)
- [ ] At least one model visible in **Explore** (e.g., `fct_exchange_rates`)
- [ ] Test query runs successfully (Lightdash can fetch data from Snowflake)
- [ ] Chart created and saved (Lightdash can visualise data)

If all checks pass, Lightdash is ready to build dashboards.

## Cost Recap

**Lightdash Cloud:**
- $2400/month (unlimited users)
- 14-day free trial
- No infrastructure costs (managed by Lightdash)

**Snowflake compute:**
- ~$25/month (estimated, based on 100 dashboard views/day)
- Uses `REPORTING` warehouse (X-Small, auto-suspend 60 seconds)

**Total monthly cost: ~$2425** (Lightdash Cloud + Snowflake compute)

For cost-conscious teams, [Self-Hosted Lightdash](7-self-hosted-lightdash.md) reduces this to ~$55/month (infrastructure + compute).

## Alternative: Self-Hosted Lightdash

If $2400/month is beyond your budget, Lightdash offers a self-hosted option:

- Deploy Lightdash to AWS ECS (infrastructure cost: ~$30-50/month)
- Same features as Cloud (minus managed hosting)
- You manage updates, backups, and scaling
- Terraform deployment (infrastructure as code)

See [Self-Hosted Lightdash](7-self-hosted-lightdash.md) for setup instructions.

## Summary

You've set up Lightdash Cloud:

- [x] Created a Lightdash Cloud account and project
- [x] Connected to GitHub (`dbt-transform` repository)
- [x] Connected to Snowflake (`SVC_LIGHTDASH` service account, `REPORTING` schema)
- [x] Compiled the dbt project and discovered models
- [x] Configured auto-sync on Git push (Lightdash stays in sync with dbt)
- [x] Created and saved a test chart
- [x] Understood production and development environments

Lightdash is now reading your dbt project and querying Snowflake. The next step is to connect your dbt project properly and define metrics in your dbt YAML files.

## What's Next

If you want to self-host instead of using Lightdash Cloud, deploy Lightdash to AWS ECS for cost savings.

Otherwise, skip to [Connect dbt Project](8-connect-dbt-project.md) to configure how Lightdash discovers your dbt models and metrics.

Continue to [Self-Hosted Lightdash](7-self-hosted-lightdash.md) → (optional)

Continue to [Connect dbt Project](8-connect-dbt-project.md) → (if using Cloud)
