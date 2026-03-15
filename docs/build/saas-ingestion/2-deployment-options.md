# Deployment Options

On this page, you will:

- [x] Compare Airbyte Cloud and self-hosted deployments
- [x] Understand the cost implications of each approach
- [x] Choose the right deployment for your team

## Overview

Airbyte offers two deployment models:

1. **Airbyte Cloud** — fully managed SaaS, pay per usage
2. **Self-Hosted** — run Airbyte on your own infrastructure

Both provide the same connectors and sync capabilities. The difference is operational overhead vs cost control.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       DEPLOYMENT OPTIONS                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Option 1: Airbyte Cloud              Option 2: Self-Hosted (ECS)           │
│  ─────────────────────────             ──────────────────────────           │
│                                                                             │
│  ┌───────────────────────┐             ┌───────────────────────┐            │
│  │  Airbyte manages:     │             │  You manage:          │            │
│  │  • Infrastructure     │             │  • ECS Fargate tasks  │            │
│  │  • Upgrades           │             │  • RDS PostgreSQL     │            │
│  │  • Scaling            │             │  • ALB                │            │
│  │  • Monitoring         │             │  • Upgrades           │            │
│  └───────────────────────┘             └───────────────────────┘            │
│                                                                             │
│  Cost: $99+/month                      Cost: ~$81/month                     │
│  Setup: Sign up + configure            Setup: Terraform + deploy            │
│  Maintenance: None                     Maintenance: Ongoing                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Airbyte Cloud

### How It Works

Airbyte Cloud is a fully managed service. You:

1. Sign up at [cloud.airbyte.com](https://cloud.airbyte.com)
2. Configure sources and destinations via the UI
3. Airbyte handles infrastructure, scaling, and upgrades

### Pricing

Airbyte Cloud uses a credit-based pricing model:

| Tier | Monthly Cost | Connections | Records Included | Record Overage |
|------|-------------|-------------|-----------------|----------------|
| Free | $0 | 5 | Limited credits | N/A |
| Starter | $99 | 4 | 4M records | $15 per 1M records |
| Team | $499 | 10 | 20M records | $10 per 1M records |
| Enterprise | Custom | Unlimited | Custom | Custom |

!!! note "Record Counting"
    Airbyte counts records across all connections. A single HubSpot contacts sync of 10,000 contacts counts as 10,000 records. Full refresh syncs re-count all records each time.

### Cost Example: HubSpot Contacts Only

For a small HubSpot instance with 5,000 contacts:

| Scenario | Monthly Records | Monthly Cost |
|----------|----------------|-------------|
| Incremental daily (100 changes/day) | ~3,000 | $99 (Starter minimum) |
| Full refresh daily | ~150,000 | $99 (within Starter allowance) |
| Full refresh + 5 other connectors | ~500,000 | $99 (within Starter allowance) |

For most small-to-medium teams, the Starter tier is sufficient.

### Advantages

- **No infrastructure to manage**: No servers, databases, or upgrades
- **Quick setup**: Functional within an hour
- **Built-in monitoring**: Dashboard, alerts, and sync history
- **Automatic updates**: New connector versions deployed automatically
- **SOC 2 compliance**: Enterprise security built in

### Disadvantages

- **Cost at scale**: Per-record pricing adds up with many connectors or high volumes
- **Data residency**: Data passes through Airbyte's infrastructure (US or EU region)
- **Less control**: Cannot customise connectors or infrastructure
- **Vendor lock-in**: Dependent on Airbyte's pricing and availability

## Self-Hosted on ECS

### How It Works

You deploy Airbyte's open-source platform on AWS ECS Fargate. This gives you full control over infrastructure and eliminates per-record pricing.

### Infrastructure Components

| Component | Service | Purpose |
|-----------|---------|---------|
| Airbyte Server | ECS Fargate | API server, scheduler, UI |
| Airbyte Workers | ECS Fargate | Run connector jobs |
| Metadata Database | RDS PostgreSQL | Stores configuration, state, logs |
| Load Balancer | ALB | HTTPS access to UI and API |
| Container Registry | ECR | Stores Airbyte Docker images |

### Cost Breakdown

| Component | Specification | Monthly Cost |
|-----------|--------------|-------------|
| ECS Fargate (server) | 1 vCPU, 2 GB | ~$30 |
| ECS Fargate (workers) | 1 vCPU, 2 GB (scales to 0) | ~$20 |
| RDS PostgreSQL | db.t4g.micro, 20 GB | ~$15 |
| ALB | 1 ALB | ~$16 |
| **Total** | | **~$81/month** |

!!! tip "Worker Scaling"
    Workers scale to zero when no syncs are running, so actual worker costs depend on sync frequency and duration. The $20 estimate assumes ~8 hours of active syncs per day.

### Advantages

- **No per-record pricing**: Pay only for infrastructure
- **Data stays in your VPC**: No data leaves your AWS account
- **Full control**: Customise connectors, infrastructure, and configuration
- **Open source**: No vendor lock-in, can migrate to another hosting method

### Disadvantages

- **Operational overhead**: You manage upgrades, scaling, and monitoring
- **Setup complexity**: Terraform infrastructure + networking + security
- **Upgrade management**: Major version upgrades can require migration steps
- **No built-in alerting**: Need to set up CloudWatch alarms separately

## Decision Matrix

| Factor | Airbyte Cloud | Self-Hosted |
|--------|:------------:|:-----------:|
| Time to first sync | Fast | Slower |
| Monthly cost (small scale) | $99 | ~$81 |
| Monthly cost (large scale) | $499+ | ~$81 (fixed) |
| Infrastructure management | None | Ongoing |
| Data residency | US/EU (Airbyte servers) | Your VPC |
| Connector customisation | No | Yes |
| Upgrades | Automatic | Manual |
| SOC 2 | Included | Your responsibility |

### Recommendation

**Start with Airbyte Cloud** if:

- You have fewer than 5 SaaS sources
- Total monthly records are under 4M
- You want to validate the approach before investing in infrastructure
- Your team is small and cannot dedicate time to infrastructure management

**Use self-hosted** if:

- You have strict data residency requirements
- Monthly record volumes exceed 20M (cost savings outweigh operational overhead)
- You need to customise connectors
- You already have ECS infrastructure and operational experience

!!! tip "Migration Path"
    You can start with Airbyte Cloud and migrate to self-hosted later. Connection configurations can be exported and imported. The Snowflake infrastructure (database, roles, schemas) remains the same regardless of deployment.

## This Guide's Approach

This guide documents **both** approaches:

- [Airbyte Cloud Setup](3-airbyte-cloud-setup.md) covers the managed service
- [Self-Hosted Setup](4-self-hosted-setup.md) covers ECS deployment (optional)

The Snowflake infrastructure, HubSpot connection, and Prefect orchestration pages apply to both deployment methods.

## Summary

You've evaluated the deployment options:

- [x] Airbyte Cloud: managed, $99+/month, no infrastructure to manage
- [x] Self-hosted on ECS: ~$81/month fixed, full control, operational overhead
- [x] Chosen your deployment approach

## What's Next

Set up your chosen deployment.

Continue to [Airbyte Cloud Setup](3-airbyte-cloud-setup.md) → (or skip to [Self-Hosted Setup](4-self-hosted-setup.md) if deploying on ECS)
