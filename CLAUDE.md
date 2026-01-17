# Production-Grade Data Stack Reference Summary

This document summarizes the recommended architecture, tools, and orchestration strategies for a modern, production-grade data stack for a small company. It is designed to be **pragmatic, incremental, and AI-friendly** for project documentation. It should be written in British English with a focus on clarity and learning.

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

* **BI Tools**: Metabase
* **Advanced analytics**: Python notebooks (Jupyter)
* **ML Workflows**: Prefect orchestrates preprocessing pipelines

### Secrets & Security

* **AWS Secrets Manager**: Store credentials for Snowflake, Kafka, dlt, dbt, Airbyte
* **RBAC**: Snowflake roles, Kafka service accounts, GitHub CODEOWNERS

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
- **GitHub organisation setup** - Repository structure, team access, branch protection
- **Local developer environment** - VS Code, iTerm, Homebrew, Git configuration (macOS focus)
- **Development workflow** - Branching strategy, PR process, code review

### Infrastructure as Code (`docs/build/infrastructure-as-code/`)
- **Terraform fundamentals** - Installation, configuration, basic concepts
- **Remote state management** - S3 backend, DynamoDB locking, workspace management
- **AWS setup for Terraform** - Creating state buckets, IAM policies, authentication
- **Deployment workflows** - CI/CD with GitHub Actions, terraform plan/apply
- **Best practices** - Module structure, naming conventions, documentation

### Data Warehouse (`docs/build/data-warehouse/`)
- **0-create-snowflake-account.md** - Manual account creation, initial admin user
- **1-terraform-setup.md** - Snowflake Terraform provider, authentication, project structure
- **2-warehouses-resource-monitors.md** - Compute resources and cost controls
- **3-databases.md** - Storage setup for dev/test/prod environments
- **4-roles-rbac.md** - Functional roles and RBAC (ANALYTICS_DEVELOPER, etc.)
- **5-users.md** - Admin users, developer users, service accounts
- **6-network-policies.md** - IP allowlisting for security
- **7-sso-setup.md** - SSO integration with GSuite, Okta, Azure AD
- **8-storage-integrations.md** - S3 storage integrations for data loading

Each data warehouse guide includes:
- Concept explanation (what and why)
- **Terraform code** (primary method, production-ready)
- **SQL reference** (in tabs, for understanding what Terraform creates)
- Working examples from `repositories/terraform/snowflake/`

### Other Build Sections (Future)
- API Ingestion (dlt pipelines)
- SaaS Ingestion (Airbyte)
- Data Transformation (dbt)
- Streaming (Kafka/Confluent)
- etc.

---

## 6. Repository Structure

The project includes ready-to-use Terraform configurations:

```
documentation/
├── repositories/
│   ├── terraform/
│   │   ├── snowflake/
│   │   │   ├── config/              # Terraform configuration files
│   │   │   │   ├── backend.tf       # S3 remote state
│   │   │   │   ├── main.tf          # Terraform & provider versions
│   │   │   │   ├── providers.tf     # Snowflake providers (per admin role)
│   │   │   │   ├── variables.tf     # Input variable definitions
│   │   │   │   ├── terraform.tfvars # Variable values (customise for your org)
│   │   │   │   ├── warehouses.tf    # Warehouse resources
│   │   │   │   ├── databases.tf     # Database resources
│   │   │   │   ├── functional_roles.tf      # ANALYTICS_* roles
│   │   │   │   ├── users.tf                 # All user resources
│   │   │   │   ├── network_policies.tf      # IP allowlisting
│   │   │   │   ├── sso_integrations.tf      # SAML2 integrations
│   │   │   │   ├── storage_integrations.tf  # S3 access
│   │   │   │   └── resource_monitor.tf      # Cost controls
│   │   │   └── modules/             # Reusable modules
│   │   │       ├── snowflake_database/
│   │   │       ├── snowflake_user/
│   │   │       ├── snowflake_warehouse/
│   │   │       ├── snowflake_role/
│   │   │       ├── snowflake_database_role/
│   │   │       ├── snowflake_schema/
│   │   │       ├── snowflake_storage_integration/
│   │   │       └── snowflake_saml2_integration/
│   │   ├── aws/                     # Future: S3 buckets, IAM, etc.
│   │   └── confluent/               # Future: Kafka configuration
│   └── setup_script/
└── docs/
    ├── getting-started/
    ├── build/
    │   ├── infrastructure-as-code/
    │   ├── data-warehouse/
    │   ├── api-ingestion/
    │   └── ...
    └── maintain/
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

This file serves as a **reference for other AI agents** or team members to understand the stack, its orchestration, documentation structure, and best practices for implementation.
