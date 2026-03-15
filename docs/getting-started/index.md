# Welcome

This guide walks you through building a production-grade modern data stack from scratch - the kind of architecture that handles real business data reliably, at scale, and without requiring a large engineering team to maintain it.

## What Is a Modern Data Stack?

A "modern data stack" is a collection of cloud-native tools that work together to move data from source systems to business insights. Each tool does one job well, and they compose together to form a complete data platform.

The stack has several distinct layers. Data is collected from sources, ingested into storage, transformed into clean models, and finally consumed by analysts and dashboards - with orchestration, observability, and DevOps concerns running across all of it.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         THE MODERN DATA STACK                                   │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  SOURCES                                                                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │  Events     │  │  Databases  │  │  REST APIs  │  │  SaaS Tools │             │
│  │  (Kafka)    │  │ (PostgreSQL)│  │  (REST)     │  │  (HubSpot)  │             │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │
│         │                │                │                │                    │
│  INGESTION               │                │                │                    │
│         │          ┌─────┴───────────┐    │         ┌──────┴──────┐             │
│         │          │   dlt           │◀───┘         │   Airbyte   │             │
│         │          │ (APIs, DBs)     │              │ (SaaS)      │             │
│         │          └─────────────────┘              └─────────────┘             │
│         │                    │                             │                    │
│  ┌──────▼──────┐             │                             │                    │
│  │ Kafka       │             │                             │                    │
│  │ Connect     │             │                             │                    │
│  └──────┬──────┘             │                             │                    │
│         │                    │                             │                    │
│  STORAGE                     ▼                             ▼                    │
│         │          ┌─────────────────────────────────────────────────┐          │
│         ├─────────▶│                  SNOWFLAKE                      │          │
│         │          │  STREAMING │ DLT │ SNOWPIPE │ AIRBYTE databases │          │
│         │          └───────────────────────┬─────────────────────────┘          │
│         │                                  │                                    │
│  ┌──────▼──────┐                           │  TRANSFORMATION                    │
│  │    S3       │                           ▼                                    │
│  │ (Raw files) │               ┌───────────────────────┐                        │
│  └─────────────┘               │           dbt         │                        │
│                                │  staging → marts      │                        │
│                                └───────────┬───────────┘                        │
│                                            │                                    │
│  ANALYTICS                                 ▼                                    │
│                     ┌───────────────────────────────────────────────┐           │
│                     │  Lightdash dashboards │ Snowsight │ Notebooks │           │
│                     └───────────────────────────────────────────────┘           │
│                                                                                 │
│  ─────────────────────────────────────────────────────────────────────────────  │
│  CROSS-CUTTING CONCERNS                                                         │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐               │
│  │   Orchestration  │  │  Observability   │  │    DevOps        │               │
│  │   (Prefect)      │  │ (Elementary,     │  │ (Terraform,      │               │
│  │                  │  │  OpenMetadata)   │  │  GitHub Actions) │               │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘               │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## The Layers

### Ingestion

Data enters the stack in two ways:

**Batch ingestion** runs on a schedule - hourly, daily, or weekly. Two tools handle this:

- **dlt** (data load tool) - a Python library for building pipelines against APIs and databases. It handles schema inference, incremental loading, and direct-to-Snowflake delivery. It is free, open source, and requires no infrastructure beyond a Prefect worker.
- **Airbyte** - a managed connector platform with hundreds of pre-built connectors for SaaS tools. It is the right choice when you need to ingest from many SaaS sources without writing custom extraction code.

**Streaming ingestion** processes events in real time - within seconds of them occurring. **Kafka** (via Confluent Cloud or AWS MSK) handles the event stream, and **Kafka Connect** sinks events directly into Snowflake as they arrive.

### Storage

All data lands in **Snowflake**, a cloud-native data warehouse running on AWS. Snowflake separates compute from storage, so you pay only for the queries you run and the data you store.

Raw data from each ingestion tool lands in its own database:

| Database | Populated By | Contents |
|----------|-------------|----------|
| `STREAMING` | Kafka Connect | Real-time events (purchases) |
| `DLT` | dlt pipelines | API and database data (products, exchange rates) |
| `SNOWPIPE` | Snowpipe (S3) | Files staged via S3 |
| `AIRBYTE` | Airbyte | SaaS data (CRM records) |

**S3** holds raw files as an immutable archive before they are loaded into Snowflake via Snowpipe.

### Transformation

**dbt** (data build tool) transforms raw Snowflake data into clean, tested, analytics-ready models. It works entirely within Snowflake using SQL - no data moves outside the warehouse during transformation.

dbt models follow a layered pattern:

```
Raw databases  →  Staging (stg_*)  →  Intermediate (int_*)  →  Marts (fct_*, dim_*)  →  Reporting
```

The output is a clean `ANALYTICS` database with `MARTS` and `REPORTING` schemas that BI tools query directly.

### Analytics

**Lightdash** is a dbt-native BI tool - metrics are defined in your dbt YAML files, and Lightdash reads them directly. This means your analytics definitions live in version-controlled code, not hidden in a dashboard tool.

**Snowsight** (Snowflake's built-in UI) is available for ad-hoc queries and quick visualisations without any additional setup.

**Python notebooks** (Snowflake Notebooks or Jupyter) support exploratory analysis and ML workloads.

### Orchestration

**Prefect** coordinates everything. It schedules and monitors all ingestion pipelines, triggers dbt runs after ingestion completes, and handles retries, alerting, and logging. Every pipeline in this stack - dlt, Airbyte, dbt - runs inside a Prefect flow.

### Observability

**Elementary** extends dbt's built-in testing to provide anomaly detection and a data observability dashboard, tracking data freshness, row counts, and schema changes over time.

**OpenMetadata** provides a data catalogue - a central place to discover what tables exist, where they came from, who owns them, and how they relate to each other.

### DevOps

**Terraform** manages all infrastructure as code: Snowflake warehouses, databases, roles, and users; AWS S3 buckets, IAM roles, and Secrets Manager; Confluent Cloud topics; and GitHub repository settings. Nothing is created manually.

**GitHub Actions** runs CI/CD pipelines: validating Terraform plans, running dbt tests, deploying Prefect flows, and publishing this documentation site.

## The Tool Choices

This stack is opinionated. Here is why each tool was chosen:

| Layer | Tool | Why |
|-------|------|-----|
| Batch ingestion (APIs/DBs) | dlt | Free, Python-native, no infrastructure |
| Batch ingestion (SaaS) | Airbyte | Pre-built connectors, low-code configuration |
| Streaming | Confluent Cloud | Managed Kafka with Schema Registry included |
| Warehouse | Snowflake | Separation of compute/storage, Terraform support, ecosystem |
| Transformation | dbt | SQL-native, version-controlled, tested models |
| BI | Lightdash | Metrics defined in dbt YAML, not duplicated in the tool |
| Orchestration | Prefect | Modern Python-native replacement for Airflow |
| Observability | Elementary + OpenMetadata | dbt-native testing + full catalogue |
| Infrastructure | Terraform | Everything as code, reproducible, auditable |

Where possible, managed cloud options are covered alongside self-hosted alternatives, so you can choose based on your budget and operational preferences.

## What You're Building

This guide uses a sales analytics use case to make the stack concrete. By the end, you will have:

- **Four data sources**: purchase events (streaming), product catalogue (PostgreSQL), exchange rates (API), and customer data (HubSpot CRM)
- **Three ingestion pipelines**: Kafka Connect for streaming purchases, dlt for products and exchange rates, and Airbyte for HubSpot
- **Four dbt models**: `fact_purchases`, `dim_products`, `fct_exchange_rates`, `dim_customers`
- **One sales report**: joining all four dimensions and fact tables into a `sales` mart with customer-level aggregations in GBP and USD
- **Full observability**: data quality tests, anomaly detection, a data catalogue, and pipeline monitoring

## How This Guide Is Structured

The guide is split into three parts:

**Getting Started** (you are here) - sets up the foundations: your development environment, cloud accounts, and Terraform infrastructure. Everything here is a prerequisite for the Build section.

**[Build](../build/index.md)** - walks through each component of the stack in dependency order. Each section explains the concepts, guides you through the Terraform and configuration, and ends with a summary of what you've built.

**Maintain** - day-to-day operations: adding users, adding data sources, handling incidents, and keeping the stack up to date.

Follow these sections in order for the smoothest experience.

## What's Next

Start by setting up your development environment and cloud accounts.

1. **[Initial Setup](initial-setup/index.md)** - GitHub organisation, local tools, development workflow, secrets management, and Claude Code setup
2. **[Account Setup](account-setup/index.md)** - AWS, Snowflake, and Prefect accounts
3. **[Terraform Setup](terraform-setup/index.md)** - Remote state, CI/CD, and managing GitHub, AWS, and Snowflake via Terraform

Continue to [Initial Setup](initial-setup/index.md) →
