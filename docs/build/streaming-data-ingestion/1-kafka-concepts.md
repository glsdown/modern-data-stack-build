# Kafka Concepts

On this page, you will:

- [x] Understand Kafka's core components (topics, partitions, producers, consumers)
- [x] Learn how consumer groups enable parallel processing
- [x] Master offsets and ordering guarantees
- [x] Discover Schema Registry for data contracts
- [x] Compare Kafka to traditional message queues

## Overview

Apache Kafka is a distributed event streaming platform. Unlike traditional message queues that delete messages after consumption, Kafka treats events as a **durable, replayable log**. This fundamental difference enables powerful capabilities: time travel (replay old events), multiple independent consumers, and guaranteed ordering.

Understanding Kafka's architecture is essential before building streaming pipelines.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        KAFKA ARCHITECTURE                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Producers              Kafka Cluster           Consumers               │
│  ─────────              ─────────────           ─────────               │
│                                                                         │
│  ┌──────────┐          ┌──────────────┐        ┌──────────┐            │
│  │ Web App  │─────────▶│ Topic: orders│───────▶│ Snowflake│            │
│  │ (Python) │          │              │        │ Connector│            │
│  └──────────┘          │ Partition 0  │        └──────────┘            │
│                        │ [msg1][msg2] │                                │
│  ┌──────────┐          │              │        ┌──────────┐            │
│  │ Mobile   │─────────▶│ Partition 1  │───────▶│ Analytics│            │
│  │ App      │          │ [msg3][msg4] │        │ Consumer │            │
│  └──────────┘          │              │        └──────────┘            │
│                        │ Partition 2  │                                │
│                        │ [msg5][msg6] │        ┌──────────┐            │
│                        └──────────────┘───────▶│ ML Model │            │
│                                                │ Consumer │            │
│                                                └──────────┘            │
│                                                                         │
│  Multiple producers → One topic → Multiple independent consumers        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Topics: The Event Log

### What is a Topic?

A **topic** is a named category of events. Think of it as an append-only log that stores events in order.

**Examples:**
- `order-events` — All order-related events (created, shipped, delivered)
- `user-clicks` — Clickstream data from website
- `sensor-readings` — IoT temperature/humidity data

### Topics are Durable

Unlike traditional queues where messages disappear after consumption, Kafka topics retain events for a configured period (e.g., 7 days, 30 days, or forever).

**Benefits:**
- **Replay** — Reprocess old events if consumer logic changes
- **Multiple consumers** — Different applications read the same events independently
- **Debugging** — Inspect historical events to troubleshoot issues

**Example:**
```
Topic: order-events (retention: 7 days)

[Day 1] order_created → order_shipped → order_delivered
[Day 2] order_created → order_cancelled
[Day 3] order_created → order_shipped

Day 8: Day 1 events automatically deleted
```

### Topic Configuration

```sh
# Create a topic with 3 partitions, 7-day retention
kafka-topics --create \
    --topic order-events \
    --partitions 3 \
    --replication-factor 3 \
    --config retention.ms=604800000  # 7 days in milliseconds
```

**Key settings:**
- `partitions` — Number of parallel streams (more = higher throughput)
- `replication-factor` — Copies of data for fault tolerance (3 = survive 2 broker failures)
- `retention.ms` — How long to keep events (7 days = 604800000 ms)

## Partitions: Parallelism and Ordering

### What is a Partition?

A **partition** is a subdivision of a topic. Each partition is an ordered, immutable sequence of events.

**Why partitions?**
- **Parallelism** — Multiple consumers read different partitions simultaneously
- **Scalability** — Distribute load across many brokers
- **Ordering** — Events in the same partition are guaranteed to be in order

### Partition Structure

```
Topic: order-events (3 partitions)

Partition 0:
[offset 0] {"order_id": 101, "status": "created"}
[offset 1] {"order_id": 104, "status": "shipped"}
[offset 2] {"order_id": 107, "status": "delivered"}

Partition 1:
[offset 0] {"order_id": 102, "status": "created"}
[offset 1] {"order_id": 105, "status": "cancelled"}

Partition 2:
[offset 0] {"order_id": 103, "status": "created"}
[offset 1] {"order_id": 106, "status": "shipped"}
```

**Offset** — Sequential ID for each message in a partition (starts at 0)

### Partition Keys

Producers use a **partition key** to determine which partition receives an event.

**Strategy 1: Key-based partitioning**
```python
# All events for order_id 101 go to the same partition
producer.send(
    topic='order-events',
    key=str(order_id).encode('utf-8'),  # Partition key
    value=json.dumps(event).encode('utf-8')
)
```

**Result:** All events for `order_id=101` are in the same partition, guaranteeing order.

**Strategy 2: Round-robin (no key)**
```python
# Events distributed evenly across partitions
producer.send(
    topic='order-events',
    value=json.dumps(event).encode('utf-8')  # No key
)
```

**Result:** Maximum throughput, but no ordering guarantee across partitions.

### Ordering Guarantees

✅ **Guaranteed:** Events in the **same partition** are ordered
❌ **Not guaranteed:** Events across **different partitions** may arrive out of order

**Example:**

```
Order 101 events (same partition key = order_id):
Partition 0: created → shipped → delivered ✓ Ordered

Order 102 and 103 (different partitions):
Partition 1: Order 102 created (timestamp 10:00:01)
Partition 2: Order 103 created (timestamp 10:00:00)

Consumer may see Order 103 before Order 102 ✗ Not globally ordered
```

**Best practice:** Use partition keys (e.g., `user_id`, `order_id`) when event order matters.

## Producers: Writing Events

### What is a Producer?

A **producer** is an application that writes events to a Kafka topic.

**Example: Python producer**

```python
from confluent_kafka import Producer
import json

# Configure producer
conf = {
    'bootstrap.servers': 'pkc-abc123.eu-west-2.aws.confluent.cloud:9092',
    'security.protocol': 'SASL_SSL',
    'sasl.mechanism': 'PLAIN',
    'sasl.username': 'API_KEY',
    'sasl.password': 'API_SECRET'
}

producer = Producer(conf)

# Produce an event
event = {
    'order_id': 101,
    'customer_id': 'cust_456',
    'status': 'created',
    'timestamp': '2026-02-20T10:00:00Z'
}

producer.produce(
    topic='order-events',
    key=str(event['order_id']).encode('utf-8'),  # Partition by order_id
    value=json.dumps(event).encode('utf-8'),
    callback=delivery_report  # Confirm delivery
)

producer.flush()  # Wait for all messages to be sent
```

### Producer Guarantees

**At-least-once delivery** (default):
- Events are retried on failure
- May result in duplicates if retry succeeds after timeout

**Exactly-once delivery** (optional, higher overhead):
- Idempotent producer + transactional writes
- No duplicates, but slower

**Configuration:**
```python
conf = {
    'acks': 'all',  # Wait for all replicas to acknowledge
    'retries': 10,  # Retry up to 10 times
    'enable.idempotence': True  # Exactly-once semantics
}
```

## Consumers: Reading Events

### What is a Consumer?

A **consumer** is an application that reads events from a Kafka topic.

**Example: Python consumer**

```python
from confluent_kafka import Consumer

conf = {
    'bootstrap.servers': 'pkc-abc123.eu-west-2.aws.confluent.cloud:9092',
    'group.id': 'snowflake-sink',  # Consumer group
    'auto.offset.reset': 'earliest'  # Start from beginning
}

consumer = Consumer(conf)
consumer.subscribe(['order-events'])

while True:
    msg = consumer.poll(1.0)  # Poll for messages (1 second timeout)

    if msg is None:
        continue
    if msg.error():
        print(f"Error: {msg.error()}")
        continue

    # Process event
    event = json.loads(msg.value().decode('utf-8'))
    print(f"Received order {event['order_id']}: {event['status']}")

    # Commit offset (mark as processed)
    consumer.commit()
```

### Offsets: Tracking Progress

An **offset** is the position of a consumer in a partition.

```
Partition 0: [0][1][2][3][4][5][6][7][8][9]
                        ↑
                  Consumer offset: 4
                  (Next message to read: 5)
```

**Offset management:**

**Auto-commit** (default):
```python
conf = {'enable.auto.commit': True}  # Commit every 5 seconds
```

**Manual commit** (more control):
```python
conf = {'enable.auto.commit': False}

# Process message
process_event(msg)

# Commit offset only after successful processing
consumer.commit()
```

**Best practice:** Use manual commit to avoid losing messages if consumer crashes mid-processing.

## Consumer Groups: Load Balancing

### What is a Consumer Group?

A **consumer group** is a set of consumers that share the work of reading a topic.

**How it works:**
- Each partition is consumed by exactly one consumer in the group
- If a consumer fails, its partitions are reassigned to others

### Example: 3 Consumers, 3 Partitions

```
Topic: order-events (3 partitions)

Consumer Group: snowflake-sink

┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│ Consumer 1   │       │ Consumer 2   │       │ Consumer 3   │
│ reads        │       │ reads        │       │ reads        │
│ Partition 0  │       │ Partition 1  │       │ Partition 2  │
└──────────────┘       └──────────────┘       └──────────────┘
```

**Benefits:**
- **Parallel processing** — 3 consumers = 3× throughput
- **Fault tolerance** — If Consumer 1 crashes, Partition 0 is reassigned to Consumer 2 or 3

### Example: More Consumers Than Partitions

```
Topic: order-events (3 partitions)

Consumer Group: snowflake-sink (5 consumers)

┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│ Consumer 1   │       │ Consumer 2   │       │ Consumer 3   │
│ Partition 0  │       │ Partition 1  │       │ Partition 2  │
└──────────────┘       └──────────────┘       └──────────────┘

┌──────────────┐       ┌──────────────┐
│ Consumer 4   │       │ Consumer 5   │
│ (idle)       │       │ (idle)       │
└──────────────┘       └──────────────┘
```

**Result:** Consumers 4 and 5 are idle. You can't have more active consumers than partitions.

**Best practice:** Number of consumers ≤ number of partitions for optimal utilisation.

### Multiple Consumer Groups

Different applications can read the same topic independently:

```
Topic: order-events

Consumer Group 1: snowflake-sink (writes to Snowflake)
Consumer Group 2: analytics-engine (real-time aggregations)
Consumer Group 3: fraud-detector (ML model scoring)

Each group maintains its own offsets independently.
```

## Schema Registry: Data Contracts

### Why Schema Registry?

**Problem:** Producers and consumers need to agree on event structure.

**Without Schema Registry:**
```python
# Producer changes event structure
producer.send({'order_id': 101, 'total': 99.99})  # Old format

producer.send({'order_id': 102, 'total_amount': 120.50})  # New format

# Consumer breaks on new format
event = json.loads(msg)
print(event['total'])  # KeyError: 'total' (field renamed!)
```

**With Schema Registry:**
- Schemas are versioned and stored centrally
- Producers must use a valid schema
- Consumers use schema to deserialise correctly
- Backward/forward compatibility enforced

### Schema Formats

#### Avro (Recommended)

**Binary format, compact, schema evolution support**

```json
{
  "type": "record",
  "name": "Order",
  "namespace": "com.company.orders",
  "fields": [
    {"name": "order_id", "type": "long"},
    {"name": "customer_id", "type": "string"},
    {"name": "total", "type": "double"},
    {"name": "status", "type": "string"},
    {"name": "created_at", "type": "long", "logicalType": "timestamp-millis"}
  ]
}
```

**Pros:**
- Compact (binary encoding, ~40% smaller than JSON)
- Schema evolution (add fields with defaults without breaking consumers)
- Fast serialisation/deserialisation

**Cons:**
- Not human-readable (need tools to inspect)
- More complex than JSON

**Use when:** Production workloads with high throughput

#### Protobuf

**Google's binary format, similar to Avro**

```protobuf
syntax = "proto3";

message Order {
  int64 order_id = 1;
  string customer_id = 2;
  double total = 3;
  string status = 4;
  int64 created_at = 5;
}
```

**Pros:**
- Backward/forward compatible
- Code generation for multiple languages
- Very fast

**Cons:**
- More boilerplate than Avro
- Schema Registry support added later (less mature)

**Use when:** Already using Protobuf elsewhere, need code generation

#### JSON Schema

**JSON with schema validation**

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "order_id": {"type": "integer"},
    "customer_id": {"type": "string"},
    "total": {"type": "number"},
    "status": {"type": "string"},
    "created_at": {"type": "string", "format": "date-time"}
  },
  "required": ["order_id", "customer_id", "total"]
}
```

**Pros:**
- Human-readable
- Familiar to developers
- Easy debugging

**Cons:**
- Larger message size (~2-3× larger than Avro)
- Slower serialisation
- Weaker schema evolution

**Use when:** Development/debugging, low throughput, human inspection required

### Schema Evolution

**Backward compatibility:** New consumers can read old events

```avro
// Old schema (v1)
{
  "fields": [
    {"name": "order_id", "type": "long"},
    {"name": "total", "type": "double"}
  ]
}

// New schema (v2) - backward compatible
{
  "fields": [
    {"name": "order_id", "type": "long"},
    {"name": "total", "type": "double"},
    {"name": "currency", "type": "string", "default": "GBP"}  // New field with default
  ]
}
```

**Forward compatibility:** Old consumers can read new events (new fields ignored)

**Schema Registry enforces these rules**, preventing breaking changes.

## Kafka Connect: Pre-Built Integrations

### What is Kafka Connect?

**Kafka Connect** is a framework for integrating Kafka with external systems using **connectors**.

**Source connectors** — Pull data into Kafka:
- Database CDC (Debezium)
- Cloud storage (S3, GCS)
- Message queues (RabbitMQ, SQS)

**Sink connectors** — Push data from Kafka:
- Snowflake (Snowpipe Streaming)
- Elasticsearch
- S3 (data lake)

### Example: Snowflake Sink Connector

```json
{
  "name": "snowflake-sink",
  "config": {
    "connector.class": "com.snowflake.kafka.connector.SnowflakeSinkConnector",
    "topics": "order-events",
    "snowflake.url.name": "your-account.snowflakecomputing.com",
    "snowflake.user.name": "SVC_KAFKA_CONNECTOR",
    "snowflake.private.key": "...",
    "snowflake.database.name": "STREAMING",
    "snowflake.schema.name": "PUBLIC",
    "buffer.count.records": "10000",
    "buffer.flush.time": "60",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "io.confluent.connect.avro.AvroConverter"
  }
}
```

**Benefits:**
- No custom code needed
- Automatic retries and offset management
- Scaling handled by Kafka Connect

**You'll configure this in page 5: Kafka Connect Snowflake.**

## Kafka vs Traditional Message Queues

| Feature | Kafka | RabbitMQ/SQS |
|---------|-------|--------------|
| **Durability** | Events stored for days/weeks | Messages deleted after consumption |
| **Replay** | ✅ Reprocess old events | ❌ Once consumed, gone forever |
| **Multiple consumers** | ✅ Many independent consumer groups | ❌ One consumer group only |
| **Ordering** | ✅ Per-partition ordering | ⚠️ Limited (FIFO queues in SQS, complex in RabbitMQ) |
| **Throughput** | ✅ Millions of events/sec | ⚠️ Thousands of messages/sec |
| **Latency** | ⚠️ Milliseconds (batch-oriented) | ✅ Sub-millisecond (message-oriented) |
| **Use case** | Event streaming, data pipelines | Task queues, job processing |

**When to use Kafka:**
- High-throughput event streams
- Multiple consumers need the same data
- Need to replay historical events
- Building real-time analytics pipelines

**When to use RabbitMQ/SQS:**
- Task queues (job processing)
- Low-latency message delivery
- Simple pub/sub without persistence
- Don't need replay capability

## Summary

You've learned Kafka fundamentals:

- [x] **Topics** — Append-only logs that retain events for a configured period
- [x] **Partitions** — Enable parallelism and guarantee ordering within a partition
- [x] **Producers** — Write events to topics with partition keys for ordering
- [x] **Consumers** — Read events, track offsets, commit progress
- [x] **Consumer groups** — Load balance across multiple consumers
- [x] **Schema Registry** — Enforce data contracts with Avro/Protobuf/JSON Schema
- [x] **Kafka Connect** — Pre-built connectors for databases, warehouses, cloud storage

Kafka's durability and replayability make it ideal for streaming data pipelines.

## What's Next

Compare deployment options: Confluent Cloud, AWS MSK, or self-hosted Redpanda.

Continue to [Deployment Options](2-deployment-options.md) →
