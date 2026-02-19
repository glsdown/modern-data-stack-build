# Products Pipeline

On this page, you will:

- [x] Understand Type 2 SCD (Slowly Changing Dimensions)
- [x] Build a dlt source using sql_database
- [x] Create a pipeline that extracts from PostgreSQL to Snowflake
- [x] Handle incremental loading with merge keys

## Overview

The products pipeline extracts data from a PostgreSQL database hosted on Clever Cloud. This demonstrates database-to-database replication, a common pattern for loading application data into your warehouse.

The source table uses Type 2 SCD format, where each row represents a version of a product with operation type (INSERT, UPDATE, DELETE) and timestamp.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       PRODUCTS PIPELINE                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐        ┌─────────────────┐        ┌─────────────────┐  │
│  │ Clever Cloud    │  dlt   │    Snowflake    │        │                 │  │
│  │ PostgreSQL      │ ────▶  │ DLT.APPLICATION │        │  Daily 06:00    │  │
│  │ raw_data.       │        │ _DATA.PRODUCTS  │        │  Incremental    │  │
│  │ products        │        │                 │        │                 │  │
│  └─────────────────┘        └─────────────────┘        └─────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Type 2 SCD Source Format

The source products table tracks changes with this structure:

| Column | Type | Description |
|--------|------|-------------|
| `audit_id` | BIGINT | Unique row identifier |
| `product_id` | BIGINT | Product business key |
| `product_name` | TEXT | Product name |
| `price_usd` | FLOAT | Price in USD |
| `operation` | TEXT | INSERT, UPDATE, or DELETE |
| `valid_ts` | TIMESTAMP | When the change occurred |

Example data:

```sql
-- Initial insert
(1, 6, 'Dog Lead', 12.99, 'INSERT', '2026-01-01 10:00:00')

-- Price update
(105, 6, 'Dog Lead', 15.99, 'UPDATE', '2026-02-01 14:30:00')

-- Product deletion
(101, 2, null, null, 'DELETE', '2026-02-01 09:00:00')
```

This format is common in application databases that need to preserve change history.

## Prerequisites

Ensure you have:

- [x] [Credentials Setup](4-credentials-setup.md) - PostgreSQL credentials in Secrets Manager
- [x] Clever Cloud PostgreSQL with products table (see [External Database Setup](https://github.com/YOUR-ORG/modern-data-stack-build/tree/main/setup_scripts/external_database))

## Project Structure

Add the products source and pipeline:

```
data-pipelines/
├── sources/
│   ├── exchange_rates/
│   ├── currencies/
│   └── products/
│       ├── __init__.py
│       └── source.py      # dlt source using sql_database
├── pipelines/
│   ├── exchange_rates.py
│   ├── currencies.py
│   └── products.py        # dlt pipeline to Snowflake
└── utils/
    └── vault_provider.py
```

## The dlt Source

dlt provides a built-in `sql_database` source for extracting from databases. This is simpler than writing custom SQL extraction.

Create `sources/products/__init__.py`:

```python
"""Products source."""
from .source import products

__all__ = ["products"]
```

Create `sources/products/source.py`:

```python
"""dlt source for products data from PostgreSQL."""

from datetime import datetime

import dlt
from dlt.sources.sql_database import sql_database


def products(incremental_start: datetime | None = None):
    """
    Source for products table from PostgreSQL.

    Uses dlt's sql_database source with incremental loading based on valid_ts.
    PostgreSQL credentials are resolved from the dlt config chain:
    - secrets.toml: [sources.clever_cloud]
    - VaultDocProvider: dlt/clever-cloud-postgres

    Args:
        incremental_start: Optional start timestamp for incremental load

    Returns:
        dlt source for products data
    """
    # Build connection string from dlt config
    # dlt resolves sources.clever_cloud.* from secrets.toml or VaultDocProvider
    creds = dlt.secrets["sources.clever_cloud"]
    conn_str = (
        f"postgresql://{creds['username']}:{creds['password']}"
        f"@{creds['host']}:{creds['port']}/{creds['database']}"
    )

    # Create the sql_database source
    source = sql_database(
        credentials=conn_str,
        schema="raw_data",
        table_names=["products"],
        reflection_level="full",  # Include all column metadata
    )

    # Configure incremental loading on valid_ts
    source.products.apply_hints(
        write_disposition="merge",
        primary_key="audit_id",  # Unique row identifier
        merge_key="audit_id",    # Merge on the unique row
        incremental=dlt.sources.incremental(
            cursor_path="valid_ts",
            initial_value=incremental_start or datetime(2026, 1, 1),
        ),
    )

    return source
```

### Key Configuration

1. **`write_disposition="merge"`**: Updates existing rows and inserts new ones
2. **`primary_key="audit_id"`**: The unique identifier for each row
3. **`merge_key="audit_id"`**: Key used for merge operations
4. **`incremental` on `valid_ts`**: Only fetch rows with `valid_ts` greater than last run

## The dlt Pipeline

Create `pipelines/products.py`:

```python
"""Products pipeline: PostgreSQL → Snowflake."""

from datetime import datetime

import dlt

from sources.products import products
from utils.vault_provider import register_aws_secrets


def create_pipeline() -> dlt.Pipeline:
    """Create the products pipeline with Snowflake destination."""

    # Register AWS Secrets Manager for production credential resolution
    register_aws_secrets()

    pipeline = dlt.pipeline(
        pipeline_name="products",
        destination="snowflake",
        dataset_name="APPLICATION_DATA",  # Schema name in DLT database
    )

    return pipeline


def run(full_refresh: bool = False, start_date: datetime | None = None):
    """
    Run the products pipeline.

    Args:
        full_refresh: If True, reload all data from start_date
        start_date: Override start date for incremental loading
    """
    pipeline = create_pipeline()

    # PostgreSQL credentials are resolved by the source from the config chain
    # (secrets.toml locally, VaultDocProvider in production)

    # Get source with incremental configuration
    if full_refresh:
        # Reset pipeline state for full refresh
        pipeline.sync_destination()

    source = products(
        incremental_start=start_date,
    )

    # Run the pipeline
    load_info = pipeline.run(source)

    print(load_info)
    return load_info


if __name__ == "__main__":
    import argparse

    parser = argparse.ArgumentParser(description="Products pipeline")
    parser.add_argument(
        "--full-refresh",
        action="store_true",
        help="Reload all data (ignore incremental state)",
    )
    parser.add_argument(
        "--start-date",
        type=lambda s: datetime.strptime(s, "%Y-%m-%d"),
        default=None,
        help="Override start date (YYYY-MM-DD)",
    )

    args = parser.parse_args()
    run(full_refresh=args.full_refresh, start_date=args.start_date)
```

## Run Locally

Test the pipeline locally (dependencies were installed in [Project Setup](2-project-setup.md)):

```sh
cd ~/projects/data/data-pipelines

# Run incremental load
uv run python -m pipelines.products

# Run full refresh (reload all data)
uv run python -m pipelines.products --full-refresh

# Run from specific date
uv run python -m pipelines.products --start-date 2026-02-01
```

## Verify in Snowflake

Check the loaded data:

```sql
-- View products table
SELECT
    audit_id,
    product_id,
    product_name,
    price_usd,
    operation,
    valid_ts,
    _dlt_load_id
FROM DLT.APPLICATION_DATA.PRODUCTS
ORDER BY audit_id DESC
LIMIT 20;

-- Check for specific product history (e.g., product_id = 6)
SELECT *
FROM DLT.APPLICATION_DATA.PRODUCTS
WHERE product_id = 6
ORDER BY valid_ts;

-- Count by operation type
SELECT operation, COUNT(*)
FROM DLT.APPLICATION_DATA.PRODUCTS
GROUP BY operation;

-- Check incremental loading (should see increasing _dlt_load_id)
SELECT
    _dlt_load_id,
    MIN(valid_ts) as min_ts,
    MAX(valid_ts) as max_ts,
    COUNT(*) as row_count
FROM DLT.APPLICATION_DATA.PRODUCTS
GROUP BY _dlt_load_id
ORDER BY _dlt_load_id;
```

## Understanding SCD Handling

With this pipeline, the raw Type 2 SCD data lands in Snowflake unchanged. Your dbt models can then transform it:

### Option 1: Get Current State

```sql
-- dbt model: products_current
WITH ranked AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY product_id
            ORDER BY valid_ts DESC
        ) as rn
    FROM {{ source('application_data', 'products') }}
    WHERE operation != 'DELETE'
)
SELECT
    product_id,
    product_name,
    price_usd,
    valid_ts as last_updated_at
FROM ranked
WHERE rn = 1
```

### Option 2: Full History with Valid Ranges

```sql
-- dbt model: products_history
SELECT
    audit_id,
    product_id,
    product_name,
    price_usd,
    operation,
    valid_ts as valid_from,
    LEAD(valid_ts) OVER (
        PARTITION BY product_id
        ORDER BY valid_ts
    ) as valid_to
FROM {{ source('application_data', 'products') }}
```

These transformations are covered in the future `build/data-transformation/` section.

## Incremental Loading Details

dlt's incremental loading:

1. **First run**: Loads all data with `valid_ts >= initial_value`
2. **Subsequent runs**: Only loads data with `valid_ts > last_max_value`
3. **State storage**: Last value stored in `.dlt/` locally or destination in production

Check the current incremental state:

```python
# In Python
pipeline = create_pipeline()
state = pipeline.state
print(state.get("sources", {}).get("products", {}))
```

## Error Handling

Common issues and solutions:

### Connection Refused

```
psycopg2.OperationalError: connection refused
```

**Solution**: Check Clever Cloud is running and credentials are correct. The free tier may have connection limits.

### Max Connections Exceeded

```
FATAL: too many connections for role
```

**Solution**: Clever Cloud DEV tier allows max 5 connections. Close other connections (DBeaver, etc.) or wait. You can close all connections in the Clever Cloud UI.

### Schema Not Found

```
relation "raw_data.products" does not exist
```

**Solution**: Run the setup script to create the schema and table in PostgreSQL.

## Performance Considerations

For large tables:

1. **Add indexes** on the source table:
   ```sql
   CREATE INDEX idx_products_valid_ts ON raw_data.products(valid_ts);
   ```

2. **Batch processing**: dlt streams data, so memory usage is low

3. **Parallel extraction**: Not applicable for single-table extraction

4. **Network latency**: Consider where your worker runs relative to Clever Cloud

## Summary

You've built the products pipeline:

- [x] Understood Type 2 SCD format with operation tracking
- [x] Used dlt's `sql_database` source for PostgreSQL extraction
- [x] Configured incremental loading on `valid_ts`
- [x] Loaded to Snowflake with merge disposition

## What's Next

With all three pipelines built, you need to orchestrate them with Prefect. The next section covers wrapping dlt pipelines in Prefect flows with schedules, retries, and alerting.

Continue to [Prefect Orchestration](8-prefect-orchestration.md) →
