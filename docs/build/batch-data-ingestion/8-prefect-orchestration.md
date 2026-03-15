# Prefect Orchestration

On this page, you will:

- [x] Wrap dlt pipelines in Prefect tasks and flows
- [x] Configure retries with exponential backoff and jitter
- [x] Create deployment configurations with schedules
- [x] Deploy flows to Prefect Cloud

## Overview

With dlt pipelines built, you need orchestration to run them on schedule, handle failures, and alert when things go wrong. Prefect provides this orchestration layer.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        PREFECT FLOWS                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────┐   ┌─────────────────────┐   ┌─────────────────────┐│
│  │ exchange_rates_daily│   │  currencies_weekly  │   │   products_daily    ││
│  │                     │   │                     │   │                     ││
│  │ Schedule: 09:00 UTC │   │ Schedule: Sun 00:00 │   │ Schedule: 06:00 UTC ││
│  │ Retries: 3          │   │ Retries: 3          │   │ Retries: 3          ││
│  │ Backoff: exp(10s)   │   │ Backoff: exp(10s)   │   │ Backoff: exp(10s)   ││
│  └──────────┬──────────┘   └──────────┬──────────┘   └──────────┬──────────┘│
│             │                         │                         │           │
│             ▼                         ▼                         ▼           │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                      dlt Pipelines (pipelines/)                        ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│             │                         │                         │           │
│             ▼                         ▼                         ▼           │
│  ┌───────────────────┐     ┌───────────────────┐     ┌───────────────────┐  │
│  │   S3 → Snowpipe   │     │     Snowflake     │     │     Snowflake     │  │
│  │   → Snowflake     │     │  DLT.OPEN_EXCH... │     │  DLT.APPLICATION_ │  │
│  └───────────────────┘     │  .CURRENCIES      │     │  DATA.PRODUCTS    │  │
│                            └───────────────────┘     └───────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Prerequisites

Ensure you have completed:

- [x] [Prefect Setup](../orchestration/index.md) — Prefect Cloud or self-hosted configured
- [x] [First Flow](../orchestration/7-first-flow.md) — Repository and basic flow working
- [x] All three dlt pipelines (currencies, exchange rates, products)

## Task/Flow Architecture

The project separates concerns into three layers:

| Layer | Directory | Responsibility |
|-------|-----------|---------------|
| **Sources** | `sources/` | Extract data from APIs and databases |
| **Pipelines** | `pipelines/` | Configure dlt pipeline (destination, dataset, credentials) |
| **Flows** | `flows/` | Orchestrate pipelines with Prefect (schedules, retries, logging) |

Flows import from pipelines, and pipelines import from sources. This keeps each layer focused:

```python
# flows/currencies.py imports from pipelines/currencies.py
from pipelines.currencies import run

# pipelines/currencies.py imports from sources/currencies/
from sources.currencies import currencies
```

## Retries with Exponential Backoff

All tasks use exponential backoff with jitter. Here's what that means in practice:

```python
@task(
    retries=3,
    retry_delay_seconds=exponential_backoff(backoff_factor=10),
    retry_jitter_factor=0.5,
)
```

| Retry | Base Delay | With Jitter (±50%) | Typical Wait |
|-------|-----------|-------------------|-------------|
| 1 | 10 seconds | 5–15 seconds | ~10s |
| 2 | 100 seconds | 50–150 seconds | ~2 min |
| 3 | 1,000 seconds | 500–1,500 seconds | ~17 min |

The exponential increase gives transient failures (API rate limits, network blips) time to recover. Jitter adds randomisation so that if multiple pipelines fail simultaneously, they don't all retry at the same instant (the "thundering herd" problem).

## Exchange Rates Flow

Create `flows/exchange_rates.py`:

```python
"""Prefect flow for exchange rates pipeline."""

from datetime import date, datetime

from prefect import flow, task, get_run_logger
from prefect.tasks import exponential_backoff


@task(
    name="load-exchange-rates",
    retries=3,
    retry_delay_seconds=exponential_backoff(backoff_factor=10),
    retry_jitter_factor=0.5,
)
def load_exchange_rates_task():
    """Load latest exchange rates to S3 (Snowpipe auto-ingests to Snowflake)."""
    logger = get_run_logger()
    logger.info("Starting exchange rates load to S3")

    from pipelines.exchange_rates import run_latest

    load_info = run_latest()
    logger.info(f"Load completed: {load_info}")

    return load_info


@task(
    name="backfill-exchange-rates",
    retries=3,
    retry_delay_seconds=exponential_backoff(backoff_factor=10),
    retry_jitter_factor=0.5,
)
def backfill_exchange_rates_task(start_date: date, end_date: date | None = None):
    """Backfill historical exchange rates to S3."""
    logger = get_run_logger()
    logger.info(f"Starting backfill from {start_date} to {end_date or 'yesterday'}")

    from pipelines.exchange_rates import run_backfill

    load_info = run_backfill(start_date, end_date)
    logger.info(f"Backfill completed: {load_info}")

    return load_info


@flow(name="exchange-rates-daily", log_prints=True)
def exchange_rates_daily_flow():
    """
    Daily flow to load latest exchange rates.

    Writes to S3; Snowpipe auto-ingests to
    SNOWPIPE.OPEN_EXCHANGE_RATES.EXCHANGE_RATES.
    """
    load_info = load_exchange_rates_task()
    print(f"Exchange rates loaded to S3: {load_info}")
    return {"status": "success", "load_info": str(load_info)}


@flow(name="exchange-rates-backfill", log_prints=True)
def exchange_rates_backfill_flow(
    start_date: str = "2026-01-01",
    end_date: str | None = None,
):
    """One-time flow to backfill historical exchange rates."""
    start = datetime.strptime(start_date, "%Y-%m-%d").date()
    end = datetime.strptime(end_date, "%Y-%m-%d").date() if end_date else None

    load_info = backfill_exchange_rates_task(start, end)
    print(f"Backfill completed: {load_info}")
    return {"status": "success", "load_info": str(load_info)}


if __name__ == "__main__":
    exchange_rates_daily_flow()
```

## Currencies Flow

Create `flows/currencies.py`:

```python
"""Prefect flow for currencies pipeline."""

from prefect import flow, task, get_run_logger
from prefect.tasks import exponential_backoff


@task(
    name="load-currencies",
    retries=3,
    retry_delay_seconds=exponential_backoff(backoff_factor=10),
    retry_jitter_factor=0.5,
)
def load_currencies_task():
    """Load currencies from Open Exchange Rates API to Snowflake."""
    logger = get_run_logger()
    logger.info("Starting currencies load to Snowflake")

    from pipelines.currencies import run

    load_info = run()
    logger.info(f"Load completed: {load_info}")

    return load_info


@flow(name="currencies-weekly", log_prints=True)
def currencies_weekly_flow():
    """
    Weekly flow to refresh currencies reference data.

    Loads directly to DLT.OPEN_EXCHANGE_RATES.CURRENCIES.
    """
    load_info = load_currencies_task()
    print(f"Currencies loaded to Snowflake: {load_info}")
    return {"status": "success", "load_info": str(load_info)}


if __name__ == "__main__":
    currencies_weekly_flow()
```

## Products Flow

Create `flows/products.py`:

```python
"""Prefect flow for products pipeline."""

from prefect import flow, task, get_run_logger
from prefect.tasks import exponential_backoff


@task(
    name="load-products",
    retries=3,
    retry_delay_seconds=exponential_backoff(backoff_factor=10),
    retry_jitter_factor=0.5,
)
def load_products_task(full_refresh: bool = False):
    """Load products from PostgreSQL to Snowflake (incremental)."""
    logger = get_run_logger()
    logger.info(f"Starting products load (full_refresh={full_refresh})")

    from pipelines.products import run

    load_info = run(full_refresh=full_refresh)
    logger.info(f"Load completed: {load_info}")

    return load_info


@flow(name="products-daily", log_prints=True)
def products_daily_flow():
    """Daily incremental products load from PostgreSQL."""
    load_info = load_products_task(full_refresh=False)
    print(f"Products loaded successfully: {load_info}")
    return {"status": "success", "load_info": str(load_info)}


@flow(name="products-full-refresh", log_prints=True)
def products_full_refresh_flow():
    """Full refresh of products (use sparingly)."""
    load_info = load_products_task(full_refresh=True)
    print(f"Products full refresh completed: {load_info}")
    return {"status": "success", "load_info": str(load_info)}


if __name__ == "__main__":
    products_daily_flow()
```

## Prefect Deployment Configuration

Update `prefect.yaml` with all deployments:

```yaml
# Prefect deployment configuration for data pipelines
name: data-pipelines

# Pull section - how to retrieve flow code
pull:
  - prefect.deployments.steps.git_clone:
      repository: https://github.com/YOUR-ORG/data-pipelines.git
      branch: main

# Deployments
deployments:
  # =========================================================================
  # Exchange Rates (→ S3 → Snowpipe → Snowflake)
  # =========================================================================
  - name: exchange-rates-daily
    entrypoint: flows/exchange_rates.py:exchange_rates_daily_flow
    description: "Daily exchange rates from Open Exchange Rates API to S3"
    work_pool:
      name: production
    schedules:
      - cron: "0 9 * * *"
        timezone: "UTC"
        active: true
    tags:
      - dlt
      - exchange-rates
      - daily

  - name: exchange-rates-backfill
    entrypoint: flows/exchange_rates.py:exchange_rates_backfill_flow
    description: "Backfill historical exchange rates"
    work_pool:
      name: production
    parameters:
      start_date: "2026-01-01"
      end_date: null
    tags:
      - dlt
      - exchange-rates
      - backfill

  # =========================================================================
  # Currencies (→ Snowflake direct)
  # =========================================================================
  - name: currencies-weekly
    entrypoint: flows/currencies.py:currencies_weekly_flow
    description: "Weekly currencies refresh (direct to Snowflake)"
    work_pool:
      name: production
    schedules:
      - cron: "0 0 * * 0"  # Sunday at midnight UTC
        timezone: "UTC"
        active: true
    tags:
      - dlt
      - currencies
      - weekly

  # =========================================================================
  # Products (→ Snowflake direct)
  # =========================================================================
  - name: products-daily
    entrypoint: flows/products.py:products_daily_flow
    description: "Daily incremental products load from PostgreSQL"
    work_pool:
      name: production
    schedules:
      - cron: "0 6 * * *"
        timezone: "UTC"
        active: true
    tags:
      - dlt
      - products
      - daily

  - name: products-full-refresh
    entrypoint: flows/products.py:products_full_refresh_flow
    description: "Full refresh of products (use sparingly)"
    work_pool:
      name: production
    tags:
      - dlt
      - products
      - maintenance
```

## Alerting

Rather than adding notification hooks to each flow individually, use Prefect automations to centralise alerting. The `dlt` tag on all deployments makes this straightforward — a single automation catches failures across all dlt pipelines.

See [Alerting](../orchestration/10-alerting.md) for the full Slack and PagerDuty setup.

## Deploy Flows

Deploy all flows to Prefect:

```sh
cd ~/projects/data/data-pipelines

# Commit your changes first
git add .
git commit -m "Add Prefect flows for dlt pipelines"
git push

# Deploy all flows
prefect deploy --all
```

For automated deployment via CI/CD (recommended for production), see [CI/CD Deployment](../orchestration/9-ci-cd-deployment.md).

## Test Flows

### Run Manually

```sh
# Test exchange rates
prefect deployment run exchange-rates-daily/production

# Test with parameters
prefect deployment run exchange-rates-backfill/production \
    --param start_date=2026-01-01 \
    --param end_date=2026-01-07

# Test currencies
prefect deployment run currencies-weekly/production

# Test products
prefect deployment run products-daily/production
```

### Check Status

```sh
# List recent runs
prefect flow-run ls --limit 5

# View specific run logs
prefect flow-run logs <run-id>
```

## Schedule Summary

| Flow | Schedule | Destination |
|------|----------|-------------|
| `exchange-rates-daily` | Daily 09:00 UTC | S3 → Snowpipe → Snowflake |
| `currencies-weekly` | Sunday 00:00 UTC | Snowflake (direct) |
| `products-daily` | Daily 06:00 UTC | Snowflake (direct) |

Off-schedule flows (run manually):

| Flow | Purpose |
|------|---------|
| `exchange-rates-backfill` | Historical data backfill |
| `products-full-refresh` | Full reload of products |

## Summary

You've configured Prefect orchestration for dlt pipelines:

- [x] Created Prefect flows wrapping each dlt pipeline
- [x] Configured retries with exponential backoff and jitter
- [x] Defined deployment configurations with schedules and tags
- [x] Deployed all flows to Prefect Cloud

## What's Next

With orchestration complete, verify everything works end-to-end and review monitoring options.

Continue to [Finishing Up](10-finishing-up.md) →
