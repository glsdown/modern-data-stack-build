# Choosing a Deployment Option

On this page, you will:

- [x] Compare Prefect Cloud vs self-hosted options
- [x] Understand the cost breakdown for each option
- [x] Learn about total cost of ownership for self-hosted
- [x] Choose the right deployment for your needs

## Overview

Prefect offers flexibility in how you deploy the control plane. All options use the same Prefect concepts - flows, tasks, deployments, work pools. The difference is where the server (API, database, UI) runs.

| Component | Prefect Cloud | Self-Hosted |
|-----------|---------------|-------------|
| **Server/API** | Managed by Prefect | You manage |
| **Database** | Managed PostgreSQL | You provide |
| **UI** | app.prefect.cloud | Self-hosted UI |
| **Workers** | You run | You run |
| **Flow code** | You run | You run |

In all cases, **you run the workers and flow code** on your own infrastructure. The only difference is where the orchestration control plane lives.

## Option Comparison

### At a Glance

| Aspect | Prefect Cloud | Docker Compose | ECS + RDS |
|--------|---------------|----------------|-----------|
| **Monthly cost** | $0 - $100+ | ~$17 | ~$67 |
| **Operational burden** | None | Medium | Low-Medium |
| **High availability** | Yes | No | Yes |
| **Data sovereignty** | US/EU (Prefect servers) | Full control | Full control |
| **Terraform support** | Yes (provider) | Partial | Yes |
| **Best for** | Most teams | Budget, sovereignty | Production HA |

### Prefect Cloud

**What you get:**

- Fully managed server, database, and UI
- Zero infrastructure to maintain
- Built-in authentication (SSO, service accounts)
- Collaboration features (workspaces, teams)
- 99.9% uptime SLA (Enterprise tier)

**What you manage:**

- Workers (run on your infrastructure)
- Flow code (your responsibility)
- Secrets (AWS Secrets Manager recommended)

**When to choose:**

- You want to get started quickly
- You don't want operational overhead
- Your team is small to medium sized
- Data sovereignty isn't a hard requirement

### Docker Compose (Self-Hosted)

**What you get:**

- Full control over the Prefect server
- Data stays in your infrastructure
- Lowest cost option
- Simple to understand and debug

**What you manage:**

- EC2 instance (or equivalent)
- Docker containers (server, database, worker)
- Backups and disaster recovery
- Upgrades and patching

**When to choose:**

- Budget is a primary concern
- Data sovereignty is required
- You're comfortable with Linux/Docker administration
- High availability isn't critical

### ECS + RDS (Self-Hosted Production)

**What you get:**

- AWS-managed container orchestration
- AWS-managed database with backups
- Auto-scaling capabilities
- High availability across AZs

**What you manage:**

- Terraform configuration
- ECS task definitions
- Networking and security groups
- Monitoring and alerting

**When to choose:**

- You need production-grade reliability
- Data sovereignty is required
- You have AWS expertise on the team
- You need auto-scaling for workers

## Detailed Cost Breakdown

### Prefect Cloud Pricing

Prefect Cloud offers tiered pricing based on team size and features:

| Tier | Monthly Cost | Users | Deployments | Key Features |
|------|-------------|-------|-------------|--------------|
| **Hobby** | Free | 2 | 5 | Basic orchestration, 7-day retention |
| **Starter** | $100 | 3 | 20 | Webhooks, bring your own compute |
| **Team** | $100/user | 4-8 | 100 | Service accounts, audit logs |
| **Pro** | Custom | 5-20 | 1,000 | Multiple workspaces, SSO |
| **Enterprise** | Custom | Unlimited | Unlimited | RBAC, SLA, PrivateLink |

!!! tip "Start with Hobby"
    The free Hobby tier is sufficient for getting started and learning. You can upgrade as your needs grow.

**Additional costs with Prefect Cloud:**

- Workers still run on your infrastructure (EC2, ECS, etc.)
- Estimate $15-50/month for worker compute depending on workload

### Docker Compose Costs

Running Prefect on a single EC2 instance with Docker Compose:

| Component | Specification | Monthly Cost |
|-----------|--------------|--------------|
| EC2 instance | t3.small (2 vCPU, 2GB RAM) | ~$15 |
| EBS storage | 20GB gp3 | ~$2 |
| **Total infrastructure** | | **~$17/month** |

!!! note "Instance Sizing"
    A t3.small is sufficient for small to medium workloads. For heavier usage, consider t3.medium (~$30/month) or t3.large (~$60/month).

**What's included:**

- Prefect server (API)
- PostgreSQL database
- Prefect UI
- Optionally, a worker process

### ECS + RDS Costs

Running Prefect on AWS ECS with RDS PostgreSQL:

| Component | Specification | Monthly Cost |
|-----------|--------------|--------------|
| ECS Fargate (server) | 0.5 vCPU, 1GB RAM, 24/7 | ~$18 |
| ECS Fargate (worker) | 0.5 vCPU, 1GB RAM, 24/7 | ~$18 |
| RDS PostgreSQL | db.t3.micro, 20GB | ~$15 |
| Application Load Balancer | Basic usage | ~$16 |
| **Total infrastructure** | | **~$67/month** |

!!! note "Worker Scaling"
    Worker costs can be reduced by scaling to zero when idle, or increased by adding more workers for parallel execution.

**Fargate pricing breakdown:**

- vCPU: $0.04048/hour = ~$29/month per vCPU
- Memory: $0.004445/hour per GB = ~$3.20/month per GB
- 0.5 vCPU + 1GB = ~$18/month running 24/7

## Total Cost of Ownership

Infrastructure costs are only part of the picture. Self-hosted options require ongoing operational effort.

### Operational Tasks (Self-Hosted)

| Task | Frequency | Estimated Time |
|------|-----------|----------------|
| Monitoring and troubleshooting | Weekly | 30-60 min |
| Prefect version upgrades | Monthly | 1-2 hours |
| Security patches (OS, Docker) | Monthly | 30-60 min |
| Database maintenance | Monthly | 30 min |
| Backup verification | Monthly | 15 min |
| Incident response | As needed | Variable |

**Estimated operational overhead:**

- **Docker Compose**: 2-4 hours/month
- **ECS + RDS**: 1-2 hours/month (more managed)

### TCO Comparison (12 months)

Assuming an engineer cost of $75/hour:

| Option | Infrastructure | Operations (12mo) | Total (Year 1) |
|--------|---------------|-------------------|----------------|
| Prefect Cloud (Hobby) | $0 + ~$30 workers | $0 | ~$360 |
| Prefect Cloud (Starter) | $1,200 + ~$360 workers | $0 | ~$1,560 |
| Docker Compose | ~$200 | ~$2,700 (36 hrs) | ~$2,900 |
| ECS + RDS | ~$800 | ~$1,350 (18 hrs) | ~$2,150 |

!!! warning "Hidden Costs"
    Self-hosted TCO calculations often underestimate:

    - Initial setup time (8-16 hours for ECS)
    - Learning curve for team members
    - Incident response during critical times
    - Opportunity cost of not building features

### When Self-Hosting Makes Sense

Self-hosting becomes more cost-effective when:

1. **Scale** - You have many deployments and workers
2. **Expertise** - Your team already manages AWS/Docker infrastructure
3. **Requirements** - Data sovereignty or compliance mandates it
4. **Existing infrastructure** - You can share EC2/ECS capacity with other workloads

## Decision Matrix

Use this matrix to guide your decision:

| If you... | Choose |
|-----------|--------|
| Are getting started | Prefect Cloud (Hobby) |
| Have a small team (< 5) with basic needs | Prefect Cloud (Hobby or Starter) |
| Need multiple workspaces or SSO | Prefect Cloud (Pro or Enterprise) |
| Have strict data sovereignty requirements | Self-hosted |
| Are budget-constrained but have ops capacity | Docker Compose |
| Need production HA and have AWS expertise | ECS + RDS |
| Want to minimise operational burden | Prefect Cloud |

## Our Recommendation

For most teams, we recommend:

1. **Start with Prefect Cloud (Hobby tier)**
   - Get familiar with Prefect concepts
   - Build your first flows and deployments
   - Validate the tool fits your needs

2. **Upgrade as needed**
   - Move to Starter/Team if you need more users or deployments
   - Migrate to self-hosted if requirements change

3. **Consider self-hosting when**
   - You've outgrown Cloud pricing
   - Data sovereignty becomes a hard requirement
   - You have dedicated platform/infrastructure engineers

!!! tip "Migration is Straightforward"
    Prefect makes it straightforward to migrate between deployment options. Your flows, tasks, and worker code remain the same — you point to a different API endpoint.

## Summary

You've learned:

- [x] The three deployment options and their trade-offs
- [x] Detailed cost breakdowns for each option
- [x] Total cost of ownership including operational overhead
- [x] How to choose the right option for your needs

## What's Next

Based on your choice:

- **Prefect Cloud** → Continue to [Prefect Cloud Setup](3-prefect-cloud-setup.md)
- **Docker Compose** → Skip to [Docker Compose Setup](4-self-hosted-docker-compose.md)
- **ECS + RDS** → Skip to [ECS Production Setup](5-self-hosted-ecs-production.md)

If you're unsure, we recommend starting with Prefect Cloud - you can always migrate later.

Continue to [Prefect Cloud Setup](3-prefect-cloud-setup.md) →
