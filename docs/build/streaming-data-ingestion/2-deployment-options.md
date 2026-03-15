# Deployment Options

On this page, you will:

- [x] Compare Confluent Cloud (Basic/Standard/Dedicated) pricing tiers
- [x] Evaluate AWS MSK (Serverless vs Provisioned) costs and trade-offs
- [x] Assess Redpanda self-hosted on ECS as a budget option
- [x] Review total cost of ownership (infrastructure + ops time)
- [x] Use decision matrix to choose the right deployment
- [x] Understand migration paths between options

## Overview

Choosing the right Kafka deployment is a balance between **cost**, **operational burden**, and **feature requirements**. Unlike batch ingestion (which is essentially free with dlt), streaming requires running a Kafka cluster 24/7.

This page provides detailed cost breakdowns and decision criteria to help you choose between:

1. **Confluent Cloud** (fully managed, easiest, highest cost)
2. **AWS MSK** (AWS-native, mid-cost, some operational burden)
3. **Redpanda self-hosted** (lowest cost, highest operational burden)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DEPLOYMENT OPTIONS COMPARISON                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Confluent Cloud                                                        │
│  ────────────────                                                       │
│  ┌──────────────────────────────────────────────────────┐              │
│  │ Fully Managed (Kafka + Schema Registry + Connect)   │              │
│  │ • Basic: $150/mo (1 CKU)                            │              │
│  │ • Standard: $300/mo (multi-AZ, better SLA)          │              │
│  │ • Dedicated: $500+/mo (isolated, private network)   │              │
│  │ • Zero ops burden                                   │              │
│  └──────────────────────────────────────────────────────┘              │
│                                                                         │
│  AWS MSK                                                                │
│  ────────                                                               │
│  ┌──────────────────────────────────────────────────────┐              │
│  │ AWS-Native Kafka                                     │              │
│  │ • Serverless: ~$0.08/GB ingested + storage          │              │
│  │ • Provisioned: ~$250/mo (3x kafka.m5.large)         │              │
│  │ • Medium ops burden (config, monitoring, upgrades)  │              │
│  │ • Need separate Schema Registry (Glue or Confluent) │              │
│  └──────────────────────────────────────────────────────┘              │
│                                                                         │
│  Redpanda Self-Hosted (ECS)                                             │
│  ───────────────────────                                                │
│  ┌──────────────────────────────────────────────────────┐              │
│  │ Kafka-Compatible, Simpler Architecture              │              │
│  │ • ECS Fargate: ~$100/mo (3 nodes + ALB)             │              │
│  │ • Built-in Schema Registry                          │              │
│  │ • High ops burden (deploy, scale, monitor, debug)   │              │
│  │ • Full control                                      │              │
│  └──────────────────────────────────────────────────────┘              │
│                                                                         │
│  Recommendation: Start with Confluent Cloud Basic                       │
│  Optimise later if streaming becomes business-critical at scale         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Confluent Cloud (Recommended)

### What is Confluent Cloud?

**Confluent Cloud** is a fully managed Kafka service run by the company that created Kafka. You get:

- **Kafka cluster** — Brokers, Zookeeper, monitoring, upgrades
- **Schema Registry** — Avro/Protobuf/JSON Schema versioning
- **Kafka Connect** — 120+ pre-built connectors (including Snowflake)
- **ksqlDB** — Stream processing (optional)
- **Global infrastructure** — Available in 70+ AWS/GCP/Azure regions

### Pricing Tiers

#### Basic Cluster

**Cost:** ~$150/month base + usage

**What's included:**
- 1 CKU (Confluent Kafka Unit) = dedicated compute capacity
- Shared infrastructure (multi-tenant brokers)
- Single-zone deployment
- 99.5% SLA
- Schema Registry included
- Kafka Connect included

**Additional costs:**
- **Ingress:** $0.08/GB (data written to Kafka)
- **Egress:** $0.08/GB (data read from Kafka, excluding same-region)
- **Storage:** $0.08/GB/month (data retention)
- **Partition overage:** $0.40/partition/month (over 100 partitions)

**Example workload:**
```
1 GB/day ingress × 30 days = 30 GB ingress = $2.50
1 GB/day egress × 30 days = 30 GB egress = $2.50
30-day retention, 30 GB stored = $2.50
5 topics, 3 partitions each = 15 partitions (no overage)

Total: $150 + $2.50 + $2.50 + $2.50 = $157.50/month
```

**Best for:**
- Getting started with streaming
- Small to medium workloads (< 10 GB/day)
- Teams that want zero operational burden
- Development and staging environments

#### Standard Cluster

**Cost:** ~$300/month base + usage

**What's included:**
- Multi-zone deployment (HA across 3 availability zones)
- 99.95% SLA (higher availability)
- Up to 4,500 partitions included
- All Basic features

**Additional costs:** Same per-GB pricing as Basic

**Best for:**
- Production workloads requiring high availability
- Multi-region applications
- Teams needing stronger SLA guarantees

#### Dedicated Cluster

**Cost:** $500+/month base + usage (varies by region and CKU count)

**What's included:**
- Isolated single-tenant cluster
- Private networking (VPC peering, AWS PrivateLink)
- Encryption at rest
- RBAC and audit logs
- Custom retention policies
- Dedicated support

**Best for:**
- Enterprise security requirements
- Compliance (GDPR, HIPAA, SOC 2)
- Very high throughput (> 100 GB/day)
- Low-latency requirements (< 10ms P99)

### Total Cost of Ownership (Confluent Cloud Basic)

| Component | Monthly Cost |
|-----------|-------------|
| **Base cluster** (1 CKU) | $150 |
| **Ingress** (1 GB/day × 30 days × $0.08/GB) | $2.50 |
| **Egress** (1 GB/day × 30 days × $0.08/GB) | $2.50 |
| **Storage** (30 GB × $0.08/GB) | $2.50 |
| **Schema Registry** | Included |
| **Kafka Connect** | Included |
| **Ops time** (monitoring, alerts) | 1-2 hours/month |
| **Total** | **~$157.50 + engineer time** |

**Engineer cost:** If your engineer costs $60/hour, add $60-120/month = **$217.50-277.50 total**

**When to upgrade to Standard:**
- Production workload with HA requirements
- Multi-region active-active setup
- Need 99.95% SLA for contractual reasons

**When to upgrade to Dedicated:**
- Enterprise security/compliance requirements
- Throughput > 100 GB/day
- Need VPC peering or PrivateLink

## AWS MSK (Managed Streaming for Kafka)

### What is AWS MSK?

**AWS Managed Streaming for Kafka** (MSK) is Amazon's managed Kafka service. It handles broker infrastructure but requires more configuration than Confluent Cloud.

**What you manage:**
- Kafka topic configuration
- Consumer/producer applications
- Monitoring and alerting
- Upgrades (with MSK handling broker patching)

**What AWS manages:**
- Broker instances
- Multi-AZ replication
- Zookeeper (or KRaft in newer versions)
- OS patching
- Backups

### Deployment Options

#### MSK Serverless

**Cost:** Pay per usage (no base cost)

**Pricing:**
- **Ingress:** $0.08/GB (data written)
- **Storage:** $0.08/GB/month (data retained)
- **Partition hours:** ~$0.0015/partition/hour = ~$1.10/partition/month

**Example workload:**
```
1 GB/day ingress × 30 days = 30 GB = $2.50
30 GB storage × $0.08 = $2.50
5 topics × 3 partitions × $1.10 = $16.50

Total: $21.50/month
```

**Pros:**
- No base cluster cost
- Scales to zero when idle
- Good for spiky workloads

**Cons:**
- Higher per-GB cost than provisioned at scale
- Limited configuration options
- No Schema Registry (need Glue or Confluent)

**Best for:**
- Development environments
- Unpredictable workloads
- Cost-sensitive use cases with low throughput

#### MSK Provisioned

**Cost:** ~$250/month (3-broker cluster)

**Instance types:**
- **kafka.t3.small** (2 vCPU, 2 GB RAM) — Dev/test only
- **kafka.m5.large** (2 vCPU, 8 GB RAM) — Small production (~10 GB/day)
- **kafka.m5.xlarge** (4 vCPU, 16 GB RAM) — Medium production (~50 GB/day)

**Example configuration:**
```
3 brokers × kafka.m5.large ($0.115/hour) × 730 hours/month = $252/month
+ Storage: 100 GB EBS per broker × 3 × $0.08/GB = $24/month
+ Data transfer: Minimal (same-AZ free)

Total: $276/month
```

**Pros:**
- Predictable pricing
- Better performance than Serverless at high throughput
- More configuration control

**Cons:**
- Always-on cost (even if idle)
- Manual capacity planning
- No built-in Schema Registry

**Best for:**
- Consistent, predictable workloads
- Production with > 10 GB/day throughput
- AWS-centric teams comfortable with Kafka

### Total Cost of Ownership (MSK Provisioned)

| Component | Monthly Cost |
|-----------|-------------|
| **MSK cluster** (3 × kafka.m5.large) | $252 |
| **EBS storage** (3 × 100 GB × $0.08) | $24 |
| **Schema Registry** (Confluent Cloud or Glue) | $20-50 |
| **Kafka Connect** (ECS Fargate, 1 vCPU) | $30 |
| **CloudWatch metrics** | $10 |
| **Ops time** (monitoring, config, upgrades) | 4-8 hours/month |
| **Total** | **~$336-366 + engineer time** |

**Engineer cost:** At $60/hour, add $240-480/month = **$576-846 total**

**Key trade-off:** MSK is cheaper on infrastructure but significantly higher on ops time compared to Confluent Cloud.

## Redpanda Self-Hosted (Budget Option)

### What is Redpanda?

**Redpanda** is a Kafka-compatible streaming platform written in C++ (not Java). Key differences:

- **No Zookeeper** — Simpler architecture (just Redpanda nodes)
- **Faster** — 10x lower latency, higher throughput than Kafka
- **Smaller footprint** — Runs on fewer resources
- **Kafka-compatible** — Drop-in replacement (same APIs, same client libraries)

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    REDPANDA ON ECS (3-NODE CLUSTER)                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌────────────────┐      ┌────────────────┐      ┌────────────────┐    │
│  │ ECS Task 1     │      │ ECS Task 2     │      │ ECS Task 3     │    │
│  │ (AZ-1)         │      │ (AZ-2)         │      │ (AZ-3)         │    │
│  │                │      │                │      │                │    │
│  │ Redpanda node  │◀────▶│ Redpanda node  │◀────▶│ Redpanda node  │    │
│  │ + Schema Reg   │      │ + Schema Reg   │      │ + Schema Reg   │    │
│  └────────────────┘      └────────────────┘      └────────────────┘    │
│         │                         │                         │           │
│         └─────────────────────────┼─────────────────────────┘           │
│                                   │                                     │
│                                   ▼                                     │
│                         ┌────────────────┐                              │
│                         │ Application    │                              │
│                         │ Load Balancer  │                              │
│                         └────────────────┘                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Infrastructure Cost

| Component | Specification | Monthly Cost |
|-----------|--------------|-------------|
| **ECS Fargate** (3 nodes) | 0.5 vCPU, 1 GB RAM each, 24/7 | $54 |
| **Application Load Balancer** | Basic usage | $16 |
| **EBS volumes** (persistent storage) | 3 × 50 GB gp3 | $15 |
| **CloudWatch Logs** | ~10 GB/month | $5 |
| **VPC data transfer** (minimal) | Same-AZ | $0 |
| **Total infrastructure** | | **$90/month** |

### Total Cost of Ownership (Redpanda Self-Hosted)

| Component | Monthly Cost |
|-----------|-------------|
| **Infrastructure** (as above) | $90 |
| **Setup time** (initial Terraform, testing) | 20-40 hours (one-time) |
| **Ops time** (monitoring, upgrades, debugging) | 8-16 hours/month |
| **Total** | **$90 + engineer time** |

**Engineer cost:** At $60/hour, add $480-960/month = **$570-1,050 total**

**Break-even analysis:**
- Infrastructure saving vs Confluent Cloud: $157 - $90 = $67/month
- Extra ops time: 6-14 hours/month more than Confluent Cloud
- At $60/hour, extra ops cost: $360-840/month

**Conclusion:** Redpanda self-hosted is **more expensive overall** unless:
- You already have DevOps engineers with spare capacity
- You have unique requirements (custom config, on-prem deployment)
- You're learning Kafka architecture for educational purposes

### Pros and Cons

**Pros:**
- Lowest infrastructure cost ($90/month)
- Full control over configuration
- Kafka-compatible (easy migration to/from Kafka)
- Built-in Schema Registry
- No Zookeeper complexity

**Cons:**
- High operational burden (deployment, scaling, monitoring, upgrades, debugging)
- Requires Terraform/Docker/ECS expertise
- Smaller community than Kafka (fewer resources, examples)
- You're responsible for security patches, backup, disaster recovery
- No managed Kafka Connect (need to self-host separately)

**Best for:**
- Learning Kafka architecture
- Unique requirements not met by managed services
- Teams with existing DevOps capacity
- On-premises deployments

## Decision Matrix

### By Team Size

| Team Size | Recommended Option | Rationale |
|-----------|-------------------|-----------|
| **Solo/1-2 engineers** | Confluent Cloud Basic | Zero ops burden, focus on data pipelines |
| **3-5 engineers** | Confluent Cloud Basic or Standard | Still worth avoiding Kafka operations |
| **5-10 engineers** | Confluent Cloud or MSK | Can consider MSK if AWS-heavy already |
| **10+ engineers** | MSK or Dedicated Confluent | Dedicated DevOps can manage MSK or justify Dedicated tier |

### By Budget

| Monthly Budget | Recommended Option | Notes |
|----------------|-------------------|-------|
| **< $100/month** | Reconsider if you need streaming | Streaming is fundamentally expensive |
| **$100-300/month** | Confluent Cloud Basic | Best balance of cost and simplicity |
| **$300-600/month** | Confluent Cloud Standard or MSK | HA requirements justify Standard |
| **$600+/month** | Confluent Cloud Dedicated or MSK | Enterprise features, compliance, scale |

### By Workload

| Workload | Recommended Option | Rationale |
|----------|-------------------|-----------|
| **< 1 GB/day, dev/test** | MSK Serverless or Confluent Basic | MSK Serverless cheaper for low volume |
| **1-10 GB/day, production** | Confluent Cloud Basic | Sweet spot for managed service |
| **10-100 GB/day, production** | Confluent Cloud Standard or MSK | Need HA, consider MSK cost advantage |
| **> 100 GB/day** | Confluent Cloud Dedicated or MSK | Dedicated resources required |

### By Expertise

| Team Expertise | Recommended Option | Rationale |
|----------------|-------------------|-----------|
| **No Kafka experience** | Confluent Cloud | Learn Kafka without operational complexity |
| **Some Kafka experience** | Confluent Cloud | Ops burden still not worth savings |
| **Strong DevOps + Kafka experts** | MSK or self-hosted | Can capture cost savings |

### By Requirements

| Requirement | Confluent Cloud | AWS MSK | Redpanda |
|-------------|----------------|---------|----------|
| **Zero ops burden** | ✅ Basic/Standard | ❌ | ❌ |
| **Multi-AZ HA** | ✅ Standard/Dedicated | ✅ Provisioned | ✅ (DIY) |
| **Schema Registry included** | ✅ | ❌ (need Glue/Confluent) | ✅ |
| **Kafka Connect included** | ✅ | ❌ (self-host) | ❌ (self-host) |
| **VPC peering/PrivateLink** | ✅ Dedicated | ✅ | ✅ (DIY) |
| **Lowest infrastructure cost** | ❌ | ⚠️ (Serverless) | ✅ |
| **Compliance (SOC 2, HIPAA)** | ✅ Dedicated | ✅ | ⚠️ (DIY) |

## Migration Paths

### Start Small, Scale Up

**Phase 1: Development (Months 0-3)**
- **Confluent Cloud Basic** — Learn streaming concepts, build initial pipelines
- **Cost:** ~$160/month
- **Effort:** Focus on data, not infrastructure

**Phase 2: Production (Months 3-12)**
- **Upgrade to Standard** — Add HA for production workload
- **Cost:** ~$310/month
- **Trigger:** Ready to deploy streaming to production

**Phase 3: Scale (Months 12+)**
- **Stay on Standard** if throughput < 50 GB/day
- **Migrate to MSK** if infrastructure cost becomes significant (> $500/month)
- **Upgrade to Dedicated** if compliance/security requirements emerge

### Migration: Confluent Cloud → MSK

**When to consider:**
- Confluent Cloud cost > $500/month
- Team has hired DevOps engineers with Kafka expertise
- AWS-centric architecture benefits from native integration

**Migration approach:**
1. Set up MSK cluster via Terraform
2. Create topics with same configuration
3. Deploy Kafka Connect separately (ECS or EC2)
4. Dual-write producers to both clusters (shadow mode)
5. Cutover consumers to MSK
6. Decommission Confluent Cloud

**Downtime:** ~30 minutes (consumer cutover)

**Effort:** 40-80 hours (planning, testing, migration, verification)

### Migration: MSK → Confluent Cloud

**When to consider:**
- Ops burden too high (team spending > 8 hours/month on Kafka)
- Lack of Kafka expertise (engineer turnover)
- Want to focus on data pipelines, not infrastructure

**Migration approach:** Same as above (MSK and Confluent are both Kafka)

**Cost change:** +$150-300/month infrastructure, -$300-600/month ops time

## Cost Comparison: 12-Month Total

### Scenario: 5 GB/day workload, 2-person data team

| Option | Infra Cost (12mo) | Ops Time (12mo) | Ops Cost @ $60/hr | Total Cost |
|--------|------------------|----------------|------------------|-----------|
| **Confluent Cloud Basic** | $2,100 | 24 hours | $1,440 | **$3,540** |
| **MSK Provisioned** | $4,032 | 96 hours | $5,760 | **$9,792** |
| **Redpanda Self-Hosted** | $1,080 | 144 hours | $8,640 | **$9,720** |

**Winner:** Confluent Cloud Basic (60% cheaper overall)

### Scenario: 50 GB/day workload, 5-person team with DevOps

| Option | Infra Cost (12mo) | Ops Time (12mo) | Ops Cost @ $60/hr | Total Cost |
|--------|------------------|----------------|------------------|-----------|
| **Confluent Cloud Standard** | $4,800 | 24 hours | $1,440 | **$6,240** |
| **MSK Provisioned (larger)** | $6,000 | 60 hours | $3,600 | **$9,600** |

**Winner:** Still Confluent Cloud (35% cheaper)

**MSK becomes competitive at:**
- Very high throughput (> 100 GB/day)
- Team already managing MSK for other use cases (amortised ops cost)
- Data sovereignty requirements (can't use Confluent Cloud)

## Summary

You've compared deployment options for Kafka:

- [x] **Confluent Cloud Basic** — $160/month, zero ops, best for most teams
- [x] **Confluent Cloud Standard** — $310/month, HA, 99.95% SLA, production workloads
- [x] **Confluent Cloud Dedicated** — $500+/month, compliance, high throughput
- [x] **AWS MSK Serverless** — $20-100/month, pay per usage, dev/test only
- [x] **AWS MSK Provisioned** — $280-600/month, AWS-native, medium ops burden
- [x] **Redpanda self-hosted** — $90/month infrastructure, high ops burden, rarely worth it
- [x] **Decision matrix** — Choose based on team size, budget, expertise, requirements
- [x] **Migration paths** — Start small (Basic), scale up (Standard), migrate if needed (MSK)

**Recommendation:** Start with **Confluent Cloud Basic** for simplicity. Your total cost (infrastructure + ops time) will almost always be lower than self-managed options.

## What's Next

Set up your Confluent Cloud account and create your first Kafka cluster.

Continue to [Confluent Cloud Setup](3-confluent-cloud-setup.md) →
