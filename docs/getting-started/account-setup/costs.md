# Cost Overview

This page provides an overview of the costs associated with the tools and services used in this data stack. Understanding costs upfront helps with budgeting and avoiding surprises.

## Cost Philosophy

This stack follows a pragmatic approach to costs:

- **Start small**: Begin with free tiers and minimal configurations
- **Pay for value**: Invest in managed services that reduce operational burden
- **Scale gradually**: Increase spend only as data volumes and team size grow
- **Monitor continuously**: Set up billing alerts before costs become a problem

## Current Stack Costs

These are the tools introduced so far in the documentation, organised by category.

### Version Control & CI/CD

| Tool | Free Tier | Paid Tier | Pricing Link |
|------|-----------|-----------|--------------|
| **GitHub** | Free (public repos, limited private) | Team: $4/user/month | [github.com/pricing](https://github.com/pricing) |
| **GitHub Actions** | 2,000 mins/month (Free), 3,000 mins (Team) | $0.008/min (Linux) after allowance | [GitHub Actions billing](https://docs.github.com/en/billing/managing-billing-for-github-actions/about-billing-for-github-actions) |

!!! info "GitHub Plan Recommendations"
    - **Solo/Learning**: Free tier is sufficient
    - **Small team (2-5)**: Team plan for protected branches and CODEOWNERS
    - **Larger team**: Enterprise for SAML SSO and advanced security

### Secrets Management

| Tool | Free Tier | Paid Tier | Pricing Link |
|------|-----------|-----------|--------------|
| **1Password** | None (14-day trial) | Teams: $19.95/month (up to 10 users) | [1password.com/teams/pricing](https://1password.com/teams/pricing/) |
| **AWS Secrets Manager** | None | $0.40/secret/month + $0.05/10,000 API calls | [AWS Secrets Manager pricing](https://aws.amazon.com/secrets-manager/pricing/) |

!!! tip "1Password Alternatives"
    If 1Password doesn't fit your budget, consider:

    - **Bitwarden Teams**: $4/user/month
    - **LastPass Teams**: $4/user/month

    The concepts in this documentation apply to any password manager with shared vaults.

### Cloud Infrastructure

| Tool | Free Tier | Paid Tier | Pricing Link |
|------|-----------|-----------|--------------|
| **AWS** | 12-month free tier for new accounts | Pay-as-you-go | [aws.amazon.com/pricing](https://aws.amazon.com/pricing/) |
| **AWS S3** | 5GB (free tier) | ~$0.023/GB/month (Standard) | [S3 pricing](https://aws.amazon.com/s3/pricing/) |
| **AWS DynamoDB** | 25GB + 25 read/write units | Pay-per-request or provisioned | [DynamoDB pricing](https://aws.amazon.com/dynamodb/pricing/) |
| **AWS VPC** | VPC, subnets, Internet Gateway: Free | NAT Gateway: ~$32/month + data | [VPC pricing](https://aws.amazon.com/vpc/pricing/) |

!!! note "S3 Data Lake Costs"
    The stack creates three S3 buckets for the data lake (dev, staging, prod). Costs depend on data volume:

    - **Storage**: ~$0.023/GB/month (Standard), less for Infrequent Access
    - **Requests**: $0.005 per 1,000 PUT requests, $0.0004 per 1,000 GET requests
    - **Data transfer**: Free within the same region, egress charges for cross-region

    For a small data platform, expect $5-50/month for S3 depending on data volume.

### Data Warehouse

| Tool | Free Tier | Paid Tier | Pricing Link |
|------|-----------|-----------|--------------|
| **Snowflake** | $400 credit trial (30 days) | Compute + Storage (varies by edition) | [snowflake.com/pricing](https://www.snowflake.com/en/data-cloud/pricing-options/) |

Snowflake pricing is based on:

- **Compute**: Credits consumed by warehouses (per-second billing, 60-second minimum)
- **Storage**: ~$23/TB/month (compressed data, typically 3-5x compression ratio)
- **Data transfer**: Egress from Snowflake (minimal for most use cases)
- **Edition**: Standard, Enterprise, or Business Critical

For a small team just starting out, expect:

- **Compute**: $50-200/month depending on query volume
- **Storage**: $23/TB/month (your raw data compresses significantly)

!!! warning "Snowflake Cost Management"
    Snowflake can become expensive quickly if warehouses are left running. Always configure:

    - `AUTO_SUSPEND` (60-300 seconds recommended)
    - `AUTO_RESUME = TRUE`
    - Resource monitors with credit limits

!!! tip "Start Small"
    Begin with X-Small warehouses and scale up only when needed. Snowflake makes it easy to resize warehouses without downtime.

## Estimated Monthly Costs

Here's a rough estimate for a small data team (3-5 people) in the early stages:

| Category | Low Estimate | High Estimate | Notes |
|----------|--------------|---------------|-------|
| GitHub Team | $12/month | $20/month | 3-5 users |
| 1Password Teams | $20/month | $20/month | Up to 10 users |
| AWS (Terraform state) | $1/month | $5/month | S3 + DynamoDB minimal usage |
| AWS (S3 Data Lake) | $5/month | $50/month | 3 buckets (dev, staging, prod) |
| AWS (VPC) | $0/month | $35/month | Optional - only if running ECS/EC2 |
| AWS Secrets Manager | $2/month | $10/month | 5-25 secrets |
| Snowflake | $50/month | $300/month | Depends heavily on usage |
| **Total** | **~$90/month** | **~$405/month** | |

!!! info "Your Costs Will Vary"
    These estimates assume minimal usage during initial setup. Costs will increase as you:

    - Add more data sources
    - Run more frequent transformations
    - Increase warehouse sizes for performance
    - Store more data

## Future Stack Costs

As you progress through the documentation, you'll encounter these additional tools:

### Orchestration (Prefect)

Prefect offers three deployment options with different cost profiles:

#### Option 1: Prefect Cloud (Recommended to Start)

| Tier | Monthly Cost | Users | Deployments | Key Features |
|------|-------------|-------|-------------|--------------|
| **Hobby** | Free | 2 | 5 | Basic orchestration, 7-day retention |
| **Starter** | $100 | 3 | 20 | Webhooks, bring your own compute |
| **Team** | $100/user | 4-8 | 100 | Service accounts, audit logs |
| **Pro** | Custom | 5-20 | 1,000 | Multiple workspaces, SSO |
| **Enterprise** | Custom | Unlimited | Unlimited | RBAC, SLA, PrivateLink |

[prefect.io/pricing](https://www.prefect.io/pricing)

!!! tip "Start with Hobby"
    The free Hobby tier is sufficient for getting started and learning. Upgrade as your needs grow.

#### Option 2: Self-Hosted - Docker Compose

Run Prefect on a single EC2 instance with Docker Compose:

| Component | Specification | Monthly Cost |
|-----------|--------------|--------------|
| EC2 instance | t3.small (2 vCPU, 2GB RAM) | ~$15 |
| EBS storage | 20GB gp3 | ~$2 |
| **Total** | | **~$17/month** |

Plus engineer time for maintenance (~2-4 hours/month).

#### Option 3: Self-Hosted - ECS + RDS (Production)

Run Prefect on AWS ECS with RDS PostgreSQL for high availability:

| Component | Specification | Monthly Cost |
|-----------|--------------|--------------|
| ECS Fargate (server) | 0.5 vCPU, 1GB RAM, 24/7 | ~$18 |
| ECS Fargate (worker) | 0.5 vCPU, 1GB RAM, 24/7 | ~$18 |
| RDS PostgreSQL | db.t3.micro, 20GB | ~$15 |
| Application Load Balancer | Basic usage | ~$16 |
| **Total** | | **~$67/month** |

Lower maintenance burden than Docker Compose (~1-2 hours/month).

#### Which Option to Choose?

| Requirement | Recommended Option |
|-------------|-------------------|
| Just getting started | Prefect Cloud (Hobby - free) |
| Small team, simple needs | Prefect Cloud (Starter - $100/mo) |
| Data sovereignty required | Self-hosted Docker Compose |
| Production HA required | Self-hosted ECS + RDS |
| Minimal operational burden | Prefect Cloud |

See [Orchestration Build Section](../../build/orchestration/2-choosing-deployment.md) for detailed comparison.

### Data Ingestion

| Tool | Free Tier | Paid Tier | Pricing Link |
|------|-----------|-----------|--------------|
| **dlt** | Open source (free) | N/A | [dlthub.com](https://dlthub.com/) |
| **Open Exchange Rates** | 1,000 req/month | From $12/month | [openexchangerates.org](https://openexchangerates.org/signup) |
| **Clever Cloud PostgreSQL** | DEV (free, testing only) | From ~€5/month | [clever-cloud.com](https://www.clever-cloud.com/postgresql/) |

### SaaS Ingestion (Airbyte)

Airbyte is used for complex SaaS connectors and reverse ETL. Two deployment options:

#### Airbyte Cloud

| Tier | Monthly Cost | Connections | Records Included | Record Overage |
|------|-------------|-------------|-----------------|----------------|
| **Free** | $0 | 5 | Limited credits | N/A |
| **Starter** | $99 | 4 | 4M records | $15 per 1M records |
| **Team** | $499 | 10 | 20M records | $10 per 1M records |
| **Enterprise** | Custom | Unlimited | Custom | Custom |

[airbyte.com/pricing](https://airbyte.com/pricing)

#### Airbyte Self-Hosted (ECS)

| Component | Specification | Monthly Cost |
|-----------|--------------|-------------|
| ECS Fargate (server) | 1 vCPU, 2GB RAM | ~$30 |
| ECS Fargate (workers) | 1 vCPU, 2GB RAM (scales to 0) | ~$20 |
| RDS PostgreSQL | db.t4g.micro, 20GB | ~$15 |
| Application Load Balancer | Basic usage | ~$16 |
| **Total** | | **~$81/month** |

!!! tip "dlt Alternative"
    For 1-2 SaaS sources with dlt verified sources (e.g., HubSpot contacts), you can use dlt instead of Airbyte at no extra cost. See [HubSpot Pipeline](../../build/batch-data-ingestion/10-hubspot-pipeline.md) for this approach.

#### Snowpipe Costs

Snowpipe auto-ingestion has minimal costs for low-volume pipelines:

| Component | Cost |
|-----------|------|
| Snowpipe compute | ~$0.06 per 1,000 files processed |
| S3 storage (staging) | ~$0.023/GB/month |
| S3 PUT requests | ~$0.005 per 1,000 requests |
| SQS notifications | Free tier (1 million requests/month) |

For the currencies pipeline (one file per week), expect < $1/month total.

### Alerting (Optional)

| Tool | Free Tier | Paid Tier | Pricing Link |
|------|-----------|-----------|--------------|
| **Slack** | Free (with limits) | Pro: $8.75/user/month | [slack.com/pricing](https://slack.com/pricing) |
| **PagerDuty** | Free (≤5 users) | Professional: $21/user/month | [pagerduty.com/pricing](https://www.pagerduty.com/pricing/) |

!!! tip "Start with Slack Only"
    For most teams starting out, Slack notifications are sufficient. PagerDuty is valuable when you need:

    - 24/7 on-call rotations
    - Escalation policies
    - Phone/SMS alerts for critical failures
    - Incident management

### Transformation

| Tool | Free Tier | Paid Tier | Pricing Link |
|------|-----------|-----------|--------------|
| **dbt Cloud** | Developer (1 seat, free) | Team: $100/seat/month | [getdbt.com/pricing](https://www.getdbt.com/pricing/) |
| **dbt Core** | Open source (free) | N/A | [docs.getdbt.com](https://docs.getdbt.com/) |

### Streaming (Optional)

| Tool | Free Tier | Paid Tier | Pricing Link |
|------|-----------|-----------|--------------|
| **Confluent Cloud** | $400 credit (limited time) | Pay-as-you-go | [confluent.io/pricing](https://www.confluent.io/confluent-cloud/pricing/) |

### Business Intelligence

| Tool | Free Tier | Paid Tier | Pricing Link |
|------|-----------|-----------|--------------|
| **Metabase** | Open source (self-hosted) | Pro: $85/user/month | [metabase.com/pricing](https://www.metabase.com/pricing/) |

## Cost Optimisation Tips

### AWS

- **Use the free tier**: New accounts get 12 months of free tier benefits
- **Set billing alerts**: Configure alerts at $10, $50, $100 thresholds
- **Review unused resources**: Regularly check for orphaned resources
- **Use S3 lifecycle policies**: Move old data to cheaper storage classes

### Snowflake

- **Right-size warehouses**: Start with X-Small, scale up only when needed
- **Use auto-suspend**: Set aggressive auto-suspend (60 seconds for dev)
- **Monitor credits**: Set up resource monitors with email alerts
- **Separate warehouses**: Different warehouses for different workloads (dev vs prod)

### General

- **Annual billing**: Many tools offer discounts for annual commitment
- **Negotiate**: Enterprise pricing is often negotiable
- **Start self-hosted**: Consider self-hosted options before committing to cloud
- **Review monthly**: Set a calendar reminder to review cloud bills

## Setting Up Billing Alerts

### AWS Budget Alerts

Create a budget alert in AWS:

```sh
aws budgets create-budget \
    --account-id YOUR_ACCOUNT_ID \
    --budget '{
        "BudgetName": "Monthly-Spend-Alert",
        "BudgetLimit": {
            "Amount": "100",
            "Unit": "USD"
        },
        "TimeUnit": "MONTHLY",
        "BudgetType": "COST"
    }' \
    --notifications-with-subscribers '[
        {
            "Notification": {
                "NotificationType": "ACTUAL",
                "ComparisonOperator": "GREATER_THAN",
                "Threshold": 80,
                "ThresholdType": "PERCENTAGE"
            },
            "Subscribers": [
                {
                    "SubscriptionType": "EMAIL",
                    "Address": "your-email@company.com"
                }
            ]
        }
    ]'
```

### Snowflake Resource Monitors

Create via Terraform (covered in later sections) or SQL:

```sql
USE ROLE ACCOUNTADMIN;

CREATE RESOURCE MONITOR monthly_limit
    WITH CREDIT_QUOTA = 100
    FREQUENCY = MONTHLY
    START_TIMESTAMP = IMMEDIATELY
    TRIGGERS
        ON 75 PERCENT DO NOTIFY
        ON 90 PERCENT DO NOTIFY
        ON 100 PERCENT DO SUSPEND;

ALTER WAREHOUSE ANALYTICS_WH SET RESOURCE_MONITOR = monthly_limit;
```

## Summary

| Phase | Expected Monthly Cost |
|-------|----------------------|
| **Getting Started** (GitHub, 1Password, minimal AWS) | $30-50 |
| **Terraform Setup** (+ Snowflake dev usage) | $80-150 |
| **Data Warehouse Build** (+ S3 data lake, storage integrations) | $100-400 |
| **Orchestration** (Prefect Cloud free or self-hosted) | $0-100 |
| **Batch Data Ingestion** (dlt, Snowpipe, minimal API costs) | $1-10 |
| **SaaS Ingestion** (Airbyte Cloud Starter or self-hosted) | $81-99 |
| **Full Stack** (orchestration, ingestion, BI) | $500-2,000+ |

The incremental approach means you can pause at any phase and control costs. Start small, monitor usage, and scale as your data needs grow.

!!! success "Key Takeaways"
    - [x] Understand pricing models before signing up
    - [x] Set up billing alerts immediately
    - [x] Start with free tiers and minimal configurations
    - [x] Monitor costs monthly and optimise regularly
