# Sources and Staging Models

On this page, you will:

- [x] Define sources for all three raw databases (DLT, SNOWPIPE, AIRBYTE)
- [x] Configure freshness checks so stale data is detected
- [x] Build staging models that clean and rename raw columns

## Overview

The staging layer is the first transformation layer. It sits directly on top of raw source tables and does only light work:

- Renaming columns to consistent conventions (snake_case, no prefixes)
- Casting data types where the loader produces incorrect types
- Deduplicating records where the loader may produce duplicates
- Removing loader-internal metadata columns

Staging models are **views** by default — they don't materialise data, they just provide a clean interface to the raw tables. This keeps the staging layer cheap to maintain.

```
Raw databases          Staging models (views)      Downstream
─────────────          ──────────────────────      ──────────

DLT.APPLICATION_DATA
  products        ────▶  stg_dlt__products    ────────┐
                                                      │
DLT.OPEN_EXCHANGE_RATES                               ├──▶ intermediate
  currencies      ────▶  stg_dlt__currencies          │
                                                      │
SNOWPIPE.OPEN_EXCHANGE_RATES                          │
  exchange_rates  ────▶  stg_snowpipe__exchange_rates ┘

AIRBYTE.HUBSPOT
  contacts        ────▶  stg_airbyte__contacts    ──────▶ intermediate
```

## Define Sources

Sources are declared in `_sources.yml` files alongside the staging models. Each source group has its own file, co-located with the models that reference it.

### DLT Sources

Create `models/staging/dlt/_sources.yml`:

```yaml
version: 2

sources:
  - name: dlt_application_data
    description: >
      Products data loaded by dlt from PostgreSQL (Clever Cloud).
      Incremental merge-based load runs daily.
    database: DLT
    schema: APPLICATION_DATA
    loader: dlt
    loaded_at_field: _dlt_load_time
    freshness:
      warn_after: {count: 25, period: hour}
      error_after: {count: 49, period: hour}
    tables:
      - name: products
        description: >
          Type 2 SCD products data from PostgreSQL. Each row represents a version
          of a product with operation type (INSERT, UPDATE, DELETE) and timestamp.
        columns:
          - name: audit_id
            description: Unique row identifier (primary key).
          - name: product_id
            description: Product business key (multiple rows per product).
          - name: product_name
            description: Product name.
          - name: price_usd
            description: Product price in USD.
          - name: operation
            description: Operation type (INSERT, UPDATE, DELETE).
          - name: valid_ts
            description: Timestamp when the change occurred.
          - name: _dlt_load_id
            description: dlt load batch identifier.
          - name: _dlt_load_time
            description: Timestamp when this row was loaded by dlt.

  - name: dlt_open_exchange_rates
    description: >
      Currency reference data loaded by dlt from Open Exchange Rates API.
      Updated weekly.
    database: DLT
    schema: OPEN_EXCHANGE_RATES
    loader: dlt
    loaded_at_field: _dlt_load_time
    freshness:
      warn_after: {count: 8, period: day}
      error_after: {count: 15, period: day}
    tables:
      - name: currencies
        description: >
          Reference list of all available currency codes and names from Open Exchange Rates.
          Updated weekly. One row per currency.
        columns:
          - name: _dlt_id
            description: Internal dlt row identifier.
          - name: _dlt_load_time
            description: Timestamp when this row was loaded by dlt.
          - name: currency_code
            description: ISO 4217 currency code (e.g. GBP, EUR, USD).
          - name: currency_name
            description: Full name of the currency (e.g. British Pound Sterling).
```

### Snowpipe Sources

Create `models/staging/snowpipe/_sources.yml`:

```yaml
version: 2

sources:
  - name: snowpipe_open_exchange_rates
    description: >
      Exchange rate data loaded via Snowpipe from S3.
      Files are deposited to S3 by dlt and auto-ingested by Snowpipe within minutes.
    database: SNOWPIPE
    schema: OPEN_EXCHANGE_RATES
    loader: snowpipe
    loaded_at_field: _loaded_at
    freshness:
      warn_after: {count: 25, period: hour}
      error_after: {count: 49, period: hour}
    tables:
      - name: exchange_rates
        description: >
          Daily USD base exchange rates for all currencies from Open Exchange Rates API.
          Loaded via Snowpipe from S3. One row per currency per day.
        columns:
          - name: rate_date
            description: The date for which these exchange rates apply.
          - name: base_currency
            description: The base currency (always USD for this source).
          - name: target_currency
            description: The target currency code (e.g. GBP, EUR).
          - name: rate
            description: Exchange rate from base to target currency.
          - name: _loaded_at
            description: Timestamp when Snowpipe loaded this row.
```

### Airbyte Sources

Create `models/staging/airbyte/_sources.yml`:

```yaml
version: 2

sources:
  - name: airbyte_hubspot
    description: >
      HubSpot CRM data loaded by Airbyte. Incremental append+dedup sync.
      Updated daily.
    database: AIRBYTE
    schema: HUBSPOT
    loader: airbyte
    loaded_at_field: _airbyte_extracted_at
    freshness:
      warn_after: {count: 25, period: hour}
      error_after: {count: 49, period: hour}
    tables:
      - name: contacts
        description: >
          HubSpot contacts. One row per contact per sync (append+dedup).
          The most recent record per contact_id is the current version.
        columns:
          - name: _airbyte_raw_id
            description: Unique row identifier assigned by Airbyte.
          - name: _airbyte_extracted_at
            description: Timestamp when Airbyte extracted this record from HubSpot.
          - name: _airbyte_meta
            description: Airbyte sync metadata (changes, sync ID).
          - name: id
            description: HubSpot contact ID. Unique per contact.
          - name: email
            description: Primary email address for the contact.
          - name: firstname
            description: Contact first name.
          - name: lastname
            description: Contact last name.
          - name: createdat
            description: Timestamp when the contact was created in HubSpot.
          - name: lastmodifieddate
            description: Timestamp when the contact was last modified in HubSpot.
          - name: hs_lifecyclestage
            description: HubSpot lifecycle stage (subscriber, lead, customer, etc.)
```

## Build Staging Models

### `stg_dlt__products`

Create `models/staging/dlt/stg_dlt__products.sql`:

```sql
with source as (

    select * from {{ source('dlt_application_data', 'products') }}

),

renamed as (

    select
        -- identifiers
        audit_id,
        product_id,

        -- attributes
        product_name,
        cast(price_usd as float) as price_usd,

        -- metadata
        upper(operation)               as operation,
        cast(valid_ts as timestamp_ntz) as valid_ts,
        _dlt_load_id                   as dlt_load_id,
        cast(_dlt_load_time as timestamp_ntz) as loaded_at

    from source

)

select * from renamed
```

Create `models/staging/dlt/stg_dlt__products.yml`:

```yaml
version: 2

models:
  - name: stg_dlt__products
    description: >
      Type 2 SCD products data from PostgreSQL via dlt.
      Each row represents a version of a product with operation type and timestamp.
    columns:
      - name: audit_id
        description: Unique row identifier (primary key for this table).
        tests:
          - unique
          - not_null
      - name: product_id
        description: Product business key. Multiple rows per product represent history.
        tests:
          - not_null
      - name: product_name
        description: Product name. Null for DELETE operations.
      - name: price_usd
        description: Product price in USD. Null for DELETE operations.
        tests:
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0
              strictly: true
              row_condition: "operation != 'DELETE'"
      - name: operation
        description: Operation type indicating how this row was created.
        tests:
          - not_null
          - accepted_values:
              values: ['INSERT', 'UPDATE', 'DELETE']
      - name: valid_ts
        description: Timestamp when this product version became valid.
        tests:
          - not_null
```

### `stg_dlt__currencies`

Create `models/staging/dlt/stg_dlt__currencies.sql`:

```sql
with source as (

    select * from {{ source('dlt_open_exchange_rates', 'currencies') }}

),

renamed as (

    select
        -- identifiers
        _dlt_id                     as dlt_row_id,

        -- dimensions
        upper(currency_code)        as currency_code,
        currency_name,

        -- metadata
        cast(_dlt_load_time as timestamp_ntz) as loaded_at

    from source

)

select * from renamed
```

Create `models/staging/dlt/stg_dlt__currencies.yml`:

```yaml
version: 2

models:
  - name: stg_dlt__currencies
    description: >
      Currency reference data from Open Exchange Rates API via dlt.
      One row per currency. Updated weekly.
    columns:
      - name: currency_code
        description: ISO 4217 currency code (e.g. GBP, EUR, USD).
        tests:
          - unique
          - not_null
      - name: currency_name
        description: Full name of the currency.
        tests:
          - not_null
```

### `stg_snowpipe__exchange_rates`

Create `models/staging/snowpipe/stg_snowpipe__exchange_rates.sql`:

```sql
with source as (

    select * from {{ source('snowpipe_open_exchange_rates', 'exchange_rates') }}

),

renamed as (

    select
        -- timestamps
        cast(_loaded_at as timestamp_ntz) as loaded_at,
        cast(rate_date as date)           as rate_date,

        -- dimensions
        upper(base_currency)        as base_currency,
        upper(target_currency)      as target_currency,

        -- measures
        cast(rate as float)         as exchange_rate

    from source

)

select * from renamed
```

Create `models/staging/snowpipe/stg_snowpipe__exchange_rates.yml`:

```yaml
version: 2

models:
  - name: stg_snowpipe__exchange_rates
    description: >
      Exchange rates from Open Exchange Rates API loaded via Snowpipe from S3.
      One row per currency per day. Base currency is always USD.
    columns:
      - name: rate_date
        description: The date for which the exchange rate applies.
        tests:
          - not_null
      - name: base_currency
        description: Base currency code. Always USD for this source.
        tests:
          - not_null
          - accepted_values:
              values: ['USD']
      - name: target_currency
        description: Target currency code (e.g. GBP, EUR).
        tests:
          - not_null
      - name: exchange_rate
        description: Exchange rate from USD to target currency.
        tests:
          - not_null
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0
              strictly: true
```

### `stg_airbyte__contacts`

Airbyte uses append+dedup sync mode — multiple versions of each contact may exist in the raw table. Select only the most recent version of each contact using the HubSpot `id`:

Create `models/staging/airbyte/stg_airbyte__contacts.sql`:

```sql
with source as (

    select * from {{ source('airbyte_hubspot', 'contacts') }}

),

-- Airbyte append+dedup: take the most recently extracted version of each contact
deduped as (

    select *
    from source
    qualify row_number() over (
        partition by id
        order by _airbyte_extracted_at desc
    ) = 1

),

renamed as (

    select
        -- identifiers
        id                                          as contact_id,

        -- personal details
        lower(trim(email))                          as email,
        initcap(trim(firstname))                    as first_name,
        initcap(trim(lastname))                     as last_name,

        -- lifecycle
        hs_lifecyclestage                           as lifecycle_stage,

        -- timestamps
        cast(createdat as timestamp_ntz)            as created_at,
        cast(lastmodifieddate as timestamp_ntz)     as last_modified_at,
        cast(_airbyte_extracted_at as timestamp_ntz) as airbyte_extracted_at

    from deduped

)

select * from renamed
```

Create `models/staging/airbyte/stg_airbyte__contacts.yml`:

```yaml
version: 2

models:
  - name: stg_airbyte__contacts
    description: >
      HubSpot contacts loaded via Airbyte with deduplication.
      One row per contact (most recent version).
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
      - name: first_name
        description: Contact first name (title case).
      - name: last_name
        description: Contact last name (title case).
      - name: lifecycle_stage
        description: HubSpot lifecycle stage.
      - name: created_at
        description: When the contact was created in HubSpot.
        tests:
          - not_null
      - name: last_modified_at
        description: When the contact was last modified in HubSpot.
```

## Run the Staging Models

```sh
# Run all staging models
dbt run --select staging

# Run a specific model
dbt run --select stg_airbyte__contacts

# Run and test together
dbt build --select staging
```

Check that the views are created in Snowflake:

```sql
USE DATABASE ANALYTICS_DEV;
SHOW SCHEMAS;
-- Should include: DBT_<YOURNAME>_STAGING

USE SCHEMA DBT_<YOURNAME>_STAGING;
SHOW VIEWS;
-- Should include: STG_DLT__PRODUCTS, STG_DLT__CURRENCIES,
--                 STG_SNOWPIPE__EXCHANGE_RATES, STG_AIRBYTE__CONTACTS
```

## Check Source Freshness

```sh
dbt source freshness
```

This queries the `loaded_at_field` on each source table and compares it against the freshness thresholds defined in `_sources.yml`. Run this as a separate step in CI/CD to alert on stale data before running transformations.

```
Found 5 sources.

Source dlt_application_data.products           ✓ Pass  (loaded 3 hours ago)
Source dlt_open_exchange_rates.currencies      ✓ Pass  (loaded 2 days ago)
Source snowpipe_open_exchange_rates.exchange_rates  ✓ Pass  (loaded 15 minutes ago)
Source airbyte_hubspot.contacts                ✓ Pass  (loaded 1 hour ago)
```

## Summary

You've defined sources and built the staging layer:

- [x] Defined sources for DLT (products, currencies), Snowpipe (exchange rates), and Airbyte (contacts) databases
- [x] Configured freshness thresholds for each source
- [x] Built staging views with cleaned and renamed columns
- [x] Handled Airbyte deduplication in `stg_airbyte__contacts`
- [x] Handled dlt Type 2 SCD in `stg_dlt__products`
- [x] Added YAML documentation and tests

## What's Next

Combine and reshape staging models into the intermediate layer.

Continue to [Intermediate Models](6-intermediate-models.md) →
