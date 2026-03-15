# HubSpot Connection

On this page, you will:

- [x] Configure the HubSpot source in Airbyte
- [x] Configure the Snowflake destination in Airbyte
- [x] Create a connection syncing contacts to Snowflake
- [x] Run the first sync and verify data

## Overview

With Airbyte deployed and Snowflake infrastructure in place, you can now create the HubSpot-to-Snowflake connection.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      HUBSPOT CONNECTION                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────┐       ┌─────────────────┐       ┌─────────────────┐        │
│  │   HubSpot   │       │   Connection    │       │   Snowflake     │        │
│  │   Source    │──────▶│   (contacts)    │──────▶│   Destination   │        │
│  │             │       │                 │       │                 │        │
│  │ API Token   │       │ Incremental     │       │ AIRBYTE.HUBSPOT │        │
│  │ Contacts    │       │ Append + Dedup  │       │ .CONTACTS       │        │
│  └─────────────┘       └─────────────────┘       └─────────────────┘        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Prerequisites

- [x] [Airbyte Cloud](3-airbyte-cloud-setup.md) or [Self-Hosted](4-self-hosted-setup.md) deployed
- [x] [Snowflake Infrastructure](5-snowflake-infrastructure.md) — AIRBYTE database and SVC_AIRBYTE user
- [x] HubSpot private app with `crm.objects.contacts.read` scope

## HubSpot Private App Setup

If you haven't already created a HubSpot private app:

1. Go to **Settings** > **Integrations** > **Private Apps** in HubSpot
2. Click **Create a private app**
3. Name it `airbyte-connector`
4. Under **Scopes**, enable:
   - `crm.objects.contacts.read`
5. Click **Create app** and copy the access token

!!! tip "Test with Sample Data"
    If your HubSpot account is empty, create a few test contacts to verify the sync works. The free CRM tier allows up to 1,000,000 contacts.

## Configure the HubSpot Source

### Via Airbyte UI

1. Navigate to **Sources** in the Airbyte UI
2. Click **New Source**
3. Search for **HubSpot** and select it
4. Configure the source:

| Field | Value |
|-------|-------|
| **Source name** | `hubspot-production` |
| **Authentication** | Private App |
| **Access Token** | Your private app access token |
| **Start date** | `2026-01-01T00:00:00Z` (or your preferred start) |

5. Click **Set up source**
6. Airbyte will test the connection and discover available streams

### Via Airbyte API

For programmatic setup, use the Airbyte API:

```sh
curl -X POST "https://api.airbyte.com/v1/sources" \
    -H "Authorization: Bearer YOUR_API_KEY" \
    -H "Content-Type: application/json" \
    -d '{
        "name": "hubspot-production",
        "workspaceId": "YOUR_WORKSPACE_ID",
        "configuration": {
            "sourceType": "hubspot",
            "credentials": {
                "credentials_title": "Private App Credentials",
                "access_token": "YOUR_ACCESS_TOKEN"
            },
            "start_date": "2026-01-01T00:00:00Z"
        }
    }'
```

Save the returned `sourceId` — you will need it to create the connection.

## Configure the Snowflake Destination

### Via Airbyte UI

1. Navigate to **Destinations** in the Airbyte UI
2. Click **New Destination**
3. Search for **Snowflake** and select it
4. Configure the destination:

| Field | Value |
|-------|-------|
| **Destination name** | `snowflake-airbyte` |
| **Host** | `xxx.eu-west-2.snowflakecomputing.com` |
| **Role** | `SVC_AIRBYTE` |
| **Warehouse** | `LOADING` |
| **Database** | `AIRBYTE` |
| **Default Schema** | `HUBSPOT` |
| **Authentication** | Key Pair |
| **Username** | `SVC_AIRBYTE` |
| **Private Key** | Contents of `svc_airbyte_rsa_key.p8` (without headers) |

5. Click **Set up destination**
6. Airbyte will test the connection to Snowflake

!!! note "Private Key Format"
    When entering the private key in Airbyte, paste the key content without the `-----BEGIN PRIVATE KEY-----` and `-----END PRIVATE KEY-----` headers. Remove all newlines so it is a single line.

### Via Airbyte API

```sh
curl -X POST "https://api.airbyte.com/v1/destinations" \
    -H "Authorization: Bearer YOUR_API_KEY" \
    -H "Content-Type: application/json" \
    -d '{
        "name": "snowflake-airbyte",
        "workspaceId": "YOUR_WORKSPACE_ID",
        "configuration": {
            "destinationType": "snowflake",
            "host": "xxx.eu-west-2.snowflakecomputing.com",
            "role": "SVC_AIRBYTE",
            "warehouse": "LOADING",
            "database": "AIRBYTE",
            "schema": "HUBSPOT",
            "credentials": {
                "auth_type": "Key Pair Authentication",
                "private_key": "YOUR_BASE64_PRIVATE_KEY"
            },
            "username": "SVC_AIRBYTE"
        }
    }'
```

## Create the Connection

### Via Airbyte UI

1. Navigate to **Connections**
2. Click **New Connection**
3. Select `hubspot-production` as the source
4. Select `snowflake-airbyte` as the destination
5. Configure the connection:

| Setting | Value |
|---------|-------|
| **Connection name** | `hubspot-to-snowflake` |
| **Schedule type** | Manual (Prefect will trigger syncs) |
| **Namespace** | Destination default (`HUBSPOT`) |

6. **Select streams**: Enable only `contacts`

| Stream | Sync Mode | Primary Key | Cursor Field |
|--------|-----------|------------|-------------|
| `contacts` | Incremental \| Append + Dedup | `id` | `updatedAt` |

7. Click **Set up connection**

!!! warning "Manual Schedule"
    Set the schedule to **Manual** because Prefect will trigger syncs via the Airbyte API. This avoids double-scheduling and gives Prefect full control over orchestration.

### Via Airbyte API

```sh
curl -X POST "https://api.airbyte.com/v1/connections" \
    -H "Authorization: Bearer YOUR_API_KEY" \
    -H "Content-Type: application/json" \
    -d '{
        "name": "hubspot-to-snowflake",
        "sourceId": "YOUR_SOURCE_ID",
        "destinationId": "YOUR_DESTINATION_ID",
        "configurations": {
            "streams": [
                {
                    "name": "contacts",
                    "syncMode": "incremental_deduped_history",
                    "primaryKey": [["id"]],
                    "cursorField": ["updatedAt"]
                }
            ]
        },
        "schedule": {
            "scheduleType": "manual"
        },
        "namespaceDefinition": "destination",
        "namespaceFormat": "HUBSPOT"
    }'
```

Save the returned `connectionId` — Prefect needs this to trigger syncs.

### Store Connection ID

Store the connection ID in AWS Secrets Manager for Prefect:

```sh
aws secretsmanager update-secret \
    --secret-id "airbyte/api-credentials" \
    --secret-string '{
        "api_key": "YOUR_API_KEY",
        "workspace_id": "YOUR_WORKSPACE_ID",
        "api_url": "https://api.airbyte.com/v1",
        "hubspot_connection_id": "YOUR_CONNECTION_ID"
    }' \
    --region eu-west-2
```

## Run the First Sync

### Via Airbyte UI

1. Navigate to **Connections** > `hubspot-to-snowflake`
2. Click **Sync now**
3. Monitor the sync progress in the UI
4. Wait for the sync to complete

### Via Airbyte API

```sh
curl -X POST "https://api.airbyte.com/v1/jobs" \
    -H "Authorization: Bearer YOUR_API_KEY" \
    -H "Content-Type: application/json" \
    -d '{
        "connectionId": "YOUR_CONNECTION_ID",
        "jobType": "sync"
    }'
```

Check sync status:

```sh
curl -s "https://api.airbyte.com/v1/jobs?connectionId=YOUR_CONNECTION_ID&limit=1" \
    -H "Authorization: Bearer YOUR_API_KEY" | jq '.data[0].status'
```

## Verify in Snowflake

After the first sync completes, verify data in Snowflake:

```sql
-- Check tables were created
SHOW TABLES IN AIRBYTE.HUBSPOT;

-- View contacts
SELECT
    id,
    firstname,
    lastname,
    email,
    createdate,
    lastmodifieddate,
    _airbyte_raw_id,
    _airbyte_extracted_at
FROM AIRBYTE.HUBSPOT.CONTACTS
ORDER BY lastmodifieddate DESC
LIMIT 10;

-- Count contacts
SELECT COUNT(*)
FROM AIRBYTE.HUBSPOT.CONTACTS;

-- Check analyst access via role hierarchy
USE ROLE ANALYTICS_DEVELOPER;
SELECT * FROM AIRBYTE.HUBSPOT.CONTACTS LIMIT 5;
```

### Understanding Airbyte Tables

Airbyte creates two types of tables per stream:

| Table | Purpose |
|-------|---------|
| `CONTACTS` | Typed, deduplicated table (primary table to query) |
| `_AIRBYTE_RAW_CONTACTS` | Raw JSON records for debugging |

The typed table has proper column types and is deduplicated by the primary key (`id`). Use this table for dbt models.

### Airbyte Metadata Columns

Each table includes metadata columns:

| Column | Description |
|--------|-------------|
| `_airbyte_raw_id` | Unique identifier for the raw record |
| `_airbyte_extracted_at` | When the record was extracted from HubSpot |
| `_airbyte_loaded_at` | When the record was loaded to Snowflake |

## Incremental Sync Behaviour

After the initial full sync, subsequent syncs only extract contacts modified since the last sync:

1. **First sync**: Extracts all contacts with `updatedAt >= start_date`
2. **Second sync**: Extracts only contacts with `updatedAt > last_sync_cursor`
3. **Deduplication**: Airbyte updates existing rows and inserts new ones (Append + Dedup mode)

This keeps sync duration short and minimises API calls to HubSpot.

## Troubleshooting

### Source Connection Failed

```
Error: Unable to connect to HubSpot
```

**Solution**: Verify the private app token has `crm.objects.contacts.read` scope and has not been revoked.

### Destination Connection Failed

```
Error: Snowflake authentication failed
```

**Solution**: Check:

1. SVC_AIRBYTE user has the RSA public key set
2. Private key in Airbyte matches the public key in Snowflake
3. Account identifier format is correct (e.g., `xxx.eu-west-2`)

### Sync Failed with Permission Error

```
Error: Insufficient privileges to operate on schema 'HUBSPOT'
```

**Solution**: Verify that `AIRBYTE_DB_WRITER` is granted to the `SVC_AIRBYTE` role:

```sql
SHOW GRANTS TO ROLE SVC_AIRBYTE;
```

### No Data After Sync

**Solution**: Check:

1. Your HubSpot account has contacts
2. The `start_date` is before your contacts were created
3. The `contacts` stream is enabled in the connection

## Summary

You've configured the HubSpot-to-Snowflake connection:

- [x] Created the HubSpot source with private app authentication
- [x] Configured the Snowflake destination with SVC_AIRBYTE credentials
- [x] Created a connection syncing contacts with incremental + dedup
- [x] Ran the first sync and verified data in Snowflake

## What's Next

With data flowing from HubSpot to Snowflake, you can optionally set up reverse ETL to sync enriched data back to SaaS tools.

Continue to [Reverse ETL](7-reverse-etl.md) →
