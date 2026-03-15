# Elementary Setup

On this page, you will:

- [x] Install Elementary dbt package to track test results
- [x] Configure Elementary CLI for anomaly detection
- [x] Deploy Elementary UI (self-hosted or Elementary Cloud)
- [x] Set up Slack alerts for test failures and anomalies
- [x] Understand Elementary's anomaly detection capabilities

## Overview

[Elementary](https://www.elementary-data.com/) is an open source data observability platform built specifically for dbt. It provides:

1. **Test result tracking** — Historical test results, trends, and failure rates
2. **Anomaly detection** — Automated detection of volume, freshness, and schema changes
3. **Slack alerts** — Notifications for test failures and anomalies
4. **Elementary UI** — Dashboard for data quality metrics and lineage

Elementary integrates with dbt by:
- Adding models to your dbt project (Elementary dbt package)
- Running CLI commands after `dbt run` (Elementary CLI)
- Visualising results in a web UI (Elementary UI or Elementary Cloud)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ELEMENTARY ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  dbt Project                Elementary Models          Elementary UI    │
│  ────────────               ──────────────────         ───────────────  │
│                                                                         │
│  ┌───────────────┐          ┌──────────────────┐      ┌─────────────┐ │
│  │ Your models   │──build──▶│ ELEMENTARY schema│─────▶│ Dashboard   │ │
│  │ • fct_*       │          │ • test_results   │      │ • Test      │ │
│  │ • dim_*       │          │ • model_runs     │      │   trends    │ │
│  │               │          │ • anomalies      │      │ • Anomalies │ │
│  └───────────────┘          └──────────────────┘      │ • Lineage   │ │
│         │                            │                └─────────────┘ │
│         │ dbt test                   │ Elementary CLI                  │
│         │                            │ (after dbt run)                 │
│         ▼                            ▼                                 │
│  Test results ────────────▶  Anomaly detection                         │
│  stored in ELEMENTARY         (volume, freshness,                      │
│                               schema changes)                           │
│                                      │                                 │
│                                      ▼                                 │
│                               Slack alerts                             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Step 1: Install Elementary dbt Package

The Elementary dbt package adds models to your dbt project that store test results and anomaly data.

### Add to packages.yml

```yaml
# dbt-transform/packages.yml
packages:
  - package: elementary-data/elementary
    version: 0.15.2  # Check for latest: https://hub.getdbt.com/elementary-data/elementary

  - package: dbt-labs/dbt_utils
    version: 1.1.1  # Elementary dependency

  - package: calogica/dbt_expectations
    version: 0.10.3  # From previous page
```

### Install packages

```sh
cd ~/projects/dbt/dbt-transform
dbt deps
```

Expected output:
```
Installing elementary-data/elementary@0.15.2
Installing dbt-labs/dbt_utils@1.1.1
Installed 2 packages in 3.2s
```

### Run Elementary models

Elementary creates models in a dedicated `ELEMENTARY` schema:

```sh
dbt run --select elementary
```

This creates tables like:
- `ELEMENTARY.DBT_TESTS` — All test results from `dbt test`
- `ELEMENTARY.DBT_RUN_RESULTS` — dbt run metadata
- `ELEMENTARY.DBT_MODELS` — Model metadata and lineage
- `ELEMENTARY.ELEMENTARY_TEST_RESULTS` — Anomaly detection results

### Verify installation

```sql
-- In Snowsight
USE ROLE ANALYTICS_DEVELOPER;
USE DATABASE ANALYTICS;
USE SCHEMA ELEMENTARY;

SHOW TABLES;
-- Should show: dbt_tests, dbt_run_results, dbt_models, etc.

SELECT COUNT(*) FROM elementary.dbt_tests;
-- Returns 0 initially (no tests have run yet)
```

## Step 2: Update dbt_project.yml

Configure Elementary to run automatically on every `dbt build`:

```yaml
# dbt-transform/dbt_project.yml
models:
  elementary:
    +schema: elementary  # Create ELEMENTARY schema
    +materialized: incremental  # Efficient for large test history

# Run Elementary models after every dbt run
on-run-end:
  - "{{ elementary.upload_dbt_artifacts() }}"  # Upload manifest, run results
  - "{{ elementary.upload_test_results() }}"   # Upload test results
```

### What `on-run-end` Does

After every `dbt run` or `dbt build`, Elementary:
1. Uploads `manifest.json` (dbt DAG) to `ELEMENTARY.DBT_MODELS`
2. Uploads `run_results.json` to `ELEMENTARY.DBT_RUN_RESULTS`
3. Uploads test results to `ELEMENTARY.DBT_TESTS`

This provides historical tracking of all dbt runs and tests.

## Step 3: Install Elementary CLI

The Elementary CLI runs anomaly detection and generates reports.

### Install Elementary

```sh
cd ~/projects/dbt/dbt-transform
uv add "elementary-data[snowflake]"
```

### Verify installation

```sh
edr --version
# Output: elementary, version 0.15.2
```

### Configure Elementary CLI

Create a profile for Elementary (reuses dbt connection):

```yaml
# ~/.dbt/profiles.yml (add to existing file)
elementary:
  target: prod
  outputs:
    prod:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      private_key_path: "{{ env_var('SNOWFLAKE_PRIVATE_KEY_PATH') }}"
      role: SVC_DBT  # Or ANALYTICS_DEVELOPER
      warehouse: TRANSFORMING
      database: ANALYTICS
      schema: ELEMENTARY
      threads: 4
```

Or use `--profiles-dir` to point to your dbt profile:

```sh
edr monitor --profiles-dir ~/.dbt
```

## Step 4: Run Anomaly Detection

Elementary detects anomalies in:
- **Volume** — Row count changes (e.g., 50% drop from 7-day average)
- **Freshness** — Data staleness (e.g., no new rows in 12 hours)
- **Schema** — Column additions, deletions, type changes

### Run anomaly detection

```sh
cd ~/projects/dbt/dbt-transform

edr monitor \
    --profiles-dir ~/.dbt \
    --profile-target prod
```

Expected output:
```
Running anomaly detection...
Detected 2 anomalies:
  - fct_exchange_rates: Row count 30% below 7-day average
  - dim_products: Schema change detected (new column: product_category_v2)

Results stored in ELEMENTARY.ELEMENTARY_TEST_RESULTS
```

### Anomaly detection configuration

Configure thresholds in `dbt_project.yml`:

```yaml
# dbt_project.yml
vars:
  elementary:
    # Volume anomalies
    days_back: 7  # Compare to last 7 days
    anomaly_sensitivity: 3  # Standard deviations (1 = sensitive, 5 = strict)

    # Freshness anomalies
    max_staleness_hours: 24  # Alert if data older than 24 hours

    # Schema change detection
    alert_on_schema_changes: true
```

### Add anomaly tests to models

Explicitly configure anomaly detection per model:

```yaml
# models/marts/core/fct_exchange_rates.yml
models:
  - name: fct_exchange_rates
    config:
      elementary:
        timestamp_column: "loaded_at"  # For freshness detection

    tests:
      # Volume anomaly detection
      - elementary.volume_anomalies:
          timestamp_column: "rate_date"
          where_expression: "base_currency = 'GBP'"
          config:
            severity: warn

      # Freshness anomaly detection
      - elementary.freshness_anomalies:
          timestamp_column: "loaded_at"
          config:
            severity: error  # Alert if data is stale

      # Schema change detection
      - elementary.schema_changes:
          config:
            severity: warn
```

## Step 5: Slack Integration

Send alerts to Slack when tests fail or anomalies are detected.

### Create Slack Webhook

1. Navigate to [Slack API](https://api.slack.com/apps)
2. Click **Create New App** → **From scratch**
3. App name: "Elementary Data Observability"
4. Workspace: Your workspace
5. Navigate to **Incoming Webhooks** → Enable
6. Click **Add New Webhook to Workspace**
7. Select channel: `#data-alerts` (create this channel first)
8. Copy webhook URL: `https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXX`

### Store webhook in environment variable

```sh
# Add to ~/.zshrc or ~/.bashrc
export ELEMENTARY_SLACK_WEBHOOK="https://hooks.slack.com/services/..."
```

Or store in AWS Secrets Manager:

```sh
aws secretsmanager put-secret-value \
    --secret-id "elementary/slack-webhook" \
    --secret-string '{"webhook_url": "https://hooks.slack.com/services/..."}' \
    --profile infrastructure-admin
```

### Send alerts to Slack

```sh
edr monitor \
    --slack-webhook "$ELEMENTARY_SLACK_WEBHOOK" \
    --slack-channel "#data-alerts"
```

Elementary sends messages like:

```
🚨 Elementary Alert: Anomaly Detected

Model: fct_exchange_rates
Anomaly: Volume anomaly
Details: Row count is 2,350 (expected ~3,500 based on 7-day average)
Severity: Warning
Timestamp: 2026-02-20 08:15:00 UTC

View in Elementary: https://your-elementary-ui.com/models/fct_exchange_rates
```

### Customise alert format

```yaml
# dbt_project.yml
vars:
  elementary:
    slack_notification_channel: "#data-alerts"
    slack_group_alerts_by: "model"  # or "test", "severity"
```

## Step 6: Deploy Elementary UI

The Elementary UI provides a dashboard for data quality metrics, test results, and lineage.

### Option A: Elementary Cloud (Managed, $50+/month)

1. Sign up at [elementary-data.com/cloud](https://www.elementary-data.com/cloud)
2. Connect to Snowflake:
   - Account: `your-account.snowflakecomputing.com`
   - User: `SVC_ELEMENTARY` (create this service account, see below)
   - Database: `ANALYTICS`
   - Schema: `ELEMENTARY`
3. Elementary Cloud queries the `ELEMENTARY` schema and displays results

**Pros:**
- Zero infrastructure management
- Automatic updates
- Hosted at `https://your-org.elementary-data.com`

**Cons:**
- $50+/month (unlimited users)
- Requires cloud access to Snowflake

### Option B: Self-Hosted (Free, ~$30/month infrastructure)

Deploy Elementary UI to ECS (similar to Lightdash self-hosted).

#### Create SVC_ELEMENTARY Service Account

```hcl
# terraform/snowflake/service-accounts/elementary.tf
module "service_user_elementary" {
  source = "../modules/snowflake_service_user"

  username    = "SVC_ELEMENTARY"
  comment     = "Service account for Elementary data observability"
  email       = "data-platform@yourcompany.com"
  rsa_public_key = var.svc_elementary_public_key

  user_create_dedicated_role = true
  dedicated_role_grants = [
    "ANALYTICS_DEVELOPER"  # Read ELEMENTARY schema + write anomalies
  ]

  default_warehouse = "TRANSFORMING"
  default_namespace = "ANALYTICS.ELEMENTARY"
  default_role      = "SVC_ELEMENTARY"

  tags = {
    Service   = "elementary"
    ManagedBy = "terraform"
  }
}
```

#### Deploy Elementary UI with Docker

```yaml
# docker-compose.yml (for local testing)
version: '3.8'

services:
  elementary-ui:
    image: elementarydata/elementary:latest
    ports:
      - "8080:8080"
    environment:
      - SNOWFLAKE_ACCOUNT=your-account.snowflakecomputing.com
      - SNOWFLAKE_USER=SVC_ELEMENTARY
      - SNOWFLAKE_DATABASE=ANALYTICS
      - SNOWFLAKE_SCHEMA=ELEMENTARY
      - SNOWFLAKE_WAREHOUSE=TRANSFORMING
      - SNOWFLAKE_ROLE=SVC_ELEMENTARY
      - SNOWFLAKE_PRIVATE_KEY_PATH=/keys/svc_elementary_rsa_key.pem
    volumes:
      - ./keys:/keys:ro
```

Run:

```sh
docker-compose up
```

Navigate to `http://localhost:8080` to view Elementary UI.

#### Deploy to ECS (Production)

Follow the same pattern as [Lightdash self-hosted](../data-analytics/7-self-hosted-lightdash.md):

- ECS Fargate service (~$15/month)
- RDS PostgreSQL for Elementary metadata (~$15/month)
- ALB for HTTPS (~$20/month)
- **Total: ~$50/month**

!!! tip "Reuse Infrastructure Modules"
    Use the same Terraform modules created for Lightdash (ECS service, RDS, ALB) to deploy Elementary UI.

## Step 7: Automate Elementary in Prefect

Run Elementary anomaly detection after every dbt run.

### Prefect Flow

```python
# data-pipelines/flows/dbt_with_elementary.py
from prefect import flow, task
from prefect.blocks.system import String
from prefect_shell import shell_run_command

@task
def run_dbt():
    """Run dbt build"""
    shell_run_command(
        command="cd ~/projects/dbt/dbt-transform && dbt build",
        return_all=True
    )

@task
def run_elementary_monitor():
    """Run Elementary anomaly detection and send Slack alerts"""
    slack_webhook = String.load("elementary-slack-webhook").value

    shell_run_command(
        command=f"""
        cd ~/projects/dbt/dbt-transform &&
        edr monitor \
            --slack-webhook {slack_webhook} \
            --slack-channel '#data-alerts'
        """,
        return_all=True
    )

@flow(name="dbt-with-elementary")
def dbt_with_elementary_flow():
    """Run dbt and Elementary anomaly detection"""
    run_dbt()
    run_elementary_monitor()  # Runs after dbt completes

if __name__ == "__main__":
    dbt_with_elementary_flow()
```

### Schedule

```sh
# Deploy to Prefect
prefect deployment build flows/dbt_with_elementary.py:dbt_with_elementary_flow \
    --name "dbt-with-elementary-daily" \
    --cron "0 8 * * *"  # Daily at 08:00 UTC

prefect deployment apply dbt_with_elementary_flow-deployment.yaml
```

Now Elementary runs automatically after every dbt run, detecting anomalies and sending Slack alerts.

## Elementary Features Overview

### Test Result Tracking

**What it does:** Stores all dbt test results in `ELEMENTARY.DBT_TESTS`.

**Use case:** Track test pass rates over time.

**Example query:**

```sql
SELECT
    test_name,
    COUNT(*) AS total_runs,
    SUM(CASE WHEN status = 'pass' THEN 1 ELSE 0 END) AS passed,
    SUM(CASE WHEN status = 'fail' THEN 1 ELSE 0 END) AS failed,
    (passed * 100.0 / total_runs) AS pass_rate_pct
FROM elementary.dbt_tests
WHERE test_execution_time >= CURRENT_DATE() - INTERVAL '30 days'
GROUP BY test_name
ORDER BY pass_rate_pct ASC;
```

### Volume Anomaly Detection

**What it does:** Detects when row counts deviate from historical averages.

**Algorithm:**
1. Calculate 7-day rolling average row count
2. Calculate standard deviation
3. Alert if current row count > `average ± (sensitivity × std_dev)`

**Example:** If average row count is 10,000 with std dev 500, and sensitivity is 3, alert if row count < 8,500 or > 11,500.

### Freshness Anomaly Detection

**What it does:** Detects when data becomes stale.

**Algorithm:**
1. Check `MAX(timestamp_column)` in table
2. Compare to current time
3. Alert if `current_time - MAX(timestamp_column) > threshold`

**Example:** If `loaded_at` column's max value is 2026-02-19 08:00, and it's now 2026-02-20 10:00 (26 hours later), and threshold is 24 hours, alert.

### Schema Change Detection

**What it does:** Detects column additions, deletions, or type changes.

**Algorithm:**
1. Store schema snapshot after each dbt run
2. Compare current schema to previous snapshot
3. Alert on differences

**Example:** Column `product_category` was `VARCHAR` yesterday, now it's `INT` → schema change detected.

## Summary

You've set up Elementary for dbt observability:

- [x] **Elementary dbt package** installed and running on every `dbt build`
- [x] **Elementary CLI** configured for anomaly detection
- [x] **Slack integration** sending alerts to `#data-alerts`
- [x] **Elementary UI** deployed (self-hosted or Elementary Cloud)
- [x] **Prefect automation** running Elementary after dbt daily runs
- [x] **Anomaly detection** configured for volume, freshness, and schema changes

Elementary provides automated data quality monitoring. Next, add a data catalog for metadata management and lineage.

## What's Next

Deploy OpenMetadata to centralize metadata from dbt, Snowflake, and Prefect.

Continue to [Data Cataloging with OpenMetadata](4-data-cataloging.md) →
