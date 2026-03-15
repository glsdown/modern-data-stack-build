# dbt Concepts

On this page, you will:

- [x] Understand how this project uses dbt's core building blocks
- [x] Know where to go to learn dbt fundamentals in depth
- [x] See the layer structure, naming conventions, and packages we use

## Learning dbt

This page is an orientation to how we implement dbt in this project — it is not a dbt tutorial. If you are new to dbt, work through the official resources first:

- [dbt Learn](https://learn.getdbt.com/) — free interactive courses covering fundamentals
- [dbt Best Practices](https://docs.getdbt.com/best-practices) — the conventions this project follows
- [dbt Documentation](https://docs.getdbt.com/) — full reference

The rest of this page assumes you have a working understanding of models, sources, `ref()`, `source()`, tests, and materialisations.

## How dbt Fits in This Stack

dbt transforms raw data that has already been loaded into Snowflake by dlt, Snowpipe, and Airbyte. It does not move data between systems.

```
DLT database        ┐
SNOWPIPE database   ├──▶  dbt (dbt-transform repo)  ──▶  ANALYTICS database
AIRBYTE database    ┘          SELECT statements              clean models
```

dbt runs in a `dbt-transform` repository — separate from the `data-pipelines` repository that contains Prefect flows. Prefect triggers dbt after ingestion completes; it does not own the dbt codebase.

## Layer Structure

This project follows the [dbt Labs three-layer architecture](https://docs.getdbt.com/best-practices/how-we-structure/1-guide-overview):

| Layer | Directory | Materialisation | Purpose |
|-------|-----------|----------------|---------|
| Staging | `models/staging/` | View | One-to-one with raw source tables. Rename and cast columns only. |
| Intermediate | `models/intermediate/` | Ephemeral | Business logic, joins across sources. Not exposed to analysts. |
| Utilities | `models/utilities/` | Table | Shared reference models used across layers and tests (e.g. date spine). |
| Marts | `models/marts/` | Table | Final analytics tables. Consumed by BI tools and downstream models. |
| Reporting | `models/marts/reporting/` | Table/View | Curated subset published to `ANALYTICS.REPORTING` for BI tool access. |

The `REPORTING` schema is managed in Terraform (from the [Schemas](../data-warehouse/6-schemas.md) page). All other schemas are created dynamically by dbt.

## Naming Conventions

Following [dbt naming conventions](https://docs.getdbt.com/best-practices/how-we-structure/2-staging#staging-models):

| Layer | Pattern | Example |
|-------|---------|---------|
| Staging | `stg_{source}__{entity}` | `stg_dlt__exchange_rates` |
| Intermediate | `int_{entity}__{verb}` | `int_exchange_rates__unioned` |
| Fact tables | `fct_{event}` | `fct_exchange_rates` |
| Dimension tables | `dim_{entity}` | `dim_currencies` |
| Reporting | `rpt_{entity}` | `rpt_contacts` |

The double underscore (`__`) separates the source or entity name from the qualifier. This makes it immediately clear where a model sits in the lineage.

## `ref()` and `source()` — Project Rules

All models in this project must use `ref()` to reference other models and `source()` to reference raw tables. Hard-coded database and schema names are not permitted.

```sql
-- ✅ Correct
select * from {{ source('dlt_open_exchange_rates', 'exchange_rates') }}
select * from {{ ref('stg_dlt__exchange_rates') }}

-- ❌ Not permitted
select * from DLT.OPEN_EXCHANGE_RATES.EXCHANGE_RATES
select * from ANALYTICS_DEV.STAGING.STG_DLT__EXCHANGE_RATES
```

This rule is enforced automatically by `dbt_project_evaluator` in CI.

## The DAG

dbt builds a directed acyclic graph (DAG) from all `ref()` and `source()` calls. Models run in dependency order — dbt never builds a model before its upstream dependencies are ready.

```
source: DLT.exchange_rates     source: SNOWPIPE.exchange_rates
         │                               │
         ▼                               ▼
stg_dlt__exchange_rates    stg_snowpipe__exchange_rates
         │                               │
         └─────────────┬─────────────────┘
                       ▼
           int_exchange_rates__unioned
                       │
                       ▼
           fct_exchange_rates  (mart)
```

The full DAG is visible in the dbt docs site and the dbt Cloud IDE.

## Packages

This project installs three packages via `packages.yml`:

| Package | Why we use it |
|---------|--------------|
| [`dbt-utils`](https://hub.getdbt.com/dbt-labs/dbt_utils/latest/) | Utility macros: `generate_surrogate_key`, `date_spine`, `union_relations`, `pivot` |
| [`dbt_expectations`](https://hub.getdbt.com/calogica/dbt_expectations/latest/) | Extended test library: range checks, regex matching, row counts, type assertions |
| [`dbt_project_evaluator`](https://hub.getdbt.com/dbt-labs/dbt_project_evaluator/latest/) | Enforces project structure rules in CI: naming conventions, test coverage, documentation coverage, no direct source references in marts |

## Summary

- [x] dbt's training docs and best practices guide are the authoritative learning resources
- [x] This project uses staging → intermediate → marts → reporting layers
- [x] Naming follows `stg_`, `int_`, `fct_`, `dim_`, `rpt_` prefixes with double-underscore separators
- [x] `ref()` and `source()` are mandatory — enforced by `dbt_project_evaluator`
- [x] Packages: dbt-utils, dbt_expectations, dbt_project_evaluator

## What's Next

Compare dbt Core and dbt Cloud to choose the right deployment for your team.

Continue to [dbt Core vs dbt Cloud](2-deployment-options.md) →
