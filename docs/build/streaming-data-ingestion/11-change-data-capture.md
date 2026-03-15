# Change Data Capture (Optional)

!!! info "Optional Advanced Topic"
    This page is **optional** and covers an **advanced pattern**. Most teams should use batch database extraction with dlt (covered in the Batch Data Ingestion section). Only implement CDC if you need real-time database replication, sub-minute data freshness, or comprehensive audit trails.

On this page, you will:

- [x] Understand Change Data Capture (CDC) vs batch extraction
- [x] Deploy Debezium for PostgreSQL CDC
- [x] Capture database changes (INSERT/UPDATE/DELETE) from transaction logs
- [x] Stream changes to Kafka topics with full event history
- [x] Handle schema evolution and DDL changes
- [x] Replicate data to Snowflake for real-time analytics
- [x] Compare CDC vs batch extraction trade-offs
- [x] Implement CDC for audit trails and event sourcing

## Overview

**Change Data Capture (CDC)** captures every database change (INSERT, UPDATE, DELETE) by reading the transaction log - without impacting the production database with queries.

**Trade-off:** Real-time replication but significantly more complexity than batch extraction.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CDC vs BATCH EXTRACTION                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Batch Extraction (dlt) - Simple, Good Enough for Most Use Cases       │
│  ─────────────────────────────────────────────────────────────────      │
│  ┌────────────┐      ┌────────────┐      ┌────────────┐               │
│  │ PostgreSQL │      │    dlt     │      │ Snowflake  │               │
│  │ (prod DB)  │─────▶│ Pipeline   │─────▶│ (ANALYTICS)│               │
│  │            │ SQL  │ (hourly)   │ bulk │            │               │
│  └────────────┘      └────────────┘      └────────────┘               │
│                                                                         │
│  • Latency: 1-60 minutes (acceptable for most analytics)               │
│  • Impact: Low (optimised queries with indexes)                        │
│  • Cost: $10-50/month (dlt + Snowflake compute)                        │
│  • Complexity: Low (Python pipeline, standard SQL)                     │
│  • Data: Current state only (snapshots)                                │
│                                                                         │
│  ─────────────────────────────────────────────────────────────────────  │
│                                                                         │
│  Change Data Capture (Debezium) - Real-Time, Full History              │
│  ──────────────────────────────────────────────────────────────         │
│  ┌────────────┐      ┌────────────┐      ┌────────────┐               │
│  │ PostgreSQL │      │ Debezium   │      │   Kafka    │               │
│  │ (prod DB)  │─────▶│ Connector  │─────▶│   Topic    │               │
│  │            │ WAL  │            │ CDC  │            │               │
│  └────────────┘      └────────────┘      └────────────┘               │
│                                               │                         │
│                                               ▼                         │
│                                          ┌────────────┐                 │
│                                          │ Snowflake  │                 │
│                                          │ (STREAMING)│                 │
│                                          └────────────┘                 │
│                                                                         │
│  • Latency: < 1 second (real-time replication)                         │
│  • Impact: Minimal (reads WAL, not table queries)                      │
│  • Cost: $300-500/month (Kafka + Debezium + Snowflake)                 │
│  • Complexity: High (Kafka, schema evolution, data modelling)          │
│  • Data: Complete history (before/after values, all operations)        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## When to Use CDC

### Decision Tree

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CDC vs BATCH DECISION TREE                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Do you need data within seconds (not minutes/hours)?                   │
│  ├─ NO  → Use batch extraction with dlt (simpler, cheaper)             │
│  └─ YES → Continue                                                      │
│                                                                         │
│  Do you need complete change history (INSERT/UPDATE/DELETE)?            │
│  ├─ NO  → Use batch extraction (current state sufficient)              │
│  └─ YES → Continue                                                      │
│                                                                         │
│  Is your database PostgreSQL, MySQL, MongoDB, SQL Server, or Oracle?    │
│  ├─ NO  → Use batch extraction (CDC not available for your database)   │
│  └─ YES → Continue                                                      │
│                                                                         │
│  Can you enable WAL/binlog on production database?                      │
│  ├─ NO  → Use batch extraction (CDC requires transaction log access)   │
│  └─ YES → Continue                                                      │
│                                                                         │
│  Do you already have Kafka infrastructure?                              │
│  ├─ NO  → Reconsider (adding Kafka for CDC alone is expensive)         │
│  └─ YES → Use CDC                                                       │
│                                                                         │
│  ✅ USE CDC: Real-time replication with full history                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Good Reasons for CDC

✅ **Real-time operational analytics** - Dashboard showing live database state
✅ **Audit trails** - Complete history of who changed what and when
✅ **Event sourcing** - Rebuild application state from event history
✅ **Data replication** - Keep read replicas in sync for reporting
✅ **Slowly changing dimensions** - Track full history for Type 2 SCDs
✅ **Zero-impact extraction** - Can't add indexes or slow down production database

### Bad Reasons

❌ **"We want real-time"** - without specific business value (batch is often sufficient)
❌ **Resume building** - Using CDC because it's interesting, not needed
❌ **Avoiding dlt** - Batch extraction is simpler for most analytics
❌ **Over-engineering** - Adding complexity before validating need

### Use Cases by Method

| Use Case | Batch (dlt) | CDC (Debezium) | Rationale |
|----------|------------|---------------|-----------|
| **Daily reports** | ✅ | ❌ | Hourly/daily latency sufficient |
| **Hourly dashboards** | ✅ | ⚠️ | dlt can run every 15 min |
| **Real-time dashboards** | ❌ | ✅ | Need < 1 min latency |
| **Audit trails** | ⚠️ | ✅ | Need DELETE events and before/after |
| **Event sourcing** | ❌ | ✅ | Need full change history |
| **Current state analytics** | ✅ | ❌ | Batch is simpler |
| **Historical trends** | ✅ | ✅ | Both work (batch is simpler) |

## Debezium Architecture

### What is Debezium?

**Debezium** is an open-source CDC platform that:

- Reads database transaction logs (PostgreSQL WAL, MySQL binlog, etc.)
- Captures every INSERT, UPDATE, DELETE operation
- Publishes changes to Kafka topics
- Handles schema evolution automatically
- Provides exactly-once delivery guarantees

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DEBEZIUM ARCHITECTURE                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Production Database (PostgreSQL)                                       │
│  ────────────────────────────────                                       │
│  ┌────────────────────────────────────────────────────┐                │
│  │ Tables: orders, customers, products                │                │
│  │                                                     │                │
│  │  Application writes:                               │                │
│  │  INSERT INTO orders VALUES (...)                   │                │
│  │  UPDATE orders SET status = 'SHIPPED' ...         │                │
│  │  DELETE FROM orders WHERE ...                      │                │
│  │                                                     │                │
│  │  Transaction Log (WAL):                            │                │
│  │  ┌──────────────────────────────────────────────┐  │                │
│  │  │ LSN: 0/12345678                              │  │                │
│  │  │ Op: INSERT, Table: orders, New: {...}       │  │                │
│  │  │ Op: UPDATE, Table: orders,                  │  │                │
│  │  │     Before: {...}, After: {...}             │  │                │
│  │  │ Op: DELETE, Table: orders, Old: {...}       │  │                │
│  │  └──────────────────────────────────────────────┘  │                │
│  └────────────────────────────────────────────────────┘                │
│                           │                                             │
│                           ▼ (reads WAL, no table queries)               │
│  Debezium Connector (Kafka Connect)                                     │
│  ───────────────────────────────────────                                │
│  ┌────────────────────────────────────────────────────┐                │
│  │ PostgreSQL Connector:                              │                │
│  │ • Establishes replication slot                     │                │
│  │ • Reads WAL continuously                           │                │
│  │ • Decodes transaction log entries                  │                │
│  │ • Converts to Kafka events with schema            │                │
│  │ • Publishes to Kafka topics                        │                │
│  └────────────────────────────────────────────────────┘                │
│                           │                                             │
│                           ▼                                             │
│  Kafka Topics (One per table)                                           │
│  ─────────────────────────────                                          │
│  ┌────────────────────────────────────────────────────┐                │
│  │ Topic: postgres.public.orders                      │                │
│  │                                                     │                │
│  │ Message 1 (INSERT):                                │                │
│  │ {                                                  │                │
│  │   "before": null,                                  │                │
│  │   "after": {                                       │                │
│  │     "order_id": 12345,                             │                │
│  │     "customer_id": "cust_123",                     │                │
│  │     "status": "CREATED",                           │                │
│  │     "total": 99.99                                 │                │
│  │   },                                               │                │
│  │   "op": "c" (create),                              │                │
│  │   "ts_ms": 1709643845123,                          │                │
│  │   "source": {                                      │                │
│  │     "db": "production",                            │                │
│  │     "table": "orders",                             │                │
│  │     "lsn": 12345678                                │                │
│  │   }                                                │                │
│  │ }                                                  │                │
│  │                                                     │                │
│  │ Message 2 (UPDATE):                                │                │
│  │ {                                                  │                │
│  │   "before": {"order_id": 12345, "status": "CREATED", ...},         │
│  │   "after": {"order_id": 12345, "status": "SHIPPED", ...},          │
│  │   "op": "u" (update),                              │                │
│  │   "ts_ms": 1709644000000                           │                │
│  │ }                                                  │                │
│  └────────────────────────────────────────────────────┘                │
│                           │                                             │
│                           ▼                                             │
│  Snowflake Sink Connector                                               │
│  ─────────────────────────                                              │
│  ┌────────────────────────────────────────────────────┐                │
│  │ Writes to STREAMING.PUBLIC.ORDERS                  │                │
│  │ Each row = one change event                        │                │
│  │ Can query: "Show me all changes today"             │                │
│  └────────────────────────────────────────────────────┘                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## PostgreSQL CDC Setup

### Prerequisites

1. **PostgreSQL 10+** with logical replication enabled
2. **Superuser access** (to create replication slot and publication)
3. **Kafka cluster** (Confluent Cloud or MSK)
4. **Kafka Connect** (Confluent Cloud or MSK Connect)

### Enable Logical Replication

Edit `postgresql.conf`:

```ini
# Enable logical replication
wal_level = logical

# Increase max replication slots
max_replication_slots = 10

# Increase max WAL senders
max_wal_senders = 10

# Set replica identity (include old values in UPDATE/DELETE)
# This is set per table, but ensure server is configured for it
```

Restart PostgreSQL:

```sh
sudo systemctl restart postgresql
```

### Create Replication User

```sql
-- Create dedicated user for Debezium
CREATE USER debezium WITH REPLICATION PASSWORD 'secure-password-here';

-- Grant permissions
GRANT SELECT ON ALL TABLES IN SCHEMA public TO debezium;
GRANT USAGE ON SCHEMA public TO debezium;

-- Allow reading replication slot
ALTER USER debezium WITH REPLICATION;

-- Set replica identity for tables (include old values in changes)
ALTER TABLE orders REPLICA IDENTITY FULL;
ALTER TABLE customers REPLICA IDENTITY FULL;
ALTER TABLE products REPLICA IDENTITY FULL;
```

!!! warning "Replica Identity"
    `REPLICA IDENTITY FULL` includes old values in UPDATE/DELETE events but increases WAL size. Use `REPLICA IDENTITY DEFAULT` (primary key only) if old values aren't needed.

### Create Publication

PostgreSQL publications define which tables to replicate:

```sql
-- Create publication for all tables
CREATE PUBLICATION debezium_publication FOR ALL TABLES;

-- Or, create publication for specific tables
CREATE PUBLICATION debezium_publication FOR TABLE orders, customers, products;

-- Verify
SELECT * FROM pg_publication;
```

## Deploy Debezium Connector

### Store PostgreSQL Credentials

```sh
# Set AWS profile
export AWS_PROFILE=data-engineer

# Create secret for Debezium PostgreSQL access
aws secretsmanager create-secret \
    --name postgres/debezium-replication \
    --description "PostgreSQL credentials for Debezium CDC connector" \
    --secret-string '{
        "host": "production-db.abc123.eu-west-2.rds.amazonaws.com",
        "port": "5432",
        "database": "production",
        "user": "debezium",
        "password": "secure-password-here"
    }' \
    --region eu-west-2
```

### Confluent Cloud Connector

Deploy Debezium PostgreSQL connector via Confluent Cloud UI:

**Step 1: Add Connector**
- Navigate to Connectors → Add connector
- Select "Debezium PostgreSQL CDC Source"

**Step 2: Configure Connection**
```yaml
name: postgres-cdc-orders
connector.class: io.debezium.connector.postgresql.PostgresConnector
database.hostname: production-db.abc123.eu-west-2.rds.amazonaws.com
database.port: 5432
database.user: debezium
database.password: ${aws_secretsmanager:postgres/debezium-replication:password}
database.dbname: production
database.server.name: production-postgres

# Publication (created earlier)
publication.name: debezium_publication

# Plugin (required for PostgreSQL)
plugin.name: pgoutput

# Slot name (unique identifier)
slot.name: debezium_orders_slot

# Topic naming
topic.prefix: postgres
table.include.list: public.orders,public.customers,public.products

# Schema evolution
schema.history.internal.kafka.bootstrap.servers: ${kafka.bootstrap.servers}
schema.history.internal.kafka.topic: schema-history.postgres

# Snapshot mode (initial load)
snapshot.mode: initial

# Output format
key.converter: org.apache.kafka.connect.json.JsonConverter
value.converter: org.apache.kafka.connect.json.JsonConverter
key.converter.schemas.enable: true
value.converter.schemas.enable: true
```

**Step 3: Launch Connector**

The connector will:
1. **Create replication slot** (`debezium_orders_slot`)
2. **Perform initial snapshot** (copy existing table data)
3. **Start streaming** (capture new changes from WAL)

### Verify Topics Created

```sh
# List topics
confluent kafka topic list

# Expected topics:
# - postgres.public.orders
# - postgres.public.customers
# - postgres.public.products
# - schema-history.postgres (internal)
```

### Check Connector Status

```sh
# Via Confluent Cloud UI
# Connectors → postgres-cdc-orders → Status: Running

# Or via API
confluent connect cluster describe <connector-id>
```

## Understanding CDC Events

### INSERT Event

```json
{
  "schema": { ... },
  "payload": {
    "before": null,
    "after": {
      "order_id": 12345,
      "customer_id": "cust_123",
      "order_status": "CREATED",
      "order_total": 99.99,
      "created_at": "2024-03-05T10:30:45Z"
    },
    "source": {
      "version": "2.4.0.Final",
      "connector": "postgresql",
      "name": "production-postgres",
      "ts_ms": 1709643845123,
      "snapshot": "false",
      "db": "production",
      "sequence": "[\"12345678\",\"12345680\"]",
      "schema": "public",
      "table": "orders",
      "txId": 1234,
      "lsn": 12345678,
      "xmin": null
    },
    "op": "c",
    "ts_ms": 1709643845126,
    "transaction": null
  }
}
```

**Fields:**
- `before`: Previous row state (null for INSERT)
- `after`: New row state
- `op`: Operation (`c` = create, `u` = update, `d` = delete, `r` = read/snapshot)
- `source.lsn`: Log Sequence Number (WAL position)
- `source.ts_ms`: Database transaction timestamp
- `ts_ms`: Connector processing timestamp

### UPDATE Event

```json
{
  "payload": {
    "before": {
      "order_id": 12345,
      "customer_id": "cust_123",
      "order_status": "CREATED",
      "order_total": 99.99,
      "created_at": "2024-03-05T10:30:45Z"
    },
    "after": {
      "order_id": 12345,
      "customer_id": "cust_123",
      "order_status": "SHIPPED",
      "order_total": 99.99,
      "created_at": "2024-03-05T10:30:45Z"
    },
    "source": { ... },
    "op": "u",
    "ts_ms": 1709644000000
  }
}
```

**Use case:** Audit trail showing what changed (`before` vs `after`).

### DELETE Event

```json
{
  "payload": {
    "before": {
      "order_id": 12345,
      "customer_id": "cust_123",
      "order_status": "CANCELLED",
      "order_total": 99.99,
      "created_at": "2024-03-05T10:30:45Z"
    },
    "after": null,
    "source": { ... },
    "op": "d",
    "ts_ms": 1709645000000
  }
}
```

**Important:** DELETE events include the deleted row's data in `before`. This is why you need `REPLICA IDENTITY FULL`.

## Stream to Snowflake

### Create CDC Tables

CDC data in Snowflake is typically stored as raw event history, then transformed with dbt.

```sql
-- Create schema for CDC events
USE DATABASE STREAMING;
CREATE SCHEMA IF NOT EXISTS CDC;

-- Table for raw order change events
CREATE TABLE CDC.ORDERS_CDC (
    RECORD_CONTENT VARIANT,
    RECORD_METADATA VARIANT,
    _KAFKA_OFFSET NUMBER,
    _KAFKA_PARTITION NUMBER,
    _KAFKA_TIMESTAMP TIMESTAMP_NTZ
);

-- Similar tables for other entities
CREATE TABLE CDC.CUSTOMERS_CDC (
    RECORD_CONTENT VARIANT,
    RECORD_METADATA VARIANT,
    _KAFKA_OFFSET NUMBER,
    _KAFKA_PARTITION NUMBER,
    _KAFKA_TIMESTAMP TIMESTAMP_NTZ
);
```

### Configure Snowflake Sink Connector

Deploy a Snowflake Sink Connector for each CDC topic:

```yaml
name: snowflake-sink-orders-cdc
connector.class: com.snowflake.kafka.connector.SnowflakeSinkConnector
topics: postgres.public.orders

# Snowflake connection
snowflake.url.name: your-account.eu-west-2.aws.snowflakecomputing.com
snowflake.user.name: SVC_KAFKA_CONNECTOR
snowflake.private.key: ${aws_secretsmanager:snowflake/svc-kafka-connector:private_key}
snowflake.database.name: STREAMING
snowflake.schema.name: CDC
snowflake.role.name: SVC_KAFKA_CONNECTOR

# Topic to table mapping
snowflake.topic2table.map: postgres.public.orders:ORDERS_CDC

# Buffer settings
buffer.count.records: 10000
buffer.flush.time: 60
buffer.size.bytes: 5000000

# Data format
value.converter: org.apache.kafka.connect.json.JsonConverter
value.converter.schemas.enable: true
key.converter: org.apache.kafka.connect.json.JsonConverter
```

### Query CDC Data in Snowflake

**View latest state (current snapshot):**

```sql
-- Get current state of orders table
-- (most recent event per order_id where op != 'd')
SELECT
    RECORD_CONTENT:payload.after.order_id::INT AS order_id,
    RECORD_CONTENT:payload.after.customer_id::STRING AS customer_id,
    RECORD_CONTENT:payload.after.order_status::STRING AS order_status,
    RECORD_CONTENT:payload.after.order_total::FLOAT AS order_total,
    RECORD_CONTENT:payload.after.created_at::TIMESTAMP AS created_at,
    RECORD_CONTENT:payload.ts_ms::TIMESTAMP AS last_modified
FROM STREAMING.CDC.ORDERS_CDC
QUALIFY ROW_NUMBER() OVER (
    PARTITION BY RECORD_CONTENT:payload.after.order_id
    ORDER BY RECORD_CONTENT:payload.ts_ms DESC
) = 1
AND RECORD_CONTENT:payload.op::STRING != 'd';  -- Exclude deleted rows
```

**View change history (audit trail):**

```sql
-- Show all changes to order 12345
SELECT
    RECORD_CONTENT:payload.op::STRING AS operation,
    RECORD_CONTENT:payload.before AS before_state,
    RECORD_CONTENT:payload.after AS after_state,
    RECORD_CONTENT:payload.source.ts_ms::TIMESTAMP AS change_timestamp,
    RECORD_CONTENT:payload.source.lsn::STRING AS log_position
FROM STREAMING.CDC.ORDERS_CDC
WHERE RECORD_CONTENT:payload.after.order_id::INT = 12345
   OR RECORD_CONTENT:payload.before.order_id::INT = 12345
ORDER BY change_timestamp;

-- Result:
-- operation | before_state | after_state         | change_timestamp
-- c         | null         | {order_id: 12345, status: "CREATED", ...}  | 2024-03-05 10:30:45
-- u         | {status: "CREATED"}  | {status: "SHIPPED"}   | 2024-03-05 12:15:30
-- u         | {status: "SHIPPED"}  | {status: "DELIVERED"} | 2024-03-06 09:45:20
```

**Slowly Changing Dimension (Type 2):**

```sql
-- Build SCD Type 2 from CDC events
CREATE OR REPLACE VIEW ANALYTICS.DIM_CUSTOMERS_SCD AS
SELECT
    RECORD_CONTENT:payload.after.customer_id::STRING AS customer_id,
    RECORD_CONTENT:payload.after.email::STRING AS email,
    RECORD_CONTENT:payload.after.status::STRING AS status,
    RECORD_CONTENT:payload.source.ts_ms::TIMESTAMP AS valid_from,
    LEAD(RECORD_CONTENT:payload.source.ts_ms::TIMESTAMP) OVER (
        PARTITION BY RECORD_CONTENT:payload.after.customer_id
        ORDER BY RECORD_CONTENT:payload.source.ts_ms
    ) AS valid_to,
    CASE
        WHEN LEAD(RECORD_CONTENT:payload.source.ts_ms) OVER (
            PARTITION BY RECORD_CONTENT:payload.after.customer_id
            ORDER BY RECORD_CONTENT:payload.source.ts_ms
        ) IS NULL THEN TRUE
        ELSE FALSE
    END AS is_current
FROM STREAMING.CDC.CUSTOMERS_CDC
WHERE RECORD_CONTENT:payload.op::STRING != 'd'
ORDER BY customer_id, valid_from;
```

## Schema Evolution

### Handling DDL Changes

Debezium automatically captures schema changes (ADD COLUMN, DROP COLUMN, etc.).

**Example: Adding a column**

```sql
-- In PostgreSQL production database
ALTER TABLE orders ADD COLUMN discount_amount NUMERIC(10,2) DEFAULT 0;
```

**What happens:**

1. **Debezium captures DDL** - Schema change is logged in WAL
2. **Schema history topic updated** - New schema version published
3. **New events include new field** - `after.discount_amount` appears in events
4. **Old events don't change** - Historical data remains as-is

**In Snowflake:**

```sql
-- Old events (before DDL)
SELECT RECORD_CONTENT:payload.after
FROM STREAMING.CDC.ORDERS_CDC
WHERE _KAFKA_TIMESTAMP < '2024-03-05 14:00:00'
LIMIT 1;

-- Result: {"order_id": 123, "customer_id": "cust_123", "order_total": 99.99}
-- (no discount_amount field)

-- New events (after DDL)
SELECT RECORD_CONTENT:payload.after
FROM STREAMING.CDC.ORDERS_CDC
WHERE _KAFKA_TIMESTAMP > '2024-03-05 14:00:00'
LIMIT 1;

-- Result: {"order_id": 124, "customer_id": "cust_456", "order_total": 149.99, "discount_amount": 10.00}
```

**Handling in dbt:**

```sql
-- Use COALESCE to handle missing fields
SELECT
    RECORD_CONTENT:payload.after.order_id::INT AS order_id,
    RECORD_CONTENT:payload.after.order_total::FLOAT AS order_total,
    COALESCE(RECORD_CONTENT:payload.after.discount_amount::FLOAT, 0) AS discount_amount
FROM STREAMING.CDC.ORDERS_CDC;
```

## Monitoring CDC Pipeline

### Check Replication Slot Lag

Monitor how far behind Debezium is reading the WAL:

```sql
-- In PostgreSQL
SELECT
    slot_name,
    active,
    restart_lsn,
    pg_current_wal_lsn(),
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
    ) AS replication_lag
FROM pg_replication_slots
WHERE slot_name = 'debezium_orders_slot';

-- Result:
-- slot_name             | active | restart_lsn | pg_current_wal_lsn | replication_lag
-- debezium_orders_slot  | t      | 0/12345678  | 0/12346000         | 2 MB

-- If lag > 1 GB, investigate (connector down, network issue, etc.)
```

### Check WAL Disk Usage

CDC increases WAL retention:

```sh
# In PostgreSQL data directory
du -sh pg_wal/

# Expected: < 10 GB
# If > 50 GB, check if replication slot is active
```

### Prefect Monitoring Flow

Create a flow to monitor CDC health:

```python
from prefect import flow, task
import psycopg2
import logging

logger = logging.getLogger(__name__)

@task
def check_replication_slot_lag():
    """Check if CDC replication slot is falling behind."""
    conn = psycopg2.connect(
        host="production-db.abc123.eu-west-2.rds.amazonaws.com",
        database="production",
        user="monitoring_user",
        password="password"
    )

    cursor = conn.cursor()
    cursor.execute("""
        SELECT
            slot_name,
            active,
            pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS lag_bytes
        FROM pg_replication_slots
        WHERE slot_name = 'debezium_orders_slot'
    """)

    result = cursor.fetchone()
    slot_name, active, lag_bytes = result

    if not active:
        logger.error(f"❌ Replication slot '{slot_name}' is INACTIVE!")
        raise Exception("CDC replication slot is inactive")

    lag_mb = lag_bytes / (1024 * 1024)

    if lag_mb > 1000:  # Alert if > 1 GB behind
        logger.warning(f"⚠️  CDC lag is {lag_mb:.0f} MB")
    else:
        logger.info(f"✅ CDC lag is {lag_mb:.0f} MB (healthy)")

    conn.close()

    return lag_mb

@flow(name="monitor-cdc-health")
def monitor_cdc_health():
    """Monitor PostgreSQL CDC replication health."""
    lag_mb = check_replication_slot_lag()

    # Additional checks: connector status, Kafka lag, Snowflake lag
    # ...

if __name__ == "__main__":
    monitor_cdc_health()
```

Schedule to run every 15 minutes.

## CDC vs Batch: Cost Comparison

### Scenario: 100 GB production database, 10 tables

| Component | Batch (dlt) | CDC (Debezium) |
|-----------|------------|---------------|
| **Infrastructure** | | |
| Kafka cluster | Not needed | $150/month (Confluent Cloud Basic) |
| Kafka Connect | Not needed | Included |
| Snowflake compute | $30/month (hourly dlt runs) | $50/month (continuous ingestion) |
| Snowflake storage | $20/month (snapshots) | $80/month (full history) |
| Ops time | 2 hours/month | 6 hours/month |
| **Total** | **$50 + $120 ops** = **$170/month** | **$280 + $360 ops** = **$640/month** |

**CDC premium:** $470/month (276% more expensive)

### When CDC is Worth It

| Business Value | Annual Value | CDC Annual Cost | ROI |
|----------------|-------------|----------------|-----|
| **Real-time fraud prevention** | $50,000 saved | $7,680 | 550% ROI ✅ |
| **Operational dashboards** | $10,000 saved | $7,680 | 30% ROI ⚠️ |
| **Compliance audit trails** | $20,000 (regulatory requirement) | $7,680 | 160% ROI ✅ |
| **"Nice to have" real-time** | $0 | $7,680 | -100% ROI ❌ |

**Rule of thumb:** CDC makes sense when business value exceeds $1,000/month.

## Summary

You've learned when and how to implement Change Data Capture:

- [x] **CDC fundamentals** - Captures every database change from transaction log
- [x] **Debezium deployment** - PostgreSQL CDC connector on Kafka Connect
- [x] **Event structure** - INSERT/UPDATE/DELETE with before/after states
- [x] **Snowflake replication** - Stream CDC events for real-time analytics
- [x] **Schema evolution** - Handle DDL changes automatically
- [x] **Monitoring** - Track replication slot lag and connector health
- [x] **Cost comparison** - 3-4× more expensive than batch extraction
- [x] **Use cases** - Audit trails, event sourcing, real-time replication

**Key insight:** CDC is powerful but complex. Use batch extraction (dlt) unless you need real-time replication with full change history.

## What's Next

You've completed all streaming data ingestion topics (core and optional). Consider these next steps:

### Consolidate Your Learning

Review the complete streaming pipeline you've built:
- Kafka cluster (Confluent Cloud or MSK)
- Schema Registry for data contracts
- Producers with Avro schemas
- Snowflake Sink for sub-10s latency
- Optional: Stream processing (ksqlDB/Flink)
- Optional: CDC for database replication

### Continue Building Your Stack

Explore other sections:
- **Data Transformation** - Process streaming and batch data with dbt
- **Business Intelligence** - Build real-time dashboards with your streaming data
- **Data Quality** - Validate streaming events with Great Expectations
- **Orchestration** - Enhance your Prefect workflows for production monitoring

### Return to Section Overview

Return to [Streaming Data Ingestion Overview](index.md) to review the complete section.

**Congratulations on mastering streaming data ingestion!** 🎉
