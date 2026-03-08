# Anomaly Detection

On this page, you will:

- [x] Understand different types of data anomalies
- [x] Configure Elementary's automated anomaly detection
- [x] Implement custom anomaly detection with SQL
- [x] Set up Great Expectations for statistical validation
- [x] Reduce false positives in anomaly alerts

## Overview

Anomaly detection catches unexpected changes in data before they cause downstream problems. Unlike explicit tests (which check known rules), anomaly detection identifies deviations from historical patterns.

**Types of anomalies:**
- **Volume anomalies** — row counts significantly different from expected
- **Freshness anomalies** — data is stale (no new rows)
- **Schema anomalies** — columns added, removed, or type changes
- **Distribution anomalies** — values outside expected ranges
- **Null rate anomalies** — sudden increase in null values

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      ANOMALY DETECTION STACK                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Data Observation      Anomaly Detection       Alerts                  │
│  ────────────────      ────────────────────    ──────                  │
│                                                                         │
│  ┌──────────────┐     ┌──────────────────┐    ┌──────────────┐        │
│  │ Row counts   │────▶│ Elementary       │───▶│ Slack        │        │
│  │ Null rates   │     │ • Volume         │    │ #data-alerts │        │
│  │ Column types │     │ • Freshness      │    └──────────────┘        │
│  │ Distributions│     │ • Schema         │                            │
│  └──────────────┘     └──────────────────┘                            │
│         │                      │                                       │
│         │              ┌───────▼───────────┐                           │
│         └─────────────▶│ Custom SQL        │                           │
│                        │ anomaly checks    │                           │
│                        └───────────────────┘                           │
│                                │                                       │
│                        ┌───────▼───────────┐                           │
│                        │ Great Expectations│                           │
│                        │ • Statistical     │                           │
│                        │   validation      │                           │
│                        └───────────────────┘                           │
│                                                                         │
│  All anomalies stored in ELEMENTARY schema for trend analysis          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Elementary Anomaly Detection

Elementary provides automated anomaly detection with minimal configuration (you've already set this up in [Elementary Setup](3-elementary-setup.md)).

### Volume Anomaly Detection

**What it detects:** Row counts that deviate significantly from historical averages.

**Algorithm:**
1. Calculate 7-day rolling average row count
2. Calculate standard deviation
3. Alert if current row count outside `average ± (sensitivity × std_dev)`

**Configuration:**

```yaml
# models/marts/core/fct_orders.yml
models:
  - name: fct_orders
    tests:
      - elementary.volume_anomalies:
          timestamp_column: "order_date"
          where_expression: "order_status != 'cancelled'"
          config:
            severity: warn
          tags: ['elementary', 'anomaly_detection']
```

**Example scenario:**

| Date | Row Count | 7-Day Avg | Std Dev | Threshold (3σ) | Anomaly? |
|------|-----------|-----------|---------|----------------|----------|
| Feb 14 | 1,000 | 980 | 50 | 830-1,130 | ✓ Normal |
| Feb 15 | 1,020 | 985 | 52 | 829-1,141 | ✓ Normal |
| Feb 16 | 450 | 990 | 55 | 825-1,155 | ✗ **Anomaly!** |

**Alert message:**
```
🚨 Volume anomaly detected in fct_orders
Expected: ~990 rows (based on 7-day average)
Actual: 450 rows
Deviation: -54.5% from expected
```

### Freshness Anomaly Detection

**What it detects:** Data hasn't been updated recently (stale data).

**Algorithm:**
1. Check `MAX(timestamp_column)` in table
2. Compare to current time
3. Alert if `current_time - MAX(timestamp_column) > threshold`

**Configuration:**

```yaml
models:
  - name: fct_orders
    config:
      elementary:
        timestamp_column: "loaded_at"  # Column tracking when data was loaded

    tests:
      - elementary.freshness_anomalies:
          timestamp_column: "loaded_at"
          config:
            severity: error
            freshness_threshold: 24  # Alert if data older than 24 hours
```

**Example scenario:**

- **Expected:** Data loaded every 6 hours
- **Current:** `MAX(loaded_at) = 2026-02-19 08:00`
- **Now:** `2026-02-20 10:00` (26 hours ago)
- **Threshold:** 24 hours
- **Result:** ✗ Anomaly! Data is stale

**Alert message:**
```
🚨 Freshness anomaly detected in fct_orders
Last updated: 2026-02-19 08:00 (26 hours ago)
Threshold: 24 hours
Action required: Check upstream pipelines (dlt, Airbyte)
```

### Schema Change Detection

**What it detects:** Columns added, removed, or type changes.

**Algorithm:**
1. Store schema snapshot after each dbt run
2. Compare current schema to previous snapshot
3. Alert on differences

**Configuration:**

```yaml
models:
  - name: fct_orders
    tests:
      - elementary.schema_changes:
          config:
            severity: warn
```

**Example scenario:**

**Previous schema:**
```
order_id (NUMBER)
customer_id (NUMBER)
order_total (DECIMAL(10,2))
order_date (DATE)
```

**Current schema:**
```
order_id (NUMBER)
customer_id (NUMBER)
order_total (DECIMAL(10,2))
order_date (DATE)
order_status (VARCHAR)  ← New column added
```

**Alert message:**
```
⚠️ Schema change detected in fct_orders
Change: Column added
Column name: order_status
Column type: VARCHAR
Impact: Downstream dashboards/models may need updates
```

### Null Rate Anomaly Detection

**What it detects:** Sudden increase in null values.

**Configuration:**

```yaml
models:
  - name: fct_orders
    columns:
      - name: customer_email
        tests:
          - elementary.anomaly_detection:
              type: null_rate
              config:
                severity: warn
                anomaly_threshold: 0.1  # Alert if null rate > 10%
```

**Example scenario:**

| Date | Total Rows | Null customer_email | Null Rate |
|------|-----------|---------------------|-----------|
| Feb 14 | 1,000 | 5 | 0.5% |
| Feb 15 | 1,020 | 8 | 0.8% |
| Feb 16 | 950 | 150 | 15.8% ← **Anomaly!** |

**Alert message:**
```
🚨 Null rate anomaly in fct_orders.customer_email
Expected null rate: ~0.5%
Actual null rate: 15.8%
Rows affected: 150 out of 950
```

### Run Elementary Anomaly Detection

Elementary anomaly detection runs automatically if configured in `on-run-end`:

```yaml
# dbt_project.yml
on-run-end:
  - "{{ elementary.upload_test_results() }}"
```

Or run manually:

```sh
cd ~/projects/dbt/dbt-transform

# Run all Elementary anomaly tests
dbt test --select tag:elementary

# Run via Elementary CLI (includes ML-based detection)
edr monitor --profiles-dir ~/.dbt
```

## Custom Anomaly Detection with SQL

For anomalies not covered by Elementary, write custom SQL tests.

### Custom Test: Spike in Refund Rate

**Business rule:** Refund rate should be <5%. Alert if >10%.

```sql
-- tests/custom/refund_rate_anomaly.sql
WITH daily_refunds AS (
    SELECT
        DATE(order_date) AS day,
        COUNT(CASE WHEN order_status = 'refunded' THEN 1 END) AS refunds,
        COUNT(*) AS total_orders,
        (refunds * 100.0 / total_orders) AS refund_rate_pct
    FROM {{ ref('fct_orders') }}
    WHERE order_date >= CURRENT_DATE() - INTERVAL '7 days'
    GROUP BY day
)

SELECT
    day,
    refund_rate_pct
FROM daily_refunds
WHERE refund_rate_pct > 10  -- Anomaly threshold
```

**Add to schema.yml:**

```yaml
models:
  - name: fct_orders
    tests:
      - custom.refund_rate_anomaly:
          config:
            severity: error
```

If this test fails, dbt will alert you to the anomaly.

### Custom Test: Sudden Drop in Revenue

**Business rule:** Daily revenue should not drop >30% from 7-day average.

```sql
-- tests/custom/revenue_drop_anomaly.sql
WITH daily_revenue AS (
    SELECT
        DATE(order_date) AS day,
        SUM(order_total) AS revenue
    FROM {{ ref('fct_orders') }}
    WHERE order_date >= CURRENT_DATE() - INTERVAL '14 days'
    GROUP BY day
),

revenue_with_avg AS (
    SELECT
        day,
        revenue,
        AVG(revenue) OVER (
            ORDER BY day
            ROWS BETWEEN 7 PRECEDING AND 1 PRECEDING
        ) AS avg_revenue_7d
    FROM daily_revenue
)

SELECT
    day,
    revenue,
    avg_revenue_7d,
    ((revenue - avg_revenue_7d) / avg_revenue_7d * 100) AS pct_change
FROM revenue_with_avg
WHERE day = CURRENT_DATE()
  AND pct_change < -30  -- Alert if >30% drop
```

### Custom Test: Unexpected Value in Column

**Business rule:** `order_status` should only contain known values.

```sql
-- tests/custom/unexpected_order_status.sql
SELECT
    order_status,
    COUNT(*) AS row_count
FROM {{ ref('fct_orders') }}
WHERE order_status NOT IN ('pending', 'confirmed', 'shipped', 'delivered', 'cancelled', 'refunded')
GROUP BY order_status
```

If this query returns any rows, dbt test fails, alerting you to unexpected values.

## Great Expectations

Great Expectations is a Python library for data validation with extensive statistical checks.

### Install Great Expectations

```sh
cd ~/projects/dbt/dbt-transform
uv add great_expectations
```

### Initialise Great Expectations

```sh
great_expectations init
```

This creates a `great_expectations/` directory:

```
great_expectations/
├── great_expectations.yml
├── expectations/
├── checkpoints/
└── plugins/
```

### Create Expectation Suite

```sh
great_expectations suite new
```

**Example expectations:**

```python
# great_expectations/expectations/fct_orders_suite.json
{
    "expectation_suite_name": "fct_orders_suite",
    "expectations": [
        {
            "expectation_type": "expect_table_row_count_to_be_between",
            "kwargs": {
                "min_value": 100,
                "max_value": 10000
            }
        },
        {
            "expectation_type": "expect_column_values_to_be_in_set",
            "kwargs": {
                "column": "order_status",
                "value_set": ["pending", "confirmed", "shipped", "delivered", "cancelled", "refunded"]
            }
        },
        {
            "expectation_type": "expect_column_mean_to_be_between",
            "kwargs": {
                "column": "order_total",
                "min_value": 20,
                "max_value": 500
            }
        },
        {
            "expectation_type": "expect_column_quantile_values_to_be_between",
            "kwargs": {
                "column": "order_total",
                "quantile_ranges": {
                    "quantiles": [0.25, 0.5, 0.75],
                    "value_ranges": [[10, 30], [40, 80], [100, 200]]
                }
            }
        }
    ]
}
```

### Run Expectations

```python
# scripts/run_great_expectations.py
import great_expectations as gx

# Connect to data
context = gx.get_context()

# Add Snowflake datasource
datasource = context.sources.add_snowflake(
    name="snowflake_analytics",
    connection_string="snowflake://user:pass@account/database/schema"
)

# Get data asset
data_asset = datasource.add_table_asset(
    name="fct_orders",
    table_name="fct_orders"
)

# Create checkpoint
checkpoint = context.add_or_update_checkpoint(
    name="fct_orders_checkpoint",
    validations=[
        {
            "batch_request": data_asset.build_batch_request(),
            "expectation_suite_name": "fct_orders_suite"
        }
    ]
)

# Run validation
result = checkpoint.run()

# Check results
if not result.success:
    print("❌ Great Expectations validation failed!")
    for validation in result.run_results.values():
        for expectation_result in validation["validation_result"]["results"]:
            if not expectation_result["success"]:
                print(f"Failed: {expectation_result['expectation_config']['expectation_type']}")
                print(f"Details: {expectation_result['result']}")
else:
    print("✅ All expectations passed")
```

### Integrate with Prefect

```python
from prefect import flow, task

@task
def run_great_expectations_validation():
    result = checkpoint.run()

    if not result.success:
        # Send alert
        slack_webhook.notify(text="🚨 Great Expectations validation failed for fct_orders")
        raise ValueError("Great Expectations validation failed")

@flow
def dbt_with_validation():
    run_dbt_build()
    run_great_expectations_validation()  # Run after dbt
```

## Reducing False Positives

Anomaly detection can generate false positives (alerts for normal behaviour). Tune thresholds to reduce noise.

### Strategy 1: Adjust Sensitivity

Elementary's `anomaly_sensitivity` parameter controls how strict detection is:

```yaml
vars:
  elementary:
    anomaly_sensitivity: 3  # Standard deviations (1 = very sensitive, 5 = very strict)
```

**Low sensitivity (1-2):** Catch more anomalies, more false positives
**Medium sensitivity (3):** Balanced (default)
**High sensitivity (4-5):** Fewer alerts, may miss subtle anomalies

**Recommendation:** Start with 3, increase to 4-5 if too many false positives.

### Strategy 2: Exclude Known Patterns

If certain days are always low (e.g., weekends), exclude them:

```yaml
tests:
  - elementary.volume_anomalies:
      where_expression: "DAYOFWEEK(order_date) NOT IN (0, 6)"  # Exclude Sat/Sun
```

### Strategy 3: Set Time-Based Thresholds

Different thresholds for different times:

```sql
-- Custom anomaly test with time-based thresholds
WITH daily_revenue AS (
    SELECT
        DATE(order_date) AS day,
        DAYOFWEEK(order_date) AS day_of_week,
        SUM(order_total) AS revenue
    FROM {{ ref('fct_orders') }}
    GROUP BY day, day_of_week
),

expected_revenue AS (
    SELECT
        day,
        revenue,
        CASE
            WHEN day_of_week IN (0, 6) THEN 5000  -- Weekend threshold
            ELSE 15000  -- Weekday threshold
        END AS expected_min_revenue
    FROM daily_revenue
)

SELECT *
FROM expected_revenue
WHERE revenue < expected_min_revenue
```

### Strategy 4: Use Anomaly Score

Instead of binary pass/fail, calculate anomaly score:

```sql
-- Anomaly score: how many standard deviations from mean
WITH daily_orders AS (
    SELECT
        DATE(order_date) AS day,
        COUNT(*) AS order_count
    FROM {{ ref('fct_orders') }}
    WHERE order_date >= CURRENT_DATE() - INTERVAL '30 days'
    GROUP BY day
),

stats AS (
    SELECT
        AVG(order_count) AS avg_count,
        STDDEV(order_count) AS stddev_count
    FROM daily_orders
),

anomaly_scores AS (
    SELECT
        d.day,
        d.order_count,
        s.avg_count,
        ABS(d.order_count - s.avg_count) / s.stddev_count AS anomaly_score
    FROM daily_orders d
    CROSS JOIN stats s
)

SELECT
    day,
    order_count,
    avg_count,
    anomaly_score,
    CASE
        WHEN anomaly_score > 3 THEN 'Critical'
        WHEN anomaly_score > 2 THEN 'Warning'
        ELSE 'Normal'
    END AS severity
FROM anomaly_scores
WHERE day = CURRENT_DATE()
  AND anomaly_score > 2  -- Only alert if score > 2
```

### Strategy 5: Alert Aggregation

Instead of alerting on every anomaly, aggregate:

```python
@task
def aggregate_anomalies():
    """Send one Slack message summarising all anomalies detected today"""

    # Query Elementary results
    anomalies = query_elementary_anomalies()

    if len(anomalies) == 0:
        return  # No anomalies, no alert

    # Group by severity
    critical = [a for a in anomalies if a['severity'] == 'critical']
    warnings = [a for a in anomalies if a['severity'] == 'warning']

    # Send aggregated alert
    slack_webhook.notify(
        text=f"📊 Daily Anomaly Summary ({len(anomalies)} total)\n\n" +
             f"🚨 Critical: {len(critical)}\n" +
             f"⚠️ Warnings: {len(warnings)}\n\n" +
             "View details: https://elementary-ui.com/anomalies"
    )
```

## Anomaly Detection Best Practices

### 1. Start with High-Impact Tables

Focus anomaly detection on critical tables first:
- `fct_orders` (revenue data)
- `fct_customers` (customer data)
- `fct_usage` (product usage)

Don't add anomaly detection to every table — it creates noise.

### 2. Combine Elementary + Custom Tests

- **Elementary:** General anomalies (volume, freshness, schema)
- **Custom tests:** Business-specific rules (refund rate, revenue drops)

### 3. Review Anomalies Weekly

Not all anomalies require immediate action. Review weekly:
- Which anomalies were true issues?
- Which were false positives?
- Adjust thresholds accordingly

### 4. Document Expected Anomalies

Some anomalies are expected (e.g., Black Friday spike):

```yaml
# models/marts/core/fct_orders.yml
models:
  - name: fct_orders
    description: |
      Expected anomalies:
      - Black Friday (last Friday of November): 10x normal volume
      - New Year's Day: 50% drop in volume
      - End of month: 2x normal volume (invoicing)
```

### 5. Alert on Trends, Not Single Points

Instead of alerting on one day's anomaly, alert on trends:

```sql
-- Alert if 3 consecutive days show anomalies
WITH daily_anomalies AS (
    -- Your anomaly detection query
    ...
),

consecutive_anomalies AS (
    SELECT
        day,
        SUM(CASE WHEN is_anomaly THEN 1 ELSE 0 END) OVER (
            ORDER BY day
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ) AS anomaly_count_3d
    FROM daily_anomalies
)

SELECT *
FROM consecutive_anomalies
WHERE anomaly_count_3d >= 3  -- Alert if 3 consecutive days
```

## Summary

You've implemented comprehensive anomaly detection:

- [x] **Elementary automated detection** — volume, freshness, schema, null rate anomalies
- [x] **Custom SQL anomaly tests** — business-specific rules (refund rate, revenue drops)
- [x] **Great Expectations** — statistical validation for distributions and value ranges
- [x] **False positive reduction** — sensitivity tuning, time-based thresholds, anomaly scores
- [x] **Best practices** — focus on high-impact tables, weekly reviews, trend-based alerting

Anomaly detection catches issues before they reach users, improving data quality proactively.

## What's Next

Wrap up the Observability section and plan your next steps.

Continue to [Finishing Up](12-finishing-up.md) →
