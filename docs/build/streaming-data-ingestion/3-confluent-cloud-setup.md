# Confluent Cloud Setup

On this page, you will:

- [x] Create a Confluent Cloud account
- [x] Set up environment and Basic cluster
- [x] Configure Schema Registry
- [x] Generate API keys with least privilege
- [x] Store credentials in AWS Secrets Manager
- [x] Test connectivity with Python
- [x] Implement security best practices

## Overview

Confluent Cloud provides a fully managed Kafka service. In this guide, you'll create your account, provision a Basic tier cluster, and configure Schema Registry for Avro schemas.

You'll generate API keys with minimal permissions, store them securely in AWS Secrets Manager, and verify connectivity with a Python producer.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CONFLUENT CLOUD SETUP FLOW                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. Create Account                                                      │
│     ┌────────────────────────────────────────────────────┐             │
│     │ Sign up → Verify email → Add payment method        │             │
│     └────────────────────────────────────────────────────┘             │
│                          │                                              │
│  2. Create Environment   ▼                                              │
│     ┌────────────────────────────────────────────────────┐             │
│     │ Environment: production                            │             │
│     │ Region: eu-west-2 (London)                         │             │
│     └────────────────────────────────────────────────────┘             │
│                          │                                              │
│  3. Create Cluster       ▼                                              │
│     ┌────────────────────────────────────────────────────┐             │
│     │ Cluster: streaming-cluster                         │             │
│     │ Type: Basic (1 CKU)                                │             │
│     │ Cloud: AWS, Region: eu-west-2                      │             │
│     └────────────────────────────────────────────────────┘             │
│                          │                                              │
│  4. Enable Schema Reg    ▼                                              │
│     ┌────────────────────────────────────────────────────┐             │
│     │ Schema Registry: eu-west-2                         │             │
│     │ Essentials package (included in Basic)             │             │
│     └────────────────────────────────────────────────────┘             │
│                          │                                              │
│  5. Create API Keys      ▼                                              │
│     ┌────────────────────────────────────────────────────┐             │
│     │ Cluster API Key (producer/consumer)                │             │
│     │ Schema Registry API Key (schema access)            │             │
│     └────────────────────────────────────────────────────┘             │
│                          │                                              │
│  6. Store in Secrets     ▼                                              │
│     ┌────────────────────────────────────────────────────┐             │
│     │ AWS Secrets Manager:                               │             │
│     │ confluent/kafka-cluster                            │             │
│     │ confluent/schema-registry                          │             │
│     └────────────────────────────────────────────────────┘             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Create Confluent Cloud Account

### Sign Up

1. Navigate to [confluent.cloud](https://confluent.cloud)
2. Click **Start free**
3. Enter your email and create a password
4. Verify your email address

### Add Payment Method

Confluent Cloud requires a payment method even for the free trial.

1. Go to **Billing & payment**
2. Add a credit/debit card
3. Review pricing (Basic tier starts at ~$150/month after trial)

!!! tip "Free Trial Credits"
    New accounts receive **$400 in free credits** valid for 30 days. This is enough to run a Basic cluster for the trial period.

!!! warning "Set Up Billing Alerts"
    Navigate to **Billing → Notifications** and configure:
    - Alert at 50% of monthly budget ($75 if budgeting $150/month)
    - Alert at 80% of monthly budget ($120)
    - Email notifications to your team

## Create Environment

Environments organise clusters by purpose (dev, staging, production).

### Via Web UI

1. Navigate to **Environments** in the left sidebar
2. Click **+ Add cloud environment**
3. Configure:
    - **Name:** `production`
    - **Stream Governance Package:** Essentials (included with Basic)
4. Click **Continue**

### Via CLI (Optional)

Install Confluent CLI:

```sh
# macOS
brew install confluentinc/tap/cli

# Verify installation
confluent version
```

**Expected output:**
```
confluent - Confluent CLI

Version:     v3.45.0
Git Ref:     e3c9d32
Build Date:  2026-02-15T10:23:45Z
Go Version:  go1.21.5 (darwin/arm64)
```

Create environment:

```sh
# Login
confluent login

# Create environment
confluent environment create production \
    --governance-package essentials

# List environments
confluent environment list
```

**Expected output:**
```
      Id      |    Name     | Stream Governance Package
--------------+-------------+--------------------------
  env-abc123  | production  | ESSENTIALS
```

## Create Kafka Cluster

### Via Web UI

1. Navigate to your `production` environment
2. Click **+ Add cluster**
3. Select **Basic** cluster type
4. Configure cluster:
    - **Cloud provider:** AWS
    - **Region:** eu-west-2 (London) - choose region closest to your Snowflake
    - **Cluster name:** `streaming-cluster`
5. Review pricing (~$150/month)
6. Click **Launch cluster**

Wait for cluster provisioning to complete.

!!! info "Region Selection"
    Choose the same region as your Snowflake account to minimise latency and data transfer costs. If your Snowflake is in `eu-west-2`, deploy Confluent in `eu-west-2`.

### Via CLI

```sh
# Use the production environment
confluent environment use env-abc123

# Create Basic cluster
confluent kafka cluster create streaming-cluster \
    --cloud aws \
    --region eu-west-2 \
    --type basic

# Wait for cluster to be ready
confluent kafka cluster list
```

**Expected output:**
```
      Id      |       Name        |  Type  | Cloud |  Region   | Status
--------------+-------------------+--------+-------+-----------+---------
  lkc-xyz789  | streaming-cluster | BASIC  | aws   | eu-west-2 | UP
```

### Cluster Configuration

Once provisioned, review your cluster settings:

1. Navigate to **Cluster overview**
2. Note the **Bootstrap server** endpoint (e.g., `pkc-abc123.eu-west-2.aws.confluent.cloud:9092`)
3. Verify **Availability:** Single-zone (Basic tier)

## Configure Schema Registry

Schema Registry is automatically provisioned with the Essentials package.

### Verify Schema Registry

1. In your environment, click **Schema Registry** tab
2. Verify **Status:** Enabled
3. Note the **API endpoint** (e.g., `https://psrc-abc123.eu-west-2.aws.confluent.cloud`)

### Schema Registry Configuration

Schema Registry settings:

- **Compatibility mode:** `BACKWARD` (default) - new schemas can read old data
- **Schema validation:** Enabled
- **Schema ID strategy:** `TopicNameStrategy` (schema per topic)

You can change compatibility mode later:

```sh
# Set compatibility for a specific subject
confluent schema-registry compatibility set BACKWARD \
    --subject order-events-value
```

## Generate API Keys

Create separate API keys for cluster access and Schema Registry access.

### Cluster API Key (Kafka)

This key authenticates producers and consumers.

**Via Web UI:**

1. Navigate to your cluster → **API keys** tab
2. Click **+ Add key**
3. Select **Global access** (or scope to specific service account later)
4. Click **Generate API key**
5. **Download** the key immediately - you cannot retrieve it later
6. Store securely (we'll add to AWS Secrets Manager next)

**Via CLI:**

```sh
# Use your cluster
confluent kafka cluster use lkc-xyz789

# Create API key
confluent api-key create --resource lkc-xyz789 \
    --description "Producer/consumer key for streaming-cluster"
```

**Expected output:**
```
API Key    | abc123xyz789
API Secret | VeryLongSecretStringHere1234567890ABCDEF
```

!!! warning "Save Immediately"
    The API secret is only shown **once**. Copy it to a secure location immediately.

### Schema Registry API Key

This key authenticates schema operations (register, retrieve schemas).

**Via Web UI:**

1. Navigate to **Schema Registry** → **API credentials**
2. Click **+ Add key**
3. Download the key

**Via CLI:**

```sh
# Create Schema Registry API key
confluent api-key create --resource lsrc-abc123 \
    --description "Schema Registry access"
```

**Expected output:**
```
API Key    | SR123ABCXYZ
API Secret | AnotherVeryLongSecretString7890
```

## Store Credentials in AWS Secrets Manager

Store Confluent credentials securely in AWS Secrets Manager for use in Prefect tasks and Snowflake connectors.

### Kafka Cluster Credentials

Create secret for Kafka cluster API key:

```sh
# Set AWS profile
export AWS_PROFILE=data-engineer

# Create secret
aws secretsmanager create-secret \
    --name confluent/kafka-cluster \
    --description "Confluent Cloud Kafka cluster credentials" \
    --secret-string '{
        "bootstrap_servers": "pkc-abc123.eu-west-2.aws.confluent.cloud:9092",
        "api_key": "abc123xyz789",
        "api_secret": "VeryLongSecretStringHere1234567890ABCDEF",
        "cluster_id": "lkc-xyz789"
    }' \
    --region eu-west-2
```

**Expected output:**
```json
{
    "ARN": "arn:aws:secretsmanager:eu-west-2:123456789012:secret:confluent/kafka-cluster-abc123",
    "Name": "confluent/kafka-cluster",
    "VersionId": "e1f2a3b4-5c6d-7e8f-9a0b-1c2d3e4f5a6b"
}
```

### Schema Registry Credentials

Create secret for Schema Registry API key:

```sh
aws secretsmanager create-secret \
    --name confluent/schema-registry \
    --description "Confluent Schema Registry credentials" \
    --secret-string '{
        "url": "https://psrc-abc123.eu-west-2.aws.confluent.cloud",
        "api_key": "SR123ABCXYZ",
        "api_secret": "AnotherVeryLongSecretString7890"
    }' \
    --region eu-west-2
```

### Retrieve Credentials (Test)

Verify you can retrieve secrets:

```sh
# Get Kafka credentials
aws secretsmanager get-secret-value \
    --secret-id confluent/kafka-cluster \
    --region eu-west-2 \
    --query SecretString \
    --output text | jq .
```

**Expected output:**
```json
{
  "bootstrap_servers": "pkc-abc123.eu-west-2.aws.confluent.cloud:9092",
  "api_key": "abc123xyz789",
  "api_secret": "VeryLongSecretStringHere1234567890ABCDEF",
  "cluster_id": "lkc-xyz789"
}
```

## Test Connectivity with Python

Verify your Kafka cluster is accessible using a Python producer.

### Install Dependencies

```sh
# Create virtual environment
uv init
uv add confluent-kafka boto3
```

### Python Producer Test

Create `test_producer.py`:

```python
#!/usr/bin/env python3
"""
Test Confluent Cloud connectivity by producing a message.
"""
import json
import boto3
from confluent_kafka import Producer


def get_kafka_config():
    """Retrieve Kafka credentials from AWS Secrets Manager."""
    session = boto3.session.Session()
    client = session.client(
        service_name='secretsmanager',
        region_name='eu-west-2'
    )

    response = client.get_secret_value(SecretId='confluent/kafka-cluster')
    secret = json.loads(response['SecretString'])

    return {
        'bootstrap.servers': secret['bootstrap_servers'],
        'security.protocol': 'SASL_SSL',
        'sasl.mechanisms': 'PLAIN',
        'sasl.username': secret['api_key'],
        'sasl.password': secret['api_secret']
    }


def delivery_report(err, msg):
    """Callback for message delivery confirmation."""
    if err is not None:
        print(f'❌ Message delivery failed: {err}')
    else:
        print(f'✅ Message delivered to {msg.topic()} [{msg.partition()}] at offset {msg.offset()}')


def main():
    print("🔗 Connecting to Confluent Cloud...")

    # Get config from Secrets Manager
    config = get_kafka_config()

    # Create producer
    producer = Producer(config)

    # Produce test message
    topic = 'test-connectivity'
    message = {
        'test_id': 1,
        'message': 'Hello from Python!',
        'timestamp': '2026-03-05T12:00:00Z'
    }

    print(f"📤 Producing message to topic: {topic}")

    producer.produce(
        topic,
        key=str(message['test_id']).encode('utf-8'),
        value=json.dumps(message).encode('utf-8'),
        callback=delivery_report
    )

    # Wait for message delivery
    producer.flush(timeout=10)

    print("✅ Test complete!")


if __name__ == '__main__':
    main()
```

### Run Test

```sh
python test_producer.py
```

**Expected output:**
```
🔗 Connecting to Confluent Cloud...
📤 Producing message to topic: test-connectivity
✅ Message delivered to test-connectivity [0] at offset 0
✅ Test complete!
```

!!! success "Connectivity Verified"
    If you see the message delivered successfully, your Confluent Cloud cluster is ready for production use!

### Troubleshooting

**Error: `Failed to connect to broker`**

Check:
- Bootstrap servers URL is correct
- API key and secret are valid
- Network allows HTTPS traffic to Confluent Cloud (port 9092)

**Error: `Broker: Topic authorization failed`**

Your API key doesn't have permission to create topics. Either:
- Enable auto-create topics in cluster settings
- Pre-create the topic via UI or CLI

**Error: `Authentication failed`**

Your API credentials are incorrect. Regenerate API key and update AWS Secrets Manager.

## Security Best Practices

### Principle of Least Privilege

Don't use global API keys for production. Create **service accounts** with scoped permissions.

**Example: Service account for producers only**

```sh
# Create service account
confluent iam service-account create producer-service-account \
    --description "Service account for event producers"

# Grant write access to specific topic
confluent kafka acl create \
    --allow \
    --service-account sa-123456 \
    --operation WRITE \
    --topic order-events

# Generate API key for this service account
confluent api-key create \
    --service-account sa-123456 \
    --resource lkc-xyz789
```

Now this API key can **only write** to the `order-events` topic.

### Rotate API Keys Regularly

Set a reminder to rotate API keys every 90 days:

1. Generate new API key
2. Update AWS Secrets Manager
3. Deploy updated credentials to Prefect
4. Delete old API key

### Audit API Key Usage

Monitor which API keys are being used:

```sh
# List all API keys
confluent api-key list
```

Delete unused keys:

```sh
confluent api-key delete <key-id>
```

### Network Security

Basic tier clusters are publicly accessible. For production:

- Use **VPC peering** (Dedicated tier) to restrict access to your VPC
- Use **PrivateLink** (Dedicated tier) for private connectivity
- Allowlist IPs in Confluent Cloud network settings

Basic tier workaround:
- Use AWS Secrets Manager to avoid exposing credentials
- Restrict AWS IAM permissions to Secrets Manager

### Enable Audit Logging (Dedicated Tier Only)

Audit logs track all cluster operations (topic creation, ACL changes, etc.). Available on Dedicated tier only.

## Summary

You've set up Confluent Cloud:

- [x] **Account created** — with billing alerts configured
- [x] **Environment created** — `production` with Essentials package
- [x] **Cluster deployed** — Basic tier in eu-west-2
- [x] **Schema Registry enabled** — for Avro schema management
- [x] **API keys generated** — cluster and Schema Registry credentials
- [x] **Credentials stored** — in AWS Secrets Manager for secure access
- [x] **Connectivity tested** — Python producer verified
- [x] **Security practices** — least privilege, rotation, audit

Your Kafka cluster is ready to receive events. Next, prepare Snowflake to consume those events.

## What's Next

Configure Snowflake infrastructure for streaming ingestion (STREAMING database, service account).

Continue to [Snowflake Infrastructure](4-snowflake-infrastructure.md) →
