# Batch Data Ingestion with dlt

In this section, you'll build data pipelines using dlt (data load tool) to ingest data from APIs and databases into your Snowflake data warehouse. These pipelines are orchestrated by Prefect, which you set up in the previous section.

## What is Batch Data Ingestion?

Batch data ingestion is the process of extracting data from source systems and loading it into your data warehouse on a schedule. Unlike streaming ingestion (which processes data continuously), batch ingestion runs at defined intervals - hourly, daily, or weekly.

Common batch ingestion patterns:

- **API extraction** - Pull data from REST APIs (e.g. exchange rates, weather, SaaS platforms)
- **Database replication** - Copy data from operational databases (e.g. PostgreSQL, MySQL)
- **File processing** - Load data from files in S3 or other storage

## Why dlt?

dlt is a Python library that simplifies data pipeline development. It handles the complex parts of data loading - schema management, incremental loading, and destination-specific optimisations - so you can focus on your data.

| Feature | dlt | Custom Python | Fivetran/Airbyte |
|---------|-----|---------------|------------------|
| **Setup** | `pip install dlt` | Write from scratch | SaaS signup or self-host |
| **Schema handling** | Automatic inference and evolution | Manual DDL | Automatic |
| **Incremental loading** | Built-in with state management | Manual implementation | Built-in |
| **Cost** | Free (open source) | Engineering time | $1+ per MAR or hosting costs |
| **Customisation** | Full Python control | Full control | Limited to connectors |
| **Connectors** | Growing library + easy custom sources | Build everything | 300+ pre-built |

dlt is ideal when you need custom extraction logic or want to avoid SaaS costs. For standard SaaS sources (Salesforce, HubSpot, Stripe), consider Airbyte which has pre-built connectors.

## Role in the Stack

dlt sits in the ingestion layer, loading raw data into dedicated loader databases:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DATA SOURCES                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐              │
│  │   REST APIs     │  │   Databases     │  │   Files (S3)    │              │
│  │  (Exchange      │  │  (PostgreSQL,   │  │  (CSV, JSON,    │              │
│  │   Rates, etc.)  │  │   MySQL, etc.)  │  │   Parquet)      │              │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘              │
│           │                    │                    │                       │
│           └────────────────────┼────────────────────┘                       │
│                                ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────┐        │
│  │                         dlt Pipelines                           │        │
│  │     (Orchestrated by Prefect - retries, scheduling, alerts)     │        │
│  └─────────────────────────────────────────────────────────────────┘        │
│                                │                                            │
│                ┌───────────────┴───────────────┐                            │
│                ▼                               ▼                            │
│  ┌─────────────────────────┐     ┌─────────────────────────┐                │
│  │     DLT Database        │     │   SNOWPIPE Database     │                │
│  │  (dlt-loaded tables)    │     │  (auto-ingest from S3)  │                │
│  └─────────────────────────┘     └─────────────────────────┘                │
│                                                                             │
│                                │                                            │
│                                ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────┐        │
│  │                    dbt Transformations                          │        │
│  │              (covered in Data Transformation section)           │        │
│  └─────────────────────────────────────────────────────────────────┘        │
│                                │                                            │
│                                ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────┐        │
│  │                     ANALYTICS Database                          │        │
│  │                   (transformed, analysis-ready)                 │        │
│  └─────────────────────────────────────────────────────────────────┘        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## What You'll Build

In this section, you'll create four data pipelines demonstrating different ingestion patterns:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        BATCH DATA INGESTION                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Pipeline 1: Currencies (API → Snowflake direct)                            │
│  ┌─────────────────┐        ┌─────────────────┐        ┌─────────────────┐  │
│  │ Open Exchange   │  dlt   │    Snowflake    │ Prefect│  Weekly refresh │  │
│  │ Rates API       │ ────▶  │ DLT.OPEN_       │ ◀───── │  replace mode   │  │
│  │ /currencies.json│        │ EXCHANGE_RATES  │        │                 │  │
│  └─────────────────┘        └─────────────────┘        └─────────────────┘  │
│                                                                             │
│  Pipeline 2: Exchange Rates (API → S3 → Snowpipe → Snowflake)               │
│  ┌─────────────────┐        ┌─────────────────┐        ┌─────────────────┐  │
│  │ Open Exchange   │  dlt   │   S3 Bucket     │Snowpipe│   SNOWPIPE.     │  │
│  │ Rates API       │ ────▶  │ /exchange-rates/ │ ────▶  │ OPEN_EXCHANGE_  │  │
│  │ /historical/    │        │                 │        │ RATES           │  │
│  └─────────────────┘        └─────────────────┘        └─────────────────┘  │
│                                                                             │
│  Pipeline 3: Products (PostgreSQL → Snowflake)                              │
│  ┌─────────────────┐        ┌─────────────────┐        ┌─────────────────┐  │
│  │ Clever Cloud    │  dlt   │    Snowflake    │ Prefect│  Daily 06:00    │  │
│  │ PostgreSQL      │ ────▶  │ DLT.APPLICATION │ ◀───── │  Type 2 SCD     │  │
│  │ (Products)      │        │ _DATA           │        │                 │  │
│  └─────────────────┘        └─────────────────┘        └─────────────────┘  │
│                                                                             │
│  Pipeline 4: HubSpot (API → Snowflake)                                      │
│  ┌─────────────────┐        ┌─────────────────┐        ┌─────────────────┐  │
│  │ HubSpot CRM     │  dlt   │    Snowflake    │ Prefect│  Daily 07:00    │  │
│  │ (contacts)      │ ────▶  │ DLT.HUBSPOT     │ ◀───── │  incremental    │  │
│  └─────────────────┘        └─────────────────┘        └─────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Key patterns demonstrated:**

- Direct API to Snowflake loading with dlt (simplest pattern)
- S3 staging with Snowpipe auto-ingestion for high-volume appends
- Database extraction with Type 2 SCD handling
- Prefect orchestration with schedules, retries, and alerting
- Incremental loading with merge keys for idempotency

## Prerequisites

Before starting this section, ensure you have completed:

- [x] [Prefect Setup](../orchestration/index.md) - Orchestration layer ready (Cloud or self-hosted)
- [x] [S3 Data Lake](../aws/1-s3-data-lake.md) - S3 buckets for staging data
- [x] [Data Warehouse](../data-warehouse/index.md) - Snowflake with storage integrations

You'll also need:

- [x] Open Exchange Rates account (free tier available)
- [x] Clever Cloud account with PostgreSQL database (free DEV tier)

## Section Overview

| Page | What You'll Learn |
|------|-------------------|
| [1. dlt Concepts](1-dlt-concepts.md) | Core concepts: sources, resources, pipelines, destinations |
| [2. Project Setup](2-project-setup.md) | Repository structure, uv, pyproject.toml, VaultDocProvider |
| [3. Snowflake Infrastructure](3-snowflake-infrastructure.md) | DLT and SNOWPIPE databases, dedicated roles, Snowpipe module |
| [4. Credentials Setup](4-credentials-setup.md) | API keys and database credentials in AWS Secrets Manager |
| [5. Currencies Pipeline](5-currencies-pipeline.md) | Your first dlt pipeline — reference data direct to Snowflake |
| [6. Exchange Rates to S3](6-exchange-rates-to-s3.md) | Historical data via S3 staging and Snowpipe auto-ingestion |
| [7. Products Pipeline](7-products-pipeline.md) | Database extraction with incremental loading |
| [8. Prefect Orchestration](8-prefect-orchestration.md) | Wrap pipelines in Prefect flows with schedules and alerting |
| [9. Finishing Up](9-finishing-up.md) | Verification and next steps |
| [10. HubSpot Pipeline](10-hubspot-pipeline.md) | CRM data ingestion with the dlt HubSpot connector |

## Cost Considerations

This section uses mostly free or low-cost services:

| Component | Cost |
|-----------|------|
| dlt | Free (open source) |
| Open Exchange Rates | Free tier: 1,000 requests/month |
| Clever Cloud PostgreSQL | Free DEV tier |
| Snowpipe | ~$0.06 per 1,000 files processed |
| S3 storage | ~$0.023/GB/month |

The main costs are your existing Snowflake compute (LOADING warehouse) and any Prefect infrastructure.

## What's Next

Start by understanding the core dlt concepts - sources, resources, pipelines, and destinations. This foundation applies to all the pipelines you'll build.

Continue to [dlt Concepts](1-dlt-concepts.md) →
