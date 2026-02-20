# Data Analytics

On this page, you will:

- [x] Understand the role of BI tools in the modern data stack
- [x] Learn about the BI tool landscape and evaluation criteria
- [x] Plan the analytics layer that connects transformed data to business users

## Overview

This section covers the final layer of the modern data stack: **business intelligence (BI)** and **analytics**. The data warehouse, ingestion pipelines, and transformation models are infrastructure for this layer — BI tools are where business value is realised. They turn analytics-ready data into dashboards, reports, and insights that drive decisions.

The analytics layer sits on top of the dbt transformation layer and provides:

- **Self-service exploration** — business users can query data without SQL knowledge
- **Scheduled dashboards** — automated reporting for stakeholders
- **Visual analysis** — charts, graphs, and visualisations for pattern recognition
- **Metrics tracking** — KPIs and business metrics monitored over time

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                          DATA ANALYTICS LAYER                                  │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  Transformation (dbt)          Analytics Tools              Business Users     │
│  ────────────────────          ───────────────              ──────────────     │
│                                                                                │
│  ┌──────────────────┐          ┌──────────────────┐                            │
│  │ ANALYTICS.MARTS  │          │  BI Tool         │         ┌────────────────┐ │
│  │ • fct_exchange   │─────────▶│  • Lightdash     │────────▶│  Executives    │ │
│  │   _rates         │          │  • Omni          │         │  Analysts      │ │
│  │ • dim_products   │          │  • Metabase      │         │  Managers      │ │
│  │ • fct_contacts   │          │  • Tableau       │         └────────────────┘ │
│  └──────────────────┘          │  • Power BI      │                            │
│                                │  • Snowsight     │                            │
│  ┌──────────────────┐          └──────────────────┘                            │
│  │ ANALYTICS.       │                   │                                      │
│  │ REPORTING        │───────────────────┘                                      │
│  │ (BI-ready models)│                                                          │
│  └──────────────────┘                                                          │
│                                                                                │
│  dbt-defined Metrics                                                           │
│  ────────────────────                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐       │
│  │  Metrics defined in dbt YAML (Lightdash, Omni)                      │       │
│  │  • Version-controlled alongside models                              │       │
│  │  • Single source of truth for business logic                        │       │
│  │  • Automatically available in BI tool                               │       │
│  └─────────────────────────────────────────────────────────────────────┘       │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
```

## The BI Tool Landscape

The BI tool market is broad, spanning from enterprise platforms costing $100+/user/month to open-source tools you can self-host for free. The right choice depends on your budget, team size, technical capability, and whether you need native dbt integration.

This section provides:

1. **Comprehensive landscape overview** — all major BI tools categorised by type (enterprise, modern/dbt-native, open source, cloud-native)
2. **Evaluation framework** — criteria for choosing the right tool for your organisation
3. **Quick win with Snowsight** — Snowflake's built-in dashboarding (free, zero setup)
4. **Hands-on Lightdash implementation** — a cost-effective, dbt-native BI tool

You are not locked into any single tool. The documentation provides the evaluation framework to choose what's right for you, then demonstrates a practical implementation with Lightdash.

## Why Lightdash as the Example?

This section uses **Lightdash** for the hands-on implementation because:

- **Free self-hosted option** — aligns with the project's cost-conscious approach
- **Native dbt integration** — reads your dbt project directly, understands metrics defined in code
- **Infrastructure as code** — Terraform deployment consistent with the rest of the stack
- **Metrics as code** — version-controlled in the dbt project, not scattered across UI configs
- **Modern architecture** — fits with the dlt, Prefect, dbt philosophy
- **Upgrade path** — can migrate to Lightdash Cloud or enterprise tools (Tableau, Looker) later

Lightdash is not the only option — the section covers when to choose Omni, Metabase, Snowsight, or enterprise tools instead.

## What You Will Build

By the end of this section:

**Quick Win (Snowsight):**
```
Snowflake
└── Snowsight dashboards
    ├── Exchange rates exploration
    └── Product catalogue queries
    └── (No additional infrastructure needed)
```

**Full Implementation (Lightdash):**
```
dbt-transform repository
└── models/
    └── marts/
        └── core/
            ├── fct_exchange_rates.yml  ← Lightdash metrics defined here
            └── dim_products.yml        ← Dimensions and measures

Snowflake infrastructure
├── SVC_LIGHTDASH service account
│   └── SVC_LIGHTDASH dedicated role
│       └── Granted ANALYTICS_REPORTER (read-only REPORTING schema)
│
└── ANALYTICS database
    └── REPORTING schema
        └── (BI-ready models published by dbt)

AWS infrastructure (self-hosted option)
├── ECS cluster
│   ├── Lightdash server service
│   └── Lightdash scheduler service
├── RDS PostgreSQL (metadata)
└── ALB with HTTPS
```

Or for Lightdash Cloud:
```
Lightdash Cloud
├── Project connected to GitHub (dbt-transform repo)
├── Snowflake connection (SVC_LIGHTDASH)
└── Dashboards and metrics (synced from dbt)
```

## Section Contents

| Page | What You Will Do |
|------|-----------------|
| [BI Tool Landscape](1-bi-tool-landscape.md) | Survey all major BI tools (enterprise, modern, open source, cloud-native) |
| [Choosing a BI Tool](2-choosing-a-bi-tool.md) | Evaluation criteria and decision framework |
| [Deployment Options](3-deployment-options.md) | SaaS vs self-hosted trade-offs |
| [Snowflake Snowsight](4-snowflake-snowsight.md) | Quick dashboards with Snowflake's built-in tool (free!) |
| [Snowflake Infrastructure](5-snowflake-infrastructure.md) | Create `SVC_LIGHTDASH` service account via Terraform |
| [Lightdash Setup](6-lightdash-setup.md) | Lightdash Cloud quick start |
| [Self-Hosted Lightdash](7-self-hosted-lightdash.md) | (Optional) ECS deployment for cost-effective self-hosting |
| [Connect dbt Project](8-connect-dbt-project.md) | Link Lightdash to GitHub dbt repository |
| [Define Metrics](9-define-metrics.md) | Add metrics to dbt models |
| [Build Dashboards](10-build-dashboards.md) | Create exchange rates and products dashboards |
| [Ad-Hoc Analytics with Notebooks](11-notebooks.md) | Jupyter, Snowflake Notebooks, exploratory analysis |
| [Finishing Up](12-finishing-up.md) | When to upgrade to enterprise tools, alternatives, next steps |

## Prerequisites

Before starting this section, ensure you have completed:

- [x] [Data Warehouse](../data-warehouse/index.md) — ANALYTICS database, REPORTING schema, ANALYTICS_REPORTER role
- [x] [Data Transformation](../data-transformation/index.md) — dbt models deployed, MARTS and REPORTING schemas populated

The transformation layer must be built and producing data before you can visualise it. If you have not yet built dbt models, complete the Data Transformation section first.

## Deployment Options Overview

You have several deployment approaches for the analytics layer:

### Snowsight (Built-in, Free)

- **Cost**: $0 (included with Snowflake)
- **Setup time**: Immediate (already have access)
- **Best for**: Quick SQL exploration, ad-hoc queries, basic dashboards
- **Limitations**: Basic visualisations, no metrics as code, requires SQL

### Snowflake Notebooks (Built-in, Free)

- **Cost**: $0 (included with Snowflake, uses warehouse compute)
- **Setup time**: Immediate (built into Snowsight)
- **Best for**: Python exploratory analysis, ML, data science workflows
- **Limitations**: Only Python, fewer libraries than full Jupyter, warehouse compute costs

### Lightdash Open Source (Self-Hosted)

- **Cost**: ~$25-40/month (ECS infrastructure)
- **Setup time**: 1-2 hours (Terraform deployment)
- **Best for**: dbt-native teams, cost-conscious, want infrastructure control
- **Limitations**: Requires self-hosting knowledge, basic visualisations

### Lightdash Cloud (Managed)

- **Cost**: $2400/month (unlimited users)
- **Setup time**: 30 minutes (SaaS signup + connect)
- **Best for**: Teams wanting managed dbt-native BI without self-hosting
- **Limitations**: Expensive ($2400/month flat rate), no free cloud tier

### Omni (Cloud Only)

- **Cost**: $20-50/user/month (3 user minimum = $60/month)
- **Setup time**: 30 minutes
- **Best for**: dbt-native teams wanting polished UX
- **Limitations**: No self-hosted option, relatively expensive

### Metabase (Open Source or Cloud)

- **Cost**: Free (self-hosted) or $85/user/month (cloud)
- **Setup time**: 1-2 hours (self-hosted)
- **Best for**: General-purpose BI, mature open source
- **Limitations**: No dbt integration, metrics in UI not code

### Enterprise Tools (Tableau, Power BI, Looker)

- **Cost**: $30-150/user/month
- **Setup time**: Days to weeks
- **Best for**: Large organisations, complex visualisations, existing contracts
- **Limitations**: Expensive, no native dbt integration (except Looker)

Full comparison tables and evaluation criteria are in [BI Tool Landscape](1-bi-tool-landscape.md) and [Choosing a BI Tool](2-choosing-a-bi-tool.md).

## Cost Summary

| Approach | Monthly Cost | Infrastructure | Notes |
|----------|-------------|----------------|-------|
| **Snowsight** | $0 | None | Free with Snowflake account |
| **Snowflake Notebooks** | $0 + compute | None | Python notebooks in Snowsight, uses warehouse |
| **Jupyter (local)** | Free | local | Open source |
| **Jupyter (self-hosted)** | ~$10-15 | EC2 or local | Open source, Snowflake connector |
| **Hex** | $70+/user | None | Notebooks + dashboards, collaborative |
| **Lightdash Self-Hosted** | ~$25-40 | ECS + RDS | Terraform-managed, dbt-native |
| **Lightdash Cloud** | $2400 | None | Flat rate, unlimited users, dbt-native, managed service |
| **Omni** | $60-150 | None | 3 user minimum, cloud only, dbt-native |
| **Metabase Self-Hosted** | ~$20-30 | ECS + RDS | Mature, no dbt integration |
| **Metabase Cloud** | $85/user | None | Managed, no dbt integration |
| **Enterprise (Tableau/etc)** | $70-150/user | Varies | Complex visualisations, no dbt integration |

Detailed cost breakdowns are in [Deployment Options](3-deployment-options.md) and the [Costs](../../getting-started/account-setup/costs.md) page.

## dbt-Native vs General-Purpose Tools

A key decision is whether you need **native dbt integration**:

### dbt-Native Tools (Lightdash, Omni)

**Pros:**
- Read your dbt project directly from GitHub
- Metrics defined in dbt YAML (version-controlled, testable)
- Understand dbt lineage, tests, and documentation
- Single source of truth for business logic
- Can use dbt Cloud semantic layer (Team+ plan)

**Cons:**
- Smaller ecosystem than general-purpose tools
- Fewer visualization options
- Requires dbt knowledge

**Best for**: Teams that have built a dbt transformation layer and want metrics as code.

### General-Purpose Tools (Metabase, Tableau, Power BI)

**Pros:**
- Mature products with extensive features
- Rich visualisation libraries
- Large communities and support
- No dbt knowledge required
- Work with any database

**Cons:**
- Metrics defined in UI (not version-controlled)
- No awareness of dbt models, tests, or lineage
- Business logic can diverge from dbt
- Manual setup for each table

**Best for**: Teams without dbt, or teams that want maximum visualisation flexibility.

## When to Use Each Layer

The analytics layer has four tiers, each serving different use cases:

| Tool | User Type | Use Case | SQL/Python Required? |
|------|-----------|----------|----------------------|
| **Notebooks** | Data scientists, analysts | Exploratory analysis, ML, complex transformations | Python/SQL |
| **Snowsight** | Analysts, engineers | Ad-hoc SQL queries, debugging, quick charts | SQL |
| **dbt-native BI** | Analysts, managers | Dashboards, metrics tracking, self-service | No (uses dbt metrics) |
| **Enterprise BI** | All stakeholders | Complex visualisations, embedded analytics | No |

Most organisations use multiple tools:
- **Notebooks** (Snowflake Notebooks, Jupyter, Hex) for exploratory data analysis and data science
- **Snowsight** for technical users doing SQL exploration
- **Lightdash/Omni** for self-service dashboards and operational reporting
- **Tableau/Looker** (optional) for executive reporting and embedded analytics

This section covers all four approaches so you can choose the right fit.

## Dashboards vs Notebooks

Two distinct paradigms serve different analytics needs:

### Dashboards (Snowsight, Lightdash, Tableau)

**Purpose**: Monitoring and reporting on known questions

- Pre-built visualizations that update automatically
- Business users can filter and drill down without code
- Answers "How is the business performing?" (known KPIs)
- Static structure, dynamic data

**Example use cases:**
- Executive KPI dashboard
- Sales team pipeline report
- Marketing campaign performance

### Notebooks (Jupyter, Snowflake Notebooks, Hex)

**Purpose**: Exploratory analysis and answering new questions

- Code-first environment (Python, SQL, R)
- Iterative investigation with narrative documentation
- Answers "Why did this happen?" or "What should we build?"
- Dynamic structure, often one-time or periodic analysis

**Example use cases:**
- Root cause analysis for metric anomalies
- Exploratory data analysis for new data sources
- Machine learning model development
- Custom analyses for stakeholder requests

You need both: dashboards for operational reporting, notebooks for deep investigation.

## Get Started

Start with the BI tool landscape to understand your options:

Continue to [BI Tool Landscape](1-bi-tool-landscape.md) →
