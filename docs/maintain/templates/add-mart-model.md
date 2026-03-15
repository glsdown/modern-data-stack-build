---
name: add-mart-model
description: Add a new mart model (fact table, dimension table, or reporting view) following the project's materialisation strategy, testing standards, and schema conventions.
allowed-tools: Read, Write, Glob, Grep, Bash
---

# Add a New Mart Model

This skill adds a new mart model to the dbt project, following the established conventions for facts, dimensions, and reporting views.

## When to Use

- Creating a new analytics table for BI tools or downstream consumers
- Building a fact table from staging or intermediate models
- Creating a dimension table for reference data
- Publishing a model to the REPORTING schema for BI tool access

## Before You Start

Gather the following information:

1. **Model type** - fact (`fct_`), dimension (`dim_`), or reporting (`rpt_`)
2. **Domain** - business area: `core`, `crm`, `finance`, etc.
3. **Source models** - which staging or intermediate models it references (use `ref()`)
4. **Materialisation** - `table` (small, static) or `incremental` (large, growing)
5. **For incremental** - `unique_key` and incremental strategy (`merge` or `delete+insert`)
6. **Whether to publish to REPORTING** - should BI users see this model?

## Reference: Existing Marts

Read `models/marts/` to see existing patterns:

```
models/marts/
├── core/
│   ├── fct_exchange_rates.sql / .yml   # Incremental fact table
│   ├── dim_currencies.sql / .yml       # Table dimension
│   └── dim_products.sql / .yml         # Table dimension
├── crm/
│   ├── fct_contacts.sql / .yml         # Table fact
└── reporting/
    ├── rpt_contacts.sql / .yml         # View over fct_contacts
```

## Steps

### 1. Create Model SQL

Create `models/marts/{domain}/{model_name}.sql`.

**For table materialisation** (small or static datasets):

```sql
{{ config(
    materialized='table'
) }}

with {source_cte} as (

    select * from {{ ref('{upstream_model}') }}

),

final as (

    select
        {columns}
    from {source_cte}

)

select * from final
```

**For incremental materialisation** (large or growing fact tables):

```sql
{{ config(
    materialized='incremental',
    unique_key='{unique_key}',
    on_schema_change='append_new_columns'
) }}

with source as (

    select * from {{ ref('{upstream_model}') }}
    {% if is_incremental() %}
        where {timestamp_column} > (select max({timestamp_column}) from {{ this }})
    {% endif %}

),

final as (

    select
        {columns}
    from source

)

select * from final
```

### 2. Create Model YAML

Create `models/marts/{domain}/{model_name}.yml`:

```yaml
version: 2

models:
  - name: {model_name}
    description: >
      Description of the model and its business purpose.
      Explain what each row represents and the grain of the table.
    columns:
      - name: {primary_key}
        description: Primary key (surrogate or natural).
        data_tests:
          - unique
          - not_null
      - name: {foreign_key}
        description: Foreign key to {referenced_model}.
        data_tests:
          - relationships:
              to: ref('{referenced_model}')
              field: {referenced_column}
      - name: {other_columns}
        description: Column description.
```

### 3. Create Reporting View (if Publishing to BI)

If the model should be visible to BI tools and the `ANALYTICS_REPORTER` role, create a reporting view.

Create `models/marts/reporting/rpt_{entity}.sql`:

```sql
/*
Reporting view over {source_mart_model}.
Published to ANALYTICS.REPORTING for BI tool access.
*/

select * from {{ ref('{mart_model}') }}
```

Create `models/marts/reporting/rpt_{entity}.yml`:

```yaml
version: 2

models:
  - name: rpt_{entity}
    description: >
      Reporting view over {mart_model}. Published to the REPORTING schema
      for BI tool access via the ANALYTICS_REPORTER role.
    columns:
      - name: {primary_key}
        description: Primary key.
        data_tests:
          - unique
          - not_null
```

### 4. Validate

```sh
dbt run -s {model_name}
dbt test -s {model_name}
```

For incremental models, also verify the incremental logic:

```sh
# Full refresh to build the complete table
dbt run -s {model_name} --full-refresh

# Run again to test incremental behaviour
dbt run -s {model_name}
```

## Schema Mapping

| Directory | Schema | Access Role |
|-----------|--------|-------------|
| `models/marts/core/` | `ANALYTICS.MARTS` | `ANALYTICS_DEVELOPER` |
| `models/marts/crm/` | `ANALYTICS.MARTS` | `ANALYTICS_DEVELOPER` |
| `models/marts/{domain}/` | `ANALYTICS.MARTS` | `ANALYTICS_DEVELOPER` |
| `models/marts/reporting/` | `ANALYTICS.REPORTING` | `ANALYTICS_REPORTER` |

The schema routing is configured in `dbt_project.yml` - you do not need to set schemas in model configs.

## Naming Rules

| Model Type | Prefix | Description | Example |
|-----------|--------|-------------|---------|
| Fact table | `fct_` | Events, transactions, measurable actions | `fct_exchange_rates` |
| Dimension table | `dim_` | Descriptive, reference, slowly changing | `dim_currencies` |
| Reporting view | `rpt_` | Curated subset for BI tools | `rpt_contacts` |

## Testing Requirements

- Primary key: `unique` + `not_null`
- Foreign keys: `relationships` test to the referenced dimension
- All columns: description in YAML file
- Incremental models: verify both full-refresh and incremental runs work correctly
