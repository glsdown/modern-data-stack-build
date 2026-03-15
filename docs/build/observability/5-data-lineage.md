# Data Lineage

On this page, you will:

- [x] Understand data lineage types (table-level, column-level, end-to-end)
- [x] View lineage in OpenMetadata UI
- [x] Perform impact analysis to understand downstream effects
- [x] Debug data quality issues using lineage
- [x] Automate lineage documentation with dbt

## Overview

Data lineage shows how data flows through your platform: from source systems through pipelines, transformations, and into dashboards. It answers critical questions:

- **"Where did this data come from?"** (upstream lineage)
- **"What uses this table?"** (downstream lineage)
- **"If I change this column, what breaks?"** (impact analysis)
- **"Why is this dashboard showing wrong data?"** (root cause debugging)

OpenMetadata automatically generates lineage by:
1. Analysing dbt DAG (`manifest.json`) for transformation lineage
2. Querying Snowflake metadata for table dependencies
3. Connecting to Lightdash for dashboard lineage
4. Tracking Prefect pipeline execution

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         DATA LINEAGE FLOW                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Source Systems        Pipelines         Transformations    Dashboards │
│  ──────────────        ─────────         ──────────────     ────────── │
│                                                                         │
│  ┌──────────┐         ┌─────────┐       ┌──────────────┐   ┌────────┐ │
│  │ HubSpot  │──dlt──▶ │ DLT_DB  │──────▶│ stg_contacts │──▶│ Lightd │ │
│  │ contacts │         │.raw_    │       │              │   │ -ash   │ │
│  │ API      │         │contacts │       └──────┬───────┘   │ KPIs   │ │
│  └──────────┘         └─────────┘              │           └────────┘ │
│                                                 ▼                       │
│  ┌──────────┐         ┌─────────┐       ┌──────────────┐   ┌────────┐ │
│  │ Shopify  │─Airby─▶ │ AIRBYTE │──────▶│ int_customer │──▶│ Revenue│ │
│  │ orders   │  te     │_DB.raw_ │       │ _orders      │   │ Dashbd │ │
│  │ API      │         │orders   │       └──────┬───────┘   └────────┘ │
│  └──────────┘         └─────────┘              │                       │
│                                                 ▼                       │
│                                          ┌──────────────┐               │
│                                          │ fct_orders   │               │
│                                          │ (final fact) │               │
│                                          └──────────────┘               │
│                                                                         │
│  Column-level lineage tracks individual columns through each step      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Lineage Types

### Table-Level Lineage

**What it shows:** Which tables depend on which other tables.

**Example:**
```
DLT_DB.raw_contacts
    ↓
stg_contacts
    ↓
int_customer_contacts
    ↓
dim_customers
```

**Use case:** Understanding overall data flow, identifying critical tables.

### Column-Level Lineage

**What it shows:** How individual columns transform from source to final table.

**Example:**
```
HubSpot API: contact.email
    ↓ (extracted by dlt)
DLT_DB.raw_contacts.properties__email
    ↓ (transformed by dbt)
stg_contacts.contact_email
    ↓ (joined and enriched)
dim_customers.primary_email
    ↓ (used in dashboard)
Lightdash: "Customer Email" metric
```

**Use case:** PII tracking, GDPR compliance, understanding data transformations.

### End-to-End Lineage

**What it shows:** Complete path from source API to dashboard chart.

**Example:**
```
HubSpot Contacts API
    ↓ (dlt pipeline in Prefect)
DLT_DB.raw_contacts
    ↓ (dbt staging model)
stg_contacts
    ↓ (dbt mart model)
dim_customers
    ↓ (Lightdash metric)
"Total Customers" KPI
    ↓ (Lightdash dashboard)
"Marketing Overview" dashboard
```

**Use case:** Complete visibility, stakeholder communication, debugging pipelines.

## View Lineage in OpenMetadata

### Navigate to a Table

1. Log into OpenMetadata at your deployed URL
2. Search for `fct_orders` in the search bar
3. Click on the table name

### View Table-Level Lineage

1. Click the **Lineage** tab
2. You'll see a graph showing:
   - **Upstream tables** (sources this table depends on)
   - **Downstream tables** (tables that depend on this one)
   - **Transformation logic** (dbt SQL from model)

**Example view:**

```
Upstream:                    Current:              Downstream:
┌─────────────┐             ┌────────────┐        ┌──────────────┐
│ stg_orders  │────────────▶│ fct_orders │───────▶│ Lightdash:   │
└─────────────┘             └────────────┘        │ Revenue KPIs │
                                                  └──────────────┘
┌─────────────┐                   │
│ stg_        │───────────────────┘
│ customers   │
└─────────────┘
```

### View Column-Level Lineage

1. Within the table view, click on a specific column (e.g., `customer_email`)
2. Click **Column Lineage**
3. OpenMetadata shows:
   - Source column in raw table
   - Transformations applied in each dbt model
   - Final column in fact/dimension table
   - Usage in dashboards (if connected)

**Example column lineage:**

```
Source: DLT_DB.raw_contacts.properties__email
    ↓ [stg_contacts model]
    SELECT properties__email AS contact_email
    ↓
Staging: stg_contacts.contact_email
    ↓ [int_customer_contacts model]
    SELECT contact_email
    ↓
Intermediate: int_customer_contacts.contact_email
    ↓ [dim_customers model]
    SELECT contact_email AS primary_email
    ↓
Final: dim_customers.primary_email
```

### Expand Lineage Depth

Use the **+** buttons on nodes to expand upstream or downstream lineage further.

**Default view:** 2 levels upstream, 2 levels downstream
**Expanded view:** Up to 10 levels in each direction

### Filter by Layer

Click **Filters** and select specific layers:
- **Sources** (raw data from APIs)
- **Staging** (dbt staging models)
- **Marts** (dbt fact/dimension tables)
- **Dashboards** (Lightdash visualisations)

## Impact Analysis

Impact analysis answers: **"If I change this, what breaks?"**

### Scenario: Rename a Column

**Question:** "I want to rename `customer_email` to `email_address` in `dim_customers`. What will break?"

**Process:**

1. Navigate to `dim_customers` in OpenMetadata
2. Click on `customer_email` column
3. Click **Column Lineage**
4. View **downstream usage**:
   - ✓ Used in `fct_orders` model (JOIN condition)
   - ✓ Used in Lightdash metric "Customer Email Domain"
   - ✓ Used in Lightdash dashboard "Customer Demographics"

**Impact:**
- **Breaking change** — 1 dbt model, 1 Lightdash metric, 1 dashboard affected
- **Required changes:**
  1. Update `fct_orders.sql` JOIN clause
  2. Update Lightdash metric definition
  3. Test all downstream dashboards

**Decision:** Either:
- Rename and update all downstream references (acceptable for 3 changes)
- Add a new column `email_address` and deprecate `customer_email` gradually
- Don't rename (not worth the effort)

### Scenario: Delete a Staging Model

**Question:** "Can I safely delete `stg_exchange_rates` model?"

**Process:**

1. Navigate to `stg_exchange_rates` in OpenMetadata
2. Click **Lineage** tab
3. Check **downstream tables**:
   - Used by `fct_orders` (currency conversion)
   - Used by `fct_revenue` (reporting currency)
   - Used by Lightdash metric "Revenue (GBP)"

**Impact:**
- **Breaking change** — 2 dbt models, multiple dashboards affected
- **Critical dependency** — revenue reporting depends on this

**Decision:** Do not delete. This is a critical dependency.

### Scenario: Change Data Type

**Question:** "What if I change `order_total` from `DECIMAL(10,2)` to `INTEGER`?"

**Process:**

1. Check column lineage for `order_total`
2. Identify downstream usage:
   - Used in aggregations: `SUM(order_total)` (OK with INTEGER)
   - Used in calculations: `order_total * 0.2` for VAT (precision loss!)
   - Used in Lightdash: "Average Order Value" (precision loss!)

**Impact:**
- **Functional impact** — Loss of decimal precision in VAT calculations
- **Business impact** — Financial reports will be incorrect

**Decision:** Do not change. Decimal precision is required for currency.

## Debugging with Lineage

### Scenario: Dashboard Shows Wrong Data

**Problem:** "The 'Revenue by Customer' dashboard shows $0 for all customers today, but yesterday it was fine."

**Debugging process:**

#### Step 1: Identify the Data Source

1. Navigate to Lightdash dashboard in OpenMetadata (if connected)
2. Click on the "Revenue by Customer" dashboard
3. View **Lineage** — shows it queries `fct_revenue` table

#### Step 2: Check the Fact Table

1. Navigate to `fct_revenue` in OpenMetadata
2. View **upstream lineage**:
   ```
   stg_orders → int_revenue_calculations → fct_revenue
   ```
3. Check data quality tests in dbt:
   - `not_null` test on `revenue_amount` — ✓ Passing
   - `expect_column_values_to_be_positive` — ✗ **FAILING**

**Finding:** The test is failing, meaning negative or zero values exist.

#### Step 3: Trace Upstream

1. Navigate to `int_revenue_calculations`
2. View lineage → depends on `stg_orders` and `stg_exchange_rates`
3. Check recent changes:
   - `stg_exchange_rates` last updated: **2 days ago**
   - `stg_orders` last updated: **2 hours ago** ✓ Fresh

**Finding:** Exchange rates are stale.

#### Step 4: Trace to Source

1. Navigate to `stg_exchange_rates`
2. View upstream lineage → `DLT_DB.raw_exchange_rates`
3. Check Prefect pipeline:
   - Last successful run: **2 days ago**
   - Last 3 runs: **Failed** (API rate limit exceeded)

**Root cause identified:** dlt pipeline for exchange rates is failing due to API rate limits.

#### Step 5: Fix and Verify

1. Fix the dlt pipeline (increase retry backoff)
2. Re-run pipeline → Loads fresh exchange rates
3. dbt automatically re-runs (Prefect automation)
4. Check dashboard → Revenue values restored

**Time to resolution:** 20 minutes (with lineage) vs 2 hours (manual investigation)

### Scenario: Duplicate Rows in Fact Table

**Problem:** "The `fct_orders` table has 2x as many rows as expected."

**Debugging process:**

#### Step 1: Check Upstream Lineage

1. Navigate to `fct_orders` in OpenMetadata
2. View upstream tables:
   ```
   stg_orders
   stg_customers
   stg_order_lines
   ```

#### Step 2: Row Count Comparison

Run row count checks in Snowflake:

```sql
-- Source
SELECT COUNT(*) FROM stg_orders;         -- 10,000 rows
SELECT COUNT(*) FROM stg_customers;      -- 5,000 rows
SELECT COUNT(*) FROM stg_order_lines;    -- 25,000 rows

-- Fact table
SELECT COUNT(*) FROM fct_orders;         -- 50,000 rows (expected: 25,000)
```

**Finding:** `fct_orders` has 2x rows expected (should match `stg_order_lines` at the grain of order line).

#### Step 3: Review dbt SQL

1. In OpenMetadata, click on `fct_orders`
2. View **SQL** tab (shows dbt model SQL)
3. Identify the issue:

```sql
-- WRONG: Missing DISTINCT or improper JOIN
SELECT
    ol.order_line_id,
    o.order_id,
    c.customer_id
FROM stg_order_lines ol
LEFT JOIN stg_orders o ON ol.order_id = o.order_id
LEFT JOIN stg_customers c ON o.customer_id = c.customer_id
LEFT JOIN stg_customers c2 ON o.billing_customer_id = c2.customer_id  -- DUPLICATE JOIN
```

**Root cause:** Joining to `stg_customers` twice without proper grain management creates a Cartesian product.

**Fix:** Remove duplicate join or use a UNION approach.

#### Step 4: Verify Downstream Impact

1. Check downstream lineage for `fct_orders`
2. Identify affected dashboards:
   - "Order Volume" dashboard (2x inflated)
   - "Revenue by Customer" (duplicated revenue)

**Fix applied:** Corrected dbt model, re-ran, verified row counts match expectations.

## Automate Lineage with dbt

dbt automatically provides lineage through the `manifest.json` file. Ensure your dbt project follows best practices for lineage clarity.

### Use `ref()` for All Dependencies

**Good:**
```sql
-- models/marts/fct_orders.sql
SELECT
    order_id,
    customer_id,
    order_total
FROM {{ ref('stg_orders') }}
LEFT JOIN {{ ref('dim_customers') }} USING (customer_id)
```

dbt generates lineage:
```
stg_orders → fct_orders
dim_customers → fct_orders
```

**Bad:**
```sql
-- AVOID: Hard-coded table references
SELECT
    order_id,
    customer_id,
    order_total
FROM analytics.staging.stg_orders  -- Lineage not tracked!
```

### Document Column Lineage in `schema.yml`

```yaml
# models/marts/core/fct_orders.yml
models:
  - name: fct_orders
    description: "Fact table of all orders with customer and product details"
    columns:
      - name: order_id
        description: "Unique order identifier from Shopify API (via Airbyte)"
        meta:
          source: "AIRBYTE_DB.raw_orders.id"

      - name: customer_email
        description: "Customer email from HubSpot Contacts (via dlt)"
        meta:
          source: "DLT_DB.raw_contacts.properties__email"
          pii: true  # Flag for GDPR compliance
```

OpenMetadata reads these `meta` tags to enhance column lineage.

### Tag PII Columns

Use dbt meta tags to identify Personal Identifiable Information (PII):

```yaml
columns:
  - name: customer_email
    meta:
      pii: true
      pii_type: "email"

  - name: customer_phone
    meta:
      pii: true
      pii_type: "phone"
```

OpenMetadata can filter lineage by PII columns, useful for:
- **GDPR compliance** — track where PII is stored and used
- **Data deletion** — identify all locations of a customer's email for right-to-erasure requests

### Use dbt Exposures for Dashboards

Define dashboards as dbt exposures to include them in lineage:

```yaml
# models/exposures.yml
exposures:
  - name: marketing_kpis_dashboard
    type: dashboard
    maturity: high
    url: https://lightdash.yourcompany.com/dashboards/marketing-kpis
    description: "Key marketing metrics for leadership team"

    depends_on:
      - ref('fct_contacts')
      - ref('fct_campaigns')
      - ref('dim_customers')

    owner:
      name: "Marketing Analytics Team"
      email: marketing-analytics@yourcompany.com
```

OpenMetadata imports exposures, showing complete lineage from source to dashboard.

## Best Practices for Lineage

### 1. Always Use `ref()` in dbt

Never hard-code table references. Always use `{{ ref('model_name') }}` or `{{ source('schema', 'table') }}`.

### 2. Document Every Model

Add descriptions to all models and columns in `schema.yml`. This context appears in OpenMetadata lineage views.

### 3. Tag Critical Data

Use dbt tags to identify critical tables:

```yaml
models:
  - name: fct_revenue
    tags: ['critical', 'financial', 'daily']
```

OpenMetadata can filter lineage by tags (e.g., show only `critical` tables).

### 4. Keep Lineage Shallow

**Good:** 3-5 transformation layers (raw → staging → intermediate → mart)
**Bad:** 10+ transformation layers (hard to understand, slow lineage queries)

If lineage graphs become too complex, consider:
- Consolidating intermediate models
- Creating clear layer boundaries
- Using views for simple transformations

### 5. Review Lineage During Code Reviews

Before merging dbt changes:
1. Check OpenMetadata lineage for affected downstream tables
2. Verify impact analysis shows expected dependencies
3. Ensure no unexpected downstream breakage

### 6. Automate Lineage Updates

Add OpenMetadata ingestion to your Prefect pipeline:

```python
# Run after dbt to update lineage
@task
def update_openmetadata_lineage():
    shell_run_command(
        command="metadata ingest -c ingestion/dbt-config.yaml",
        return_all=True
    )
```

This ensures lineage stays current after every dbt run.

## Lineage Limitations

### What OpenMetadata Cannot Track

1. **External BI tools without connectors** — If using Tableau/Power BI without OpenMetadata connectors, dashboard lineage won't appear
2. **Manual SQL queries** — Ad-hoc queries in Snowsight aren't tracked
3. **Spreadsheet exports** — If analysts export to Excel, that lineage is lost
4. **Cross-platform transformations** — Data transformed in Python scripts outside dbt may not appear

### Workarounds

**For Python transformations:**
- Add metadata to OpenMetadata via API
- Document in dbt model descriptions (e.g., "This table is post-processed by `scripts/enrich_data.py`")

**For external BI tools:**
- Use dbt exposures to manually document dashboards
- Migrate to Lightdash for automatic lineage

**For manual queries:**
- Encourage analysts to use dbt or saved views instead
- Document common queries as dbt metrics

## Summary

You've learned how to use data lineage for observability:

- [x] **Table-level lineage** shows dependencies between tables
- [x] **Column-level lineage** tracks individual columns from source to dashboard
- [x] **Impact analysis** identifies what breaks when making changes
- [x] **Debugging with lineage** speeds up root cause analysis from hours to minutes
- [x] **dbt best practices** ensure lineage is accurate and complete
- [x] **PII tracking** for GDPR compliance using meta tags

Lineage transforms data debugging from guesswork into systematic investigation.

## What's Next

Monitor pipeline health with Prefect observability features.

Continue to [Prefect Monitoring](6-prefect-monitoring.md) →
