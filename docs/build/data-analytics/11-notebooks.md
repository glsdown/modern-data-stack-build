# Ad-Hoc Analytics with Notebooks

On this page, you will:

- [x] Understand when to use notebooks versus dashboards
- [x] Set up Snowflake Notebooks (built-in, free, Python-based)
- [x] Configure Jupyter locally with Snowflake connector
- [x] Learn how to share notebooks using jupytext for Git version control
- [x] Build an example ML notebook to predict GBP/USD exchange rates
- [x] Explore Hex as a premium collaborative notebook platform

## Overview

While dashboards answer **known questions** ("What is our revenue this month?"), notebooks help answer **new questions** ("Why did metric X spike last week?" or "Can we predict next month's exchange rate?").

Notebooks combine code (Python, SQL, R), visualisations, and narrative documentation in a single interactive environment. They're essential for:
- **Exploratory data analysis** — investigating new data sources
- **Root cause analysis** — debugging anomalies in metrics
- **Machine learning** — building and testing models
- **Custom analyses** — one-off requests from stakeholders
- **Data science workflows** — feature engineering, model training, evaluation

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       DASHBOARDS VS NOTEBOOKS                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Dashboards                           Notebooks                         │
│  ──────────                           ─────────                         │
│                                                                         │
│  • Answer known questions             • Answer NEW questions            │
│  • For business users                 • For analysts, data scientists   │
│  • Pre-built visualisations           • Code-first exploration          │
│  • Scheduled, repeatable              • Ad-hoc, iterative              │
│  • Examples:                          • Examples:                       │
│    - KPI dashboard                      - "Why did conversions drop?"  │
│    - Sales pipeline report              - "Predict next month revenue" │
│                                         - "Cluster customer segments"   │
│                                                                         │
│  You need BOTH for a complete analytics stack.                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Option 1: Snowflake Notebooks (Built-In, Free)

Snowflake Notebooks are Python notebooks built directly into Snowsight. They run in Snowflake's infrastructure using Snowpark.

### Pros and Cons

**Pros:**
- **Zero setup** — already included with Snowflake
- **Native Snowflake integration** — query tables directly with Snowpark
- **Free** (uses warehouse compute, same as SQL queries)
- **Shareable** — share notebooks with other Snowflake users
- **Version history** — built-in versioning

**Cons:**
- **Python only** — no R, Julia, or other languages
- **Limited libraries** — not all PyPI packages available
- **Slower iteration** — compute runs in Snowflake warehouse (not local)
- **No Git integration** — notebooks stored in Snowflake (not version-controlled in Git)

**Best for:** Quick Python analysis on Snowflake data without local setup.

### Creating a Snowflake Notebook

1. Navigate to **Snowsight** → **Notebooks** in the left sidebar
2. Click **+ Notebook**
3. Configure:
   - **Name**: "Exchange Rate Analysis"
   - **Role**: `ANALYTICS_DEVELOPER` or `ANALYTICS_REPORTER`
   - **Warehouse**: `TRANSFORMING` (or `REPORTING` for read-only)
   - **Database**: `ANALYTICS`
   - **Schema**: `REPORTING`
4. Click **Create**

### Example: Query Exchange Rates

```python
# Cell 1: Import Snowpark
from snowflake.snowpark import Session
from snowflake.snowpark.functions import col, avg, stddev, count
import pandas as pd
import matplotlib.pyplot as plt

# Session is already initialized in Snowflake Notebooks
# Use the global 'session' object
```

```python
# Cell 2: Query exchange rates
df = session.table("ANALYTICS.REPORTING.FCT_EXCHANGE_RATES") \
    .filter(col("BASE_CURRENCY") == "GBP") \
    .filter(col("TARGET_CURRENCY").in_(["USD", "EUR", "JPY"])) \
    .filter(col("RATE_DATE") >= "2025-08-01") \
    .select("RATE_DATE", "TARGET_CURRENCY", "EXCHANGE_RATE")

# Convert to Pandas for plotting
df_pandas = df.to_pandas()
df_pandas.head()
```

```python
# Cell 3: Plot exchange rate trends
import matplotlib.pyplot as plt
import seaborn as sns

sns.set_style("whitegrid")

# Pivot for plotting
df_pivot = df_pandas.pivot(index='RATE_DATE', columns='TARGET_CURRENCY', values='EXCHANGE_RATE')

# Plot
df_pivot.plot(figsize=(12, 6), title="GBP Exchange Rates (Aug 2025 - Present)")
plt.ylabel("Exchange Rate")
plt.xlabel("Date")
plt.legend(title="Currency")
plt.show()
```

```python
# Cell 4: Calculate volatility
volatility = session.table("ANALYTICS.REPORTING.FCT_EXCHANGE_RATES") \
    .filter(col("BASE_CURRENCY") == "GBP") \
    .filter(col("RATE_DATE") >= "2025-08-01") \
    .group_by("TARGET_CURRENCY") \
    .agg(
        stddev(col("EXCHANGE_RATE")).alias("VOLATILITY"),
        avg(col("EXCHANGE_RATE")).alias("AVG_RATE"),
        count(col("EXCHANGE_RATE")).alias("NUM_OBSERVATIONS")
    ) \
    .sort(col("VOLATILITY").desc())

volatility.show()
```

### Saving and Sharing

1. Snowflake auto-saves notebooks as you work
2. To share: Click **Share** → enter Snowflake usernames
3. To download: **File** → **Download as .ipynb** (Jupyter format)

!!! warning "No Git Integration"
    Snowflake Notebooks don't integrate with Git. For version control, export to `.ipynb` and commit to your repository manually, or use Jupyter locally.

## Option 2: Jupyter (Self-Hosted or Local)

Jupyter is the de facto standard for data science notebooks. Run it locally or self-host on AWS EC2.

### Pros and Cons

**Pros:**
- **Full Python ecosystem** — install any PyPI package
- **Fast iteration** — compute runs locally (no network latency)
- **Git version control** — commit `.ipynb` files (or use jupytext for `.py` files)
- **Multi-language** — Python, R, Julia, SQL, Scala kernels available
- **Extensions** — rich ecosystem of Jupyter extensions

**Cons:**
- **Setup required** — install Python, Jupyter, Snowflake connector
- **Local compute** — limited by your machine (no auto-scaling)
- **Sharing** — requires exporting notebooks or using JupyterHub (self-hosted)

**Best for:** Analysts and data scientists who want full control and flexibility.

### Install Jupyter Locally

```sh
# Create a new Python virtual environment for notebooks
cd ~
mkdir notebooks
cd notebooks

uv init
uv add jupyterlab snowflake-connector-python pandas matplotlib seaborn scikit-learn

# Start JupyterLab
jupyter lab
```

JupyterLab opens in your browser at `http://localhost:8888`.

### Connect to Snowflake

Create a notebook and configure the Snowflake connection:

```python
# Cell 1: Import libraries
import snowflake.connector
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns

# Cell 2: Connect to Snowflake
conn = snowflake.connector.connect(
    user='YOUR_SNOWFLAKE_USERNAME',
    account='your-account.snowflakecomputing.com',
    authenticator='externalbrowser',  # SSO authentication
    warehouse='REPORTING',
    database='ANALYTICS',
    schema='REPORTING',
    role='ANALYTICS_DEVELOPER'
)

# Create a cursor
cur = conn.cursor()
```

!!! tip "Authentication Methods"
    - **SSO (externalbrowser)**: Opens browser for Okta/Azure AD login
    - **Username/password**: `password='your-password'` (not recommended)
    - **Key-pair**: Use private key for service account authentication

### Query Data

```python
# Cell 3: Query exchange rates
query = """
SELECT
    rate_date,
    target_currency,
    exchange_rate
FROM analytics.reporting.fct_exchange_rates
WHERE base_currency = 'GBP'
    AND target_currency IN ('USD', 'EUR', 'JPY')
    AND rate_date >= '2025-08-01'
ORDER BY rate_date, target_currency
"""

# Execute and fetch as Pandas DataFrame
cur.execute(query)
df = cur.fetch_pandas_all()

df.head()
```

```python
# Cell 4: Plot trends
df_pivot = df.pivot(index='RATE_DATE', columns='TARGET_CURRENCY', values='EXCHANGE_RATE')

df_pivot.plot(figsize=(12, 6), title="GBP Exchange Rates")
plt.ylabel("Exchange Rate")
plt.xlabel("Date")
plt.show()
```

### Version Control with Jupytext

Jupyter notebooks (`.ipynb` files) are JSON and difficult to review in Git. Use **jupytext** to save notebooks as `.py` files:

```sh
# Install jupytext
uv add jupytext

# Convert notebook to Python script
jupytext --to py exchange_rate_analysis.ipynb

# This creates exchange_rate_analysis.py (readable, Git-friendly)
```

**Pair the notebook with a `.py` file:**

```sh
# Configure jupytext to auto-sync .ipynb ↔ .py
jupytext --set-formats ipynb,py:percent exchange_rate_analysis.ipynb
```

Now changes to the `.ipynb` file automatically update the `.py` file. Commit the `.py` file to Git (ignore `.ipynb` in `.gitignore`).

**Example `.gitignore`:**

```gitignore
# Jupyter
.ipynb_checkpoints/
*.ipynb
```

**Commit only the `.py` files:**

```sh
git add exchange_rate_analysis.py
git commit -m "Add exchange rate analysis notebook"
git push
```

Teammates can recreate the `.ipynb` from the `.py` file:

```sh
jupytext --to notebook exchange_rate_analysis.py
```

!!! success "Best of Both Worlds"
    Jupytext gives you the interactivity of `.ipynb` with the Git-friendliness of `.py` files.

## Example: ML Notebook to Predict GBP/USD Exchange Rate

### Objective

Build a simple time-series forecasting model to predict the GBP/USD exchange rate for the next 30 days using historical data.

### Step 1: Load and Prepare Data

```python
# Cell 1: Import libraries
import snowflake.connector
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from sklearn.linear_model import LinearRegression
from sklearn.metrics import mean_absolute_error, mean_squared_error
from datetime import datetime, timedelta

# Cell 2: Connect to Snowflake
conn = snowflake.connector.connect(
    user='your_username',
    account='your-account.snowflakecomputing.com',
    authenticator='externalbrowser',
    warehouse='REPORTING',
    database='ANALYTICS',
    schema='REPORTING',
    role='ANALYTICS_DEVELOPER'
)

cur = conn.cursor()
```

```python
# Cell 3: Query GBP/USD historical data
query = """
SELECT
    rate_date,
    exchange_rate
FROM analytics.reporting.fct_exchange_rates
WHERE base_currency = 'GBP'
    AND target_currency = 'USD'
    AND rate_date >= '2024-01-01'
ORDER BY rate_date
"""

cur.execute(query)
df = cur.fetch_pandas_all()

# Convert date to datetime
df['RATE_DATE'] = pd.to_datetime(df['RATE_DATE'])
df = df.sort_values('RATE_DATE').reset_index(drop=True)

print(f"Loaded {len(df)} days of GBP/USD exchange rates")
df.head()
```

### Step 2: Feature Engineering

```python
# Cell 4: Create features for ML model
# Features: day of year, 7-day rolling average, 30-day rolling average

df['DAY_OF_YEAR'] = df['RATE_DATE'].dt.dayofyear
df['ROLLING_AVG_7'] = df['EXCHANGE_RATE'].rolling(window=7).mean()
df['ROLLING_AVG_30'] = df['EXCHANGE_RATE'].rolling(window=30).mean()
df['LAG_1'] = df['EXCHANGE_RATE'].shift(1)  # Previous day's rate
df['LAG_7'] = df['EXCHANGE_RATE'].shift(7)  # Rate 7 days ago

# Drop NaN rows (from rolling windows and lags)
df = df.dropna()

df.head()
```

### Step 3: Train/Test Split

```python
# Cell 5: Split into train and test sets
# Use last 30 days as test set
train_size = len(df) - 30
train_df = df.iloc[:train_size]
test_df = df.iloc[train_size:]

print(f"Train set: {len(train_df)} days")
print(f"Test set: {len(test_df)} days")
print(f"Train period: {train_df['RATE_DATE'].min()} to {train_df['RATE_DATE'].max()}")
print(f"Test period: {test_df['RATE_DATE'].min()} to {test_df['RATE_DATE'].max()}")
```

### Step 4: Train Model

```python
# Cell 6: Train a simple linear regression model
features = ['DAY_OF_YEAR', 'ROLLING_AVG_7', 'ROLLING_AVG_30', 'LAG_1', 'LAG_7']
target = 'EXCHANGE_RATE'

X_train = train_df[features]
y_train = train_df[target]

X_test = test_df[features]
y_test = test_df[target]

# Train model
model = LinearRegression()
model.fit(X_train, y_train)

print("Model trained!")
print(f"Coefficients: {model.coef_}")
print(f"Intercept: {model.intercept_}")
```

### Step 5: Evaluate Model

```python
# Cell 7: Predict on test set
y_pred = model.predict(X_test)

# Calculate metrics
mae = mean_absolute_error(y_test, y_pred)
rmse = np.sqrt(mean_squared_error(y_test, y_pred))

print(f"Mean Absolute Error (MAE): {mae:.4f}")
print(f"Root Mean Squared Error (RMSE): {rmse:.4f}")
```

```python
# Cell 8: Visualise predictions vs actual
plt.figure(figsize=(12, 6))
plt.plot(test_df['RATE_DATE'], y_test, label='Actual', marker='o')
plt.plot(test_df['RATE_DATE'], y_pred, label='Predicted', marker='x', linestyle='--')
plt.title("GBP/USD Exchange Rate: Actual vs Predicted")
plt.xlabel("Date")
plt.ylabel("Exchange Rate")
plt.legend()
plt.grid(True)
plt.show()
```

### Step 6: Forecast Future 30 Days

```python
# Cell 9: Forecast next 30 days
# Note: This is a simplified example. Real forecasting would use ARIMA, Prophet, or LSTM.

last_date = df['RATE_DATE'].max()
forecast_dates = pd.date_range(start=last_date + timedelta(days=1), periods=30, freq='D')

# For simplicity, use the last known rolling averages and lags
# In production, you'd recursively forecast day-by-day
forecast_data = []

for date in forecast_dates:
    # Use last 30 days to compute rolling average (simplified)
    recent_rates = df.tail(30)['EXCHANGE_RATE'].values
    rolling_avg_30 = np.mean(recent_rates)
    rolling_avg_7 = np.mean(recent_rates[-7:])
    lag_1 = recent_rates[-1]
    lag_7 = recent_rates[-7] if len(recent_rates) >= 7 else lag_1
    day_of_year = date.dayofyear

    features_input = [[day_of_year, rolling_avg_7, rolling_avg_30, lag_1, lag_7]]
    predicted_rate = model.predict(features_input)[0]

    forecast_data.append({
        'DATE': date,
        'PREDICTED_RATE': predicted_rate
    })

forecast_df = pd.DataFrame(forecast_data)
forecast_df.head()
```

```python
# Cell 10: Visualise forecast
plt.figure(figsize=(14, 6))
plt.plot(df['RATE_DATE'], df['EXCHANGE_RATE'], label='Historical', color='blue')
plt.plot(forecast_df['DATE'], forecast_df['PREDICTED_RATE'], label='Forecast (Next 30 Days)', color='red', linestyle='--', marker='o')
plt.axvline(x=last_date, color='gray', linestyle='--', label='Forecast Start')
plt.title("GBP/USD Exchange Rate Forecast")
plt.xlabel("Date")
plt.ylabel("Exchange Rate")
plt.legend()
plt.grid(True)
plt.show()

print(f"Forecasted GBP/USD rate in 30 days: {forecast_df['PREDICTED_RATE'].iloc[-1]:.4f}")
```

!!! warning "Model Limitations"
    This is a simplified example for learning purposes. Real exchange rate forecasting requires:
    - Advanced models (ARIMA, Prophet, LSTM, transformers)
    - Exogenous variables (interest rates, economic indicators)
    - Backtesting and validation on multiple time periods
    - Uncertainty quantification (confidence intervals)

## Option 3: Hex (Premium Collaborative Notebooks)

Hex is a commercial notebook platform that combines SQL, Python, and R in a collaborative, cloud-based environment.

### Pros and Cons

**Pros:**
- **Collaborative** — multiple users can edit simultaneously (like Google Docs)
- **SQL + Python + R** — mix languages in one notebook
- **Built-in visualisations** — drag-and-drop charts (no matplotlib code)
- **App builder** — turn notebooks into interactive apps for stakeholders
- **Version control** — built-in Git integration
- **Scheduled runs** — automate notebook execution

**Cons:**
- **Expensive** — $70+/user/month (Team plan)
- **Cloud-only** — no self-hosted option
- **Smaller ecosystem** — fewer users than Jupyter

**Best for:** Teams wanting collaborative, polished notebooks with built-in visualisations and app building.

### Getting Started with Hex

1. Sign up at [hex.tech](https://hex.tech/)
2. Connect to Snowflake:
   - Navigate to **Workspace Settings** → **Data Connections**
   - Add Snowflake connection (use `SVC_LIGHTDASH` or create a dedicated `SVC_HEX` account)
3. Create a new notebook:
   - Click **+ New Project**
   - Add SQL cells (query Snowflake directly)
   - Add Python cells (process results with pandas)
   - Add visualisation cells (drag-and-drop charts)

### Example Hex Notebook

**Cell 1 (SQL):**
```sql
SELECT
    rate_date,
    target_currency,
    exchange_rate
FROM analytics.reporting.fct_exchange_rates
WHERE base_currency = 'GBP'
    AND target_currency IN ('USD', 'EUR', 'JPY')
    AND rate_date >= CURRENT_DATE() - INTERVAL '6 months'
```

**Cell 2 (Python):**
```python
import pandas as pd

# Reference SQL cell result as DataFrame
df = sql_result  # Hex auto-names SQL results

df_pivot = df.pivot(index='rate_date', columns='target_currency', values='exchange_rate')
df_pivot.head()
```

**Cell 3 (Chart):**
- Drag-and-drop chart builder
- X-axis: `rate_date`
- Y-axis: `exchange_rate`
- Group by: `target_currency`
- Chart type: Line chart

**Cell 4 (App Input):**
- Add an input widget (dropdown for currency selection)
- Users can interact without editing code

## Comparison Summary

| Feature | Snowflake Notebooks | Jupyter (Local) | Jupyter (Self-Hosted) | Hex |
|---------|-------------------|----------------|---------------------|-----|
| **Cost** | Free (compute only) | Free | ~$15/month (EC2) | $70+/user/month |
| **Setup** | Zero | Local install | AWS deployment | Sign up + connect |
| **Languages** | Python | Python, R, Julia | Python, R, Julia | SQL, Python, R |
| **Collaboration** | Share in Snowflake | Export `.ipynb` | JupyterHub | Real-time co-editing |
| **Git integration** | Manual export | Yes (jupytext) | Yes | Built-in |
| **Performance** | Warehouse compute | Local machine | Self-hosted server | Cloud compute |
| **Best for** | Quick Snowflake analysis | Power users, data scientists | Teams, shared environments | Collaborative teams, polished UX |

## When to Use Each Tool

| Use Case | Recommended Tool |
|----------|------------------|
| Quick SQL exploration | **Snowsight** (not notebooks) |
| Python analysis on Snowflake data (no local setup) | **Snowflake Notebooks** |
| Data science / ML with full Python ecosystem | **Jupyter (local or self-hosted)** |
| Collaborative analysis with stakeholders | **Hex** |
| Share reproducible analysis via Git | **Jupyter + jupytext** |
| Build interactive apps from notebooks | **Hex** |

## Summary

You've learned about ad-hoc analytics with notebooks:

- [x] **Snowflake Notebooks** — built-in, free, Python-only, good for quick analysis
- [x] **Jupyter (local)** — full ecosystem, fast iteration, Git version control with jupytext
- [x] **Jupyter (self-hosted)** — shared JupyterHub for teams
- [x] **Hex** — collaborative, polished, expensive, great for teams building interactive apps
- [x] **Example ML notebook** — predicting GBP/USD exchange rates with linear regression
- [x] **Jupytext** — version control notebooks as `.py` files for readable Git diffs

Use notebooks for exploratory analysis and custom investigations. Use dashboards (Lightdash, Tableau) for operational reporting and known questions.

## What's Next

You've built the complete analytics stack: dashboards for monitoring, notebooks for exploration. Learn when to upgrade to enterprise BI tools and what comes after the data analytics layer.

Continue to [Finishing Up](12-finishing-up.md) →
