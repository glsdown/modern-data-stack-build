# Runbook: Backfills

## Summary

Reprocess historical data when source corrections, schema changes, or logic updates require rebuilding pipeline output. This covers dlt pipeline backfills, Airbyte resyncs, Snowpipe replays, and dbt full refreshes.

## When to Use

- Source data was corrected retroactively and downstream tables are stale
- A pipeline schema change requires reloading historical data
- An incremental dbt model's logic changed and needs rebuilding
- A new data source is added and historical data must be loaded
- Data quality issues were found in previously loaded data

## Prerequisites

- [ ] Access: Prefect Cloud (to trigger runs), Snowflake (to verify data), relevant repository
- [ ] Context: Date range for the backfill, affected pipelines and models
- [ ] Timing: Schedule during low-usage hours - backfills consume significant compute

!!! warning "Coordinate Backfills"
    Large backfills can spike Snowflake credit usage and slow down other workloads. Notify the team before running backfills against production. Consider using a dedicated warehouse sized appropriately for the volume.

## Steps

### 1. Identify the Backfill Scope

Determine which layer needs backfilling:

| Symptom | Backfill Layer | Action |
|---------|---------------|--------|
| Source API data was corrected | **Ingestion** | Re-run the pipeline for affected dates |
| dlt/Airbyte schema changed | **Ingestion** | Full re-sync of the source |
| dbt incremental logic changed | **Transformation** | `dbt run --full-refresh` on affected models |
| New source added, need history | **Ingestion** | Run historical backfill flow |
| Downstream aggregations are wrong | **Transformation** | Rebuild affected models and downstream |

### 2. Backfill Ingestion Layer

=== "dlt Pipeline"

    Use the dedicated backfill flows created during the [Prefect Orchestration](../build/batch-data-ingestion/8-prefect-orchestration.md) setup.

    **Trigger via Prefect CLI:**

    ```sh
    prefect deployment run exchange-rates-backfill/production \
        --param start_date=2026-01-01 \
        --param end_date=2026-02-28
    ```

    **Or trigger via Prefect UI:**

    1. Navigate to **Deployments** → find the backfill deployment
    2. Click **Run** → **Custom**
    3. Set `start_date` and `end_date` parameters
    4. Click **Submit**

    For pipelines without a dedicated backfill flow, create a one-off flow or modify the pipeline's `write_disposition`:

    ```python
    # Temporarily use "replace" to reload all data
    pipeline.run(source(), write_disposition="replace")
    ```

    !!! danger "Replace vs Merge"
        Using `write_disposition="replace"` drops and recreates the table. For incremental sources, prefer `"merge"` with a primary key to upsert records without losing data that isn't in the backfill range.

=== "Airbyte"

    1. Navigate to **Airbyte** → **Connections** → select the connection
    2. Click the **Settings** tab
    3. Under **Sync mode**, click **Reset data** for the affected streams
    4. Click **Save** and then **Sync now**

    Airbyte will clear the destination table and reload from the source. For large sources, this can take significant time.

    !!! info "Partial Resyncs"
        Airbyte does not support date-range backfills natively. A reset reloads all data for the stream. If you need a partial backfill, consider using dlt for that specific load.

=== "Snowpipe"

    Snowpipe does not replay automatically. To backfill:

    1. Re-upload the source files to S3 with new filenames (Snowpipe tracks processed files)
    2. Or use `ALTER PIPE ... REFRESH` to force reprocessing:

        ```sql
        USE ROLE SYSADMIN;
        ALTER PIPE SNOWPIPE.OPEN_EXCHANGE_RATES.EXCHANGE_RATES_PIPE
          REFRESH
          PREFIX = 'exchange-rates/2026/01/';
        ```

    3. Verify the pipe status:

        ```sql
        SELECT SYSTEM$PIPE_STATUS('SNOWPIPE.OPEN_EXCHANGE_RATES.EXCHANGE_RATES_PIPE');
        ```

### 3. Backfill Transformation Layer (dbt)

After ingestion backfills complete, rebuild affected dbt models.

**Full refresh a single model:**

```sh
dbt run --select fct_exchange_rates --full-refresh
```

**Full refresh a model and everything downstream:**

```sh
dbt run --select fct_exchange_rates+ --full-refresh
```

**Full refresh all models from a source:**

```sh
dbt run --select source:dlt_open_exchange_rates+ --full-refresh
```

**Run tests after the backfill:**

```sh
dbt test --select fct_exchange_rates+
```

!!! warning "Incremental Models Only"
    `--full-refresh` only affects incremental models. Views and ephemeral models rebuild automatically. Tables without incremental config are always fully rebuilt.

### 4. Handle Large Backfills

For backfills that process significant data volumes:

1. **Use a larger warehouse temporarily:**

    ```sql
    USE ROLE SYSADMIN;
    ALTER WAREHOUSE TRANSFORMING SET WAREHOUSE_SIZE = 'LARGE';
    ```

2. **Run the backfill:**

    ```sh
    dbt run --select fct_exchange_rates+ --full-refresh
    ```

3. **Scale back down immediately after:**

    ```sql
    ALTER WAREHOUSE TRANSFORMING SET WAREHOUSE_SIZE = 'X-SMALL';
    ```

4. **Monitor credit consumption** in Snowflake → Admin → Cost Management during the backfill.

## Verification

- [ ] Pipeline run completed successfully in Prefect UI
- [ ] Row counts match expectations:

    ```sql
    SELECT COUNT(*), MIN(rate_date), MAX(rate_date)
    FROM <DATABASE>.<SCHEMA>.<TABLE>;
    ```

- [ ] No duplicate records (for merge-based pipelines):

    ```sql
    SELECT primary_key, COUNT(*)
    FROM <TABLE>
    GROUP BY primary_key
    HAVING COUNT(*) > 1;
    ```

- [ ] dbt tests pass on affected models:

    ```sh
    dbt test --select <model>+
    ```

- [ ] Source freshness check passes:

    ```sh
    dbt source freshness
    ```

- [ ] Downstream dashboards show correct data

## Rollback

If the backfill introduces data issues:

1. **Snowflake Time Travel** - restore tables to their pre-backfill state:

    ```sql
    -- Restore to state before backfill (within retention period, default 1 day)
    CREATE OR REPLACE TABLE <DATABASE>.<SCHEMA>.<TABLE>
      CLONE <DATABASE>.<SCHEMA>.<TABLE> AT (OFFSET => -3600);
    ```

2. **dbt incremental models** - Time Travel works here too:

    ```sql
    CREATE OR REPLACE TABLE ANALYTICS.MARTS.FCT_EXCHANGE_RATES
      CLONE ANALYTICS.MARTS.FCT_EXCHANGE_RATES AT (OFFSET => -3600);
    ```

3. **Re-run the normal incremental pipeline** to resume from the restored state

!!! info "Time Travel Retention"
    Default retention is 1 day (Standard edition) or up to 90 days (Enterprise edition). Check your account's `DATA_RETENTION_TIME_IN_DAYS` setting. For critical backfills, consider taking an explicit clone as a backup before starting.

## Escalation

- **First contact**: Data Engineering team in #data-eng Slack
- **Escalation**: Pipeline team lead (for ingestion), analytics engineering lead (for dbt)

## See Also

- [Prefect Orchestration](../build/batch-data-ingestion/8-prefect-orchestration.md) - Backfill flow patterns
- [dbt Testing and Documentation](../build/data-transformation/8-testing-and-documentation.md) - Test strategies
- [Snowflake Monitoring](../build/observability/7-snowflake-monitoring.md) - Credit usage monitoring
