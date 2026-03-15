# Reverse ETL

On this page, you will:

- [x] Understand what reverse ETL is and when to use it
- [x] Configure Snowflake as an Airbyte source
- [x] Sync enriched data from Snowflake back to HubSpot
- [x] Consider operational patterns and best practices

## Overview

Reverse ETL is the process of syncing data **from** your data warehouse **back to** SaaS tools. This closes the loop: raw data flows into Snowflake, gets transformed by dbt, and the enriched results are pushed back to operational tools like HubSpot.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        REVERSE ETL FLOW                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Forward ETL (data in)                    Reverse ETL (data out)            │
│  ─────────────────────                    ──────────────────────            │
│                                                                             │
│  ┌─────────────┐       ┌───────────┐       ┌─────────────┐                  │
│  │  HubSpot    │──────▶│ Snowflake │──────▶│  HubSpot    │                  │
│  │  (raw CRM)  │       │           │       │  (enriched) │                  │
│  └─────────────┘       │ dbt model │       └─────────────┘                  │
│                        │ transforms│                                        │
│  ┌─────────────┐       │ and       │       ┌─────────────┐                  │
│  │  Stripe     │──────▶│ enriches  │──────▶│  Slack      │                  │
│  │  (payments) │       │ the data  │       │  (alerts)   │                  │
│  └─────────────┘       └───────────┘       └─────────────┘                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## When to Use Reverse ETL

| Use Case | Example |
|----------|---------|
| **Lead scoring** | Calculate a lead score in dbt, push it to HubSpot as a contact property |
| **Customer segmentation** | Segment customers by behaviour, update HubSpot lists |
| **Lifetime value** | Calculate LTV in Snowflake, sync to CRM for sales prioritisation |
| **Product usage** | Aggregate product metrics, push to CRM for customer success |
| **Alerting** | Identify churn risk, notify via Slack or CRM task |

### When NOT to Use Reverse ETL

- **Real-time requirements**: Reverse ETL is batch-oriented (hourly or daily). For real-time, use event streams or webhooks.
- **Simple field mapping**: If you just need to copy a field from one SaaS tool to another, a direct Zapier/Make integration may be simpler.
- **No transformation needed**: If the data doesn't need enrichment, reverse ETL adds unnecessary complexity.

## Architecture

Airbyte supports Snowflake as a **source**, enabling reverse ETL through the same platform used for forward ETL.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        AIRBYTE REVERSE ETL                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Source                   Connection                  Destination           │
│  ──────                   ──────────                  ───────────           │
│                                                                             │
│  ┌────────────────-─┐     ┌───────────────────┐     ┌─────────────────┐     │
│  │ Snowflake        │     │ retl-contacts-    │     │ HubSpot         │     │
│  │ Source           │────▶│ to-hubspot        │────▶│ Destination     │     │
│  │                  │     │                   │     │                 │     │
│  │ View/Table:      │     │ Incremental       │     │ Update contacts │     │
│  │ MARTS.CRM.       │     │ Append + Dedup    │     │ via API         │     │
│  │ CONTACTS_ENRICHED│     └───────────────────┘     └─────────────────┘     │
│  └─────────────────-┘                                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Prepare the Source View

Create a view in Snowflake that contains some example enriched data you want to sync back to HubSpot.

### Create the CRM Schema via Terraform

The `ANALYTICS.CRM` schema should be managed as infrastructure. Add it to your Snowflake Terraform configuration:

```hcl
# terraform/snowflake/schemas.tf

module "schema_analytics_crm" {
  source = "./modules/snowflake_schema"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  schema_name          = "CRM"
  schema_database_name = "ANALYTICS"
  schema_comment       = "CRM-domain views and models for analytics and reverse ETL."
}
```

Deploy via CI/CD, then create the view.

!!! note "dbt Models"
    In practice, this view would be a dbt model in your transformation layer. The data transformation section (future) covers building these models. For reverse ETL testing, a manual view works.

### Example: Enriched Contacts View

```sql
CREATE OR REPLACE VIEW ANALYTICS.CRM.CONTACTS_ENRICHED AS
SELECT
    id AS hubspot_contact_id
    , email
    , firstname
    , lastname
    -- Example enriched fields to push back to HubSpot
    , UNIFORM(1, 10, RANDOM()) AS total_orders
    , UNIFORM(1, 10000::number(10, 2), RANDOM()) AS lifetime_value_usd
    , ARRAY_CONSTRUCT('low','medium','high')[UNIFORM(0, 3, RANDOM())] AS customer_tier
    CURRENT_TIMESTAMP() AS _synced_at
FROM AIRBYTE.HUBSPOT.CONTACTS
```

### Grant Read Access

The reverse ETL source needs read access to the view. If using the same SVC_AIRBYTE account:

```sql
-- Grant usage on the schema containing the view
GRANT USAGE ON DATABASE ANALYTICS TO ROLE SVC_AIRBYTE;
GRANT USAGE ON SCHEMA ANALYTICS.CRM TO ROLE SVC_AIRBYTE;
GRANT SELECT ON VIEW ANALYTICS.CRM.CONTACTS_ENRICHED TO ROLE SVC_AIRBYTE;
```

!!! warning "Least Privilege"
    Only grant access to the specific views needed for reverse ETL. Do not grant broad read access to the analytics database.

## Configure Snowflake as a Source

### Via Airbyte UI

1. Navigate to **Sources** > **New Source**
2. Search for **Snowflake** and select it
3. Configure:

| Field | Value |
|-------|-------|
| **Source name** | `snowflake-reverse-etl` |
| **Host** | `xxx.eu-west-2.snowflakecomputing.com` |
| **Role** | `SVC_AIRBYTE` |
| **Warehouse** | `LOADING` |
| **Database** | `ANALYTICS` |
| **Schema** | `CRM` |
| **Authentication** | Key Pair |
| **Username** | `SVC_AIRBYTE` |
| **Private Key** | Same key as the destination |

4. Click **Set up source**

## Configure HubSpot as a Destination

### Via Airbyte UI

1. Navigate to **Destinations** > **New Destination**
2. Search for **HubSpot** and select it
3. Configure:

| Field | Value |
|-------|-------|
| **Destination name** | `hubspot-reverse-etl` |
| **Authentication** | Private App |
| **Access Token** | Your private app token (needs write scope) |

!!! warning "Write Scope Required"
    For reverse ETL, the HubSpot private app needs write scopes:

    - `crm.objects.contacts.write`

    This is a separate scope from the read scope used for forward ETL. Consider using a separate private app for reverse ETL to maintain least-privilege access.

4. Click **Set up destination**

## Create the Reverse ETL Connection

### Via Airbyte UI

1. Navigate to **Connections** > **New Connection**
2. Select `snowflake-reverse-etl` as the source
3. Select `hubspot-reverse-etl` as the destination
4. Configure:

| Setting | Value |
|---------|-------|
| **Connection name** | `retl-contacts-to-hubspot` |
| **Schedule** | Manual (triggered by Prefect) |

5. **Select streams**: Enable `CONTACTS_ENRICHED`

| Stream | Sync Mode | Primary Key |
|--------|-----------|------------|
| `CONTACTS_ENRICHED` | Incremental \| Append + Dedup | `hubspot_contact_id` |

6. **Map fields**: Configure how Snowflake columns map to HubSpot properties

| Snowflake Column | HubSpot Property |
|-----------------|-----------------|
| `hubspot_contact_id` | Contact ID (lookup key) |
| `total_orders` | `total_orders` (custom property) |
| `lifetime_value_usd` | `lifetime_value` (custom property) |
| `customer_tier` | `customer_tier` (custom property) |

7. Click **Set up connection**

### Create Custom Properties in HubSpot

Before the first sync, create the custom contact properties in HubSpot:

1. Go to **Settings** > **Properties** > **Contact Properties**
2. Click **Create property** for each:

| Property | Type | Group |
|----------|------|-------|
| `total_orders` | Number | Contact information |
| `lifetime_value` | Number (currency) | Contact information |
| `customer_tier` | Single-line text | Contact information |

## Run and Verify

### Trigger the Sync

```sh
# Via Airbyte API
curl -X POST "https://api.airbyte.com/v1/jobs" \
    -H "Authorization: Bearer YOUR_API_KEY" \
    -H "Content-Type: application/json" \
    -d '{
        "connectionId": "YOUR_RETL_CONNECTION_ID",
        "jobType": "sync"
    }'
```

### Verify in HubSpot

1. Open a contact in HubSpot
2. Check the custom properties section
3. Verify `total_orders`, `lifetime_value`, and `customer_tier` are populated

## Best Practices

### 1. Use Views, Not Tables

Source reverse ETL from views or dbt models, not raw tables. Views ensure:

- Data is always current (computed at query time)
- Transformations are applied consistently
- You can change the logic without reconfiguring Airbyte

### 2. Include a Sync Timestamp

Add a `_synced_at` or `_updated_at` column to your source view. This helps with debugging and incremental sync tracking.

### 3. Limit Record Volume

Only sync records that have changed. Use incremental sync mode and include a cursor field (e.g., `_updated_at`) to avoid re-syncing unchanged records.

### 4. Separate Read and Write Scopes

Use separate HubSpot private apps for forward ETL (read-only) and reverse ETL (write). This follows the principle of least privilege.

### 5. Test in a Sandbox

HubSpot offers sandbox accounts for testing. Configure a separate Airbyte connection pointing to the sandbox before running reverse ETL in production.

## Sync Frequency Considerations

| Frequency | Use Case | HubSpot API Impact |
|-----------|----------|-------------------|
| Daily | Lead scoring, customer segmentation | Low |
| Hourly | Time-sensitive alerts | Moderate |
| Real-time | Not supported by Airbyte | Use webhooks instead |

For most use cases, a daily sync is sufficient and stays well within HubSpot's API rate limits.

## Summary

You've set up reverse ETL:

- [x] Understood when reverse ETL is valuable
- [x] Configured Snowflake as an Airbyte source
- [x] Set up HubSpot as a destination with write access
- [x] Created a connection to sync enriched contact data
- [x] Learned best practices for reverse ETL

## What's Next

With forward and reverse ETL configured, orchestrate everything with Prefect.

Continue to [Prefect Orchestration](8-prefect-orchestration.md) →
