# Data Pipelines Repository

This repository contains dlt data pipelines orchestrated by Prefect. It handles all batch data ingestion from APIs, databases, and SaaS tools into Snowflake, following a three-layer architecture that separates extraction, pipeline configuration, and orchestration.

There is a skill available to help with adding new pipelines - `add-dlt-pipeline`.

---

## Repository Structure

```
data-pipelines/
├── sources/                    # dlt source definitions (extraction logic)
│   ├── __init__.py
│   ├── currencies/
│   │   ├── __init__.py
│   │   └── source.py
│   ├── exchange_rates/
│   │   ├── __init__.py
│   │   └── source.py
│   ├── products/
│   │   ├── __init__.py
│   │   └── source.py
│   └── hubspot/
│       ├── __init__.py
│       └── source.py
├── pipelines/                  # dlt pipeline configurations
│   ├── __init__.py
│   ├── currencies.py
│   ├── exchange_rates.py
│   ├── products.py
│   └── hubspot.py
├── flows/                      # Prefect flow definitions (orchestration)
│   ├── __init__.py
│   ├── currencies.py
│   ├── exchange_rates.py
│   ├── products.py
│   └── hubspot.py
├── utils/                      # Shared utilities
│   ├── __init__.py
│   └── vault_provider.py       # AWS Secrets Manager provider for dlt
├── .dlt/
│   ├── config.toml             # dlt runtime configuration
│   └── secrets.toml            # Local credentials (gitignored)
├── prefect.yaml                # Prefect deployment configuration
├── pyproject.toml              # Dependencies (managed by uv)
├── .envrc.example              # Template for local environment
└── .github/workflows/          # CI/CD for deploying flows
```

---

## Three-Layer Architecture

| Layer | Directory | Responsibility |
|-------|-----------|---------------|
| **Sources** | `sources/` | Extract data from APIs and databases (pure dlt, no orchestration) |
| **Pipelines** | `pipelines/` | Configure dlt pipeline: destination, dataset, credentials |
| **Flows** | `flows/` | Orchestrate with Prefect: schedules, retries, logging, alerting |

Each layer is independent. Flows import pipelines. Pipelines import sources. This separation means you can test extraction logic without Prefect and test pipeline configuration without scheduling.

---

## Key Conventions

### Source Pattern

Each source is a Python package under `sources/`:

```python
@dlt.source(section="{source_name}")
def source_name():
    """Extract data from Source Name."""
    return resource_name()

@dlt.resource(write_disposition="{disposition}")
def resource_name():
    """Fetch data."""
    yield data
```

- Use `@dlt.source(section=)` to group configuration in `.dlt/secrets.toml`
- Use `@dlt.resource` with explicit `write_disposition`
- Write dispositions: `replace` (reference data), `append` (events/logs), `merge` (with `primary_key` and `merge_key` for deduplication)

### Pipeline Pattern

Each pipeline configures destination and dataset:

```python
pipeline = dlt.pipeline(
    pipeline_name="{name}",
    destination="snowflake",
    dataset_name="{SCHEMA_NAME}",
)

load_info = pipeline.run(source())
```

- Use DuckDB for local testing via `DLT__DESTINATION=duckdb` environment variable
- Credentials resolved from `.dlt/secrets.toml` (local) or AWS Secrets Manager (CI/CD) via `vault_provider.py`

### Flow Pattern

Each flow wraps pipeline execution in Prefect tasks:

```python
@task(retries=3, retry_delay_seconds=[10, 30, 90])
def run_pipeline():
    from pipelines.name import run_name_pipeline
    return run_name_pipeline()

@flow(name="pipeline-name")
def pipeline_flow():
    run_pipeline()
```

- **Always** include retries: minimum 3 with exponential backoff
- Use `retry_delay_seconds` as a list for exponential intervals (e.g. `[10, 30, 90]`)
- Import pipeline functions inside the task (avoids circular imports)
- All flows deployed via `prefect.yaml`

### Credentials

| Context | Location | Format |
|---------|----------|--------|
| Local development | `.dlt/secrets.toml` | TOML sections matching source names |
| CI/CD | AWS Secrets Manager | JSON via `vault_provider.py` |
| Snowflake | Key-pair authentication | Private key in secrets |

---

## Destination Patterns

| Pattern | When to Use | Example |
|---------|-------------|---------|
| API → Snowflake | Direct, small/medium loads | currencies, products |
| API → S3 → Snowpipe | Large or historical data, append-only | exchange_rates |
| Verified source | Well-known SaaS APIs with pre-built extractors | HubSpot contacts |
| Database → Snowflake | SQL source extraction with incremental merge | products (PostgreSQL) |

---

## Common Operations

### Adding a New dlt Pipeline

Use the `add-dlt-pipeline` skill. The process is:

1. Create source package in `sources/{name}/`
2. Create pipeline config in `pipelines/{name}.py`
3. Create Prefect flow in `flows/{name}.py`
4. Add deployment to `prefect.yaml`
5. Test locally with DuckDB, then against Snowflake
6. Deploy via CI/CD

### Running Locally

```sh
# Test a pipeline with DuckDB (no Snowflake needed)
DLT__DESTINATION=duckdb python -c "from pipelines.name import run; run()"

# Test a Prefect flow
python flows/name.py

# Deploy all flows
prefect deploy --all
```

Use bare `prefect` and `dbt` commands - direnv activates `.venv` automatically.

---

## Safety Rules

- **Never** commit `.dlt/secrets.toml` or `.envrc` to git
- **Never** use `pip install` - always use `uv add` for dependencies
- **Always** test pipelines locally with DuckDB before running against Snowflake
- **Always** include retries in Prefect tasks (minimum 3 with exponential backoff)
- **Never** hard-code credentials in source code
- Use bare `prefect` commands - direnv activates `.venv` automatically

---

## Authentication

| Context | Method |
|---------|--------|
| Local development | `.dlt/secrets.toml` + `.envrc` (direnv activates `.venv`) |
| CI/CD | AWS Secrets Manager + GitHub Actions secrets |
| Snowflake | `SVC_DLT` with key-pair authentication |

---

## Style

- Use British English: organisation, customise, analyse, optimise
- Use spaced hyphens ( - ) for parenthetical statements, not em dashes
