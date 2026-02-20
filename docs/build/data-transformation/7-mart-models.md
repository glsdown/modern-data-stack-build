# Mart Models

On this page, you will:

- [x] Build fact and dimension tables in the marts layer
- [x] Configure incremental materialisation for large fact tables
- [x] Publish curated models to the REPORTING schema for BI tool access
- [x] Understand the difference between MARTS and REPORTING schemas

## Overview

Mart models are the final analytics tables — clean, well-tested, and ready for BI tools or downstream consumers. They sit in two schemas:

| Schema | Purpose | Access |
|--------|---------|--------|
| `ANALYTICS.MARTS` | All final models: facts, dimensions, aggregates | `ANALYTICS_DEVELOPER` |
| `ANALYTICS.REPORTING` | Curated subset for BI tools and dashboards | `ANALYTICS_REPORTER` |

Not everything in MARTS is published to REPORTING. REPORTING contains the models you explicitly want BI users to see — typically presentation-ready views or tables over the mart layer, without internal or exploratory models.

```
Intermediate (ephemeral)            Marts (tables)            Reporting (views/tables)
─────────────────────────           ──────────────            ───────────────────────

stg_snowpipe__exchange_rates ────▶  fct_exchange_rates
                                    (incremental)

stg_dlt__currencies          ────▶  dim_currencies

int_products__current        ────▶  dim_products
                                                             ┌────────────────────────┐
int_contacts__enriched       ────▶  fct_contacts   ────────▶ │  rpt_contacts          │
                                                             │  (reporting view)      │
                                                             └────────────────────────┘
```

## Directory Structure

```
models/marts/
├── core/
│   ├── fct_exchange_rates.sql
│   ├── fct_exchange_rates.yml
│   ├── dim_currencies.sql
│   ├── dim_currencies.yml
│   ├── dim_products.sql
│   └── dim_products.yml
├── crm/
│   ├── fct_contacts.sql
│   └── fct_contacts.yml
└── reporting/
    ├── rpt_contacts.sql
    └── rpt_contacts.yml
```

The `reporting/` subdirectory is where models destined for `ANALYTICS.REPORTING` live. They reference models from `core/` or `crm/` via `ref()` — they are presentation layers over the mart tables, not new business logic.

## Build Core Mart Models

### `fct_exchange_rates`

Exchange rates are a good candidate for incremental materialisation — the table grows daily, and historical rows never change. Incremental models run faster because they only process new or updated rows instead of rebuilding the entire table.

Create `models/marts/core/fct_exchange_rates.sql`:

```sql
/*
Daily exchange rates fact table.
One row per target currency per date.

Materialised incrementally based on rate_date to handle growing data efficiently.
Surrogate key generated from date + currency pair.
*/

{{ config(
    materialized='incremental',
    unique_key='exchange_rate_id',
    on_schema_change='append_new_columns'
) }}

with exchange_rates as (

    select * from {{ ref('stg_snowpipe__exchange_rates') }}

    {% if is_incremental() %}
    -- Only process new data on incremental runs
    where rate_date > (select max(rate_date) from {{ this }})
    {% endif %}

),

currencies as (

    select * from {{ ref('stg_dlt__currencies') }}

),

final as (

    select
        -- surrogate key
        {{ dbt_utils.generate_surrogate_key(['r.rate_date', 'r.target_currency']) }}
            as exchange_rate_id,

        -- dates
        r.rate_date,
        year(r.rate_date)   as rate_year,
        month(r.rate_date)  as rate_month,

        -- currencies
        r.base_currency,
        r.target_currency,
        c.currency_name,

        -- measures
        r.exchange_rate,
        round(1.0 / nullif(r.exchange_rate, 0), 6) as inverse_rate,

        -- metadata
        r.loaded_at

    from exchange_rates r
    left join currencies c
        on r.target_currency = c.currency_code

)

select * from final
```

Create `models/marts/core/fct_exchange_rates.yml`:

```yaml
version: 2

models:
  - name: fct_exchange_rates
    description: >
      Daily USD base exchange rates for all currencies loaded via Snowpipe.
      One row per target currency per date.
      Materialised incrementally based on rate_date.
    config:
      materialized: incremental
      unique_key: exchange_rate_id
      on_schema_change: append_new_columns
    columns:
      - name: exchange_rate_id
        description: Surrogate key derived from rate_date and target_currency.
        tests:
          - unique
          - not_null
      - name: rate_date
        description: The date for which the exchange rate applies.
        tests:
          - not_null
      - name: base_currency
        description: Base currency. Always USD.
        tests:
          - not_null
          - accepted_values:
              values: ['USD']
      - name: target_currency
        description: Target currency ISO code.
        tests:
          - not_null
      - name: exchange_rate
        description: Exchange rate from USD to target currency.
        tests:
          - not_null
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0
              strictly: true
      - name: inverse_rate
        description: Inverse exchange rate (target currency to USD).
```

!!! note "Incremental configuration"
    - **`materialized='incremental'`**: Build as incremental model (append new rows, update existing)
    - **`unique_key='exchange_rate_id'`**: Identifies rows for updates (used with `merge` strategy)
    - **`on_schema_change='append_new_columns'`**: Automatically add new columns when schema changes
    - **`is_incremental()` filter**: Only processes rows with `rate_date` greater than the maximum date already in the table

    On the first run, dbt builds the full table. On subsequent runs, it only processes new dates.

### `dim_currencies`

Create `models/marts/core/dim_currencies.sql`:

```sql
/*
Currency reference dimension.
One row per currency code with the full currency name.
*/

with currencies as (

    select * from {{ ref('stg_dlt__currencies') }}

),

final as (

    select
        -- surrogate key
        {{ dbt_utils.generate_surrogate_key(['currency_code']) }}
            as currency_id,

        -- attributes
        currency_code,
        currency_name,

        -- flags
        case
            when currency_code in ('USD', 'EUR', 'GBP', 'JPY', 'AUD', 'CAD', 'CHF')
            then true
            else false
        end as is_major_currency

    from currencies

)

select * from final
```

Create `models/marts/core/dim_currencies.yml`:

```yaml
version: 2

models:
  - name: dim_currencies
    description: >
      Currency reference dimension with ISO codes and full currency names.
      One row per currency. Includes a flag for major currencies.
    columns:
      - name: currency_id
        description: Surrogate key for currency.
        tests:
          - unique
          - not_null
      - name: currency_code
        description: ISO 4217 currency code (e.g. GBP, EUR, USD).
        tests:
          - unique
          - not_null
      - name: currency_name
        description: Full currency name (e.g. British Pound Sterling).
        tests:
          - not_null
      - name: is_major_currency
        description: True if this is a commonly used major currency.
        tests:
          - not_null
          - accepted_values:
              values: [true, false]
```

### `dim_products`

Create `models/marts/core/dim_products.sql`:

```sql
/*
Product dimension showing the current state of all active products.
Built from the Type 2 SCD products data via the int_products__current intermediate model.
*/

with products as (

    select * from {{ ref('int_products__current') }}

),

final as (

    select
        -- surrogate key
        {{ dbt_utils.generate_surrogate_key(['product_id']) }}
            as product_key,

        -- business key
        product_id,

        -- attributes
        product_name,
        price_usd,

        -- metadata
        last_updated_at,
        loaded_at

    from products

)

select * from final
```

Create `models/marts/core/dim_products.yml`:

```yaml
version: 2

models:
  - name: dim_products
    description: >
      Product dimension showing the current state of all active products.
      One row per product. Historical versions and deleted products are excluded.
    columns:
      - name: product_key
        description: Surrogate key for product dimension.
        tests:
          - unique
          - not_null
      - name: product_id
        description: Product business key from source system.
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
        description: When this product was last updated in the source system.
        tests:
          - not_null
```

## Build CRM Mart Models

### `fct_contacts`

Create `models/marts/crm/fct_contacts.sql`:

```sql
/*
HubSpot contacts fact table.
One row per contact with enriched attributes and segmentation.
*/

with contacts as (

    select * from {{ ref('int_contacts__enriched') }}

),

final as (

    select
        -- identifiers
        contact_id,
        email,

        -- name
        first_name,
        last_name,
        full_name,

        -- lifecycle and segmentation
        lifecycle_stage,
        contact_age_bucket,

        -- metrics
        days_since_created,
        days_since_modified,

        -- flags
        case
            when lifecycle_stage in ('customer', 'evangelist')
            then true else false
        end as is_customer,

        case
            when lower(email) like '%@example.com'
                or lower(email) like '%@test.com'
            then true else false
        end as is_internal,

        -- timestamps
        created_at,
        last_modified_at,
        airbyte_extracted_at

    from contacts
    where email is not null  -- exclude contacts with no email

)

select * from final
```

Create `models/marts/crm/fct_contacts.yml`:

```yaml
version: 2

models:
  - name: fct_contacts
    description: >
      HubSpot contacts with enriched attributes and segmentation.
      One row per contact. Excludes contacts with no email address.
    columns:
      - name: contact_id
        description: HubSpot contact ID.
        tests:
          - unique
          - not_null
      - name: email
        description: Contact email address.
        tests:
          - unique
          - not_null
      - name: full_name
        description: Full name combining first and last name.
      - name: lifecycle_stage
        description: HubSpot lifecycle stage.
      - name: contact_age_bucket
        description: Segmentation bucket based on contact age (new, recent, established).
        tests:
          - accepted_values:
              values: ['new', 'recent', 'established']
      - name: is_customer
        description: True if the contact is a current customer.
        tests:
          - not_null
          - accepted_values:
              values: [true, false]
      - name: is_internal
        description: True if the email matches an internal domain (test/example).
        tests:
          - not_null
          - accepted_values:
              values: [true, false]
```

## Build Reporting Models

Reporting models are published to `ANALYTICS.REPORTING` — the Terraform-managed schema with BI tool access. They are presentation-layer models: typically views or tables that select from mart models and add any final formatting needed for BI tools.

### `rpt_contacts`

Create `models/marts/reporting/rpt_contacts.sql`:

```sql
/*
Contact reporting view for BI tools.
Published to the REPORTING schema for ANALYTICS_REPORTER access.
Selects from fct_contacts and presents the most relevant columns.
*/

select
    contact_id,
    full_name,
    email,
    lifecycle_stage,
    contact_age_bucket,
    is_customer,
    days_since_created,
    created_at,
    last_modified_at

from {{ ref('fct_contacts') }}
where not is_internal  -- exclude internal test contacts from BI dashboards
```

Create `models/marts/reporting/rpt_contacts.yml`:

```yaml
version: 2

models:
  - name: rpt_contacts
    description: >
      Contact reporting view for BI tools. Published to ANALYTICS.REPORTING.
      Excludes internal test contacts. Simplified columns for dashboard use.
    columns:
      - name: contact_id
        description: HubSpot contact ID.
      - name: full_name
        description: Contact full name.
      - name: email
        description: Contact email address.
      - name: lifecycle_stage
        description: HubSpot lifecycle stage.
      - name: contact_age_bucket
        description: Segmentation bucket (new, recent, established).
      - name: is_customer
        description: True if the contact is a customer.
```

!!! note "Why a separate reporting model?"
    `fct_contacts` in MARTS is the source of truth for analytics engineers — it includes all contacts, internal flags, and metadata columns. `rpt_contacts` in REPORTING is what BI users see: simplified, pre-filtered, and without technical columns. This separation means you can update the mart without worrying about breaking BI dashboards.

## Run the Mart Models

```sh
# Build everything from sources through to marts
dbt build

# Build only the mart layer and its upstream dependencies
dbt build --select +marts

# Build a specific mart
dbt build --select +fct_exchange_rates

# Build everything that will land in REPORTING
dbt build --select +tag:reporting

# Full refresh an incremental model (rebuild from scratch)
dbt build --select fct_exchange_rates --full-refresh
```

Verify in Snowflake:

```sql
-- Check marts schema
USE DATABASE ANALYTICS_DEV;
USE SCHEMA DBT_<YOURNAME>_MARTS;
SHOW TABLES;
-- Should show: FCT_EXCHANGE_RATES, DIM_CURRENCIES, DIM_PRODUCTS, FCT_CONTACTS

-- Check incremental model
SELECT COUNT(*), MIN(rate_date), MAX(rate_date)
FROM FCT_EXCHANGE_RATES;

-- Check reporting schema
USE SCHEMA DBT_<YOURNAME>_REPORTING;
SHOW VIEWS;
-- Should show: RPT_CONTACTS

-- Test BI tool access (as ANALYTICS_REPORTER in production)
USE ROLE ANALYTICS_REPORTER;
USE DATABASE ANALYTICS;
SELECT COUNT(*) FROM ANALYTICS.REPORTING.RPT_CONTACTS;  -- should succeed
SELECT COUNT(*) FROM ANALYTICS.MARTS.FCT_CONTACTS;       -- should fail (no access)
```

## Understanding Incremental Models

Incremental models are critical for scalability. Without incremental logic, `fct_exchange_rates` would rebuild millions of historical rows every day. With incremental logic:

1. **First run**: dbt builds the full table (`dbt run --select fct_exchange_rates`)
2. **Subsequent runs**: dbt only processes rows with `rate_date > max(rate_date)` from the existing table
3. **Full refresh**: Forces a complete rebuild when needed (`dbt run --select fct_exchange_rates --full-refresh`)

The `unique_key` configuration tells dbt how to identify rows for updates. If a row with the same `exchange_rate_id` already exists, dbt will update it (useful for late-arriving corrections). If not, dbt inserts it.

## Summary

You've built the mart and reporting layers:

- [x] `fct_exchange_rates` — daily exchange rates with incremental materialisation
- [x] `dim_currencies` — currency reference dimension with major currency flag
- [x] `dim_products` — current state product dimension from Type 2 SCD data
- [x] `fct_contacts` — enriched HubSpot contacts with segmentation flags
- [x] `rpt_contacts` — curated view published to REPORTING schema for BI access
- [x] Verified role-based access (ANALYTICS_REPORTER can see REPORTING, not MARTS)
- [x] Each model has its own YAML file for documentation and tests

## What's Next

Add comprehensive tests and documentation, configure `dbt_project_evaluator` to enforce project standards in CI, and learn about doc blocks for reusable documentation.

Continue to [Testing and Documentation](8-testing-and-documentation.md) →
