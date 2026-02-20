# Finishing Up

On this page, you will:

- [x] Verify the complete transformation pipeline end-to-end
- [x] Review the full architecture you have built
- [x] Understand cost and operational considerations
- [x] Plan next steps

## Verification Checklist

### Snowflake Infrastructure

```sql
-- Service account exists
SHOW USERS LIKE 'SVC_DBT';
DESCRIBE USER SVC_DBT;
-- default_warehouse should be TRANSFORMING
-- default_role should be SVC_DBT

-- Role grants
SHOW GRANTS TO USER SVC_DBT;
-- Should include: SVC_DBT role, ANALYTICS_TRANSFORMER (inherited)

-- dbt-created schemas exist in ANALYTICS
USE DATABASE ANALYTICS;
SHOW SCHEMAS;
-- Should include: STAGING, INTERMEDIATE (if not ephemeral), MARTS, REPORTING

-- Models exist
USE SCHEMA MARTS;
SHOW TABLES;
-- Should include: FCT_EXCHANGE_RATES, DIM_CURRENCIES, FCT_CONTACTS

USE SCHEMA REPORTING;
SHOW VIEWS;
-- Should include: RPT_CONTACTS
```

### Role-Based Access

```sql
-- ANALYTICS_DEVELOPER can read MARTS
USE ROLE ANALYTICS_DEVELOPER;
SELECT COUNT(*) FROM ANALYTICS.MARTS.FCT_EXCHANGE_RATES;
SELECT COUNT(*) FROM ANALYTICS.MARTS.FCT_CONTACTS;

-- ANALYTICS_REPORTER can only read REPORTING
USE ROLE ANALYTICS_REPORTER;
SELECT COUNT(*) FROM ANALYTICS.REPORTING.RPT_CONTACTS;    -- should succeed
SELECT COUNT(*) FROM ANALYTICS.MARTS.FCT_CONTACTS;         -- should fail

-- SVC_DBT can write to ANALYTICS
USE ROLE SVC_DBT;
USE WAREHOUSE TRANSFORMING;
SELECT COUNT(*) FROM ANALYTICS.MARTS.FCT_EXCHANGE_RATES;   -- should succeed
```

### dbt Build

Run a full build locally to confirm everything is working:

```sh
cd dbt-transform
dbt deps
dbt source freshness
dbt build
```

Expected output:

```
Running with dbt=1.8.x

Found 8 models, 4 sources, 25 tests, 3 seeds

Concurrency: 4 threads

1 of 8 START sql view model ANALYTICS_DEV.STAGING.STG_DLT__EXCHANGE_RATES ........ [RUN]
1 of 8 OK created sql view model ANALYTICS_DEV.STAGING.STG_DLT__EXCHANGE_RATES .. [CREATE VIEW in 0.8s]
...

Finished running 8 models, 25 tests in 45.2s.

PASS=33 WARN=0 ERROR=0 SKIP=0 TOTAL=33
```

### CI/CD

=== "dbt Core"

    1. Create a branch in `dbt-transform`
    2. Make a small change to a model (e.g. add a column alias)
    3. Open a pull request
    4. Confirm the `dbt CI` workflow runs in GitHub Actions
    5. Check that only the changed model (and downstream) is built
    6. Merge the PR and confirm `dbt Production Deploy` runs successfully
    7. Confirm `manifest.json` is updated in S3
    8. Confirm the docs site is updated at the CloudFront URL

=== "dbt Cloud"

    1. Create a branch in the dbt Cloud IDE or locally
    2. Make a small change to a model
    3. Open a pull request
    4. Confirm the `CI - Pull Request` job runs in dbt Cloud
    5. Check the PR status check in GitHub
    6. Merge the PR and confirm the `Production - Daily` job runs (or trigger it manually)
    7. Confirm the docs site is updated in dbt Cloud

### Prefect End-to-End

Trigger the full daily pipeline manually and confirm it runs ingestion followed by transformation:

```sh
prefect deployment run daily-pipeline/production
```

In the Prefect UI, you should see:

- `exchange_rates_flow` — Completed
- `hubspot_airbyte_flow` — Completed
- `dbt-transformation` (or `dbt-cloud-transformation`) — Completed

Check that the mart tables were updated after the dbt run:

```sql
SELECT MAX(loaded_at) FROM ANALYTICS.MARTS.FCT_EXCHANGE_RATES;
-- Should be today
```

## Architecture Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    COMPLETE DATA TRANSFORMATION STACK                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Source Systems                                                             │
│  ──────────────                                                             │
│  Open Exchange Rates API ──▶ dlt ──▶ DLT.OPEN_EXCHANGE_RATES                │
│  Open Exchange Rates API ──▶ dlt ──▶ S3 ──▶ Snowpipe ──▶ SNOWPIPE.*         │
│  HubSpot CRM ──▶ Airbyte ──────────────────────────────▶ AIRBYTE.HUBSPOT    │
│                                                                             │
│  Transformation (dbt-transform repo)                                        │
│  ───────────────────────────────────                                        │
│                                                                             │
│  STAGING             INTERMEDIATE         MARTS           REPORTING         │
│  ───────             ────────────         ─────           ─────────         │
│  stg_dlt__*    ───▶  int_exchange         fct_exchange    rpt_contacts      │
│  stg_snowpipe__* ──▶   _rates__unioned ─▶   _rates                          │
│  stg_airbyte__* ──▶  int_contacts__    ─▶ fct_contacts ─▶ (ANALYTICS_       │
│                         enriched          dim_currencies    REPORTER        │
│                                                             access)         │
│                                                                             │
│  Orchestration (data-pipelines repo)                                        │
│  ────────────────────────────────────                                       │
│  Prefect daily pipeline                                                     │
│  ├── Ingestion: dlt + Airbyte flows (parallel)                              │
│  └── Transformation: dbt Core CLI or dbt Cloud API                          │
│                                                                             │
│  CI/CD                                                                      │
│  ─────                                                                      │
│  dbt-transform PRs ──▶ slim CI (changed models only, defer to prod)         │
│  dbt-transform main ──▶ full build + manifest → S3 + docs → CloudFront      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Repository Summary

| Repository | Contents | CI/CD trigger |
|------------|----------|--------------|
| `terraform` | All infrastructure as code | Push to main → Terraform plan/apply |
| `data-pipelines` | Prefect flows (ingestion + orchestration) | Push to main → `prefect deploy --all` |
| `dbt-transform` | dbt models, tests, docs | PR → slim CI; push to main → full build |

## Cost Summary

| Component | dbt Core | dbt Cloud |
|-----------|----------|-----------|
| dbt licence | Free | Free (1 seat) / $100/seat/month (Team) |
| S3 artifacts bucket | ~$0.01/month | Not needed |
| CloudFront + S3 docs | ~$1-5/month | Not needed (hosted by dbt Cloud) |
| Additional Snowflake compute | ~$5-20/month (TRANSFORMING warehouse) | Same |
| GitHub Actions minutes | Included in most plans | Less usage (dbt Cloud runs jobs) |
| **Total additional** | **~$6-25/month** | **$0-100+/month** |

The Snowflake `TRANSFORMING` warehouse is the main variable cost. With auto-suspend (60 seconds recommended) and typical daily batch runs, expect 1-2 credits per day for small to medium data volumes.

## Common Operations

### Backfill a Model

```sh
# Re-run a model and all downstream models
dbt build --select fct_exchange_rates+ --full-refresh
```

### Add a New Source

1. Add the source definition to the relevant `sources.yml`
2. Create a staging model in the appropriate `models/staging/{source}/` directory
3. Add tests and a description to the model YAML
4. Run `dbt_project_evaluator` to check for violations
5. Create intermediate and mart models as needed
6. Open a pull request — CI will build and test the new models

### Debug a Failing Test

```sh
# Compile the test SQL to inspect it
dbt test --select fct_exchange_rates --store-failures

# Check the stored failures table
SELECT * FROM ANALYTICS_DEV.DBT_TEST__AUDIT.FCT_EXCHANGE_RATES_EXCHANGE_RATE_NOT_NULL;
```

### Update a Model's Materialisation

To change a view to a table (e.g. for a frequently-queried staging model):

```yaml
# In the model's .yml file or dbt_project.yml
models:
  - name: stg_dlt__exchange_rates
    config:
      materialised: table
```

## Troubleshooting

### `Object 'ANALYTICS.STAGING.STG_DBT__*' does not exist`

dbt-created schemas in `ANALYTICS` are lowercase in the compiled SQL but uppercase in Snowflake. Confirm the `generate_schema_name` macro is in place and returning uppercase schema names. Check with:

```sh
dbt compile --select stg_dlt__exchange_rates
cat target/compiled/dbt_transform/models/staging/dlt/stg_dlt__exchange_rates.sql
```

### `Insufficient privileges to operate on schema 'STAGING'`

The `SVC_DBT` user does not have `CREATE SCHEMA` privileges on `ANALYTICS`. Check:

```sql
SHOW GRANTS TO ROLE SVC_DBT;
SHOW GRANTS TO ROLE ANALYTICS_TRANSFORMER;
```

`ANALYTICS_TRANSFORMER` should include `CREATE SCHEMA ON DATABASE ANALYTICS`. If missing, this was not set in the database module's grants — check `terraform/snowflake/modules/snowflake_database/main.tf`.

### `Git clone failed` (dbt Core Prefect flow)

The GitHub token stored in `terraform/github-token` in Secrets Manager may have expired or have insufficient permissions. The token needs `repo` scope (read access) to clone `dbt-transform`.

## What You've Built

- [x] **`SVC_DBT` service account** with dedicated role and `ANALYTICS_TRANSFORMER` access
- [x] **Staging models** — clean views over DLT, Snowpipe, and Airbyte raw tables
- [x] **Intermediate models** — business logic layer with cross-source joins
- [x] **Mart models** — final analytics tables (`fct_*`, `dim_*`) in `ANALYTICS.MARTS`
- [x] **Reporting models** — curated subset in `ANALYTICS.REPORTING` for BI access
- [x] **Tests** — generic and singular tests, `dbt_expectations`, `dbt_project_evaluator`
- [x] **CI/CD** — slim CI on PRs, full production build on merge to main
- [x] **State deferral** — fast local development against production data
- [x] **Docs site** — hosted and auto-updated after each production run
- [x] **Prefect integration** — dbt triggered after ingestion completes

## What's Next

With clean, tested, analytics-ready data in Snowflake, the next steps are:

1. **Connect a BI tool**: Point Metabase or another tool at `ANALYTICS.REPORTING` using a read-only connection via `ANALYTICS_REPORTER`

2. **Add more models**: Follow the staging → intermediate → marts pattern for each new data source

3. **Observability**: Set up OpenMetadata or similar for data catalogue, lineage, and quality monitoring

### Future Documentation

- `build/data-analytics/` — BI tool setup and dashboard development
- `build/observability/` — Data quality monitoring and alerting

## Feedback

Found an issue or have suggestions? Open an issue in the repository.
