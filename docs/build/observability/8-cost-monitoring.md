# Cost Monitoring

On this page, you will:

- [x] Track costs across all components of your modern data stack
- [x] Implement cost allocation tags for chargeback
- [x] Set up budget alerts in AWS and Snowflake
- [x] Forecast monthly costs based on usage trends
- [x] Optimise spending without sacrificing performance

## Overview

A modern data stack incurs costs across multiple services. Without visibility, costs can spiral unexpectedly. Comprehensive cost monitoring tracks:

- **Snowflake** — Compute credits, storage
- **AWS** — S3, ECS, RDS, data transfer
- **Third-party SaaS** — Airbyte Cloud, Prefect Cloud, dbt Cloud
- **Self-hosted infrastructure** — EC2, Fargate, ALB

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      COST MONITORING STACK                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Data Warehouse        Cloud Infrastructure    SaaS Tools              │
│  ───────────────       ────────────────────    ──────────              │
│                                                                         │
│  ┌──────────────┐     ┌──────────────────┐    ┌──────────────┐        │
│  │ Snowflake    │     │ AWS              │    │ Prefect Cloud│        │
│  │ • Compute    │     │ • S3 storage     │    │ $0-$450/mo   │        │
│  │   credits    │     │ • ECS Fargate    │    └──────────────┘        │
│  │ • Storage    │     │ • RDS databases  │                            │
│  │ • Data       │     │ • Data transfer  │    ┌──────────────┐        │
│  │   transfer   │     │ • ALB            │    │ Airbyte Cloud│        │
│  └──────────────┘     └──────────────────┘    │ $99+/mo      │        │
│         │                      │              └──────────────┘        │
│         │                      │                      │                │
│         └──────────────────────┴──────────────────────┘                │
│                                │                                       │
│                                ▼                                       │
│                    ┌────────────────────────┐                          │
│                    │ Cost Dashboard         │                          │
│                    │ • By service           │                          │
│                    │ • By team              │                          │
│                    │ • By project           │                          │
│                    │ • Forecast vs actual   │                          │
│                    └────────────────────────┘                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Total Cost of Ownership (TCO)

### Budget Build Monthly Costs

| Component | Service | Monthly Cost |
|-----------|---------|--------------|
| **Data Warehouse** | Snowflake (Standard Edition) | £500-1,000 |
| **Orchestration** | Prefect Cloud (Free tier) | £0 |
| **Batch Ingestion** | dlt (open source) | £0 |
| **SaaS Ingestion** | Airbyte Cloud (5 sources) | £99-199 |
| **BI Tool** | Lightdash (self-hosted) | £30 |
| **Observability** | Elementary (self-hosted) | £30 |
| **Data Catalog** | OpenMetadata (self-hosted) | £50 |
| **AWS Infrastructure** | S3 + ECS + RDS + ALB | £200-300 |
| **GitHub Actions** | CI/CD runners | £0 (free tier) |
| **Total** | | **£900-1,700/month** |

### Cost Drivers

**Snowflake (55-60% of total):**
- Compute credits (query execution)
- Storage (compressed data)
- Data transfer (egress to AWS)

**AWS (15-20% of total):**
- ECS Fargate (Lightdash, Airbyte self-hosted, OpenMetadata)
- RDS (PostgreSQL for Lightdash, OpenMetadata)
- S3 (data lake storage)
- Data transfer (between regions, to internet)

**SaaS Tools (10-15% of total):**
- Airbyte Cloud (per connector per month)
- Prefect Cloud (optional paid plans)
- dbt Cloud (if using instead of dlt + self-hosted dbt)

**Self-Hosted Infrastructure (10-15% of total):**
- Compute (ECS tasks)
- Databases (RDS instances)
- Load balancers (ALB)

## Snowflake Cost Monitoring

### Track Daily Credit Usage

```sql
USE ROLE ACCOUNTADMIN;

SELECT
    DATE(start_time) AS usage_date,
    warehouse_name,
    SUM(credits_used) AS daily_credits,
    SUM(credits_used) * 2.5 AS daily_cost_gbp  -- Adjust rate for your region
FROM snowflake.account_usage.warehouse_metering_history
WHERE start_time >= DATEADD(day, -30, CURRENT_TIMESTAMP())
GROUP BY usage_date, warehouse_name
ORDER BY usage_date DESC, daily_credits DESC;
```

**Create a view for easier access:**

```sql
CREATE OR REPLACE VIEW analytics.observability.daily_snowflake_costs AS
SELECT
    DATE(start_time) AS usage_date,
    warehouse_name,
    SUM(credits_used) AS daily_credits,
    SUM(credits_used) * 2.5 AS daily_cost_gbp,
    SUM(SUM(credits_used)) OVER (
        PARTITION BY DATE_TRUNC('month', start_time)
        ORDER BY DATE(start_time)
    ) AS month_to_date_credits
FROM snowflake.account_usage.warehouse_metering_history
GROUP BY usage_date, warehouse_name;
```

Query the view:

```sql
SELECT * FROM analytics.observability.daily_snowflake_costs
WHERE usage_date >= DATEADD(day, -7, CURRENT_DATE())
ORDER BY usage_date DESC;
```

### Storage Costs

Snowflake charges for storage separately from compute. Track storage growth:

```sql
SELECT
    DATE(usage_date) AS month,
    AVG(storage_bytes) / POW(1024, 4) AS avg_storage_tb,
    AVG(stage_bytes) / POW(1024, 4) AS avg_stage_tb,
    AVG(failsafe_bytes) / POW(1024, 4) AS avg_failsafe_tb,
    (avg_storage_tb + avg_stage_tb + avg_failsafe_tb) * 23 AS estimated_monthly_cost_gbp
FROM snowflake.account_usage.storage_usage
WHERE usage_date >= DATEADD(month, -6, CURRENT_DATE())
GROUP BY DATE_TRUNC('month', usage_date)
ORDER BY month DESC;
```

**Storage pricing:** ~$23/TB/month (~£18/TB/month) for on-demand storage.

**Cost optimisation:**
- Drop unused tables: `DROP TABLE IF EXISTS staging.temp_analysis_20250101;`
- Reduce Time Travel retention for non-critical data: `ALTER TABLE SET DATA_RETENTION_TIME_IN_DAYS = 1;`
- Archive old data to S3 and drop from Snowflake

### Data Transfer Costs

Snowflake charges for data egress (data leaving Snowflake to external systems).

```sql
SELECT
    DATE(start_time) AS transfer_date,
    source_cloud,
    source_region,
    target_cloud,
    target_region,
    SUM(bytes_transferred) / POW(1024, 3) AS gb_transferred,
    SUM(bytes_transferred) / POW(1024, 3) * 0.09 AS estimated_cost_gbp  -- ~$0.09/GB
FROM snowflake.account_usage.data_transfer_history
WHERE start_time >= DATEADD(month, -1, CURRENT_TIMESTAMP())
GROUP BY transfer_date, source_cloud, source_region, target_cloud, target_region
ORDER BY gb_transferred DESC;
```

**Common egress scenarios:**
- Exporting data to S3 (for archives or ML)
- Unloading data to BI tools in different cloud providers
- Cross-region replication

**Cost optimisation:**
- Use same cloud provider and region for Snowflake and S3/ECS
- Compress exports: `COPY INTO @s3_stage FILE_FORMAT = (TYPE = PARQUET COMPRESSION = SNAPPY);`
- Limit export frequency (weekly instead of daily)

## AWS Cost Monitoring

### AWS Cost Explorer

1. Log into AWS Console
2. Navigate to **Billing** → **Cost Explorer**
3. View costs by service:
   - **S3** — Data lake storage
   - **ECS** — Fargate tasks (Lightdash, Airbyte, OpenMetadata)
   - **RDS** — PostgreSQL databases
   - **EC2** — ALB, NAT Gateway
   - **Data Transfer** — Cross-region, internet egress

### Cost Allocation Tags

Tag all Terraform-managed resources for cost attribution.

#### Define Tags in Terraform

```hcl
# terraform/modules/ecs_service/variables.tf
variable "tags" {
  description = "Resource tags for cost allocation"
  type        = map(string)
  default     = {}
}

# terraform/ecs/lightdash/main.tf
module "lightdash_service" {
  source = "../../modules/ecs_service"

  # ...

  tags = {
    Project     = "DataPlatform"
    Service     = "Lightdash"
    Environment = "Production"
    ManagedBy   = "Terraform"
    Team        = "DataEngineering"
  }
}
```

#### Enable Cost Allocation Tags in AWS

1. Navigate to **Billing** → **Cost Allocation Tags**
2. Activate tags:
   - `Project`
   - `Service`
   - `Environment`
   - `Team`
3. Wait 24 hours for tags to appear in Cost Explorer

#### Query Costs by Tag

In Cost Explorer:
1. **Group by:** Tag → `Service`
2. **Time range:** Last 30 days
3. View breakdown:
   - Lightdash: £30/month
   - Airbyte: £45/month (if self-hosted)
   - OpenMetadata: £50/month
   - Data Lake (S3): £15/month

### AWS Budgets

Set up budget alerts to prevent overspending.

#### Create a Budget

1. Navigate to **AWS Budgets** → **Create budget**
2. Budget type: **Cost budget**
3. Budget name: `DataPlatform-Monthly`
4. Budgeted amount: £300/month
5. Budget scope: Filter by tag `Project=DataPlatform`
6. Alert thresholds:
   - 80% of budgeted amount (£240)
   - 100% of budgeted amount (£300)
   - 120% of budgeted amount (£360)
7. Alert recipients: `data-engineering@yourcompany.com`

#### Create via Terraform

```hcl
# terraform/aws/budgets.tf
resource "aws_budgets_budget" "data_platform" {
  name              = "DataPlatform-Monthly"
  budget_type       = "COST"
  limit_amount      = "300"
  limit_unit        = "GBP"
  time_unit         = "MONTHLY"

  cost_filter {
    name = "TagKeyValue"
    values = [
      "Project$DataPlatform"
    ]
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = ["data-engineering@yourcompany.com"]
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 100
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = ["data-engineering@yourcompany.com"]
  }
}
```

### Track Specific Services

#### S3 Storage Costs

```sh
aws ce get-cost-and-usage \
    --time-period Start=2026-02-01,End=2026-03-01 \
    --granularity MONTHLY \
    --metrics "UnblendedCost" \
    --group-by Type=SERVICE \
    --filter file://filter.json

# filter.json
{
  "Dimensions": {
    "Key": "SERVICE",
    "Values": ["Amazon Simple Storage Service"]
  }
}
```

#### ECS Fargate Costs

Track Fargate costs by service:

```sql
-- Use AWS Cost and Usage Reports (CUR) in Athena
SELECT
    line_item_usage_type,
    resource_tags_user_service AS service,
    SUM(line_item_unblended_cost) AS cost_usd
FROM cur_database.cost_usage_report
WHERE line_item_product_code = 'AmazonECS'
  AND line_item_usage_start_date >= DATE '2026-02-01'
GROUP BY line_item_usage_type, resource_tags_user_service
ORDER BY cost_usd DESC;
```

## Third-Party SaaS Costs

### Airbyte Cloud

Airbyte Cloud pricing:
- **Free tier:** 1 source, 1 destination, limited rows
- **Team tier:** $99/month base + $20/connector/month

**Track usage:**
1. Log into Airbyte Cloud
2. Navigate to **Billing**
3. View:
   - Active connectors
   - Rows synced per month
   - Cost per connector

**Optimisation:**
- Reduce sync frequency (daily → weekly for non-critical sources)
- Use incremental syncs instead of full refreshes
- Deactivate unused connectors

### Prefect Cloud

Prefect Cloud pricing:
- **Free tier:** Unlimited flows, 7 days log retention
- **Pro tier:** $450/month, 30 days log retention, SLA support

**Cost decision:**
- If 7-day logs sufficient: stay on free tier
- If need longer retention: export logs to CloudWatch/S3 (cheaper than $450/month)

### dbt Cloud vs Self-Hosted dbt

**dbt Cloud:**
- Developer tier: $50/month per user
- Team tier: $100/month per user + $1/day per job
- 5 users = $500+/month

**Self-hosted dbt (with Prefect):**
- dbt Core: Free
- Orchestration: Prefect (free tier)
- Total: £0/month

**Budget build uses:** Self-hosted dbt + Prefect (£0)

## Forecasting Costs

### Historical Trend Analysis

Query last 6 months of Snowflake costs:

```sql
SELECT
    DATE_TRUNC('month', start_time) AS month,
    SUM(credits_used) AS monthly_credits,
    SUM(credits_used) * 2.5 AS monthly_cost_gbp
FROM snowflake.account_usage.warehouse_metering_history
WHERE start_time >= DATEADD(month, -6, CURRENT_TIMESTAMP())
GROUP BY month
ORDER BY month;
```

**Example output:**

| month | monthly_credits | monthly_cost_gbp |
|-------|-----------------|------------------|
| 2025-09 | 180 | £450 |
| 2025-10 | 195 | £488 |
| 2025-11 | 210 | £525 |
| 2025-12 | 225 | £563 |
| 2026-01 | 245 | £613 |
| 2026-02 | 265 | £663 |

**Trend:** ~8% month-over-month growth

**Forecast for March 2026:** 265 × 1.08 = 286 credits = £715

### Growth Drivers

**Typical growth drivers:**
1. **Data volume growth** — more rows ingested → longer dbt runs
2. **New users** — more analysts → more BI queries
3. **New data sources** — added Airbyte connectors → more warehouse load
4. **Complexity growth** — more dbt models → more compute

**Track metrics to attribute growth:**

```sql
-- Data volume growth
SELECT
    DATE_TRUNC('month', loaded_at) AS month,
    COUNT(*) AS row_count
FROM analytics.marts.fct_orders
GROUP BY month
ORDER BY month;

-- Query volume growth
SELECT
    DATE_TRUNC('month', start_time) AS month,
    COUNT(*) AS query_count
FROM snowflake.account_usage.query_history
WHERE execution_status = 'SUCCESS'
GROUP BY month
ORDER BY month;
```

### Forecast Model

Simple linear regression forecast:

```python
import pandas as pd
from sklearn.linear_model import LinearRegression

# Historical data
data = {
    'month': [1, 2, 3, 4, 5, 6],  # Sept to Feb
    'credits': [180, 195, 210, 225, 245, 265]
}
df = pd.DataFrame(data)

# Train model
model = LinearRegression()
model.fit(df[['month']], df['credits'])

# Forecast next 3 months
future_months = [[7], [8], [9]]  # Mar, Apr, May
forecast = model.predict(future_months)

print(f"March forecast: {forecast[0]:.0f} credits = £{forecast[0] * 2.5:.0f}")
print(f"April forecast: {forecast[1]:.0f} credits = £{forecast[1] * 2.5:.0f}")
print(f"May forecast: {forecast[2]:.0f} credits = £{forecast[2] * 2.5:.0f}")
```

**Output:**
```
March forecast: 285 credits = £713
April forecast: 300 credits = £750
May forecast: 315 credits = £788
```

### Scenario Planning

**Scenario 1: Current trajectory**
- Forecast: £715/month (March)
- No action needed if within budget

**Scenario 2: New data sources added**
- Adding 3 new Airbyte sources
- Estimated: +50 credits/month
- New forecast: £715 + £125 = £840/month

**Scenario 3: Optimisation efforts**
- Optimise slow dbt models (reduce runtime 20%)
- Estimated: -50 credits/month
- New forecast: £715 - £125 = £590/month

## Cost Optimisation Strategies

### 1. Snowflake Warehouse Auto-Suspend

Ensure warehouses suspend when idle:

```sql
ALTER WAREHOUSE TRANSFORMING
SET AUTO_SUSPEND = 60;  -- Suspend after 60 seconds of inactivity
```

**Impact:** Saves ~30% on compute costs for development warehouses

### 2. Reduce dbt Model Complexity

Identify expensive dbt models:

```sql
SELECT
    LEFT(query_text, 200) AS model_preview,
    total_elapsed_time / 1000 AS duration_seconds,
    credits_used_cloud_services
FROM snowflake.account_usage.query_history
WHERE query_text ILIKE '%dbt%'
  AND start_time >= DATEADD(day, -7, CURRENT_TIMESTAMP())
ORDER BY total_elapsed_time DESC
LIMIT 10;
```

**Optimisation tactics:**
- Break large models into smaller incremental models
- Add clustering keys to reduce partition scanning
- Use views instead of tables for simple transformations

### 3. Right-Size Warehouses

Monitor warehouse utilisation:

```sql
SELECT
    warehouse_name,
    AVG(avg_running) AS avg_concurrency,
    MAX(avg_running) AS peak_concurrency
FROM snowflake.account_usage.warehouse_load_history
WHERE start_time >= DATEADD(day, -30, CURRENT_TIMESTAMP())
GROUP BY warehouse_name;
```

**If `avg_concurrency < 0.5`:** Warehouse is over-provisioned, reduce size

**If `peak_concurrency > 8`:** Warehouse is under-provisioned, enable multi-cluster scaling or increase size

### 4. Archive Cold Data

Move data older than 2 years to S3:

```sql
-- Export to S3
COPY INTO @s3_archive/fct_orders/
FROM (
    SELECT * FROM analytics.marts.fct_orders
    WHERE order_date < DATEADD(year, -2, CURRENT_DATE())
)
FILE_FORMAT = (TYPE = PARQUET COMPRESSION = SNAPPY);

-- Delete from Snowflake
DELETE FROM analytics.marts.fct_orders
WHERE order_date < DATEADD(year, -2, CURRENT_DATE());
```

**Cost impact:**
- S3 storage: $0.023/GB/month (~£0.018/GB/month)
- Snowflake storage: $0.04/GB/month (~£0.032/GB/month)
- **Savings:** ~45% on storage for archived data

### 5. Use Cheaper AWS Regions

**UK deployment:**
- eu-west-2 (London): More expensive, lower latency
- eu-west-1 (Ireland): ~10% cheaper, slightly higher latency

**Consideration:** For data stored in UK (GDPR), use London. For global teams, Ireland may be sufficient.

### 6. Reserved Capacity (Advanced)

For predictable workloads, pre-purchase Snowflake credits at a discount:

- **On-demand:** $2-4 per credit
- **Capacity commitment (1 year):** ~20% discount
- **Capacity commitment (3 years):** ~40% discount

**Use case:** If consistently using 500+ credits/month, consider annual commitment.

## Cost Dashboard

Create a unified cost dashboard in Snowflake or Lightdash.

### Snowflake Cost Summary View

```sql
CREATE OR REPLACE VIEW analytics.observability.cost_summary AS
WITH snowflake_costs AS (
    SELECT
        'Snowflake Compute' AS cost_category,
        DATE_TRUNC('month', start_time) AS month,
        SUM(credits_used) * 2.5 AS cost_gbp
    FROM snowflake.account_usage.warehouse_metering_history
    GROUP BY month
),
snowflake_storage AS (
    SELECT
        'Snowflake Storage' AS cost_category,
        DATE_TRUNC('month', usage_date) AS month,
        AVG(storage_bytes + stage_bytes) / POW(1024, 4) * 23 AS cost_gbp
    FROM snowflake.account_usage.storage_usage
    GROUP BY month
),
aws_costs AS (
    -- Manually updated monthly or imported from AWS CUR
    SELECT
        'AWS Infrastructure' AS cost_category,
        month,
        cost_gbp
    FROM analytics.observability.aws_monthly_costs
),
saas_costs AS (
    SELECT cost_category, month, cost_gbp
    FROM (
        VALUES
            ('Airbyte Cloud', '2026-02-01'::DATE, 139),
            ('Prefect Cloud', '2026-02-01'::DATE, 0)
    ) AS t(cost_category, month, cost_gbp)
)
SELECT * FROM snowflake_costs
UNION ALL SELECT * FROM snowflake_storage
UNION ALL SELECT * FROM aws_costs
UNION ALL SELECT * FROM saas_costs;
```

Query total costs:

```sql
SELECT
    month,
    SUM(cost_gbp) AS total_monthly_cost
FROM analytics.observability.cost_summary
GROUP BY month
ORDER BY month DESC;
```

**Visualise in Lightdash:** Create a dashboard showing cost trends by category.

## Summary

You've implemented comprehensive cost monitoring:

- [x] **Snowflake costs** — track compute, storage, and data transfer separately
- [x] **AWS costs** — use Cost Explorer, budgets, and cost allocation tags
- [x] **SaaS costs** — monitor Airbyte, Prefect, and dbt Cloud usage
- [x] **Forecasting** — predict future costs using historical trends
- [x] **Optimisation** — right-size warehouses, archive cold data, auto-suspend

Cost monitoring transforms unpredictable spending into managed budgets with actionable insights.

## What's Next

Set up alerting and incident response workflows.

Continue to [Alerting and Incidents](9-alerting-and-incidents.md) →
