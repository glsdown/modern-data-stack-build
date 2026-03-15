# Observability

On this page, you will:

- [x] Understand what data observability means and why it matters
- [x] Learn the three pillars: data quality, pipeline health, and cost management
- [x] Survey the observability tools covered in this section
- [x] Plan your observability stack based on budget and team size

## Overview

You've built a modern data stack: data flows from sources through pipelines, transformations, and into dashboards. But **how do you know it's working correctly?** How do you catch bad data before it reaches executives? How do you debug a failed pipeline at 2am? How do you prevent Snowflake bills from spiralling out of control?

This is where **observability** comes in. Observability provides visibility into your data platform's health, quality, and performance. It helps you answer:

- **Data quality**: Is the data accurate, complete, and timely?
- **Pipeline health**: Are pipelines running successfully? What broke and why?
- **Cost management**: Where is money being spent? Are costs predictable?

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       OBSERVABILITY LAYER                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Data Quality          Pipeline Health         Cost Management         │
│  ────────────          ───────────────         ────────────────        │
│                                                                         │
│  ┌──────────────┐      ┌──────────────┐       ┌──────────────┐        │
│  │ dbt tests    │      │ Prefect      │       │ Snowflake    │        │
│  │ Elementary   │      │ monitoring   │       │ Resource     │        │
│  │ Great        │      │ Flow states  │       │ Monitors     │        │
│  │ Expectations │      │ Alerts       │       │ Cost by      │        │
│  └──────────────┘      └──────────────┘       │ warehouse    │        │
│        │                      │               └──────────────┘        │
│        │                      │                      │                │
│        └──────────────────────┴──────────────────────┘                │
│                               │                                       │
│                               ▼                                       │
│                    ┌────────────────────────┐                         │
│                    │  OpenMetadata          │                         │
│                    │  (Unified Catalog)     │                         │
│                    │  • Lineage             │                         │
│                    │  • Documentation       │                         │
│                    │  • Quality metrics     │                         │
│                    │  • Usage analytics     │                         │
│                    └────────────────────────┘                         │
│                                                                         │
│  Alerts sent to Slack, PagerDuty, email for immediate action.          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## The Three Pillars of Data Observability

### 1. Data Quality

**Question:** Is the data correct?

Data quality ensures that the data in your warehouse is accurate, complete, consistent, timely, and valid. Poor data quality leads to wrong business decisions, lost trust, and wasted analyst time.

**Tools covered:**
- **dbt tests** — Built-in data validation (unique, not_null, accepted_values, relationships)
- **dbt_expectations** — Statistical tests (distribution checks, anomaly detection)
- **Elementary** — Automated anomaly detection and test result tracking
- **Great Expectations** — Python-based expectation suites for advanced validation

**Example quality checks:**
- Row counts match expected ranges (not 10x higher or lower than yesterday)
- No null values in critical columns (customer_id, order_date)
- Foreign keys are valid (all order.customer_id exist in customers table)
- Distributions are stable (average order value within expected range)

### 2. Pipeline Health

**Question:** Are pipelines running successfully?

Pipeline health monitors data ingestion, transformation, and orchestration. When a pipeline fails, you need to know immediately, understand what broke, and fix it quickly.

**Tools covered:**
- **Prefect monitoring** — Flow run states, task retries, error tracking
- **Snowflake query monitoring** — Query performance, warehouse utilisation
- **CloudWatch Logs** — Centralized logging for ECS services (Airbyte, Lightdash)
- **Alerting** — Slack, PagerDuty, email notifications for failures

**Example health checks:**
- dbt daily run completed successfully
- Airbyte sync finished within expected duration
- No failed Prefect tasks in the last 24 hours
- Snowflake warehouse auto-resumed when queries started

### 3. Cost Management

**Question:** Where is money being spent?

Data platforms can be expensive. Snowflake credits, AWS infrastructure, and third-party tools add up. Cost observability tracks spending, attributes costs to teams, and prevents budget overruns.

**Tools covered:**
- **Snowflake Resource Monitors** — Budget alerts, auto-suspend on credit limits
- **AWS Cost Explorer** — Infrastructure costs (S3, ECS, RDS, data transfer)
- **Query tagging** — Attribute Snowflake costs to teams, projects, or users
- **Cost allocation tags** — Track Terraform-managed resources

**Example cost monitoring:**
- Daily Snowflake credit usage by warehouse
- Alert when monthly spend exceeds budget by 20%
- Identify most expensive queries (top 10 by cost)
- Forecast next month's costs based on trends

## What You Will Build

By the end of this section:

```
Observability Stack
├── Data Quality
│   ├── dbt tests (schema.yml files with generic + custom tests)
│   ├── dbt_expectations package (statistical validation)
│   └── Elementary (anomaly detection, Slack alerts)
│       ├── Elementary dbt package (test results tracking)
│       ├── Elementary CLI (anomaly detection)
│       └── Elementary UI (self-hosted or Elementary Cloud)
│
├── Data Catalog
│   └── OpenMetadata (self-hosted on ECS)
│       ├── Snowflake connector (metadata extraction)
│       ├── dbt connector (lineage from manifest.json)
│       ├── Prefect connector (pipeline metadata)
│       └── UI (search, lineage, data quality dashboard)
│
├── Pipeline Monitoring
│   ├── Prefect Cloud UI (flow runs, states, logs)
│   ├── Prefect automations (retry on failure, Slack alerts)
│   └── Snowflake query history (performance profiling)
│
├── Cost Monitoring
│   ├── Snowflake Resource Monitors (budget alerts)
│   ├── AWS Cost Explorer (infrastructure spend)
│   └── Cost allocation tags (Terraform-managed resources)
│
└── Logging & Debugging
    ├── CloudWatch Logs (ECS services: Prefect, Airbyte, Lightdash)
    ├── Snowflake query logs (QUERY_HISTORY view)
    └── dbt logs (run_results.json, compiled SQL)
```

**Service accounts:**
- `SVC_ELEMENTARY` — Read dbt run results, write anomaly detections
- `SVC_OPENMETADATA` — Read Snowflake ACCOUNT_USAGE for metadata extraction

**Infrastructure:**
- Elementary UI: ECS Fargate (~$15/month) + RDS PostgreSQL (~$15/month)
- OpenMetadata: ECS Fargate (~$30/month) + RDS PostgreSQL (~$20/month)
- Total: ~$80/month for full observability stack

## Section Contents

| Page | What You Will Do |
|------|------------------|
| [Observability Fundamentals](1-observability-fundamentals.md) | Core concepts, SLIs/SLOs/SLAs, observability vs monitoring |
| [Data Quality with dbt](2-data-quality-with-dbt.md) | dbt tests deep dive, dbt_expectations, custom tests, CI/CD |
| [Elementary Setup](3-elementary-setup.md) | Install Elementary package, CLI, UI; configure anomaly detection |
| [Data Cataloging with OpenMetadata](4-data-cataloging.md) | Deploy OpenMetadata, connect to Snowflake/dbt/Prefect |
| [Data Lineage](5-data-lineage.md) | Column-level lineage, impact analysis, debugging with lineage |
| [Prefect Monitoring](6-prefect-monitoring.md) | Flow states, notifications, automation rules, custom metrics |
| [Snowflake Monitoring](7-snowflake-monitoring.md) | Query profiling, warehouse sizing, Resource Monitors |
| [Cost Monitoring](8-cost-monitoring.md) | Credit usage, cost attribution, budget alerts, chargeback |
| [Alerting and Incidents](9-alerting-and-incidents.md) | Alert routing, runbooks, incident response, on-call |
| [Logging and Debugging](10-logging-and-debugging.md) | CloudWatch Logs, log aggregation, debugging workflows |
| [Anomaly Detection](11-anomaly-detection.md) | Elementary anomaly detection, Great Expectations, custom checks |
| [Finishing Up](12-finishing-up.md) | Observability maturity model, when to upgrade, next steps |

## Prerequisites

Before starting this section, ensure you have completed:

- [x] [Data Warehouse](../data-warehouse/index.md) — Snowflake with role-based access control
- [x] [Orchestration](../orchestration/index.md) — Prefect running data pipelines
- [x] [Data Transformation](../data-transformation/index.md) — dbt models with tests
- [x] [Data Analytics](../data-analytics/index.md) — Dashboards and notebooks

The observability layer monitors the stack you've already built. You need working pipelines before you can observe them.

## Deployment Options Overview

### Budget Approach (~$80-100/month)

**What you get:**
- Elementary self-hosted (anomaly detection, Slack alerts)
- OpenMetadata self-hosted (data catalog, lineage)
- dbt tests (unlimited, free)
- Prefect monitoring (included with Prefect Cloud Free)
- Snowflake Resource Monitors (included with Snowflake)
- CloudWatch Logs (minimal cost, ~$5/month)

**Total infrastructure cost:** ~$80/month (ECS + RDS for Elementary + OpenMetadata)

**Best for:** Small teams (1-10 people), cost-conscious, comfortable with self-hosting

### Premium Approach ($1000+/month)

**What you get:**
- Elementary Cloud ($50+/month) or Monte Carlo ($$$, quote-based)
- Datadog ($15+/host/month for unified observability)
- Atlan (enterprise data catalog, quote-based)
- Great Expectations Cloud (managed validation)

**Total cost:** $1000-5000/month depending on tools chosen

**Best for:** Large teams (50+ people), enterprise compliance requirements, prefer managed services

### Minimal Approach ($0/month)

**What you get:**
- dbt tests only (no Elementary)
- Prefect Cloud Free monitoring (basic)
- Snowflake query history (manual review)
- CloudWatch Logs (view in AWS Console)
- Slack for manual alerts

**Total cost:** $0 (no additional infrastructure)

**Best for:** Very small teams (1-3 people), early-stage projects, minimal budget

This documentation focuses on the **budget approach** (Elementary + OpenMetadata self-hosted) with notes on upgrading to premium tools.

## Observability vs Monitoring vs Testing

These terms are often used interchangeably, but they have distinct meanings:

| Concept | Definition | Example |
|---------|------------|---------|
| **Testing** | Validating data against known rules at a point in time | dbt test: "customer_id must be unique" |
| **Monitoring** | Tracking metrics over time and alerting on thresholds | "Alert if dbt run duration > 10 minutes" |
| **Observability** | Understanding system state from outputs; asking new questions | "Why did revenue spike last Tuesday?" (lineage + logs + metrics) |

**Testing** is proactive (define rules upfront).
**Monitoring** is reactive (alert when rules are violated).
**Observability** is exploratory (investigate unknowns with full context).

You need all three:
- **dbt tests** (testing) ensure known rules are enforced
- **Elementary anomaly detection** (monitoring) catches unexpected changes
- **OpenMetadata lineage** (observability) helps debug root causes

## Cost Summary

| Component | Monthly Cost | Notes |
|-----------|-------------|-------|
| **dbt tests** | $0 | Built-in, runs in Snowflake compute |
| **dbt_expectations** | $0 | Open source dbt package |
| **Elementary (self-hosted)** | ~$30 | ECS + RDS for UI |
| **Elementary Cloud** | $50+ | Managed service, unlimited users |
| **OpenMetadata (self-hosted)** | ~$50 | ECS + RDS for backend + UI |
| **Atlan** | $$$ | Enterprise data catalog (quote-based) |
| **Prefect monitoring** | $0 | Included with Prefect Cloud Free |
| **Snowflake Resource Monitors** | $0 | Included with Snowflake |
| **CloudWatch Logs** | ~$5 | Log storage and queries |
| **Datadog** | $15+/host | Premium unified observability |
| **Monte Carlo** | $$$ | Commercial data observability (quote-based) |
| **Great Expectations Cloud** | $$$ | Managed validation (quote-based) |

**Budget build:** ~$85/month (Elementary + OpenMetadata self-hosted + CloudWatch)
**Premium build:** $1000-5000/month (managed services, enterprise tools)

## Why Observability Matters

### Without Observability

**Scenario:** Revenue dashboard shows a 50% drop on Tuesday.

**What happens:**
1. Analyst notices the drop on Wednesday morning
2. Analyst manually checks dbt models (is the query wrong?)
3. Analyst checks Snowflake (did the data load?)
4. Analyst checks Airbyte (did the sync fail?)
5. Analyst checks source system (is the API broken?)
6. **2 hours later:** Discovers Airbyte sync failed silently on Monday night
7. Re-run sync, dbt, dashboards update
8. **Total time to resolution:** 1 day

### With Observability

**Scenario:** Same revenue drop.

**What happens:**
1. Elementary anomaly detection alerts in Slack at 2am Tuesday: "Row count for `raw_orders` down 90% from 7-day average"
2. On-call engineer checks OpenMetadata lineage: `raw_orders` ← Airbyte ← Shopify API
3. Checks Prefect: Airbyte sync failed at 1:45am with API rate limit error
4. Re-runs Airbyte sync with rate limit handling
5. dbt automatically re-runs via Prefect automation
6. Revenue dashboard shows correct data by 8am
7. **Total time to resolution:** 6 hours (mostly automated)

**Impact:**
- Faster detection (2am vs 10am next day)
- Faster debugging (lineage shows root cause immediately)
- Automated remediation (Prefect retries)
- Less analyst time wasted (6 hours vs 1 day)

## Get Started

Start by understanding observability fundamentals and core concepts.

Continue to [Observability Fundamentals](1-observability-fundamentals.md) →
