# Testing and Documentation

On this page, you will:

- [x] Understand the full testing strategy for a dbt project
- [x] Configure `dbt_project_evaluator` to enforce project standards with custom checks
- [x] Write model and column descriptions for the docs site
- [x] Use doc blocks for reusable documentation
- [x] Generate and view the dbt documentation locally

## Overview

A well-tested, well-documented dbt project is the difference between a data warehouse that analysts trust and one they work around. This page covers:

1. **Generic tests** — column-level assertions declared in YAML
2. **Singular tests** — custom SQL assertions in the `tests/` directory
3. **`dbt_project_evaluator`** — automated checks for project structure and coverage with custom rules
4. **Documentation** — model and column descriptions that power the docs site
5. **Doc blocks** — reusable documentation snippets for consistency

## Testing Strategy

### Layer-by-Layer Testing Approach

Different layers warrant different levels of testing:

| Layer | Priority tests | Why |
|-------|---------------|-----|
| **Staging** | `unique`, `not_null` on primary keys; `accepted_values` on categorical columns | Catch issues in raw data early, before they propagate |
| **Intermediate** | Relationship tests between joined tables; range checks on derived metrics | Validate business logic in the transformation |
| **Marts** | Comprehensive column tests; `unique_combination_of_columns` on composite keys; freshness | Final validation before data reaches analysts |
| **Reporting** | Light tests (marts already tested); relationship back to mart models | Confirm the reporting view returns the expected rows |

### Generic Tests

Generic tests are declared in `.yml` files. dbt ships with four built-ins:

```yaml
columns:
  - name: exchange_rate_id
    tests:
      - unique           # no duplicate values
      - not_null         # no nulls

  - name: lifecycle_stage
    tests:
      - accepted_values:
          values: ['subscriber', 'lead', 'marketingqualifiedlead',
                   'salesqualifiedlead', 'opportunity', 'customer', 'evangelist', 'other']

  - name: base_currency
    tests:
      - relationships:
          to: ref('dim_currencies')
          field: currency_code
```

### `dbt_expectations` Tests

The `dbt_expectations` package provides column-level tests modelled on Great Expectations:

```yaml
columns:
  - name: exchange_rate
    tests:
      - not_null
      - dbt_expectations.expect_column_values_to_be_between:
          min_value: 0
          strictly: true           # > 0, not >= 0
      - dbt_expectations.expect_column_values_to_be_of_type:
          column_type: float

  - name: email
    tests:
      - dbt_expectations.expect_column_values_to_match_regex:
          regex: '^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$'

  - name: rate_date
    tests:
      - dbt_expectations.expect_column_values_to_be_between:
          min_value: "'2020-01-01'::date"
          max_value: "current_date + interval '1 day'"
          row_condition: "rate_date is not null"
```

Useful `dbt_expectations` tests:

| Test | Use case |
|------|---------|
| `expect_column_values_to_be_between` | Numeric ranges, date ranges |
| `expect_column_values_to_match_regex` | Email format, currency codes, IDs |
| `expect_column_values_to_be_of_type` | Type assertions |
| `expect_table_row_count_to_be_between` | Volume checks |
| `expect_column_pair_values_a_to_be_greater_than_b` | Field comparisons (e.g. `end_date > start_date`) |
| `expect_column_to_exist` | Structural validation |

## Utility Models

Some tests and models share a common building block: a complete sequence of dates. Rather than generating this inline with `dbt_utils.date_spine` in every place it's needed, the project maintains a single `util_date_spine` model in `models/utilities/`.

### `util_date_spine`

Create `models/utilities/util_date_spine.sql`:

```sql
/*
A complete sequence of calendar dates from the start of data collection
to today. Materialised as a table so downstream tests and models can
join against it cheaply without recomputing the spine each time.

Extend `start_date` if you backfill historical data before 2020-01-01.
*/

{{ config(materialised='table') }}

{{
    dbt_utils.date_spine(
        datepart='day',
        start_date="cast('2020-01-01' as date)",
        end_date="current_date + interval '1 day'"
    )
}}
```

Add a YAML file `models/utilities/util_date_spine.yml`:

```yaml
version: 2

models:
  - name: util_date_spine
    description: >
      A complete sequence of calendar dates from 2020-01-01 to today.
      Used as a reference table for completeness tests and date-based joins.
    columns:
      - name: date_day
        description: A single calendar date.
        tests:
          - unique
          - not_null
```

### Singular Tests

Singular tests are custom SQL queries in the `tests/` directory. A test passes if it returns **zero rows**. Use them for business rules too complex for generic tests.

#### Exchange rate completeness

This test uses `util_date_spine` to assert that exchange rate data exists for every expected calendar day. A left join against the fact table reveals any gaps:

Create `tests/assert_exchange_rates_complete.sql`:

```sql
/*
Asserts that fct_exchange_rates contains a GBP rate for every calendar day
from the start of data collection up to yesterday.

GBP is used as a proxy: if GBP is missing for a given date, the daily load
was either skipped or failed. A missing row will cause this test to fail.
*/

with expected_dates as (

    select date_day
    from {{ ref('util_date_spine') }}
    where date_day >= '2026-01-01'
      and date_day < current_date  -- yesterday is the latest we expect

),

actual_dates as (

    select distinct rate_date
    from {{ ref('fct_exchange_rates') }}
    where target_currency = 'GBP'

),

missing_dates as (

    select
        ed.date_day         as missing_date,
        'GBP' :: varchar    as expected_currency

    from expected_dates ed
    left join actual_dates ad
        on ed.date_day = ad.rate_date
    where ad.rate_date is null

)

select * from missing_dates
```

Create `tests/assert_no_duplicate_contacts.sql`:

```sql
/*
Asserts that fct_contacts has no duplicate email addresses.
Each email should appear exactly once.
*/

select
    email,
    count(*) as row_count
from {{ ref('fct_contacts') }}
group by email
having count(*) > 1
```

### Test Severity

Not all test failures should block a production run. Configure severity levels:

```yaml
columns:
  - name: exchange_rate
    tests:
      - not_null:
          severity: error          # blocks production run
      - dbt_expectations.expect_column_values_to_be_between:
          min_value: 0
          strictly: true
          severity: warn           # logs warning, does not block
```

Use `error` for tests that indicate corrupt or missing data, and `warn` for anomaly checks that may trigger for valid business reasons.

## `dbt_project_evaluator`

`dbt_project_evaluator` is a dbt package that runs as a set of models — it queries your project's metadata and flags violations of best practices. Run it as part of CI to prevent technical debt from accumulating.

### What It Checks

| Category | Rule | Example violation |
|----------|------|-------------------|
| **Coverage** | All models must have a description | `fct_contacts` has no description |
| **Coverage** | All mart models must have at least one test | `dim_currencies` has no tests |
| **Naming** | Models follow naming conventions | Model named `contacts_final` instead of `fct_contacts` |
| **Structure** | Staging models don't ref other staging models | `stg_a` does `ref('stg_b')` |
| **Structure** | Sources are referenced by at least one model | Source defined but never used |
| **Performance** | No direct references to sources in marts | Mart bypasses staging layer |

### Variable Overrides

The `vars` block in `dbt_project.yml` contains variables for the project. `dbt_project_evaluator` uses a number of variables which can be [customised](https://dbt-labs.github.io/dbt-project-evaluator/latest/customization/overriding-variables/) to provide different thresholds for each rule.

In addition, there are variables which control which folder names and prefixes are considered valid for each model type. Add these to `dbt_project.yml`:

```yaml
vars:
  # dbt_project_evaluator variable overrides
  dbt_project_evaluator:
    model_types: ['staging', 'intermediate', 'marts', 'other']

    staging_folder_name: 'staging'
    intermediate_folder_name: 'intermediate'
    marts_folder_name: 'marts'

    staging_prefixes: ['stg_']
    intermediate_prefixes: ['int_']
    marts_prefixes: ['fct_', 'dim_']
    other_prefixes: ['rpt_']
```

To add a model type (for example, a `utilities` layer):

```yaml
vars:
  dbt_project_evaluator:
    model_types: ['staging', 'intermediate', 'marts', 'utilities', 'other']
    utilities_folder_name: 'utilities'
    utilities_prefixes: ['util_']
```

### Exceptions Seed

The package ships with an empty `dbt_project_evaluator_exceptions` seed that lets you suppress specific violations. Disable the package seed in `dbt_project.yml` so you can provide your own version:

```yaml
# Seed configurations
seeds:
  dbt_transform:
    +schema: seeds
  # Disable the package's built-in exceptions seed so we can provide our own
  dbt_project_evaluator:
    dbt_project_evaluator_exceptions:
      +enabled: false
```

Create `seeds/dbt_project_evaluator_exceptions.csv`:

```
fct_name,column_name,id_to_exclude,comment
```

The columns are:

| Column | Description |
|--------|-------------|
| `fct_name` | The project_evaluator fact table where the exception applies (e.g. `fct_multiple_sources_joined`) |
| `column_name` | The column in that fact table to match against |
| `id_to_exclude` | The value to exclude — supports `like` wildcards (e.g. `stg_%_unioned`) |
| `comment` | Explanation of why this exception exists |

### On-Run-End Hook

Add an `on-run-end` hook in `dbt_project.yml` to print evaluator issues after every run. This makes failures visible even when running locally:

```yaml
on-run-end:
  # Print dbt_project_evaluator issues to the logs after every run
  - "{{ dbt_project_evaluator.print_dbt_project_evaluator_issues() }}"
```

Example output:

```
┌─────────────────────────────────────────────────────────────────────────┐
│ dbt_project_evaluator issues                                            │
├─────────────────────────┬───────────────────────────────────────────────┤
│ fct_name                │ issue                                         │
├─────────────────────────┼───────────────────────────────────────────────┤
│ fct_undocumented_models │ stg_dlt__products has no description          │
└─────────────────────────┴───────────────────────────────────────────────┘
```

### Custom Checks

Extend the evaluator with project-specific rules by creating `fct_` models under `models/utilities/dbt_project_evaluator/`. These follow the same pattern as the package's own fact tables: they query `stg_nodes` and other staging models exposed by `dbt_project_evaluator`, and use the `dbt_project_evaluator.is_empty` generic test — which passes when the model returns zero rows.

**Example: require `on_schema_change: append_new_columns` on incremental models**

The default dbt behaviour for incremental models is `on_schema_change: ignore`, which silently drops new columns from incremental runs. This check enforces the safer `append_new_columns` setting across all incremental models in the project.

`stg_nodes` (exposed by `dbt_project_evaluator`) contains a row for every node in the project graph, including the `on_schema_change` configuration for each model.

Create `models/utilities/dbt_project_evaluator/fct_incremental_on_schema_change.sql`:

```sql
{{ config(materialized='table') }}

with nodes as (
    select * from {{ ref('stg_nodes') }}
)

select
    unique_id,
    name,
    file_path,
    on_schema_change     as current_setting,
    'append_new_columns' as expected_setting
from nodes
where resource_type = 'model'
  and package_name = 'dbt_transform'
  and materialized = 'incremental'
  and coalesce(on_schema_change, 'ignore') != 'append_new_columns'
```

Create `models/utilities/dbt_project_evaluator/fct_incremental_on_schema_change.yml`:

```yaml
version: 2

models:
  - name: fct_incremental_on_schema_change
    description: >
      Incremental models in this project that do not set on_schema_change: append_new_columns.
      The default dbt behaviour (ignore) silently drops new columns from incremental runs.
      This model fails if any incremental model uses the unsafe default.
    tests:
      - dbt_project_evaluator.is_empty
```

To add further custom checks, create additional `fct_` models in the same folder following the same pattern.

### CI Selector

Create `selectors.yml` in the project root to run project_evaluator rules, custom checks, and the exceptions seed together in a single command:

```yaml
selectors:
  - name: ci_quality_checks
    description: dbt_project_evaluator rules, custom check models, and exceptions seed
    definition:
      union:
        - method: package
          value: dbt_project_evaluator
        - method: path
          value: models/utilities/dbt_project_evaluator
        - method: path
          value: seeds/dbt_project_evaluator_exceptions.csv
```

In CI and locally:

```sh
# Run all quality checks
dbt build --selector ci_quality_checks

# Run only project_evaluator rules
dbt build --select package:dbt_project_evaluator

# Run only custom checks
dbt build --select path:models/utilities/dbt_project_evaluator
```

!!! tip "Fail fast in CI, warn locally"
    Configure the evaluator to `error` in CI (blocking PRs with violations) and `warn` locally (letting developers iterate before they're ready to commit). Set the severity via environment variable in your CI workflow.

## Writing Good Documentation

### Model Descriptions

Every model should have a description that explains:
- What the model represents (one sentence)
- Where the data comes from
- Any important business logic or caveats
- Granularity (one row per X)

```yaml
models:
  - name: fct_exchange_rates
    description: >
      Daily USD base exchange rates for all currencies. Combines data from the
      dlt direct pipeline and the Snowpipe pipeline, with dlt taking precedence
      when both sources are available for the same date and currency.
      One row per target currency per date. Base currency is always USD.
    meta:
      owner: "@analytics-team"
      contains_pii: false
```

### Column Descriptions

Document every column in mart models — analysts will read these in the docs site:

```yaml
columns:
  - name: exchange_rate_id
    description: Surrogate key derived from rate_date and target_currency.
  - name: rate_date
    description: The date for which the exchange rate applies.
  - name: exchange_rate
    description: >
      Exchange rate from USD to the target currency. A value of 0.79
      means 1 USD = 0.79 of the target currency.
  - name: inverse_rate
    description: Inverse exchange rate (1 / exchange_rate). Useful for converting
      from the target currency back to USD. Null if exchange_rate is zero.
```

### Source Descriptions

Sources should document what the raw table contains, the loader, and the update frequency:

```yaml
sources:
  - name: dlt_open_exchange_rates
    description: >
      Raw exchange rate and currency data loaded by dlt from the Open Exchange Rates API.
      Exchange rates are updated daily; currencies are updated weekly.
    meta:
      owner: "@data-engineering"
      update_frequency: "daily (exchange_rates), weekly (currencies)"
```

## Doc Blocks

Doc blocks allow you to write reusable documentation that can be referenced across multiple models. This is especially useful for column descriptions that repeat across models (like IDs, timestamps, or common business terms).

### Creating Doc Blocks

Doc blocks live in Markdown files in the `docs/` directory. Create `docs/column_descriptions.md`:

````markdown
{% docs currency_code %}
ISO 4217 currency code (e.g. GBP, EUR, USD).
{% enddocs %}

{% docs exchange_rate %}
Exchange rate from USD to the target currency. A value of 0.79 means 1 USD = 0.79 of the target currency.
{% enddocs %}

{% docs created_at %}
Timestamp when this record was created in the source system.
{% enddocs %}

{% docs loaded_at %}
Timestamp when this record was loaded into Snowflake by the ingestion pipeline.
{% enddocs %}

{% docs surrogate_key %}
Surrogate key generated by dbt using `generate_surrogate_key` from dbt-utils. This is a deterministic hash of the business key columns.
{% enddocs %}
````

### Referencing Doc Blocks

Reference doc blocks in your YAML files using the `"{{ doc('block_name') }}"` syntax:

```yaml
models:
  - name: fct_exchange_rates
    columns:
      - name: exchange_rate_id
        description: "{{ doc('surrogate_key') }}"
      - name: target_currency
        description: "{{ doc('currency_code') }}"
      - name: exchange_rate
        description: "{{ doc('exchange_rate') }}"
      - name: loaded_at
        description: "{{ doc('loaded_at') }}"

  - name: dim_currencies
    columns:
      - name: currency_code
        description: "{{ doc('currency_code') }}"
```

### Benefits of Doc Blocks

1. **Consistency**: The same term is described identically across all models
2. **Maintainability**: Update the description once, and it propagates everywhere
3. **Reusability**: Common columns (timestamps, IDs, flags) documented once
4. **Documentation as code**: Doc blocks are versioned with your dbt project

!!! tip "Organise doc blocks by domain"
    Create separate doc files for different domains:
    - `docs/common_columns.md` — timestamps, IDs, metadata columns
    - `docs/business_terms.md` — domain-specific terminology
    - `docs/metrics.md` — calculated metrics and KPIs

## Generate and View Documentation

### Local Development

```sh
# Generate the docs site
dbt docs generate

# Serve the docs site at http://localhost:8080
dbt docs serve
```

The docs site includes:

- **Project overview**: all models, sources, and their descriptions
- **DAG lineage**: interactive dependency graph showing how models relate
- **Model details**: columns, tests, and descriptions for each model
- **Source freshness**: last loaded timestamps and freshness status

### Docs Hosting

For dbt Core, the generated docs are deployed to S3 + CloudFront as part of the production CI/CD pipeline. See [dbt Core Deployment](9-dbt-core-deployment.md) for the GitHub Actions workflow.

For dbt Cloud, docs are generated and hosted automatically after each job run.

## Running All Tests

```sh
# Run all tests
dbt test

# Run tests for a specific model and its upstreams
dbt test --select +fct_exchange_rates

# Run only tests tagged as critical
dbt test --select tag:critical

# Run tests in parallel (faster for large projects)
dbt test --threads 8

# Run source freshness checks
dbt source freshness
```

## Summary

You've built a comprehensive testing and documentation strategy:

- [x] Generic tests (`unique`, `not_null`, `accepted_values`, `relationships`) on all key columns
- [x] `dbt_expectations` for numeric ranges, regex matching, and type assertions
- [x] Singular tests for custom business rule validation
- [x] `dbt_project_evaluator` configured with variable overrides, exceptions seed, custom checks, and CI selector
- [x] Model and column descriptions following a consistent format
- [x] Doc blocks for reusable documentation
- [x] Local docs generation working

## What's Next

Set up CI/CD for dbt Core — slim CI on pull requests, production runs on merge, state deferral, and docs hosting.

Continue to [dbt Core Deployment](9-dbt-core-deployment.md) →
