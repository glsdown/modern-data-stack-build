# Airbyte Concepts

On this page, you will:

- [x] Understand Airbyte's architecture and components
- [x] Learn about sources, destinations, and connections
- [x] Understand sync modes and normalization

## Overview

Airbyte is an open-source data integration platform. It provides pre-built connectors for extracting data from SaaS tools and loading it into data warehouses. Think of it as a managed ELT pipeline specifically designed for SaaS-to-warehouse syncs.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AIRBYTE ARCHITECTURE                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐     ┌──────────────────────────────┐     ┌─────────────┐  │
│  │              │     │         Airbyte              │     │             │  │
│  │    Source    │────▶│  ┌────────────────────────┐  │────▶│ Destination │  │
│  │  (HubSpot)   │     │  │     Connection         │  │     │ (Snowflake) │  │
│  │              │     │  │  • Sync mode           │  │     │             │  │
│  └──────────────┘     │  │  • Schedule            │  │     └─────────────┘  │
│                       │  │  • Selected streams    │  │                      │
│                       │  └────────────────────────┘  │                      │
│                       │                              │                      │
│                       │  ┌────────────────────────┐  │                      │
│                       │  │     Orchestrator       │  │                      │
│                       │  │  • Job scheduling      │  │                      │
│                       │  │  • State management    │  │                      │
│                       │  │  • Error handling      │  │                      │
│                       │  └────────────────────────┘  │                      │
│                       └──────────────────────────────┘                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Core Concepts

### Sources

A **source** is a connector that extracts data from an external system. Each source:

- Connects to a specific SaaS tool or database
- Requires authentication (API key, OAuth, etc.)
- Exposes one or more **streams** (tables/endpoints)

Examples:

| Source | Streams | Authentication |
|--------|---------|---------------|
| HubSpot | contacts, companies, deals, tickets | Private app token |
| Salesforce | accounts, contacts, opportunities, leads | OAuth 2.0 |
| Stripe | charges, customers, invoices, subscriptions | API key |
| Google Ads | campaigns, ad_groups, ads, keywords | OAuth 2.0 |

### Destinations

A **destination** is where Airbyte loads data. For this project, the destination is Snowflake. Airbyte's Snowflake destination:

- Creates tables automatically
- Handles schema evolution
- Supports multiple loading modes (raw JSON, normalised tables)

### Connections

A **connection** links a source to a destination. It defines:

- **Which streams** to sync (e.g., only contacts from HubSpot)
- **Sync mode** per stream (full refresh or incremental)
- **Schedule** (how often to sync)
- **Namespace** (which Snowflake schema to write to)

```
┌──────────────┐         Connection          ┌──────────────┐
│              │  ┌──────────────────────┐   │              │
│   HubSpot    │──│ Streams: contacts    │──▶│  Snowflake   │
│   Source     │  │ Mode: incremental    │   │  Destination │
│              │  │ Schedule: daily      │   │              │
└──────────────┘  │ Namespace: HUBSPOT   │   └──────────────┘
                  └──────────────────────┘
```

### Streams

A **stream** is a single data entity from a source — roughly equivalent to a table. The HubSpot source exposes these streams:

| Stream | Description | Typical Sync Mode |
|--------|-------------|------------------|
| `contacts` | CRM contact records | Incremental |
| `companies` | Organisation records | Incremental |
| `deals` | Deal/opportunity records | Incremental |
| `tickets` | Support tickets | Incremental |
| `products` | Product catalogue | Full refresh |
| `quotes` | Price proposals | Incremental |

You select which streams to sync when creating a connection.

## Sync Modes

Airbyte supports several sync modes that control how data is extracted and loaded.

### Full Refresh | Overwrite

The source extracts **all** records every sync. The destination **replaces** the entire table.

```
Sync 1: Source has [A, B, C]     → Destination: [A, B, C]
Sync 2: Source has [A, B, C, D]  → Destination: [A, B, C, D]
Sync 3: Source has [A, B, D]     → Destination: [A, B, D]  (C removed)
```

**Use when**: Reference data that changes infrequently (e.g., product catalogue, currency codes).

### Full Refresh | Append

The source extracts **all** records every sync. The destination **appends** to the existing table.

```
Sync 1: Source has [A, B, C]     → Destination: [A, B, C]
Sync 2: Source has [A, B, C, D]  → Destination: [A, B, C, A, B, C, D]
```

**Use when**: You want to track snapshots over time. Requires deduplication in dbt.

### Incremental | Append

The source extracts only **new or modified** records since the last sync. The destination **appends** them.

```
Sync 1: Source has [A, B, C]     → Destination: [A, B, C]
Sync 2: B modified, D added      → Destination: [A, B, C, B', D]
```

**Use when**: High-volume tables where full refresh is too slow. Handle deduplication in dbt.

### Incremental | Append + Dedup

The source extracts only **new or modified** records. The destination **upserts** — updates existing rows and appends new ones.

```
Sync 1: Source has [A, B, C]     → Destination: [A, B, C]
Sync 2: B modified, D added      → Destination: [A, B', C, D]
```

**Use when**: You want the destination to always reflect the current state. This is the most common mode for CRM data.

### Recommended Modes for HubSpot

| Stream | Recommended Mode | Reason |
|--------|-----------------|--------|
| `contacts` | Incremental \| Append + Dedup | Contacts are updated frequently, want current state |
| `companies` | Incremental \| Append + Dedup | Same as contacts |
| `deals` | Incremental \| Append + Dedup | Deals progress through stages |
| `products` | Full Refresh \| Overwrite | Small, infrequently changing |

## Normalization

When Airbyte loads data into Snowflake, it can apply different levels of normalization.

### Raw JSON (No Normalization)

Airbyte writes a single `_airbyte_raw_*` table per stream with columns:

| Column | Description |
|--------|-------------|
| `_airbyte_raw_id` | Unique row identifier |
| `_airbyte_data` | Full record as JSON VARIANT |
| `_airbyte_loaded_at` | When the record was loaded |
| `_airbyte_extracted_at` | When the record was extracted from source |

This is fast to load but requires parsing JSON in dbt.

### Typing and Deduplication (Recommended)

Airbyte creates typed tables with proper columns alongside the raw tables:

```sql
-- Typed table (created by Airbyte)
SELECT
    id,
    firstname,      -- VARCHAR
    lastname,       -- VARCHAR
    email,          -- VARCHAR
    createdate,     -- TIMESTAMP
    _airbyte_raw_id,
    _airbyte_extracted_at
FROM AIRBYTE.HUBSPOT.CONTACTS;
```

This is the default in Airbyte Cloud and is recommended for most use cases.

## State Management

Airbyte tracks sync state to support incremental syncs:

1. **Cursor values**: For incremental syncs, Airbyte stores the last cursor value (e.g., `lastmodifieddate = 2026-02-15T10:30:00Z`)
2. **Sync history**: Each sync's metadata (start time, records synced, bytes transferred)
3. **Schema catalog**: The discovered schema from the source

State is stored in Airbyte's internal database (managed for Cloud, PostgreSQL for self-hosted).

## Airbyte vs dlt Comparison

| Feature | Airbyte | dlt |
|---------|---------|-----|
| **Connectors** | 600+ pre-built | 30+ verified sources |
| **Configuration** | UI or API | Python code |
| **Sync modes** | Full refresh, incremental, append+dedup | Configurable via code |
| **Reverse ETL** | Built-in | Not supported |
| **Schema evolution** | Automatic detection | Automatic handling |
| **State management** | Managed internally | Pipeline state files |
| **Debugging** | UI logs, container logs | Standard Python debugging |
| **Cost** | $99+/month or self-hosted | Free |
| **Custom logic** | Fork connector | Native Python |

Both tools are valid for data ingestion. The decision comes down to your team's needs, technical ability, and number of SaaS sources.

## Key Terminology

| Term | Definition |
|------|-----------|
| **Source** | A connector that reads data from an external system |
| **Destination** | A connector that writes data to a target system |
| **Connection** | A configured link between a source and destination |
| **Stream** | A single data entity (table) from a source |
| **Sync** | A single execution of a connection |
| **Sync mode** | How data is extracted (full/incremental) and loaded (append/overwrite/dedup) |
| **Catalog** | The schema describing available streams and their fields |
| **Cursor field** | The field used for incremental sync tracking (e.g., `updatedAt`) |
| **Primary key** | The field(s) used for deduplication |
| **Namespace** | The target schema in the destination (e.g., `HUBSPOT`) |

## Summary

You now understand Airbyte's core concepts:

- [x] Sources extract data from SaaS tools via pre-built connectors
- [x] Destinations load data into warehouses like Snowflake
- [x] Connections define which streams to sync and how
- [x] Sync modes control full refresh vs incremental and append vs dedup
- [x] Normalization creates typed tables from raw JSON

## What's Next

With concepts understood, you need to decide how to deploy Airbyte.

Continue to [Deployment Options](2-deployment-options.md) →
