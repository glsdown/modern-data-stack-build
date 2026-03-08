# Kafka Connect Snowflake

On this page, you will:

- [x] Understand Snowpipe Streaming vs Snowpipe (batch)
- [x] Deploy Snowflake Sink Connector via Confluent Cloud UI
- [x] Configure buffer settings (count and time thresholds)
- [x] Test with sample events
- [x] Monitor connector status and throughput
- [x] Troubleshoot common issues

## Overview

The **Snowflake Sink Connector** streams events from Kafka topics into Snowflake tables in near real-time. It uses **Snowpipe Streaming** for low-latency ingestion (seconds, not minutes).

You'll deploy the connector via Confluent Cloud's managed Kafka Connect service, configure buffer settings, and verify end-to-end data flow.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    KAFKA CONNECT ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Kafka Topic                Connector               Snowflake           │
│  ───────────                ─────────               ─────────           │
│                                                                         │
│  ┌────────────┐           ┌────────────┐           ┌────────────┐      │
│  │ order-     │──────────▶│ Snowflake  │──────────▶│ STREAMING  │      │
│  │ events     │           │ Sink       │           │ .ORDER_    │      │
│  │            │           │ Connector  │           │ EVENTS     │      │
│  │ Partition 0│           │            │           │            │      │
│  │ Partition 1│           │ Buffer:    │           │ Snowpipe   │      │
│  │ Partition 2│           │ 10k records│           │ Streaming  │      │
│  └────────────┘           │ or 60 sec  │           │            │      │
│                           └────────────┘           └────────────┘      │
│                                  │                         │            │
│                                  ▼                         ▼            │
│                           ┌────────────┐           ┌────────────┐      │
│                           │ Credentials│           │ Sub-second │      │
│                           │ from AWS   │           │ latency    │      │
│                           │ Secrets    │           │            │      │
│                           └────────────┘           └────────────┘      │
│                                                                         │
│  Flow: Events → Connector buffer → Flush → Snowpipe Streaming → Table  │
│  Latency: Seconds (not hours like batch)                               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Snowpipe Streaming vs Snowpipe (Batch)

### Snowpipe (Batch)

Traditional Snowpipe loads files from S3:

- **Trigger:** S3 event notification (file created)
- **Latency:** 1-2 minutes (file upload + processing)
- **Use case:** Batch ingestion from dlt, Airbyte

**How it works:**
```
dlt → S3 file → SQS notification → Snowpipe → Snowflake table
```

### Snowpipe Streaming (Real-Time)

Snowpipe Streaming ingests data directly via API:

- **Trigger:** Continuous stream from Kafka Connect
- **Latency:** Sub-second to a few seconds
- **Use case:** Real-time event ingestion

**How it works:**
```
Kafka → Kafka Connect → Snowpipe Streaming API → Snowflake table
```

### Comparison

| Feature | Snowpipe (Batch) | Snowpipe Streaming |
|---------|-----------------|-------------------|
| **Latency** | 1-2 minutes | < 10 seconds |
| **Trigger** | File in S3 | Kafka Connect push |
| **Buffer** | File size | Record count or time |
| **Cost** | $0.06 per 1,000 files | Included in compute |
| **Use case** | Batch (dlt, Airbyte) | Streaming (Kafka) |

**Key difference:** Snowpipe Streaming bypasses file staging, reducing latency from minutes to seconds.

## Deploy Snowflake Sink Connector

### Via Confluent Cloud UI

1. Navigate to your cluster (`streaming-cluster`)
2. Click **Connectors** in the left sidebar
3. Click **+ Add connector**
4. Search for **Snowflake Sink**
5. Click **Snowflake Sink** tile

### Configure Connector: Kafka Credentials

**Step 1: Authentication**

- **Kafka cluster:** `streaming-cluster` (auto-selected)
- **API key:** Use existing cluster API key

### Configure Connector: Snowflake Credentials

**Step 2: Snowflake connection**

Retrieve credentials from AWS Secrets Manager:

```sh
aws secretsmanager get-secret-value \
    --secret-id snowflake/svc-kafka-connector \
    --region eu-west-2 \
    --query SecretString \
    --output text | jq -r .
```

Fill in the form:

- **Snowflake URL:** `your-account.eu-west-2.aws.snowflakecomputing.com`
- **User name:** `SVC_KAFKA_CONNECTOR`
- **Private key:** Paste full private key including headers:
  ```
  -----BEGIN RSA PRIVATE KEY-----
  MIIJKAIBAAKCAgEA...
  -----END RSA PRIVATE KEY-----
  ```
- **Database name:** `STREAMING`
- **Schema name:** `PUBLIC`
- **Role name:** `SVC_KAFKA_CONNECTOR`

!!! warning "Private Key Format"
    Paste the **entire** private key including `-----BEGIN RSA PRIVATE KEY-----` and `-----END RSA PRIVATE KEY-----` headers. The connector needs the full PEM format.

### Configure Connector: Topics

**Step 3: Select topics**

- **Topics to consume:** `order-events` (you'll create this topic next)
- **Input Kafka record value format:** `AVRO` (Schema Registry)
- **Input Kafka record key format:** `STRING`

### Configure Connector: Buffer Settings

**Step 4: Sizing**

- **Buffer count records:** `10000` — Flush after 10,000 records
- **Buffer flush time:** `60` — Or flush after 60 seconds (whichever comes first)
- **Tasks:** `1` — Number of parallel workers (increase for higher throughput)

**Buffer tuning:**

| Workload | Buffer Count | Buffer Time | Rationale |
|----------|-------------|-------------|-----------|
| **Low volume** (< 100/min) | 1000 | 60 sec | Flush frequently for low latency |
| **Medium volume** (1000/min) | 10000 | 60 sec | Balance latency and efficiency |
| **High volume** (10000+/min) | 50000 | 30 sec | Larger batches for throughput |

**Trade-off:**
- **Smaller buffers** → Lower latency, more frequent flushes (higher cost)
- **Larger buffers** → Higher latency, fewer flushes (better efficiency)

### Configure Connector: Table Creation

**Step 5: Configuration**

- **Snowflake metadata:** Default (RECORD_METADATA, RECORD_CONTENT columns)
- **Auto-create tables:** `true` — Connector creates tables automatically
- **Auto-evolve schema:** `false` — Require manual schema changes (safer)

!!! info "Auto-Created Table Structure"
    The connector creates tables with:
    - `RECORD_METADATA` (variant) — Kafka metadata (topic, partition, offset, timestamp)
    - `RECORD_CONTENT` (variant) — Event payload (JSON/Avro data)

    You'll transform these variant columns into typed columns with dbt.

### Configure Connector: Review

**Step 6: Connector name and review**

- **Connector name:** `snowflake-sink-order-events`
- **Connector class:** `SnowflakeSink` (auto-selected)

Review configuration and click **Launch**.

## Create Kafka Topic

Before testing, create the `order-events` topic.

### Via Confluent Cloud UI

1. Navigate to **Topics** in your cluster
2. Click **+ Add topic**
3. Configure:
    - **Topic name:** `order-events`
    - **Partitions:** `3` (allows 3 parallel consumers)
    - **Retention:** `7 days` (604800000 ms)
    - **Cleanup policy:** `delete` (remove old messages)
4. Click **Create**

### Via CLI

```sh
# Use your cluster
confluent kafka cluster use lkc-xyz789

# Create topic
confluent kafka topic create order-events \
    --partitions 3 \
    --config retention.ms=604800000
```

**Expected output:**
```
Created topic "order-events".
```

### Verify Topic

```sh
confluent kafka topic describe order-events
```

**Expected output:**
```
Topic Name: order-events
Partitions: 3
Replication Factor: 3
Retention: 7 days
```

## Test with Sample Events

Produce test events to verify the connector works.

### Register Avro Schema

Create `order_events_schema.avsc`:

```json
{
  "type": "record",
  "name": "OrderEvent",
  "namespace": "com.company.streaming",
  "fields": [
    {
      "name": "order_id",
      "type": "long",
      "doc": "Unique order identifier"
    },
    {
      "name": "customer_id",
      "type": "string",
      "doc": "Customer identifier"
    },
    {
      "name": "order_status",
      "type": {
        "type": "enum",
        "name": "OrderStatus",
        "symbols": ["CREATED", "CONFIRMED", "SHIPPED", "DELIVERED", "CANCELLED"]
      },
      "doc": "Current order status"
    },
    {
      "name": "order_total",
      "type": "double",
      "doc": "Order total in GBP"
    },
    {
      "name": "created_at",
      "type": "long",
      "logicalType": "timestamp-millis",
      "doc": "Event timestamp (Unix milliseconds)"
    }
  ]
}
```

Register the schema:

```sh
# Get Schema Registry credentials
aws secretsmanager get-secret-value \
    --secret-id confluent/schema-registry \
    --region eu-west-2 \
    --query SecretString \
    --output text | jq -r .

# Register schema
curl -X POST \
    -H "Content-Type: application/vnd.schemaregistry.v1+json" \
    -u "SR_API_KEY:SR_API_SECRET" \
    --data @order_events_schema.avsc \
    https://psrc-abc123.eu-west-2.aws.confluent.cloud/subjects/order-events-value/versions
```

**Expected output:**
```json
{"id": 1}
```

### Python Producer with Avro

Create `produce_order_events.py`:

```python
#!/usr/bin/env python3
"""
Produce sample order events to Kafka with Avro serialisation.
"""
import json
import time
import boto3
from confluent_kafka import Producer
from confluent_kafka.schema_registry import SchemaRegistryClient
from confluent_kafka.schema_registry.avro import AvroSerializer
from confluent_kafka.serialization import SerializationContext, MessageField


def get_kafka_config():
    """Retrieve Kafka credentials from AWS Secrets Manager."""
    session = boto3.session.Session()
    client = session.client(service_name='secretsmanager', region_name='eu-west-2')
    response = client.get_secret_value(SecretId='confluent/kafka-cluster')
    secret = json.loads(response['SecretString'])
    return {
        'bootstrap.servers': secret['bootstrap_servers'],
        'security.protocol': 'SASL_SSL',
        'sasl.mechanisms': 'PLAIN',
        'sasl.username': secret['api_key'],
        'sasl.password': secret['api_secret']
    }


def get_schema_registry_config():
    """Retrieve Schema Registry credentials."""
    session = boto3.session.Session()
    client = session.client(service_name='secretsmanager', region_name='eu-west-2')
    response = client.get_secret_value(SecretId='confluent/schema-registry')
    secret = json.loads(response['SecretString'])
    return {
        'url': secret['url'],
        'basic.auth.user.info': f"{secret['api_key']}:{secret['api_secret']}"
    }


def delivery_report(err, msg):
    """Callback for delivery confirmation."""
    if err:
        print(f'❌ Failed: {err}')
    else:
        print(f'✅ Delivered to {msg.topic()} [{msg.partition()}] @ offset {msg.offset()}')


def main():
    print("🔗 Connecting to Confluent Cloud...")

    # Configure producer
    producer_config = get_kafka_config()
    producer = Producer(producer_config)

    # Configure Schema Registry
    sr_config = get_schema_registry_config()
    schema_registry_client = SchemaRegistryClient(sr_config)

    # Load Avro schema
    with open('order_events_schema.avsc', 'r') as f:
        schema_str = f.read()

    avro_serialiser = AvroSerializer(
        schema_registry_client,
        schema_str,
        lambda obj, ctx: obj  # Object is already a dict
    )

    # Produce sample events
    sample_events = [
        {
            'order_id': 1001,
            'customer_id': 'cust_abc123',
            'order_status': 'CREATED',
            'order_total': 99.99,
            'created_at': int(time.time() * 1000)
        },
        {
            'order_id': 1002,
            'customer_id': 'cust_def456',
            'order_status': 'CONFIRMED',
            'order_total': 149.50,
            'created_at': int(time.time() * 1000)
        },
        {
            'order_id': 1003,
            'customer_id': 'cust_ghi789',
            'order_status': 'SHIPPED',
            'order_total': 75.25,
            'created_at': int(time.time() * 1000)
        }
    ]

    print(f"📤 Producing {len(sample_events)} events...")

    for event in sample_events:
        producer.produce(
            topic='order-events',
            key=str(event['order_id']),
            value=avro_serialiser(event, SerializationContext('order-events', MessageField.VALUE)),
            callback=delivery_report
        )

    # Wait for delivery
    producer.flush(timeout=10)

    print("✅ All events produced!")


if __name__ == '__main__':
    main()
```

### Install Dependencies

```sh
uv add "confluent-kafka[avro]" boto3
```

### Run Producer

```sh
python produce_order_events.py
```

**Expected output:**
```
🔗 Connecting to Confluent Cloud...
📤 Producing 3 events...
✅ Delivered to order-events [0] @ offset 0
✅ Delivered to order-events [1] @ offset 0
✅ Delivered to order-events [2] @ offset 0
✅ All events produced!
```

## Verify Data in Snowflake

Check that events arrived in Snowflake.

### Query Snowflake

```sql
USE ROLE ANALYTICS_DEVELOPER;
USE DATABASE STREAMING;
USE SCHEMA PUBLIC;

-- Show tables (connector auto-creates table)
SHOW TABLES;

-- Result:
-- ORDER_EVENTS

-- Query events
SELECT * FROM ORDER_EVENTS LIMIT 10;

-- View metadata
SELECT
    RECORD_METADATA:topic::STRING AS topic,
    RECORD_METADATA:partition::INT AS partition,
    RECORD_METADATA:offset::INT AS offset,
    RECORD_METADATA:timestamp::TIMESTAMP AS kafka_timestamp,
    RECORD_CONTENT:order_id::INT AS order_id,
    RECORD_CONTENT:customer_id::STRING AS customer_id,
    RECORD_CONTENT:order_status::STRING AS order_status,
    RECORD_CONTENT:order_total::FLOAT AS order_total
FROM ORDER_EVENTS
ORDER BY RECORD_METADATA:timestamp DESC;
```

**Expected result:**
```
topic        | partition | offset | kafka_timestamp          | order_id | customer_id  | order_status | order_total
-------------+-----------+--------+-------------------------+----------+--------------+--------------+-------------
order-events | 0         | 0      | 2026-03-05 12:30:15.123 | 1001     | cust_abc123  | CREATED      | 99.99
order-events | 1         | 0      | 2026-03-05 12:30:15.124 | 1002     | cust_def456  | CONFIRMED    | 149.50
order-events | 2         | 0      | 2026-03-05 12:30:15.125 | 1003     | cust_ghi789  | SHIPPED      | 75.25
```

!!! success "Streaming Verified!"
    Events are flowing from Kafka → Connector → Snowflake in seconds!

## Monitor Connector Status

### Via Confluent Cloud UI

1. Navigate to **Connectors** in your cluster
2. Click `snowflake-sink-order-events`
3. View:
    - **Status:** Running / Failed / Paused
    - **Throughput:** Records/sec, Bytes/sec
    - **Lag:** Consumer lag per partition
    - **Errors:** Recent failures

### Via CLI

```sh
# List connectors
confluent connect cluster list

# Describe connector
confluent connect cluster describe <connector-id>

# View connector logs
confluent connect cluster logs <connector-id>
```

### Key Metrics

| Metric | What it Means | Good Value |
|--------|---------------|-----------|
| **Status** | Connector health | Running |
| **Tasks running** | Active workers | 1+ (matches config) |
| **Records processed** | Total events ingested | Increasing |
| **Consumer lag** | Events waiting | < 1000 (low lag) |
| **Error rate** | Failed deliveries | 0% |

!!! warning "High Consumer Lag"
    If lag grows continuously:
    - Increase `tasks` (parallel workers)
    - Increase `buffer.count.records` (larger batches)
    - Check Snowflake warehouse size (may be slow)

## Troubleshooting Common Issues

### Error: `Authentication failed`

**Cause:** Invalid Snowflake credentials (user, private key, role)

**Fix:**
1. Verify credentials in AWS Secrets Manager
2. Test authentication with Python script (page 4)
3. Re-enter private key in connector config (full PEM format)

### Error: `Insufficient privileges`

**Cause:** Service account lacks write permission to STREAMING database

**Fix:**
```sql
-- Check grants
SHOW GRANTS TO ROLE SVC_KAFKA_CONNECTOR;

-- Should include STREAMING_DB_WRITER
-- If missing, re-run Terraform apply
```

### Error: `Schema Registry authentication failed`

**Cause:** Schema Registry API key invalid or missing

**Fix:**
1. Verify Schema Registry credentials in AWS Secrets Manager
2. Re-generate Schema Registry API key in Confluent Cloud
3. Update connector configuration

### Warning: `Consumer lag increasing`

**Cause:** Connector can't keep up with event production rate

**Fix:**
1. **Increase tasks:** Scale connector workers (2-4 tasks)
2. **Increase buffer size:** `buffer.count.records = 50000`
3. **Increase Snowflake warehouse:** Scale up INGEST_WH (XSMALL → SMALL)

### Error: `Table not found`

**Cause:** Connector failed to auto-create table

**Fix:**
1. Ensure `auto.create.tables = true` in connector config
2. Manually create table:
   ```sql
   CREATE TABLE STREAMING.PUBLIC.ORDER_EVENTS (
       RECORD_METADATA VARIANT,
       RECORD_CONTENT VARIANT
   );
   ```

### Data arriving but malformed

**Cause:** Schema mismatch between producer and Schema Registry

**Fix:**
1. Verify schema in Schema Registry matches producer schema
2. Check compatibility mode (BACKWARD, FORWARD, FULL)
3. Re-register schema with correct version

## Summary

You've deployed the Snowflake Kafka Connector:

- [x] **Snowpipe Streaming** — Sub-second latency from Kafka to Snowflake
- [x] **Connector deployed** — via Confluent Cloud managed Kafka Connect
- [x] **Credentials configured** — retrieved from AWS Secrets Manager
- [x] **Buffer settings tuned** — 10,000 records or 60 seconds
- [x] **Topic created** — `order-events` with 3 partitions
- [x] **Schema registered** — Avro schema in Schema Registry
- [x] **Test events produced** — Python producer with Avro serialisation
- [x] **Data verified** — events visible in Snowflake within seconds
- [x] **Monitoring configured** — connector status and throughput metrics

Your streaming pipeline is operational. Next, build a production-grade Python producer integrated with Prefect.

## What's Next

Create Python producers with Avro schemas and integrate with Prefect tasks.

Continue to [Producing Events](6-producing-events.md) →
