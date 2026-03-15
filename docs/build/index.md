# Build Your Modern Data Stack

This section guides you through building each component of the stack in order. By the end, you will have a fully operational data platform: data flowing from four sources, through ingestion pipelines and Snowflake, transformed by dbt, and visible in Lightdash dashboards.

## What Gets Built

Each section builds a distinct layer of the stack. Together they form a complete, production-ready data platform:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    WHAT YOU BUILD IN THIS SECTION                               │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  ① AWS Infrastructure          S3 data lake, VPC networking                    │
│  ┌─────────────────────────────────────────────────────────────────────┐        │
│  │  S3 (raw file archive)  │  VPC (network foundation for self-hosted) │        │
│  └─────────────────────────────────────────────────────────────────────┘        │
│                                    │                                           │
│                                    ▼                                           │
│  ② Data Warehouse              Snowflake foundation                             │
│  ┌─────────────────────────────────────────────────────────────────────┐        │
│  │  Warehouses │ Databases │ Roles │ Users │ Schemas │ Network Policies │        │
│  └─────────────────────────────────────────────────────────────────────┘        │
│                                    │                                           │
│                                    ▼                                           │
│  ③ Orchestration               Prefect control plane                            │
│  ┌─────────────────────────────────────────────────────────────────────┐        │
│  │  Prefect Cloud (or self-hosted)  │  Work pools  │  CI/CD deployment │        │
│  └─────────────────────────────────────────────────────────────────────┘        │
│                     │                │                    │                    │
│                     ▼                ▼                    ▼                    │
│  ④ Batch Ingestion  ⑤ SaaS Ingestion  ⑥ Streaming                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────────┐           │
│  │ dlt          │  │ Airbyte      │  │ Confluent Cloud / MSK        │           │
│  │ • products   │  │ • HubSpot    │  │ • Kafka topics               │           │
│  │ • currencies │  │   (CRM)      │  │ • Kafka Connect → Snowflake  │           │
│  │ • Snowpipe   │  │ • Reverse ETL│  │ • Purchase events            │           │
│  └──────┬───────┘  └──────┬───────┘  └─────────────┬────────────────┘           │
│         │                │                         │                           │
│         ▼                ▼                         ▼                           │
│  ┌──────────────────────────────────────────────────────────┐                   │
│  │                      SNOWFLAKE                           │                   │
│  │  DLT database │ AIRBYTE database │ STREAMING database    │                   │
│  └──────────────────────────────────┬───────────────────────┘                   │
│                                     │                                          │
│                                     ▼                                          │
│  ⑦ Data Transformation          dbt models                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐        │
│  │  stg_* staging  │  int_* intermediate  │  fct_* / dim_* marts       │        │
│  │  → ANALYTICS database (STAGING, INTERMEDIATE, MARTS, REPORTING)     │        │
│  └─────────────────────────────────────────────────────────────────────┘        │
│                                     │                                          │
│                                     ▼                                          │
│  ⑧ Data Analytics              Dashboards and notebooks                         │
│  ┌─────────────────────────────────────────────────────────────────────┐        │
│  │  Lightdash dashboards  │  Snowsight  │  Python notebooks            │        │
│  └─────────────────────────────────────────────────────────────────────┘        │
│                                                                                 │
│  ─────────────────────────────────────────────────────────────────────────────  │
│  RUNNING ACROSS ALL SECTIONS                                                    │
│                                                                                 │
│  ⑨ Observability   Elementary │ OpenMetadata │ Prefect monitoring               │
│  ⑩ Documentation   MkDocs site │ Multirepo plugin │ Runbooks                    │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## The Data Journey

This is the specific data that flows through the stack you're building:

| Source | Tool | Raw Database | dbt Model | Final Mart |
|--------|------|-------------|-----------|------------|
| Purchase events (React app → Kafka) | Kafka Connect | `STREAMING` | `stg_streaming__purchases` | `fct_purchases` |
| Product catalogue (PostgreSQL) | dlt | `DLT` | `stg_dlt__products` | `dim_products` |
| Exchange rates (REST API) | dlt + Snowpipe | `DLT` / `SNOWPIPE` | `stg_dlt__exchange_rates` | `fct_exchange_rates` |
| Customer data (HubSpot CRM) | Airbyte | `AIRBYTE` | `stg_airbyte__contacts` | `dim_customers` |

All four are joined in a `sales` mart table:

```
fct_purchases  ──┐
dim_products   ──┤──▶  sales  (customer totals, converted to GBP and USD)
fct_exchange_rates   ──┤
dim_customers  ──┘
```

## Build Order and Dependencies

Sections must be completed in order. Each section depends on what came before:

```
Getting Started (Terraform, accounts, local environment)
        │
        ▼
① AWS Infrastructure  ─────────────────────────────────────────────────────┐
        │                                                                   │
        ▼                                                                   │
② Data Warehouse (Snowflake: warehouses, databases, roles, users)           │
        │                                                                   │
        ▼                                                                   │
③ Orchestration (Prefect: control plane, work pools, CI/CD)                 │
        │                                                                   │
        ├──────────────────────┬──────────────────────┐                     │
        ▼                      ▼                      ▼                     │
④ Batch Ingestion          ⑤ SaaS Ingestion       ⑥ Streaming              │
  (dlt pipelines)           (Airbyte)               (Kafka + Connect)       │
        │                      │                      │                     │
        └──────────────────────┴──────────────────────┘                     │
                               │                                            │
                               ▼                                            │
                    ⑦ Data Transformation (dbt models)           Uses VPC ──┘
                               │
                               ▼
                    ⑧ Data Analytics (Lightdash, Snowsight)
                               │
                               ▼
                    ⑨ Observability (Elementary, OpenMetadata)
                               │
                               ▼
                    ⑩ Documentation (MkDocs site, runbooks)
```

## Sections at a Glance

| # | Section | What You Build | Key Tools |
|---|---------|---------------|-----------|
| [①](aws/index.md) | [AWS Infrastructure](aws/index.md) | S3 data lake, VPC networking | Terraform, S3 |
| [②](data-warehouse/index.md) | [Data Warehouse](data-warehouse/index.md) | Snowflake warehouses, databases, roles, users | Terraform, Snowflake |
| [③](orchestration/index.md) | [Orchestration](orchestration/index.md) | Prefect control plane, work pools, flow deployment | Prefect, Terraform |
| [④](batch-data-ingestion/index.md) | [Batch Data Ingestion](batch-data-ingestion/index.md) | dlt pipelines for products, exchange rates; Snowpipe | dlt, Prefect |
| [⑤](saas-ingestion/index.md) | [SaaS Ingestion](saas-ingestion/index.md) | Airbyte HubSpot connector, reverse ETL | Airbyte, Prefect |
| [⑥](streaming-data-ingestion/index.md) | [Streaming Ingestion](streaming-data-ingestion/index.md) | Kafka topics, Kafka Connect sink to Snowflake | Confluent Cloud |
| [⑦](data-transformation/index.md) | [Data Transformation](data-transformation/index.md) | dbt staging, intermediate, and mart models | dbt Core / Cloud |
| [⑧](data-analytics/index.md) | [Data Analytics](data-analytics/index.md) | Lightdash dashboards, Snowsight, notebooks | Lightdash, Snowflake |
| [⑨](observability/index.md) | [Observability](observability/index.md) | Data quality monitoring, lineage, cost alerts | Elementary, OpenMetadata |
| [⑩](documentation/index.md) | [Documentation](documentation/index.md) | MkDocs site integrating all three repositories | MkDocs, GitHub Pages |

## Prerequisites

Before starting the Build section, complete all three parts of [Getting Started](../getting-started/index.md):

- [x] [Initial Setup](../getting-started/initial-setup/index.md) - local environment, GitHub organisation, development workflow
- [x] [Account Setup](../getting-started/account-setup/index.md) - AWS, Snowflake, and Prefect accounts created
- [x] [Terraform Setup](../getting-started/terraform-setup/index.md) - remote state, CI/CD, and GitHub/AWS/Snowflake providers configured

## What's Next

Start with AWS Infrastructure to create the S3 data lake and network foundation.

Continue to [AWS Infrastructure](aws/index.md) →
