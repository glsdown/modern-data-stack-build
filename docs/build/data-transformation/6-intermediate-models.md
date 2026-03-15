# Intermediate Models

On this page, you will:

- [x] Understand the role of the intermediate layer in the transformation pipeline
- [x] Build intermediate models that combine and reshape staging data
- [x] Apply business logic before it reaches the mart layer
- [x] Transform Type 2 SCD data to current state

## Overview

The intermediate layer sits between staging and marts. It handles business logic that would make marts too complex to read, and joins that are shared across multiple downstream models.

Intermediate models are materialised as **ephemeral** by default — they compile into CTEs that are inlined into the mart models that reference them, producing no Snowflake objects. When a model is complex enough to warrant its own table for query performance, change its materialisation to `table` or `incremental`.

```
Staging (views)                Intermediate (ephemeral)          Marts (tables)
───────────────                ─────────────────────────         ──────────────

stg_snowpipe__exchange_rates ────▶  (referenced directly by marts)

stg_dlt__currencies          ────▶  (referenced directly by marts)

stg_dlt__products            ────▶  int_products__current        ──▶  fct_products

stg_airbyte__contacts        ────▶  int_contacts__enriched       ──▶  fct_contacts
```

## Naming Conventions

Intermediate models follow the pattern `int_{entity}__{verb}` or `int_{entity}__{transformation}`:

| Model | What it does |
|-------|-------------|
| `int_products__current` | Gets the current state of products from Type 2 SCD |
| `int_contacts__enriched` | Enriches contacts with calculated fields |
| `int_exchange_rates__pivoted` | Pivots rates from long to wide format |

The double underscore (`__`) separates the entity from the transformation verb, consistent with the staging naming convention.

## Create the Intermediate Directory Structure

```sh
mkdir -p models/intermediate
```

Intermediate models live in a flat `intermediate/` directory, not split by source. Unlike staging (which mirrors source structure), intermediate models combine across sources and don't belong to any single origin.

## Build Intermediate Models

### `int_products__current`

The products staging model contains Type 2 SCD data — multiple rows per product showing the history of changes. This intermediate model extracts only the current state of each product.

Create `models/intermediate/int_products__current.sql`:

```sql
/*
Extracts the current state of products from Type 2 SCD history.

Takes the most recent non-DELETE operation for each product_id.
DELETE operations are excluded since they represent products that no longer exist.
*/

with products_history as (

    select * from {{ ref('stg_dlt__products') }}

),

-- Get the most recent version of each product
current_products as (

    select *
    from products_history
    where operation != 'DELETE'
    qualify row_number() over (
        partition by product_id
        order by valid_ts desc
    ) = 1

)

select
    product_id,
    product_name,
    price_usd,
    operation,
    valid_ts as last_updated_at,
    loaded_at

from current_products
```

Create `models/intermediate/int_products__current.yml`:

```yaml
version: 2

models:
  - name: int_products__current
    description: >
      Current state of products extracted from Type 2 SCD history.
      One row per active product (excludes deleted products).
    columns:
      - name: product_id
        description: Product business key.
        tests:
          - unique
          - not_null
      - name: product_name
        description: Current product name.
        tests:
          - not_null
      - name: price_usd
        description: Current product price in USD.
        tests:
          - not_null
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0
              strictly: true
      - name: last_updated_at
        description: When this product was last updated.
        tests:
          - not_null
```

### `int_contacts__enriched`

Enrich contacts with derived fields before building the mart:

Create `models/intermediate/int_contacts__enriched.sql`:

```sql
/*
Enriches contacts with derived fields:
- Full name combining first and last name
- Days since creation and last modification
- Contact age bucket for segmentation
*/

with contacts as (

    select * from {{ ref('stg_airbyte__contacts') }}

),

enriched as (

    select
        -- identifiers
        contact_id,
        email,

        -- name
        first_name,
        last_name,
        trim(coalesce(first_name, '') || ' ' || coalesce(last_name, ''))
            as full_name,

        -- lifecycle
        lifecycle_stage,

        -- derived metrics
        datediff(
            'day', created_at, current_timestamp()
        )                                               as days_since_created,

        datediff(
            'day', last_modified_at, current_timestamp()
        )                                               as days_since_modified,

        -- segmentation bucket
        case
            when datediff('day', created_at, current_timestamp()) <= 30
                then 'new'
            when datediff('day', created_at, current_timestamp()) <= 180
                then 'recent'
            else 'established'
        end                                             as contact_age_bucket,

        -- timestamps
        created_at,
        last_modified_at,
        airbyte_extracted_at

    from contacts

)

select * from enriched
```

Create `models/intermediate/int_contacts__enriched.yml`:

```yaml
version: 2

models:
  - name: int_contacts__enriched
    description: >
      HubSpot contacts enriched with derived fields including full name,
      days since creation, and contact age bucket for segmentation.
    columns:
      - name: contact_id
        description: HubSpot contact ID.
        tests:
          - unique
          - not_null
      - name: email
        description: Contact email address (lowercased and trimmed).
        tests:
          - not_null
      - name: full_name
        description: Full name derived from first_name and last_name.
      - name: days_since_created
        description: Number of days since the contact was created in HubSpot.
        tests:
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0
      - name: days_since_modified
        description: Number of days since the contact was last modified.
        tests:
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0
      - name: contact_age_bucket
        description: Segmentation bucket based on days since creation.
        tests:
          - accepted_values:
              values: ['new', 'recent', 'established']
```

### Optional: `int_exchange_rates__pivoted`

Some downstream consumers want exchange rates in wide format — one row per date with columns for each currency. This is optional but demonstrates `dbt_utils.pivot`.

!!! note "Why exchange rates come from two sources"
    In this guide, exchange rates are ingested via two separate paths: the dlt pipeline (direct API → `DLT` database) and S3 → Snowpipe (`SNOWPIPE` database). This is intentional - it demonstrates both batch ingestion patterns using a single real-world dataset. In a production system you would pick one ingestion method. The `stg_dlt__exchange_rates` and `stg_snowpipe__exchange_rates` staging models cover the same data; an `int_exchange_rates__unioned` model (shown in the section overview) would union both before marts consume them. The pivoted model below reads only from the Snowpipe source for simplicity.

Create `models/intermediate/int_exchange_rates__pivoted.sql`:

```sql
/*
Pivots exchange rates from long format (one row per currency per date)
to wide format (one row per date, one column per major currency).

Only includes the major currencies most commonly used in reporting.
*/

{% set major_currencies = ['GBP', 'EUR', 'JPY', 'CAD', 'AUD', 'CHF'] %}

with exchange_rates as (

    select
        rate_date,
        target_currency,
        exchange_rate

    from {{ ref('stg_snowpipe__exchange_rates') }}
    where target_currency in ({{ "'" + "', '".join(major_currencies) + "'" }})

)

select
    rate_date,
    {{ dbt_utils.pivot(
        'target_currency',
        major_currencies,
        agg='max',
        then_value='exchange_rate',
        prefix='rate_',
        suffix=''
    ) }}
from exchange_rates
group by rate_date
order by rate_date
```

Create `models/intermediate/int_exchange_rates__pivoted.yml`:

```yaml
version: 2

models:
  - name: int_exchange_rates__pivoted
    description: >
      Exchange rates pivoted to wide format with one column per major currency.
      One row per date. Only includes commonly used currencies.
    columns:
      - name: rate_date
        description: The date for which exchange rates apply.
        tests:
          - unique
          - not_null
      - name: rate_gbp
        description: GBP exchange rate.
      - name: rate_eur
        description: EUR exchange rate.
      - name: rate_jpy
        description: JPY exchange rate.
      - name: rate_cad
        description: CAD exchange rate.
      - name: rate_aud
        description: AUD exchange rate.
      - name: rate_chf
        description: CHF exchange rate.
```

!!! note "Materialise this as a table"
    If this pivoted model is queried frequently, override the ephemeral default with `+materialized: table` in `dbt_project.yml` or a model-level config block:

    ```sql
    {{ config(materialized='table') }}
    ```

## Run the Intermediate Models

```sh
# Build intermediate models and their upstream dependencies
dbt build --select +intermediate

# Build a specific model and its upstreams
dbt build --select +int_products__current
```

Because intermediate models are ephemeral, `dbt run --select intermediate` will do nothing — ephemeral models are only built when a downstream model references them. Use `dbt build` (which runs both `dbt run` and `dbt test`) with a downstream model selected:

```sh
# This will build the intermediate and the downstream mart together
dbt build --select +fct_products
```

## When to Use Intermediate vs Staging

| Layer | Use for |
|-------|--------|
| **Staging** | Renaming, casting, deduplicating raw columns — one model per source table |
| **Intermediate** | Business logic, joining across sources, reshaping (pivot, unpivot, aggregation), extracting current state from SCD |
| **Marts** | Final analytics tables ready for BI tools — no complex logic here |

If you find yourself putting a complex `JOIN` or `CASE WHEN` directly in a mart model, move it to an intermediate model instead. Marts should read cleanly.

## Summary

You've built the intermediate layer:

- [x] Extracted current state from Type 2 SCD products data
- [x] Enriched contacts with derived fields and segmentation buckets
- [x] (Optional) Pivoted exchange rates to wide format
- [x] Documented all intermediate models with separate YAML files per model

## What's Next

Build the mart layer — the final analytics tables that downstream tools consume.

Continue to [Mart Models](7-mart-models.md) →
