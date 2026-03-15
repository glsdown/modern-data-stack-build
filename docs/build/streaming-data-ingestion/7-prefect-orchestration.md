# Prefect Orchestration

On this page, you will:

- [x] Monitor Kafka consumer lag with Prefect tasks
- [x] Create flows to produce events on schedules
- [x] Build event-driven workflows triggered by Kafka
- [x] Set up alerting on connector failures
- [x] Track throughput and latency metrics
- [x] Integrate streaming with batch pipelines

## Overview

Prefect orchestrates your streaming infrastructure by:

1. **Monitoring** — Track consumer lag, connector health, throughput
2. **Producing** — Schedule event generation flows
3. **Event-driven workflows** — Trigger dbt runs when events arrive
4. **Alerting** — Notify on failures or anomalies
5. **Integration** — Combine streaming and batch pipelines

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PREFECT STREAMING ORCHESTRATION                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────────────────────────────────────────────────┐          │
│  │ Prefect Flow: Monitor Streaming Infrastructure           │          │
│  ├──────────────────────────────────────────────────────────┤          │
│  │                                                          │          │
│  │  Task 1: Check Kafka consumer lag                       │          │
│  │  ├─▶ Query Confluent Cloud API                          │          │
│  │  └─▶ Alert if lag > 10,000 events                       │          │
│  │                                                          │          │
│  │  Task 2: Check connector status                         │          │
│  │  ├─▶ Query Kafka Connect API                            │          │
│  │  └─▶ Alert if status != RUNNING                         │          │
│  │                                                          │          │
│  │  Task 3: Track throughput                               │          │
│  │  ├─▶ Count events in last 5 minutes (Snowflake)         │          │
│  │  └─▶ Alert if throughput drops > 50%                    │          │
│  │                                                          │          │
│  │  Schedule: Every 5 minutes                              │          │
│  └──────────────────────────────────────────────────────────┘          │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────┐          │
│  │ Prefect Flow: Event-Driven dbt Transformation            │          │
│  ├──────────────────────────────────────────────────────────┤          │
│  │                                                          │          │
│  │  Task 1: Check for new events (Snowflake)               │          │
│  │  ├─▶ Query STREAMING.ORDER_EVENTS                       │          │
│  │  └─▶ Count events in last 5 minutes                     │          │
│  │                                                          │          │
│  │  Task 2: Run dbt incremental models (if events > 0)     │          │
│  │  ├─▶ dbt run --select tag:streaming                     │          │
│  │  └─▶ Transform events into ANALYTICS.fct_orders         │          │
│  │                                                          │          │
│  │  Schedule: Every 5 minutes                              │          │
│  └──────────────────────────────────────────────────────────┘          │
│                                                                         │
│  Pattern: Monitor → Alert → Transform → Monitor                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Monitor Consumer Lag

Consumer lag measures how far behind a consumer is from the latest message in a topic. High lag indicates the connector can't keep up with event production.

### Consumer Lag Monitoring Task

Create `tasks/monitor_kafka_lag.py`:

```python
"""
Monitor Kafka consumer lag via Confluent Cloud API.
"""
import requests
import json
import logging
import boto3
from prefect import task


logger = logging.getLogger(__name__)


@task(retries=2, retry_delay_seconds=10)
def get_consumer_lag(
    consumer_group: str,
    alert_threshold: int = 10000
) -> dict:
    """
    Check Kafka consumer lag via Confluent Cloud API.

    Args:
        consumer_group: Consumer group name (e.g., 'snowflake-sink')
        alert_threshold: Alert if lag exceeds this value

    Returns:
        Dict with lag metrics per partition
    """
    logger.info(f"Checking consumer lag for group: {consumer_group}")

    # Get credentials from Secrets Manager
    secrets_client = boto3.client('secretsmanager', region_name='eu-west-2')
    response = secrets_client.get_secret_value(SecretId='confluent/kafka-cluster')
    creds = json.loads(response['SecretString'])

    # Confluent Cloud API endpoint
    # Note: This uses Kafka's consumer group describe API
    # For production, use Confluent Cloud Metrics API (requires separate API key)

    # Simplified example - query Snowflake for lag proxy
    lag_metrics = _query_snowflake_lag()

    # Check threshold
    total_lag = sum(lag_metrics.values())
    if total_lag > alert_threshold:
        logger.warning(f"⚠️  High consumer lag detected: {total_lag} messages")
        return {
            'status': 'WARNING',
            'total_lag': total_lag,
            'lag_per_partition': lag_metrics,
            'alert': True
        }

    logger.info(f"✅ Consumer lag OK: {total_lag} messages")
    return {
        'status': 'OK',
        'total_lag': total_lag,
        'lag_per_partition': lag_metrics,
        'alert': False
    }


def _query_snowflake_lag() -> dict:
    """
    Query Snowflake to estimate lag based on event timestamps.

    Returns:
        Dict of partition -> lag estimate
    """
    import snowflake.connector
    from cryptography.hazmat.primitives import serialization
    from cryptography.hazmat.backends import default_backend

    # Get Snowflake credentials
    secrets_client = boto3.client('secretsmanager', region_name='eu-west-2')
    response = secrets_client.get_secret_value(SecretId='snowflake/svc-kafka-connector')
    creds = json.loads(response['SecretString'])

    # Convert private key
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

    # Connect to Snowflake
    conn = snowflake.connector.connect(
        account=creds['account'],
        user=creds['user'],
        private_key=private_key_bytes,
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema=creds['schema'],
        role=creds['role']
    )

    # Query lag (time since last event per partition)
    query = """
    SELECT
        RECORD_METADATA:partition::INT AS partition,
        DATEDIFF(second, MAX(RECORD_METADATA:timestamp::TIMESTAMP), CURRENT_TIMESTAMP()) AS lag_seconds,
        COUNT(*) AS event_count
    FROM ORDER_EVENTS
    WHERE RECORD_METADATA:timestamp >= DATEADD(minute, -15, CURRENT_TIMESTAMP())
    GROUP BY partition
    ORDER BY partition
    """

    cursor = conn.cursor()
    cursor.execute(query)
    results = cursor.fetchall()

    lag_metrics = {}
    for row in results:
        partition = row[0]
        lag_seconds = row[1]
        # Estimate lag in messages (rough approximation)
        lag_metrics[partition] = lag_seconds * 10  # Assume 10 msgs/sec

    cursor.close()
    conn.close()

    return lag_metrics
```

### Consumer Lag Flow

Create `flows/monitor_streaming_flow.py`:

```python
"""
Prefect flow to monitor streaming infrastructure.
"""
from prefect import flow, task
from tasks.monitor_kafka_lag import get_consumer_lag
import logging


logger = logging.getLogger(__name__)


@task
def check_connector_status() -> dict:
    """
    Check Snowflake Kafka Connector status via Confluent Cloud API.

    Returns:
        Dict with connector status
    """
    # Simplified example - in production, query Confluent Cloud API
    # https://docs.confluent.io/cloud/current/api.html

    logger.info("Checking connector status...")

    # Mock status check
    status = "RUNNING"

    if status != "RUNNING":
        logger.error(f"❌ Connector is {status}")
        return {'status': status, 'alert': True}

    logger.info(f"✅ Connector is {status}")
    return {'status': status, 'alert': False}


@task
def check_throughput() -> dict:
    """
    Check event throughput in last 5 minutes.

    Returns:
        Dict with throughput metrics
    """
    import snowflake.connector
    import boto3
    import json
    from cryptography.hazmat.primitives import serialization
    from cryptography.hazmat.backends import default_backend

    logger.info("Checking event throughput...")

    # Get credentials
    secrets_client = boto3.client('secretsmanager', region_name='eu-west-2')
    response = secrets_client.get_secret_value(SecretId='snowflake/svc-kafka-connector')
    creds = json.loads(response['SecretString'])

    # Convert private key
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

    # Connect
    conn = snowflake.connector.connect(
        account=creds['account'],
        user=creds['user'],
        private_key=private_key_bytes,
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema=creds['schema'],
        role=creds['role']
    )

    # Query throughput
    query = """
    SELECT COUNT(*) AS event_count
    FROM ORDER_EVENTS
    WHERE RECORD_METADATA:timestamp >= DATEADD(minute, -5, CURRENT_TIMESTAMP())
    """

    cursor = conn.cursor()
    cursor.execute(query)
    result = cursor.fetchone()
    event_count = result[0]

    cursor.close()
    conn.close()

    events_per_min = event_count / 5
    logger.info(f"✅ Throughput: {events_per_min:.1f} events/min")

    return {
        'event_count': event_count,
        'events_per_min': events_per_min,
        'alert': events_per_min < 1  # Alert if < 1 event/min
    }


@flow(name="monitor-streaming", log_prints=True)
def monitor_streaming_flow():
    """
    Monitor streaming infrastructure health.

    Checks:
    - Consumer lag
    - Connector status
    - Event throughput
    """
    print("🔍 Monitoring streaming infrastructure...")

    # Check lag
    lag_metrics = get_consumer_lag(
        consumer_group='snowflake-sink',
        alert_threshold=10000
    )

    # Check connector
    connector_status = check_connector_status()

    # Check throughput
    throughput_metrics = check_throughput()

    # Aggregate alerts
    alerts = []
    if lag_metrics['alert']:
        alerts.append(f"High consumer lag: {lag_metrics['total_lag']} messages")
    if connector_status['alert']:
        alerts.append(f"Connector not running: {connector_status['status']}")
    if throughput_metrics['alert']:
        alerts.append(f"Low throughput: {throughput_metrics['events_per_min']:.1f} events/min")

    if alerts:
        print("⚠️  ALERTS:")
        for alert in alerts:
            print(f"  - {alert}")
        # Send to Slack/PagerDuty (covered in alerting section)
    else:
        print("✅ All checks passed!")

    return {
        'lag': lag_metrics,
        'connector': connector_status,
        'throughput': throughput_metrics,
        'alerts': alerts
    }


if __name__ == '__main__':
    monitor_streaming_flow()
```

### Deploy Monitoring Flow

```sh
# Deploy with 5-minute schedule
prefect deploy flows/monitor_streaming_flow.py:monitor_streaming_flow \
    --name "Streaming Health Monitor" \
    --pool "default-agent-pool" \
    --cron "*/5 * * * *"
```

## Event-Driven Workflows

Trigger dbt transformations when new events arrive in Snowflake.

### Event-Driven dbt Flow

Create `flows/streaming_dbt_flow.py`:

```python
"""
Event-driven dbt transformation for streaming data.
"""
from prefect import flow, task
import logging
import subprocess


logger = logging.getLogger(__name__)


@task
def count_new_events(since_minutes: int = 5) -> int:
    """
    Count new events in STREAMING.ORDER_EVENTS since N minutes ago.

    Args:
        since_minutes: Look back period in minutes

    Returns:
        Number of new events
    """
    import snowflake.connector
    import boto3
    import json
    from cryptography.hazmat.primitives import serialization
    from cryptography.hazmat.backends import default_backend

    logger.info(f"Checking for new events in last {since_minutes} minutes...")

    # Get credentials
    secrets_client = boto3.client('secretsmanager', region_name='eu-west-2')
    response = secrets_client.get_secret_value(SecretId='snowflake/svc-kafka-connector')
    creds = json.loads(response['SecretString'])

    # Convert private key
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

    # Connect
    conn = snowflake.connector.connect(
        account=creds['account'],
        user=creds['user'],
        private_key=private_key_bytes,
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema=creds['schema'],
        role=creds['role']
    )

    # Count new events
    query = f"""
    SELECT COUNT(*) AS event_count
    FROM ORDER_EVENTS
    WHERE RECORD_METADATA:timestamp >= DATEADD(minute, -{since_minutes}, CURRENT_TIMESTAMP())
    """

    cursor = conn.cursor()
    cursor.execute(query)
    result = cursor.fetchone()
    event_count = result[0]

    cursor.close()
    conn.close()

    logger.info(f"Found {event_count} new events")
    return event_count


@task
def run_dbt_streaming_models() -> dict:
    """
    Run dbt models tagged with 'streaming'.

    Returns:
        Dict with run results
    """
    logger.info("Running dbt streaming models...")

    # Run dbt with streaming tag
    result = subprocess.run(
        ['dbt', 'run', '--select', 'tag:streaming'],
        cwd='/path/to/dbt/project',
        capture_output=True,
        text=True
    )

    if result.returncode != 0:
        logger.error(f"dbt run failed: {result.stderr}")
        raise Exception(f"dbt run failed: {result.stderr}")

    logger.info("✅ dbt streaming models completed")
    return {
        'status': 'success',
        'stdout': result.stdout
    }


@flow(name="streaming-dbt-transform", log_prints=True)
def streaming_dbt_flow(
    since_minutes: int = 5,
    min_events_threshold: int = 1
):
    """
    Event-driven dbt transformation flow.

    Runs dbt incremental models only if new events exist.

    Args:
        since_minutes: Look back period
        min_events_threshold: Minimum events to trigger dbt run
    """
    print(f"🔍 Checking for new events in last {since_minutes} minutes...")

    # Count new events
    event_count = count_new_events(since_minutes=since_minutes)

    # Run dbt only if events exist
    if event_count >= min_events_threshold:
        print(f"✅ Found {event_count} events, running dbt transformations...")
        dbt_result = run_dbt_streaming_models()
        print("✅ Transformation complete!")
        return {'events': event_count, 'dbt_status': 'ran'}
    else:
        print(f"⏭️  Only {event_count} events, skipping dbt run")
        return {'events': event_count, 'dbt_status': 'skipped'}


if __name__ == '__main__':
    streaming_dbt_flow()
```

### Deploy Event-Driven Flow

```sh
# Deploy with 5-minute schedule
prefect deploy flows/streaming_dbt_flow.py:streaming_dbt_flow \
    --name "Streaming dbt Transformation" \
    --pool "default-agent-pool" \
    --cron "*/5 * * * *"
```

## Alerting on Failures

Send alerts to Slack when streaming infrastructure fails.

### Slack Alerting Task

Create `tasks/send_slack_alert.py`:

```python
"""
Send alerts to Slack.
"""
import requests
import logging
from prefect import task


logger = logging.getLogger(__name__)


@task
def send_slack_alert(message: str, channel: str = "#data-alerts") -> None:
    """
    Send alert to Slack.

    Args:
        message: Alert message
        channel: Slack channel
    """
    # Get Slack webhook URL from environment or Secrets Manager
    import os
    webhook_url = os.getenv('SLACK_WEBHOOK_URL')

    if not webhook_url:
        logger.warning("SLACK_WEBHOOK_URL not configured, skipping alert")
        return

    payload = {
        'channel': channel,
        'username': 'Prefect Alert Bot',
        'icon_emoji': ':rotating_light:',
        'text': f"⚠️  *Streaming Alert*\n{message}"
    }

    response = requests.post(webhook_url, json=payload)

    if response.status_code == 200:
        logger.info(f"✅ Alert sent to {channel}")
    else:
        logger.error(f"❌ Failed to send alert: {response.text}")
```

### Update Monitoring Flow with Alerts

Update `monitor_streaming_flow`:

```python
from tasks.send_slack_alert import send_slack_alert

@flow(name="monitor-streaming", log_prints=True)
def monitor_streaming_flow():
    """Monitor streaming infrastructure and alert on failures."""
    print("🔍 Monitoring streaming infrastructure...")

    # ... (previous monitoring tasks)

    # Send alerts if needed
    if alerts:
        alert_message = "\n".join([f"• {alert}" for alert in alerts])
        send_slack_alert(alert_message, channel="#data-alerts")

    return {
        'lag': lag_metrics,
        'connector': connector_status,
        'throughput': throughput_metrics,
        'alerts': alerts
    }
```

## Track Throughput Metrics

Create a dashboard to visualise streaming metrics over time.

### Metrics Collection Task

Create `tasks/collect_streaming_metrics.py`:

```python
"""
Collect streaming metrics and store in Snowflake for dashboards.
"""
from prefect import task
import logging
import time


logger = logging.getLogger(__name__)


@task
def collect_metrics(
    lag_metrics: dict,
    throughput_metrics: dict
) -> None:
    """
    Store streaming metrics in Snowflake for visualisation.

    Args:
        lag_metrics: Consumer lag metrics
        throughput_metrics: Event throughput metrics
    """
    import snowflake.connector
    import boto3
    import json
    from cryptography.hazmat.primitives import serialization
    from cryptography.hazmat.backends import default_backend

    logger.info("Storing streaming metrics...")

    # Get credentials
    secrets_client = boto3.client('secretsmanager', region_name='eu-west-2')
    response = secrets_client.get_secret_value(SecretId='snowflake/svc-kafka-connector')
    creds = json.loads(response['SecretString'])

    # Convert private key
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

    # Connect
    conn = snowflake.connector.connect(
        account=creds['account'],
        user=creds['user'],
        private_key=private_key_bytes,
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema=creds['schema'],
        role=creds['role']
    )

    # Create metrics table if not exists
    cursor = conn.cursor()
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS STREAMING_METRICS (
        timestamp TIMESTAMP_NTZ,
        metric_type STRING,
        metric_value FLOAT,
        metadata VARIANT
    )
    """)

    # Insert metrics
    cursor.execute("""
    INSERT INTO STREAMING_METRICS
    SELECT
        CURRENT_TIMESTAMP() AS timestamp,
        'consumer_lag' AS metric_type,
        %s AS metric_value,
        PARSE_JSON(%s) AS metadata
    """, (lag_metrics['total_lag'], json.dumps(lag_metrics)))

    cursor.execute("""
    INSERT INTO STREAMING_METRICS
    SELECT
        CURRENT_TIMESTAMP() AS timestamp,
        'throughput' AS metric_type,
        %s AS metric_value,
        PARSE_JSON(%s) AS metadata
    """, (throughput_metrics['events_per_min'], json.dumps(throughput_metrics)))

    cursor.close()
    conn.close()

    logger.info("✅ Metrics stored successfully")
```

## Summary

You've orchestrated streaming infrastructure with Prefect:

- [x] **Consumer lag monitoring** — query Kafka API or Snowflake for lag metrics
- [x] **Connector health checks** — verify connector status via Confluent API
- [x] **Throughput tracking** — count events per minute, alert on drops
- [x] **Event-driven workflows** — trigger dbt runs only when new events arrive
- [x] **Slack alerting** — send notifications on failures or anomalies
- [x] **Metrics collection** — store metrics in Snowflake for dashboards
- [x] **Scheduled flows** — run monitoring every 5 minutes automatically

Your streaming pipeline is monitored and orchestrated. Next, verify end-to-end latency and review costs.

## What's Next

Verify end-to-end latency, review costs, and learn when to use streaming vs batch.

Continue to [Finishing Up](8-finishing-up.md) →
