# Exchange Rates to S3

On this page, you will:

- [x] Build a dlt source for exchange rates (latest and historical)
- [x] Create a pipeline that writes to S3
- [x] Deploy Snowpipe infrastructure using the Terraform module
- [x] Configure S3 event notifications for auto-ingestion
- [x] Verify the full pipeline: API → S3 → Snowpipe → Snowflake

## Overview

The exchange rates pipeline demonstrates a more advanced ingestion pattern: dlt writes JSON files to S3, and Snowpipe automatically ingests them into Snowflake. This decoupled approach is useful when you want to:

- **Stage files** before loading (raw files persist in S3 for auditing or reprocessing)
- **Accumulate historical data** with auto-scaling ingestion
- **Separate concerns** between extraction (dlt) and loading (Snowpipe)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    EXCHANGE RATES PIPELINE                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐        │
│  │ Open Exchange   │ dlt │   S3 Bucket     │ SQS │    Snowpipe     │        │
│  │ Rates API       │────▶│ /exchange-rates/│────▶│  (auto-ingest)  │        │
│  │ /latest.json    │     │                 │     │                 │        │
│  │ /historical.json│     └─────────────────┘     └────────┬────────┘        │
│  └─────────────────┘                                      │                 │
│                                                           ▼                 │
│                                               ┌─────────────────┐           │
│                                               │    Snowflake    │           │
│                                               │ SNOWPIPE.OPEN_  │           │
│                                               │ EXCHANGE_RATES  │           │
│                                               │ .EXCHANGE_RATES │           │
│                                               └─────────────────┘           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## API Endpoints

Open Exchange Rates provides two relevant endpoints:

| Endpoint | Use Case | Rate Limit (Free) |
|----------|----------|-------------------|
| `/latest.json` | Current rates | 1,000 req/month |
| `/historical/{date}.json` | Historical rates | 1,000 req/month total |

Both return the same structure:

```json
{
    "timestamp": 1704067200,
    "base": "USD",
    "rates": {
        "GBP": 0.785,
        "EUR": 0.918,
        "JPY": 141.52
    }
}
```

The nested `rates` object contains ~170 currency codes. With `max_table_nesting = 1` (set in `.dlt/config.toml`), dlt flattens this into individual columns (`rates__gbp`, `rates__eur`, `rates__jpy`, etc.) rather than creating a separate child table.

## The dlt Source

Create `sources/exchange_rates/__init__.py`:

```python
"""Open Exchange Rates source."""
from .source import exchange_rates, exchange_rates_historical

__all__ = ["exchange_rates", "exchange_rates_historical"]
```

Create `sources/exchange_rates/source.py`:

```python
"""dlt source for Open Exchange Rates API."""

from datetime import date, datetime, timedelta
from typing import Iterator

import dlt
import requests


BASE_URL = "https://openexchangerates.org/api"


@dlt.source(section="open_exchange_rates")
def exchange_rates(api_key: str = dlt.secrets.value):
    """
    Source for latest exchange rates.

    Args:
        api_key: Open Exchange Rates App ID (auto-resolved by dlt)
    """
    return latest_rates(api_key)


@dlt.resource(
    name="exchange_rates",
    write_disposition="append",
)
def latest_rates(api_key: str) -> Iterator[dict]:
    """
    Fetch the latest exchange rates.

    Uses append disposition since Snowpipe handles the loading.
    Each run produces a new file in S3 for Snowpipe to ingest.
    """
    response = requests.get(
        f"{BASE_URL}/latest.json",
        params={"app_id": api_key},
        timeout=30,
    )
    response.raise_for_status()

    data = response.json()
    rate_date = datetime.utcfromtimestamp(data["timestamp"]).date().isoformat()

    yield {
        "date": rate_date,
        "timestamp": data["timestamp"],
        "base": data["base"],
        "rates": data["rates"],  # dlt flattens this into columns
        "_loaded_at": datetime.utcnow().isoformat(),
    }


@dlt.source(section="open_exchange_rates")
def exchange_rates_historical(
    api_key: str = dlt.secrets.value,
    start_date: date = date(2026, 1, 1),
    end_date: date | None = None,
):
    """
    Source for historical exchange rates.

    Fetches rates for each day in the date range.

    Args:
        api_key: Open Exchange Rates App ID (auto-resolved by dlt)
        start_date: First date to fetch (inclusive)
        end_date: Last date to fetch (inclusive), defaults to yesterday
    """
    return historical_rates(api_key, start_date, end_date)


@dlt.resource(
    name="exchange_rates",
    write_disposition="append",
)
def historical_rates(
    api_key: str,
    start_date: date,
    end_date: date | None = None,
) -> Iterator[dict]:
    """
    Fetch historical exchange rates for a date range.

    Yields one record per day. Each backfill run produces
    files in S3 for Snowpipe to ingest.
    """
    if end_date is None:
        end_date = date.today() - timedelta(days=1)

    current_date = start_date
    while current_date <= end_date:
        response = requests.get(
            f"{BASE_URL}/historical/{current_date.isoformat()}.json",
            params={"app_id": api_key},
            timeout=30,
        )

        # Skip if date not available
        if response.status_code == 400:
            current_date += timedelta(days=1)
            continue

        response.raise_for_status()
        data = response.json()

        yield {
            "date": current_date.isoformat(),
            "timestamp": data["timestamp"],
            "base": data["base"],
            "rates": data["rates"],
            "_loaded_at": datetime.utcnow().isoformat(),
        }

        current_date += timedelta(days=1)
```

### Why Append Disposition?

Unlike the currencies pipeline (which uses `replace` for full refresh), exchange rates use `append`:

- Each run produces a new JSON file in S3
- Snowpipe ingests each file independently
- Historical data accumulates — you don't want to overwrite previous days
- Deduplication (if needed) can be handled in dbt downstream

## The dlt Pipeline

Create `pipelines/exchange_rates.py`:

```python
"""Exchange rates pipeline: Open Exchange Rates API → S3 → Snowpipe → Snowflake."""

from datetime import date, datetime

import dlt

from sources.exchange_rates import exchange_rates, exchange_rates_historical
from utils.vault_provider import register_aws_secrets


def create_pipeline() -> dlt.Pipeline:
    """Create the exchange rates pipeline with S3 filesystem destination."""

    # Register AWS Secrets Manager for production
    register_aws_secrets()

    pipeline = dlt.pipeline(
        pipeline_name="exchange_rates",
        destination="filesystem",
        dataset_name="exchange_rates",
    )

    return pipeline


def run_latest():
    """Run pipeline for latest exchange rates (daily run)."""
    pipeline = create_pipeline()

    source = exchange_rates()
    load_info = pipeline.run(source)

    print(load_info)
    return load_info


def run_backfill(start_date: date, end_date: date | None = None):
    """
    Run pipeline for historical exchange rates (backfill).

    Args:
        start_date: First date to backfill
        end_date: Last date to backfill (defaults to yesterday)
    """
    pipeline = create_pipeline()

    source = exchange_rates_historical(
        start_date=start_date,
        end_date=end_date,
    )
    load_info = pipeline.run(source)

    print(load_info)
    return load_info


if __name__ == "__main__":
    import argparse

    parser = argparse.ArgumentParser(description="Exchange rates pipeline")
    parser.add_argument(
        "--backfill",
        action="store_true",
        help="Run backfill instead of latest",
    )
    parser.add_argument(
        "--start-date",
        type=lambda s: datetime.strptime(s, "%Y-%m-%d").date(),
        default=date(2026, 1, 1),
        help="Backfill start date (YYYY-MM-DD)",
    )
    parser.add_argument(
        "--end-date",
        type=lambda s: datetime.strptime(s, "%Y-%m-%d").date(),
        default=None,
        help="Backfill end date (YYYY-MM-DD), defaults to yesterday",
    )

    args = parser.parse_args()

    if args.backfill:
        run_backfill(args.start_date, args.end_date)
    else:
        run_latest()
```

The `bucket_url` and S3 credentials are resolved from `.dlt/secrets.toml` (locally) or via the `AWSSecretsManagerProvider` (production). In production on ECS/EC2, IAM roles provide S3 access without explicit credentials.

## Deploy Snowpipe Infrastructure

Before Snowpipe can ingest data, you need to create the stage, file format, and pipe using the module defined in [Snowflake Infrastructure](3-snowflake-infrastructure.md).

### Create the Snowpipe

Navigate to your Snowflake Terraform directory:

```sh
cd ~/projects/data/data-stack-infrastructure/terraform/snowflake
```

Add to `snowpipe.tf`:

```hcl
# =============================================================================
# Snowpipes
# =============================================================================

# -----------------------------------------------------------------------------
# Variables
# -----------------------------------------------------------------------------
variable "data_lake_bucket" {
  description = "S3 data lake bucket name"
  type        = string
}

# -----------------------------------------------------------------------------
# Exchange Rates
# -----------------------------------------------------------------------------
# Auto-ingests exchange rates data written to S3 by the dlt pipeline
module "snowpipe_exchange_rates" {
  source = "./modules/snowflake_snowpipe"

  providers = {
    snowflake.sys_admin = snowflake.sys_admin
  }

  database = module.database_snowpipe.database_name
  schema   = module.schema_snowpipe_open_exchange_rates.schema_name

  name                = "EXCHANGE_RATES"
  stage_url           = "s3://${var.data_lake_bucket}/exchange-rates/"
  storage_integration = "DATA_LAKE_PROD"
  target_table        = "EXCHANGE_RATES"
  format_type         = "JSON"
}
```

Add to `outputs.tf`:

```hcl
output "snowpipe_exchange_rates_notification_channel" {
  description = "SQS queue ARN for exchange rates Snowpipe notifications"
  value       = module.snowpipe_exchange_rates.notification_channel
}
```

!!! note "Target Table"
    Snowpipe needs the target table to exist before it can ingest data. The table is created when you first run the exchange rates pipeline (dlt writes the schema to S3 which Snowpipe uses). Alternatively, create it manually:

    ```sql
    CREATE TABLE IF NOT EXISTS SNOWPIPE.OPEN_EXCHANGE_RATES.EXCHANGE_RATES (
        date VARCHAR,
        timestamp NUMBER,
        base VARCHAR,
        rates VARIANT,
        _loaded_at VARCHAR,
        _dlt_load_id VARCHAR,
        _dlt_id VARCHAR
    );
    ```

### Deploy the Snowpipe

=== "CI/CD"

    ```sh
    cd ~/projects/data/data-stack-infrastructure
    git checkout -b feat/exchange-rates-snowpipe
    git add terraform/snowflake/snowpipe.tf terraform/snowflake/outputs.tf
    git commit -m "Add Snowpipe for exchange rates auto-ingestion"
    git push -u origin feat/exchange-rates-snowpipe
    ```

    Open a PR, review the plan, then merge to apply.

=== "Local"

    ```sh
    cd ~/projects/data/data-stack-infrastructure/terraform/snowflake
    terraform plan
    terraform apply
    ```

After deployment, note the `snowpipe_exchange_rates_notification_channel` output value. You'll need it for S3 event notifications.

### Configure S3 Event Notifications

Snowpipe auto-ingest works by listening for S3 event notifications. When dlt writes a new file to S3, an SQS message triggers Snowpipe to load it.

Navigate to your AWS Terraform directory:

```sh
cd ~/projects/data/data-stack-infrastructure/terraform/aws
```

Add to `snowpipe_notifications.tf`:

```hcl
# =============================================================================
# Snowpipe S3 Event Notifications
# =============================================================================

# -----------------------------------------------------------------------------
# Variables
# -----------------------------------------------------------------------------
variable "snowpipe_exchange_rates_sqs_arn" {
  description = "SQS ARN from Snowflake Snowpipe for exchange rates"
  type        = string
  default     = ""  # Set after Snowpipe is created
}

# -----------------------------------------------------------------------------
# Exchange Rates S3 → Snowpipe
# -----------------------------------------------------------------------------
resource "aws_s3_bucket_notification" "snowpipe_exchange_rates" {
  count  = var.snowpipe_exchange_rates_sqs_arn != "" ? 1 : 0
  bucket = module.data_lake["prod"].bucket_id

  queue {
    queue_arn     = var.snowpipe_exchange_rates_sqs_arn
    events        = ["s3:ObjectCreated:*"]
    filter_prefix = "exchange-rates/"
  }
}
```

Set the SQS ARN from the Snowpipe output and deploy:

=== "CI/CD"

    Update `terraform.tfvars` with the SQS ARN:

    ```hcl
    snowpipe_exchange_rates_sqs_arn = "arn:aws:sqs:eu-west-2:..."
    ```

    ```sh
    cd ~/projects/data/data-stack-infrastructure
    git checkout -b feat/snowpipe-s3-notifications
    git add terraform/aws/snowpipe_notifications.tf terraform/aws/terraform.tfvars
    git commit -m "Add S3 event notifications for exchange rates Snowpipe"
    git push -u origin feat/snowpipe-s3-notifications
    ```

=== "Local"

    ```sh
    cd ~/projects/data/data-stack-infrastructure/terraform/aws
    terraform plan -var='snowpipe_exchange_rates_sqs_arn=arn:aws:sqs:eu-west-2:...'
    terraform apply
    ```

### Deployment Order

Due to dependencies between Snowflake and AWS, deploy in this order:

1. **Snowflake**: Snowpipe resources (stage, format, pipe)
2. **Copy output**: Note the `notification_channel` SQS ARN
3. **AWS**: S3 event notifications pointing to the SQS ARN

## Run the Pipeline

Test the pipeline locally:

```sh
cd ~/projects/data/data-pipelines

# Run latest rates (writes one file to S3)
uv run python -m pipelines.exchange_rates

# Run backfill for a week (writes one file per day)
uv run python -m pipelines.exchange_rates --backfill --start-date 2026-01-01 --end-date 2026-01-07
```

## Verify S3 Files

Check that files were written to S3:

```sh
aws s3 ls --profile data-engineer s3://your-data-lake-bucket/exchange-rates/ --recursive
```

View the contents of a file:

```sh
aws s3 cp --profile data-engineer s3://your-data-lake-bucket/exchange-rates/exchange_rates/exchange_rates/1704067200.json - | head -20
```

## Verify Snowpipe Ingestion

Snowpipe should automatically ingest files within a few minutes. Check the status:

```sql
-- Check pipe status
SELECT SYSTEM$PIPE_STATUS('SNOWPIPE.OPEN_EXCHANGE_RATES.EXCHANGE_RATES_PIPE');

-- Check recent copy history
SELECT *
FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME => 'SNOWPIPE.OPEN_EXCHANGE_RATES.EXCHANGE_RATES',
    START_TIME => DATEADD(hours, -1, CURRENT_TIMESTAMP())
));
```

Query the loaded data:

```sql
-- View recent exchange rates
SELECT
    date,
    base,
    rates__gbp,
    rates__eur,
    rates__jpy,
    _loaded_at
FROM SNOWPIPE.OPEN_EXCHANGE_RATES.EXCHANGE_RATES
ORDER BY date DESC
LIMIT 10;

-- Check date range
SELECT MIN(date), MAX(date), COUNT(*)
FROM SNOWPIPE.OPEN_EXCHANGE_RATES.EXCHANGE_RATES;

-- Check for duplicates (if any, handle in dbt)
SELECT date, COUNT(*) as cnt
FROM SNOWPIPE.OPEN_EXCHANGE_RATES.EXCHANGE_RATES
GROUP BY date
HAVING COUNT(*) > 1;
```

## Schema Evolution

dlt automatically handles schema changes. When Open Exchange Rates adds new currencies:

1. dlt detects the new keys in the `rates` dictionary
2. New files include the additional fields
3. Snowpipe loads them into `VARIANT` columns or new columns (depending on table definition)
4. No pipeline code changes needed

## Rate Limit Considerations

The free tier allows 1,000 requests/month. With this pipeline:

- **Daily runs**: ~31 requests/month (one per day)
- **Backfill**: One request per historical day

For a full year backfill (365 days), you'd need multiple months of quota. Consider:

1. **Upgrade to paid tier** for unlimited requests
2. **Backfill incrementally** over several months
3. **Use a different source** for bulk historical data

## Troubleshooting Snowpipe

### Data Not Appearing

If data doesn't appear in Snowflake after a few minutes:

1. **Check S3 event notifications**:

    ```sh
    aws s3api get-bucket-notification-configuration \
        --bucket your-data-lake-bucket
    ```

    Should show a queue configuration for the `exchange-rates/` prefix.

2. **Check if the pipe is paused**:

    ```sql
    SHOW PIPES LIKE 'EXCHANGE_RATES_PIPE';
    ```

    Resume if paused:

    ```sql
    ALTER PIPE SNOWPIPE.OPEN_EXCHANGE_RATES.EXCHANGE_RATES_PIPE
        SET PIPE_EXECUTION_PAUSED = FALSE;
    ```

3. **Check file format errors**:

    ```sql
    COPY INTO SNOWPIPE.OPEN_EXCHANGE_RATES.EXCHANGE_RATES
    FROM @SNOWPIPE.OPEN_EXCHANGE_RATES.EXCHANGE_RATES_STAGE
    FILE_FORMAT = (FORMAT_NAME = 'SNOWPIPE.OPEN_EXCHANGE_RATES.EXCHANGE_RATES_FORMAT')
    VALIDATION_MODE = 'RETURN_ERRORS';
    ```

4. **Check the target table exists** — Snowpipe fails silently if the table is missing.

## When to Use This Pattern

| Scenario | S3 + Snowpipe | Direct Load |
|----------|---------------|-------------|
| High-volume ingestion | Yes | No |
| Keep raw files for audit | Yes | No |
| Event-driven architecture | Yes | No |
| Simple API extraction | No | Yes |
| Real-time requirements | No (1-5 min latency) | No |
| Development/testing | No | Yes (faster debugging) |

For daily exchange rates, either pattern works. This pipeline demonstrates the S3 + Snowpipe pattern so you have it available for higher-volume use cases.

## Summary

You've built the exchange rates pipeline with S3 staging:

- [x] dlt source for `/latest.json` and `/historical/{date}.json`
- [x] Pipeline writing to S3 filesystem destination
- [x] Snowpipe Terraform module deployed (stage + format + pipe)
- [x] S3 event notifications configured for auto-ingestion
- [x] End-to-end verification: API → S3 → Snowpipe → Snowflake

## What's Next

The final pipeline demonstrates database extraction: pulling data from PostgreSQL with Type 2 SCD (Slowly Changing Dimensions) handling.

Continue to [Products Pipeline](7-products-pipeline.md) →
