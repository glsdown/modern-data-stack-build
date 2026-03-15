# Snowflake Monitoring

On this page, you will:

- [x] Profile slow queries and identify optimisation opportunities
- [x] Right-size warehouses based on usage patterns
- [x] Set up Resource Monitors to prevent budget overruns
- [x] Monitor credit consumption by warehouse and user
- [x] Understand Snowflake's query performance metrics

## Overview

Snowflake provides comprehensive monitoring capabilities for compute usage, query performance, and costs. Effective Snowflake monitoring ensures:

- **Cost control** — Prevent surprise bills from runaway queries or over-provisioned warehouses
- **Performance optimisation** — Identify and fix slow queries
- **Capacity planning** — Right-size warehouses for workload demands
- **Usage attribution** — Track who is using what resources

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SNOWFLAKE MONITORING LAYERS                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Query Performance      Warehouse Usage         Cost Monitoring        │
│  ──────────────────     ───────────────         ───────────────        │
│                                                                         │
│  ┌─────────────────┐   ┌─────────────────┐     ┌─────────────────┐    │
│  │ QUERY_HISTORY   │   │ WAREHOUSE_      │     │ Resource        │    │
│  │ • Duration      │   │ METERING_       │     │ Monitors        │    │
│  │ • Bytes scanned │   │ HISTORY         │     │ • Budget alerts │    │
│  │ │ Partitions    │   │ • Credit usage  │     │ • Auto-suspend  │    │
│  │   pruned        │   │ • Utilisation   │     │ • Notifications │    │
│  └─────────────────┘   └─────────────────┘     └─────────────────┘    │
│         │                       │                       │              │
│         └───────────────────────┴───────────────────────┘              │
│                                 │                                      │
│                                 ▼                                      │
│                      ┌────────────────────┐                            │
│                      │ ACCOUNT_USAGE      │                            │
│                      │ (System Views)     │                            │
│                      │ • 1 year retention │                            │
│                      │ • All accounts     │                            │
│                      └────────────────────┘                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Query Profiling

Snowflake's `QUERY_HISTORY` view shows all queries executed, with performance metrics.

### View Recent Queries

```sql
USE ROLE ACCOUNTADMIN;

SELECT
    query_id,
    query_text,
    user_name,
    warehouse_name,
    execution_status,
    total_elapsed_time / 1000 AS duration_seconds,
    bytes_scanned,
    rows_produced,
    credits_used_cloud_services,
    start_time
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD(day, -7, CURRENT_TIMESTAMP())
ORDER BY total_elapsed_time DESC
LIMIT 20;
```

**Key columns:**
- `total_elapsed_time` — Query duration in milliseconds
- `bytes_scanned` — Data read from storage
- `rows_produced` — Rows returned
- `partitions_scanned` — Micro-partitions read
- `partitions_total` — Total micro-partitions in table (for pruning ratio)

### Identify Slow Queries

Find queries taking >60 seconds:

```sql
SELECT
    query_id,
    LEFT(query_text, 100) AS query_preview,
    user_name,
    warehouse_name,
    total_elapsed_time / 1000 AS duration_seconds,
    bytes_scanned / POW(1024, 3) AS gb_scanned,
    start_time
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD(day, -7, CURRENT_TIMESTAMP())
  AND total_elapsed_time > 60000  -- 60 seconds
  AND execution_status = 'SUCCESS'
ORDER BY total_elapsed_time DESC
LIMIT 10;
```

**Example output:**

| query_id | query_preview | user_name | warehouse_name | duration_seconds | gb_scanned |
|----------|---------------|-----------|----------------|------------------|------------|
| `01a...` | `SELECT * FROM fct_orders WHERE ...` | SVC_DBT | TRANSFORMING | 245.3 | 120.5 |
| `01b...` | `CREATE TABLE staging.temp_...` | ALICE | ANALYTICS_WH | 180.2 | 85.2 |

**Next steps:** Analyse these queries for optimisation (see below).

### Analyse Query Profile

For a specific slow query:

1. In **Snowsight**, navigate to **Activity** → **Query History**
2. Find the query by `query_id` or time range
3. Click on the query → **Query Profile**

**Query Profile shows:**
- **Execution time breakdown** — time spent on each operator (scan, join, aggregate)
- **Data scanned** — bytes read from storage
- **Partition pruning** — how many partitions were skipped
- **Spilling** — whether data spilled to disk (bad for performance)

### Common Performance Issues

#### Issue 1: Full Table Scan

**Symptom:** `partitions_scanned` = `partitions_total` (no pruning)

**Cause:** Query lacks filtering on clustering key or partition column.

**Example:**

```sql
-- BAD: Full table scan
SELECT * FROM fct_orders
WHERE customer_name = 'Alice';  -- customer_name not clustered

-- Scans all partitions (e.g., 1000 partitions for 1GB table)
```

**Fix:** Add clustering key or filter on partition column:

```sql
-- GOOD: Partition pruning
SELECT * FROM fct_orders
WHERE order_date >= '2026-01-01'  -- order_date is clustering key
  AND customer_name = 'Alice';

-- Scans only partitions for Jan 2026+ (e.g., 50 partitions)
```

#### Issue 2: Excessive Data Spilling

**Symptom:** Query Profile shows "Bytes spilled to local storage" or "Bytes spilled to remote storage"

**Cause:** Warehouse too small for query's memory requirements.

**Fix:** Use a larger warehouse:

```sql
-- Temporarily use larger warehouse for this query
USE WAREHOUSE TRANSFORMING_LARGE;

-- Run query
SELECT ...;

-- Switch back
USE WAREHOUSE TRANSFORMING;
```

Or optimise the query:
- Reduce window function complexity
- Break into smaller CTEs
- Use `LIMIT` for exploratory queries

#### Issue 3: Inefficient JOIN

**Symptom:** Long duration on "Join" operator in Query Profile

**Cause:** Joining large tables without proper filters

**Fix:**

```sql
-- BAD: Unrestricted join
SELECT *
FROM fct_orders o
LEFT JOIN dim_customers c ON o.customer_id = c.customer_id;

-- GOOD: Filter before joining
SELECT *
FROM fct_orders o
LEFT JOIN dim_customers c ON o.customer_id = c.customer_id
WHERE o.order_date >= '2026-01-01';  -- Reduces fct_orders rows first
```

#### Issue 4: SELECT *

**Symptom:** High `bytes_scanned` for simple queries

**Cause:** Selecting all columns when only a few are needed

**Fix:**

```sql
-- BAD: Reads all columns
SELECT * FROM fct_orders WHERE order_id = 12345;

-- GOOD: Only read needed columns
SELECT order_id, order_total, order_date
FROM fct_orders
WHERE order_id = 12345;
```

Snowflake stores data in columnar format, so reading fewer columns = less data scanned.

## Warehouse Sizing

Snowflake warehouses come in sizes from X-Small to 6X-Large. Choosing the right size balances cost and performance.

### Warehouse Sizes and Costs

| Size | Credits/Hour | Relative Cost | Use Case |
|------|-------------|---------------|----------|
| **X-Small** | 1 | 1x | Development, small queries |
| **Small** | 2 | 2x | Light production workloads |
| **Medium** | 4 | 4x | Standard production (dbt, BI queries) |
| **Large** | 8 | 8x | Heavy transformations, complex queries |
| **X-Large** | 16 | 16x | Data science, ML workloads |
| **2X-Large** | 32 | 32x | Rare, very large batch jobs |

**Pricing:** ~$2.50 per credit (varies by region and Snowflake edition)

**Example monthly costs (assuming 8 hours/day, 22 days/month):**
- X-Small: 1 credit/hr × 8 hrs/day × 22 days = 176 credits/month = ~$440/month
- Medium: 4 credits/hr × 8 hrs/day × 22 days = 704 credits/month = ~$1,760/month

### Monitor Warehouse Utilisation

Query warehouse credit usage:

```sql
SELECT
    warehouse_name,
    SUM(credits_used) AS total_credits,
    SUM(credits_used) * 2.5 AS estimated_cost_usd,
    COUNT(DISTINCT DATE(start_time)) AS days_active,
    SUM(credits_used) / days_active AS avg_credits_per_day
FROM snowflake.account_usage.warehouse_metering_history
WHERE start_time >= DATEADD(month, -1, CURRENT_TIMESTAMP())
GROUP BY warehouse_name
ORDER BY total_credits DESC;
```

**Example output:**

| warehouse_name | total_credits | estimated_cost_usd | days_active | avg_credits_per_day |
|----------------|---------------|-------------------|-------------|---------------------|
| TRANSFORMING | 125.3 | $313.25 | 30 | 4.18 |
| ANALYTICS_WH | 45.2 | $113.00 | 28 | 1.61 |
| LOADING | 12.5 | $31.25 | 30 | 0.42 |

### Right-Sizing Strategies

#### Strategy 1: Monitor Query Queueing

If queries are queueing (waiting for warehouse capacity):

```sql
-- Check for queueing
SELECT
    warehouse_name,
    COUNT(*) AS queued_queries,
    AVG(queued_overload_time) / 1000 AS avg_queue_seconds
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD(day, -7, CURRENT_TIMESTAMP())
  AND queued_overload_time > 0
GROUP BY warehouse_name
ORDER BY queued_queries DESC;
```

**If queueing is frequent:** Increase warehouse size or add multi-cluster scaling.

#### Strategy 2: Monitor Warehouse Load

Check average query concurrency:

```sql
WITH hourly_concurrency AS (
    SELECT
        warehouse_name,
        DATE_TRUNC('hour', start_time) AS hour,
        COUNT(*) AS queries_in_hour
    FROM snowflake.account_usage.query_history
    WHERE start_time >= DATEADD(day, -7, CURRENT_TIMESTAMP())
      AND execution_status = 'SUCCESS'
    GROUP BY warehouse_name, hour
)
SELECT
    warehouse_name,
    AVG(queries_in_hour) AS avg_queries_per_hour,
    MAX(queries_in_hour) AS peak_queries_per_hour
FROM hourly_concurrency
GROUP BY warehouse_name;
```

**If peak concurrency is high:** Enable multi-cluster scaling.

#### Strategy 3: Test Warehouse Sizes

Run the same query on different warehouse sizes and compare:

```sql
-- Run on X-Small
USE WAREHOUSE TRANSFORMING_XSMALL;
SELECT /* query */ FROM fct_orders WHERE ...;
-- Note duration and cost

-- Run on Small
USE WAREHOUSE TRANSFORMING_SMALL;
SELECT /* same query */ FROM fct_orders WHERE ...;
-- Compare duration and cost
```

**Rule of thumb:**
- Doubling warehouse size = ~2x cost, ~1.5x faster (not linear due to overhead)
- If query finishes in 5 minutes on Small and 10 minutes on X-Small, Small is more cost-effective

### Multi-Cluster Warehouses

For workloads with variable concurrency (e.g., BI dashboards with many users):

```sql
ALTER WAREHOUSE ANALYTICS_WH SET
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 3
    SCALING_POLICY = 'STANDARD';  -- or 'ECONOMY'
```

**Scaling policies:**
- **STANDARD** — Starts additional clusters immediately when queue depth increases (favour performance)
- **ECONOMY** — Waits longer before starting clusters (favour cost)

**Use case:** Lightdash dashboards during business hours (8am-6pm) may spike to 20 concurrent queries. Multi-cluster warehouse scales from 1 to 3 clusters automatically.

## Resource Monitors

Resource Monitors prevent budget overruns by setting credit limits and alerting when thresholds are crossed.

### Create a Resource Monitor

```sql
USE ROLE ACCOUNTADMIN;

CREATE RESOURCE MONITOR monthly_budget
WITH CREDIT_QUOTA = 500  -- 500 credits per month (~$1,250)
    FREQUENCY = MONTHLY
    START_TIMESTAMP = IMMEDIATELY
    TRIGGERS
        ON 75 PERCENT DO NOTIFY  -- Alert at 75% usage
        ON 90 PERCENT DO NOTIFY  -- Alert at 90% usage
        ON 100 PERCENT DO SUSPEND  -- Suspend warehouses at 100%
        ON 110 PERCENT DO SUSPEND_IMMEDIATE;  -- Immediately suspend (kill running queries)
```

**Trigger actions:**
- **NOTIFY** — Send email alert to account admins
- **SUSPEND** — Prevent new queries, allow running queries to finish
- **SUSPEND_IMMEDIATE** — Kill all running queries and suspend

### Assign Monitor to Warehouses

```sql
-- Apply to specific warehouse
ALTER WAREHOUSE TRANSFORMING
SET RESOURCE_MONITOR = monthly_budget;

-- Apply to all warehouses (account-level)
ALTER ACCOUNT
SET RESOURCE_MONITOR = monthly_budget;
```

### View Resource Monitor Status

```sql
SELECT
    name AS monitor_name,
    credit_quota,
    used_credits,
    remaining_credits,
    (used_credits / credit_quota) * 100 AS percent_used
FROM snowflake.account_usage.resource_monitors
WHERE name = 'monthly_budget';
```

**Example output:**

| monitor_name | credit_quota | used_credits | remaining_credits | percent_used |
|--------------|-------------|--------------|-------------------|--------------|
| monthly_budget | 500 | 387.5 | 112.5 | 77.5% |

### Configure Notifications

By default, Snowflake sends emails to `ACCOUNTADMIN` users. To send to Slack:

#### Option A: Email-to-Slack Forwarding

1. Create an email forwarding rule in Gmail/Outlook
2. Forward Snowflake alerts to Slack email integration
3. Example: `data-alerts@your-workspace.slack.com`

#### Option B: Snowflake + Lambda + Slack

1. Create Snowflake notification integration (sends webhook to Lambda)
2. Lambda function formats message and sends to Slack

**Lambda function:**

```python
import json
import urllib3

def lambda_handler(event, context):
    # Parse Snowflake notification
    notification = json.loads(event['body'])

    # Format Slack message
    message = {
        "text": f"🚨 Snowflake Resource Monitor Alert",
        "blocks": [
            {
                "type": "section",
                "text": {
                    "type": "mrkdwn",
                    "text": f"*Monitor:* {notification['monitor_name']}\n*Usage:* {notification['percent_used']}%\n*Threshold:* {notification['threshold']}%"
                }
            }
        ]
    }

    # Send to Slack
    http = urllib3.PoolManager()
    response = http.request(
        'POST',
        'https://hooks.slack.com/services/YOUR/WEBHOOK/URL',
        body=json.dumps(message),
        headers={'Content-Type': 'application/json'}
    )

    return {'statusCode': 200}
```

### Best Practices for Resource Monitors

1. **Set account-level monitor** — prevents runaway costs across all warehouses
2. **Use tiered alerts** — 75% (warning), 90% (urgent), 100% (suspend)
3. **Don't use SUSPEND_IMMEDIATE lightly** — kills running queries, may corrupt incremental loads
4. **Set realistic budgets** — review last 3 months' usage to set appropriate quota
5. **Monitor monthly trends** — adjust quota if usage consistently grows

## Credit Consumption by Warehouse

Track which warehouses consume the most credits:

```sql
SELECT
    warehouse_name,
    DATE(start_time) AS usage_date,
    SUM(credits_used) AS daily_credits,
    SUM(credits_used) * 2.5 AS daily_cost_usd
FROM snowflake.account_usage.warehouse_metering_history
WHERE start_time >= DATEADD(day, -30, CURRENT_TIMESTAMP())
GROUP BY warehouse_name, usage_date
ORDER BY warehouse_name, usage_date;
```

**Visualise in Snowsight:**

1. Run the query
2. Click **Chart** tab
3. Select **Line chart**
4. X-axis: `usage_date`
5. Y-axis: `daily_credits`
6. Group by: `warehouse_name`

**Example chart:**

```
Credits/Day
20 │                                    ╱──TRANSFORMING
   │                                ╱───
15 │                            ╱───
   │                        ╱───
10 │                    ╱───
   │    ANALYTICS_WH───────────────────────────────
 5 │────LOADING───────────────────────────────────
   └────────────────────────────────────────────────
   Jan 1         Jan 15         Feb 1       Today
```

**Insight:** `TRANSFORMING` warehouse credit usage spiking in February → investigate if due to data volume growth or inefficient queries.

## Credit Consumption by User

Attribute costs to users or teams:

```sql
SELECT
    user_name,
    warehouse_name,
    SUM(credits_used_cloud_services) AS cloud_services_credits,
    COUNT(*) AS query_count,
    AVG(total_elapsed_time) / 1000 AS avg_duration_seconds
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD(day, -30, CURRENT_TIMESTAMP())
  AND execution_status = 'SUCCESS'
GROUP BY user_name, warehouse_name
ORDER BY cloud_services_credits DESC
LIMIT 20;
```

**Example output:**

| user_name | warehouse_name | cloud_services_credits | query_count | avg_duration_seconds |
|-----------|----------------|----------------------|-------------|---------------------|
| SVC_DBT | TRANSFORMING | 15.3 | 450 | 12.5 |
| ALICE | ANALYTICS_WH | 8.7 | 1250 | 3.2 |
| BOB | ANALYTICS_WH | 5.2 | 890 | 2.8 |

**Chargeback strategy:** Allocate costs to teams based on their warehouse usage.

## Query Performance Metrics

### Key Metrics to Track

1. **P95 query duration** — 95th percentile query time (ignores outliers)
2. **Queries per hour** — workload intensity
3. **Average bytes scanned per query** — data efficiency
4. **Cache hit rate** — percentage of queries served from cache

### P95 Query Duration

```sql
SELECT
    warehouse_name,
    APPROX_PERCENTILE(total_elapsed_time / 1000, 0.95) AS p95_duration_seconds
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD(day, -7, CURRENT_TIMESTAMP())
  AND execution_status = 'SUCCESS'
GROUP BY warehouse_name;
```

**Target SLO:** P95 duration < 10 seconds for BI queries, < 30 minutes for dbt runs.

### Cache Hit Rate

Snowflake caches query results. Repeated identical queries return instantly from cache.

```sql
SELECT
    COUNT(CASE WHEN query_result_used_from_cache = TRUE THEN 1 END) AS cached_queries,
    COUNT(*) AS total_queries,
    (cached_queries * 100.0 / total_queries) AS cache_hit_rate_pct
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD(day, -7, CURRENT_TIMESTAMP());
```

**High cache hit rate (>30%):** Good — users are running similar queries (dashboards, reports)
**Low cache hit rate (<10%):** Queries are unique (exploratory analysis, one-off queries)

## Summary

You've set up Snowflake monitoring:

- [x] **Query profiling** — identify slow queries and optimise using Query Profile
- [x] **Warehouse sizing** — right-size warehouses based on usage and concurrency
- [x] **Resource Monitors** — prevent budget overruns with credit quotas and alerts
- [x] **Credit attribution** — track costs by warehouse, user, and team
- [x] **Performance metrics** — monitor P95 duration, cache hit rate, and query trends

Snowflake's built-in monitoring provides comprehensive visibility into compute usage and costs.

## What's Next

Implement comprehensive cost monitoring across your entire data stack.

Continue to [Cost Monitoring](8-cost-monitoring.md) →
