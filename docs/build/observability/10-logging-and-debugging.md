# Logging and Debugging

On this page, you will:

- [x] Centralise logs from Prefect, dbt, Airb yte, and Snowflake
- [x] Configure CloudWatch Logs for ECS services
- [x] Search and filter logs efficiently
- [x] Debug common data pipeline issues using logs
- [x] Set up log retention and archiving

## Overview

Logs are the detailed record of what happened in your data platform. When dashboards show wrong data or pipelines fail, logs provide the context needed to understand why.

Effective logging requires:
1. **Centralisation** — all logs in one place
2. **Structure** — consistent format for parsing
3. **Searchability** — find relevant logs quickly
4. **Retention** — keep logs long enough for debugging, but not forever (cost)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        LOGGING ARCHITECTURE                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Log Sources            Log Collection          Log Storage            │
│  ───────────            ──────────────          ───────────            │
│                                                                         │
│  ┌──────────────┐      ┌──────────────┐        ┌──────────────┐       │
│  │ Prefect      │─────▶│ CloudWatch   │───────▶│ CloudWatch   │       │
│  │ flow runs    │      │ Logs Agent   │        │ Log Groups   │       │
│  └──────────────┘      └──────────────┘        │ • /ecs/      │       │
│                                │                │   prefect    │       │
│  ┌──────────────┐              │                │ • /ecs/      │       │
│  │ dbt          │──────────────┘                │   airbyte    │       │
│  │ models       │                               │ • /ecs/      │       │
│  └──────────────┘                               │   lightdash  │       │
│                                                 └──────────────┘       │
│  ┌──────────────┐      ┌──────────────┐                │              │
│  │ Airbyte      │─────▶│ ECS          │────────────────┘              │
│  │ syncs        │      │ (auto logs)  │                               │
│  └──────────────┘      └──────────────┘                               │
│                                                                         │
│  ┌──────────────┐      ┌──────────────┐        ┌──────────────┐       │
│  │ Snowflake    │─────▶│ QUERY_       │───────▶│ Snowflake    │       │
│  │ queries      │      │ HISTORY      │        │ (internal)   │       │
│  └──────────────┘      └──────────────┘        └──────────────┘       │
│                                                                         │
│  Search logs via CloudWatch Insights or export to S3 for long-term     │
│  retention.                                                             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## CloudWatch Logs for ECS Services

All ECS services (Lightdash, Airbyte self-hosted, OpenMetadata) automatically send logs to CloudWatch.

### View Logs in CloudWatch

1. Log into AWS Console
2. Navigate to **CloudWatch** → **Logs** → **Log groups**
3. Find log group by service:
   - `/ecs/lightdash`
   - `/ecs/airbyte`
   - `/ecs/openmetadata`
   - `/ecs/prefect-worker` (if self-hosted)

### View Real-Time Logs

1. Click on a log group (e.g., `/ecs/lightdash`)
2. Click on a log stream (e.g., `lightdash/main/abc123...`)
3. Logs appear in real-time as the container runs

**Example logs:**

```
2026-02-20 08:15:23 INFO: Lightdash server started on port 8080
2026-02-20 08:15:45 INFO: Connected to database: lightdash-prod
2026-02-20 08:16:02 INFO: User alice@company.com logged in
2026-02-20 08:16:15 ERROR: Query failed: invalid column name "revenue_gbp"
```

### Search Logs with CloudWatch Insights

For complex queries across log streams:

1. Navigate to **CloudWatch** → **Logs** → **Insights**
2. Select log groups: `/ecs/lightdash`, `/ecs/airbyte`
3. Enter query:

```
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 100
```

**Example queries:**

#### Find all errors in the last 24 hours

```
fields @timestamp, @message
| filter @message like /ERROR/ or @message like /FATAL/
| sort @timestamp desc
```

#### Find slow queries (duration > 10 seconds)

```
fields @timestamp, @message, duration
| filter duration > 10000
| sort duration desc
```

#### Count errors by hour

```
fields @timestamp
| filter @message like /ERROR/
| stats count() by bin(@timestamp, 1h)
```

## Prefect Logs

Prefect Cloud stores logs for all flow runs.

### View Logs in Prefect Cloud

1. Navigate to [Prefect Cloud](https://app.prefect.cloud)
2. Click **Flow Runs**
3. Select a flow run
4. Click **Logs** tab

**Log levels:**
- **DEBUG** — detailed diagnostic information
- **INFO** — informational messages (default)
- **WARNING** — something unexpected, but not an error
- **ERROR** — error occurred, task failed

### Customise Log Output

Use Prefect's logger in your flows:

```python
from prefect import flow, task, get_run_logger

@task
def extract_data():
    logger = get_run_logger()

    logger.debug("Starting data extraction...")
    logger.info("Connecting to API: https://api.example.com")

    try:
        data = fetch_from_api()
        logger.info(f"Extracted {len(data)} rows")
        return data
    except Exception as e:
        logger.error(f"Extraction failed: {str(e)}")
        raise

@flow
def data_pipeline():
    logger = get_run_logger()
    logger.info("Starting data pipeline")

    extract_data()

    logger.info("Data pipeline completed successfully")
```

**Output:**

```
2026-02-20 08:15:23 | INFO | Flow run 'data-pipeline-abc123' - Starting data pipeline
2026-02-20 08:15:24 | INFO | Task run 'extract_data-xyz789' - Connecting to API: https://api.example.com
2026-02-20 08:15:28 | INFO | Task run 'extract_data-xyz789' - Extracted 1523 rows
2026-02-20 08:15:30 | INFO | Flow run 'data-pipeline-abc123' - Data pipeline completed successfully
```

### Export Prefect Logs to CloudWatch

For long-term retention (Prefect Cloud Free retains logs for only 7 days):

```python
import boto3
from prefect import flow, get_run_logger

@flow(on_completion=[export_logs])
def data_pipeline():
    # Your pipeline logic
    pass

def export_logs(flow, flow_run, state):
    """Export flow run logs to CloudWatch after completion"""
    logger = get_run_logger()

    cloudwatch = boto3.client('logs', region_name='eu-west-2')

    # Create log stream
    log_stream_name = f"{flow_run.id}"

    try:
        cloudwatch.create_log_stream(
            logGroupName='/prefect/flow-runs',
            logStreamName=log_stream_name
        )
    except cloudwatch.exceptions.ResourceAlreadyExistsException:
        pass  # Stream already exists

    # Get logs from Prefect
    logs = get_flow_run_logs(flow_run.id)

    # Send to CloudWatch
    cloudwatch.put_log_events(
        logGroupName='/prefect/flow-runs',
        logStreamName=log_stream_name,
        logEvents=[
            {
                'timestamp': int(log['timestamp'].timestamp() * 1000),
                'message': log['message']
            }
            for log in logs
        ]
    )

    logger.info(f"Exported logs to CloudWatch: /prefect/flow-runs/{log_stream_name}")
```

## dbt Logs

dbt writes logs to `logs/dbt.log` and `target/run_results.json`.

### View dbt Logs Locally

```sh
cd ~/projects/dbt/dbt-transform

# View latest run logs
tail -100 logs/dbt.log

# View compiled SQL for a model
cat target/compiled/dbt_transform/models/marts/fct_orders.sql

# View run results (JSON)
cat target/run_results.json | jq '.results[] | {name: .unique_id, status: .status, execution_time: .execution_time}'
```

**Example output:**

```json
{"name": "model.dbt_transform.stg_orders", "status": "success", "execution_time": 2.5}
{"name": "model.dbt_transform.fct_orders", "status": "success", "execution_time": 8.3}
{"name": "test.dbt_transform.unique_fct_orders_order_id", "status": "pass", "execution_time": 1.2}
```

### Send dbt Logs to CloudWatch

When running dbt in Prefect:

```python
from prefect import flow, task
from prefect_shell import shell_run_command
import boto3
import json

@task
def run_dbt_with_logging():
    # Run dbt
    result = shell_run_command(
        command="cd ~/projects/dbt/dbt-transform && dbt build --target prod",
        return_all=True
    )

    # Parse dbt output
    stdout = result.stdout
    stderr = result.stderr

    # Send to CloudWatch
    cloudwatch = boto3.client('logs', region_name='eu-west-2')

    cloudwatch.put_log_events(
        logGroupName='/dbt/builds',
        logStreamName=f"build-{datetime.now().strftime('%Y%m%d-%H%M%S')}",
        logEvents=[
            {
                'timestamp': int(datetime.now().timestamp() * 1000),
                'message': stdout + stderr
            }
        ]
    )

    return result

@flow
def dbt_pipeline():
    run_dbt_with_logging()
```

### Parse dbt Logs for Errors

dbt logs contain rich information about model execution:

```python
import re

def parse_dbt_errors(log_content: str) -> list:
    """Extract error messages from dbt logs"""
    errors = []

    # Match lines like: "Database Error in model fct_orders (models/marts/fct_orders.sql)"
    pattern = r"(Database Error|Compilation Error) in model (\w+) \((.*?)\)"

    for match in re.finditer(pattern, log_content):
        error_type = match.group(1)
        model_name = match.group(2)
        file_path = match.group(3)

        errors.append({
            'type': error_type,
            'model': model_name,
            'file': file_path
        })

    return errors
```

### Track dbt Run Metadata in Snowflake

Store dbt run metadata for historical analysis:

```sql
-- Create dbt run history table
CREATE TABLE IF NOT EXISTS analytics.observability.dbt_run_history (
    run_id VARCHAR,
    run_timestamp TIMESTAMP,
    status VARCHAR,
    total_models INT,
    successful_models INT,
    failed_models INT,
    total_tests INT,
    passed_tests INT,
    failed_tests INT,
    execution_time_seconds FLOAT
);
```

Insert run results after each dbt run:

```python
@task
def record_dbt_run_metadata():
    # Parse run_results.json
    with open('~/projects/dbt/dbt-transform/target/run_results.json') as f:
        run_results = json.load(f)

    # Extract metadata
    total_models = sum(1 for r in run_results['results'] if r['unique_id'].startswith('model.'))
    successful_models = sum(1 for r in run_results['results'] if r['unique_id'].startswith('model.') and r['status'] == 'success')

    # Insert into Snowflake
    conn = snowflake.connector.connect(...)
    cursor = conn.cursor()

    cursor.execute(f"""
        INSERT INTO analytics.observability.dbt_run_history
        VALUES (
            '{run_results['metadata']['run_id']}',
            '{run_results['metadata']['generated_at']}',
            '{"success" if all(r['status'] == 'success' for r in run_results['results']) else "failed"}',
            {total_models},
            {successful_models},
            ...
        )
    """)
```

Query historical dbt runs:

```sql
SELECT
    DATE(run_timestamp) AS run_date,
    status,
    AVG(execution_time_seconds) AS avg_execution_time,
    AVG(passed_tests * 100.0 / total_tests) AS avg_test_pass_rate
FROM analytics.observability.dbt_run_history
WHERE run_timestamp >= DATEADD(month, -3, CURRENT_TIMESTAMP())
GROUP BY run_date, status
ORDER BY run_date DESC;
```

## Snowflake Query Logs

Snowflake logs all queries in `QUERY_HISTORY`.

### View Recent Queries

```sql
SELECT
    query_id,
    query_text,
    user_name,
    execution_status,
    error_message,
    start_time,
    total_elapsed_time / 1000 AS duration_seconds
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD(hour, -1, CURRENT_TIMESTAMP())
ORDER BY start_time DESC
LIMIT 100;
```

### Find Failed Queries

```sql
SELECT
    query_id,
    LEFT(query_text, 200) AS query_preview,
    user_name,
    error_code,
    error_message,
    start_time
FROM snowflake.account_usage.query_history
WHERE execution_status = 'FAIL'
  AND start_time >= DATEADD(day, -7, CURRENT_TIMESTAMP())
ORDER BY start_time DESC;
```

**Common error codes:**
- **2003** — Compilation error (SQL syntax invalid)
- **2043** — Object does not exist (table/column not found)
- **630** — Statement reached its timeout (query > timeout limit)

### Correlate Query Logs with dbt Runs

dbt queries include a comment with metadata:

```sql
-- dbt model: fct_orders
-- dbt version: 1.7.0
SELECT ...
```

Search for dbt queries:

```sql
SELECT
    query_id,
    LEFT(query_text, 100) AS model_name,
    execution_status,
    total_elapsed_time / 1000 AS duration_seconds
FROM snowflake.account_usage.query_history
WHERE query_text ILIKE '%dbt model:%'
  AND start_time >= DATEADD(day, -1, CURRENT_TIMESTAMP())
ORDER BY total_elapsed_time DESC;
```

## Debugging Common Issues

### Issue: "Table or view not found"

**Symptom:**
```
ERROR: Object 'ANALYTICS.MARTS.FCT_ORDERS' does not exist
```

**Diagnosis:**

1. Check if table exists:
   ```sql
   SHOW TABLES LIKE 'FCT_ORDERS' IN SCHEMA ANALYTICS.MARTS;
   ```

2. If table doesn't exist, check dbt run logs:
   ```sh
   tail -100 ~/projects/dbt/dbt-transform/logs/dbt.log | grep fct_orders
   ```

3. Check if dbt model is disabled:
   ```yaml
   # models/marts/core/fct_orders.yml
   models:
     - name: fct_orders
       config:
         enabled: false  # ← Disabled!
   ```

**Resolution:**
- If model is disabled: re-enable and run `dbt run --select fct_orders`
- If dbt run failed: check error message, fix SQL, re-run
- If table was dropped manually: re-run dbt to recreate

### Issue: "Null values found in column"

**Symptom:**
```
dbt test failed: not_null test on fct_orders.customer_id
```

**Diagnosis:**

1. Query the table to find null rows:
   ```sql
   SELECT *
   FROM analytics.marts.fct_orders
   WHERE customer_id IS NULL
   LIMIT 10;
   ```

2. Trace upstream to find where nulls originated:
   ```sql
   SELECT *
   FROM analytics.staging.stg_orders
   WHERE order_id IN (SELECT order_id FROM analytics.marts.fct_orders WHERE customer_id IS NULL);
   ```

3. Check source data:
   ```sql
   SELECT *
   FROM dlt_db.raw_orders
   WHERE customer_id IS NULL OR customer_id = '';
   ```

**Resolution:**
- If nulls are in source data: filter them out in dbt staging model
- If nulls are introduced by dbt JOIN: fix JOIN logic (use INNER JOIN instead of LEFT JOIN, or add COALESCE)
- If nulls are valid: adjust test to allow nulls in specific conditions:
  ```yaml
  tests:
    - not_null:
        where: "order_status != 'cancelled'"  # Allow nulls for cancelled orders
  ```

### Issue: "Duplicate rows in table"

**Symptom:**
```
dbt test failed: unique test on fct_orders.order_id
```

**Diagnosis:**

1. Find duplicate rows:
   ```sql
   SELECT
       order_id,
       COUNT(*) AS row_count
   FROM analytics.marts.fct_orders
   GROUP BY order_id
   HAVING COUNT(*) > 1
   ORDER BY row_count DESC
   LIMIT 10;
   ```

2. Inspect duplicate rows:
   ```sql
   SELECT *
   FROM analytics.marts.fct_orders
   WHERE order_id IN (
       SELECT order_id
       FROM analytics.marts.fct_orders
       GROUP BY order_id
       HAVING COUNT(*) > 1
   )
   ORDER BY order_id;
   ```

3. Check dbt model for incorrect JOINs or missing DISTINCT:
   ```sh
   cat ~/projects/dbt/dbt-transform/models/marts/fct_orders.sql
   ```

**Resolution:**
- If duplicates from JOIN: add `DISTINCT` or fix JOIN grain
- If duplicates from source: de-duplicate in staging model using `ROW_NUMBER()`:
  ```sql
  SELECT *
  FROM (
      SELECT
          *,
          ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY loaded_at DESC) AS rn
      FROM staging.stg_orders
  )
  WHERE rn = 1
  ```

### Issue: "Query timeout"

**Symptom:**
```
ERROR: Statement reached its maximum execution time of 300 seconds and was cancelled
```

**Diagnosis:**

1. Check query in Snowflake Query Profile:
   - Find query_id in error message or logs
   - Navigate to Snowsight → Activity → Query History
   - Open Query Profile

2. Identify bottleneck:
   - Large table scan? → Add filters or clustering key
   - Spilling to disk? → Increase warehouse size
   - Inefficient JOIN? → Optimise JOIN order

**Resolution:**
- **Temporary:** Increase query timeout:
  ```sql
  ALTER SESSION SET STATEMENT_TIMEOUT_IN_SECONDS = 600;
  ```
- **Long-term:** Optimise query (see [Snowflake Monitoring](7-snowflake-monitoring.md))

### Issue: "Out of memory"

**Symptom:**
```
ERROR: Out of memory. Warehouse size may be too small
```

**Diagnosis:**

Query Profile shows "Bytes spilled to remote storage" (worst case) or "Bytes spilled to local storage".

**Resolution:**
- Use a larger warehouse temporarily:
  ```sql
  USE WAREHOUSE TRANSFORMING_LARGE;
  ```
- Optimise query:
  - Reduce window function complexity
  - Break query into smaller CTEs
  - Filter early in the query

## Log Retention and Archiving

### CloudWatch Log Retention

Set retention policies to manage costs:

1. Navigate to **CloudWatch** → **Logs** → **Log groups**
2. Select log group (e.g., `/ecs/lightdash`)
3. Click **Actions** → **Edit retention**
4. Choose retention period:
   - **7 days** — debug logs (Airbyte, Lightdash)
   - **30 days** — pipeline logs (Prefect, dbt)
   - **1 year** — audit logs (access logs, compliance)

**Cost impact:**
- CloudWatch Logs: $0.50/GB ingested, $0.03/GB/month stored
- 7-day retention: ~$0.50/GB total
- 1-year retention: ~$0.50 + (12 × $0.03) = ~$0.86/GB total

### Export Logs to S3

For long-term archival (>1 year), export to S3:

1. Navigate to **CloudWatch** → **Logs** → **Log groups**
2. Select log group
3. Click **Actions** → **Create export task**
4. Destination S3 bucket: `s3://data-platform-logs/cloudwatch-archive/`
5. Time range: Last 30 days
6. Click **Export**

**S3 archival cost:** ~$0.023/GB/month (Glacier Deep Archive: ~$0.001/GB/month)

### Automate Log Exports

Use CloudWatch Logs subscription filter + Lambda:

```python
# Lambda function to export logs to S3
import boto3
import gzip
import json
from datetime import datetime

def lambda_handler(event, context):
    s3 = boto3.client('s3')

    # Decode CloudWatch log event
    log_data = event['awslogs']['data']
    decoded = gzip.decompress(base64.b64decode(log_data))
    log_events = json.loads(decoded)

    # Write to S3
    bucket = 'data-platform-logs'
    key = f"cloudwatch-archive/{log_events['logGroup']}/{datetime.now().strftime('%Y/%m/%d')}.json.gz"

    s3.put_object(
        Bucket=bucket,
        Key=key,
        Body=gzip.compress(json.dumps(log_events).encode())
    )

    return {'statusCode': 200}
```

Trigger via CloudWatch Logs subscription:

```sh
aws logs put-subscription-filter \
    --log-group-name "/ecs/lightdash" \
    --filter-name "export-to-s3" \
    --filter-pattern "" \
    --destination-arn "arn:aws:lambda:eu-west-2:123456789012:function:log-exporter"
```

## Summary

You've set up comprehensive logging and debugging:

- [x] **CloudWatch Logs** — centralised logs for all ECS services
- [x] **Prefect logs** — flow run logs with custom log levels
- [x] **dbt logs** — model execution logs and run metadata
- [x] **Snowflake query logs** — all queries tracked in QUERY_HISTORY
- [x] **Debugging workflows** — systematic approaches to common issues
- [x] **Log retention** — cost-effective archival with S3 exports

Centralised logging transforms scattered log files into a unified debugging tool.

## What's Next

Implement anomaly detection to catch issues before they become incidents.

Continue to [Anomaly Detection](11-anomaly-detection.md) →
