# Build Your First Modern Data Stack

A step-by-step guide to building a production-grade data platform from scratch - the kind of architecture that handles real business data reliably, at scale, and without requiring a large team to maintain it.

By the end of this guide, you will have a fully working data stack: streaming events landing in your warehouse within seconds, batch pipelines running on a schedule, dbt models transforming raw data into clean analytics tables, and dashboards your team can actually use. Every piece of infrastructure is version-controlled, every pipeline is monitored, and every decision is explained.

---

## What You'll Build

The guide uses a sales analytics use case to make the stack concrete. You will build a platform that:

- Streams **purchase events** from a web app into Snowflake in real time via Kafka
- Loads **product catalogue data** from a PostgreSQL database on a schedule with dlt
- Fetches **exchange rate data** from a public API and stages it through S3
- Syncs **customer records** from HubSpot via Airbyte
- Transforms all four sources into clean dbt models - `fact_purchases`, `dim_products`, `fct_exchange_rates`, `dim_customers`
- Joins everything into a `sales` mart with per-customer revenue in GBP and USD
- Surfaces the results in a Lightdash dashboard with metrics defined in code

The use case is intentionally simple. The stack is not. You are building the real thing.

---

## The Stack

| Layer | Tool | Why |
|---|---|---|
| Streaming ingestion | Confluent Cloud (Kafka) | Managed Kafka with Schema Registry and Connect included |
| Batch ingestion - APIs and databases | dlt | Free, Python-native, handles schema inference automatically |
| Batch ingestion - SaaS | Airbyte | Hundreds of pre-built connectors, low-code configuration |
| Data warehouse | Snowflake | Separates compute from storage, excellent Terraform support |
| Transformation | dbt | SQL-native, version-controlled, tested models |
| BI and dashboards | Lightdash | Metrics defined in dbt YAML, not duplicated in the tool |
| Orchestration | Prefect | Modern Python-native replacement for Airflow |
| Observability | Elementary + OpenMetadata | dbt-native testing plus a full data catalogue |
| Infrastructure | Terraform | Everything as code - nothing created by hand |
| CI/CD | GitHub Actions | Terraform plans, dbt tests, and Prefect deployments |
| Secrets | AWS Secrets Manager | Centralised credential storage for all services |

Where a tool has a self-hosted alternative, both options are covered. You can choose based on your budget and how much infrastructure you want to manage.

---

## How This Guide Is Structured

```
Getting Started  →  Build  →  Maintain
```

**[Getting Started](getting-started/index.md)** covers everything you need before writing any infrastructure code. You will set up your GitHub organisation, configure your local development environment, create accounts with AWS, Snowflake, and Prefect, and get Terraform running with remote state and CI/CD. This section is a prerequisite for everything that follows.

**[Build](build/index.md)** walks through each layer of the stack in dependency order - from the data warehouse and object storage up through ingestion, transformation, analytics, observability, streaming, and documentation. Each section explains the concepts, guides you through the Terraform and configuration, and ends with a summary of what you have built.

**[Maintain](maintain/index.md)** covers day-to-day operations: adding users, adding data sources, handling backfills, responding to incidents, and keeping the stack up to date.

---

## What Makes This Guide Different

**It is production-grade from the start.** Infrastructure is managed with Terraform from day one. There are no "and now you would add monitoring in production" caveats - monitoring is built in as you go.

**It is opinionated, but explains why.** Every tool choice has a rationale. Where there are trade-offs between managed and self-hosted options, both are covered honestly.

**It is incremental.** Each section builds on the last. You end every section with something working, not a half-built system waiting for later chapters to make sense.

**It uses real patterns.** The Terraform modules and pipeline patterns in this guide are based on real client deployments, not toy examples. You can take them and adapt them directly.

**It includes AI-assisted workflows.** The [Maintain](maintain/index.md) section includes CLAUDE.md templates and Claude Code skills for your Terraform, dbt, and Prefect repositories - so an AI assistant can help with routine tasks like adding users or creating new data sources.

---

## Before You Start

This guide assumes you are comfortable with:

- Python (you will write dlt pipelines and Prefect flows)
- SQL (you will write dbt models)
- The command line
- Git and GitHub

You do not need prior experience with any of the specific tools. Each section introduces concepts before diving into implementation.

You will need accounts with GitHub, AWS, Snowflake, Prefect, and Confluent Cloud - we will cover how to set these up in the getting started section. The [Costs](getting-started/account-setup/costs.md) page breaks down what each service costs at the scale used in this guide.

---

## Get Started

[Begin with the Initial Setup →](getting-started/index.md)
