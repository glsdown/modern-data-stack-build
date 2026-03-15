# Currencies Pipeline

On this page, you will:

- [x] Build a dlt source for the currencies endpoint
- [x] Create a pipeline that loads directly to Snowflake
- [x] Understand how dlt resolves credentials automatically
- [x] Verify data in Snowflake

## Overview

The currencies pipeline is your first dlt pipeline. It extracts currency reference data from the [Open Exchange Rates API](https://openexchangerates.org/api) and loads it directly into Snowflake. This is the simplest ingestion pattern: API → Snowflake.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       CURRENCIES PIPELINE                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐        ┌─────────────────┐        ┌─────────────────┐  │
│  │ Open Exchange   │  dlt   │    Snowflake    │        │                 │  │
│  │ Rates API       │ ────▶  │ DLT.OPEN_       │        │  Weekly run     │  │
│  │ /currencies.json│        │ EXCHANGE_RATES  │        │  Full refresh   │  │
│  │                 │        │ .CURRENCIES     │        │                 │  │
│  └─────────────────┘        └─────────────────┘        └─────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## The Currencies Endpoint

The `/currencies.json` endpoint returns a mapping of currency codes to names:

```json
{
    "AED": "United Arab Emirates Dirham",
    "AFN": "Afghan Afghani",
    "ALL": "Albanian Lek",
    "GBP": "British Pound Sterling",
    "USD": "United States Dollar"
}
```

This is reference data that changes rarely, so a weekly full refresh is sufficient.

## The dlt Source

Create `sources/currencies/__init__.py`:

```python
"""Currencies source."""
from .source import currencies

__all__ = ["currencies"]
```

Create `sources/currencies/source.py`:

```python
"""dlt source for Open Exchange Rates currencies endpoint."""

from datetime import datetime
from typing import Iterator

import dlt
import requests


BASE_URL = "https://openexchangerates.org/api"


@dlt.source(section="open_exchange_rates")
def currencies(api_key: str = dlt.secrets.value):
    """
    Source for Open Exchange Rates currencies endpoint.

    Returns the list of all available currencies with their names.
    The api_key is resolved automatically by dlt from:
    - Environment: SOURCES__OPEN_EXCHANGE_RATES__API_KEY
    - secrets.toml: [sources.open_exchange_rates] api_key
    - VaultDocProvider: dlt/open-exchange-rates → {"api_key": "..."}

    Args:
        api_key: Open Exchange Rates App ID (auto-resolved by dlt)
    """
    return currency_list(api_key)


@dlt.resource(
    name="currencies",
    write_disposition="replace",  # Full refresh each time
)
def currency_list(api_key: str) -> Iterator[dict]:
    """
    Fetch the list of available currencies.

    Uses replace disposition since this is reference data
    that should be fully refreshed each run.
    """
    response = requests.get(
        f"{BASE_URL}/currencies.json",
        params={"app_id": api_key},
        timeout=30,
    )
    response.raise_for_status()

    data = response.json()
    loaded_at = datetime.utcnow().isoformat()

    # Transform key-value pairs into rows
    for code, name in data.items():
        yield {
            "code": code,
            "name": name,
            "_loaded_at": loaded_at,
        }
```

### Key Design Decisions

1. **`section="open_exchange_rates"`**: This tells dlt to look for configuration under `sources.open_exchange_rates`, so both the currencies and exchange rates sources share the same API key config path.

2. **`api_key: str = dlt.secrets.value`**: The `dlt.secrets.value` sentinel tells dlt to resolve this parameter from the configuration chain automatically. You don't need to pass it explicitly.

3. **`write_disposition="replace"`**: Unlike exchange rates (which accumulate over time), currencies is reference data — the full list should be refreshed each run. Old codes that are removed from the API disappear from the table.

## The dlt Pipeline

Create `pipelines/currencies.py`:

```python
"""Currencies pipeline: Open Exchange Rates API → Snowflake."""

import dlt

from sources.currencies import currencies
from utils.vault_provider import register_aws_secrets


def create_pipeline() -> dlt.Pipeline:
    """Create the currencies pipeline with Snowflake destination."""

    # Register AWS Secrets Manager for production credential resolution.
    # Locally, secrets.toml values take priority (provider is never reached).
    register_aws_secrets()

    pipeline = dlt.pipeline(
        pipeline_name="currencies",
        destination="snowflake",
        dataset_name="OPEN_EXCHANGE_RATES",  # Schema in DLT database
    )

    return pipeline


def run():
    """Run the currencies pipeline."""
    pipeline = create_pipeline()

    # api_key is resolved automatically by dlt from the config chain
    source = currencies()
    load_info = pipeline.run(source)

    print(load_info)
    return load_info


if __name__ == "__main__":
    run()
```

!!! info "No Explicit Credentials"
    Notice that the pipeline code doesn't fetch credentials explicitly. dlt resolves `destination.snowflake.credentials` and `sources.open_exchange_rates.api_key` from the configuration chain:

    - **Locally**: from `.dlt/secrets.toml` (your own Snowflake user)
    - **Production**: from the `AWSSecretsManagerProvider` (SVC_DLT service account)

    The `register_aws_secrets()` call adds the provider to the chain. If `secrets.toml` exists (local dev), it takes priority and the AWS provider is never queried.

## Run Locally

Test the pipeline locally:

```sh
cd ~/projects/data/data-pipelines

# Run the pipeline
python -m pipelines.currencies
```

Expected output:

```
Pipeline currencies completed in X.XX seconds
1 load package(s) were loaded to destination snowflake and target schema OPEN_EXCHANGE_RATES
The snowflake destination used snowflake://YOUR_USER@orgname-accountname/DLT location to store data
```

## Verify in Snowflake

Check the loaded data:

```sql
-- View currencies
SELECT code, name, _loaded_at
FROM DLT.OPEN_EXCHANGE_RATES.CURRENCIES
ORDER BY code
LIMIT 20;

-- Count total currencies (should be ~170)
SELECT COUNT(*)
FROM DLT.OPEN_EXCHANGE_RATES.CURRENCIES;

-- Check dlt metadata tables
SHOW TABLES IN DLT.OPEN_EXCHANGE_RATES;
-- Should show: CURRENCIES, _DLT_LOADS, _DLT_VERSION, _DLT_PIPELINE_STATE
```

The `_dlt_*` tables are managed by dlt:

| Table | Purpose |
|-------|---------|
| `_DLT_LOADS` | Load history (timestamps, row counts, status) |
| `_DLT_VERSION` | Schema version tracking |
| `_DLT_PIPELINE_STATE` | Pipeline state for incremental loading and recovery |

## Test with DuckDB

During development, you can use DuckDB to test without connecting to Snowflake:

```sh
# Override destination via environment variable
DESTINATION__NAME=duckdb python -m pipelines.currencies
```

This creates a local `currencies.duckdb` file. You can query it:

```python
# test_duckdb.py
import duckdb

conn = duckdb.connect("currencies.duckdb")
print(conn.sql("SELECT * FROM OPEN_EXCHANGE_RATES.currencies LIMIT 5").df())
```

```sh
python test_duckdb.py
rm test_duckdb.py currencies.duckdb
```

## Testing

Create a test to verify the source works:

```python
# tests/test_currencies.py
from unittest.mock import MagicMock, patch

from sources.currencies import currencies


def test_currency_list_returns_rows():
    """Test that currencies returns one row per currency."""
    mock_response = MagicMock()
    mock_response.json.return_value = {
        "GBP": "British Pound Sterling",
        "USD": "United States Dollar",
        "EUR": "Euro",
    }
    mock_response.raise_for_status = MagicMock()

    with patch("sources.currencies.source.requests.get", return_value=mock_response):
        source = currencies(api_key="test")
        records = list(source)

        assert len(records) == 3
        codes = {r["code"] for r in records}
        assert codes == {"GBP", "USD", "EUR"}
        assert all("name" in r for r in records)
        assert all("_loaded_at" in r for r in records)
```

```sh
pytest tests/test_currencies.py -v
```

## Summary

You've built the currencies pipeline:

- [x] dlt source for the `/currencies.json` endpoint
- [x] Pipeline loading directly to Snowflake (`DLT.OPEN_EXCHANGE_RATES.CURRENCIES`)
- [x] Replace write disposition for full refresh of reference data
- [x] Automatic credential resolution via dlt's config chain
- [x] DuckDB testing for local development

## What's Next

The next pipeline demonstrates a different pattern: loading to S3 for Snowpipe auto-ingestion. This is useful for high-volume or historical data where you want files staged in S3 before loading.

Continue to [Exchange Rates to S3](6-exchange-rates-to-s3.md) →
