# Build Your Data Warehouse

In this section, you'll build a production-ready Snowflake data warehouse using Terraform. You'll create reusable modules that encapsulate best practices and set up a complete environment with warehouses, databases, roles, and users.

## What is a Data Warehouse?

A data warehouse is a centralised repository for storing and analysing structured data from multiple sources. Unlike operational databases designed for transactions, data warehouses are optimised for analytical queries - aggregating, filtering, and joining large datasets to answer business questions.

**Snowflake** is a cloud-native data warehouse that separates compute from storage, allowing you to scale each independently. Key concepts:

- **Warehouses**: Virtual compute clusters that run your queries (you pay for compute time)
- **Databases**: Logical containers for your data (you pay for storage)
- **Schemas**: Namespaces within databases to organise tables and views
- **Roles**: Access control primitives that define what users can do

## What You'll Build

By the end of this section, you'll have a complete Snowflake environment:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           SNOWFLAKE ACCOUNT                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  WAREHOUSES (Compute)                                                       │
│  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐ ┌───────────────┐    │
│  │   LOADING     │ │ TRANSFORMING  │ │  REPORTING    │ │  DEVELOPER    │    │
│  │   (X-Small)   │ │   (Small)     │ │   (Small)     │ │   (X-Small)   │    │
│  └───────────────┘ └───────────────┘ └───────────────┘ └───────────────┘    │
│                                                                             │
│  DATABASES (Storage)                                                        │
│  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐ ┌───────────────┐    │
│  │   ANALYTICS   │ │ ANALYTICS_DEV │ │     ADMIN     │ │  <LOADERS>    │    │
│  │ (Production)  │ │ (Development) │ │  (Governance) │ │ (Per source)  │    │
│  └───────────────┘ └───────────────┘ └───────────────┘ └───────────────┘    │
│                                                                             │
│  FUNCTIONAL ROLES (Access Control)                                          │
│  ┌─────────────────────--┐ ┌─────────────────────-┐ ┌─────────────────────┐ │
│  │ ANALYTICS_DEVELOPER   │ │ ANALYTICS_TRANSFORMER│ │ ANALYTICS_REPORTER  │ │
│  │ (Read sources,        │ │ (Read sources,       │ │ (Read reporting     │ │
│  │  write ANALYTICS_DEV) │ │  write ANALYTICS)    │ │  schemas only)      │ │
│  └─────────────────────--┘ └─────────────────────-┘ └─────────────────────┘ │
│                                                                             │
│  USERS                                                                      │
│  ┌─────────────────────┐ ┌─────────────────────┐ ┌─────────────────────┐    │
│  │    Admin Users      │ │   Developer Users   │ │  Service Accounts   │    │
│  │ (JBLOGGS_ADMIN)     │ │     (JBLOGGS)       │ │ (SVC_DBT, SVC_...)  │    │
│  └─────────────────────┘ └─────────────────────┘ └─────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Architecture Patterns

This section implements several production-ready patterns:

### Functional Roles

Rather than granting permissions directly to users, we create **functional roles** that represent job functions:

| Role | Purpose | Access |
|------|---------|--------|
| `ANALYTICS_DEVELOPER` | Data team members developing models | Read sources, read/write dev database, read prod |
| `ANALYTICS_TRANSFORMER` | dbt and transformation tools | Read sources, write production analytics |
| `ANALYTICS_REPORTER` | BI tools like Metabase | Read reporting schemas only |
| `ANALYTICS_SOURCES_READER` | Read access to all source data | Granted to developers and transformers |

Users are assigned functional roles as their `default_role`, ensuring they operate with appropriate privileges.

### Database Roles

Each database gets two **database roles** automatically:

- `<DATABASE>_DB_READER` - Read-only access to all schemas and objects
- `<DATABASE>_DB_WRITER` - Full access to create, modify, and delete

These database roles are then granted to functional roles, creating a clean hierarchy:

```
ANALYTICS_DEVELOPER
  └── ANALYTICS_DB_READER       (read production)
  └── ANALYTICS_DEV_DB_WRITER   (write development)
  └── ANALYTICS_SOURCES_READER
        └── AIRBYTE_DB_READER   (read source data)
        └── DLT_DB_READER       (read source data)
```

### Dedicated Warehouses

Different workloads get dedicated warehouses:

- **LOADING**: For data ingestion (Airbyte, dlt)
- **TRANSFORMING**: For dbt transformations
- **REPORTING**: For BI tool queries
- **DEVELOPER**: For ad-hoc queries and development

This allows you to:

- Size each warehouse appropriately for its workload
- Track costs by workload type
- Suspend idle warehouses independently
- Set different auto-suspend timeouts

### Separation of Admin and Regular Accounts

Each admin person has two accounts:

- `JBLOGGS_ADMIN` - For administrative tasks, uses SYSADMIN/ACCOUNTADMIN
- `JBLOGGS` - For daily work, uses ANALYTICS_DEVELOPER

This separation ensures that administrative privileges aren't accidentally used for routine queries, reducing the risk of unintended changes.

## Terraform Modules

You'll build reusable modules that encapsulate these patterns:

| Module | Creates | Used For |
|--------|---------|----------|
| `snowflake_warehouse` | Warehouse + usage grants | Compute resources |
| `snowflake_database` | Database + DB_READER/DB_WRITER roles + grants | Data storage |
| `snowflake_database_role` | Database role + schema/object grants | Fine-grained access |
| `snowflake_role` | Account role | Functional roles |
| `snowflake_schema` | Schema + grants | Organising tables |
| `snowflake_user` | User + dedicated warehouse + role grants | Human and service accounts |
| `snowflake_storage_integration` | S3 integration + IAM trust | Loading from S3 |

Each module creates a complete resource with all its associated permissions - no manual grant management required.

## Prerequisites

Before starting this section, ensure you have completed:

- [x] [Snowflake Account Setup](../../getting-started/account-setup/snowflake.md) - Created your Snowflake account
- [x] [AWS Account Setup](../../getting-started/account-setup/aws.md) - Set up AWS for Terraform state
- [x] [Terraform Setup](../../getting-started/terraform-setup/index.md) - Configured Terraform with remote state
- [x] [Add Snowflake to Terraform](../../getting-started/terraform-setup/snowflake/index.md) - Created SVC_TERRAFORM service account

You should have a working `terraform/snowflake/` directory with:

- `SVC_TERRAFORM` service account with key-pair authentication
- Basic user management via `users.auto.tfvars`
- CI/CD workflows deploying Snowflake changes

## Section Overview

| Page | What You'll Build |
|------|-------------------|
| [1. Project Structure](1-project-structure.md) | Multiple providers, modules directory |
| [2. Warehouses](2-warehouses.md) | LOADING, TRANSFORMING, REPORTING, DEVELOPER warehouses |
| [3. Databases](3-databases.md) | ANALYTICS, ANALYTICS_DEV, ADMIN databases |
| [4. Functional Roles](4-functional-roles.md) | ANALYTICS_DEVELOPER, TRANSFORMER, REPORTER roles |
| [5. Users](5-users.md) | Admin users, developers, service accounts |
| [6. Schemas](6-schemas.md) | REPORTING schema with fine-grained access |
| [7. Network Policies](7-network-policies.md) | IP allowlisting for security |
| [8. Storage Integrations](8-storage-integrations.md) | S3 access for data loading |
| [9. SSO Setup](9-sso-setup.md) | SAML2 integration (optional) |
| [10. Finishing Up](10-finishing-up.md) | Documentation and next steps |

## Cost Considerations

Snowflake charges for:

- **Compute**: Warehouse runtime (per-second billing, minimum 60 seconds)
- **Storage**: Data at rest (per-TB per-month)
- **Data transfer**: Egress from Snowflake (minimal for most use cases)

The configuration in this guide is designed for cost efficiency:

- Warehouses auto-suspend after 60 seconds of inactivity
- X-Small warehouses for low-volume workloads
- Resource monitors to alert on unexpected spend

For a small team just starting out, expect:

- **Compute**: $50-200/month depending on query volume
- **Storage**: $23/TB/month (compressed, typically 3-5x compression)

!!! tip "Start Small"
    Begin with X-Small warehouses and scale up only when needed. Snowflake makes it easy to resize warehouses without downtime.

## What's Next

Start by setting up the project structure with multiple providers and the modules directory.

Continue to [Project Structure](1-project-structure.md) →
