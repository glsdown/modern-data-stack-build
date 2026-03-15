# Finishing Up

On this page, you will:

- [x] Verify your complete analytics stack is working end-to-end
- [x] Understand when to upgrade from Lightdash to enterprise BI tools
- [x] Learn when to choose alternative BI tools (Metabase, Omni, Tableau)
- [x] Review what you've built across the entire modern data stack
- [x] Explore next steps for scaling your data platform

## Overview

You've built a complete analytics stack: dashboards for operational reporting, notebooks for exploratory analysis, and metrics defined in code. This page helps you verify everything works, understand when to evolve your tooling, and plan next steps.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      COMPLETE ANALYTICS STACK                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Raw Data        Transformation      Analytics Layer                   │
│  ────────        ───────────────     ────────────────                  │
│                                                                         │
│  ┌──────────┐   ┌──────────────┐    ┌────────────────┐                │
│  │ DLT      │──▶│ dbt (MARTS)  │───▶│ Lightdash      │                │
│  │ SNOWPIPE │   │ • Staging    │    │ • Dashboards   │                │
│  │ AIRBYTE  │   │ • Int models │    │ • Metrics      │                │
│  └──────────┘   │ • Fact/Dim   │    └────────────────┘                │
│                 │ • REPORTING  │                                       │
│                 └──────────────┘    ┌────────────────┐                │
│                                     │ Snowsight      │                │
│                                     │ • SQL queries  │                │
│                                     └────────────────┘                │
│                                                                         │
│                                     ┌────────────────┐                │
│                                     │ Notebooks      │                │
│                                     │ • Jupyter      │                │
│                                     │ • Snowflake    │                │
│                                     │ • Hex          │                │
│                                     └────────────────┘                │
│                                                                         │
│  Orchestrated by Prefect, secured by role-based access control.        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Verification Checklist

Confirm your analytics stack is fully operational:

### Data Flow

- [ ] **Raw data loaded**: `DLT`, `SNOWPIPE`, `AIRBYTE` databases contain recent data
  ```sql
  -- Check latest load timestamps
  SELECT MAX(loaded_at) FROM dlt.currencies;
  SELECT MAX(loaded_at) FROM snowpipe.exchange_rates;
  SELECT MAX(loaded_at) FROM airbyte.contacts;
  ```

- [ ] **dbt models built**: `ANALYTICS.MARTS` and `ANALYTICS.REPORTING` contain transformed data
  ```sql
  -- Check latest dbt run
  SELECT COUNT(*) FROM analytics.marts.fct_exchange_rates;
  SELECT COUNT(*) FROM analytics.reporting.fct_exchange_rates;
  ```

- [ ] **Orchestration working**: Prefect flows run successfully and trigger dbt
  - Navigate to Prefect UI → Check flow run history
  - Verify no failed runs in the last 7 days

### Analytics Layer

- [ ] **Snowsight accessible**: Can query `REPORTING` schema and create basic charts
  - Open Snowsight → Run a query against `fct_exchange_rates`
  - Create a simple chart → Verify it displays

- [ ] **Lightdash connected**: Can explore dbt models and see defined metrics
  - Open Lightdash → Navigate to **Explore** → Select `Exchange Rates`
  - Verify metrics appear (Average Exchange Rate, Volatility, etc.)

- [ ] **Dashboards live**: Created dashboards show data and refresh automatically
  - Open "GBP Exchange Rates Dashboard"
  - Verify charts display data (not "No data")
  - Apply a filter → Verify charts update

- [ ] **Notebooks working**: Can connect to Snowflake and query data
  - Open a notebook (Snowflake Notebooks or Jupyter)
  - Query `fct_exchange_rates` → Verify data loads
  - Create a simple plot → Verify it renders

### Access Control

- [ ] **Service accounts working**: `SVC_LIGHTDASH` can query `REPORTING` schema
  ```sql
  USE ROLE SVC_LIGHTDASH;
  SELECT COUNT(*) FROM analytics.reporting.fct_exchange_rates;
  -- Should succeed

  SELECT COUNT(*) FROM analytics.marts.fct_exchange_rates;
  -- Should fail (ANALYTICS_REPORTER has no access to MARTS)
  ```

- [ ] **Roles configured**: Analysts have `ANALYTICS_DEVELOPER` or `ANALYTICS_REPORTER`
  ```sql
  SHOW GRANTS TO ROLE ANALYTICS_REPORTER;
  -- Should show grants on REPORTING schema only
  ```

### Metrics as Code

- [ ] **Metrics in dbt YAML**: `fct_exchange_rates.yml` contains metrics definitions
  ```sh
  cat models/marts/core/fct_exchange_rates.yml | grep -A 5 "metrics:"
  ```

- [ ] **Auto-sync working**: Pushing to `main` branch triggers Lightdash recompile
  - Make a small change to a dbt YAML file (e.g., update a metric label)
  - Push to GitHub → Check Lightdash compiles within 2 minutes

!!! success "Verification Complete"
    If all checks pass, your modern data stack is operational from ingestion → transformation → analytics.

## When to Upgrade to Enterprise BI Tools

Lightdash (or Metabase, Omni) is excellent for small to mid-size teams. Upgrade to enterprise tools (Tableau, Looker, Power BI) when:

### 1. You Need Advanced Visualisations

**Upgrade when:**
- Users request geospatial analysis (maps with custom shapefiles)
- Complex custom visualisations (D3.js charts, network graphs)
- Advanced interactivity (drill-through to transaction-level detail across multiple models)

**Lightdash limitation**: Basic chart types (line, bar, pie, table). No maps, custom D3.js charts, or advanced interactivity.

**Enterprise solution**: Tableau excels at complex visualisations. Power BI good for Microsoft-heavy orgs. Looker for LookML-based metric definitions.

### 2. You Have 50+ Business Users

**Upgrade when:**
- 50+ non-technical users need self-service dashboards
- Multiple departments (finance, marketing, sales, product) each need customised views
- Users request mobile apps for on-the-go access

**Lightdash limitation**: Works well for 5-30 users. At 50+ users, you may outgrow the UI/UX polish and want enterprise support.

**Enterprise solution**: Tableau and Power BI have mature self-service interfaces. Power BI integrates with Microsoft 365 (familiar to business users).

### 3. You Need Embedded Analytics

**Upgrade when:**
- You want to embed dashboards in customer-facing applications
- White-labelling required (remove BI tool branding)
- Row-level security for multi-tenant applications

**Lightdash limitation**: Basic embedding support. Not designed for customer-facing, white-labelled analytics.

**Enterprise solution**: Tableau, Looker, and Power BI support embedded analytics with extensive customisation options.

### 4. You Need Enterprise Governance

**Upgrade when:**
- Compliance requires detailed audit logs (who viewed which dashboard, when)
- Row-level security based on complex user attributes
- Certified datasets and data lineage tracking across BI + warehouse + dbt

**Lightdash limitation**: Basic role-based access control. Limited audit logging and governance features.

**Enterprise solution**: Looker and Tableau have robust governance, audit logs, and certification workflows.

### 5. Budget Allows and Stakeholders Demand It

**Upgrade when:**
- Budget exists for $5000+/month on BI tools
- Executives or board members request "enterprise-grade" BI
- Existing company-wide contracts (e.g., Microsoft E5 includes Power BI)

**Lightdash limitation**: Smaller brand recognition. Stakeholders may prefer "Tableau" on the roadmap for credibility.

**Enterprise solution**: Tableau and Looker have strong brand recognition and enterprise sales support.

## Migration Paths

### From Lightdash to Tableau/Looker

**Challenge**: Metrics defined in dbt YAML (Lightdash) don't transfer directly to Tableau (UI-based) or Looker (LookML).

**Strategy:**
1. **Keep dbt as source of truth** — continue transforming data with dbt
2. **Publish to `REPORTING` schema** — Tableau/Looker query the same schema Lightdash uses
3. **Rebuild metrics in BI tool** — manually recreate aggregations in Tableau or LookML
4. **Document mapping** — maintain a mapping of dbt metrics → Tableau calculated fields

**Effort**: 1-2 weeks for 10-20 dashboards (rebuilding charts, filters, permissions).

### From Lightdash to Looker

**More straightforward path**: Looker's LookML is code-based (like dbt metrics). You can translate dbt YAML metrics to LookML views.

**Example:**

**dbt YAML (Lightdash):**
```yaml
metrics:
  average_exchange_rate:
    type: average
    sql: "${exchange_rate}"
```

**LookML (Looker):**
```lookml
measure: average_exchange_rate {
  type: average
  sql: ${exchange_rate} ;;
}
```

This is more mechanical than Tableau (UI-based metrics).

### From Lightdash to Power BI

**Microsoft-heavy orgs**: If you use Office 365, Azure, and Excel, Power BI is a natural fit.

**Migration**:
- Connect Power BI to Snowflake
- Import `REPORTING` schema tables
- Rebuild metrics as DAX measures (Power BI's formula language)

**Effort**: 2-3 weeks for 10-20 dashboards.

## When to Choose Alternative BI Tools

### Choose Metabase Instead of Lightdash

**When:**
- You **don't have a dbt transformation layer** (or don't plan to build one)
- You want a **user-friendly UI** for non-technical business users (Metabase is more accessible than Lightdash for SQL beginners)
- You need **mature open source** (Metabase has been around since 2015)

**Trade-off**: No dbt integration. Metrics defined in Metabase UI (not code). Business logic lives in BI tool, not dbt.

### Choose Omni Instead of Lightdash

**When:**
- Budget allows $60-300/month for a small team (3-15 users)
- You want **polished UX** without self-hosting
- You value **dbt-native metrics** but don't want to manage ECS/RDS infrastructure

**Trade-off**: More expensive than Lightdash self-hosted (~$60/month vs ~$30/month). No self-hosted option (cloud-only).

### Choose Tableau/Looker Instead of Lightdash

**When:**
- Budget allows $5000+/month
- You need **advanced visualisations** (maps, custom charts)
- You have **50+ users** across multiple departments
- **Stakeholders demand enterprise-grade BI**

**Trade-off**: Expensive. Metrics defined in UI (Tableau) or LookML (Looker), not dbt YAML. No dbt-native integration.

### Choose Power BI Instead of Lightdash

**When:**
- Your organisation is **Microsoft-heavy** (Office 365, Azure)
- Budget is $500-2000/month (Power BI cheaper than Tableau/Looker)
- Users are familiar with **Excel** (Power BI feels similar)

**Trade-off**: Best experience requires Windows. No dbt integration. Metrics in DAX (not dbt).

### Stick with Snowsight (No Dedicated BI Tool)

**When:**
- Team is small (1-5 analysts) and **SQL-proficient**
- Budget is $0 and cannot justify even $30/month self-hosting
- You only need **basic dashboards** and ad-hoc queries

**Trade-off**: Requires SQL knowledge. Not self-service for business users. Basic visualisations only.

## Summary

Across this documentation, you've built a complete modern data stack:

### Infrastructure Layer

- **AWS**: S3 data lake, VPC, IAM roles, Secrets Manager
- **Terraform**: Infrastructure as code for AWS, Snowflake, and GitHub
- **GitHub**: Repositories for Terraform, dbt, Prefect, and documentation

### Data Warehouse Layer

- **Snowflake**: ANALYTICS database, role-based access control, warehouses
- **Schemas**: STAGING, INTERMEDIATE, MARTS, REPORTING
- **Roles**: ANALYTICS_TRANSFORMER, ANALYTICS_DEVELOPER, ANALYTICS_REPORTER

### Ingestion Layer

- **dlt**: Lightweight pipelines for APIs and databases (free)
- **Snowpipe**: Continuous S3-to-Snowflake ingestion (Snowflake-native)
- **Airbyte**: SaaS connectors for HubSpot and other tools (Cloud or self-hosted)

### Orchestration Layer

- **Prefect**: Workflow orchestration for dlt, Snowpipe, and dbt (Cloud or self-hosted)
- **Schedules**: Daily data ingestion → dbt transformation → BI refresh

### Transformation Layer

- **dbt**: Models, tests, documentation (separate repository)
- **dbt Core or dbt Cloud**: CI/CD, state deferral, docs hosting

### Analytics Layer

- **Snowsight**: Free, built-in SQL exploration and basic dashboards
- **Lightdash**: dbt-native BI tool with metrics as code (Cloud or self-hosted)
- **Jupyter/Snowflake Notebooks**: Ad-hoc analysis and ML
- **Dashboards**: Exchange rates, product catalogue, and custom views

### Observability and Governance

- **dbt tests**: Data quality checks on every model
- **Prefect monitoring**: Flow run history and alerting
- **Snowflake query history**: Audit all data access
- **Role-based access control**: Least privilege for all service accounts

## What's Next: Scaling the Data Platform

You've built the foundation. Here's what comes next as you scale:

### 1. Expand Data Sources

- Add more SaaS tools (Stripe, Salesforce, Zendesk) via Airbyte
- Ingest product analytics (Segment, Amplitude) via dlt or Snowpipe
- Connect to operational databases (PostgreSQL, MySQL) via dlt

### 2. Build More dbt Models

- **Sales analytics**: Revenue, pipeline, conversion funnels
- **Marketing analytics**: Campaign performance, attribution
- **Product analytics**: User engagement, retention cohorts
- **Finance analytics**: P&L, cash flow, forecasting

### 3. Implement Data Quality Monitoring

- **dbt tests**: Expand beyond basic not_null and unique tests
- **dbt_expectations**: Add statistical tests (mean within range, no anomalies)
- **Elementary**: Open source data observability for dbt
- **Monte Carlo / Anomalo**: Commercial data quality platforms

### 4. Add Reverse ETL

- **Hightouch or Census**: Sync warehouse data back to SaaS tools
- Use case: Export lead scores from Snowflake → Salesforce
- Use case: Sync customer segments → email marketing platform

### 5. Implement a Semantic Layer

- **dbt Metrics**: Define metrics once, query from BI tools
- **dbt Cloud semantic layer**: Team/Enterprise only ($100+/user/month)
- **Cube.dev**: Open source semantic layer with caching

### 6. Improve BI Maturity

- Upgrade to Looker, Tableau, or Power BI if needed
- Build **metric catalogues** (centralised registry of all KPIs)
- Implement **data governance** (ownership, definitions, lineage)

### 7. Scale Infrastructure

- **Auto-scaling Snowflake warehouses** based on query load
- **Multi-environment dbt** (staging, QA, production)
- **dbt Mesh**: Split dbt project into domains (finance, marketing, product)

### 8. Machine Learning in Production

- **Feature stores** (Feast, Tecton) for ML feature management
- **Model deployment** (SageMaker, Vertex AI) for production ML models
- **Real-time inference** (Snowpark, AWS Lambda) for low-latency predictions

### 9. Real-Time Data

- **Change Data Capture (CDC)** from databases (Debezium, Airbyte)
- **Streaming pipelines** (Kafka, Kinesis, Flink)
- **Real-time warehouses** (ClickHouse, Apache Druid) for sub-second queries

### 10. Cost Optimisation

- **Monitor Snowflake spend** — set up Resource Monitors and alerts
- **Optimise dbt models** — use incremental materialisation, cluster keys
- **Right-size warehouses** — start with X-Small, scale up only when needed

## Cost Summary: Running the Full Stack

| Component | Monthly Cost | Notes |
|-----------|-------------|-------|
| **AWS Infrastructure** | $30-50 | S3, VPC, Secrets Manager, ECS (if self-hosting Prefect/Lightdash) |
| **Snowflake** | $50-200 | Compute-based; varies with query volume |
| **Prefect Cloud** | $0 (Free) or $450+ (Team) | Self-hosted free; Cloud $450/month for teams |
| **dlt** | $0 | Open source, free |
| **Airbyte Cloud** | $99+/month | Or $81/month self-hosted (ECS) |
| **dbt Core** | $0 | Open source, free |
| **dbt Cloud** | $0 (Developer) or $100+/user (Team) | Developer free, Team $100/user/month |
| **Lightdash Self-Hosted** | ~$30 | ECS + RDS |
| **Lightdash Cloud** | $2400 | Flat rate, unlimited users |
| **Snowsight** | $0 | Included with Snowflake |
| **Jupyter (local)** | $0 | Free, runs locally |
| **Total (Budget Build)** | ~$250-400/month | Self-hosted, free tiers, Snowflake moderate usage |
| **Total (Premium Build)** | ~$3500+/month | dbt Cloud, Prefect Cloud, Lightdash Cloud, Airbyte Cloud |

Your budget build (~$250-400/month) provides:
- Production-grade data warehouse
- Automated ingestion and transformation
- BI dashboards and ad-hoc analytics
- Role-based access control and security

This is far cheaper than enterprise data platforms ($10,000+/month) while providing similar capabilities.

## Congratulations!

You've built a **modern data stack from scratch**:

- [x] Infrastructure as code (Terraform for AWS, Snowflake, GitHub)
- [x] Data warehouse (Snowflake with role-based access control)
- [x] Data ingestion (dlt, Snowpipe, Airbyte)
- [x] Orchestration (Prefect)
- [x] Transformation (dbt)
- [x] Analytics (Snowsight, Lightdash, Jupyter)

You can now:
- Ingest data from APIs, databases, and SaaS tools
- Transform raw data into analytics-ready models
- Create dashboards for business users
- Perform ad-hoc analysis with notebooks
- Manage everything with code (Git, Terraform, dbt, Prefect)

## What's Next Beyond This Documentation

This documentation has focused on **building the stack**. Next steps:

1. **Operate the stack** — monitoring, alerting, incident response
2. **Scale the stack** — add data sources, models, users
3. **Optimise the stack** — performance tuning, cost reduction
4. **Govern the stack** — data catalogues, access controls, compliance

Consider exploring:
- **Data catalogues** (Atlan, Collibra, Amundsen) for metadata management
- **Data lineage** (Elementary, Marquez) for impact analysis
- **Data contracts** (dbt contracts, Great Expectations) for API-like data interfaces
- **Data mesh** (dbt Mesh, federated data ownership) for scaling beyond one team

The modern data stack is not static — it evolves with your organisation's needs. You've built the foundation. Now you can scale, optimise, and adapt as you grow.

## Additional Resources

- **dbt Learn**: [learn.getdbt.com](https://learn.getdbt.com/) — official dbt courses
- **Prefect University**: [university.prefect.io](https://university.prefect.io/) — free Prefect training
- **Locally Optimistic**: Community blog for analytics engineers
- **Data Engineering Podcast**: Weekly podcast on data tooling
- **Analytics Engineer Slack**: Community for dbt and analytics engineering

Thank you for following this documentation. You're now equipped to build, operate, and evolve modern data platforms.

---

**End of Data Analytics Section**

**End of Build Section**

If you've completed all build sections:
- ✅ AWS Infrastructure
- ✅ Data Warehouse
- ✅ Orchestration
- ✅ Batch Data Ingestion
- ✅ SaaS Ingestion
- ✅ Data Transformation
- ✅ Data Analytics

...you have a **complete, production-grade modern data stack**.

Next: **Maintain** (monitoring, troubleshooting, cost optimisation) — coming soon.
