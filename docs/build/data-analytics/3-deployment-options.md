# Deployment Options

On this page, you will:

- [x] Understand the trade-offs between SaaS and self-hosted BI tools
- [x] Learn the true cost of self-hosting (infrastructure + engineering time)
- [x] Know when to choose cloud SaaS versus self-hosted deployment
- [x] Understand security and compliance considerations for each approach

## Overview

BI tools typically offer two deployment models: **cloud SaaS** (vendor-hosted) and **self-hosted** (you manage infrastructure). Each has distinct trade-offs in cost, control, maintenance, and complexity.

This page helps you understand which deployment model fits your organisation's capabilities, budget, and requirements.

## Deployment Models Explained

### Cloud SaaS (Software as a Service)

The BI tool vendor hosts and manages the application. You sign up, connect your data warehouse, and start building dashboards. The vendor handles servers, updates, backups, and scaling.

**Examples:**
- Tableau Online (vs Tableau Server)
- Lightdash Cloud (vs Lightdash self-hosted)
- Metabase Cloud (vs Metabase self-hosted)
- Power BI Service (vs Power BI Report Server)
- Looker (cloud-only, no self-hosted option)
- Omni (cloud-only)

### Self-Hosted (On-Premises or Your Cloud)

You deploy the BI tool to infrastructure you control — AWS, GCP, Azure, or on-premises servers. You're responsible for updates, backups, scaling, and maintenance.

**Examples:**
- Lightdash (Docker, ECS, Kubernetes)
- Metabase (Docker, ECS, Kubernetes)
- Apache Superset (Docker, Kubernetes)
- Tableau Server
- Power BI Report Server

!!! info "Hybrid Models"
    Some tools (Hex, Mode, dbt Cloud) are cloud-only with no self-hosted option. Others (Snowsight) are embedded in the platform and don't have separate deployment options.

## The SaaS vs Self-Hosted Trade-Off

| Factor | Cloud SaaS | Self-Hosted |
|--------|-----------|-------------|
| **Setup time** | Minutes to hours | Hours to days |
| **Infrastructure cost** | Included in price | $20-100+/month (your cloud bill) |
| **Maintenance** | Vendor manages | You manage (4-8 hours/month) |
| **Updates** | Automatic | Manual (test, deploy, rollback plan) |
| **Scalability** | Automatic | Manual (resize instances, add nodes) |
| **Data residency** | Vendor's cloud | Your cloud/data centre |
| **Customisation** | Limited | Full control (within application limits) |
| **Total cost** | Higher monthly fee | Lower fee + infrastructure + eng time |
| **Uptime SLA** | Vendor SLA (99.9%+) | Your responsibility |
| **Security** | Vendor's security + your auth | Your full responsibility |

Neither option is universally better — the right choice depends on your organisation's priorities and capabilities.

## Cost Comparison: The Full Picture

### Cloud SaaS Costs

Cloud SaaS pricing is straightforward: a monthly fee per user or flat rate.

**Example: Lightdash Cloud**
- $2400/month flat rate (unlimited users)
- No infrastructure costs
- No engineering time for maintenance
- **Total: $2400/month**

**Example: Omni**
- $20-50/user/month
- 10 users = $200-500/month
- No infrastructure costs
- No engineering time
- **Total: $200-500/month**

**Example: Tableau Online**
- $70/user/month (Creator), $42/user/month (Explorer)
- 5 Creators + 20 Explorers = $350 + $840 = $1190/month
- No infrastructure costs
- No engineering time
- **Total: $1190/month**

### Self-Hosted Costs

Self-hosted costs are multi-layered: infrastructure + engineering time + potential downtime.

**Example: Lightdash Self-Hosted**

**Infrastructure (AWS ECS + RDS):**
- ECS Fargate: 1 vCPU, 2GB RAM = ~$15/month
- RDS PostgreSQL: db.t3.micro = ~$15/month
- Application Load Balancer = ~$20/month
- **Subtotal: ~$50/month**

**Engineering Time:**
- Initial setup: 4-8 hours (one-time)
- Monthly maintenance: 2-4 hours (updates, monitoring)
- Quarterly upgrade: 4 hours
- **Average: ~4 hours/month**

At $100/hour engineering cost:
- **Subtotal: ~$400/month in engineering time**

**Total Self-Hosted Cost: ~$450/month**

This is **cheaper** than Lightdash Cloud ($2400/month) if you have fewer than 10-15 users but **more expensive** than you might expect from the "$0" open source price tag.

!!! warning "Engineering Time Is Real Cost"
    Self-hosted tools appear free, but engineering time has opportunity cost. Those 4 hours/month could be spent building pipelines, models, or dashboards instead of updating infrastructure.

### The Break-Even Point

For most tools, self-hosted becomes cheaper than SaaS once you cross a certain team size:

**Lightdash:**
- Self-hosted: ~$450/month (infrastructure + eng time)
- Cloud: $2400/month (flat rate, unlimited users)
- **Break-even: Always self-host if cost-conscious** (unless you value zero-maintenance)

**Metabase:**
- Self-hosted: ~$400/month (infrastructure + eng time)
- Cloud: $85/user/month
- 5 users: $425/month (SaaS) vs $400/month (self-hosted) → **roughly equivalent**
- 10 users: $850/month (SaaS) vs $400/month (self-hosted) → **self-hosted saves $450/month**
- **Break-even: ~5 users**

**Omni vs Lightdash Self-Hosted:**
- Omni Cloud: $60-150/month (3 users)
- Lightdash Self-Hosted: ~$450/month
- **Break-even: Omni is cheaper until ~5-7 users** (if you value low maintenance)

## Infrastructure Requirements for Self-Hosted

### Minimum Viable Setup

Most self-hosted BI tools require:

1. **Application server** (Docker container, ECS task, Kubernetes pod)
   - 1-2 vCPUs, 2-4GB RAM
   - Cost: ~$15-30/month (AWS ECS Fargate, t3.small EC2)

2. **Database** (PostgreSQL or MySQL for metadata)
   - Stores dashboards, users, queries, cached results
   - db.t3.micro (1 vCPU, 1GB RAM) sufficient for small teams
   - Cost: ~$15-25/month (RDS)

3. **Load balancer** (for HTTPS termination)
   - Application Load Balancer (AWS ALB)
   - Cost: ~$20/month + data transfer

4. **Object storage** (optional, for cached query results)
   - S3 bucket
   - Cost: ~$1-5/month (minimal for BI tool caching)

**Total: $50-80/month** for basic self-hosted BI infrastructure.

### Production-Grade Setup (Larger Teams)

For teams with 50+ users or high query volumes:

1. **Auto-scaling application servers**
   - 2-4 ECS tasks or Kubernetes pods
   - Cost: ~$60-120/month

2. **Database with read replicas**
   - Primary + 1 read replica
   - Cost: ~$40-80/month

3. **Redis for caching**
   - Elasticache Redis (cache.t3.micro)
   - Cost: ~$15/month

4. **CloudWatch monitoring and logs**
   - Cost: ~$10-20/month

**Total: $125-235/month** for production-grade infrastructure.

Add 6-10 hours/month engineering time for a larger deployment.

## Maintenance Overhead

### Cloud SaaS: Vendor-Managed

With SaaS, the vendor handles:
- ✅ Software updates and security patches
- ✅ Database backups and disaster recovery
- ✅ Scaling for increased usage
- ✅ Monitoring and alerting
- ✅ Uptime SLAs (99.9%+)

**Your responsibilities:**
- User management (SSO, roles, permissions)
- Dashboard creation and maintenance
- Query performance optimisation (warehouse-side)

**Time investment: ~1-2 hours/month**

### Self-Hosted: You Manage Everything

With self-hosted, you're responsible for:

**Monthly (2-4 hours):**
- Monitoring application health and logs
- Reviewing performance metrics
- User support (login issues, access problems)

**Quarterly (4-6 hours):**
- Upgrading to new releases (test, deploy, verify)
- Database maintenance (vacuum, reindex if needed)
- Reviewing and rotating secrets/credentials

**Annually (8-12 hours):**
- Infrastructure review and right-sizing
- Disaster recovery testing
- Security audit and updates

**Ad-hoc (varies):**
- Scaling for increased load
- Debugging issues and downtime
- Restoring from backups

**Total: ~6-10 hours/month average**

!!! tip "Reduce Maintenance with Automation"
    Use Terraform for infrastructure-as-code, automated backups (RDS), and container health checks (ECS). This reduces manual work but still requires initial setup and monitoring.

## Security and Compliance

### Data Residency and Access

**Cloud SaaS:**
- BI tool hosted in vendor's cloud (typically AWS, GCP, or Azure)
- Vendor has access to metadata (dashboard definitions, user queries, cached results)
- Your data warehouse data is **not** copied to the BI tool (it's queried on-demand)
- Most tools cache query results (aggregated, not raw data)

**Self-Hosted:**
- BI tool hosted in your cloud or data centre
- You control all data and metadata
- No third-party access (unless you grant it)
- Full control over data residency and compliance

!!! info "How BI Tools Access Your Data"
    Both SaaS and self-hosted tools query your warehouse on-demand. They don't copy your raw data. Some tools cache **query results** (e.g., "sum of sales by region") to improve performance, but this is aggregated data, not raw customer records.

### Compliance Considerations

| Requirement | Cloud SaaS | Self-Hosted |
|-------------|-----------|-------------|
| **GDPR** | Vendor must be GDPR-compliant | Your responsibility |
| **SOC 2** | Check vendor's SOC 2 report | Your audit scope |
| **HIPAA** | Vendor must sign BAA | Your responsibility |
| **Data residency (EU-only)** | Vendor must offer EU hosting | Deploy in EU region |
| **Customer data isolation** | Multi-tenant (shared infra) | Single-tenant (your infra) |

**For regulated industries** (healthcare, finance): Self-hosted is often required or strongly preferred for compliance and audit reasons.

**For most teams**: SaaS is compliant enough. Check vendor's security page and SOC 2 report.

### Authentication and Access Control

**Cloud SaaS:**
- Managed SSO integrations (Okta, Azure AD, Google Workspace)
- Vendor-managed user authentication
- You control authorisation (who sees which dashboards)

**Self-Hosted:**
- You configure SSO (requires SAML setup, testing)
- You manage authentication infrastructure
- Full control over access policies

## When to Choose Cloud SaaS

Choose cloud SaaS if:

- ✅ **Small engineering team** — no dedicated DevOps/infrastructure resources
- ✅ **Limited time** — you'd rather build dashboards than manage infrastructure
- ✅ **Budget allows** — $200-2400/month for BI tool is acceptable
- ✅ **Compliance permits** — no strict data residency requirements
- ✅ **Prefer simplicity** — value vendor-managed updates and zero maintenance
- ✅ **Fast time-to-value** — need BI running in hours, not days

**Best SaaS options:**
- **Omni** ($60-300/month for small teams, dbt-native)
- **Power BI** ($100-500/month for medium teams, Microsoft ecosystem)
- **Lightdash Cloud** ($2400/month for larger teams, unlimited users)

## When to Choose Self-Hosted

Choose self-hosted if:

- ✅ **Budget-constrained** — willing to invest engineering time to save money
- ✅ **DevOps expertise** — team already manages self-hosted services (Prefect, Airflow)
- ✅ **Data residency required** — compliance mandates data stays in specific regions
- ✅ **Customisation needed** — want full control over deployment and configuration
- ✅ **Large team** — 20+ users where SaaS per-user costs are prohibitive
- ✅ **Infrastructure-as-code culture** — everything Terraform-managed already

**Best self-hosted options:**
- **Lightdash** (dbt-native, Terraform-friendly, ~$50/month infra)
- **Metabase** (user-friendly, mature, ~$50/month infra)
- **Superset** (advanced viz, Apache project, ~$60/month infra)

## Hybrid Approach: Start SaaS, Move to Self-Hosted Later

Some teams start with SaaS for speed, then migrate to self-hosted as they scale:

**Phase 1: SaaS (Months 0-6)**
- Fast setup (hours, not days)
- Learn what you actually need
- Get stakeholder buy-in with quick wins
- Evaluate usage patterns

**Phase 2: Migrate to Self-Hosted (Month 6+)**
- Once you have 10+ users or predictable usage
- When engineering team has capacity
- When budget requires cost optimisation
- Export dashboards and redeploy to self-hosted

**Works well for:**
- Omni → Lightdash self-hosted (both dbt-native, similar features)
- Metabase Cloud → Metabase self-hosted (same product, different hosting)

**Harder migrations:**
- Tableau → Lightdash (different paradigms, rebuild dashboards)
- Power BI → Superset (different data models, significant rework)

## Decision Matrix

Use this matrix to choose your deployment approach:

|  | Cloud SaaS | Self-Hosted |
|--|------------|-------------|
| **Team size** | < 10 people | 5+ with DevOps skills |
| **Budget** | $200-2400/month | $50-100/month (infra only) |
| **Engineering time available** | Limited (< 5 hours/month) | 6-10 hours/month |
| **Compliance** | Standard (GDPR, SOC 2) | Strict (HIPAA, PCI-DSS, data residency) |
| **Infrastructure expertise** | Limited | Strong (Docker, ECS, Terraform) |
| **Time to value** | Immediate (hours) | Days to weeks |
| **Vendor lock-in concern** | Low (can export dashboards) | None (you control everything) |

## Cost Projection Example

Let's model Lightdash costs for a growing team over 2 years:

| Team Size | Cloud SaaS (Lightdash Cloud) | Self-Hosted (Lightdash) |
|-----------|----------------------------|------------------------|
| **5 users** | $2400/month | $450/month (infra + 4 hrs eng time) |
| **10 users** | $2400/month | $450/month (same) |
| **20 users** | $2400/month | $550/month (upgraded infra, 6 hrs eng time) |
| **50 users** | $2400/month | $700/month (production setup, 8 hrs eng time) |

**Break-even: Never** (self-hosted always cheaper for Lightdash due to flat-rate SaaS pricing)

But if engineering time is $150/hour (senior engineer):
- Self-hosted at 5 users: $450 + ($150 × 4 hrs) = **$1050/month**
- **Break-even: ~15-20 users** when factoring expensive engineering time

## Summary

You've learned about deployment options:

- [x] **Cloud SaaS** costs more but requires zero infrastructure management (ideal for small teams)
- [x] **Self-hosted** costs less in subscription fees but requires infrastructure ($50-100/month) and engineering time (6-10 hours/month)
- [x] **True cost of self-hosting** includes infrastructure, engineering time (opportunity cost), and maintenance overhead
- [x] **Security and compliance** may require self-hosted for regulated industries or data residency requirements
- [x] **Hybrid approach** (start SaaS, migrate to self-hosted) works well for cost optimisation as you scale

For most teams following this documentation (small, budget-conscious, DevOps-capable), **self-hosted Lightdash** offers the best value. For teams wanting zero maintenance, **Omni** (cloud SaaS, dbt-native) is worth the premium.

## What's Next

Now that you understand deployment trade-offs, start with Snowflake's built-in dashboarding for a quick win before deploying a dedicated BI tool.

Continue to [Snowflake Snowsight](4-snowflake-snowsight.md) →
