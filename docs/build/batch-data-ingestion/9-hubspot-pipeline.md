# HubSpot Pipeline

On this page, you will:

- [x] Understand when to use dlt vs Airbyte for SaaS data
- [x] Build a dlt pipeline using the verified HubSpot source
- [x] Load HubSpot contacts into Snowflake
- [x] Add the HubSpot flow to Prefect orchestration

## Overview

The HubSpot pipeline extracts CRM contacts from the [HubSpot API](https://developers.hubspot.com/docs/api/crm/contacts) using dlt's [verified HubSpot source](https://dlthub.com/docs/dlt-ecosystem/verified-sources/hubspot). This demonstrates how to add a SaaS connector to the existing dlt infrastructure.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       HUBSPOT PIPELINE                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐        ┌─────────────────┐        ┌─────────────────┐  │
│  │ HubSpot CRM     │  dlt   │    Snowflake    │        │                 │  │
│  │ API             │ ────▶  │ DLT.HUBSPOT     │        │  Daily 07:00    │  │
│  │ /contacts       │        │ .CONTACTS       │        │  Incremental    │  │
│  │                 │        │                 │        │                 │  │
│  └─────────────────┘        └─────────────────┘        └─────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

!!! warning "This page covers an alternative approach - not the canonical pipeline for this guide"
    The complete guide includes a dedicated [SaaS Ingestion](../saas-ingestion/index.md) section that uses Airbyte to load HubSpot contacts into the `AIRBYTE` database. The dbt `dim_customers` mart model reads from that Airbyte source. **If you are following the full guide, skip this page** - your canonical HubSpot pipeline is in the SaaS Ingestion section.

    This page is for teams that want to ingest HubSpot contacts via dlt instead of Airbyte - for example, if you are not setting up Airbyte at all. Do not run both in the same environment, as you would have the same CRM data duplicated across two databases with no single source of truth.

## Why dlt for HubSpot?

dlt has a [verified HubSpot source](https://dlthub.com/docs/dlt-ecosystem/verified-sources/hubspot) that handles authentication, pagination, and incremental loading out of the box. For a single SaaS source like HubSpot contacts, dlt is the right choice because it uses the existing infrastructure and costs nothing extra.

| Consideration | dlt | Airbyte |
|---------------|-----|---------|
| **Cost** | Free (open source) | $99+/month (Cloud) or ~$81/month (self-hosted) |
| **Infrastructure** | Uses existing SVC_DLT role, DLT database | Requires new database, service account, Airbyte deployment |
| **Setup** | Add source + pipeline files | Deploy Airbyte Cloud/self-hosted, configure via UI |
| **Best for** | 1-2 SaaS sources with verified dlt sources | 5+ SaaS sources, reverse ETL, non-engineer configuration |

!!! tip "When to Consider Airbyte"
    If you need **reverse ETL** (writing data back to SaaS tools), **5+ SaaS connectors**, or **non-engineers managing connections** via a UI, consider Airbyte. See the [SaaS Ingestion](../saas-ingestion/index.md) section for the full Airbyte setup.

## Prerequisites

Ensure you have:

- [x] [Snowflake Infrastructure](3-snowflake-infrastructure.md) — DLT.HUBSPOT schema created
- [x] [Credentials Setup](4-credentials-setup.md) — HubSpot API key secret container created
- [x] [Prefect Orchestration](8-prefect-orchestration.md) — Prefect flows deployed
- [x] HubSpot account (free CRM tier available at [hubspot.com](https://www.hubspot.com/))

## HubSpot Private App Setup

HubSpot uses private apps for API authentication (API keys were deprecated in November 2022).

1. Go to **Settings** > **Integrations** > **Private Apps** in HubSpot
2. Click **Create a private app**
3. Name it `dlt-pipeline`
4. Under **Scopes**, enable:
   - `crm.objects.contacts.read`
5. Click **Create app** and copy the access token

### Generate Test Data

If your HubSpot account is empty, generate test contacts using [Mockaroo](https://www.mockaroo.com/):

1. Go to [mockaroo.com](https://www.mockaroo.com/)
2. Create a schema with fields: `First Name`, `Last Name`, `Email`, `Company Name`, `Phone`
3. Download as CSV (100 rows is enough for testing)
4. In HubSpot, go to **Contacts** > **Import** > upload the CSV

This gives you realistic test data without connecting to a real CRM.

### Store the API Key

Set the secret value in the container created in [Snowflake Infrastructure](3-snowflake-infrastructure.md):

```sh
aws secretsmanager put-secret-value \
    --secret-id "dlt/hubspot-api-key" \
    --secret-string '{"api_key": "YOUR_ACCESS_TOKEN_HERE"}' \
    --profile infrastructure-admin
```

Update `.dlt/secrets.toml` for local development:

```toml
[sources.hubspot]
api_key = "your-private-app-access-token-here"
```

## Install the HubSpot Source

```sh
cd ~/projects/data/data-pipelines
uv add "dlt[hubspot]"
```

## The dlt Source

dlt provides a [verified HubSpot source](https://dlthub.com/docs/dlt-ecosystem/verified-sources/hubspot) that handles pagination, rate limiting, and incremental loading automatically.

Create `sources/hubspot/__init__.py`:

```python
"""HubSpot CRM source."""
from .source import hubspot_contacts

__all__ = ["hubspot_contacts"]
```

Create `sources/hubspot/source.py`:

```python
"""dlt source for HubSpot CRM data (contacts only)."""

import dlt
from dlt.sources.hubspot import hubspot


@dlt.source(section="hubspot")
def hubspot_contacts(api_key: str = dlt.secrets.value):
    """
    Source for HubSpot CRM contacts.

    Uses dlt's verified HubSpot source, filtered to contacts only.
    The verified source handles:
    - Pagination (HubSpot paginates at 100 records)
    - Rate limiting (respects HubSpot API limits)
    - Incremental loading (tracks last modified date)

    Args:
        api_key: HubSpot private app access token (auto-resolved by dlt)
    """
    source = hubspot(api_key=api_key)
    return source.with_resources("contacts")
```

### Available Resources

The verified HubSpot source supports these resources, though we only use `contacts`:

| Resource | Description |
|----------|-------------|
| `contacts` | Visitors, leads, and customers |
| `companies` | Organisation records |
| `deals` | Deal/opportunity records |
| `tickets` | Support ticket records |
| `products` | Product catalogue |
| `quotes` | Price proposals |

To add more resources later, update the `with_resources()` call:

```python
source.with_resources("contacts", "companies", "deals")
```

## The dlt Pipeline

Create `pipelines/hubspot.py`:

```python
"""HubSpot pipeline: HubSpot CRM API → Snowflake."""

import dlt

from sources.hubspot import hubspot_contacts
from utils.vault_provider import register_aws_secrets


def create_pipeline() -> dlt.Pipeline:
    """Create the HubSpot pipeline with Snowflake destination."""

    register_aws_secrets()

    pipeline = dlt.pipeline(
        pipeline_name="hubspot",
        destination="snowflake",
        dataset_name="HUBSPOT",  # Schema in DLT database
    )

    return pipeline


def run():
    """
    Run the HubSpot contacts pipeline.

    The verified source handles incremental loading automatically,
    only fetching contacts modified since the last run.
    """
    pipeline = create_pipeline()

    # api_key is resolved automatically by dlt
    source = hubspot_contacts()
    load_info = pipeline.run(source)

    print(load_info)
    return load_info


if __name__ == "__main__":
    run()
```

## Run Locally

Test the pipeline locally:

```sh
python -m pipelines.hubspot
```

Expected output:

```
Pipeline hubspot completed in 12.3 seconds
1 load package(s) were loaded to destination snowflake and target schema HUBSPOT
Load package contains 1 table(s): contacts
```

## Verify in Snowflake

Check the loaded data:

```sql
-- Check the table exists
SHOW TABLES IN DLT.HUBSPOT;

-- View recent contacts
SELECT
    id,
    firstname,
    lastname,
    email,
    createdate,
    lastmodifieddate,
    _dlt_load_id
FROM DLT.HUBSPOT.CONTACTS
ORDER BY lastmodifieddate DESC
LIMIT 10;

-- Count contacts
SELECT COUNT(*)
FROM DLT.HUBSPOT.CONTACTS;

-- Verify analyst access via the existing role chain
-- DLT_DB_READER → ANALYTICS_SOURCES_READER → ANALYTICS_DEVELOPER
USE ROLE ANALYTICS_DEVELOPER;
SELECT * FROM DLT.HUBSPOT.CONTACTS LIMIT 5;
```

## Prefect Flow

Create `flows/hubspot.py`:

```python
"""Prefect flow for HubSpot pipeline."""

from prefect import flow, task, get_run_logger
from prefect.tasks import exponential_backoff


@task(
    name="load-hubspot-contacts",
    retries=3,
    retry_delay_seconds=exponential_backoff(backoff_factor=10),
    retry_jitter_factor=0.5,
)
def load_hubspot_contacts_task():
    """Load contacts from HubSpot CRM to Snowflake."""
    logger = get_run_logger()
    logger.info("Starting HubSpot contacts load")

    from pipelines.hubspot import run

    load_info = run()
    logger.info(f"Load completed: {load_info}")

    return load_info


@flow(name="hubspot-daily", log_prints=True)
def hubspot_daily_flow():
    """Daily flow to load HubSpot contacts (07:00 UTC)."""
    load_info = load_hubspot_contacts_task()
    print(f"HubSpot contacts loaded successfully: {load_info}")
    return {"status": "success", "load_info": str(load_info)}


if __name__ == "__main__":
    hubspot_daily_flow()
```

### Add Deployment

Add to `prefect.yaml`:

```yaml
  # =========================================================================
  # HubSpot (→ Snowflake direct)
  # =========================================================================
  - name: hubspot-daily
    entrypoint: flows/hubspot.py:hubspot_daily_flow
    description: "Daily HubSpot contacts sync via dlt"
    work_pool:
      name: production
    schedules:
      - cron: "0 7 * * *"
        timezone: "UTC"
        active: true
    tags:
      - dlt
      - hubspot
      - daily
```

### Deploy

```sh
prefect deploy --all
```

### Test

```sh
# Trigger manual run
prefect deployment run hubspot-daily/production

# Check status
prefect flow-run ls --limit 5
```

## HubSpot API Rate Limits

HubSpot enforces API rate limits depending on your tier:

| Tier | Rate Limit |
|------|-----------|
| Free / Starter | 100 requests per 10 seconds |
| Professional / Enterprise | 150 requests per 10 seconds |

The verified source respects these limits automatically. For the free CRM tier with a small number of contacts, a daily sync is well within limits.

## Troubleshooting

### Invalid API Key

```
hubspot.exceptions.HubSpotApiError: 401 Unauthorized
```

**Solution**: Check your private app access token is correct and the app has `crm.objects.contacts.read` scope.

### Rate Limited

```
hubspot.exceptions.HubSpotApiError: 429 Too Many Requests
```

**Solution**: The verified source handles retries automatically. If you see persistent 429s, check if other integrations are consuming your rate limit.

### No Contacts Returned

**Solution**: Ensure your HubSpot account has contacts. The free CRM tier allows up to 1,000,000 contacts. Use the Mockaroo import described above to generate test data.

## Summary

You've added the HubSpot pipeline to your data stack:

- [x] Understood when to use dlt vs Airbyte for SaaS sources
- [x] Used dlt's verified HubSpot source for contacts
- [x] Loaded to Snowflake in the existing DLT database (`DLT.HUBSPOT.CONTACTS`)
- [x] Added Prefect orchestration with daily schedule

## What's Next

With HubSpot contacts landing in Snowflake alongside your other data sources, you can:

1. **Transform with dbt**: Create staging and mart models for CRM data
2. **Join with other sources**: Combine contacts with product and revenue data
3. **Add more HubSpot resources**: Expand to companies, deals, or tickets by updating `with_resources()`
4. **Consider Airbyte**: If you need reverse ETL or 5+ SaaS connectors, see the [SaaS Ingestion](../saas-ingestion/index.md) section

For the full Airbyte setup covering complex SaaS connectors and reverse ETL, continue to [SaaS Ingestion](../saas-ingestion/index.md) →
