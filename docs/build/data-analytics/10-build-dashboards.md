# Build Dashboards

On this page, you will:

- [x] Create an exchange rates dashboard with trend charts and volatility analysis
- [x] Build a product catalogue dashboard with pricing insights
- [x] Configure filters, drill-downs, and interactive elements
- [x] Schedule dashboard refreshes and email reports
- [x] Share dashboards with stakeholders and set permissions
- [x] Understand dashboard best practices for clarity and performance

## Overview

Dashboards in Lightdash combine multiple charts into a single view, providing at-a-glance insights into your data. They can be filtered, scheduled, and shared with stakeholders who don't write SQL.

This page walks through building two dashboards using the metrics you defined in the previous step:

1. **Exchange Rates Dashboard** — GBP currency trends, volatility, and comparisons
2. **Product Catalogue Dashboard** — Product pricing analysis and category breakdown

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        DASHBOARD ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Dashboard                         dbt Models                          │
│  ─────────                         ──────────                          │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────┐      │
│  │ Exchange Rates Dashboard                                     │      │
│  │ ┌────────────────┐  ┌────────────────┐  ┌────────────────┐  │      │
│  │ │ Trend Chart    │  │ Volatility     │  │ Current Rates  │  │      │
│  │ │ (line chart)   │  │ (bar chart)    │  │ (table)        │  │      │
│  │ └────────────────┘  └────────────────┘  └────────────────┘  │      │
│  │         ▲                   ▲                   ▲           │      │
│  │         └───────────────────┴───────────────────┘           │      │
│  │                             │                               │      │
│  │                   fct_exchange_rates                        │      │
│  │                   (metrics from YAML)                       │      │
│  └──────────────────────────────────────────────────────────────┘      │
│                                                                         │
│  All charts query the same dbt model with different filters/groupings. │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Dashboard 1: Exchange Rates Dashboard

### Step 1: Create the Dashboard

1. Navigate to **Dashboards** in Lightdash
2. Click **+ New dashboard**
3. Name: "GBP Exchange Rates Dashboard"
4. Description: "Daily GBP exchange rate trends, volatility analysis, and current rates"
5. Click **Create**

### Step 2: Add Chart 1 — Exchange Rate Trends

**Create the chart:**

1. Click **+ Add tile** → **Saved chart** (or **Create new**)
2. If creating new:
   - Select **Explore** → **Exchange Rates** (`fct_exchange_rates`)
   - **Dimensions**: `Date` (grouped by Week), `Target Currency`
   - **Metrics**: `Average Exchange Rate`
   - **Filters**:
     - `Date` >= Last 6 months
     - `Target Currency` IN (`USD`, `EUR`, `JPY`, `AUD`, `CAD`)
   - Click **Run query**

3. **Configure chart**:
   - Chart type: **Line chart**
   - X-axis: `Date (Week)`
   - Y-axis: `Average Exchange Rate`
   - Group by / Series: `Target Currency`
   - Title: "GBP Exchange Rate Trends (6 Months)"

4. **Customise**:
   - Show legend: Yes
   - Y-axis label: "Exchange Rate"
   - X-axis label: "Week"
   - Colour scheme: Assign distinct colours to each currency

5. Click **Save** → Choose the dashboard you created above

**Resize and position:**
- Drag the tile to the top-left
- Resize to span 2/3 of the dashboard width

### Step 3: Add Chart 2 — Currency Volatility

**Create the chart:**

1. Click **+ Add tile** → **Create new**
2. **Explore** → **Exchange Rates**
3. **Dimensions**: `Target Currency`
4. **Metrics**: `Exchange Rate Volatility`
5. **Filters**:
   - `Date` >= Last 90 days
   - `Target Currency` IN (`USD`, `EUR`, `JPY`, `AUD`, `CAD`, `CHF`, `NZD`, `SEK`)
6. **Sort**: `Exchange Rate Volatility` DESC

**Configure chart:**
- Chart type: **Bar chart** (horizontal)
- X-axis: `Exchange Rate Volatility`
- Y-axis: `Target Currency`
- Title: "Currency Volatility (90 Days)"
- Subtitle: "Standard deviation of exchange rates"

**Save and position:**
- Save to the dashboard
- Position in top-right (1/3 width)

### Step 4: Add Chart 3 — Current Exchange Rates Table

**Create the chart:**

1. **Explore** → **Exchange Rates**
2. **Dimensions**: `Target Currency`, `Date`
3. **Metrics**: `Average Exchange Rate`
4. **Filters**:
   - `Date` = Latest date available (use custom SQL: `rate_date = (SELECT MAX(rate_date) FROM analytics.reporting.fct_exchange_rates)`)
   - Alternatively: `Date` >= Today - 1 day (if data is updated daily)
5. **Sort**: `Target Currency` ASC

**Configure:**
- Chart type: **Table**
- Columns: `Target Currency`, `Average Exchange Rate`
- Column formats:
  - `Average Exchange Rate`: 4 decimal places
- Title: "Current GBP Exchange Rates"
- Subtitle: "As of [latest date]"

**Save and position:**
- Save to dashboard
- Position below the trend chart (full width or left half)

### Step 5: Add Chart 4 — Min/Max Range

**Create the chart:**

1. **Explore** → **Exchange Rates**
2. **Dimensions**: `Target Currency`
3. **Metrics**: `Minimum Exchange Rate`, `Maximum Exchange Rate`, `Average Exchange Rate`
4. **Filters**:
   - `Date` >= Last 90 days
   - `Target Currency` IN (`USD`, `EUR`, `JPY`)

**Configure:**
- Chart type: **Bar chart** (grouped)
- X-axis: `Target Currency`
- Y-axis: All three metrics (min, max, average) as separate series
- Title: "90-Day Exchange Rate Range"
- Subtitle: "Min, Max, and Average rates"

**Save and position:**
- Save to dashboard
- Position bottom-right

### Step 6: Add Dashboard Filter

Add a filter that applies to all tiles:

1. Click **Edit dashboard**
2. Click **+ Add filter**
3. Select **Target Currency** (from `fct_exchange_rates`)
4. Filter type: **Multi-select dropdown**
5. Default value: `USD`, `EUR`, `JPY` (pre-selected)
6. Apply to tiles: Select all tiles that use `Target Currency`

Now viewers can adjust the currency selection and all charts update automatically.

### Step 7: Add Text Tile (Optional)

Add context for dashboard users:

1. Click **+ Add tile** → **Markdown**
2. Add explanatory text:

```markdown
## GBP Exchange Rates Dashboard

This dashboard shows GBP (British Pound) exchange rates against major currencies.

**Data Sources:**
- European Central Bank (ECB) API
- Yahoo Finance API

**Refresh Schedule:** Daily at 08:00 UTC

**Key Metrics:**
- **Volatility**: Standard deviation of daily rates (higher = more volatile)
- **Current Rates**: Latest available exchange rate
- **Trends**: 6-month historical trends by week
```

3. Save and position at the top of the dashboard

## Dashboard 2: Product Catalogue Dashboard

### Step 1: Create the Dashboard

1. Navigate to **Dashboards**
2. Click **+ New dashboard**
3. Name: "Product Catalogue Dashboard"
4. Description: "Product pricing, categories, and inventory insights"
5. Click **Create**

### Step 2: Add Chart 1 — Product Count by Category

**Create the chart:**

1. **Explore** → **Products** (`dim_products`)
2. **Dimensions**: `Category`
3. **Metrics**: `count_current_products` (or row count if metric not defined)
4. **Filters**:
   - `Current Version` = TRUE (to avoid counting historical Type 2 SCD versions)
   - `Active Product` = TRUE
5. **Sort**: Count DESC

**Configure:**
- Chart type: **Bar chart**
- X-axis: `Category`
- Y-axis: `Count of Products`
- Title: "Product Count by Category"
- Subtitle: "Active products only"

### Step 3: Add Chart 2 — Average Price by Category

**Create the chart:**

1. **Explore** → **Products**
2. **Dimensions**: `Category`
3. **Metrics**: `Average Product Price`
4. **Filters**:
   - `Current Version` = TRUE
   - `Active Product` = TRUE

**Configure:**
- Chart type: **Bar chart** (horizontal)
- X-axis: `Average Product Price`
- Y-axis: `Category`
- Title: "Average Product Price by Category"
- Format: Currency (GBP), 2 decimal places

### Step 4: Add Chart 3 — Price Distribution

**Create the chart:**

1. **Explore** → **Products**
2. **Dimensions**: `Product Price` (bucketed)
3. **Metrics**: Count of products
4. **Filters**:
   - `Current Version` = TRUE
   - `Active Product` = TRUE

**Note:** Lightdash doesn't have built-in histogram support. Instead, create price buckets manually or use a table:

**Alternative: Table of Price Ranges**
- Show `Minimum Price`, `Maximum Price`, `Average Price` by category

**Configure:**
- Chart type: **Table** or **Scatter plot** (Price vs Category)
- Title: "Product Price Distribution"

### Step 5: Add Chart 4 — Most Expensive Products

**Create the chart:**

1. **Explore** → **Products**
2. **Dimensions**: `Product Name`, `Category`, `Product Price`
3. **Metrics**: None (showing raw data)
4. **Filters**:
   - `Current Version` = TRUE
   - `Active Product` = TRUE
5. **Sort**: `Product Price` DESC
6. **Limit**: Top 10

**Configure:**
- Chart type: **Table**
- Columns: `Product Name`, `Category`, `Product Price`
- Title: "Top 10 Most Expensive Products"

### Step 6: Add Chart 5 — Active vs Inactive Products

**Create the chart:**

1. **Explore** → **Products**
2. **Dimensions**: `Active Product`
3. **Metrics**: Count
4. **Filters**:
   - `Current Version` = TRUE

**Configure:**
- Chart type: **Pie chart** or **Donut chart**
- Segments: `Active Product` (True/False)
- Title: "Active vs Inactive Products"

### Step 7: Add Dashboard Filter

1. Click **Edit dashboard**
2. Click **+ Add filter**
3. Select **Category**
4. Filter type: **Multi-select dropdown**
5. Default value: All categories
6. Apply to all tiles

## Scheduling and Sharing

### Schedule Dashboard Refresh

Dashboards in Lightdash query live data — they're always up to date when you view them. However, you can schedule **email reports**:

1. Open the dashboard
2. Click **Schedule** (or **⋮** menu → **Schedule delivery**)
3. Configure:
   - **Recipients**: Enter email addresses
   - **Frequency**: Daily, Weekly, Monthly
   - **Time**: 08:00 UTC
   - **Format**: PDF or link to dashboard
4. Click **Save schedule**

Recipients will receive an email with the dashboard snapshot or a link to view it in Lightdash.

### Share Dashboard

**Share with specific users:**

1. Click **Share** at the top-right
2. Enter usernames or emails
3. Set permissions:
   - **View**: Can view dashboard but not edit
   - **Edit**: Can modify tiles and filters
4. Click **Share**

**Share with a role (recommended):**

If you've configured roles in Lightdash (mirroring Snowflake roles):
1. Click **Share**
2. Select **Share with role** → `ANALYTICS_REPORTER` or equivalent
3. All users with that role can view the dashboard

**Public link (not recommended for production):**
- Lightdash Cloud supports public links for embedding
- Self-hosted: Configure authentication if needed

## Dashboard Best Practices

### Design Principles

1. **Start with key metrics at the top**
   - Most important chart (trend, KPI) should be top-left (first thing users see)

2. **Limit to 4-8 tiles**
   - Too many tiles overwhelm users
   - Create multiple dashboards instead of one crowded dashboard

3. **Use consistent colour schemes**
   - Same currency = same colour across charts
   - Follow brand guidelines if applicable

4. **Add context with text tiles**
   - Explain what the dashboard shows
   - Link to documentation or data dictionaries

5. **Enable filters for interactivity**
   - Let users explore (e.g., filter by date range, category, currency)
   - But don't overload with 10+ filters

### Performance Optimisation

1. **Limit data volume**
   - Use filters to reduce rows queried (e.g., last 6 months, not all-time)
   - Aggregate in dbt models, not in Lightdash

2. **Materialise as tables, not views**
   - `fct_exchange_rates` should be a `table` in dbt (not a view)
   - Views query raw data on every dashboard load (slow)

3. **Use Snowflake caching**
   - Snowflake caches identical queries for 24 hours
   - Dashboards with repeated queries (hourly refresh) benefit from caching

4. **Pre-aggregate in dbt**
   - Create a `fct_exchange_rates_monthly` model if you always group by month
   - Queries run faster on pre-aggregated tables

### Naming Conventions

- **Dashboard names**: Clear, action-oriented ("GBP Exchange Rate Monitoring", not "Dashboard 1")
- **Chart titles**: Descriptive and specific ("GBP/USD Trend - Last 6 Months", not "Line Chart")
- **Metric labels**: Business-friendly ("Average Exchange Rate", not "avg_rate")

## Example Dashboard Layout

```
┌────────────────────────────────────────────────────────────────────┐
│ GBP Exchange Rates Dashboard                                      │
├────────────────────────────────────────────────────────────────────┤
│ ┌────────────────────────────────────────────────┐                 │
│ │ Explanatory Text (Markdown)                   │                 │
│ │ • Data sources: ECB, Yahoo Finance             │                 │
│ │ • Refresh: Daily at 08:00 UTC                  │                 │
│ └────────────────────────────────────────────────┘                 │
│                                                                    │
│ Filters: [Currency ▼] [Date Range ▼]                              │
│                                                                    │
│ ┌───────────────────────────────────┐  ┌────────────────────────┐ │
│ │ Exchange Rate Trends (Line Chart) │  │ Volatility (Bar Chart) │ │
│ │                                   │  │                        │ │
│ │ (2/3 width)                       │  │ (1/3 width)            │ │
│ └───────────────────────────────────┘  └────────────────────────┘ │
│                                                                    │
│ ┌───────────────────────────────────┐  ┌────────────────────────┐ │
│ │ Current Rates (Table)             │  │ Min/Max Range          │ │
│ │                                   │  │ (Bar Chart)            │ │
│ │ (1/2 width)                       │  │ (1/2 width)            │ │
│ └───────────────────────────────────┘  └────────────────────────┘ │
└────────────────────────────────────────────────────────────────────┘
```

## Embedding Dashboards (Optional)

Lightdash Cloud supports embedding dashboards in external applications (e.g., internal company portal):

1. Navigate to the dashboard
2. Click **⋮** menu → **Share** → **Embed**
3. Copy the `<iframe>` code
4. Paste into your application

**Requirements:**
- Lightdash Cloud only (not self-hosted, unless you configure authentication)
- Users must have Lightdash accounts (SSO recommended)
- Or use public links (not recommended for sensitive data)

## Troubleshooting

### Dashboard Loads Slowly

**Diagnose:**
1. Open the dashboard
2. Click on a slow tile
3. View the query and run it manually in Snowsight
4. Check query execution time

**Fixes:**
- Add indexes/clustering keys to the Snowflake table
- Reduce date range (e.g., 6 months instead of all-time)
- Pre-aggregate data in dbt (create a monthly rollup model)
- Materialise as `table` instead of `view`

### Chart Shows "No Data"

**Check:**
1. Are filters too restrictive? (e.g., date range excludes all data)
2. Does the underlying model have data? Query it directly in Snowsight
3. Did dbt run successfully? Check `REPORTING` schema for the table

### Filters Don't Apply to All Tiles

**Fix:**
1. Edit the dashboard
2. Click the filter
3. Ensure **"Apply to tiles"** includes all relevant tiles
4. Save changes

## Summary

You've built dashboards in Lightdash:

- [x] **Created GBP Exchange Rates Dashboard** with trend charts, volatility analysis, and current rates
- [x] **Created Product Catalogue Dashboard** with pricing analysis and category breakdowns
- [x] **Added interactive filters** for currency and category selection
- [x] **Configured scheduled email reports** for stakeholders
- [x] **Shared dashboards** with appropriate roles and permissions
- [x] **Applied best practices** for clarity, performance, and usability

Your dashboards are now live, querying dbt models with metrics defined in code. Business users can explore data without writing SQL.

## What's Next

Dashboards serve operational reporting. For deeper, exploratory analysis, use notebooks (Jupyter, Snowflake Notebooks) to investigate data with Python and SQL.

Continue to [Ad-Hoc Analytics with Notebooks](11-notebooks.md) →
