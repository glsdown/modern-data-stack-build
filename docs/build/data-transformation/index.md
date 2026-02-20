# Data Transformation

On this page, you will:

- [x] Understand where dbt fits in the modern data stack
- [x] Learn how the transformation layer connects raw data to analytics
- [x] Plan the infrastructure and repository structure needed

## Overview

This section covers building the transformation layer using [dbt](https://www.getdbt.com/) (data build tool). dbt transforms the raw data loaded by dlt, Snowpipe, and Airbyte into clean, tested, analytics-ready models.

The transformation layer lives in its own repository — separate from your Prefect pipelines. This separation reflects the different ownership and release cadence of analytics engineering work (iterative model development, schema changes) versus data engineering work (pipeline reliability, infrastructure).

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                       DATA TRANSFORMATION LAYER                                │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  Raw Data (Snowflake)          dbt-transform repo           Analytics          │
│  ────────────────────          ──────────────────           ─────────          │
│                                                                                │
│  ┌──────────────────┐          ┌──────────────────┐                            │
│  │ DLT database     │          │ Staging models   │                            │
│  │ • currencies     │─────────▶│ stg_dlt__*       │                            │
│  │ • exchange_rates │          │ stg_airbyte__*   │         ┌────────────────┐ │
│  │ • products       │          └──────────────────┘         │  ANALYTICS     │ │
│  └──────────────────┘                   │                   │  database      │ │
│                                         ▼                   │                │ │
│  ┌──────────────────┐          ┌──────────────────┐         │  STAGING       │ │
│  │ SNOWPIPE database│          │ Intermediate     │────────▶│  INTERMEDIATE  │ │
│  │ • exchange_rates │─────────▶│ int_*            │         │  MARTS         │ │
│  └──────────────────┘          └──────────────────┘         │  REPORTING     │ │
│                                         │                   └────────────────┘ │
│  ┌──────────────────┐                   ▼                                      │
│  │ AIRBYTE database │          ┌──────────────────┐                            │
│  │ • hubspot        │─────────▶│ Mart models      │                            │
│  └──────────────────┘          │ fct_*, dim_*     │                            │
│                                └──────────────────┘                            │
│                                                                                │
│  Orchestration                                                                 │
│  ─────────────                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐       │
│  │  Prefect (data-pipelines repo)                                      │       │
│  │  • Waits for ingestion flows to complete                            │       │
│  │  • Triggers dbt run via dbt Core CLI or dbt Cloud API               │       │
│  │  • Alerts on failure                                                │       │
│  └─────────────────────────────────────────────────────────────────────┘       │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
```

## The Transformation Repository

The dbt project lives in a dedicated `dbt-transform` repository. This is separate from the `data-pipelines` repository that contains Prefect flows. Prefect can trigger dbt runs, but it does not own the dbt codebase.

```
GitHub Organisation
├── terraform/               ← Infrastructure as code
├── data-pipelines/          ← Prefect flows (ingestion orchestration)
└── dbt-transform/           ← dbt project (this section)
```

This separation means:

- **Analytics engineers** can develop models, run tests, and deploy changes without touching Prefect code
- **Data engineers** can update pipeline infrastructure without affecting model logic
- **CI/CD is independent**: dbt's slim CI runs on dbt code changes; Prefect deployments run on flow changes

## Schema Model

The transformation layer writes to four schemas in the `ANALYTICS` database (and their `ANALYTICS_DEV` counterparts for development):

| Schema | Created by | Contains | Access |
|--------|-----------|----------|--------|
| `STAGING` | dbt | One-to-one views/tables over raw sources | `ANALYTICS_DEVELOPER` |
| `INTERMEDIATE` | dbt | Business logic, joined datasets | `ANALYTICS_DEVELOPER` |
| `MARTS` | dbt | Final analytics tables (fct/dim) | `ANALYTICS_DEVELOPER` |
| `REPORTING` | Terraform (already exists) | Curated subset for BI tools | `ANALYTICS_REPORTER` |

The `STAGING`, `INTERMEDIATE`, and `MARTS` schemas are created automatically by dbt when models run. Terraform's database-level future grants on `ANALYTICS` already cover permissions for these schemas — no additional Terraform configuration is needed.

The `REPORTING` schema was created in the [Schemas](../data-warehouse/6-schemas.md) page and has schema-level grants that restrict `ANALYTICS_REPORTER` to just this schema. dbt publishes final BI-ready models here.

## What You Will Build

By the end of this section:

```
dbt-transform repository
├── models/
│   ├── staging/
│   │   ├── dlt/
│   │   │   ├── stg_dlt__currencies.sql
│   │   │   ├── stg_dlt__exchange_rates.sql
│   │   │   └── stg_dlt__products.sql
│   │   └── airbyte/
│   │       └── stg_airbyte__contacts.sql
│   ├── intermediate/
│   │   └── int_exchange_rates__unioned.sql
│   └── marts/
│       ├── core/
│       │   ├── fct_exchange_rates.sql
│       │   └── dim_products.sql
│       └── crm/
│           └── fct_contacts.sql
├── tests/
├── macros/
├── seeds/
├── dbt_project.yml
└── packages.yml
```

Snowflake infrastructure:

```
Snowflake
├── SVC_DBT service account
│   └── SVC_DBT dedicated role
│       └── Granted ANALYTICS_TRANSFORMER
│
└── ANALYTICS database (existing)
    ├── STAGING schema     ← created by dbt
    ├── INTERMEDIATE schema ← created by dbt
    ├── MARTS schema       ← created by dbt
    └── REPORTING schema   ← already in Terraform
```

## Deployment Options

You have two options for running dbt:

| | dbt Core | dbt Cloud |
|--|----------|-----------|
| **Cost** | Free (open source) | Free (1 developer seat), $100/seat/month (Team) |
| **IDE** | Your own editor + dbt CLI | Browser-based IDE |
| **Scheduling** | Via Prefect | Via dbt Cloud jobs or Prefect |
| **Docs site** | Self-hosted (S3 + CloudFront) | Hosted by dbt Cloud |
| **Deferral** | `--defer --state` with S3 artifacts | Native in dbt Cloud |
| **Semantic layer** | Not available | Team/Enterprise only |
| **Best for** | Code-first teams with existing Prefect | Teams wanting a managed analytics platform |

Both options are covered in full. Start with [dbt Core vs dbt Cloud](2-deployment-options.md) to choose your approach.

## Section Contents

| Page | What You Will Do |
|------|-----------------|
| [dbt Concepts](1-dbt-concepts.md) | Learn models, sources, tests, macros, and materialisations |
| [dbt Core vs dbt Cloud](2-deployment-options.md) | Compare options and choose your deployment |
| [Project Setup](3-project-setup.md) | Create the `dbt-transform` repo and configure packages |
| [Snowflake Infrastructure](4-snowflake-infrastructure.md) | Create `SVC_DBT` service account via Terraform |
| [Sources and Staging](5-sources-and-staging.md) | Define sources and build staging models |
| [Intermediate Models](6-intermediate-models.md) | Build business logic layer |
| [Mart Models](7-mart-models.md) | Build final analytics tables |
| [Testing and Documentation](8-testing-and-documentation.md) | Add tests, descriptions, and enforce project standards |
| [dbt Core Deployment](9-dbt-core-deployment.md) | CI/CD, state deferral, and hosted docs site |
| [dbt Cloud Setup](10-dbt-cloud-setup.md) | Environments, jobs, IDE, and native deferral |
| [Prefect Orchestration](11-prefect-orchestration.md) | Trigger dbt from Prefect after ingestion completes |
| [Finishing Up](12-finishing-up.md) | Verification, architecture summary, next steps |

## Prerequisites

Before starting this section, ensure you have completed:

- [x] [Data Warehouse](../data-warehouse/index.md) — ANALYTICS and ANALYTICS_DEV databases, ANALYTICS_TRANSFORMER role, TRANSFORMING warehouse, REPORTING schema
- [x] [Batch Data Ingestion](../batch-data-ingestion/index.md) — DLT and SNOWPIPE databases with raw data
- [x] [SaaS Ingestion](../saas-ingestion/index.md) — AIRBYTE database with HubSpot data
- [x] [Orchestration](../orchestration/index.md) — Prefect for triggering dbt runs

## Cost Summary

| Approach | Monthly Cost | Notes |
|----------|-------------|-------|
| **dbt Core** | $0 | Open source; S3 for state artifacts (~$0.01/month) |
| **dbt Cloud (Developer)** | $0 | 1 seat free, limited to one project |
| **dbt Cloud (Team)** | $100/seat/month | Multi-seat, semantic layer, SSO |

Detailed cost breakdowns are in [dbt Core vs dbt Cloud](2-deployment-options.md) and the [Costs](../../getting-started/account-setup/costs.md) page.

## Get Started

Continue to [dbt Concepts](1-dbt-concepts.md) →
