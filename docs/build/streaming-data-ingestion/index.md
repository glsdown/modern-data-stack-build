# Streaming Data Ingestion

On this page, you will:

- [x] Understand when to use streaming vs batch ingestion
- [x] Learn Kafka/Confluent fundamentals
- [x] Survey real-time use cases (clickstreams, IoT, CDC)
- [x] Review deployment options (Confluent Cloud vs self-hosted)
- [x] Plan your streaming infrastructure

## Overview

You've built batch data pipelines that load data on schedules (hourly, daily). But some use cases require **real-time** data - within seconds of events occurring. This is where **streaming data ingestion** excels.

Streaming ingestion processes data continuously as events happen, rather than waiting for the next scheduled batch run.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    STREAMING DATA INGESTION                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Event Sources         Kafka/Confluent       Snowflake                 │
│  ──────────────        ───────────────       ─────────                 │
│                                                                         │
│  ┌──────────────┐     ┌──────────────┐      ┌──────────────┐          │
│  │ Web clicks   │────▶│              │      │              │          │
│  │ Mobile app   │────▶│  Kafka       │─────▶│  Snowpipe    │          │
│  │ IoT sensors  │────▶│  Topics      │      │  Streaming   │          │
│  │ DB changes   │────▶│              │      │              │          │
│  │ (CDC)        │     │              │      │  STREAMING   │          │
│  └──────────────┘     └──────────────┘      │  database    │          │
│                              │               └──────────────┘          │
│                              │                      │                  │
│                              ▼                      ▼                  │
│                       ┌──────────────┐      ┌──────────────┐          │
│                       │ Schema       │      │ dbt          │          │
│                       │ Registry     │      │ incremental  │          │
│                       │ (Avro/Proto) │      │ models       │          │
│                       └──────────────┘      └──────────────┘          │
│                                                                         │
│  Latency: Seconds (not hours)                                          │
│  Use case: Real-time dashboards, fraud detection, operational analytics│
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Streaming vs Batch Ingestion

### When to Use Streaming

✅ **Use streaming when:**

- **Latency matters** — need data within seconds/minutes (not hours)
- **High event volume** — thousands of events per second
- **Real-time decision-making** — fraud detection, dynamic pricing, alerting
- **Event-driven workflows** — trigger actions based on specific events
- **Audit trails** — capture every state change in a database

**Examples:**
- Clickstream analytics (user behaviour tracking)
- IoT sensor data (temperature, location, machine telemetry)
- Fraud detection (flag suspicious transactions immediately)
- Change Data Capture (replicate database changes in real-time)
- Real-time dashboards (live KPIs updating every second)

### When to Use Batch

✅ **Use batch when:**

- **Hourly/daily is sufficient** — don't need sub-minute latency
- **Lower event volume** — hundreds/thousands of events per day
- **Source system doesn't support streaming** — APIs with rate limits, databases without CDC
- **Simpler infrastructure** — avoid Kafka cluster management

**Examples:**
- Daily sales reports
- Weekly reference data updates (currencies, product catalogues)
- SaaS platform data (HubSpot, Salesforce) loaded nightly
- Historical data backfills

### Comparison

| Aspect | Batch (dlt/Airbyte) | Streaming (Kafka) |
|--------|---------------------|-------------------|
| **Latency** | Minutes to hours | Seconds |
| **Volume** | Low to medium | High |
| **Infrastructure** | Simple | Complex (Kafka cluster) |
| **Cost** | Lower | Higher |
| **Use cases** | Reports, analytics | Real-time dashboards, alerts |
| **Setup complexity** | Low | Medium to high |
| **When data arrives** | On schedule | Continuously |

**You need both:** Use streaming for real-time operational data, batch for everything else.

## Kafka and Confluent Fundamentals

### What is Kafka?

**Apache Kafka** is a distributed event streaming platform. It acts as a durable, high-throughput message queue that:
- Receives events from **producers** (applications generating data)
- Stores events in **topics** (categories of events)
- Delivers events to **consumers** (applications processing data)

### Core Concepts

**Topics** — Categories of events (e.g., `web-clicks`, `orders`, `sensor-readings`)

**Partitions** — Topics are split into partitions for parallel processing and scalability

**Producers** — Applications that write events to Kafka topics

**Consumers** — Applications that read events from Kafka topics

**Consumer Groups** — Multiple consumers that share the work of processing a topic

**Schema Registry** — Centralized repository for event schemas (Avro, Protobuf, JSON Schema)

### What is Confluent?

**Confluent** is the company founded by Kafka's creators. They offer:
- **Confluent Cloud** — Fully managed Kafka as a service
- **Confluent Platform** — On-premises Kafka distribution with additional features
- **Kafka Connect** — Pre-built connectors for databases, cloud storage, and data warehouses

### Why Confluent Cloud?

Kafka is operationally complex to run (clusters, brokers, Zookeeper, monitoring). Confluent Cloud abstracts this away:

✅ **Fully managed** — No cluster management, automatic upgrades
✅ **Elastic scaling** — Scale up/down automatically based on load
✅ **Global availability** — Multi-region replication built-in
✅ **Security** — Encryption, authentication, authorization out of the box
✅ **Schema Registry** — Managed schema versioning and validation

**Trade-off:** Higher cost (~$150-200/month minimum) vs self-hosted (~$100/month for Redpanda on ECS).

## Real-Time Use Cases

### 1. Clickstream Analytics

**Scenario:** Track user behaviour on your website in real-time.

**Flow:**
```
Browser → JavaScript tracker → Kafka topic → Snowflake → Real-time dashboard
```

**Business value:**
- See which pages users visit right now
- Identify drop-off points in checkout flow
- Personalise content based on recent behaviour

**Latency requirement:** Seconds

### 2. IoT Sensor Data

**Scenario:** Monitor temperature sensors in warehouses.

**Flow:**
```
IoT devices → MQTT broker → Kafka topic → Snowflake → Alert if temp > 25°C
```

**Business value:**
- Prevent spoilage by detecting temperature spikes immediately
- Historical trend analysis for optimisation
- Predictive maintenance based on sensor patterns

**Latency requirement:** Seconds to minutes

### 3. Change Data Capture (CDC)

**Scenario:** Replicate your PostgreSQL production database to Snowflake for analytics without impacting production.

**Flow:**
```
PostgreSQL WAL → Debezium → Kafka topic → Snowflake → Analytics queries
```

**Business value:**
- Real-time analytics without slowing production database
- Audit trail of all database changes
- Data replication for disaster recovery

**Latency requirement:** Seconds

### 4. Fraud Detection

**Scenario:** Detect suspicious payment transactions immediately.

**Flow:**
```
Payment gateway → Kafka topic → Fraud ML model → Alert/block transaction
```

**Business value:**
- Stop fraudulent transactions before they complete
- Reduce chargebacks and losses
- Real-time risk scoring

**Latency requirement:** Sub-second

### 5. Real-Time Dashboards

**Scenario:** Live sales dashboard updating every second.

**Flow:**
```
E-commerce checkout → Kafka topic → Snowflake → Lightdash dashboard (auto-refresh)
```

**Business value:**
- Monitor Black Friday sales in real-time
- React quickly to marketing campaign performance
- Executive dashboards with live KPIs

**Latency requirement:** Seconds

## Deployment Options

### Option 1: Confluent Cloud (Recommended)

**What you get:** Fully managed Kafka + Schema Registry + Kafka Connect

**Pricing:**
- **Basic cluster:** ~$150/month (1 CKU - Confluent Kafka Unit)
- **Standard cluster:** ~$300/month (improved SLA, multi-AZ)
- **Dedicated cluster:** $500+/month (isolated resources, private networking)
- **Additional costs:** Data transfer ($0.08/GB), storage ($0.08/GB/month)

**Monthly estimate:** ~$150-200 for small workload (1 GB/day ingress, 30-day retention)

**Pros:**
- Zero infrastructure management
- Automatic scaling
- 99.95% SLA (Standard tier)
- Global replication built-in
- Enterprise security (SOC 2, GDPR)

**Cons:**
- Higher cost than self-hosted
- Less control over infrastructure

**Best for:** Teams that want to focus on data, not Kafka operations

### Option 2: AWS MSK (Managed Streaming for Kafka)

**What you get:** Managed Kafka cluster on AWS

**Pricing:**
- **Serverless:** Pay per GB ingested (~$0.08/GB) + storage
- **Provisioned:** 3-broker cluster ~$250/month (kafka.m5.large)

**Monthly estimate:** ~$250-300 for provisioned cluster

**Pros:**
- Integrated with AWS (VPC, IAM, CloudWatch)
- Lower cost than Confluent Cloud for large workloads
- More control than fully managed

**Cons:**
- You manage Kafka configuration, monitoring, upgrades
- No Schema Registry (need Confluent Schema Registry or AWS Glue)
- More operational burden

**Best for:** AWS-centric teams comfortable with Kafka operations

### Option 3: Redpanda Self-Hosted (Budget)

**What you get:** Kafka-compatible platform (simpler, faster)

**Infrastructure:**
- 3-node Redpanda cluster on ECS Fargate
- No Zookeeper required (simpler than Kafka)
- Built-in Schema Registry

**Monthly cost:** ~$100 (3 × ECS tasks, ALB, storage)

**Pros:**
- Lowest cost option
- Kafka-compatible (drop-in replacement)
- Simpler architecture than Kafka
- Full control

**Cons:**
- You manage everything (deployment, upgrades, scaling, monitoring)
- Smaller community than Kafka
- No managed Schema Registry

**Best for:** Cost-conscious teams with DevOps expertise

### Decision Matrix

| Factor | Confluent Cloud | AWS MSK | Redpanda Self-Hosted |
|--------|----------------|---------|---------------------|
| **Cost** | $150-200/mo | $250-300/mo | $100/mo |
| **Operational burden** | Zero | Medium | High |
| **AWS integration** | Via connectors | Native | Via connectors |
| **Schema Registry** | Included | Separate (Glue) | Built-in |
| **Scaling** | Automatic | Manual | Manual |
| **Best for** | Production, focus on data | AWS-heavy shops | Budget, DevOps-savvy |

**Recommendation:** Start with **Confluent Cloud Basic** for simplicity. Migrate to self-hosted later if cost becomes a concern at scale.

## What You'll Build

In this section, you'll create a complete streaming data pipeline:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   STREAMING PIPELINE ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. Confluent Cloud Setup                                               │
│     ┌────────────────────────────────────────────────────┐             │
│     │ Kafka Cluster (Basic tier)                        │             │
│     │ • Topic: order-events                             │             │
│     │ • Schema Registry (Avro schemas)                  │             │
│     │ • Kafka Connect (Snowflake Sink Connector)        │             │
│     └────────────────────────────────────────────────────┘             │
│                              │                                          │
│  2. Event Production         ▼                                          │
│     ┌────────────────────────────────────────────────────┐             │
│     │ Python Producer (Prefect task)                    │             │
│     │ • Generate sample order events                    │             │
│     │ • Validate against Avro schema                    │             │
│     │ • Publish to order-events topic                   │             │
│     └────────────────────────────────────────────────────┘             │
│                              │                                          │
│  3. Snowflake Integration    ▼                                          │
│     ┌────────────────────────────────────────────────────┐             │
│     │ Snowpipe Streaming                                │             │
│     │ • STREAMING.ORDER_EVENTS table                    │             │
│     │ • Sub-second latency                              │             │
│     │ • SVC_KAFKA_CONNECTOR service account             │             │
│     └────────────────────────────────────────────────────┘             │
│                              │                                          │
│  4. dbt Transformation       ▼                                          │
│     ┌────────────────────────────────────────────────────┐             │
│     │ Incremental dbt models                            │             │
│     │ • Process new events every 5 minutes              │             │
│     │ • ANALYTICS.fct_orders_realtime                   │             │
│     └────────────────────────────────────────────────────┘             │
│                              │                                          │
│  5. Monitoring               ▼                                          │
│     ┌────────────────────────────────────────────────────┐             │
│     │ Prefect + Confluent Cloud Metrics                 │             │
│     │ • Monitor consumer lag                            │             │
│     │ • Alert on connector failures                     │             │
│     │ • Track throughput and latency                    │             │
│     └────────────────────────────────────────────────────┘             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Core Pipeline (9 Pages)

You'll learn:
1. Kafka concepts (topics, partitions, schemas)
2. Set up Confluent Cloud (cluster, Schema Registry, connectors)
3. Configure Snowflake for streaming (STREAMING database, service account)
4. Deploy Snowflake Kafka Connector
5. Write Python producers
6. Monitor with Prefect
7. Verify end-to-end latency

### Optional Advanced Topics (3 Pages)

- **Self-hosted MSK** (alternative to Confluent Cloud)
- **Stream processing** (ksqlDB or Flink for transformations)
- **Change Data Capture** (Debezium for PostgreSQL replication)

## Prerequisites

Before starting this section, ensure you have completed:

- [x] [Orchestration](../orchestration/index.md) — Prefect for task scheduling
- [x] [Data Warehouse](../data-warehouse/index.md) — Snowflake with role-based access
- [x] [Batch Data Ingestion](../batch-data-ingestion/index.md) — Understanding of data loading patterns
- [x] [Data Transformation](../data-transformation/index.md) — dbt for processing streamed data

You'll extend these foundations to handle real-time data.

## Cost Summary

### Confluent Cloud Approach (~$200/month)

| Component | Monthly Cost |
|-----------|-------------|
| **Confluent Cloud Basic** (1 CKU) | $150 |
| **Data transfer** (1 GB/day ingress) | $2.50 |
| **Storage** (30-day retention, 30 GB) | $2.50 |
| **Schema Registry** | Included |
| **Kafka Connect** | Included |
| **AWS Secrets Manager** (Confluent credentials) | $0.50 |
| **Snowflake compute** (Snowpipe Streaming) | $20-50 |
| **Total** | **~$175-205/month** |

### Self-Hosted Redpanda Approach (~$100/month)

| Component | Monthly Cost |
|-----------|-------------|
| **ECS Fargate** (3 × Redpanda nodes) | $60 |
| **ALB** | $20 |
| **Storage** (EBS volumes) | $15 |
| **CloudWatch Logs** | $5 |
| **Total** | **~$100/month** |

### Comparison to Batch

**Batch ingestion (dlt):** $0/month (just compute)
**Streaming ingestion (Confluent Cloud):** $200/month

**When streaming is worth it:** When latency reduction (hours → seconds) provides business value exceeding $200/month (e.g., fraud prevention, real-time personalisation).

## Section Contents

| Page | What You'll Do |
|------|----------------|
| [Kafka Concepts](1-kafka-concepts.md) | Understand topics, partitions, producers, consumers, Schema Registry |
| [Deployment Options](2-deployment-options.md) | Compare Confluent Cloud, MSK, Redpanda - detailed costs and decision criteria |
| [Confluent Cloud Setup](3-confluent-cloud-setup.md) | Create cluster, configure Schema Registry, generate API keys |
| [Snowflake Infrastructure](4-snowflake-infrastructure.md) | STREAMING database, SVC_KAFKA_CONNECTOR service account |
| [Kafka Connect Snowflake](5-kafka-connect-snowflake.md) | Deploy Snowflake Sink Connector, configure Snowpipe Streaming |
| [Producing Events](6-producing-events.md) | Python producers with Avro schemas, Prefect task integration |
| [Prefect Orchestration](7-prefect-orchestration.md) | Monitor consumer lag, orchestrate event-driven flows |
| [Finishing Up](8-finishing-up.md) | Verify end-to-end latency, cost review, next steps |
| [MSK Self-Hosted](9-msk-self-hosted.md) (Optional) | AWS MSK cluster via Terraform, alternative to Confluent |
| [Stream Processing](10-stream-processing.md) (Optional) | ksqlDB or Flink for real-time transformations |
| [Change Data Capture](11-change-data-capture.md) (Optional) | Debezium for PostgreSQL CDC, database replication |

## Why Streaming Matters

Streaming data ingestion unlocks use cases that batch processing cannot address:

**Real-time fraud detection** — Block suspicious transactions immediately, not hours later

**Live operational dashboards** — Monitor Black Friday sales as they happen, react to issues instantly

**Event-driven automation** — Trigger workflows when specific events occur (e.g., send welcome email when user signs up)

**IoT monitoring** — Alert maintenance teams immediately when sensors detect anomalies

**Database replication** — Keep analytics warehouse in sync with production database in real-time

**Audit trails** — Capture every change to critical data for compliance

You're building batch pipelines for most data. Now add streaming for the data that **must** be real-time.

## Get Started

Begin by understanding Kafka fundamentals: topics, partitions, and consumer groups.

Continue to [Kafka Concepts](1-kafka-concepts.md) →
