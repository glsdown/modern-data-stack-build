---
name: add-source-and-staging
description: Add a new dbt source definition and staging model, following the project's naming conventions (stg_{source}__{table}), testing standards (unique, not_null, freshness), and one-YAML-per-model pattern.
allowed-tools: Read, Write, Glob, Grep, Bash
---

# Add a New Source and Staging Model

This skill adds a dbt source definition and staging model for a new raw table, following the established conventions.

## When to Use

- A new raw table is available in a source database (DLT, SNOWPIPE, AIRBYTE, STREAMING)
- A new schema was added to an existing source database
- Setting up dbt for a newly ingested data source

## Before You Start

Gather the following information:

1. **Source system name** - the loader database: `dlt`, `snowpipe`, `airbyte`, or `streaming`
2. **Source database and schema** - e.g. `DLT.APPLICATION_DATA`
3. **Table name(s)** - the raw table name(s)
4. **Column names and types** - query the raw table to see what columns exist
5. **Primary key column** - the column (or columns) that uniquely identify a row
6. **Freshness requirements** - `warn_after` and `error_after` thresholds
7. **Loaded-at field** - e.g. `_dlt_load_time`, `_airbyte_extracted_at`, `_loaded_at`

## Reference: Existing Sources

Read `models/staging/` to see existing source definitions and staging models:

```
models/staging/
├── dlt/
│   ├── _sources.yml              # DLT.APPLICATION_DATA + DLT.OPEN_EXCHANGE_RATES
│   ├── stg_dlt__products.sql
│   ├── stg_dlt__products.yml
│   ├── stg_dlt__currencies.sql
│   └── stg_dlt__currencies.yml
├── snowpipe/
│   ├── _sources.yml              # SNOWPIPE.OPEN_EXCHANGE_RATES
│   ├── stg_snowpipe__exchange_rates.sql
│   └── stg_snowpipe__exchange_rates.yml
└── airbyte/
    ├── _sources.yml              # AIRBYTE.HUBSPOT
    ├── stg_airbyte__contacts.sql
    └── stg_airbyte__contacts.yml
```

## Steps

### 1. Create or Update Source Definition

If the source system directory does not exist, create it:

```sh
mkdir -p models/staging/{source_system}
```

Create or update `models/staging/{source_system}/_sources.yml`:

```yaml
version: 2

sources:
  - name: {source_system}_{schema_name}
    description: >
      Description of what data this source contains and how it is loaded.
    database: {DATABASE}
    schema: {SCHEMA}
    loader: {loader_tool}
    loaded_at_field: {timestamp_field}
    freshness:
      warn_after: {count: 25, period: hour}
      error_after: {count: 49, period: hour}
    tables:
      - name: {table_name}
        description: >
          Description of this table and what each row represents.
        columns:
          - name: {primary_key}
            description: Primary key.
```

If the `_sources.yml` already exists for this source system, add the new source group or table to it.

### 2. Create Staging Model SQL

Create `models/staging/{source_system}/stg_{source_system}__{table_name}.sql`:

```sql
with source as (

    select * from {{ source('{source_system}_{schema}', '{table_name}') }}

),

renamed as (

    select
        -- Primary key
        {pk_column} as {clean_pk_name},

        -- Dimensions
        {dimension_columns},

        -- Timestamps
        {timestamp_columns},

        -- Metadata
        {loaded_at_field} as loaded_at

    from source

)

select * from renamed
```

The staging model should:

- Rename columns to clean `snake_case` names
- Cast data types where the loader produces incorrect types
- Remove loader-internal metadata columns (`_dlt_id`, `_dlt_load_id`, `_airbyte_raw_id`, etc.)
- Keep a single `loaded_at` timestamp from the loader metadata
- **Not** add business logic - that belongs in intermediate or mart models

### 3. Create Model YAML

Create `models/staging/{source_system}/stg_{source_system}__{table_name}.yml`:

```yaml
version: 2

models:
  - name: stg_{source_system}__{table_name}
    description: >
      Staging model for {table_name} from {source_system}.
      Cleans column names, casts types, and removes loader metadata.
    columns:
      - name: {primary_key}
        description: Primary key.
        data_tests:
          - unique
          - not_null
      - name: {other_columns}
        description: Column description.
```

Every column must have a description. The primary key must have `unique` and `not_null` tests.

### 4. Validate

```sh
dbt run -s stg_{source_system}__{table_name}
dbt test -s stg_{source_system}__{table_name}
dbt source freshness -s source:{source_system}_{schema_name}
```

Verify:

- Model compiles and runs without errors
- All tests pass
- Source freshness check returns within thresholds

## Naming Rules

| Element | Convention | Example |
|---------|-----------|---------|
| Model name | `stg_{source}__{table}` (double underscore) | `stg_dlt__products` |
| Source name | `{source}_{schema}` (single underscore) | `dlt_application_data` |
| Column names | `snake_case`, no loader prefixes | `product_name` not `PRODUCT_NAME` |
| Metadata | Remove all `_dlt_*`, `_airbyte_*`, `_sdc_*` columns except `loaded_at` | Keep one timestamp |

## Testing Requirements

- Primary key: `unique` + `not_null`
- Source freshness: `warn_after` + `error_after`
- All columns: description in YAML file
- Additional tests as appropriate: `accepted_values`, `relationships`, `dbt_expectations`
