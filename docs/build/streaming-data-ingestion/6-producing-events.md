# Producing Events

On this page, you will:

- [x] Build production Python producers with confluent-kafka library
- [x] Register and version Avro schemas in Schema Registry
- [x] Produce events with serialisation and partition keys
- [x] Handle errors, retries, and delivery guarantees
- [x] Integrate producers with Prefect tasks
- [x] Test event-driven workflows

## Overview

Producers are applications that write events to Kafka topics. In this guide, you'll build a production-grade producer that:

- Retrieves credentials from AWS Secrets Manager
- Registers Avro schemas with Schema Registry
- Serialises events with type safety
- Handles errors and retries
- Integrates with Prefect for orchestration

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PRODUCER ARCHITECTURE                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Application              Producer                   Kafka              │
│  ───────────              ────────                   ─────              │
│                                                                         │
│  ┌────────────┐         ┌────────────┐           ┌────────────┐        │
│  │ Prefect    │────────▶│ Python     │──────────▶│ order-     │        │
│  │ Task       │         │ Producer   │           │ events     │        │
│  │            │         │            │           │ topic      │        │
│  │ Scheduled  │         │ • Get      │           │            │        │
│  │ or event-  │         │   creds    │           │ Partition  │        │
│  │ driven     │         │ • Register │           │ by order_id│        │
│  └────────────┘         │   schema   │           └────────────┘        │
│                         │ • Serialise│                  │              │
│                         │ • Retry    │                  ▼              │
│                         └────────────┘           ┌────────────┐        │
│                                                  │ Schema     │        │
│                                                  │ Registry   │        │
│                                                  │ (Avro)     │        │
│                                                  └────────────┘        │
│                                                                         │
│  Pattern: Task → Producer → Schema validation → Kafka → Connector      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Python Producer Class

Create a reusable producer class with schema management, error handling, and retries.

### Producer Module

Create `kafka_producer.py`:

```python
"""
Production-grade Kafka producer with Avro schema support.
"""
import json
import logging
from typing import Dict, Any, Optional, Callable
import boto3
from confluent_kafka import Producer, KafkaException
from confluent_kafka.schema_registry import SchemaRegistryClient
from confluent_kafka.schema_registry.avro import AvroSerializer
from confluent_kafka.serialization import SerializationContext, MessageField


logger = logging.getLogger(__name__)


class KafkaProducer:
    """
    Kafka producer with Schema Registry integration.

    Features:
    - Retrieves credentials from AWS Secrets Manager
    - Registers Avro schemas
    - Handles serialisation and delivery
    - Retries on transient failures
    """

    def __init__(
        self,
        topic: str,
        schema: Dict[str, Any],
        aws_region: str = 'eu-west-2',
        kafka_secret_name: str = 'confluent/kafka-cluster',
        schema_registry_secret_name: str = 'confluent/schema-registry'
    ):
        """
        Initialise Kafka producer.

        Args:
            topic: Kafka topic name
            schema: Avro schema as dict
            aws_region: AWS region for Secrets Manager
            kafka_secret_name: Secret name for Kafka credentials
            schema_registry_secret_name: Secret name for Schema Registry credentials
        """
        self.topic = topic
        self.schema = schema

        # Retrieve credentials
        self.kafka_config = self._get_kafka_config(aws_region, kafka_secret_name)
        self.schema_registry_config = self._get_schema_registry_config(
            aws_region, schema_registry_secret_name
        )

        # Initialise producer and schema registry client
        self.producer = Producer(self.kafka_config)
        self.schema_registry_client = SchemaRegistryClient(self.schema_registry_config)

        # Create Avro serialiser
        self.avro_serialiser = AvroSerializer(
            self.schema_registry_client,
            json.dumps(schema),
            lambda obj, ctx: obj  # Object is already a dict
        )

        logger.info(f"Kafka producer initialised for topic: {topic}")

    def _get_kafka_config(self, region: str, secret_name: str) -> Dict[str, str]:
        """Retrieve Kafka credentials from AWS Secrets Manager."""
        client = boto3.client('secretsmanager', region_name=region)
        response = client.get_secret_value(SecretId=secret_name)
        secret = json.loads(response['SecretString'])

        return {
            'bootstrap.servers': secret['bootstrap_servers'],
            'security.protocol': 'SASL_SSL',
            'sasl.mechanisms': 'PLAIN',
            'sasl.username': secret['api_key'],
            'sasl.password': secret['api_secret'],
            # Production settings
            'acks': 'all',  # Wait for all replicas
            'retries': 10,  # Retry up to 10 times
            'enable.idempotence': True,  # Exactly-once semantics
            'compression.type': 'snappy',  # Compress messages
            'linger.ms': 100,  # Batch messages for 100ms
            'batch.size': 16384  # Batch up to 16KB
        }

    def _get_schema_registry_config(self, region: str, secret_name: str) -> Dict[str, str]:
        """Retrieve Schema Registry credentials from AWS Secrets Manager."""
        client = boto3.client('secretsmanager', region_name=region)
        response = client.get_secret_value(SecretId=secret_name)
        secret = json.loads(response['SecretString'])

        return {
            'url': secret['url'],
            'basic.auth.user.info': f"{secret['api_key']}:{secret['api_secret']}"
        }

    def produce(
        self,
        event: Dict[str, Any],
        key: Optional[str] = None,
        callback: Optional[Callable] = None
    ) -> None:
        """
        Produce an event to Kafka.

        Args:
            event: Event data as dict (must match Avro schema)
            key: Partition key (optional)
            callback: Delivery callback (optional)
        """
        try:
            # Serialise event
            serialised_value = self.avro_serialiser(
                event,
                SerializationContext(self.topic, MessageField.VALUE)
            )

            # Produce to Kafka
            self.producer.produce(
                topic=self.topic,
                key=key.encode('utf-8') if key else None,
                value=serialised_value,
                callback=callback or self._default_delivery_callback
            )

            logger.debug(f"Event produced to {self.topic}: key={key}")

        except Exception as e:
            logger.error(f"Failed to produce event: {e}")
            raise

    def _default_delivery_callback(self, err, msg):
        """Default callback for delivery confirmation."""
        if err:
            logger.error(f"Message delivery failed: {err}")
        else:
            logger.info(
                f"Message delivered to {msg.topic()} "
                f"[partition {msg.partition()}] at offset {msg.offset()}"
            )

    def flush(self, timeout: float = 10.0) -> int:
        """
        Wait for all messages to be delivered.

        Args:
            timeout: Maximum time to wait (seconds)

        Returns:
            Number of messages still in queue
        """
        remaining = self.producer.flush(timeout)
        if remaining > 0:
            logger.warning(f"{remaining} messages still in queue after flush")
        return remaining

    def close(self) -> None:
        """Close the producer and clean up resources."""
        self.flush()
        logger.info("Kafka producer closed")
```

### Configuration Explanation

**Producer settings:**

```python
'acks': 'all',  # Wait for all replicas to acknowledge (durability)
'retries': 10,  # Retry failed sends up to 10 times
'enable.idempotence': True,  # Exactly-once delivery (no duplicates)
'compression.type': 'snappy',  # Compress messages (reduce bandwidth)
'linger.ms': 100,  # Wait 100ms to batch messages (throughput)
'batch.size': 16384  # Batch up to 16KB before sending
```

**Trade-offs:**

| Setting | Low Latency | High Throughput |
|---------|------------|-----------------|
| `acks` | 1 (leader only) | all (all replicas) |
| `linger.ms` | 0 (send immediately) | 100-1000 (batch) |
| `batch.size` | Small (1KB) | Large (64KB) |
| `compression.type` | none | snappy/lz4 |

**Recommendation:** Use settings above for production (balance latency and throughput).

## Produce Order Events

Create a script to produce sample order events.

### Order Events Producer

Create `produce_orders.py`:

```python
#!/usr/bin/env python3
"""
Produce sample order events to Kafka.
"""
import time
import logging
from kafka_producer import KafkaProducer


# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)


# Avro schema for order events
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


def generate_order_events(count: int = 10):
    """Generate sample order events."""
    for i in range(count):
        yield {
            'order_id': 1000 + i,
            'customer_id': f'cust_{1000 + i}',
            'order_status': ['CREATED', 'CONFIRMED', 'SHIPPED'][i % 3],
            'order_total': round(50.0 + (i * 10.5), 2),
            'created_at': int(time.time() * 1000)
        }


def main():
    logger.info("🚀 Starting order events producer...")

    # Initialise producer
    producer = KafkaProducer(
        topic='order-events',
        schema=ORDER_EVENTS_SCHEMA
    )

    # Produce events
    event_count = 0
    for event in generate_order_events(count=10):
        producer.produce(
            event=event,
            key=str(event['order_id'])  # Partition by order_id
        )
        event_count += 1

    # Wait for delivery
    remaining = producer.flush(timeout=10)

    if remaining == 0:
        logger.info(f"✅ Successfully produced {event_count} events")
    else:
        logger.error(f"❌ {remaining} events failed to deliver")

    producer.close()


if __name__ == '__main__':
    main()
```

### Run Producer

```bash
python produce_orders.py
```

**Expected output:**
```
2026-03-05 12:00:00 - __main__ - INFO - 🚀 Starting order events producer...
2026-03-05 12:00:00 - kafka_producer - INFO - Kafka producer initialised for topic: order-events
2026-03-05 12:00:01 - kafka_producer - INFO - Message delivered to order-events [0] at offset 10
2026-03-05 12:00:01 - kafka_producer - INFO - Message delivered to order-events [1] at offset 8
...
2026-03-05 12:00:02 - __main__ - INFO - ✅ Successfully produced 10 events
2026-03-05 12:00:02 - kafka_producer - INFO - Kafka producer closed
```

### Verify in Snowflake

```sql
SELECT
    RECORD_CONTENT:order_id::INT AS order_id,
    RECORD_CONTENT:customer_id::STRING AS customer_id,
    RECORD_CONTENT:order_status::STRING AS order_status,
    RECORD_CONTENT:order_total::FLOAT AS order_total,
    RECORD_METADATA:timestamp::TIMESTAMP AS kafka_timestamp
FROM STREAMING.PUBLIC.ORDER_EVENTS
WHERE RECORD_METADATA:timestamp >= DATEADD(minute, -5, CURRENT_TIMESTAMP())
ORDER BY kafka_timestamp DESC
LIMIT 10;
```

## Integrate with Prefect

Wrap the producer in a Prefect task for orchestration.

### Prefect Flow

Create `flows/produce_order_events_flow.py`:

```python
"""
Prefect flow to produce order events on a schedule.
"""
from prefect import flow, task
from kafka_producer import KafkaProducer
import logging
import time


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


@task(retries=3, retry_delay_seconds=5)
def produce_events(event_count: int = 10) -> int:
    """
    Produce order events to Kafka.

    Args:
        event_count: Number of events to produce

    Returns:
        Number of successfully produced events
    """
    logger.info(f"Producing {event_count} order events...")

    producer = KafkaProducer(
        topic='order-events',
        schema=ORDER_EVENTS_SCHEMA
    )

    produced = 0
    for i in range(event_count):
        event = {
            'order_id': int(time.time() * 1000) + i,  # Unique ID
            'customer_id': f'cust_{1000 + i}',
            'order_status': ['CREATED', 'CONFIRMED', 'SHIPPED'][i % 3],
            'order_total': round(50.0 + (i * 10.5), 2),
            'created_at': int(time.time() * 1000)
        }

        producer.produce(event=event, key=str(event['order_id']))
        produced += 1

    # Flush and close
    remaining = producer.flush(timeout=10)
    producer.close()

    if remaining > 0:
        raise Exception(f"{remaining} events failed to deliver")

    logger.info(f"✅ Successfully produced {produced} events")
    return produced


@flow(name="produce-order-events", log_prints=True)
def produce_order_events_flow(event_count: int = 10):
    """
    Flow to produce order events on a schedule.

    Args:
        event_count: Number of events to produce per run
    """
    print(f"🚀 Starting order events production flow...")

    # Produce events
    produced = produce_events(event_count=event_count)

    print(f"✅ Flow complete! Produced {produced} events")
    return produced


if __name__ == '__main__':
    # Run locally for testing
    produce_order_events_flow(event_count=5)
```

### Deploy to Prefect Cloud

```bash
# Build deployment
prefect deploy flows/produce_order_events_flow.py:produce_order_events_flow \
    --name "Order Events Producer" \
    --pool "default-agent-pool" \
    --cron "*/5 * * * *"  # Every 5 minutes

# Trigger manually
prefect deployment run produce-order-events/Order\ Events\ Producer
```

### Schedule Options

| Schedule | Cron Expression | Use Case |
|----------|----------------|----------|
| **Every 5 minutes** | `*/5 * * * *` | High-frequency events (testing) |
| **Every hour** | `0 * * * *` | Moderate volume |
| **Every day at 9am** | `0 9 * * *` | Daily batch events |
| **Weekdays only** | `0 9 * * 1-5` | Business hours events |

## Error Handling and Retries

### Handling Serialisation Errors

Invalid events that don't match the Avro schema will fail serialisation:

```python
try:
    producer.produce(event=event, key=key)
except Exception as e:
    logger.error(f"Serialisation failed for event: {event}")
    logger.error(f"Error: {e}")
    # Send to dead letter queue or skip
```

### Handling Delivery Failures

Delivery failures are retried automatically (up to `retries` config):

```python
def delivery_callback(err, msg):
    if err:
        # Permanent failure after retries
        logger.error(f"Permanent delivery failure: {err}")
        # Alert or write to DLQ
    else:
        # Success
        logger.info(f"Delivered: {msg.topic()} [{msg.partition()}] @ {msg.offset()}")
```

### Dead Letter Queue Pattern

For permanent failures, send events to a DLQ topic:

```python
def produce_with_dlq(producer, event, key, dlq_producer):
    """Produce event with DLQ fallback."""
    def callback(err, msg):
        if err:
            logger.error(f"Failed to deliver: {err}")
            # Send to DLQ
            dlq_producer.produce(
                event={'original_event': event, 'error': str(err)},
                key=key
            )

    producer.produce(event=event, key=key, callback=callback)
```

## Summary

You've built production Python producers:

- [x] **Reusable producer class** — with Schema Registry integration and error handling
- [x] **Avro schema registration** — type-safe event serialisation
- [x] **Partition keys** — events for the same order_id always in same partition
- [x] **Production settings** — exactly-once delivery, compression, batching
- [x] **Error handling** — retries, delivery callbacks, DLQ pattern
- [x] **Prefect integration** — scheduled flows to produce events
- [x] **Monitoring** — delivery confirmations and logging

Your producers are production-ready. Next, orchestrate and monitor with Prefect.

## What's Next

Monitor Kafka consumer lag, orchestrate event-driven workflows, and set up alerts with Prefect.

Continue to [Prefect Orchestration](7-prefect-orchestration.md) →
