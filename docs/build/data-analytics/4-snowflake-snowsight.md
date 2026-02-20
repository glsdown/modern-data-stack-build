# Snowflake Snowsight

On this page, you will:

- [x] Understand Snowsight's built-in dashboarding and visualisation capabilities
- [x] Create SQL worksheets and basic charts from your dbt models
- [x] Build a simple dashboard with exchange rates and product data
- [x] Learn when to use Snowsight versus dedicated BI tools
- [x] Explore Snowflake Notebooks for Python-based analysis

## Overview

Snowsight is Snowflake's modern web interface, replacing the classic Snowflake UI. It includes SQL worksheets, basic visualisations, and dashboards — all built into your Snowflake account at no additional cost.

This makes Snowsight the fastest way to visualise your dbt models. There's zero setup: you already have access, and it queries your warehouse natively with no data movement.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       SNOWSIGHT ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  dbt Models (ANALYTICS)       Snowsight                                 │
│  ──────────────────────       ─────────                                 │
│                                                                         │
│  ┌──────────────────┐         ┌──────────────────┐                     │
│  │ MARTS            │         │ SQL Worksheets   │                     │
│  │ • fct_exchange_  │────────▶│ • Query models   │                     │
│  │   rates          │         │ • Basic charts   │                     │
│  │ • dim_products   │         └──────────────────┘                     │
│  └──────────────────┘                  │                               │
│                                        ▼                                │
│  ┌──────────────────┐         ┌──────────────────┐                     │
│  │ REPORTING        │         │ Dashboards       │                     │
│  │ (BI-ready)       │────────▶│ • Exchange rates │                     │
│  └──────────────────┘         │ • Products       │                     │
│                               └──────────────────┘                     │
│                                                                         │
│  No additional infrastructure. Runs in Snowflake.                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## What Is Snowsight?

Snowsight is Snowflake's unified interface for:

1. **SQL Worksheets** — write and execute SQL queries
2. **Visualisations** — create charts (line, bar, scatter, heatmap) from query results
3. **Dashboards** — combine multiple charts into dashboards
4. **Snowflake Notebooks** — Python notebooks (similar to Jupyter) with Snowpark
5. **Data exploration** — browse databases, schemas, tables, and views
6. **Query history** — review past queries and performance

All of this is included with your Snowflake account. There's no additional licence fee — you pay only for warehouse compute when queries run.

!!! info "Snowsight vs Classic UI"
    Snowsight replaced the older "Classic Console". If you're still seeing the classic interface, switch to Snowsight by clicking "Try Snowsight" in the top-right corner.

## Accessing Snowsight

Navigate to your Snowflake account URL:

```
https://<your-account>.snowflakecomputing.com
```

Log in with your Snowflake credentials. You'll land on the Snowsight home page showing recent queries, worksheets, and dashboards.

!!! tip "Bookmark Snowsight"
    Add Snowsight to your bookmarks for quick access. Analysts should have Snowsight open alongside their dbt editor for testing queries.

## Creating SQL Worksheets

### Step 1: Create a New Worksheet

1. Click **+ Worksheet** in the top-right corner
2. Select your role: `ANALYTICS_DEVELOPER` or `ANALYTICS_REPORTER` (depending on your access)
3. Select your warehouse: `TRANSFORMING` or `REPORTING` (use `REPORTING` for read-only queries to save costs)
4. Select your database: `ANALYTICS`
5. Select your schema: `MARTS` or `REPORTING`

### Step 2: Query Your dbt Models

Write a query against one of your dbt models:

```sql
-- Query the exchange rates fact table
SELECT
    rate_date,
    base_currency,
    target_currency,
    exchange_rate
FROM analytics.marts.fct_exchange_rates
WHERE base_currency = 'GBP'
    AND rate_date >= DATEADD(day, -30, CURRENT_DATE())
ORDER BY rate_date DESC, target_currency;
```

Click **Run** (or press `Cmd+Enter` / `Ctrl+Enter`).

Results appear in the **Results** pane below your query.

### Step 3: Save the Worksheet

Click the three-dot menu (⋮) at the top-right → **Save As**:

- **Name**: `Exchange Rates - GBP Last 30 Days`
- **Folder**: Create a folder called `Analytics` to organise worksheets

Saved worksheets appear in the left sidebar under **Worksheets**.

!!! tip "Keyboard Shortcuts"
    - `Cmd+Enter` (Mac) / `Ctrl+Enter` (Windows): Run query
    - `Cmd+S` / `Ctrl+S`: Save worksheet
    - `Cmd+/` / `Ctrl+/`: Comment/uncomment lines

## Creating Visualisations

### Step 1: Query Data for a Chart

Write a query that aggregates data for visualisation:

```sql
-- Exchange rate trends for GBP → USD, EUR over the last 90 days
SELECT
    rate_date,
    target_currency,
    exchange_rate
FROM analytics.marts.fct_exchange_rates
WHERE base_currency = 'GBP'
    AND target_currency IN ('USD', 'EUR')
    AND rate_date >= DATEADD(day, -90, CURRENT_DATE())
ORDER BY rate_date, target_currency;
```

Run the query.

### Step 2: Create a Chart

1. Click the **Chart** button at the top of the Results pane
2. Snowsight auto-detects a reasonable chart type (usually a line chart for time-series data)
3. Configure the chart:
   - **X-axis**: `RATE_DATE`
   - **Y-axis**: `EXCHANGE_RATE`
   - **Group by**: `TARGET_CURRENCY` (this creates separate lines for USD and EUR)
   - **Chart type**: Line chart

4. Customise:
   - **Title**: "GBP Exchange Rates (Last 90 Days)"
   - **Axis labels**: X = "Date", Y = "Exchange Rate"
   - **Legend**: Show (to distinguish USD vs EUR lines)

### Step 3: Save the Chart

The chart is tied to the worksheet. When you save the worksheet, the chart is saved with it.

!!! info "Chart Types in Snowsight"
    Snowsight supports:
    - **Line charts** (time-series trends)
    - **Bar charts** (comparisons)
    - **Scatter plots** (correlations)
    - **Heatmaps** (density, patterns)
    - **Scorecards** (single metric, large number)

    This is more limited than Tableau or Metabase, but sufficient for most analytical needs.

## Building a Dashboard

Dashboards combine multiple charts into a single view. They can be shared with stakeholders who don't write SQL.

### Step 1: Create Dashboard Worksheets

Create separate worksheets for each chart you want on the dashboard:

**Worksheet 1: Exchange Rates Trend**
```sql
SELECT
    rate_date,
    target_currency,
    exchange_rate
FROM analytics.marts.fct_exchange_rates
WHERE base_currency = 'GBP'
    AND target_currency IN ('USD', 'EUR', 'JPY')
    AND rate_date >= DATEADD(day, -90, CURRENT_DATE())
ORDER BY rate_date, target_currency;
```

Create a line chart (as above).

**Worksheet 2: Latest Exchange Rates**
```sql
SELECT
    target_currency,
    exchange_rate,
    rate_date
FROM analytics.marts.fct_exchange_rates
WHERE base_currency = 'GBP'
    AND rate_date = (SELECT MAX(rate_date) FROM analytics.marts.fct_exchange_rates)
ORDER BY target_currency;
```

Create a bar chart showing current rates for all currencies.

**Worksheet 3: Product Count by Category**
```sql
SELECT
    product_category,
    COUNT(*) AS product_count
FROM analytics.marts.dim_products
GROUP BY product_category
ORDER BY product_count DESC;
```

Create a bar chart.

### Step 2: Create the Dashboard

1. Navigate to **Dashboards** in the left sidebar
2. Click **+ Dashboard**
3. Name it: `Analytics Overview`
4. Click **+ Tile** to add charts
5. Select **From Worksheet** → choose your saved worksheets
6. Add each chart as a tile
7. Arrange tiles by dragging them
8. Resize tiles to fit

### Step 3: Add Filters (Optional)

Dashboards can have filters that apply to multiple tiles:

1. Click **+ Filter** at the top of the dashboard
2. Choose a column (e.g., `TARGET_CURRENCY`)
3. Select filter type (dropdown, multi-select, date range)
4. Apply the filter to relevant tiles

Viewers can now adjust filters without editing SQL.

### Step 4: Share the Dashboard

1. Click the **Share** button at the top-right
2. Select users or roles to share with:
   - **Role-based**: Share with `ANALYTICS_REPORTER` (all users with that role can view)
   - **User-based**: Share with specific Snowflake users
3. Set permissions:
   - **View**: Can view dashboard but not edit
   - **Edit**: Can modify dashboard

!!! tip "Role-Based Sharing"
    Share dashboards with roles (`ANALYTICS_REPORTER`) rather than individual users. This scales better as your team grows.

## Snowflake Notebooks (Python Analysis)

Snowsight includes **Snowflake Notebooks** — Python notebooks similar to Jupyter, integrated directly into Snowflake.

### When to Use Notebooks vs Dashboards

| Use Case | Tool |
|----------|------|
| **Exploratory analysis** — investigate new data, test hypotheses | Notebooks |
| **Data science / ML** — build models, feature engineering | Notebooks |
| **Ad-hoc questions** — "Why did metric X spike last week?" | Notebooks |
| **Operational reporting** — daily/weekly KPI dashboards | Dashboards (Snowsight or BI tool) |
| **Executive dashboards** — polished, filtered, scheduled | BI tool (Lightdash, Tableau) |

### Creating a Snowflake Notebook

1. Navigate to **Notebooks** in the left sidebar
2. Click **+ Notebook**
3. Select your role, warehouse, and database
4. Choose a Python runtime (Snowpark Python)

**Example notebook cell:**

```python
# Import Snowpark
from snowflake.snowpark import Session

# Query exchange rates using Snowpark
df = session.table("ANALYTICS.MARTS.FCT_EXCHANGE_RATES") \
    .filter(col("BASE_CURRENCY") == "GBP") \
    .filter(col("RATE_DATE") >= dateadd("day", -90, current_date())) \
    .select("RATE_DATE", "TARGET_CURRENCY", "EXCHANGE_RATE")

# Display as Pandas DataFrame
df.to_pandas()
```

Snowflake Notebooks can:
- Query Snowflake tables with Python (Snowpark API)
- Use pandas, numpy, matplotlib, seaborn for analysis
- Visualise data with Python plotting libraries
- Train ML models (scikit-learn, XGBoost)

**Limitations:**
- Compute runs in Snowflake warehouse (not local)
- Limited library support (not all PyPI packages available)
- Slower than local Jupyter for rapid iteration

!!! info "Notebooks vs Worksheets"
    - **Worksheets**: SQL-only, fast, great for querying and basic charts
    - **Notebooks**: Python + SQL, slower, great for data science and exploratory analysis

For more advanced Python work, consider [Jupyter with Snowflake connector](11-notebooks.md) or [Hex](11-notebooks.md).

## Example: Exchange Rates Dashboard

Let's build a complete exchange rates dashboard in Snowsight.

### Worksheet 1: GBP Exchange Rate Trends

```sql
-- GBP to major currencies over time
SELECT
    rate_date,
    target_currency,
    exchange_rate
FROM analytics.marts.fct_exchange_rates
WHERE base_currency = 'GBP'
    AND target_currency IN ('USD', 'EUR', 'JPY', 'AUD', 'CAD')
    AND rate_date >= DATEADD(month, -6, CURRENT_DATE())
ORDER BY rate_date, target_currency;
```

**Chart type**: Line chart
- X-axis: `RATE_DATE`
- Y-axis: `EXCHANGE_RATE`
- Group by: `TARGET_CURRENCY`
- Title: "GBP Exchange Rates (6 Months)"

### Worksheet 2: Currency Volatility

```sql
-- Calculate 30-day volatility (standard deviation of daily rates)
SELECT
    target_currency,
    STDDEV(exchange_rate) AS volatility,
    AVG(exchange_rate) AS avg_rate,
    MIN(exchange_rate) AS min_rate,
    MAX(exchange_rate) AS max_rate
FROM analytics.marts.fct_exchange_rates
WHERE base_currency = 'GBP'
    AND rate_date >= DATEADD(day, -30, CURRENT_DATE())
GROUP BY target_currency
ORDER BY volatility DESC;
```

**Chart type**: Bar chart
- X-axis: `TARGET_CURRENCY`
- Y-axis: `VOLATILITY`
- Title: "Currency Volatility (30 Days)"

### Worksheet 3: Latest Rates Summary

```sql
-- Latest exchange rates with previous day comparison
WITH latest AS (
    SELECT MAX(rate_date) AS max_date
    FROM analytics.marts.fct_exchange_rates
),
current_rates AS (
    SELECT
        target_currency,
        exchange_rate AS current_rate,
        rate_date
    FROM analytics.marts.fct_exchange_rates
    CROSS JOIN latest
    WHERE base_currency = 'GBP'
        AND rate_date = max_date
),
previous_rates AS (
    SELECT
        target_currency,
        exchange_rate AS previous_rate
    FROM analytics.marts.fct_exchange_rates
    CROSS JOIN latest
    WHERE base_currency = 'GBP'
        AND rate_date = DATEADD(day, -1, max_date)
)
SELECT
    c.target_currency,
    c.current_rate,
    p.previous_rate,
    ((c.current_rate - p.previous_rate) / p.previous_rate) * 100 AS change_percent
FROM current_rates c
LEFT JOIN previous_rates p ON c.target_currency = p.target_currency
ORDER BY ABS(change_percent) DESC;
```

**Chart type**: Table (or bar chart for `CHANGE_PERCENT`)
- Title: "Latest Exchange Rates vs Previous Day"

### Create the Dashboard

1. Save all three worksheets
2. Create a new dashboard: "GBP Exchange Rate Monitoring"
3. Add tiles from each worksheet
4. Arrange in a grid (trend chart on top, volatility and summary below)
5. Share with `ANALYTICS_REPORTER` role

## When to Use Snowsight vs Dedicated BI Tools

### Use Snowsight When:

✅ **Your users are SQL-proficient**
- Data analysts, analytics engineers, data engineers
- Comfortable writing SQL queries

✅ **You need quick, ad-hoc analysis**
- Investigating data quality issues
- Testing queries before adding to dbt
- Exploring new data sources

✅ **Budget is $0**
- No additional BI tool cost is acceptable
- Team is small and technical

✅ **You want basic dashboards**
- Line charts, bar charts, tables
- No advanced visualisations needed

### Use a Dedicated BI Tool (Lightdash, Tableau) When:

✅ **Non-technical business users need dashboards**
- Executives, managers, marketing team
- Users who don't write SQL

✅ **You need self-service analytics**
- Business users building their own reports
- Drag-and-drop interface required

✅ **You want metrics as code** (dbt-native tools)
- Metrics version-controlled in dbt YAML
- Lightdash or Omni integration

✅ **You need advanced visualisations**
- Custom charts, geographic maps, complex interactivity
- Tableau, Looker, Superset

✅ **You need polished, scheduled dashboards**
- Automated daily/weekly email reports
- Embedded dashboards in other applications

!!! success "Hybrid Approach"
    Most teams use both:
    - **Snowsight** for analysts doing SQL-based exploration
    - **Lightdash/Tableau** for business users consuming polished dashboards

## Limitations of Snowsight

Be aware of these limitations compared to dedicated BI tools:

| Feature | Snowsight | Dedicated BI Tools |
|---------|----------|-------------------|
| **No-code interface** | ❌ SQL required | ✅ Drag-and-drop |
| **Metrics as code** | ❌ Queries in UI | ✅ dbt YAML (Lightdash/Omni) |
| **Advanced visualisations** | ❌ Basic charts only | ✅ Custom charts, maps, etc. |
| **Scheduled email reports** | ❌ No native scheduling | ✅ Yes |
| **Embedded dashboards** | ❌ No embedding | ✅ Yes (Tableau, Looker) |
| **Version control** | ❌ No Git integration | ✅ Dashboards as code (Lightdash) |
| **Collaboration** | ⚠️ Basic sharing | ✅ Comments, annotations, alerts |
| **Mobile app** | ⚠️ Basic mobile web | ✅ Native apps (Tableau) |

For most teams, Snowsight is a **starting point**, not the final destination. Use it for early dashboards while evaluating Lightdash or other BI tools.

## Performance and Cost Considerations

### Warehouse Compute Costs

Every query in Snowsight runs in a Snowflake warehouse. Compute costs are based on:
- **Warehouse size** (X-Small, Small, Medium, etc.)
- **Query duration** (billed per-second, 60-second minimum)
- **Warehouse auto-suspend** (pause when idle)

**Best practices:**
1. Use `REPORTING` warehouse (X-Small) for dashboard queries
2. Set auto-suspend to 60 seconds (queries often come in bursts)
3. Avoid `SELECT *` — query only the columns you need
4. Use dbt models (pre-aggregated) instead of querying raw tables

**Example costs:**
- `REPORTING` warehouse (X-Small): $2/hour = $0.033/minute
- Typical dashboard query: 2-5 seconds = $0.0011-0.0028 per query
- 100 dashboard views/day: ~$0.20/day = **$6/month** in compute

This is negligible compared to BI tool subscription costs.

### Query Result Caching

Snowflake caches query results for 24 hours. If the same query runs again (same SQL, same data), results are returned instantly from cache at no compute cost.

This makes dashboards with repeated queries (refreshed hourly) very cost-effective.

## Summary

You've learned how to use Snowsight for analytics:

- [x] **Snowsight is free** — included with Snowflake, zero additional cost (only warehouse compute)
- [x] **SQL worksheets** let you query dbt models and create basic charts
- [x] **Dashboards** combine multiple charts and can be shared with roles or users
- [x] **Snowflake Notebooks** provide Python analysis with Snowpark
- [x] **Limitations**: Requires SQL, basic visualisations only, no metrics as code
- [x] **Best for**: SQL-proficient analysts, ad-hoc exploration, budget-conscious teams
- [x] **Upgrade to dedicated BI tools** when you need self-service for non-technical users or metrics as code

Snowsight is an excellent quick win. Build a few dashboards here to demonstrate value, then decide whether to invest in Lightdash or another BI tool.

## What's Next

Now that you've built quick dashboards in Snowsight, set up the Snowflake infrastructure for Lightdash to enable dbt-native, metrics-as-code analytics.

Continue to [Snowflake Infrastructure](5-snowflake-infrastructure.md) →
