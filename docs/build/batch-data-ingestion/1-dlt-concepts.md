# dlt Concepts

On this page, you will:

- [x] Understand dlt's core building blocks
- [x] Learn how sources, resources, and pipelines work together
- [x] Understand schema evolution and incremental loading
- [x] Understand how pipeline state works in production

## Overview

dlt (data load tool) is built around a few core concepts that compose together to create data pipelines. Understanding these concepts will help you build and debug pipelines effectively.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            dlt PIPELINE                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐                              ┌─────────────────┐       │
│  │     SOURCE      │                              │   DESTINATION   │       │
│  │  ┌───────────┐  │                              │                 │       │
│  │  │ Resource  │  │     ┌─────────────────┐      │   Snowflake     │       │
│  │  │ (table 1) │──┼────▶│    Pipeline     │─────▶│   S3            │       │
│  │  ├───────────┤  │     │                 │      │   BigQuery      │       │
│  │  │ Resource  │──┼────▶│  • Extract      │      │   DuckDB        │       │
│  │  │ (table 2) │  │     │  • Normalise    │      │   etc.          │       │
│  │  ├───────────┤  │     │  • Load         │      │                 │       │
│  │  │ Resource  │──┼────▶│                 │      │                 │       │
│  │  │ (table 3) │  │     └─────────────────┘      └─────────────────┘       │
│  │  └───────────┘  │                                                        │
│  └─────────────────┘                                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Sources and Resources

!!! tip
    Don't worry about understanding the specific syntax here - the code blocks are to provide an overview only. We'll introduce the syntax in more detail later.

### Resources

A **resource** is a function that yields data. It typically represents a single table or endpoint. Resources are decorated with `@dlt.resource`:

```python
import dlt

@dlt.resource(write_disposition="replace")
def exchange_rates():
    """Fetch current exchange rates from API."""
    response = requests.get("https://api.example.com/rates")
    yield response.json()["rates"]
```

Key resource options:

| Option | Description | Example |
|--------|-------------|---------|
| `name` | Table name in destination | `name="daily_rates"` |
| `write_disposition` | How to handle existing data | `"replace"`, `"append"`, `"merge"` |
| `primary_key` | Column(s) for deduplication | `primary_key="id"` |
| `merge_key` | Column(s) for merge operations | `merge_key="date"` |

### Sources

A **source** is a collection of related resources. It groups resources that come from the same system:

```python
@dlt.source
def open_exchange_rates(api_key: str):
    """Source for Open Exchange Rates API."""

    @dlt.resource(write_disposition="merge", merge_key="date")
    def latest_rates():
        response = requests.get(
            f"https://openexchangerates.org/api/latest.json?app_id={api_key}"
        )
        data = response.json()
        yield {
            "date": data["timestamp"],
            "base": data["base"],
            "rates": data["rates"]
        }

    @dlt.resource(write_disposition="replace")
    def currencies():
        response = requests.get(
            f"https://openexchangerates.org/api/currencies.json?app_id={api_key}"
        )
        yield from [
            {"code": code, "name": name}
            for code, name in response.json().items()
        ]

    return latest_rates, currencies
```

When you call a source, you get back its resources. You can then select which resources to load:

```python
# Load all resources
source = open_exchange_rates(api_key="xxx")
pipeline.run(source)

# Load only specific resources
pipeline.run(source.with_resources("latest_rates"))
```

## Pipelines

A **pipeline** is the execution context that moves data from source to destination. It handles:

- **Extraction** - Calling resources and collecting data
- **Normalisation** - Flattening nested structures into tables
- **Loading** - Writing data to the destination

```python
pipeline = dlt.pipeline(
    pipeline_name="exchange_rates",
    destination="snowflake",
    dataset_name="open_exchange_rates"  # Schema name in Snowflake
)

# Run the pipeline
load_info = pipeline.run(open_exchange_rates(api_key="xxx"))
print(load_info)
```

### Pipeline State

Pipelines maintain state between runs. This enables:

- **Incremental loading** - Track what's been loaded, fetch only new data
- **Deduplication** - Remember primary keys to avoid duplicates
- **Recovery** - Resume from failures

State is stored locally by default (in the `.dlt/` folder) and is also automatically persisted to the destination.

### Remote State for Production

In production, pipelines typically run on ephemeral infrastructure (containers, CI/CD workers) where the local filesystem is wiped between runs. dlt handles this automatically:

1. **State is saved to the destination**: After each successful load, dlt writes the pipeline state to a `_dlt_pipeline_state` table at the destination (e.g. Snowflake).
2. **State is restored automatically**: When a pipeline starts on a clean filesystem, dlt detects the missing local state and restores it from the destination.
3. **State is identified by three values**: `pipeline_name`, destination location, and `dataset_name`. As long as these remain consistent between runs, state is preserved.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       PIPELINE STATE LIFECYCLE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Run 1 (Container A)                     Run 2 (Container B)                │
│  ────────────────────                    ────────────────────               │
│                                                                             │
│  1. No local state found                 1. No local state found            │
│  2. Check destination ──▶ Not found      2. Check destination ──▶ Found!    │
│  3. Start fresh                          3. Restore state                   │
│  4. Extract + Load data                  4. Extract only NEW data           │
│  5. Save state to destination            5. Save updated state              │
│                                                                             │
│              ┌─────────────────────────────────--─┐                         │
│              │    _dlt_pipeline_state table       │                         │
│              │    (in Snowflake / destination)    │                         │
│              │                                    │                         │
│              │  pipeline_name: exchange_rates     │                         │
│              │  dataset_name: open_exchange_rates │                         │
│              │  state: {last incremental values}  │                         │
│              └──────────────────────────────────--┘                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

This means:

- **No manual state management** is needed for production deployments
- **Incremental loading works** across container restarts and re-deployments
- **Multiple environments** (dev, staging, prod) maintain independent state via different `dataset_name` values

!!! tip "Consistent Naming"
    Always use the same `pipeline_name` and `dataset_name` for a given pipeline. If you change these, dlt treats it as a new pipeline and starts from scratch.

If you're running pipelines with persistent storage (e.g. a dedicated VM), you can disable remote state restoration by setting `restore_from_destination=false` in `.dlt/config.toml` to avoid the overhead.

## Write Dispositions

The `write_disposition` controls how dlt handles existing data:

### Replace

Drops and recreates the table on each run. Use for:

- Dimension tables that should be fully refreshed
- Small reference data
- Development/testing

```python
@dlt.resource(write_disposition="replace")
def currencies():
    yield from fetch_all_currencies()
```

### Append

Adds new rows without modifying existing data. Use for:

- Event/log data
- Time-series data
- When you handle deduplication elsewhere

```python
@dlt.resource(write_disposition="append")
def events():
    yield from fetch_new_events()
```

### Merge

Updates existing rows and inserts new ones based on a key. Use for:

- Slowly changing dimensions
- Data that gets updated at source
- Idempotent pipelines

```python
@dlt.resource(
    write_disposition="merge",
    primary_key="id",
    merge_key="id"
)
def products():
    yield from fetch_products()
```

With merge, dlt generates `MERGE` statements (or equivalent) for your destination.

## Schema Evolution

dlt automatically handles schema changes:

1. **New columns** - Added to the table automatically
2. **Type changes** - Widened when possible (e.g. int → bigint)
3. **Nested data** - Flattened into separate tables with foreign keys

```python
# First run - creates table with columns: id, name
yield {"id": 1, "name": "Product A"}

# Second run - automatically adds 'price' column
yield {"id": 2, "name": "Product B", "price": 9.99}
```

### Nested Data Normalisation

dlt flattens nested structures into related tables:

```python
# Source data
yield {
    "order_id": 123,
    "customer": "John",
    "items": [
        {"product": "Widget", "qty": 2},
        {"product": "Gadget", "qty": 1}
    ]
}

# Creates two tables:
# 1. orders (order_id, customer)
# 2. orders__items (order_id, product, qty)
```

## Incremental Loading

For large datasets, you don't want to reload everything on each run. dlt supports incremental loading:

```python
@dlt.resource(
    write_disposition="merge",
    primary_key="id"
)
def orders(
    updated_after: dlt.sources.incremental[str] = dlt.sources.incremental(
        "updated_at",
        initial_value="2024-01-01"
    )
):
    """Fetch orders updated since last run."""
    last_value = updated_after.last_value

    for order in api.get_orders(updated_after=last_value):
        yield order
```

dlt tracks the maximum value of the incremental field and uses it on the next run.

## Destinations

dlt supports many destinations out of the box:

| Destination | Use Case |
|-------------|----------|
| `snowflake` | Production data warehouse |
| `bigquery` | Google Cloud data warehouse |
| `redshift` | AWS data warehouse |
| `duckdb` | Local development and testing |
| `filesystem` | S3, GCS, Azure Blob (for staging) |
| `postgres` | PostgreSQL database |

Each destination has specific configuration. For Snowflake:

```python
pipeline = dlt.pipeline(
    pipeline_name="my_pipeline",
    destination="snowflake",
    dataset_name="my_schema"
)
```

Credentials are loaded from environment variables or `.dlt/secrets.toml`:

```toml
# .dlt/secrets.toml
[destination.snowflake.credentials]
database = "DLT"
warehouse = "LOADING"
role = "DLT_LOADER"
username = "SVC_DLT"
password = "xxx"
host = "xxx.snowflakecomputing.com"
```

## Configuration

dlt uses a layered configuration system:

1. **secrets.toml** - Sensitive values (API keys, passwords)
2. **config.toml** - Non-sensitive configuration
3. **Environment variables** - Override any config value

```
.dlt/
├── secrets.toml     # API keys, database passwords
├── config.toml      # Pipeline settings, timeouts
└── pipeline_name/   # Pipeline state (auto-managed)
```

In production, you'll typically use environment variables or a secrets manager instead of files. dlt uses a specific naming convention for environment variables — sections are separated with double underscores and names are capitalised:

| Config Path | Environment Variable |
|-------------|---------------------|
| `destination.snowflake.credentials.database` | `DESTINATION__SNOWFLAKE__CREDENTIALS__DATABASE` |
| `sources.open_exchange_rates.api_key` | `SOURCES__OPEN_EXCHANGE_RATES__API_KEY` |

dlt also supports **vault providers** for fetching secrets from external systems like AWS Secrets Manager. We'll set this up in [Project Setup](2-project-setup.md).

## Summary

You've learned dlt's core concepts:

- [x] **Resources** - Functions that yield data (tables)
- [x] **Sources** - Collections of related resources
- [x] **Pipelines** - Execute extraction, normalisation, and loading
- [x] **Write dispositions** - Control how data is written (replace, append, merge)
- [x] **Schema evolution** - Automatic handling of schema changes
- [x] **Incremental loading** - Load only new/changed data
- [x] **Remote state** - Pipeline state persists automatically in production

## What's Next

Now that you understand the concepts, let's set up a project structure for your dlt pipelines.

Continue to [Project Setup](2-project-setup.md) →
