# dbt Transformation Repository

This repository contains the dbt project for transforming raw data into analytics-ready models in Snowflake. It implements a four-layer architecture (staging, intermediate, marts, reporting) with strict naming conventions and testing requirements.

There are skills available to help with common maintenance tasks - `add-source-and-staging` and `add-mart-model`.

---

## Repository Structure

```
dbt-transform/
├── dbt_project.yml             # Project configuration and schema routing
├── packages.yml                # dbt packages (dbt_utils, dbt_expectations, etc.)
├── profiles.yml                # Generated locally by direnv, not committed
├── .envrc.example              # Template for local Snowflake credentials
├── pyproject.toml              # Python dependencies (managed by uv)
├── models/
│   ├── staging/
│   │   ├── dlt/
│   │   │   ├── _sources.yml
│   │   │   ├── stg_dlt__products.sql
│   │   │   ├── stg_dlt__products.yml
│   │   │   ├── stg_dlt__currencies.sql
│   │   │   └── stg_dlt__currencies.yml
│   │   ├── snowpipe/
│   │   │   ├── _sources.yml
│   │   │   ├── stg_snowpipe__exchange_rates.sql
│   │   │   └── stg_snowpipe__exchange_rates.yml
│   │   └── airbyte/
│   │       ├── _sources.yml
│   │       ├── stg_airbyte__contacts.sql
│   │       └── stg_airbyte__contacts.yml
│   ├── intermediate/
│   │   ├── products/
│   │   │   ├── int_products__current.sql
│   │   │   └── int_products__current.yml
│   │   └── contacts/
│   │       ├── int_contacts__enriched.sql
│   │       └── int_contacts__enriched.yml
│   └── marts/
│       ├── core/
│       │   ├── fct_exchange_rates.sql / .yml
│       │   ├── dim_currencies.sql / .yml
│       │   └── dim_products.sql / .yml
│       ├── crm/
│       │   ├── fct_contacts.sql / .yml
│       │   └── fct_contacts.yml
│       └── reporting/
│           ├── rpt_contacts.sql
│           └── rpt_contacts.yml
├── tests/                      # Custom singular tests
├── macros/                     # Custom macros
├── seeds/                      # Static reference data
└── .github/workflows/          # CI/CD (slim CI + deploy)
```

---

## Model Layers

| Layer | Schema | Materialisation | Naming Pattern | Purpose |
|-------|--------|-----------------|----------------|---------|
| Staging | `ANALYTICS.STAGING` | View | `stg_{source}__{table}` | Clean, rename, cast raw columns |
| Intermediate | (ephemeral) | Ephemeral | `int_{entity}__{verb}` | Business logic, deduplication |
| Marts | `ANALYTICS.MARTS` | Table / Incremental | `fct_{entity}` / `dim_{entity}` | Final analytics tables |
| Reporting | `ANALYTICS.REPORTING` | View / Table | `rpt_{entity}` | Curated subset for BI tools |

---

## Key Conventions

### Naming

- Sources: `stg_{source_system}__{table_name}` (double underscore separates source from table)
- Intermediate: `int_{entity}__{description}` (e.g. `int_products__current`)
- Facts: `fct_{entity}` (events, transactions, measurable)
- Dimensions: `dim_{entity}` (descriptive, reference data)
- Reporting: `rpt_{entity}` (presentation-layer views over marts)

### YAML Files

- **One YAML file per model** (not per directory) - e.g. `stg_dlt__products.yml` alongside `stg_dlt__products.sql`
- Sources defined in `_sources.yml` files alongside staging models (e.g. `models/staging/dlt/_sources.yml`)
- Every model must have a YAML file with a description and column-level tests

### Testing Requirements

- All primary keys: `unique` + `not_null` tests
- All foreign keys: `relationships` test to the referenced dimension
- Source freshness: `warn_after` + `error_after` on all source definitions
- Use `dbt_expectations` for complex validations (regex patterns, ranges, etc.)
- `dbt_project_evaluator` configured for convention enforcement

### Materialisation Rules

- **Staging**: Views (cheap, always current, no storage cost)
- **Intermediate**: Ephemeral (compiled into downstream queries, no storage)
- **Marts**: Tables for small datasets, incremental for large or growing fact tables
- **Reporting**: Views over mart tables (thin presentation layer)

### Incremental Models

For large fact tables, use incremental materialisation:

```sql
{{ config(
    materialized='incremental',
    unique_key='surrogate_key',
    on_schema_change='append_new_columns'
) }}

...

{% if is_incremental() %}
    where timestamp_column > (select max(timestamp_column) from {{ this }})
{% endif %}
```

---

## Common Operations

### Adding a New Source

1. Create `_sources.yml` in `models/staging/{source_system}/`
2. Create staging model SQL + YAML: `stg_{source}__{table}.sql` and `.yml`
3. Run: `dbt run -s stg_{source}__{table}`
4. Run: `dbt test -s stg_{source}__{table}`

### Adding a New Mart Model

1. Create model SQL + YAML in `models/marts/{domain}/`
2. Choose materialisation (`table` or `incremental`)
3. Run: `dbt run -s {model_name}`
4. Run: `dbt test -s {model_name}`

### Running dbt Locally

```sh
dbt deps          # Install packages
dbt debug         # Verify connection
dbt build         # Run models + tests
dbt run -s +model # Run a model and all upstream dependencies
dbt test -s model # Run tests for a specific model
```

Dev is the default target - never use `--target dev` explicitly.

---

## Safety Rules

- **Never** run dbt against production without state deferral (`--defer`)
- **Always** run `dbt build` (not just `dbt run`) to include tests
- **Never** skip tests in CI/CD pipelines
- **Never** use `pip install` or `uv pip install` - always use `uv add` for dependencies
- Use bare `dbt` commands - direnv activates `.venv` automatically
- Run `sqlfluff` and `yamllint` before committing (pre-commit hooks handle this)
- Use dev target for local development (the default - never specify `--target dev`)

---

## Authentication

| Context | Method |
|---------|--------|
| Local development | `.envrc` with Snowflake credentials, direnv activates `.venv` |
| CI/CD | AWS Secrets Manager for Snowflake credentials |
| Service account | `SVC_DBT` with `ANALYTICS_TRANSFORMER` role, key-pair auth |

---

## Style

- Use British English: materialisation, organisation, customise, analyse, optimise
- Use spaced hyphens ( - ) for parenthetical statements, not em dashes
