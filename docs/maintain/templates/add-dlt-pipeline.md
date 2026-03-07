---
name: add-dlt-pipeline
description: Add a new dlt data pipeline with Prefect orchestration, following the three-layer source/pipeline/flow pattern with DuckDB local testing and exponential retry backoff.
allowed-tools: Read, Write, Glob, Grep, Bash
---

# Add a New dlt Pipeline with Prefect Flow

This skill adds a new data pipeline following the three-layer architecture: source (extraction), pipeline (configuration), and flow (orchestration).

## When to Use

- Ingesting data from a new API or database
- Adding a new endpoint to an existing data source
- Setting up scheduled data loading for a new source

## Before You Start

Gather the following information:

1. **Source name** - identifier for the data source (e.g. `stripe`, `github`, `postgres_orders`)
2. **API or database details** - base URL, endpoints, connection string
3. **Authentication method** - API key, OAuth, database credentials
4. **Destination** - Snowflake direct or S3 (for Snowpipe auto-ingest)
5. **Write disposition** - `replace` (reference data), `append` (events), or `merge` (deduplication)
6. **Schedule** - frequency and time (cron expression or RRule)
7. **Target database and schema** - must already exist in Snowflake (created via Terraform)

## Reference: Existing Pipelines

Read the `sources/`, `pipelines/`, and `flows/` directories to see existing patterns:

| Pipeline | Source Type | Disposition | Destination | Schedule |
|----------|-----------|-------------|-------------|----------|
| currencies | REST API | replace | DLT.OPEN_EXCHANGE_RATES | Weekly |
| exchange_rates | REST API | append | S3 → Snowpipe | Daily |
| products | PostgreSQL | merge | DLT.APPLICATION_DATA | Daily |
| hubspot | dlt verified source | merge | DLT.HUBSPOT | Daily |

## Steps

### 1. Create Source Package

```sh
mkdir -p sources/{source_name}
touch sources/{source_name}/__init__.py
```

Create `sources/{source_name}/__init__.py`:

```python
"""{Source Name} source."""
from .source import {source_name}

__all__ = ["{source_name}"]
```

Create `sources/{source_name}/source.py`:

```python
"""{Source Name} dlt source."""
import dlt


@dlt.source(section="{source_name}")
def {source_name}():
    """Extract data from {Source Name}."""
    return {resource_name}()


@dlt.resource(
    write_disposition="{disposition}",
    # For merge: primary_key="{pk}", merge_key="{mk}"
)
def {resource_name}():
    """Fetch {resource} data from {Source Name}."""
    # API or database extraction logic here
    yield data
```

Key conventions:

- Use `@dlt.source(section="{source_name}")` to group credentials in `.dlt/secrets.toml`
- Use explicit `write_disposition` on every resource
- For merge operations, specify both `primary_key` and `merge_key`
- For incremental loading, use `dlt.sources.incremental("{timestamp_field}")`

### 2. Create Pipeline Configuration

Create `pipelines/{source_name}.py`:

```python
"""{Source Name} pipeline configuration."""
import dlt

from sources.{source_name} import {source_name}


def run_{source_name}_pipeline():
    """Run the {source_name} pipeline to Snowflake."""
    pipeline = dlt.pipeline(
        pipeline_name="{source_name}",
        destination="snowflake",
        dataset_name="{SCHEMA_NAME}",
    )

    load_info = pipeline.run({source_name}())
    print(load_info)
    return load_info


if __name__ == "__main__":
    run_{source_name}_pipeline()
```

For S3 destination (Snowpipe pattern), use `destination="filesystem"` with S3 bucket configuration instead.

### 3. Create Prefect Flow

Create `flows/{source_name}.py`:

```python
"""{Source Name} Prefect flow."""
from prefect import flow, task


@task(retries=3, retry_delay_seconds=[10, 30, 90])
def run_{source_name}():
    """Run the {source_name} dlt pipeline."""
    from pipelines.{source_name} import run_{source_name}_pipeline

    return run_{source_name}_pipeline()


@flow(name="{source_name}")
def {source_name}_flow():
    """Orchestrate {source_name} data ingestion."""
    run_{source_name}()


if __name__ == "__main__":
    {source_name}_flow()
```

Key conventions:

- **Always** include `retries=3` with `retry_delay_seconds=[10, 30, 90]` (exponential backoff)
- Import pipeline functions inside the task body (avoids circular imports and ensures clean module loading)
- Include `if __name__ == "__main__"` for local testing

### 4. Add Deployment to `prefect.yaml`

Add a new deployment entry:

```yaml
deployments:
  # ... existing deployments ...

  - name: {source_name}-{frequency}
    entrypoint: flows/{source_name}.py:{source_name}_flow
    work_pool:
      name: {work_pool_name}
    schedule:
      cron: "{cron_expression}"
      timezone: "UTC"
```

Common cron patterns:

- Daily at 06:00 UTC: `0 6 * * *`
- Hourly: `0 * * * *`
- Weekly on Monday at 06:00 UTC: `0 6 * * 1`

### 5. Add Credentials

**Local development** - add to `.dlt/secrets.toml`:

```toml
[sources.{source_name}]
api_key = "your-api-key-here"
```

**CI/CD** - create or update AWS Secrets Manager secret:

```sh
aws secretsmanager create-secret \
  --name "{source_name}/api-credentials" \
  --secret-string '{"api_key": "your-api-key-here"}' \
  --profile data-engineer
```

The `vault_provider.py` utility resolves secrets from AWS Secrets Manager at runtime in CI/CD.

### 6. Test Locally

```sh
# Test with DuckDB first (no Snowflake credentials needed)
DLT__DESTINATION=duckdb python pipelines/{source_name}.py

# Test the Prefect flow locally
python flows/{source_name}.py

# Test against Snowflake (requires credentials in .dlt/secrets.toml)
python pipelines/{source_name}.py
```

Always test with DuckDB first to verify extraction logic before loading to Snowflake.

### 7. Deploy

```sh
prefect deploy -n {source_name}-{frequency}
```

Or deploy all flows:

```sh
prefect deploy --all
```

## Pre-PR Checklist

- [ ] Source extracts data correctly (tested with DuckDB)
- [ ] Pipeline loads to Snowflake (tested locally)
- [ ] Flow runs with retries configured (3 retries, exponential backoff)
- [ ] Deployment added to `prefect.yaml` with schedule
- [ ] Credentials in `.dlt/secrets.toml` for local dev (gitignored)
- [ ] AWS Secrets Manager secret created or documented for CI/CD
- [ ] Target schema exists in Snowflake (created via Terraform `add-data-source` skill)

## Safety Rules

- **Never** commit `.dlt/secrets.toml` or `.envrc` to git
- **Never** use `pip install` - always use `uv add` for new dependencies
- **Always** test with DuckDB before running against Snowflake
- **Always** include retries (minimum 3) with exponential backoff in Prefect tasks
- **Never** hard-code credentials in source code - use `.dlt/secrets.toml` or AWS Secrets Manager
- Ensure the Snowflake database and schema exist (via Terraform) before running the pipeline
