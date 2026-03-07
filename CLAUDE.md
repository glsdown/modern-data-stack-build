# Production-Grade Data Stack Reference Summary

This document summarizes the recommended architecture, tools, and orchestration strategies for a modern, production-grade data stack for a small company. It is designed to be **pragmatic, incremental, and AI-friendly** for project documentation. It should be written in British English with a focus on clarity and learning.

There is a skill available to help with building documentation pages - `generate-docs`

---

## 1. Core Principles

* **Incremental build**: Build the stack step by step rather than all at once.
* **Managed vs self-managed options**: Provide flexibility depending on budget and operational requirements.
* **Orchestration-centric**: Central orchestration layer (Prefect) coordinates batch, streaming, and SaaS ingestion.
* **Observability & reliability**: Schema enforcement, logging, retries, and monitoring included.
* **Modularity**: Each layer can evolve independently.

---

## 2. Stack Components

### Streaming

* **Managed**: Confluent Cloud (Kafka + Connect + Schema Registry)
* **Self-Managed**: MSK or Redpanda + self-hosted Kafka Connect
* **Notes**: Prefect can orchestrate tasks triggered by Kafka events; schema contracts enforced via Schema Registry.

### Batch Ingestion

* **Core sources (databases, APIs)**: **dlt** pipelines
* **SaaS sources (Salesforce, Stripe, HubSpot, etc.)**: **Airbyte**
* **Orchestration**: Prefect tasks wrap dlt and Airbyte runs
* **Deployment options**: Managed cloud (Prefect Cloud, Airbyte Cloud) or self-hosted

### Storage

* **Raw Zone**: S3 (append-only, immutable)
* **Analytics Zone**: Snowflake on AWS
* **Notes**: Prefect orchestrates load pipelines; dlt enforces schemas; dbt handles downstream transformations

### Transformation

* **dbt**: Managed (dbt Cloud) or self-managed (dbt Core)
* **Responsibilities**: Business logic, testing, documentation, semantic layer

### Analytics / BI / ML

* **BI Tools**: Lightdash (dbt-native, Cloud or self-hosted)
* **Built-in**: Snowflake Snowsight for quick dashboards
* **Advanced analytics**: Python notebooks (Jupyter, Snowflake Notebooks, Hex)
* **ML Workflows**: Prefect orchestrates preprocessing pipelines

### Secrets & Security

* **AWS Secrets Manager**: Store credentials for Snowflake, Kafka, dlt, dbt, Airbyte
* **RBAC**: Snowflake roles, Kafka service accounts, GitHub CODEOWNERS

### AWS IAM Role Model

The project uses a three-role model for AWS access:

| Role | State File Access | Purpose |
|------|------------------|---------|
| **DataEngineerRole** | Read-only | Day-to-day data platform work (queries, pipelines, monitoring) |
| **InfrastructureAdminRole** | Read/Write | Local Terraform operations during setup/debugging |
| **TerraformGitHubActionsRole** | Read/Write | CI/CD pipeline (primary method for infrastructure changes) |

**Key principles**:
- DataEngineerRole has explicit **Deny** policies preventing writes to Terraform state files (S3) and lock table (DynamoDB)
- InfrastructureAdminRole is only used during initial import of existing resources
- After CI/CD is configured, all infrastructure changes go through pull requests and the TerraformGitHubActionsRole
- OIDC authentication eliminates long-lived credentials for GitHub Actions

**AWS CLI profiles**:
- `data-engineer` - assumes DataEngineerRole (default for data work)
- `infrastructure-admin` - assumes InfrastructureAdminRole (Terraform operations)
- `admin` - assumes AdminRole (account administration)

### AWS IAM Policy Patterns

**Use `aws_iam_policy_document` for all policies** - this is the Terraform-native approach:
- Validates at plan time, catching errors early
- References variables and resources directly (no hard-coded ARNs)
- No separate JSON template files - all policies inline
- Type-safe Terraform syntax

**Single-role assume policies** - one policy per role for flexible user assignment:
- `AssumeAdminRolePolicy`, `AssumeDataEngineerRolePolicy`, `AssumeInfrastructureAdminRolePolicy`
- Users get multiple policies attached based on needs (configured in `iam_users.auto.tfvars`)
- Policies reference actual IAM role resources (e.g. `aws_iam_role.admin.arn`)

**Secrets Manager** - containers managed by Terraform, values set via CLI:
- AWS section only imports existing `terraform/github-token` secret
- Snowflake credentials secret created in the Snowflake section (not prematurely in AWS)
- Pattern documented for adding new secrets without creating them before they're needed

### Snowflake Patterns

**Service account naming**: Use `SVC_` prefix for service accounts (e.g. `SVC_TERRAFORM`, `SVC_DBT`, `SVC_AIRBYTE`)

**Terraform authentication**: Key-pair authentication (not password) for service accounts - more secure and required for CI/CD

**Credentials storage**: Snowflake service account credentials stored in AWS Secrets Manager (`terraform/snowflake-credentials`)

### Metadata & Governance

* **OpenMetadata**: Tracks Kafka topics, Snowflake tables, dlt pipelines, dbt models
* **Data Quality**: dbt tests, dlt expectations, schema contracts
* **Observability**: Prefect UI for logging, retries, and alerts

### CI/CD & Infrastructure

* **GitHub + Actions**: Terraform, dbt, Prefect flows
* **Terraform**: Manage AWS, Snowflake, Confluent Cloud, secrets, networking
* **Environment separation**: dev / staging / prod with isolated state and deployments

---

## 3. Orchestration with Prefect

* Prefect replaces Airflow entirely
* Prefect tasks wrap **dlt pipelines** and **Airbyte syncs**
* Prefect tasks also orchestrate **dbt transformations**, BI refreshes, and ML preprocessing
* Supports **dynamic workflows, retries, logging, alerts, and event-driven triggers**

### Example ETL Flow in Prefect

```python
from prefect import flow, task
import dlt, requests, subprocess

@task
def run_airbyte_sync(): ...
@task
def run_dlt_pipeline(): ...
@task
def run_dbt_models(): ...

@flow
def etl_flow():
    run_airbyte_sync()
    run_dlt_pipeline()
    run_dbt_models()
```

* Centralized orchestration for **batch, streaming, and SaaS sources**
* Prefect handles **retries, logging, and alerts**

---

## 4. Build Order / Incremental Phases

1. **Getting Started**: GitHub organisation, local development environment setup (macOS)
2. **Foundations**: Terraform fundamentals, remote state management (S3/DynamoDB)
3. **Data Warehouse**: Snowflake setup via Terraform (warehouses, databases, roles, users, SSO)
4. **Core Ingestion**: dlt pipelines wrapped in Prefect tasks
5. **SaaS Ingestion**: Airbyte pipelines orchestrated by Prefect
6. **Transformations**: dbt tasks added to Prefect flows
7. **Streaming**: Kafka topics and Connect sinks
8. **Observability & Metadata**: OpenMetadata, dbt/dlt tests, Prefect dashboards
9. **BI / ML**: Dashboards and notebooks orchestrated via Prefect

---

## 5. Documentation Structure

The documentation follows an incremental learning path:

### Getting Started (`docs/getting-started/`)
- **Initial Setup** - GitHub organisation, local dev environment, development workflow, secrets management, Claude Code setup
- **Account Setup** - AWS account (with AdminRole, DataEngineerRole, InfrastructureAdminRole), Snowflake account, Prefect account, cost overview
- **Terraform Setup** - Remote state, local setup, repository structure, GitHub provider, CI/CD deployment

#### Terraform Setup Subfolders
- **terraform-setup/github/** - Import GitHub organisation, teams, users into Terraform
- **terraform-setup/aws/** - Import IAM roles, state infrastructure, users, budgets, Secrets Manager
- **terraform-setup/snowflake/** - Import admin user, configure Snowflake provider

**Key pattern**: All Terraform operations during initial import use the `infrastructure-admin` AWS profile. After CI/CD is configured, the TerraformGitHubActionsRole handles all infrastructure changes through GitHub Actions.

### Build Sections (`docs/build/`)

| Section | Directory | Pages | Description |
|---------|-----------|-------|-------------|
| AWS Infrastructure | `aws/` | 2 | S3 data lake, VPC networking |
| Data Warehouse | `data-warehouse/` | 10 | Snowflake setup via Terraform (warehouses, databases, roles, users, SSO) |
| Orchestration | `orchestration/` | 11 | Prefect Cloud + self-hosted, work pools, flows, CI/CD, alerting |
| Batch Data Ingestion | `batch-data-ingestion/` | 10 | dlt pipelines + Snowpipe + HubSpot |
| SaaS Ingestion | `saas-ingestion/` | 9 | Airbyte Cloud + self-hosted + reverse ETL |
| Data Transformation | `data-transformation/` | 12 | dbt Core + dbt Cloud, staging/intermediate/mart models |
| Data Analytics | `data-analytics/` | 12 | Lightdash + Snowsight + notebooks |
| Observability | `observability/` | 12 | Elementary, data cataloging, lineage, monitoring, alerting |
| Streaming Data Ingestion | `streaming-data-ingestion/` | 11 | Confluent Cloud Kafka, Connect, MSK, CDC |

Each build guide includes:
- Concept explanation (what and why)
- **Terraform code** where applicable (production-ready)
- **SQL reference** in tabs for Snowflake sections (for understanding what Terraform creates)
- Working examples from `repositories/`

### Maintain (`docs/maintain/`)
- Day-to-day operations, adding resources, using AI agents
- Template CLAUDE.md files and skills for the three repositories

---

## 6. Repository Structure

The documentation project and its example repositories:

```
documentation/
├── repositories/
│   ├── terraform/               # Git submodule - example Terraform repo
│   │   ├── CLAUDE.md            # Agent reference for Terraform repo
│   │   ├── .claude/skills/      # Agent skills (add-snowflake-user, add-data-source)
│   │   ├── github/              # GitHub provider config
│   │   └── terraform.tf         # Root config
│   └── setup_script/            # Setup utilities
└── docs/
    ├── getting-started/
    │   ├── initial-setup/       # GitHub, local env, workflow, secrets, Claude Code
    │   ├── account-setup/       # AWS, Snowflake, Prefect accounts, costs
    │   └── terraform-setup/     # Remote state, GitHub/AWS/Snowflake providers
    ├── build/
    │   ├── aws/                 # S3 data lake, VPC networking
    │   ├── data-warehouse/      # Snowflake via Terraform (10 pages)
    │   ├── orchestration/       # Prefect Cloud + self-hosted (11 pages)
    │   ├── batch-data-ingestion/ # dlt + Snowpipe (10 pages)
    │   ├── saas-ingestion/      # Airbyte Cloud + self-hosted (9 pages)
    │   ├── data-transformation/ # dbt Core + Cloud (12 pages)
    │   ├── data-analytics/      # Lightdash + Snowsight (12 pages)
    │   ├── observability/       # Elementary, monitoring (12 pages)
    │   └── streaming-data-ingestion/ # Confluent Kafka (11 pages)
    └── maintain/
        ├── index.md             # Section overview
        └── templates/           # CLAUDE.md and skill templates for dbt/Prefect repos
```

---

## 7. Terraform Approach

**Philosophy**: Production-ready, modular, infrastructure-as-code from day one

**Key characteristics**:
- **Modular design**: Reusable modules based on brooklyn-heights patterns
- **Multiple providers**: Separate Snowflake providers for ACCOUNTADMIN, SYSADMIN, SECURITYADMIN, USERADMIN
- **Functional roles**: Purpose-based roles (ANALYTICS_DEVELOPER, ANALYTICS_TRANSFORMER, etc.)
- **Service account patterns**: Dedicated workflows for data loaders, transformers, reporters
- **Network policies**: IP allowlisting built into user/service account setup
- **SSO integrations**: SAML2 setup for GSuite, Okta, Azure AD
- **Remote state**: S3 backend with DynamoDB locking
- **CI/CD ready**: GitHub Actions for validation and deployment
- **Educational tabs**: SQL equivalents shown in documentation for learning

**Module structure** (based on brooklyn-heights):
- Each module creates a complete resource with all associated permissions
- Example: `snowflake_database` module creates database + reader/writer database roles + grants
- Example: `snowflake_user` module creates user + dedicated warehouse + role grants

---

## 8. Key Benefits of This Stack

* **Unified orchestration** for batch, streaming, and SaaS pipelines
* **Schema enforcement** via dlt and dbt
* **Low-code SaaS ingestion** via Airbyte
* **Central observability**: Prefect, OpenMetadata, dbt tests
* **Flexible deployment**: Managed or self-hosted layers
* **Infrastructure as Code**: All infrastructure version-controlled and repeatable
* **Production-ready from start**: Modular Terraform patterns proven in real client deployments
* **Incremental, production-grade, maintainable** for small to medium teams

---

## 9. Optional Extensions

* Feature stores for ML
* Reverse ETL for operational analytics
* Additional SaaS connectors as needed
* Advanced monitoring via Datadog/Grafana
* Multi-cloud deployments
* Additional environments (staging, QA)

---

## 10. Style

* Should be written in British English
* Should be conversational in tone
* Use spaced hyphens ( - ) for parenthetical statements, not em dashes (—): "you'll need this - it's required" **NOT** "you'll need this—it's required"
* When writing e.g. don't include a following comma: "e.g." **NOT** "e.g.,"

---

This file serves as a **reference for other AI agents** or team members to understand the stack, its orchestration, documentation structure, and best practices for implementation.
