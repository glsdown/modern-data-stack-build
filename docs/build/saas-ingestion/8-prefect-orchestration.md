# Prefect Orchestration

On this page, you will:

- [x] Install the prefect-airbyte integration package
- [x] Create Prefect flows to trigger Airbyte syncs
- [x] Configure schedules and retries
- [x] Set up failure alerting

## Overview

Rather than using Airbyte's built-in scheduler, you trigger syncs from Prefect. This provides a single orchestration layer for both dlt and Airbyte pipelines, with consistent alerting, retries, and monitoring.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PREFECT + AIRBYTE ORCHESTRATION                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Prefect Flows                                                              │
│  ─────────────                                                              │
│                                                                             │
│  ┌───────────────────────────┐     ┌───────────────────────────┐            │
│  │  hubspot-airbyte-daily    │     │  retl-contacts-daily      │            │
│  │                           │     │                           │            │
│  │  Schedule: 07:00 UTC      │     │  Schedule: 08:00 UTC      │            │
│  │  Retries: 2               │     │  (after dbt transforms)   │            │
│  │  Timeout: 30 min          │     │  Retries: 2               │            │
│  └─────────────┬─────────────┘     └─────────────┬─────────────┘            │
│                │                                 │                          │
│                ▼                                 ▼                          │
│  ┌─────────────────────────────────────────────────────────────┐            │
│  │                    Airbyte API                              │            │
│  │  POST /v1/jobs  (trigger sync)                              │            │
│  │  GET  /v1/jobs  (check status)                              │            │
│  └─────────────────────────────────────────────────────────────┘            │
│                │                                                            │
│                ▼                                                            │
│  ┌───────────────────────────────────────┐                                  │
│  │  On Failure: Slack notification       │                                  │
│  │  Same automations as dlt pipelines    │                                  │
│  └───────────────────────────────────────┘                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Install Dependencies

Add the `prefect-airbyte` integration package to your `data-pipelines` project:

```sh
uv add prefect-airbyte
```

## Project Structure

Add Airbyte flows to the existing data-pipelines repository:

```
data-pipelines/
├── flows/
│   ├── exchange_rates.py       # dlt flow
│   ├── currencies.py           # dlt flow
│   ├── products.py             # dlt flow
│   ├── hubspot_airbyte.py      # Airbyte flow
│   └── retl_contacts.py        # Reverse ETL flow
├── pyproject.toml
├── uv.lock
└── prefect.yaml
```

## HubSpot Airbyte Flow

The `prefect-airbyte` package provides built-in tasks for triggering Airbyte syncs and waiting for completion.

Create `flows/hubspot_airbyte.py`:

```python
"""Prefect flow for HubSpot sync via Airbyte."""

from prefect import flow, task, get_run_logger
from prefect.tasks import exponential_backoff
from prefect_airbyte import AirbyteConnection, AirbyteServer


AIRBYTE_SERVER = AirbyteServer(
    server_host="api.airbyte.com",
    server_port=443,
    api_version="v1",
)


@task(
    name="trigger-hubspot-sync",
    retries=2,
    retry_delay_seconds=exponential_backoff(backoff_factor=10),
    retry_jitter_factor=0.5,
)
def trigger_airbyte_sync(connection_id: str) -> str:
    """Trigger an Airbyte sync and wait for completion."""
    logger = get_run_logger()
    logger.info(f"Triggering Airbyte sync for connection {connection_id}")

    connection = AirbyteConnection(
        airbyte_server=AIRBYTE_SERVER,
        connection_id=connection_id,
        poll_interval_s=30,
        status_updates=True,
        timeout=1800,  # 30 minutes
    )
    result = connection.trigger()
    logger.info(f"Sync completed: {result}")
    return result.job_id


@flow(name="hubspot-airbyte-daily", log_prints=True)
def hubspot_airbyte_daily_flow(connection_id: str):
    """
    Daily flow to sync HubSpot contacts via Airbyte.

    Runs at 07:00 UTC each day. Triggers the Airbyte connection
    and waits for completion.
    """
    job_id = trigger_airbyte_sync(connection_id)
    print(f"HubSpot Airbyte sync completed. Job ID: {job_id}")
    return {"status": "success", "job_id": job_id}


if __name__ == "__main__":
    import os
    hubspot_airbyte_daily_flow(
        connection_id=os.environ["AIRBYTE_HUBSPOT_CONNECTION_ID"]
    )
```

### Authenticating with Airbyte Cloud

The `prefect-airbyte` package reads the Airbyte API key from the `AIRBYTE_API_KEY` environment variable. In CI/CD, fetch it from Secrets Manager before running the flow:

```sh
export AIRBYTE_API_KEY=$(aws secretsmanager get-secret-value \
    --secret-id "airbyte/api-credentials" \
    --query 'SecretString' \
    --output text | jq -r '.api_key')
```

For local development, set it in your shell environment or `.env` file (not committed to git).

### Key Design Decisions

1. **`prefect-airbyte` for sync management**: The package handles polling and status updates, so your flow code stays clean.
2. **30-minute timeout**: HubSpot contacts syncs typically complete in minutes. The timeout catches stuck syncs.
3. **Retries with exponential backoff**: Retries ~10s, ~100s, ±50% jitter — handles transient Airbyte API errors without hammering the service.
4. **Connection ID as parameter**: Pass the connection ID as a flow parameter so the same flow code works for any Airbyte connection.

## Reverse ETL Flow

Create `flows/retl_contacts.py`:

```python
"""Prefect flow for reverse ETL: Snowflake → HubSpot via Airbyte."""

from prefect import flow, get_run_logger

from flows.hubspot_airbyte import trigger_airbyte_sync, AIRBYTE_SERVER


@flow(name="retl-contacts-daily", log_prints=True)
def retl_contacts_daily_flow(connection_id: str):
    """
    Daily flow to sync enriched contacts from Snowflake to HubSpot.

    Runs at 08:00 UTC, after dbt transformations complete.
    Triggers the reverse ETL Airbyte connection.
    """
    job_id = trigger_airbyte_sync(connection_id)
    print(f"Reverse ETL sync completed. Job ID: {job_id}")
    return {"status": "success", "job_id": job_id}


if __name__ == "__main__":
    import os
    retl_contacts_daily_flow(
        connection_id=os.environ["AIRBYTE_RETL_CONNECTION_ID"]
    )
```

## Manual API Approach (Alternative)

If you need more control than `prefect-airbyte` provides (custom error handling, detailed logging), you can call the Airbyte API directly:

```python
"""Alternative: call the Airbyte API directly."""

import time
import json
import boto3
import requests
from prefect import task, get_run_logger


def _get_airbyte_credentials() -> dict:
    """Fetch Airbyte API credentials from AWS Secrets Manager."""
    client = boto3.client("secretsmanager", region_name="eu-west-2")
    response = client.get_secret_value(SecretId="airbyte/api-credentials")
    return json.loads(response["SecretString"])


@task(retries=2)
def trigger_airbyte_sync_manual(connection_id: str) -> str:
    creds = _get_airbyte_credentials()
    headers = {"Authorization": f"Bearer {creds['api_key']}"}
    base_url = creds["api_url"]
    # ... trigger and poll
```

!!! note "Choose One Approach"
    Use either the `prefect-airbyte` package or the manual API approach — not both. The package approach is simpler and recommended.

## Deployment Configuration

Add to `prefect.yaml`:

```yaml
  # =========================================================================
  # Airbyte Syncs
  # =========================================================================
  - name: hubspot-airbyte-daily
    entrypoint: flows/hubspot_airbyte.py:hubspot_airbyte_daily_flow
    description: "Daily HubSpot contacts sync via Airbyte"
    work_pool:
      name: production
    schedules:
      - cron: "0 7 * * *"
        timezone: "UTC"
        active: true
    tags:
      - airbyte
      - hubspot
      - daily

  - name: retl-contacts-daily
    entrypoint: flows/retl_contacts.py:retl_contacts_daily_flow
    description: "Daily reverse ETL: enriched contacts to HubSpot"
    work_pool:
      name: production
    schedules:
      - cron: "0 8 * * *"  # After dbt transformations
        timezone: "UTC"
        active: true
    tags:
      - airbyte
      - reverse-etl
      - daily
```

### Schedule Ordering

The flows should run in sequence:

| Time (UTC) | Flow | Purpose |
|------------|------|---------|
| 06:00 | `products-daily` | Load products from PostgreSQL |
| 07:00 | `hubspot-airbyte-daily` | Load contacts from HubSpot |
| 07:30 | (dbt transformations) | Transform and enrich data |
| 08:00 | `retl-contacts-daily` | Push enriched data back to HubSpot |
| 09:00 | `exchange-rates-daily` | Load exchange rates |

!!! tip "Dependency-Based Scheduling"
    The schedule above uses time-based ordering. For strict dependency ordering (e.g., reverse ETL only runs after dbt completes), consider using Prefect's event-based triggers or flow dependencies.

## Deploy

```sh
uv run prefect deploy --all
```

## Test

### Trigger Manual Run

```sh
# Test HubSpot forward sync
uv run prefect deployment run hubspot-airbyte-daily/production

# Test reverse ETL
uv run prefect deployment run retl-contacts-daily/production
```

### Check Status

```sh
uv run prefect flow-run ls --limit 5
uv run prefect flow-run logs <run-id>
```

## Failure Alerting

Airbyte flows use the same alerting patterns as dlt flows. The `airbyte` and `reverse-etl` tags allow filtering in Prefect automations.

### Tag-Based Automations

If you have an automation for the `dlt` tag (from [Alerting](../orchestration/10-alerting.md)), create a similar one for `airbyte`:

1. Navigate to **Automations** in Prefect Cloud
2. Create a new automation:
   - **Trigger**: Flow run enters `Failed` or `Crashed`
   - **Filter**: Tag equals `airbyte`
   - **Action**: Send Slack notification

Or via Terraform:

```hcl
# terraform/prefect/automations.tf

resource "prefect_automation" "airbyte_failure_alert" {
  name        = "Airbyte Sync Failure Alert"
  description = "Alert on any Airbyte sync failure"

  trigger = {
    type   = "event"
    events = ["prefect.flow-run.Failed", "prefect.flow-run.Crashed"]
    match_related = {
      "prefect.tag" = "airbyte"
    }
  }

  actions = [
    {
      type     = "send-notification"
      block_id = prefect_block.slack_webhook_alerts.id
      body     = "Airbyte sync failed: {{ flow.name }} - {{ flow_run.state.message }}"
    }
  ]
}
```

## Monitoring

### Combined Dashboard

In Prefect Cloud, you can view all pipeline flows together:

| Tag | Flows |
|-----|-------|
| `dlt` | exchange-rates-daily, currencies-weekly, products-daily, hubspot-daily |
| `airbyte` | hubspot-airbyte-daily |
| `reverse-etl` | retl-contacts-daily |

### Airbyte Sync Monitoring

In addition to Prefect, monitor syncs in the Airbyte UI:

- **Sync history**: Duration, records synced, bytes transferred
- **Error logs**: Detailed connector-level error messages
- **Schema changes**: Notifications when upstream schemas change

## Summary

You've configured Prefect orchestration for Airbyte:

- [x] Installed `prefect-airbyte` integration package
- [x] Created flows to trigger Airbyte syncs via API
- [x] Configured schedules (forward ETL at 07:00, reverse ETL at 08:00)
- [x] Set up failure alerting with tag-based automations
- [x] Deployed all flows to Prefect Cloud

## What's Next

Verify everything works end-to-end and review the cost summary.

Continue to [Finishing Up](9-finishing-up.md) →
