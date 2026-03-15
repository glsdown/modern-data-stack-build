# Runbook: Performance Optimisation

## Summary

Diagnose and resolve performance issues across the data stack - slow Snowflake queries, long-running dbt models, pipeline timeouts, and dashboard latency. This runbook covers systematic diagnosis and targeted fixes.

## When to Use

- Snowflake queries are running slower than expected
- dbt models exceed their usual run time
- Pipeline tasks are timing out or consuming excessive credits
- Dashboards are slow to load
- Resource monitor alerts fire on credit usage spikes

## Prerequisites

- [ ] Access: Snowflake with ACCOUNTADMIN or SYSADMIN role (for query history and warehouse management)
- [ ] Access: Prefect Cloud dashboard (for flow run timing)
- [ ] Context: Which queries, models, or pipelines are affected

## Steps

### 1. Identify the Bottleneck

Start by determining which layer has the performance issue:

| Symptom | Likely Layer | Diagnostic |
|---------|-------------|-----------|
| Slow dashboard | Snowflake query or BI tool | Check Snowflake query history |
| Long dbt run | Snowflake query or model design | Check dbt run timing + query profile |
| Pipeline timeout | Source API or Snowflake load | Check Prefect task logs |
| Credit spike | Warehouse sizing or runaway query | Check resource monitor + query history |

### 2. Diagnose Snowflake Query Performance

#### Check Query History

```sql
USE ROLE ACCOUNTADMIN;

-- Find slow queries in the last 24 hours
SELECT
    query_id,
    user_name,
    warehouse_name,
    warehouse_size,
    execution_status,
    total_elapsed_time / 1000 AS elapsed_seconds,
    bytes_scanned / (1024 * 1024 * 1024) AS gb_scanned,
    rows_produced,
    query_text
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time > DATEADD(hour, -24, CURRENT_TIMESTAMP())
  AND total_elapsed_time > 30000  -- queries > 30 seconds
ORDER BY total_elapsed_time DESC
LIMIT 20;
```

#### Check Query Profile

For a specific slow query:

1. Copy the `query_id` from the query above
2. Navigate to **Snowsight** → **Activity** → **Query History**
3. Find the query and open **Query Profile**
4. Look for:
    - **Bytes spilled to local/remote storage** → warehouse too small
    - **Partition pruning** → missing clustering or filters
    - **Exploding JOINs** → missing or incorrect JOIN conditions
    - **Full table scans** → add WHERE clauses or clustering keys

#### Check Warehouse Utilisation

```sql
-- Warehouse load over the last 7 days
SELECT
    warehouse_name,
    DATE_TRUNC('hour', start_time) AS hour,
    COUNT(*) AS query_count,
    AVG(total_elapsed_time) / 1000 AS avg_seconds,
    SUM(credits_used_compute) AS credits_used
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time > DATEADD(day, -7, CURRENT_TIMESTAMP())
GROUP BY warehouse_name, hour
ORDER BY credits_used DESC
LIMIT 50;
```

### 3. Fix Snowflake Performance Issues

=== "Warehouse Sizing"

    **Symptom:** Queries spilling to disk, high elapsed time relative to data volume.

    **Temporary fix** - scale up for a specific workload:

    ```sql
    USE ROLE SYSADMIN;
    ALTER WAREHOUSE TRANSFORMING SET WAREHOUSE_SIZE = 'MEDIUM';
    -- Run the workload
    ALTER WAREHOUSE TRANSFORMING SET WAREHOUSE_SIZE = 'X-SMALL';
    ```

    **Permanent fix** - update Terraform if the warehouse is consistently undersized:

    ```hcl
    module "warehouse_transforming" {
      source = "./modules/snowflake_warehouse"
      # ...
      warehouse_size = "SMALL"  # was X-SMALL
    }
    ```

    !!! warning "Cost Impact"
        Each size increase doubles the credit cost per second. A SMALL warehouse costs 2x an X-SMALL. Always scale back down after one-off workloads.

=== "Clustering Keys"

    **Symptom:** Full table scans on large tables, poor partition pruning in query profile.

    Add clustering keys to frequently filtered columns:

    ```sql
    -- For a table commonly filtered by date
    ALTER TABLE ANALYTICS.MARTS.FCT_EXCHANGE_RATES
      CLUSTER BY (rate_date);

    -- For a table commonly filtered by date and category
    ALTER TABLE ANALYTICS.MARTS.FCT_ORDERS
      CLUSTER BY (order_date, region);
    ```

    Or in dbt model config:

    ```sql
    {{
      config(
        materialized='incremental',
        cluster_by=['rate_date']
      )
    }}
    ```

    !!! info "When to Cluster"
        Clustering is only beneficial for tables larger than ~1GB. For smaller tables, the overhead of maintaining clusters outweighs the benefit.

=== "Query Optimisation"

    **Symptom:** Inefficient query patterns identified in query profile.

    Common fixes:

    - **Filter early**: Push WHERE clauses into CTEs rather than filtering at the end
    - **Reduce SELECT ***: Select only needed columns, especially from wide tables
    - **Avoid correlated subqueries**: Rewrite as JOINs or window functions
    - **Break complex CTEs**: Split into intermediate dbt models for reuse and caching

### 4. Optimise dbt Models

#### Identify Slow Models

```sh
# Run with timing output
dbt run --select <model>+ 2>&1 | grep "OK\|ERROR"
```

Or check the `run_results.json` after a run:

```sh
# Show models sorted by execution time
cat target/run_results.json | \
  python -c "import json,sys; results=json.load(sys.stdin)['results']; \
  [print(f'{r[\"execution_time\"]:.1f}s {r[\"unique_id\"]}') \
  for r in sorted(results, key=lambda x: x['execution_time'], reverse=True)[:10]]"
```

#### Common dbt Optimisations

| Issue | Fix |
|-------|-----|
| View rebuilds every query | Change to `table` or `incremental` materialisation |
| Full table rebuild daily | Switch to `incremental` with a reliable `updated_at` column |
| Slow incremental merge | Add `unique_key` and filter with `is_incremental()` |
| Unused intermediate models | Set to `ephemeral` materialisation |
| Large staging model scanned repeatedly | Materialise as `table` if used by 3+ downstream models |

#### Incremental Model Template

```sql
{{
  config(
    materialized='incremental',
    unique_key='order_id',
    on_schema_change='append_new_columns'
  )
}}

select *
from {{ ref('stg_dlt__orders') }}

{% if is_incremental() %}
  where loaded_at > (select max(loaded_at) from {{ this }})
{% endif %}
```

### 5. Optimise Pipeline Performance

#### dlt Pipelines

- **Use `merge` write disposition** instead of `replace` for incremental loads
- **Add pagination** to API sources to avoid memory issues
- **Increase `workers` parameter** for parallel loading:

    ```python
    pipeline.run(source(), workers=4)
    ```

#### Prefect Flows

- **Increase task concurrency** for independent tasks:

    ```python
    @flow(task_runner=ConcurrentTaskRunner())
    def my_flow():
        task_a.submit()
        task_b.submit()  # runs concurrently with task_a
    ```

- **Adjust retry backoff** if retries are too aggressive or too slow

### 6. Optimise Dashboard Performance

- **Pre-aggregate in dbt** - create reporting models that pre-compute common aggregations
- **Use Snowflake result caching** - identical queries return cached results (free, enabled by default)
- **Limit dashboard queries** - avoid SELECT * in dashboard queries, use specific columns
- **Consider materialised views** for real-time aggregation needs

## Verification

- [ ] Slow queries now run within acceptable time
- [ ] dbt run completes within the expected window
- [ ] Credit usage returns to normal levels:

    ```sql
    SELECT
        warehouse_name,
        SUM(credits_used) AS total_credits
    FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
    WHERE start_time > DATEADD(day, -7, CURRENT_TIMESTAMP())
    GROUP BY warehouse_name
    ORDER BY total_credits DESC;
    ```

- [ ] Dashboards load within acceptable response times
- [ ] No new alerts from resource monitors

## Rollback

Performance changes are generally safe to revert:

1. **Warehouse size changes** - scale back down via Terraform PR or SQL
2. **Clustering keys** - drop with `ALTER TABLE ... DROP CLUSTERING KEY`
3. **dbt materialisation changes** - revert the model config and run `dbt run --full-refresh`
4. **Model restructuring** - revert the PR in the dbt-transform repository

## Escalation

- **First contact**: Data Engineering team in #data-eng Slack
- **Escalation**: Snowflake account admin (for platform-level issues), dbt project owner (for model issues)

## See Also

- [Snowflake Monitoring](../build/observability/7-snowflake-monitoring.md) - Query history and performance dashboards
- [Cost Monitoring](../build/observability/8-cost-monitoring.md) - Credit usage tracking
- [Warehouses](../build/data-warehouse/2-warehouses.md) - Warehouse sizing guide
