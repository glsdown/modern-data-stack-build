# Maintain Your Data Stack

You have built a complete, production-ready data stack spanning infrastructure, ingestion, transformation, analytics, and observability. This section covers how to keep it running, evolve it over time, and handle routine maintenance tasks - with or without AI agents.

## What You'll Learn

This section focuses on day-to-day operations - adding new resources, onboarding team members, and keeping your platform healthy:

- Adding new users, data sources, and models
- Backfill strategies and performance optimisation
- Disaster recovery and security hardening
- Troubleshooting common issues

!!! tip "Claude Code Setup"
    If you haven't already, set up CLAUDE.md files and skills for your repositories so Claude Code can assist with these maintenance tasks. See the [Claude Code Setup](../getting-started/initial-setup/5-claude-code-setup.md) page in the Getting Started section.

## Your Three Repositories

By now you have three repositories that work together:

```
GitHub Organisation
├── terraform/           Infrastructure as code
│   ├── github/          GitHub organisation, teams, users
│   ├── aws/             S3, IAM, VPC, Secrets Manager
│   └── snowflake/       Warehouses, databases, roles, users
│
├── data-pipelines/      Ingestion and orchestration
│   ├── sources/         dlt source definitions
│   ├── pipelines/       dlt pipeline configurations
│   └── flows/           Prefect flow definitions
│
└── dbt-transform/       Data transformation
    └── models/
        ├── staging/     Clean raw data (views)
        ├── intermediate/ Business logic (ephemeral)
        ├── marts/       Analytics tables (tables/incremental)
        └── reporting/   BI-facing subset (views)
```

Each repository has its own conventions, module patterns, and safety rules. Maintaining them means understanding these patterns - or having an AI agent that already does.

## Section Overview

!!! note "Coming Soon"
    This section is being built. Future pages will cover adding users, adding data sources, backfill strategies, performance optimisation, disaster recovery, security hardening, and troubleshooting.

## Prerequisites

Before starting this section, ensure you have completed:

- [x] [Data Warehouse](../build/data-warehouse/index.md) - Terraform modules for Snowflake resources
- [x] [Orchestration](../build/orchestration/index.md) - Prefect flows and deployments
- [x] [Batch Data Ingestion](../build/batch-data-ingestion/index.md) - dlt pipelines
- [x] At least one of: [SaaS Ingestion](../build/saas-ingestion/index.md), [Data Transformation](../build/data-transformation/index.md), or [Streaming](../build/streaming-data-ingestion/index.md)
