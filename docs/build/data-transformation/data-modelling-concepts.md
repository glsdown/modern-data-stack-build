# Data Modelling Concepts

This page introduces dimensional modelling — the approach used to structure the
mart models in your dbt project. Understanding these concepts helps you design
tables that are performant for analytics queries and intuitive for BI tool users.

On this page, you will:

- [x] Understand star schema and dimensional modelling
- [x] Distinguish fact tables from dimension tables
- [x] Learn about slowly changing dimensions (SCDs)
- [x] See how these concepts map to dbt naming conventions

## Overview

Dimensional modelling organises data into two types of tables — **facts** and
**dimensions** — arranged in a pattern called a **star schema**. This approach
was popularised by Ralph Kimball and remains the standard for analytics
warehouses.

```
                    ┌──────────────────┐
                    │  dim_currencies  │
                    │                  │
                    │  currency_code   │
                    │  currency_name   │
                    │  symbol          │
                    └────────┬─────────┘
                             │
┌──────────────┐    ┌────────┴──────────┐    ┌──────────────┐
│ dim_products  │    │ fct_exchange_rates│    │  dim_dates   │
│               │────│                  │────│              │
│ product_id    │    │ base_currency    │    │ date_key     │
│ product_name  │    │ target_currency  │    │ day_of_week  │
│ category      │    │ rate             │    │ month_name   │
│ price         │    │ rate_date        │    │ quarter      │
└──────────────┘    │ loaded_at        │    │ is_weekend   │
                    └───────────────────┘    └──────────────┘
```

The fact table sits at the centre. Dimension tables surround it like points of a
star — hence the name.

## Fact Tables

Fact tables record **events**, **transactions**, or **measurements** — things
that happen over time. Each row represents a single occurrence at a specific
grain (level of detail).

### Characteristics

| Property | Description |
|----------|-------------|
| **Grain** | One row per event/measurement (e.g. one row per currency pair per day) |
| **Numeric measures** | Contain values you aggregate — amounts, counts, rates |
| **Foreign keys** | Reference dimension tables for context |
| **Append-mostly** | New rows are added; existing rows rarely change |
| **Large** | Often the biggest tables in the warehouse |

### Types of Fact Tables

| Type | Description | Example |
|------|-------------|---------|
| **Transaction** | One row per event | `fct_orders` — one row per order |
| **Periodic snapshot** | One row per entity per period | `fct_account_balances` — daily balance per account |
| **Accumulating snapshot** | One row per process, updated as stages complete | `fct_order_fulfilment` — tracks order through stages |
| **Factless** | Records events with no numeric measure | `fct_page_views` — tracks that a view occurred |

### In This Project

The dbt project uses the `fct_` prefix for fact tables:

- `fct_exchange_rates` — daily exchange rates (periodic snapshot)
- `fct_contacts` — HubSpot contact events (transaction)

## Dimension Tables

Dimension tables describe the **who, what, where, when** context around facts.
They contain descriptive attributes that analysts use to filter, group, and label
query results.

### Characteristics

| Property | Description |
|----------|-------------|
| **Descriptive** | Text labels, categories, hierarchies |
| **Relatively small** | Fewer rows than fact tables |
| **Slowly changing** | Attributes update occasionally (see SCDs below) |
| **Wide** | Many columns of descriptive attributes |
| **Surrogate keys** | Often use generated keys rather than natural keys |

### Common Dimension Types

| Dimension | Describes | Typical Columns |
|-----------|-----------|-----------------|
| **Date** | When events happened | Day of week, month, quarter, fiscal year, is_weekend |
| **Customer** | Who was involved | Name, segment, region, signup date |
| **Product** | What was transacted | Name, category, price, supplier |
| **Geography** | Where it happened | Country, region, city, postcode |

### In This Project

The dbt project uses the `dim_` prefix for dimension tables:

- `dim_currencies` — currency codes and names
- `dim_products` — current product catalogue (derived from Type 2 SCD source)

## Star Schema vs Snowflake Schema

Two common arrangements of facts and dimensions:

### Star Schema

Dimensions connect directly to the fact table. Denormalised — dimension tables
may contain redundant data.

```
dim_customer ──── fct_orders ──── dim_product
                      │
                  dim_date
```

**Advantages**: Simpler queries (fewer joins), better query performance,
easier for BI tools to navigate.

**Disadvantage**: Some data redundancy in dimensions.

### Snowflake Schema

Dimensions are normalised into sub-dimensions. For example, `dim_product` links
to `dim_category`, which links to `dim_department`.

```
dim_category ── dim_product ── fct_orders ── dim_customer ── dim_region
                                   │
                               dim_date
```

**Advantages**: Less data redundancy, cleaner for data stewards.

**Disadvantage**: More joins, slower queries, harder for BI tools.

!!! tip "Recommendation: Star Schema"
    For analytics warehouses (especially with Snowflake's columnar storage),
    star schemas are preferred. The storage cost of denormalised dimensions is
    negligible, and the query performance and simplicity benefits are
    significant. This project follows the star schema approach.

## Grain

The **grain** defines what one row in a fact table represents. Establishing grain
is the most important decision in dimensional modelling — get it wrong and your
metrics will be incorrect.

### Examples

| Fact Table | Grain | One Row Equals |
|------------|-------|----------------|
| `fct_exchange_rates` | One currency pair per day | GBP→USD on 2024-01-15 |
| `fct_orders` | One order | Order #12345 |
| `fct_order_lines` | One line item in an order | Order #12345, Product A |
| `fct_daily_active_users` | One user per day | User 789 on 2024-01-15 |

### Why Grain Matters

If your grain is "one order" but you accidentally join to a table that creates
multiple rows per order (e.g. order lines), your aggregates will be inflated.
Always document the grain in your dbt model YAML:

```yaml
models:
  - name: fct_exchange_rates
    description: >
      Daily exchange rates from base currency to target currencies.
      Grain: one row per base_currency + target_currency + rate_date.
    columns:
      - name: rate_date
        description: "The date of the exchange rate."
        tests:
          - not_null
```

## Slowly Changing Dimensions

Dimension attributes change over time. A product's price increases. A customer
moves to a new city. How you handle these changes determines your SCD type.

### Type 1: Overwrite

Replace the old value with the new one. No history is preserved.

| customer_id | name | city |
|-------------|------|------|
| 1 | Jane | Manchester |

After Jane moves to London:

| customer_id | name | city |
|-------------|------|------|
| 1 | Jane | London |

**Use when**: You only care about the current state (e.g. a customer's current
email address).

### Type 2: Add a New Row

Create a new row for each change, with validity dates. Full history is preserved.

| surrogate_key | customer_id | name | city | valid_from | valid_to | is_current |
|---------------|-------------|------|------|------------|----------|------------|
| 1001 | 1 | Jane | Manchester | 2023-01-01 | 2024-06-15 | false |
| 1002 | 1 | Jane | London | 2024-06-15 | 9999-12-31 | true |

**Use when**: Historical accuracy matters (e.g. "what region was this customer in
when they placed the order?").

### Type 3: Add a Column

Keep the previous value in a separate column. Only one level of history.

| customer_id | name | city | previous_city |
|-------------|------|------|---------------|
| 1 | Jane | London | Manchester |

**Use when**: You only need the immediately previous value.

### In This Project

The products pipeline uses **Type 2 SCD** at the source level — the raw data
contains `valid_from` and `valid_to` columns. The intermediate model
`int_products__current` extracts the current state (where `is_current = true`)
for the `dim_products` dimension:

```sql
-- models/intermediate/int_products__current.sql
select
    product_id,
    product_name,
    category,
    price,
    valid_from as current_since
from {{ ref('stg_dlt__products') }}
where is_current = true
```

See [Intermediate Models](6-intermediate-models.md) for the full implementation.

## Mapping to dbt Layers

The dbt project layers map naturally to dimensional modelling concepts:

| dbt Layer | Dimensional Concept | Naming Convention |
|-----------|-------------------|-------------------|
| **Staging** (`stg_`) | Source-conformed data (cleaned, typed) | `stg_{source}__{table}` |
| **Intermediate** (`int_`) | Business logic, SCD extraction, enrichment | `int_{entity}__{action}` |
| **Marts — Facts** (`fct_`) | Fact tables (events, measurements) | `fct_{event_or_process}` |
| **Marts — Dimensions** (`dim_`) | Dimension tables (entities, descriptors) | `dim_{entity}` |
| **Reporting** (`rpt_`) | Presentation views for BI tools | `rpt_{purpose}` |

```
Source Data          Staging              Intermediate          Marts
──────────          ───────              ────────────          ─────

exchange_rates  ──▶ stg_snowpipe__       (none needed)   ──▶  fct_exchange_rates
                    exchange_rates

products        ──▶ stg_dlt__products ──▶ int_products__  ──▶  dim_products
                                          current

contacts        ──▶ stg_airbyte__    ──▶ int_contacts__  ──▶  fct_contacts
                    contacts              enriched
```

## Conformed Dimensions

When multiple fact tables share the same dimension (e.g. `dim_currencies` is
used by both `fct_exchange_rates` and `fct_orders`), that dimension is
**conformed** — it has a single, consistent definition across all facts.

Conformed dimensions enable cross-process analysis. You can join
`fct_exchange_rates` and `fct_orders` on the same `dim_currencies` table and get
consistent results.

!!! info "Practical Rule"
    If two fact tables both reference "currency", they should both join to the
    same `dim_currencies` model. Never create separate currency dimensions per
    fact table — that leads to inconsistent reporting.

## Best Practices

### Design

- **Define grain first** — before writing any SQL, write down what one row
  represents
- **Prefer star over snowflake** — denormalise dimensions for query performance
- **Use surrogate keys** — generate integer or hash keys rather than relying on
  source natural keys (which may change or collide across sources)
- **Build conformed dimensions** — share dimension tables across fact tables

### dbt Implementation

- **Materialise facts as incremental** — large fact tables should not be
  rebuilt from scratch on every run
- **Materialise dimensions as tables** — dimensions are small enough for full
  rebuilds and benefit from simple logic
- **Test grain with unique tests** — add a `dbt_utils.unique_combination_of_columns`
  test on the grain columns of every fact table
- **Document grain in YAML** — make the grain explicit in the model description

### Naming

- **Facts**: `fct_{event}` (e.g. `fct_exchange_rates`, `fct_orders`)
- **Dimensions**: `dim_{entity}` (e.g. `dim_currencies`, `dim_products`)
- **Date dimensions**: `dim_dates` (generated, not from source data)
- **Bridge tables**: `bridge_{relationship}` (for many-to-many relationships)

## Further Reading

- [The Data Warehouse Toolkit](https://www.kimballgroup.com/data-warehouse-business-intelligence-resources/books/) by Ralph Kimball — the definitive reference on dimensional modelling
- [dbt Best Practices: How we structure our dbt projects](https://docs.getdbt.com/best-practices/how-we-structure/1-guide-overview) — dbt Labs' guide to project structure
- [Mart Models](7-mart-models.md) — the practical implementation of facts and dimensions in this project
- [Intermediate Models](6-intermediate-models.md) — how SCD Type 2 extraction works in the intermediate layer

## Summary

!!! success "What You've Learned"
    - [x] Star schema organises data into **fact tables** (events, measurements) and **dimension tables** (entities, descriptors)
    - [x] **Grain** defines what one row represents — always define it before writing SQL
    - [x] **Slowly changing dimensions** (Types 1, 2, 3) handle attribute changes over time
    - [x] dbt layers map to dimensional concepts: staging → source-conformed, intermediate → business logic, marts → facts and dimensions

## What's Next

With these concepts in mind, continue building your dbt models.

Continue to [Sources and Staging](5-sources-and-staging.md) →
