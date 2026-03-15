# Define Metrics

On this page, you will:

- [x] Add Lightdash-compatible metrics to your existing dbt models
- [x] Define dimensions, metrics, and custom SQL aggregations
- [x] Configure formatting, labels, and data types for optimal UX
- [x] Test metrics in Lightdash Explore
- [x] Understand metrics as code workflow (Git → Lightdash sync)

## Overview

Metrics in Lightdash are defined in your dbt YAML files, not in a BI tool UI. This means:
- **Version-controlled** — metrics live in Git alongside your models
- **Testable** — metrics definitions are code, reviewed in pull requests
- **Single source of truth** — business logic defined once, used everywhere
- **Automatic sync** — changes in dbt flow to Lightdash automatically

This page walks through adding metrics to the dbt models you built in the [Data Transformation](../data-transformation/index.md) section.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       METRICS AS CODE WORKFLOW                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. Define Metrics in dbt        2. Git Commit & Push                  │
│  ──────────────────────          ──────────────────                    │
│                                                                         │
│  fct_exchange_rates.yml          git add models/                       │
│  ───────────────────────          git commit -m "Add metrics"          │
│  columns:                         git push origin main                 │
│    - name: exchange_rate                                               │
│      meta:                                  │                          │
│        metrics:                             │                          │
│          average_rate:                      ▼                          │
│            type: average                                               │
│                                  3. Lightdash Auto-Sync                │
│                                  ───────────────────                   │
│                                                                         │
│                                  • Detects push to main                │
│                                  • Clones dbt-transform                │
│                                  • Runs dbt compile                    │
│                                  • Updates Explores                    │
│                                                                         │
│                                             │                          │
│                                             ▼                          │
│                                  4. Metrics Available in UI            │
│                                  ────────────────────────             │
│                                                                         │
│                                  Lightdash Explore → fct_exchange_rates │
│                                  • Dimensions: rate_date, currency     │
│                                  • Metrics: average_rate ✅            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Update fct_exchange_rates.yml

Edit your existing `fct_exchange_rates.yml` to add Lightdash metrics:

```yaml
# models/marts/core/fct_exchange_rates.yml
version: 2

models:
  - name: fct_exchange_rates
    description: |
      Daily exchange rates for GBP to various currencies.
      Data sourced from ECB and Yahoo Finance via dlt and Snowpipe.

    meta:
      lightdash:
        label: "Exchange Rates"
        group: "Finance"

    columns:
      - name: exchange_rate_id
        description: Surrogate key for this exchange rate fact
        tests:
          - unique
          - not_null
        meta:
          dimension:
            type: string
            hidden: true  # Hide surrogate keys from analysts

      - name: rate_date
        description: Date of the exchange rate
        tests:
          - not_null
        meta:
          dimension:
            type: date
            label: "Date"
            time_intervals: [DAY, WEEK, MONTH, QUARTER, YEAR]
          # Date-based metrics
          metrics:
            count_of_rates:
              type: count
              label: "Number of Exchange Rates"
              description: "Count of daily exchange rate records"

      - name: base_currency
        description: Base currency (always GBP for this dataset)
        tests:
          - not_null
          - accepted_values:
              values: ['GBP']
        meta:
          dimension:
            type: string
            label: "Base Currency"

      - name: target_currency
        description: Target currency (USD, EUR, JPY, etc.)
        tests:
          - not_null
        meta:
          dimension:
            type: string
            label: "Target Currency"
          metrics:
            distinct_currencies:
              type: count_distinct
              label: "Number of Currencies"
              description: "Count of unique target currencies in the dataset"

      - name: exchange_rate
        description: |
          Exchange rate from base to target currency.
          1 GBP = X target currency units.
        tests:
          - not_null
        meta:
          dimension:
            type: number
            round: 4
            label: "Exchange Rate"
          metrics:
            average_exchange_rate:
              type: average
              label: "Average Exchange Rate"
              description: "Mean exchange rate across selected period"
              round: 4

            min_exchange_rate:
              type: min
              label: "Minimum Exchange Rate"
              description: "Lowest exchange rate in selected period"
              round: 4

            max_exchange_rate:
              type: max
              label: "Maximum Exchange Rate"
              description: "Highest exchange rate in selected period"
              round: 4

            median_exchange_rate:
              type: median
              label: "Median Exchange Rate"
              description: "Median exchange rate (50th percentile)"
              round: 4

      - name: data_source
        description: Source system for this exchange rate (ECB, YAHOO_FINANCE)
        meta:
          dimension:
            type: string
            label: "Data Source"

      - name: loaded_at
        description: Timestamp when this record was loaded into the warehouse
        meta:
          dimension:
            type: timestamp
            hidden: true  # Not useful for business analysis
```

### Custom SQL Metrics

Add a custom metric for volatility (standard deviation):

```yaml
      # Add to fct_exchange_rates.yml under the model's meta section
    meta:
      lightdash:
        label: "Exchange Rates"
        group: "Finance"
        metrics:
          # Custom SQL metric: volatility
          exchange_rate_volatility:
            type: number
            label: "Exchange Rate Volatility (Std Dev)"
            description: "Standard deviation of exchange rates (measure of volatility)"
            sql: "STDDEV(${exchange_rate})"
            round: 4
            format: number

          # Custom SQL metric: coefficient of variation
          coefficient_of_variation:
            type: number
            label: "Coefficient of Variation (%)"
            description: "Volatility relative to mean (std dev / mean * 100)"
            sql: "(STDDEV(${exchange_rate}) / AVG(${exchange_rate})) * 100"
            round: 2
            format: percent
```

## Update dim_products.yml

Add metrics to your products dimension:

```yaml
# models/marts/core/dim_products.yml
version: 2

models:
  - name: dim_products
    description: Product master data with current product information

    meta:
      lightdash:
        label: "Products"
        group: "Catalogue"

    columns:
      - name: product_key
        description: Surrogate key for the product dimension
        tests:
          - unique
          - not_null
        meta:
          dimension:
            type: string
            hidden: true

      - name: product_id
        description: Natural key from source system
        tests:
          - unique
          - not_null
        meta:
          dimension:
            type: string
            label: "Product ID"

      - name: product_name
        description: Product name
        tests:
          - not_null
        meta:
          dimension:
            type: string
            label: "Product Name"

      - name: product_category
        description: Product category
        meta:
          dimension:
            type: string
            label: "Category"
          metrics:
            distinct_categories:
              type: count_distinct
              label: "Number of Categories"

      - name: product_subcategory
        description: Product subcategory
        meta:
          dimension:
            type: string
            label: "Subcategory"

      - name: product_price
        description: Current product price in GBP
        meta:
          dimension:
            type: number
            round: 2
            format: currency
            currency: GBP
            label: "Product Price"
          metrics:
            average_price:
              type: average
              label: "Average Product Price"
              description: "Mean price across selected products"
              round: 2
              format: currency
              currency: GBP

            total_price_sum:
              type: sum
              label: "Total Price (Sum)"
              description: "Sum of product prices (not revenue - sum of prices only)"
              round: 2
              format: currency
              currency: GBP

            min_price:
              type: min
              label: "Cheapest Product Price"
              round: 2
              format: currency
              currency: GBP

            max_price:
              type: max
              label: "Most Expensive Product Price"
              round: 2
              format: currency
              currency: GBP

      - name: is_active
        description: Whether the product is currently active
        meta:
          dimension:
            type: boolean
            label: "Active Product"

      - name: created_at
        description: When the product was created in the source system
        meta:
          dimension:
            type: timestamp
            label: "Created At"

      - name: valid_from
        description: Start of validity period for this product version (Type 2 SCD)
        meta:
          dimension:
            type: timestamp
            label: "Valid From"
            hidden: true

      - name: valid_to
        description: End of validity period (NULL if current)
        meta:
          dimension:
            type: timestamp
            label: "Valid To"
            hidden: true

      - name: is_current
        description: Whether this is the current version of the product
        tests:
          - not_null
        meta:
          dimension:
            type: boolean
            label: "Current Version"
            hidden: false  # Useful for filtering to current products only
```

### Custom Metrics for Products

Add a metric for counting active products:

```yaml
    # Add to dim_products.yml under model's meta section
    meta:
      lightdash:
        label: "Products"
        group: "Catalogue"
        metrics:
          count_active_products:
            type: number
            label: "Number of Active Products"
            description: "Count of currently active products"
            sql: "COUNT(CASE WHEN ${is_active} = TRUE THEN 1 END)"

          count_current_products:
            type: number
            label: "Number of Current Product Versions"
            description: "Count of products with is_current = TRUE"
            sql: "COUNT(CASE WHEN ${is_current} = TRUE THEN 1 END)"

          average_days_since_created:
            type: number
            label: "Average Age (Days)"
            description: "Average number of days since product was created"
            sql: "AVG(DATEDIFF(day, ${created_at}, CURRENT_DATE()))"
            round: 0
```

## Add Metrics to fct_contacts.yml (Optional)

If you loaded HubSpot contacts via Airbyte, add metrics:

```yaml
# models/marts/crm/fct_contacts.yml
version: 2

models:
  - name: fct_contacts
    description: HubSpot contacts with basic contact information

    meta:
      lightdash:
        label: "Contacts"
        group: "CRM"

    columns:
      - name: contact_id
        description: HubSpot contact ID
        tests:
          - unique
          - not_null
        meta:
          dimension:
            type: string
            label: "Contact ID"
            hidden: true

      - name: email
        description: Contact email address
        meta:
          dimension:
            type: string
            label: "Email"

      - name: first_name
        description: Contact first name
        meta:
          dimension:
            type: string
            label: "First Name"

      - name: last_name
        description: Contact last name
        meta:
          dimension:
            type: string
            label: "Last Name"

      - name: company
        description: Contact company name
        meta:
          dimension:
            type: string
            label: "Company"

      - name: lifecycle_stage
        description: HubSpot lifecycle stage (lead, MQL, SQL, customer, etc.)
        meta:
          dimension:
            type: string
            label: "Lifecycle Stage"
          metrics:
            distinct_lifecycle_stages:
              type: count_distinct
              label: "Number of Lifecycle Stages"

      - name: created_at
        description: When the contact was created in HubSpot
        meta:
          dimension:
            type: timestamp
            label: "Created At"
            time_intervals: [DAY, WEEK, MONTH, QUARTER, YEAR]

      - name: loaded_at
        description: When this record was loaded into the warehouse
        meta:
          dimension:
            type: timestamp
            hidden: true

    meta:
      lightdash:
        label: "Contacts"
        group: "CRM"
        metrics:
          total_contacts:
            type: count
            label: "Total Contacts"
            description: "Total number of contacts in HubSpot"

          contacts_created_this_month:
            type: number
            label: "Contacts Created This Month"
            sql: |
              COUNT(
                CASE
                  WHEN DATE_TRUNC('month', ${created_at}) = DATE_TRUNC('month', CURRENT_DATE())
                  THEN 1
                END
              )
```

## Formatting and Display Options

### Number Formatting

```yaml
meta:
  dimension:
    type: number
    round: 2  # Round to 2 decimal places
    format: number  # Options: number, currency, percent, id
    currency: GBP  # For currency format
```

### Date Granularity

```yaml
meta:
  dimension:
    type: date
    time_intervals: [DAY, WEEK, MONTH, QUARTER, YEAR]
    # Users can group by day, week, month, quarter, or year
```

### Boolean Display

```yaml
meta:
  dimension:
    type: boolean
    label: "Active?"
    # Lightdash shows checkboxes for boolean filters
```

## Commit and Push to GitHub

After adding metrics to your dbt YAML files:

```sh
cd ~/projects/dbt/dbt-transform

# Check what changed
git status
# Should show: modified: models/marts/core/fct_exchange_rates.yml
#              modified: models/marts/core/dim_products.yml

# Review changes
git diff models/marts/core/fct_exchange_rates.yml

# Stage and commit
git add models/marts/core/
git commit -m "$(cat <<'EOF'
Add Lightdash metrics to mart models

- fct_exchange_rates: average, min, max, median, volatility metrics
- dim_products: average price, count metrics, custom SQL aggregations
- Hide surrogate keys and technical fields from Lightdash UI

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
EOF
)"

# Push to GitHub
git push origin main
```

## Sync Lightdash

### If Auto-Sync is Enabled

Lightdash automatically detects the push to `main` and recompiles the project. Check the Lightdash UI:

1. Navigate to **Settings** → **Project**
2. Look for "Last compiled" timestamp — should update shortly after the push
3. If compilation succeeds, metrics are immediately available

### If Auto-Sync is Disabled

Manually trigger a compile:

1. Navigate to **Settings** → **Project**
2. Click **Compile project**
3. Wait 30-90 seconds for compilation to complete

## Test Metrics in Lightdash

### Step 1: Navigate to Explore

1. Click **Explore** in the left sidebar
2. Select **Exchange Rates** (from the Finance group)

You should see:

**Dimensions:**
- Date
- Base Currency
- Target Currency
- Exchange Rate
- Data Source

**Metrics:**
- Average Exchange Rate
- Minimum Exchange Rate
- Maximum Exchange Rate
- Median Exchange Rate
- Exchange Rate Volatility
- Coefficient of Variation
- Number of Exchange Rates
- Number of Currencies

### Step 2: Build a Query

1. **Select dimensions**: `Date` (grouped by Month), `Target Currency`
2. **Select metrics**: `Average Exchange Rate`
3. **Add filter**: `Target Currency` IN (`USD`, `EUR`, `JPY`)
4. **Add filter**: `Date` > Last 6 months
5. Click **Run query**

Lightdash generates SQL:

```sql
SELECT
    DATE_TRUNC('month', rate_date) AS rate_date_month,
    target_currency,
    AVG(exchange_rate) AS average_exchange_rate
FROM analytics.reporting.fct_exchange_rates
WHERE base_currency = 'GBP'
    AND target_currency IN ('USD', 'EUR', 'JPY')
    AND rate_date >= DATEADD(month, -6, CURRENT_DATE())
GROUP BY 1, 2
ORDER BY 1 DESC, 2;
```

### Step 3: Visualise

1. Click **Chart**
2. Select **Line chart**
3. Configure:
   - **X-axis**: `Date (Month)`
   - **Y-axis**: `Average Exchange Rate`
   - **Group by**: `Target Currency`

You now have a line chart showing GBP exchange rate trends by currency over the last 6 months.

### Step 4: Save the Chart

1. Click **Save chart**
2. Name: "GBP Exchange Rates (6 Month Trend)"
3. Space: Create new space called "Exchange Rate Analysis"
4. Click **Save**

## Verify Metrics Work Correctly

### Test Average Calculation

Build a query:
- **Dimension**: `Target Currency` = `USD`
- **Metric**: `Average Exchange Rate`
- **Filter**: `Date` = Last 30 days

Compare the Lightdash result with a manual SQL query:

```sql
-- Manual verification
SELECT AVG(exchange_rate) AS avg_rate
FROM analytics.reporting.fct_exchange_rates
WHERE target_currency = 'USD'
    AND rate_date >= DATEADD(day, -30, CURRENT_DATE());
```

The results should match (within rounding).

### Test Custom SQL Metrics

Query the volatility metric:
- **Dimension**: `Target Currency`
- **Metric**: `Exchange Rate Volatility`
- **Filter**: `Date` = Last 90 days

Verify manually:

```sql
SELECT
    target_currency,
    STDDEV(exchange_rate) AS volatility
FROM analytics.reporting.fct_exchange_rates
WHERE rate_date >= DATEADD(day, -90, CURRENT_DATE())
GROUP BY target_currency
ORDER BY volatility DESC;
```

## Metrics as Code Workflow Summary

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   METRICS AS CODE BEST PRACTICES                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ✅ DO                                   ❌ DON'T                       │
│  ──────                                  ─────────                      │
│                                                                         │
│  • Define metrics in dbt YAML           • Define metrics in BI tool UI │
│  • Version control metric definitions   • Manually configure each BI   │
│  • Review metrics in pull requests      • Skip metric documentation    │
│  • Test metrics with dbt tests          • Use ambiguous metric names   │
│  • Document business logic              • Hard-code business logic in  │
│  • Use descriptive labels                 SQL without docs             │
│  • Hide technical fields                • Expose surrogate keys to     │
│                                            analysts                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Common Metric Patterns

### Count Metrics

```yaml
metrics:
  total_records:
    type: count
    label: "Total Records"

  distinct_values:
    type: count_distinct
    label: "Unique Values"
```

### Percentage Metrics

```yaml
metrics:
  percentage_active:
    type: number
    label: "% Active Products"
    sql: "(COUNT(CASE WHEN ${is_active} THEN 1 END) * 100.0) / COUNT(*)"
    format: percent
    round: 1
```

### Ratio Metrics

```yaml
metrics:
  price_to_average_ratio:
    type: number
    label: "Price vs Market Average"
    sql: "${product_price} / AVG(${product_price}) OVER ()"
    round: 2
```

### Conditional Aggregations

```yaml
metrics:
  revenue_usd_only:
    type: sum
    label: "Total Revenue (USD)"
    sql: "SUM(CASE WHEN ${currency} = 'USD' THEN ${amount} ELSE 0 END)"
    format: currency
    currency: USD
```

## Troubleshooting

### Metric Not Appearing in Lightdash

**Check:**
1. Did you recompile the project after adding the metric?
2. Is the YAML indentation correct? (Metrics must be nested under `columns.meta.metrics`)
3. Is the column referenced in the metric actually in the model?

**Debug:**
```yaml
# CORRECT indentation
columns:
  - name: exchange_rate
    meta:
      metrics:
        average_rate:
          type: average

# WRONG indentation (too far left)
columns:
  - name: exchange_rate
  meta:
    metrics:
      average_rate:
        type: average
```

### Metric Returns NULL

**Cause**: The metric aggregates a column with NULL values.

**Fix**: Add a filter or use `COALESCE`:

```yaml
metrics:
  average_non_null:
    type: number
    sql: "AVG(COALESCE(${column_name}, 0))"
```

### Custom SQL Metric Has Syntax Error

**Cause**: Invalid SQL in the `sql` field.

**Debug**: Copy the SQL from the error message and run it manually in Snowflake to identify the issue.

## Summary

You've added metrics to your dbt models:

- [x] **Updated `fct_exchange_rates.yml`** with average, min, max, median, and volatility metrics
- [x] **Updated `dim_products.yml`** with price aggregations and count metrics
- [x] **Configured formatting** (currency, percentages, rounding) for optimal display
- [x] **Committed and pushed** metrics to GitHub
- [x] **Synced Lightdash** to discover new metrics
- [x] **Tested metrics** in Lightdash Explore and verified results
- [x] **Understood metrics as code workflow** — Git is the source of truth

Your metrics are now version-controlled, testable, and automatically synced to Lightdash. Changes to business logic happen in dbt, not in a BI tool UI.

## What's Next

Now that metrics are defined, build dashboards to visualise exchange rates and product data for business users.

Continue to [Build Dashboards](10-build-dashboards.md) →
