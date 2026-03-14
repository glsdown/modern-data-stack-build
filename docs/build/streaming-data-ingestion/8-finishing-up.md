# Finishing Up

On this page, you will:

- [x] Verify end-to-end latency (event production to Snowflake query)
- [x] Review total cost of ownership for Confluent Cloud approach
- [x] Use decision tree for streaming vs batch ingestion
- [x] Learn pattern for adding new topics and connectors
- [x] Complete monitoring checklist
- [x] Understand next steps and advanced topics

## Overview

You've built a complete streaming data pipeline from Kafka to Snowflake. This final page verifies performance, reviews costs, and provides patterns for extending the system.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COMPLETE STREAMING PIPELINE                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Producer              Kafka              Connector         Snowflake   │
│  ────────              ─────              ─────────         ─────────   │
│                                                                         │
│  ┌────────┐          ┌────────┐         ┌────────┐        ┌────────┐  │
│  │ Python │─────────▶│ order- │────────▶│ Snowfl │───────▶│ ORDER_ │  │
│  │ Producer│  <1s    │ events │  <5s    │ ake    │  <2s   │ EVENTS │  │
│  │        │          │ topic  │         │ Sink   │        │ table  │  │
│  └────────┘          └────────┘         └────────┘        └────────┘  │
│      │                    │                   │                 │       │
│      ▼                    ▼                   ▼                 ▼       │
│  ┌────────┐          ┌────────┐         ┌────────┐        ┌────────┐  │
│  │ Schema │          │ Schema │         │ Monitor│        │ dbt    │  │
│  │ Registry│          │ validate│         │ lag    │        │ models │  │
│  └────────┘          └────────┘         └────────┘        └────────┘  │
│                                                                         │
│  Total Latency: ~8 seconds (producer → queryable in Snowflake)         │
│  Cost: ~$200/month (Confluent Cloud Basic + Snowflake compute)         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Verify End-to-End Latency

Measure how long it takes for an event to flow from producer to Snowflake query.

### Latency Test Script

Create `test_latency.py`:

```python
#!/usr/bin/env python3
"""
Measure end-to-end latency from producer to Snowflake.
"""
import time
import json
import logging
from kafka_producer import KafkaProducer
import snowflake.connector
import boto3
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.backends import default_backend


logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


ORDER_EVENTS_SCHEMA = {
    "type": "record",
    "name": "OrderEvent",
    "namespace": "com.company.streaming",
    "fields": [
        {"name": "order_id", "type": "long"},
        {"name": "customer_id", "type": "string"},
        {
            "name": "order_status",
            "type": {
                "type": "enum",
                "name": "OrderStatus",
                "symbols": ["CREATED", "CONFIRMED", "SHIPPED", "DELIVERED", "CANCELLED"]
            }
        },
        {"name": "order_total", "type": "double"},
        {"name": "created_at", "type": "long", "logicalType": "timestamp-millis"}
    ]
}


def get_snowflake_connection():
    """Get authenticated Snowflake connection."""
    secrets_client = boto3.client('secretsmanager', region_name='eu-west-2')
    response = secrets_client.get_secret_value(SecretId='snowflake/svc-kafka-connector')
    creds = json.loads(response['SecretString'])

    private_key = serialization.load_pem_private_key(
        creds['private_key'].encode('utf-8'),
        password=None,
        backend=default_backend()
    )
    private_key_bytes = private_key.private_bytes(
        encoding=serialization.Encoding.DER,
        format=serialization.PrivateFormat.PKCS8,
        encryption_algorithm=serialization.NoEncryption()
    )

    return snowflake.connector.connect(
        account=creds['account'],
        user=creds['user'],
        private_key=private_key_bytes,
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema=creds['schema'],
        role=creds['role']
    )


def main():
    print("🚀 Starting end-to-end latency test...\n")

    # Generate unique test order ID
    test_order_id = int(time.time() * 1000)

    # Step 1: Produce event
    print(f"📤 Step 1: Producing test event (order_id={test_order_id})...")
    start_time = time.time()

    producer = KafkaProducer(
        topic='order-events',
        schema=ORDER_EVENTS_SCHEMA
    )

    event = {
        'order_id': test_order_id,
        'customer_id': 'latency_test',
        'order_status': 'CREATED',
        'order_total': 99.99,
        'created_at': int(time.time() * 1000)
    }

    producer.produce(event=event, key=str(test_order_id))
    producer.flush(timeout=10)
    producer.close()

    produce_time = time.time() - start_time
    print(f"   ✅ Produced in {produce_time:.2f}s\n")

    # Step 2: Wait for event in Snowflake
    print("⏳ Step 2: Waiting for event to appear in Snowflake...")
    conn = get_snowflake_connection()
    cursor = conn.cursor()

    max_wait = 60  # Wait up to 60 seconds
    poll_interval = 2  # Check every 2 seconds
    elapsed = 0
    found = False

    while elapsed < max_wait:
        cursor.execute(f"""
        SELECT
            RECORD_CONTENT:order_id::INT AS order_id,
            RECORD_METADATA:timestamp::TIMESTAMP AS kafka_timestamp
        FROM ORDER_EVENTS
        WHERE RECORD_CONTENT:order_id = {test_order_id}
        LIMIT 1
        """)

        result = cursor.fetchone()
        if result:
            found = True
            total_latency = time.time() - start_time
            print(f"   ✅ Event found in Snowflake!\n")
            print(f"📊 Latency Breakdown:")
            print(f"   • Produce to Kafka: {produce_time:.2f}s")
            print(f"   • Kafka to Snowflake: {total_latency - produce_time:.2f}s")
            print(f"   • Total end-to-end: {total_latency:.2f}s")
            break

        time.sleep(poll_interval)
        elapsed += poll_interval
        print(f"   ⏳ Still waiting... ({elapsed}s elapsed)")

    cursor.close()
    conn.close()

    if not found:
        print(f"   ❌ Event not found after {max_wait}s - check connector status")
        return

    # Success summary
    print(f"\n✅ Latency test complete!")
    if total_latency < 10:
        print(f"   🎉 Excellent latency: {total_latency:.2f}s")
    elif total_latency < 30:
        print(f"   ✅ Good latency: {total_latency:.2f}s")
    else:
        print(f"   ⚠️  High latency: {total_latency:.2f}s - investigate connector buffer settings")


if __name__ == '__main__':
    main()
```

### Run Latency Test

```sh
python test_latency.py
```

**Expected output:**
```
🚀 Starting end-to-end latency test...

📤 Step 1: Producing test event (order_id=1709643845123)...
   ✅ Produced in 0.45s

⏳ Step 2: Waiting for event to appear in Snowflake...
   ⏳ Still waiting... (2s elapsed)
   ⏳ Still waiting... (4s elapsed)
   ⏳ Still waiting... (6s elapsed)
   ✅ Event found in Snowflake!

📊 Latency Breakdown:
   • Produce to Kafka: 0.45s
   • Kafka to Snowflake: 7.32s
   • Total end-to-end: 7.77s

✅ Latency test complete!
   🎉 Excellent latency: 7.77s
```

### Latency Benchmarks

| Latency | Rating | Notes |
|---------|--------|-------|
| **< 10s** | Excellent | Typical for Basic tier with 60s buffer |
| **10-30s** | Good | Expected during high load or larger buffers |
| **30-60s** | Acceptable | May need buffer tuning |
| **> 60s** | Poor | Investigate connector, warehouse size, or network |

### Improving Latency

If latency is too high:

1. **Reduce buffer time:** `buffer.flush.time = 30` (from 60)
2. **Reduce buffer count:** `buffer.count.records = 5000` (from 10000)
3. **Increase connector tasks:** `tasks = 2` (parallel processing)
4. **Scale Snowflake warehouse:** XSMALL → SMALL
5. **Upgrade Confluent cluster:** Basic → Standard (multi-AZ, lower P99 latency)

## Cost Summary

### Confluent Cloud Approach (~$200/month)

| Component | Monthly Cost | Notes |
|-----------|-------------|-------|
| **Confluent Cloud Basic** | $150 | 1 CKU base cluster |
| **Data ingress** | $2.50 | 1 GB/day × 30 days × $0.08/GB |
| **Data egress** | $2.50 | Connector reads 1 GB/day |
| **Storage** | $2.50 | 30 GB retained (7 days) × $0.08/GB |
| **Schema Registry** | Included | Part of Basic tier |
| **Kafka Connect** | Included | Managed by Confluent |
| **AWS Secrets Manager** | $1.50 | 3 secrets × $0.40/month + API calls |
| **Snowflake compute** | $30-50 | INGEST_WH (XSMALL), ~2-4 hours/day active |
| **Snowflake storage** | $5 | STREAMING database, compressed |
| **Prefect Cloud** | $0 | Hobby tier (sufficient for monitoring) |
| **Developer time** | $60-120 | 1-2 hours/month monitoring (@ $60/hour) |
| **Total** | **$254-334/month** | |

### Cost Breakdown by Volume

| Daily Ingress | Monthly Confluent Cost | Monthly Snowflake Cost | Total |
|---------------|----------------------|----------------------|-------|
| **1 GB/day** | $157 | $35 | **$192** |
| **10 GB/day** | $174 | $50 | **$224** |
| **50 GB/day** | $270 | $100 | **$370** |
| **100 GB/day** | $390 | $150 | **$540** |

### Cost vs Batch Comparison

| Workload | Streaming Cost | Batch Cost (dlt) | Premium |
|----------|---------------|-----------------|---------|
| **1 GB/day** | $192/month | $5/month | $187/month |
| **10 GB/day** | $224/month | $10/month | $214/month |

**When the premium is worth it:**
- Latency reduction (hours → seconds) enables real-time decisions worth > $200/month
- Fraud prevention, dynamic pricing, operational dashboards
- Business value exceeds infrastructure cost

## Streaming vs Batch Decision Tree

Use this decision tree to choose between streaming and batch ingestion:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    STREAMING VS BATCH DECISION TREE                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Do you need data within seconds (not hours)?                           │
│  ├─ NO  → Use batch (dlt/Airbyte)                                       │
│  └─ YES → Continue                                                      │
│                                                                         │
│  Is event volume > 1,000 events/day?                                    │
│  ├─ NO  → Use batch (streaming overhead not justified)                  │
│  └─ YES → Continue                                                      │
│                                                                         │
│  Does business value exceed $200/month?                                 │
│  ├─ NO  → Use batch (streaming too expensive)                           │
│  └─ YES → Continue                                                      │
│                                                                         │
│  Can source system produce events in real-time?                         │
│  ├─ NO  → Use batch (e.g., API with rate limits, daily extracts)       │
│  └─ YES → Use streaming                                                 │
│                                                                         │
│  ✅ USE STREAMING: Kafka + Snowpipe Streaming                           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Use Cases by Ingestion Method

| Use Case | Ingestion Method | Rationale |
|----------|-----------------|-----------|
| **Clickstream analytics** | Streaming | High volume, need real-time dashboards |
| **IoT sensor data** | Streaming | Continuous events, alerting on anomalies |
| **Fraud detection** | Streaming | Sub-second decisions, prevent losses |
| **Database CDC** | Streaming | Real-time replication, audit trails |
| **Daily sales reports** | Batch | Once per day is sufficient |
| **SaaS platform data** | Batch | API rate limits, hourly/daily sync |
| **Weekly reference data** | Batch | Low frequency, small volume |
| **Historical backfills** | Batch | One-time loads, not time-sensitive |

## Adding New Topics and Connectors

Follow this pattern to add additional streaming pipelines.

### Pattern: New Topic and Connector

**Step 1: Define Avro schema**

Create `schemas/customer_events.avsc`:

```json
{
  "type": "record",
  "name": "CustomerEvent",
  "namespace": "com.company.streaming",
  "fields": [
    {"name": "customer_id", "type": "string"},
    {"name": "event_type", "type": "string"},
    {"name": "event_timestamp", "type": "long", "logicalType": "timestamp-millis"}
  ]
}
```

**Step 2: Create topic**

```sh
confluent kafka topic create customer-events \
    --partitions 3 \
    --config retention.ms=604800000
```

**Step 3: Register schema**

```sh
curl -X POST \
    -H "Content-Type: application/vnd.schemaregistry.v1+json" \
    -u "SR_API_KEY:SR_API_SECRET" \
    --data @schemas/customer_events.avsc \
    https://psrc-abc123.eu-west-2.aws.confluent.cloud/subjects/customer-events-value/versions
```

**Step 4: Deploy connector**

Via Confluent Cloud UI:
- Add connector → Snowflake Sink
- Topic: `customer-events`
- Database: `STREAMING`
- Schema: `PUBLIC`
- Table: Auto-created as `CUSTOMER_EVENTS`

**Step 5: Create producer**

```python
producer = KafkaProducer(
    topic='customer-events',
    schema=CUSTOMER_EVENTS_SCHEMA
)

producer.produce(
    event={'customer_id': 'cust_123', 'event_type': 'login', 'event_timestamp': ...},
    key='cust_123'
)
```

**Step 6: Update monitoring**

Add new topic to monitoring flow:
```python
check_throughput(topic='customer-events')
get_consumer_lag(consumer_group='snowflake-sink-customer-events')
```

## Monitoring Checklist

Ensure you have monitoring for all components:

### Infrastructure Monitoring

- [ ] **Consumer lag** — alert if lag > 10,000 messages
- [ ] **Connector status** — alert if status != RUNNING
- [ ] **Event throughput** — alert if drops > 50%
- [ ] **Confluent Cloud billing** — alert at 50%, 80% of budget
- [ ] **Snowflake credits** — resource monitor at 75%, 90%, 100%

### Data Quality Monitoring

- [ ] **Schema validation failures** — count rejected events in connector logs
- [ ] **Duplicate events** — check for duplicate order_id in Snowflake
- [ ] **Missing events** — compare producer count vs Snowflake count
- [ ] **Latency spikes** — track P50, P95, P99 latency over time
- [ ] **Data freshness** — alert if no new events in > 10 minutes

### Operational Monitoring

- [ ] **Prefect flow runs** — alert on failed monitoring flows
- [ ] **API key expiry** — rotate every 90 days
- [ ] **Snowflake warehouse auto-suspend** — verify not running when idle
- [ ] **Kafka topic retention** — ensure old data is deleted per policy
- [ ] **AWS Secrets Manager access** — audit who accessed streaming credentials

## What's Next

### Production Hardening

1. **Enable Schema Registry ACLs** — restrict who can register schemas
2. **Set up VPC peering** — (Dedicated tier) private connectivity to Snowflake
3. **Configure audit logging** — (Dedicated tier) track all cluster operations
4. **Implement dead letter queues** — capture permanently failed events
5. **Add integration tests** — end-to-end latency tests in CI/CD

### Advanced Topics (Optional)

Explore additional streaming capabilities:

#### 9. MSK Self-Hosted (Optional)

Deploy AWS MSK (Managed Streaming for Kafka) as an alternative to Confluent Cloud:
- Lower cost at high volume
- AWS-native integration (VPC, IAM, CloudWatch)
- More operational burden

See [MSK Self-Hosted](9-msk-self-hosted.md) →

#### 10. Stream Processing (Optional)

Process events in real-time before loading to Snowflake:
- **ksqlDB** — SQL queries on Kafka topics (aggregations, joins, filtering)
- **Flink** — Complex event processing, stateful transformations
- Use cases: real-time aggregations, enrichment, filtering

See [Stream Processing](10-stream-processing.md) →

#### 11. Change Data Capture (Optional)

Replicate PostgreSQL database changes to Kafka in real-time:
- **Debezium** — CDC connector for PostgreSQL, MySQL, MongoDB
- Capture INSERT, UPDATE, DELETE operations
- Real-time analytics without impacting production database

See [Change Data Capture](11-change-data-capture.md) →

## Summary

You've completed the streaming data ingestion section:

- [x] **End-to-end latency verified** — ~8 seconds from producer to Snowflake
- [x] **Cost reviewed** — ~$200/month for Confluent Cloud Basic approach
- [x] **Decision tree** — know when to use streaming vs batch
- [x] **Pattern documented** — add new topics and connectors systematically
- [x] **Monitoring checklist** — comprehensive health checks in place
- [x] **Next steps identified** — production hardening and advanced topics

**Key achievements:**

✅ Confluent Cloud cluster running (Basic tier)
✅ Snowflake STREAMING database receiving events
✅ Python producers with Avro schemas
✅ Snowpipe Streaming for sub-10s latency
✅ Prefect orchestration and monitoring
✅ End-to-end pipeline tested and verified

Your streaming infrastructure is production-ready. You can now ingest real-time events, transform with dbt, and visualise in dashboards.

## What's Next

### Within Streaming

Explore optional advanced topics:
- **MSK Self-Hosted** — AWS-native Kafka for cost optimisation at scale
- **Stream Processing** — real-time aggregations with ksqlDB or Flink
- **Change Data Capture** — replicate databases with Debezium

### Other Sections

Continue building your modern data stack:
- **[Documentation](../documentation/index.md)** — build a unified docs site across all repositories with runbooks, CI/CD, and Claude skills
- **Data Transformation** — process streaming and batch data with dbt
- **Business Intelligence** — build real-time dashboards with Metabase
- **Data Quality** — implement tests and monitoring with Great Expectations

**Congratulations on building a production streaming pipeline!** 🎉
