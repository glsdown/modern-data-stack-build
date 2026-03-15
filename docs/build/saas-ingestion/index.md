# SaaS Data Ingestion

On this page, you will:

- [x] Understand when to use Airbyte vs dlt for SaaS data
- [x] Learn Airbyte's role in the modern data stack
- [x] Plan the infrastructure needed for SaaS ingestion

## Overview

This section covers setting up [Airbyte](https://airbyte.com/) for SaaS data ingestion. Airbyte provides 600+ pre-built connectors, a UI for non-engineers, and reverse ETL capabilities.

If you only need one or two simple SaaS sources, you may not need Airbyte at all. The [HubSpot Pipeline](../batch-data-ingestion/9-hubspot-pipeline.md) page shows how to add a SaaS connector using dlt's verified sources within your existing batch pipeline infrastructure.

This section is for when you outgrow that approach.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        SAAS DATA INGESTION LAYER                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SaaS Sources                    Airbyte                     Snowflake      │
│  ────────────                    ───────                     ─────────      │
│                                                                             │
│  ┌─────────────────┐     ┌─────────────────────────┐                        │
│  │    HubSpot      │     │                         │     ┌───────────────┐  │
│  │  (CRM data)     │────▶│    Airbyte Cloud        │────▶│   AIRBYTE.    │  │
│  │                 │     │    or Self-Hosted       │     │   HUBSPOT     │  │
│  └─────────────────┘     │                         │     └───────────────┘  │
│                          │  ┌───────────────────┐  │                        │
│  ┌─────────────────┐     │  │ HubSpot Connector │  │     ┌───────────────┐  │
│  │   Salesforce    │────▶│  │ Salesforce Conn.  │  │────▶│   AIRBYTE.    │  │
│  │   (example)     │     │  │ Stripe Connector  │  │     │   SALESFORCE  │  │
│  └─────────────────┘     │  └───────────────────┘  │     └───────────────┘  │
│                          │                         │                        │
│  ┌─────────────────┐     │  Reverse ETL:           │                        │
│  │     Stripe      │────▶│  Snowflake → SaaS       │                        │
│  │   (example)     │     │                         │                        │
│  └─────────────────┘     └─────────────────────────┘                        │
│                                     │                                       │
│                          ┌──────────▼──────────┐                            │
│                          │      Prefect        │                            │
│                          │  • Trigger syncs    │                            │
│                          │  • Monitor status   │                            │
│                          │  • Alerting         │                            │
│                          └─────────────────────┘                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## When to Use Airbyte

| Scenario | Recommended Tool |
|----------|-----------------|
| Simple REST APIs (Exchange Rates, etc.) | dlt |
| Database extraction (PostgreSQL, MySQL) | dlt |
| 1-2 SaaS sources with dlt verified sources | dlt |
| Complex SaaS (Salesforce, NetSuite, Workday) | **Airbyte** |
| Non-engineers need to configure connections | **Airbyte** |
| Reverse ETL (Snowflake → HubSpot/Salesforce) | **Airbyte** |
| 5+ SaaS sources | **Airbyte** |
| Cost-sensitive, code-first team | dlt |

### Why not Airbyte for everything?

You might wonder why the batch ingestion section uses dlt rather than Airbyte for all data sources. The reasons are:

| Consideration | dlt | Airbyte |
|---------------|-----|---------|
| **Simple REST APIs** | Native Python, full control | Overkill — connector overhead |
| **Custom extraction logic** | Easy to customise | Requires forking a connector |
| **Database extraction** | `sql_database` source works well | Debezium-based, more complex |
| **Cost** | Free (open source) | Per-record/credit pricing |
| **Debugging** | Standard Python debugging | Container logs, UI inspection |

### Why not dlt for everything?

dlt works well for SaaS sources when verified sources exist (like HubSpot). But Airbyte has advantages for complex SaaS:

1. **OAuth flows**: Salesforce, Google Ads, and similar require complex OAuth — Airbyte handles this via its UI
2. **Schema management**: Airbyte auto-detects and tracks upstream schema changes in SaaS tools
3. **Connector breadth**: 600+ connectors means coverage for almost any tool
4. **Reverse ETL**: Airbyte supports syncing data from Snowflake back to SaaS tools
5. **Non-engineer access**: The Airbyte UI lets non-engineers add and configure connections

## What You Will Build

By the end of this section:

```
Snowflake
├── AIRBYTE database
│   ├── HUBSPOT schema
│   │   └── CONTACTS table (synced from HubSpot CRM)
│   └── (additional schemas per SaaS source)
│
├── SVC_AIRBYTE service account
│   └── SVC_AIRBYTE dedicated role (created by user module)
│       └── Granted AIRBYTE_DB_WRITER
│
└── Role hierarchy
    └── AIRBYTE_DB_READER → ANALYTICS_SOURCES_READER
        └── Analysts can read all Airbyte-loaded data
```

Prefect orchestration:

```
Prefect
├── hubspot-airbyte-daily flow
│   └── Triggers Airbyte sync via API
└── Automations
    └── Alerts on sync failure
```

## Infrastructure Decisions

### Airbyte Cloud vs Self-Hosted

| Factor | Airbyte Cloud | Self-Hosted (ECS) |
|--------|--------------|-------------------|
| **Setup** | Sign up, configure via UI | Deploy infrastructure with Terraform |
| **Cost** | $99+/month (Starter) | ~$81/month (ECS + RDS) |
| **Maintenance** | Managed by Airbyte | You manage upgrades, scaling |
| **Connectors** | All available | All available |
| **Best for** | Getting started, small teams | Cost optimisation at scale |

This section documents both approaches.

### Database Naming

Following the same pattern as the batch ingestion section (databases named after the loader tool):

| Loader | Database | Example Schema |
|--------|----------|---------------|
| dlt | `DLT` | `DLT.OPEN_EXCHANGE_RATES` |
| Snowpipe | `SNOWPIPE` | `SNOWPIPE.OPEN_EXCHANGE_RATES` |
| **Airbyte** | **`AIRBYTE`** | **`AIRBYTE.HUBSPOT`** |

### Role Pattern

The `SVC_AIRBYTE` service account uses the `user_create_dedicated_role = true` pattern from the user Terraform module. This automatically creates a `SVC_AIRBYTE` role that is granted to the user. The database module then grants `AIRBYTE_DB_WRITER` to this role.

For read access, `AIRBYTE_DB_READER` is granted to `ANALYTICS_SOURCES_READER`, so all analyst roles can query Airbyte-loaded data through the existing role hierarchy.

## Section Contents

| Page | What You Will Do |
|------|-----------------|
| [Airbyte Concepts](1-airbyte-concepts.md) | Learn sources, destinations, connections, and sync modes |
| [Deployment Options](2-deployment-options.md) | Compare Cloud vs self-hosted, choose your approach |
| [Airbyte Cloud Setup](3-airbyte-cloud-setup.md) | Create workspace, generate API credentials |
| [Self-Hosted Setup](4-self-hosted-setup.md) | Deploy Airbyte on ECS with Terraform (optional) |
| [Snowflake Infrastructure](5-snowflake-infrastructure.md) | Create AIRBYTE database, SVC_AIRBYTE user, role grants |
| [HubSpot Connection](6-hubspot-connection.md) | Configure HubSpot source, Snowflake destination, first sync |
| [Reverse ETL](7-reverse-etl.md) | Sync enriched data from Snowflake back to SaaS tools |
| [Prefect Orchestration](8-prefect-orchestration.md) | Trigger syncs from Prefect, scheduling, monitoring |
| [Finishing Up](9-finishing-up.md) | Verification, cost summary, next steps |

## Prerequisites

Before starting this section, ensure you have completed:

- [x] [Snowflake Terraform Setup](../../getting-started/terraform-setup/snowflake/index.md) - Snowflake managed by Terraform
- [x] [Data Warehouse](../data-warehouse/index.md) - Databases, roles, and users module
- [x] [Orchestration](../orchestration/index.md) - Prefect Cloud or self-hosted
- [x] [Batch Data Ingestion](../batch-data-ingestion/index.md) - Understanding of dlt patterns (recommended)

## Cost Summary

| Approach | Monthly Cost | Notes |
|----------|-------------|-------|
| **dlt only** (Option A from batch section) | $0 | Free, uses existing infrastructure |
| **Airbyte Cloud** | $99+ | Starter tier, per-record overage |
| **Airbyte Self-Hosted** | ~$81 | ECS + RDS infrastructure costs |

Detailed cost breakdowns are covered in [Deployment Options](2-deployment-options.md) and the [Costs](../../getting-started/account-setup/costs.md) page.

## Get Started

Continue to [Airbyte Concepts](1-airbyte-concepts.md) →
