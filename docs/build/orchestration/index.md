# Orchestration with Prefect

In this section, you'll set up Prefect as the central orchestration layer for your data platform. Prefect coordinates all your data workflows - from ingestion pipelines to transformations to ML preprocessing.

## What is Orchestration?

Orchestration is the coordination layer that ties your data stack together. It answers questions like:

- **When** should pipelines run? (Schedules, triggers, events)
- **What** happens when something fails? (Retries, alerts, recovery)
- **How** do we track what happened? (Logging, lineage, observability)
- **Where** do workloads execute? (Workers, containers, serverless)

Without orchestration, you end up with a collection of cron jobs, manual scripts, and tribal knowledge. With orchestration, you get a single control plane for your entire data platform.

## Why Prefect?

Prefect is a modern workflow orchestration platform that replaces traditional tools like Airflow. Key advantages:

| Feature | Prefect | Airflow |
|---------|---------|---------|
| **Setup** | Minutes (Cloud) or simple Docker | Complex (scheduler, webserver, workers, database) |
| **DAG definition** | Pure Python decorators | DSL with operators |
| **Dynamic workflows** | Native support | Limited, requires workarounds |
| **Local development** | Run flows locally, deploy when ready | Requires Airflow environment |
| **Failure handling** | Automatic retries, caching, timeouts | Manual configuration |
| **UI** | Modern, real-time | Functional but dated |

Prefect's philosophy is "orchestration as code" - your workflows are just Python functions with decorators. This makes them easy to test, version, and debug.

## Role in the Stack

Prefect sits at the centre of your data platform, orchestrating everything:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              PREFECT                                        │
│                       (Orchestration Layer)                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │     dlt     │  │   Airbyte   │  │     dbt     │  │  ML/Python  │         │
│  │  Pipelines  │  │    Syncs    │  │   Models    │  │   Scripts   │         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘         │
│         │                │                │                │                │
│         ▼                ▼                ▼                ▼                │
│  ┌─────────────────────────────────────────────────────────────────┐        │
│  │                         Prefect Flows                           │        │
│  │  • Scheduling      • Retries       • Logging      • Alerts      │        │
│  │  • Dependencies    • Caching       • Lineage      • Triggers    │        │
│  └─────────────────────────────────────────────────────────────────┘        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │          Snowflake            │
                    │       (Data Warehouse)        │
                    └───────────────────────────────┘
```

Prefect tasks wrap your:

- **dlt pipelines** - API and database ingestion
- **Airbyte syncs** - SaaS source ingestion
- **dbt models** - Data transformations
- **Python scripts** - ML preprocessing, custom logic

## Deployment Options

This section covers three deployment options to suit different needs:

| Option | Control Plane Cost | Worker Cost | Best For |
|--------|-------------------|-------------|----------|
| [Prefect Cloud](3-prefect-cloud-setup.md) | Free - $100+/mo | + ~$15-50/mo (EC2/ECS) | Most teams (recommended to start) |
| [Docker Compose](4-self-hosted-docker-compose.md) | ~$17/mo (includes worker) | Included | Budget-conscious, data sovereignty |
| [ECS + RDS](5-self-hosted-ecs-production.md) | ~$67/mo (includes worker) | Included | Production HA requirements |

!!! note "Worker Infrastructure"
    With all options, workers run on **your** infrastructure. Prefect Cloud manages the control plane (API, UI, scheduling) but you provide compute for flow execution. The self-hosted options include a worker in the base cost estimate.

All three options use the same Prefect concepts - flows, tasks, deployments, work pools. The only difference is where the control plane runs.

!!! tip "Start with Prefect Cloud"
    We recommend starting with Prefect Cloud's free tier. It gets you running quickly with zero infrastructure management. You can migrate to self-hosted later if needed.

## What You'll Build

By the end of this section, you'll have:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ORCHESTRATION SETUP                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Control Plane (choose one)                                                 │
│  ┌─────────────────────────────────────────────────────────────────┐        │
│  │  Prefect Cloud    OR    Docker Compose    OR    ECS + RDS       │        │
│  └─────────────────────────────────────────────────────────────────┘        │
│                                                                             │
│  Work Pools & Workers                                                       │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐              │
│  │  Process Pool   │  │   Docker Pool   │  │    ECS Pool     │              │
│  │  (Local/EC2)    │  │  (Containers)   │  │  (Serverless)   │              │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘              │
│                                                                             │
│  Infrastructure as Code                                                     │
│  ┌─────────────────────────────────────────────────────────────────┐        │
│  │  Terraform: Work pools, service accounts, EC2/ECS resources     │        │
│  └─────────────────────────────────────────────────────────────────┘        │
│                                                                             │
│  Secrets Integration                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐        │
│  │  AWS Secrets Manager: Snowflake, API keys, credentials          │        │
│  └─────────────────────────────────────────────────────────────────┘        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Prerequisites

Before starting this section, ensure you have completed:

- [x] [AWS Account Setup](../../getting-started/account-setup/aws.md) - AWS account with CLI configured
- [x] [Snowflake Setup](../data-warehouse/index.md) - Data warehouse ready for pipelines
- [x] [Terraform Setup](../../getting-started/terraform-setup/index.md) - Infrastructure as code ready

If using Prefect Cloud:

- [x] [Prefect Cloud Account](../../getting-started/account-setup/prefect.md) - Account and API key ready

## Section Overview

**Core path (recommended):**

| Page | What You'll Learn |
|------|-------------------|
| [1. Prefect Concepts](1-prefect-concepts.md) | Core concepts: flows, tasks, deployments, work pools |
| [2. Choosing a Deployment](2-choosing-deployment.md) | Compare options, understand costs and trade-offs |
| [3. Prefect Cloud Setup](3-prefect-cloud-setup.md) | Set up Prefect Cloud with Terraform (recommended) |
| [6. Work Pools & Workers](6-work-pools-and-workers.md) | Reference for work pool types and configuration |
| [7. Your First Flow](7-first-flow.md) | Create a repository and deploy your first flow |
| [8. Secrets & Blocks](8-secrets-and-blocks.md) | Integrate with AWS Secrets Manager and S3 |
| [9. Finishing Up](9-finishing-up.md) | Summary and next steps |

**Self-hosted options (advanced - requires VPC):**

| Page | What You'll Learn |
|------|-------------------|
| [4. Docker Compose Setup](4-self-hosted-docker-compose.md) | Simple self-hosted on EC2 (~$17/mo) |
| [5. ECS Production Setup](5-self-hosted-ecs-production.md) | Production-grade self-hosted on AWS (~$67/mo) |

!!! note "Self-Hosted Requirements"
    The self-hosted options require AWS VPC and networking infrastructure not covered in this guide. Most users should use Prefect Cloud.

## Cost Considerations

Orchestration costs vary significantly by deployment option. See [Choosing a Deployment](2-choosing-deployment.md) for detailed cost breakdowns, or the [Cost Overview](../../getting-started/account-setup/costs.md) for a summary.

Quick reference:

| Option | Infrastructure | Operational Burden |
|--------|---------------|-------------------|
| Prefect Cloud Free | $0 | None |
| Prefect Cloud Starter | $100/mo | None |
| Docker Compose | ~$17/mo | Medium (2-4 hrs/mo) |
| ECS + RDS | ~$67/mo | Low-Medium |

## What's Next

Start by understanding the core Prefect concepts - flows, tasks, deployments, and work pools. This foundation applies to all deployment options.

Continue to [Prefect Concepts](1-prefect-concepts.md) →
