# Finishing Up

On this page, you will:

- [x] Verify all pipelines are running correctly
- [x] Review monitoring and observability options
- [x] Understand common operational tasks
- [x] Plan next steps

## Verification Checklist

### Snowflake Infrastructure

Verify the databases, schemas, and roles exist:

```sql
-- Databases
SHOW DATABASES LIKE 'DLT';
SHOW DATABASES LIKE 'SNOWPIPE';

-- Schemas
SHOW SCHEMAS IN DATABASE DLT;
-- Should show: OPEN_EXCHANGE_RATES, APPLICATION_DATA, HUBSPOT

SHOW SCHEMAS IN DATABASE SNOWPIPE;
-- Should show: OPEN_EXCHANGE_RATES

-- Dedicated role
SHOW ROLES LIKE 'SVC_DLT';

-- Service account
SHOW USERS LIKE 'SVC_DLT';
DESCRIBE USER SVC_DLT;
-- default_role should be SVC_DLT

-- Snowpipe
SHOW PIPES IN DATABASE SNOWPIPE;
```

### Data in Snowflake

Verify data has been loaded:

```sql
-- Currencies (direct to Snowflake)
SELECT COUNT(*)
FROM DLT.OPEN_EXCHANGE_RATES.CURRENCIES;

-- Exchange rates (via Snowpipe)
SELECT COUNT(*), MIN(date), MAX(date)
FROM SNOWPIPE.OPEN_EXCHANGE_RATES.EXCHANGE_RATES;

-- Products
SELECT COUNT(*), MIN(valid_ts), MAX(valid_ts)
FROM DLT.APPLICATION_DATA.PRODUCTS;
```

### Prefect Deployments

Verify deployments are active:

```sh
# List deployments
prefect deployment ls

# Check schedules
prefect deployment inspect exchange-rates-daily/production
```

Should show:

- `exchange-rates-daily` — Scheduled daily at 09:00 UTC
- `currencies-weekly` — Scheduled Sundays at 00:00 UTC
- `products-daily` — Scheduled daily at 06:00 UTC

### Recent Flow Runs

Check recent runs completed successfully:

```sh
prefect flow-run ls --limit 10
```

## Architecture Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      BATCH DATA INGESTION                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Data Sources              Pipelines                  Destinations          │
│  ─────────────              ─────────                  ────────────         │
│                                                                             │
│  ┌─────────────┐     ┌─────────────────────┐     ┌─────────────────────┐    │
│  │ Open Exch.  │     │                     │     │ DLT.OPEN_EXCHANGE_  │    │
│  │ Rates API   │────▶│  currencies.py      │────▶│ RATES.CURRENCIES    │    │
│  │ /currencies │     │ (Weekly Sun 00:00)  │     │                     │    │
│  └─────────────┘     └─────────────────────┘     └─────────────────────┘    │
│                                                                             │
│  ┌─────────────┐     ┌─────────────────────┐     ┌─────────────────────┐    │
│  │ Open Exch.  │     │                     │     │     S3 Bucket       │    │
│  │ Rates API   │────▶│  exchange_rates.py  │────▶│  /exchange-rates/   │    │
│  │ /latest     │     │  (Daily 09:00 UTC)  │     └──────────┬──────────┘    │
│  └─────────────┘     └─────────────────────┘                │               │
│                                                             ▼               │
│                                                  ┌─────────────────────┐    │
│                                                  │     Snowpipe        │    │
│                                                  │   (auto-ingest)     │    │
│                                                  └──────────┬──────────┘    │
│                                                             ▼               │
│                                                  ┌─────────────────────┐    │
│                                                  │ SNOWPIPE.OPEN_      │    │
│                                                  │ EXCHANGE_RATES.     │    │
│                                                  │ EXCHANGE_RATES      │    │
│                                                  └─────────────────────┘    │
│                                                                             │
│  ┌─────────────┐     ┌─────────────────────┐     ┌─────────────────────┐    │
│  │ Clever Cloud│     │                     │     │ DLT.APPLICATION_    │    │
│  │ PostgreSQL  │────▶│  products.py        │────▶│ DATA.PRODUCTS       │    │
│  │ products    │     │  (Daily 06:00 UTC)  │     │                     │    │
│  └─────────────┘     └─────────────────────┘     └─────────────────────┘    │
│                                                                             │
│                      Orchestration                                          │
│                      ─────────────                                          │
│                      ┌─────────────────────┐                                │
│                      │  Prefect Cloud      │                                │
│                      │  • Schedules        │                                │
│                      │  • Retries          │                                │
│                      │  • Alerting         │                                │
│                      └─────────────────────┘                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Monitoring

### Prefect Cloud Dashboard

The Prefect Cloud dashboard provides:

- **Flow run history**: Success/failure trends over time
- **Task-level visibility**: Which tasks failed and why
- **Logs**: Searchable logs from all runs
- **Alerts**: Notification of failures (see [Alerting](../orchestration/10-alerting.md))

### Snowflake Query History

Monitor pipeline activity in Snowflake:

```sql
-- Recent queries by SVC_DLT
SELECT
    query_id,
    query_text,
    start_time,
    total_elapsed_time / 1000 as seconds,
    rows_inserted,
    bytes_scanned
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE user_name = 'SVC_DLT'
    AND start_time > DATEADD(day, -7, CURRENT_TIMESTAMP())
ORDER BY start_time DESC
LIMIT 20;

-- Snowpipe copy history
SELECT *
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME => 'SNOWPIPE.OPEN_EXCHANGE_RATES.EXCHANGE_RATES',
    START_TIME => DATEADD(day, -7, CURRENT_TIMESTAMP())
));
```

## Common Operations

### Trigger Manual Run

```sh
# Run immediately
prefect deployment run exchange-rates-daily/production

# Run with custom parameters
prefect deployment run exchange-rates-backfill/production \
    --param start_date=2026-02-01 \
    --param end_date=2026-02-07
```

### Pause/Resume Schedules

```sh
# Pause
prefect deployment pause exchange-rates-daily/production

# Resume
prefect deployment resume exchange-rates-daily/production
```

### View Logs

```sh
# Get run ID from recent runs
prefect flow-run ls --limit 5

# View logs
prefect flow-run logs <run-id>
```

## Troubleshooting

### Pipeline Fails with Authentication Error

**Symptoms**: `Snowflake authentication failed` or `Invalid credentials`

**Check**:
1. AWS Secrets Manager has correct values
2. SVC_DLT user has RSA public key set in Snowflake
3. Worker has IAM permissions to read `dlt/*` secrets

### Snowpipe Not Ingesting

**Symptoms**: Files in S3 but no data in Snowflake

**Check**:
1. Pipe is running (not paused)
2. S3 event notifications configured for `exchange-rates/` prefix
3. Target table exists

```sql
SELECT SYSTEM$PIPE_STATUS('SNOWPIPE.OPEN_EXCHANGE_RATES.EXCHANGE_RATES_PIPE');
```

### Flow Stuck in Pending

**Symptoms**: Flow run stays in `Pending` state

**Check**:
1. Worker is running and connected to work pool
2. Work pool has capacity

```sh
prefect worker ls
prefect work-pool ls
```

## Cost Summary

| Component | Monthly Cost |
|-----------|-------------|
| dlt | Free (open source) |
| Open Exchange Rates | Free tier (1,000 req/month) |
| Clever Cloud PostgreSQL | Free (DEV tier) |
| Snowpipe | ~$0.06 per 1,000 files |
| S3 storage | ~$0.023 per GB |
| SQS notifications | Free tier |
| Prefect Cloud | Free tier or $100/mo starter |

Total incremental cost for these pipelines: **< $1/month** (assuming Prefect Cloud free tier)

## Summary

Congratulations! You've built a complete batch data ingestion system:

- [x] **DLT database** for dlt-loaded data (currencies, products)
- [x] **SNOWPIPE database** for auto-ingested data (exchange rates)
- [x] **SVC_DLT** service account with dedicated role
- [x] **Currencies pipeline** (API → Snowflake direct)
- [x] **Exchange rates pipeline** (API → S3 → Snowpipe → Snowflake)
- [x] **Products pipeline** (PostgreSQL → Snowflake)
- [x] **Prefect orchestration** with schedules and retries
- [x] **AWS Secrets Manager** integration via VaultDocProvider

## What's Next

The next page demonstrates adding a SaaS connector (HubSpot) using dlt's verified source — a pattern for when you need one or two SaaS sources without the overhead of Airbyte.

Continue to [HubSpot Pipeline](9-hubspot-pipeline.md) →
