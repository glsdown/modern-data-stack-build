# Data Quality with dbt

On this page, you will:

- [x] Master dbt's four generic tests (unique, not_null, accepted_values, relationships)
- [x] Install and use dbt_expectations for statistical and distribution tests
- [x] Write custom singular tests for complex business logic
- [x] Configure test severity levels (warn vs error)
- [x] Add where clauses and incremental model testing strategies
- [x] Integrate tests into CI/CD to fail builds on test failures

## Overview

dbt tests are the foundation of data quality in the modern data stack. They run SQL queries that return failing rows — if any rows are returned, the test fails. Tests run after every `dbt build` or `dbt test` command, providing immediate feedback on data quality.

This page builds on the basic dbt tests covered in [Data Transformation](../data-transformation/8-testing-and-documentation.md), adding advanced patterns and packages.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       dbt TESTING HIERARCHY                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Generic Tests              dbt_expectations           Singular Tests   │
│  ──────────────             ─────────────────          ───────────────  │
│  (Built-in)                 (Package)                  (Custom SQL)     │
│                                                                         │
│  • unique                   • expect_column_values_    • Complex        │
│  • not_null                   to_be_between              business      │
│  • accepted_values          • expect_column_values_      logic         │
│  • relationships              to_match_regex           • Multi-table   │
│                             • expect_table_row_          validation    │
│                               count_to_be_between      • Custom        │
│                             • expect_column_stddev_      aggregations  │
│                               to_be_between                            │
│                                                                         │
│  All tests run via: dbt test                                           │
│  Failed tests → Alert (SEV2) or Warn (SEV3) depending on config        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Recap: The Four Generic Tests

dbt ships with four built-in generic tests. These cover 80% of common data quality checks.

### 1. unique

**Ensures:** Column values are unique (no duplicates).

**Use case:** Primary keys, natural keys, unique identifiers.

```yaml
# models/marts/core/fct_exchange_rates.yml
columns:
  - name: exchange_rate_id
    tests:
      - unique
```

**Generated SQL:**
```sql
SELECT exchange_rate_id, COUNT(*) AS count
FROM analytics.marts.fct_exchange_rates
GROUP BY exchange_rate_id
HAVING COUNT(*) > 1;
```

If any rows are returned (duplicates exist), the test fails.

### 2. not_null

**Ensures:** Column has no NULL values.

**Use case:** Required fields, foreign keys, critical business data.

```yaml
columns:
  - name: customer_id
    tests:
      - not_null
```

**Generated SQL:**
```sql
SELECT *
FROM analytics.marts.fct_orders
WHERE customer_id IS NULL;
```

### 3. accepted_values

**Ensures:** Column values are in a predefined list.

**Use case:** Enums, status fields, categorical data.

```yaml
columns:
  - name: order_status
    tests:
      - accepted_values:
          values: ['pending', 'shipped', 'delivered', 'cancelled']
```

**Generated SQL:**
```sql
SELECT *
FROM analytics.marts.fct_orders
WHERE order_status NOT IN ('pending', 'shipped', 'delivered', 'cancelled');
```

### 4. relationships

**Ensures:** Foreign key values exist in the referenced table.

**Use case:** Referential integrity, joins.

```yaml
columns:
  - name: customer_id
    tests:
      - relationships:
          to: ref('dim_customers')
          field: customer_id
```

**Generated SQL:**
```sql
SELECT *
FROM analytics.marts.fct_orders
WHERE customer_id NOT IN (
    SELECT customer_id FROM analytics.marts.dim_customers
);
```

## dbt_expectations Package

[dbt_expectations](https://github.com/calogica/dbt-expectations) is a community package that adds 50+ tests inspired by Great Expectations. These tests cover statistical validation, distribution checks, and regex patterns.

### Installation

Add to `packages.yml`:

```yaml
# packages.yml
packages:
  - package: calogica/dbt_expectations
    version: 0.10.3
```

Install:

```sh
dbt deps
```

### Statistical Tests

#### expect_column_values_to_be_between

**Ensures:** Numeric values fall within a range.

```yaml
# models/marts/core/fct_exchange_rates.yml
columns:
  - name: exchange_rate
    tests:
      - dbt_expectations.expect_column_values_to_be_between:
          min_value: 0.01
          max_value: 10.0
          row_condition: "base_currency = 'GBP'"  # Optional filter
```

**Use case:** Validate exchange rates are within realistic bounds.

#### expect_column_mean_to_be_between

**Ensures:** Column mean is within expected range.

```yaml
columns:
  - name: order_total
    tests:
      - dbt_expectations.expect_column_mean_to_be_between:
          min_value: 50
          max_value: 200
          group_by: [order_date]  # Calculate mean per day
```

**Use case:** Detect anomalies in average order value.

#### expect_column_stdev_to_be_between

**Ensures:** Standard deviation is within expected range (detects volatility changes).

```yaml
columns:
  - name: exchange_rate
    tests:
      - dbt_expectations.expect_column_stdev_to_be_between:
          min_value: 0.001
          max_value: 0.1
          group_by: [target_currency]
```

**Use case:** Alert if exchange rate volatility suddenly increases.

### Row Count Tests

#### expect_table_row_count_to_be_between

**Ensures:** Table has expected number of rows.

```yaml
# models/marts/core/fct_exchange_rates.yml
models:
  - name: fct_exchange_rates
    tests:
      - dbt_expectations.expect_table_row_count_to_be_between:
          min_value: 1000  # At least 1000 rows
          max_value: 1000000  # No more than 1M rows
```

**Use case:** Detect data pipeline failures (too few rows) or duplication (too many rows).

#### expect_table_row_count_to_equal_other_table

**Ensures:** Two tables have the same row count.

```yaml
models:
  - name: fct_exchange_rates
    tests:
      - dbt_expectations.expect_table_row_count_to_equal_other_table:
          compare_model: source('raw', 'exchange_rates')
```

**Use case:** Validate no rows were lost in transformation.

### Format Validation

#### expect_column_values_to_match_regex

**Ensures:** Values match a regex pattern.

```yaml
columns:
  - name: email
    tests:
      - dbt_expectations.expect_column_values_to_match_regex:
          regex: '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$'
```

**Use case:** Validate email format, phone numbers, postcodes.

#### expect_column_values_to_match_like_pattern

**Ensures:** Values match a SQL LIKE pattern.

```yaml
columns:
  - name: product_id
    tests:
      - dbt_expectations.expect_column_values_to_match_like_pattern:
          like_pattern: 'PROD-%'
```

**Use case:** Validate IDs follow naming conventions.

### Distribution Tests

#### expect_column_quantile_values_to_be_between

**Ensures:** Percentiles fall within expected ranges.

```yaml
columns:
  - name: order_total
    tests:
      - dbt_expectations.expect_column_quantile_values_to_be_between:
          quantile: 0.95  # 95th percentile
          min_value: 100
          max_value: 500
```

**Use case:** Detect outliers in order totals.

## Custom Singular Tests

For complex business logic that can't be expressed with generic tests, write custom SQL tests.

### Example: Revenue Reconciliation

**Business rule:** Total revenue in `fct_orders` must match sum of line items in `fct_order_lines`.

```sql
-- tests/revenue_reconciliation.sql
WITH order_totals AS (
    SELECT
        order_id,
        order_total
    FROM {{ ref('fct_orders') }}
),
line_item_totals AS (
    SELECT
        order_id,
        SUM(quantity * unit_price) AS line_item_total
    FROM {{ ref('fct_order_lines') }}
    GROUP BY order_id
)
SELECT
    o.order_id,
    o.order_total,
    l.line_item_total,
    ABS(o.order_total - l.line_item_total) AS difference
FROM order_totals o
INNER JOIN line_item_totals l
    ON o.order_id = l.order_id
WHERE ABS(o.order_total - l.line_item_total) > 0.01  -- Allow 1 cent rounding
;
```

If any rows are returned (mismatches exist), the test fails.

### Example: Temporal Consistency

**Business rule:** `created_at` must always be before `updated_at`.

```sql
-- tests/created_before_updated.sql
SELECT *
FROM {{ ref('dim_customers') }}
WHERE created_at > updated_at
;
```

### Example: Cross-Table Consistency

**Business rule:** Every order must have at least one line item.

```sql
-- tests/orders_have_line_items.sql
SELECT o.order_id
FROM {{ ref('fct_orders') }} o
LEFT JOIN {{ ref('fct_order_lines') }} l
    ON o.order_id = l.order_id
WHERE l.order_id IS NULL
;
```

## Test Severity: Warn vs Error

Not all test failures require immediate action. Use `severity` to control behavior.

### Error (Default)

**Behavior:** Test failure causes `dbt build` or `dbt test` to exit with error code 1.

**Use case:** Critical data quality checks (primary keys, not_null on critical fields).

```yaml
columns:
  - name: customer_id
    tests:
      - unique:
          config:
            severity: error  # Default; can be omitted
```

**CI/CD impact:** Pull request fails if test fails.

### Warn

**Behavior:** Test failure logs a warning but doesn't fail the build.

**Use case:** Non-critical checks, exploratory tests, tests in development.

```yaml
columns:
  - name: product_description
    tests:
      - not_null:
          config:
            severity: warn  # Build succeeds even if test fails
```

**CI/CD impact:** Pull request passes, but warning is visible in logs.

### When to Use Each

| Severity | Use When | Example |
|----------|----------|---------|
| **error** | Test failure blocks business | `customer_id` is unique (duplicate customers break billing) |
| **error** | Referential integrity | Foreign keys exist (broken joins = wrong reports) |
| **warn** | Nice-to-have quality | Product descriptions present (missing OK, but not ideal) |
| **warn** | Test in development | New test; not confident in thresholds yet |
| **warn** | Statistical anomalies | Row count 10% lower than average (investigate, but don't block) |

## Where Clauses and Conditional Testing

Apply tests to a subset of rows using `where` or `row_condition`.

### where Clause (dbt Native)

**Filter rows before testing:**

```yaml
columns:
  - name: customer_id
    tests:
      - not_null:
          where: "order_status != 'cancelled'"
```

**Generated SQL:**
```sql
SELECT *
FROM analytics.marts.fct_orders
WHERE order_status != 'cancelled'  -- Applied first
    AND customer_id IS NULL;        -- Then test
```

**Use case:** "customer_id must be not null for non-cancelled orders."

### row_condition (dbt_expectations)

**Same as `where`, but for dbt_expectations tests:**

```yaml
columns:
  - name: order_total
    tests:
      - dbt_expectations.expect_column_values_to_be_between:
          min_value: 0
          max_value: 10000
          row_condition: "order_status IN ('shipped', 'delivered')"
```

**Use case:** "order_total must be between 0 and 10,000 for completed orders."

## Testing Incremental Models

Incremental models only process new rows (`is_incremental()`). Test strategies differ.

### Strategy 1: Test Full Table

**Approach:** Test the entire table, not just new rows.

```yaml
# Default behavior — tests run against full table
columns:
  - name: exchange_rate_id
    tests:
      - unique  # Checks all rows, including historical
```

**Pros:** Catches issues in historical data.
**Cons:** Slow for large tables (millions of rows).

### Strategy 2: Test Recent Data Only

**Approach:** Use `where` to test only recent rows.

```yaml
columns:
  - name: exchange_rate_id
    tests:
      - unique:
          where: "loaded_at >= CURRENT_DATE() - INTERVAL '7 days'"
```

**Pros:** Fast (tests only last 7 days).
**Cons:** Misses issues in older data.

### Strategy 3: Test on `dbt build --full-refresh`

**Approach:** Run full table tests only on full refresh (weekly).

```yaml
columns:
  - name: exchange_rate_id
    tests:
      - unique:
          config:
            enabled: "{{ target.name == 'prod' and not is_incremental() }}"
```

**Pros:** Fast incremental runs, thorough weekly validation.
**Cons:** Complex configuration.

**Recommended:** Strategy 2 (test recent data) for daily runs + Strategy 3 (full test) for weekly full refresh.

## CI/CD Integration

Fail builds on test failures to prevent bad data from reaching production.

### GitHub Actions Workflow

```yaml
# .github/workflows/dbt_ci.yml (in dbt-transform repo)
name: dbt CI

on:
  pull_request:
    branches:
      - main

jobs:
  dbt-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dbt
        run: uv add dbt-snowflake dbt-expectations

      - name: Add venv to PATH
        run: echo "$GITHUB_WORKSPACE/.venv/bin" >> $GITHUB_PATH

      - name: Install dbt packages
        run: dbt deps

      - name: Run dbt tests
        run: |
          dbt test --select state:modified+ \
            --defer --state .artifacts/ \
            --fail-fast
        env:
          DBT_SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
          DBT_SNOWFLAKE_USER: ${{ secrets.SNOWFLAKE_USER }}
          DBT_SNOWFLAKE_PRIVATE_KEY: ${{ secrets.SNOWFLAKE_PRIVATE_KEY }}
```

**Behavior:**
- Pull request triggers CI
- dbt runs tests on modified models only (`state:modified+`)
- If any test fails (severity: error), CI fails
- Pull request cannot merge until tests pass

### Test Selectors

Run specific subsets of tests:

```sh
# Run all tests
dbt test

# Run tests on one model
dbt test --select fct_exchange_rates

# Run tests on modified models + downstream
dbt test --select state:modified+

# Run only dbt_expectations tests
dbt test --select test_type:generic,package:dbt_expectations

# Run only error-severity tests
dbt test --select config.severity:error
```

## Best Practices

### 1. Test Every Model

**Rule:** Every model should have at least one test (typically on the primary key).

```yaml
# Minimum viable testing
models:
  - name: fct_exchange_rates
    columns:
      - name: exchange_rate_id
        tests:
          - unique
          - not_null
```

### 2. Test Critical Columns

**Focus on:**
- Primary keys (unique, not_null)
- Foreign keys (relationships)
- Required business fields (not_null)
- Enums (accepted_values)

### 3. Use Severity Appropriately

**error:** Breaks business logic, blocks dashboards
**warn:** Nice-to-have, exploratory

### 4. Document Test Logic

**Explain non-obvious tests:**

```yaml
columns:
  - name: order_total
    tests:
      - dbt_expectations.expect_column_values_to_be_between:
          min_value: 0
          max_value: 1000000
          config:
            meta:
              description: "Order totals capped at £1M to catch data entry errors"
```

### 5. Monitor Test Results Over Time

**Use Elementary (next page) to track:**
- Test pass rate trends
- Which tests fail most frequently
- When tests started failing (correlate with code changes)

## Example: Complete fct_exchange_rates Testing

```yaml
# models/marts/core/fct_exchange_rates.yml
version: 2

models:
  - name: fct_exchange_rates
    description: Daily exchange rates for GBP to various currencies

    # Table-level tests
    tests:
      - dbt_expectations.expect_table_row_count_to_be_between:
          min_value: 1000
          max_value: 1000000
          config:
            severity: warn

    columns:
      - name: exchange_rate_id
        description: Surrogate key
        tests:
          - unique:
              config:
                severity: error
          - not_null:
              config:
                severity: error

      - name: rate_date
        description: Date of the exchange rate
        tests:
          - not_null
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: "'2020-01-01'"
              max_value: "CURRENT_DATE() + INTERVAL '1 day'"
              config:
                severity: error

      - name: base_currency
        tests:
          - accepted_values:
              values: ['GBP']

      - name: target_currency
        tests:
          - not_null
          - dbt_expectations.expect_column_values_to_be_in_set:
              value_set: ['USD', 'EUR', 'JPY', 'AUD', 'CAD', 'CHF', 'NZD', 'SEK']
              config:
                severity: warn  # New currencies added occasionally

      - name: exchange_rate
        tests:
          - not_null
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0.01
              max_value: 1000
              config:
                severity: error
          - dbt_expectations.expect_column_mean_to_be_between:
              min_value: 0.5
              max_value: 2.0
              group_by: [target_currency]
              config:
                severity: warn
```

## Summary

You've mastered dbt data quality testing:

- [x] **Generic tests** — unique, not_null, accepted_values, relationships
- [x] **dbt_expectations** — Statistical validation, distribution checks, regex patterns
- [x] **Custom singular tests** — Complex business logic in SQL
- [x] **Severity levels** — error (blocks builds) vs warn (logs only)
- [x] **Where clauses** — Test subsets of rows
- [x] **Incremental model testing** — Test recent data or full table on refresh
- [x] **CI/CD integration** — Fail builds on test failures

dbt tests provide the foundation. Next, add automated anomaly detection with Elementary.

## What's Next

Install Elementary to track test results over time and detect anomalies automatically.

Continue to [Elementary Setup](3-elementary-setup.md) →
