# Finishing Up

On this page, you will:

- [x] Verify the complete SaaS ingestion setup
- [x] Review monitoring and operational tasks
- [x] Understand costs and optimisation options
- [x] Plan next steps

## Verification Checklist

### Snowflake Infrastructure

```sql
-- Database exists
SHOW DATABASES LIKE 'AIRBYTE';

-- Schemas exist
SHOW SCHEMAS IN DATABASE AIRBYTE;
-- Should show: HUBSPOT

-- Database roles exist
SHOW DATABASE ROLES IN DATABASE AIRBYTE;
-- Should show: AIRBYTE_DB_READER, AIRBYTE_DB_WRITER

-- Service account
SHOW USERS LIKE 'SVC_AIRBYTE';
DESCRIBE USER SVC_AIRBYTE;
-- default_role should be SVC_AIRBYTE
-- default_warehouse should be LOADING

-- Role grants
SHOW GRANTS TO ROLE SVC_AIRBYTE;
-- Should include: AIRBYTE_DB_WRITER, LOADING warehouse usage

-- Reader access chain
SHOW GRANTS OF DATABASE ROLE AIRBYTE.AIRBYTE_DB_READER;
-- Should show: ANALYTICS_SOURCES_READER
```

### Data in Snowflake

```sql
-- HubSpot contacts loaded
SELECT COUNT(*), MIN(lastmodifieddate), MAX(lastmodifieddate)
FROM AIRBYTE.HUBSPOT.CONTACTS;

-- Analyst access works
USE ROLE ANALYTICS_DEVELOPER;
SELECT * FROM AIRBYTE.HUBSPOT.CONTACTS LIMIT 5;
```

### Airbyte Connections

Verify in the Airbyte UI or API:

```sh
# List connections
curl -s "https://api.airbyte.com/v1/connections?workspaceId=YOUR_WORKSPACE_ID" \
    -H "Authorization: Bearer YOUR_API_KEY" | jq '.data[].name'
```

Should show:

- `hubspot-to-snowflake` — Forward ETL (contacts)
- `retl-contacts-to-hubspot` — Reverse ETL (enriched contacts, if configured)

### Prefect Deployments

```sh
prefect deployment ls
```

Should include:

- `hubspot-airbyte-daily` — Scheduled daily at 07:00 UTC
- `retl-contacts-daily` — Scheduled daily at 08:00 UTC (if configured)

### Recent Flow Runs

```sh
prefect flow-run ls --limit 10
```

Check that states are `Completed`.

## Architecture Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      SAAS DATA INGESTION                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Forward ETL                                                                │
│  ───────────                                                                │
│  ┌─────────────┐     ┌─────────────────────┐     ┌─────────────────────┐    │
│  │ HubSpot CRM │     │  Airbyte            │     │ AIRBYTE.HUBSPOT     │    │
│  │ API         │────▶│  hubspot-to-        │────▶│ .CONTACTS           │    │
│  │ /contacts   │     │  snowflake          │     │                     │    │
│  └─────────────┘     │  (Daily 07:00 UTC)  │     └─────────────────────┘    │
│                      └─────────────────────┘                                │
│                                                                             │
│  Reverse ETL (Optional)                                                     │
│  ──────────────────────                                                     │
│  ┌─────────────────┐  ┌─────────────────────┐     ┌─────────────────────┐   │
│  │ Snowflake       │  │  Airbyte            │     │ HubSpot CRM         │   │
│  │ ANALYTICS.CRM.  │─▶│  retl-contacts-    │────▶│ Custom properties    │   │
│  │ CONTACTS_       │  │  to-hubspot         │     │ (enriched data)     │   │
│  │ ENRICHED        │  │  (Daily 08:00 UTC)  │     └─────────────────────┘   │
│  └─────────────────┘  └─────────────────────┘                               │
│                                                                             │
│  Orchestration                                                              │
│  ─────────────                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  Prefect Cloud                                                      │    │
│  │  • Triggers syncs via Airbyte API                                   │    │
│  │  • Monitors completion / failure                                    │    │
│  │  • Alerts via Slack automations                                     │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Monitoring

### Prefect Cloud

- **Flow run history**: Success/failure trends for Airbyte flows
- **Logs**: Sync trigger and completion logs
- **Alerts**: Slack notifications on failure

### Airbyte Dashboard

- **Sync history**: Duration, records synced, bytes transferred per connection
- **Schema changes**: Notifications when HubSpot schema changes
- **Error details**: Connector-level error messages for debugging

### Snowflake Query History

```sql
-- Recent queries by SVC_AIRBYTE
SELECT
    query_id,
    query_text,
    start_time,
    total_elapsed_time / 1000 as seconds,
    rows_inserted,
    bytes_scanned
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE user_name = 'SVC_AIRBYTE'
    AND start_time > DATEADD(day, -7, CURRENT_TIMESTAMP())
ORDER BY start_time DESC
LIMIT 20;
```

## Common Operations

### Trigger Manual Sync

```sh
# Via Prefect
prefect deployment run hubspot-airbyte-daily/production

# Via Airbyte API
curl -X POST "https://api.airbyte.com/v1/jobs" \
    -H "Authorization: Bearer YOUR_API_KEY" \
    -H "Content-Type: application/json" \
    -d '{"connectionId": "YOUR_CONNECTION_ID", "jobType": "sync"}'
```

### Reset a Connection

If you need to re-sync all data from scratch:

1. In Airbyte UI: Navigate to the connection > **Settings** > **Reset data**
2. This clears the sync state and re-extracts all records on the next sync

### Add a New SaaS Source

To add another SaaS source (e.g., Salesforce):

1. Create a new schema in the AIRBYTE database via Terraform
2. Configure the source in Airbyte UI
3. Create a connection to the existing Snowflake destination
4. Add a Prefect flow to trigger the sync
5. Update `prefect.yaml` with the new deployment

### Pause/Resume Syncs

In Prefect:

```sh
# Pause
prefect deployment pause hubspot-airbyte-daily/production

# Resume
prefect deployment resume hubspot-airbyte-daily/production
```

## Cost Summary

### Airbyte Cloud

| Component | Monthly Cost |
|-----------|-------------|
| Airbyte Cloud (Starter) | $99 |
| Additional records (if over 4M) | $15 per 1M |
| **Total** | **$99+** |

### Self-Hosted

| Component | Monthly Cost |
|-----------|-------------|
| ECS Fargate (server) | ~$30 |
| ECS Fargate (workers) | ~$20 |
| RDS PostgreSQL | ~$15 |
| ALB | ~$16 |
| **Total** | **~$81** |

### Snowflake (Additional)

| Component | Monthly Cost |
|-----------|-------------|
| LOADING warehouse (shared with dlt) | Already provisioned |
| AIRBYTE database storage | < $1 (small datasets) |
| **Total additional** | **< $1** |

### Comparison with dlt Option

| Approach | Monthly Cost | Best For |
|----------|-------------|----------|
| dlt (Option A) | $0 | 1-2 SaaS sources, code-first teams |
| Airbyte Cloud (Option B) | $99+ | Multiple SaaS sources, reverse ETL, non-engineers |
| Airbyte Self-Hosted | ~$81 | Cost-sensitive, data residency requirements |

## Troubleshooting

### Sync Timeout in Prefect

**Symptoms**: Prefect flow shows `TimeoutError`

**Solution**: The default timeout is 30 minutes. If syncs take longer:

1. Check Airbyte UI for the actual sync duration
2. Increase the `max_wait_seconds` in the flow code
3. Consider switching to incremental sync if using full refresh

### Schema Drift

**Symptoms**: Airbyte detects schema changes and pauses the connection

**Solution**:

1. Review the schema change in the Airbyte UI
2. Accept the change (usually safe for additive changes like new columns)
3. For breaking changes, update your dbt models before accepting

### Connection Quota Exceeded

**Symptoms**: Cannot create new connections (Airbyte Cloud)

**Solution**: Upgrade your Airbyte Cloud tier or remove unused connections.

## What You've Built

You've set up a complete SaaS data ingestion system:

- [x] **AIRBYTE database** in Snowflake with proper role hierarchy
- [x] **SVC_AIRBYTE** service account with dedicated role
- [x] **HubSpot → Snowflake** forward ETL (contacts)
- [x] **Snowflake → HubSpot** reverse ETL (enriched contacts)
- [x] **Prefect orchestration** with schedules and alerting
- [x] **Analyst access** via AIRBYTE_DB_READER → ANALYTICS_SOURCES_READER

## What's Next

With SaaS data landing in Snowflake alongside your batch data, you can:

1. **Transform with dbt**: Build staging and mart models for CRM data
   - Create `stg_hubspot__contacts` from `AIRBYTE.HUBSPOT.CONTACTS`
   - Join with other data sources for enrichment
   - Power the reverse ETL views

2. **Add more SaaS sources**: Follow the same pattern for:
   - Salesforce (accounts, opportunities)
   - Stripe (charges, customers, subscriptions)
   - Google Ads (campaigns, ad performance)
   - Zendesk (tickets, users)

3. **Build dashboards**: Connect BI tools to the transformed data

### Future Documentation

- `build/data-transformation/` — dbt modelling layer
- `build/observability/` — Data quality and monitoring
- `build/streaming-data-ingestion/` — Real-time data ingestion

## Feedback

Found an issue or have suggestions? Open an issue in the repository.
