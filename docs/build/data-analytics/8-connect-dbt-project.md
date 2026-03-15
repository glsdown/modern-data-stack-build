# Connect dbt Project

On this page, you will:

- [x] Understand how Lightdash discovers dbt models and metrics
- [x] Configure your dbt project for optimal Lightdash integration
- [x] Learn the `meta` tag structure for defining metrics and dimensions
- [x] Set up Lightdash-specific YAML configuration
- [x] Troubleshoot common discovery issues

## Overview

Lightdash is **dbt-native** — it reads your dbt project directly from GitHub, parses YAML files, and discovers models, metrics, and dimensions. There's no separate UI configuration for metrics. Everything is defined in code, version-controlled, and testable.

This page covers how to structure your dbt project so Lightdash can discover and understand your models.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  LIGHTDASH + DBT INTEGRATION                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  dbt-transform Repository                                               │
│  ────────────────────────                                               │
│                                                                         │
│  models/                                                                │
│  ├── marts/                                                             │
│  │   └── core/                                                          │
│  │       ├── fct_exchange_rates.sql    ← SQL model                     │
│  │       └── fct_exchange_rates.yml    ← Metrics, dimensions, meta     │
│  │                                                                      │
│  │ Lightdash reads YAML files:                                          │
│  │ • `meta:` tags → metrics, dimensions, hidden fields                  │
│  │ • `description:` → field documentation                               │
│  │ • `columns:` → discover dimensions                                   │
│  │                                                                      │
│  └── Generates:                                                          │
│      • Lightdash Explore view                                           │
│      • Drag-and-drop interface                                          │
│      • Pre-defined metrics                                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## How Lightdash Discovers Models

When you connect Lightdash to your dbt repository and compile the project, Lightdash:

1. **Clones the dbt repository** (GitHub, GitLab, etc.)
2. **Runs `dbt deps`** to install packages from `packages.yml`
3. **Runs `dbt compile`** to generate `manifest.json` (the dbt DAG)
4. **Parses `manifest.json`** to discover:
   - All models in `models/` directory
   - Column definitions from YAML files
   - `meta` tags for metrics, dimensions, and configuration
   - Relationships and joins between models

5. **Creates Lightdash Explores** — one Explore per dbt model

Each Explore provides a drag-and-drop interface with:
- **Dimensions** (columns from the model)
- **Metrics** (aggregations defined in YAML)
- **Filters** (WHERE clauses)
- **Joins** (relationships to other models)

## dbt Project Structure for Lightdash

Lightdash works best with a well-structured dbt project:

```
dbt-transform/
├── models/
│   ├── marts/
│   │   ├── core/
│   │   │   ├── fct_exchange_rates.sql
│   │   │   ├── fct_exchange_rates.yml       ← Metrics defined here
│   │   │   ├── dim_products.sql
│   │   │   └── dim_products.yml
│   │   └── crm/
│   │       ├── fct_contacts.sql
│   │       └── fct_contacts.yml
│   └── staging/
│       └── ...  (not exposed to Lightdash — use REPORTING schema)
├── dbt_project.yml
└── packages.yml
```

**Key principles:**
- **One YAML file per model** (not combined)
- **Use `meta` tags** to define metrics and configure Lightdash behavior
- **Only expose BI-ready models** to Lightdash (via `REPORTING` schema or Lightdash config)

## Configuring dbt_project.yml for Lightdash

Add Lightdash-specific configuration to `dbt_project.yml`:

```yaml
# dbt_project.yml
name: dbt_transform
version: 1.0.0
config-version: 2

profile: dbt_transform

model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]

target-path: "target"
clean-targets:
  - "target"
  - "dbt_packages"

models:
  dbt_transform:
    # Hide staging and intermediate models from Lightdash
    staging:
      +meta:
        lightdash:
          enabled: false  # Don't create Explores for staging models

    intermediate:
      +meta:
        lightdash:
          enabled: false  # Don't create Explores for intermediate models

    # Expose only mart models to Lightdash
    marts:
      +meta:
        lightdash:
          enabled: true
      +materialized: table
      +schema: marts
```

This configuration:
- **Hides staging and intermediate models** from Lightdash (they're not BI-ready)
- **Exposes only mart models** (fact and dimension tables)

!!! tip "Alternative: Use REPORTING Schema"
    Instead of `enabled: false`, you can control exposure by only querying the `REPORTING` schema in Lightdash. Models not published to `REPORTING` won't be discoverable.

## Defining Metrics in dbt YAML

Metrics are defined in the `meta` section of your model YAML files. Lightdash uses this to generate aggregation options in the UI.

### Example: fct_exchange_rates.yml

```yaml
# models/marts/core/fct_exchange_rates.yml
version: 2

models:
  - name: fct_exchange_rates
    description: Daily exchange rates for GBP to various currencies
    meta:
      lightdash:
        label: "Exchange Rates"  # Display name in Lightdash
        group: "Finance"  # Group in sidebar

    columns:
      - name: rate_date
        description: Date of the exchange rate
        meta:
          dimension:
            type: date  # Lightdash will treat this as a date dimension

      - name: base_currency
        description: Base currency (always GBP)
        meta:
          dimension:
            type: string

      - name: target_currency
        description: Target currency (USD, EUR, etc.)
        meta:
          dimension:
            type: string

      - name: exchange_rate
        description: Exchange rate from base to target currency
        meta:
          dimension:
            type: number
            round: 4  # Round to 4 decimal places in Lightdash
          metrics:
            # Define metrics that aggregate this column
            average_exchange_rate:
              type: average
              label: "Average Exchange Rate"
              description: "Average exchange rate across selected date range"
            min_exchange_rate:
              type: min
              label: "Minimum Exchange Rate"
            max_exchange_rate:
              type: max
              label: "Maximum Exchange Rate"

      - name: exchange_rate_id
        description: Surrogate key for this fact
        meta:
          dimension:
            hidden: true  # Hide from Lightdash UI (not useful for analysis)
```

### Metrics Types

Lightdash supports these metric types:

| Type | Description | Example |
|------|-------------|---------|
| `count` | Count of rows | `COUNT(*)` |
| `count_distinct` | Distinct count | `COUNT(DISTINCT target_currency)` |
| `sum` | Sum of values | `SUM(exchange_rate)` |
| `average` | Average of values | `AVG(exchange_rate)` |
| `min` | Minimum value | `MIN(exchange_rate)` |
| `max` | Maximum value | `MAX(exchange_rate)` |
| `median` | Median value (if database supports) | `MEDIAN(exchange_rate)` |
| `percentile` | Percentile (requires `percentile` param) | `PERCENTILE_CONT(0.95)` |

### Dimension Types

Lightdash automatically infers dimension types from dbt column types, but you can override:

| Type | Used For | Features |
|------|----------|----------|
| `string` | Text fields | Filters: equals, contains, starts with |
| `number` | Numeric values | Filters: equals, greater than, less than, between |
| `date` | Date fields | Date pickers, date ranges, date grouping (day/week/month/year) |
| `timestamp` | Datetime fields | DateTime pickers, time-based grouping |
| `boolean` | True/false fields | Checkbox filters |

## Example: dim_products.yml

```yaml
# models/marts/core/dim_products.yml
version: 2

models:
  - name: dim_products
    description: Product dimension table
    meta:
      lightdash:
        label: "Products"
        group: "Catalogue"

    columns:
      - name: product_key
        description: Surrogate key for the product
        meta:
          dimension:
            hidden: true

      - name: product_id
        description: Natural key (product ID from source system)
        meta:
          dimension:
            type: string
            label: "Product ID"

      - name: product_name
        description: Product name
        meta:
          dimension:
            type: string

      - name: product_category
        description: Product category
        meta:
          dimension:
            type: string

      - name: product_price
        description: Current product price in GBP
        meta:
          dimension:
            type: number
            round: 2
            format: currency
            currency: GBP
          metrics:
            average_price:
              type: average
              label: "Average Product Price"
              format: currency
              currency: GBP
            max_price:
              type: max
              label: "Most Expensive Product"

      - name: is_active
        description: Whether the product is currently active
        meta:
          dimension:
            type: boolean
            label: "Active Product"

      - name: created_at
        description: When the product was first added
        meta:
          dimension:
            type: timestamp
```

## Advanced: Custom SQL Metrics

You can define custom SQL metrics for more complex aggregations:

```yaml
# models/marts/core/fct_exchange_rates.yml
version: 2

models:
  - name: fct_exchange_rates
    # ... columns ...

    meta:
      lightdash:
        metrics:
          # Custom SQL metric
          gbp_usd_volatility:
            type: number
            label: "GBP/USD Volatility (30 days)"
            description: "Standard deviation of GBP/USD exchange rate over last 30 days"
            sql: |
              STDDEV(
                CASE
                  WHEN ${target_currency} = 'USD'
                  THEN ${exchange_rate}
                END
              )
            format: number
            round: 4

          # Percentage of rates above 1.25
          high_rate_percentage:
            type: number
            label: "% of Rates Above 1.25"
            sql: |
              (COUNT(CASE WHEN ${exchange_rate} > 1.25 THEN 1 END) * 100.0)
              / COUNT(*)
            format: percent
            round: 2
```

!!! warning "SQL Injection Risk"
    Custom SQL metrics use `${column_name}` syntax for column references. Always reference existing columns, never accept user input directly in custom SQL.

## Joins and Relationships

Define relationships between models to enable joins in Lightdash:

```yaml
# models/marts/core/fct_exchange_rates.yml
version: 2

models:
  - name: fct_exchange_rates
    description: Exchange rates fact table

    meta:
      lightdash:
        joins:
          # Join to dim_products (hypothetical — exchange rates don't actually join to products)
          - join: dim_products
            sql_on: ${fct_exchange_rates.target_currency} = ${dim_products.currency_code}
            type: left  # left, inner, full
```

More commonly, you'll join facts to dimensions:

```yaml
# models/marts/crm/fct_contacts.yml
version: 2

models:
  - name: fct_contacts
    description: HubSpot contacts fact table

    meta:
      lightdash:
        joins:
          - join: dim_companies
            sql_on: ${fct_contacts.company_id} = ${dim_companies.company_id}
            type: left
```

## Hidden Fields and Models

### Hide Specific Columns

```yaml
columns:
  - name: internal_notes
    meta:
      dimension:
        hidden: true  # Don't show in Lightdash Explore
```

### Hide Entire Models

```yaml
# In dbt_project.yml
models:
  dbt_transform:
    staging:
      +meta:
        lightdash:
          enabled: false
```

Or in the model YAML:

```yaml
# models/marts/core/internal_model.yml
version: 2

models:
  - name: internal_model
    meta:
      lightdash:
        enabled: false  # This model won't appear in Lightdash
```

## Lightdash-Specific Configuration Options

Full list of `meta` options:

```yaml
meta:
  lightdash:
    # Model-level config
    enabled: true | false  # Show/hide this model
    label: "Display Name"  # Override model name
    group: "Category"  # Group in sidebar (e.g., "Finance", "Marketing")
    description: "Override description"

    # Join configuration
    joins:
      - join: other_model_name
        sql_on: "${this_model.column} = ${other_model.column}"
        type: left | inner | full

    # Custom metrics (model-level)
    metrics:
      metric_name:
        type: number | string | date
        sql: "CUSTOM SQL EXPRESSION"
        label: "Display Name"
        description: "Metric description"
        format: number | percent | currency | id
        round: 2
        currency: GBP

# Column-level config
columns:
  - name: column_name
    meta:
      dimension:
        type: string | number | date | timestamp | boolean
        label: "Display Name"
        description: "Override column description"
        hidden: true | false
        round: 2  # For numbers
        format: currency | percent | number | id
        currency: GBP  # For currency format
        time_intervals: [DAY, WEEK, MONTH, YEAR]  # For date dimensions

      # Column-level metrics
      metrics:
        metric_name:
          type: count | count_distinct | sum | average | min | max | median
          label: "Display Name"
          description: "Metric description"
```

## Compiling and Syncing

After updating your dbt YAML files:

### Lightdash Cloud

1. Commit and push changes to GitHub
2. If auto-sync is enabled: Lightdash automatically recompiles
3. If not: Navigate to **Settings** → **Project** → **Compile project**

### Self-Hosted Lightdash

Same process as Cloud — Lightdash pulls from GitHub and recompiles.

### Local Testing (Optional)

Test Lightdash locally before deploying:

```sh
# Clone the Lightdash CLI (optional)
npm install -g @lightdash/cli

# Login to your Lightdash instance
lightdash login https://your-lightdash-url

# Compile and preview changes
lightdash preview
```

This starts a local Lightdash instance with your changes for testing before pushing to GitHub.

## Troubleshooting Discovery Issues

### Model Not Appearing in Lightdash

**Check:**
1. Is the model materialised? (`dbt run` completed successfully)
2. Is the model in the correct schema? (Lightdash queries `REPORTING` if configured)
3. Is `enabled: false` set anywhere in the YAML or `dbt_project.yml`?
4. Did you recompile the project in Lightdash after adding the model?

### Metrics Not Showing

**Check:**
1. Is the `meta.metrics` block correctly indented under the column?
2. Is the metric type valid? (`sum`, `average`, `count`, etc.)
3. Did you recompile after adding metrics?

**Debug:**
```sh
# Check the compiled manifest
cat target/manifest.json | jq '.nodes."model.dbt_transform.fct_exchange_rates".meta'
```

If `meta` is empty, the YAML is not being parsed correctly.

### Joins Not Working

**Check:**
1. Are both models enabled in Lightdash?
2. Is the `sql_on` condition valid SQL?
3. Are you using `${model.column}` syntax (not bare column names)?

## Example: Full fct_exchange_rates.yml

```yaml
version: 2

models:
  - name: fct_exchange_rates
    description: |
      Daily exchange rates for GBP to various currencies.
      Source: ECB and Yahoo Finance via dlt.

    meta:
      lightdash:
        label: "Exchange Rates"
        group: "Finance"

    columns:
      - name: exchange_rate_id
        description: Surrogate key
        tests:
          - unique
          - not_null
        meta:
          dimension:
            hidden: true

      - name: rate_date
        description: Date of the exchange rate
        tests:
          - not_null
        meta:
          dimension:
            type: date
            label: "Date"
            time_intervals: [DAY, WEEK, MONTH, QUARTER, YEAR]

      - name: base_currency
        description: Base currency (always GBP)
        tests:
          - not_null
          - accepted_values:
              values: ['GBP']
        meta:
          dimension:
            type: string

      - name: target_currency
        description: Target currency
        tests:
          - not_null
        meta:
          dimension:
            type: string
            label: "Currency"

      - name: exchange_rate
        description: Exchange rate (1 GBP = X target currency)
        tests:
          - not_null
        meta:
          dimension:
            type: number
            round: 4
            label: "Exchange Rate"
          metrics:
            average_rate:
              type: average
              label: "Average Exchange Rate"
              round: 4
            min_rate:
              type: min
              label: "Minimum Rate"
              round: 4
            max_rate:
              type: max
              label: "Maximum Rate"
              round: 4

      - name: loaded_at
        description: Timestamp when this row was loaded
        meta:
          dimension:
            type: timestamp
            hidden: true
```

## Summary

You've learned how to connect your dbt project to Lightdash:

- [x] **Lightdash reads dbt YAML** — metrics, dimensions, and relationships defined in code
- [x] **Use `meta` tags** to configure Lightdash behavior (labels, types, hidden fields)
- [x] **Define metrics** in `columns.meta.metrics` (average, sum, count, custom SQL)
- [x] **Configure dimensions** with types, formats, and display options
- [x] **Hide staging/intermediate models** to expose only BI-ready tables
- [x] **Joins** defined in `meta.lightdash.joins` enable cross-model queries
- [x] **Recompile after changes** to sync Lightdash with your dbt project

Your dbt project is now the single source of truth for metrics and business logic. Changes in Git automatically flow to Lightdash.

## What's Next

Now that Lightdash understands your dbt models, define metrics in your existing mart models to enable self-service analytics.

Continue to [Define Metrics](9-define-metrics.md) →
