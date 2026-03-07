# Claude Code Setup

On this page, you will:

- [x] Understand how CLAUDE.md and skills guide Claude Code in your repositories
- [x] Add a CLAUDE.md and maintenance skills to your Terraform repository
- [x] Create a CLAUDE.md and skill templates for your dbt repository
- [x] Create a CLAUDE.md and skill template for your Prefect repository

## Overview

As you build repositories throughout this documentation - Terraform, dbt, and Prefect - each one develops its own conventions, module patterns, and safety rules. When an AI agent works in these repositories - adding users, creating pipelines, building models - it needs to follow those conventions precisely. A wrong naming pattern or a missing grant can break downstream access.

**CLAUDE.md** is a project reference file that sits at the root of a repository. It tells Claude Code about the repository's structure, conventions, and safety rules. Think of it as onboarding documentation - but for an AI agent rather than a human.

**Skills** are reusable task templates stored in `.claude/skills/`. Each skill provides step-by-step instructions for a common task, ensuring Claude Code follows the correct procedure every time.

```
your-repository/
├── CLAUDE.md                  ← Project context and conventions
├── .claude/
│   └── skills/
│       ├── add-user/
│       │   └── SKILL.md       ← Step-by-step task instructions
│       └── add-data-source/
│           └── SKILL.md
└── (project files)
```

!!! info "Add These as You Build"
    You don't need to create all three CLAUDE.md files now. Add each one when you reach the corresponding section: Terraform CLAUDE.md during the [Build](../../build/data-warehouse/index.md) sections, dbt during [Data Transformation](../../build/data-transformation/index.md), and Prefect during [Orchestration](../../build/orchestration/index.md).

## Terraform Repository

The Terraform repository has the most complex conventions - multiple Snowflake provider aliases, module patterns that create resources with all associated grants, service account naming, and the reader access chain. Getting any of these wrong means broken permissions or orphaned resources.

### CLAUDE.md

The CLAUDE.md for your Terraform repository covers:

- **Repository structure** - the `github/`, `aws/`, and `snowflake/` directories with their `config/` and `modules/` subdirectories
- **Module patterns** - how each module creates a complete resource (e.g. `snowflake_database` creates the database plus `DB_READER` and `DB_WRITER` roles and all grants)
- **Provider aliases** - the four Snowflake providers (`sys_admin`, `security_admin`, `user_admin`, `account_admin`) and when to use each
- **Service account pattern** - `SVC_` prefix, `user_create_dedicated_role = true`, key-pair authentication
- **Reader access chain** - `{DB}_DB_READER` → `ANALYTICS_SOURCES_READER` → analyst roles
- **Safety rules** - never apply locally, always plan first, no hard-coded ARNs

If you built the Terraform repository following the documentation, the CLAUDE.md is already included. You can find it at the repository root:

```
terraform/
├── CLAUDE.md                    ← Already included
├── .claude/
│   └── skills/
│       ├── add-snowflake-user/
│       │   └── SKILL.md         ← Already included
│       └── add-data-source/
│           └── SKILL.md         ← Already included
├── github/
├── aws/
└── snowflake/
```

### Skills

Two maintenance skills are included:

**`add-snowflake-user`** - Adds a new Snowflake user (admin, developer, or service account) using the `snowflake_user` module. It determines the correct user category, adds the entry to the right configuration file, sets the appropriate role and warehouse, and validates with `terraform plan`.

Invoke with:

```
/add-snowflake-user
```

**`add-data-source`** - Adds complete infrastructure for a new data source: database (named after the loader tool), service account with dedicated role, schemas, reader grants to `ANALYTICS_SOURCES_READER`, and an AWS Secrets Manager container for credentials.

Invoke with:

```
/add-data-source
```

## dbt Repository

The dbt repository follows strict naming conventions - `stg_{source}__{table}` with double underscores, one YAML file per model, specific materialisation rules per layer - that are critical for the project to remain consistent and for CI/CD checks to pass.

### CLAUDE.md

Create a `CLAUDE.md` at the root of your `dbt-transform` repository. Copy the template below:

??? example "CLAUDE.md for dbt-transform (click to expand)"

    ```markdown
    # dbt Transformation Repository

    This repository contains the dbt project for transforming raw data into
    analytics-ready models in Snowflake. It implements a four-layer architecture
    (staging, intermediate, marts, reporting) with strict naming conventions and
    testing requirements.

    There are skills available to help with common maintenance tasks -
    `add-source-and-staging` and `add-mart-model`.

    ---

    ## Repository Structure

    ```text
    dbt-transform/
    ├── dbt_project.yml
    ├── packages.yml
    ├── profiles.yml           # Generated locally, not committed
    ├── .envrc.example
    ├── pyproject.toml         # Dependencies (managed by uv)
    ├── models/
    │   ├── staging/           # stg_{source}__{table} (views)
    │   ├── intermediate/      # int_{entity}__{verb} (ephemeral)
    │   └── marts/
    │       ├── core/          # fct_ / dim_ (tables/incremental)
    │       ├── crm/           # Domain-specific marts
    │       └── reporting/     # rpt_ (views for BI tools)
    ├── tests/
    ├── macros/
    └── seeds/
    ```

    ## Model Layers

    | Layer | Schema | Materialisation | Naming |
    |-------|--------|-----------------|--------|
    | Staging | ANALYTICS.STAGING | View | stg_{source}__{table} |
    | Intermediate | (ephemeral) | Ephemeral | int_{entity}__{verb} |
    | Marts | ANALYTICS.MARTS | Table/Incremental | fct_ / dim_ |
    | Reporting | ANALYTICS.REPORTING | View | rpt_ |

    ## Key Conventions

    - One YAML file per model (not per directory)
    - Sources in _sources.yml alongside staging models
    - Every model needs a YAML file with description and tests
    - Primary keys: unique + not_null tests
    - Foreign keys: relationships test
    - Source freshness: warn_after + error_after
    - Use dbt_expectations for complex validations

    ## Safety Rules

    - Never run against production without --defer
    - Always run dbt build (not just dbt run) to include tests
    - Never use pip install; use uv add
    - Use bare dbt commands (direnv activates .venv)
    - Dev is the default target; never use --target dev

    ## Authentication

    - Local: .envrc with Snowflake credentials
    - CI/CD: AWS Secrets Manager
    - Service account: SVC_DBT with ANALYTICS_TRANSFORMER role

    ## Style

    - British English: materialisation, organisation, customise
    - Spaced hyphens for parenthetical statements
    ```

!!! tip "Full Template"
    The complete CLAUDE.md template with all sections and detailed conventions is available in the [dbt CLAUDE.md template](../../maintain/templates/dbt-claude.md).

### Skills

Create the skills directory structure in your `dbt-transform` repository:

```sh
mkdir -p .claude/skills/add-source-and-staging
mkdir -p .claude/skills/add-mart-model
```

**`add-source-and-staging`** - Creates a source definition (`_sources.yml`), staging model SQL, and model YAML with the correct naming conventions (`stg_{source}__{table}`), freshness checks, and testing requirements.

Copy the [add-source-and-staging skill template](../../maintain/templates/add-source-and-staging.md) to `.claude/skills/add-source-and-staging/SKILL.md`.

**`add-mart-model`** - Creates a fact table, dimension table, or reporting view with the correct materialisation strategy, schema routing, and test coverage.

Copy the [add-mart-model skill template](../../maintain/templates/add-mart-model.md) to `.claude/skills/add-mart-model/SKILL.md`.

## Prefect Repository

The Prefect repository follows a three-layer architecture that separates extraction (sources), pipeline configuration (pipelines), and orchestration (flows). Every flow must include retries with exponential backoff.

### CLAUDE.md

Create a `CLAUDE.md` at the root of your `data-pipelines` repository. Copy the template below:

??? example "CLAUDE.md for data-pipelines (click to expand)"

    ```markdown
    # Data Pipelines Repository

    This repository contains dlt data pipelines orchestrated by Prefect.
    It handles all batch data ingestion from APIs, databases, and SaaS tools
    into Snowflake, following a three-layer architecture.

    There is a skill available - `add-dlt-pipeline`.

    ---

    ## Repository Structure

    ```text
    data-pipelines/
    ├── sources/               # dlt source definitions
    ├── pipelines/             # dlt pipeline configurations
    ├── flows/                 # Prefect flow definitions
    ├── utils/
    │   └── vault_provider.py  # AWS Secrets Manager provider
    ├── .dlt/
    │   └── secrets.toml       # Local credentials (gitignored)
    ├── prefect.yaml           # Deployment configuration
    └── pyproject.toml         # Dependencies (managed by uv)
    ```

    ## Three-Layer Architecture

    | Layer | Directory | Responsibility |
    |-------|-----------|---------------|
    | Sources | sources/ | Extract data (pure dlt) |
    | Pipelines | pipelines/ | Configure destination and dataset |
    | Flows | flows/ | Orchestrate with Prefect |

    Flows import pipelines. Pipelines import sources.

    ## Key Conventions

    - @dlt.source(section=) for config grouping
    - Explicit write_disposition on every resource
    - DuckDB for local testing (DLT__DESTINATION=duckdb)
    - Retries: 3 with exponential backoff [10, 30, 90]
    - Import pipeline functions inside task body

    ## Safety Rules

    - Never commit .dlt/secrets.toml or .envrc
    - Never use pip install; use uv add
    - Always test with DuckDB before Snowflake
    - Always include retries (minimum 3)
    - Never hard-code credentials

    ## Authentication

    - Local: .dlt/secrets.toml + .envrc
    - CI/CD: AWS Secrets Manager via vault_provider.py
    - Snowflake: SVC_DLT with key-pair auth

    ## Style

    - British English: organisation, customise, analyse
    - Spaced hyphens for parenthetical statements
    ```

!!! tip "Full Template"
    The complete CLAUDE.md template with all sections, destination patterns, and detailed conventions is available in the [Prefect CLAUDE.md template](../../maintain/templates/prefect-claude.md).

### Skills

Create the skills directory in your `data-pipelines` repository:

```sh
mkdir -p .claude/skills/add-dlt-pipeline
```

**`add-dlt-pipeline`** - Creates the full three-layer pipeline: source package, pipeline configuration, Prefect flow with retries, deployment entry in `prefect.yaml`, and credentials setup. Includes a pre-PR checklist and DuckDB testing.

Copy the [add-dlt-pipeline skill template](../../maintain/templates/add-dlt-pipeline.md) to `.claude/skills/add-dlt-pipeline/SKILL.md`.

## Verifying Your Setup

After adding the CLAUDE.md and skills to a repository, verify that Claude Code recognises them:

1. Open the repository in Claude Code
2. Ask Claude about the project structure - it should reference the CLAUDE.md conventions
3. Invoke a skill (e.g. `/add-snowflake-user`) and verify it follows the documented steps
4. Check that safety rules are respected - Claude should refuse to hard-code credentials or skip tests

!!! info "Skills Are Invocable"
    Skills are invoked by typing `/{skill-name}` in Claude Code. For example, `/add-snowflake-user` invokes the add-snowflake-user skill. Claude Code reads the SKILL.md and follows the steps, asking you for the required information along the way.

## Summary

!!! success "What You've Accomplished"
    - [x] Understood how CLAUDE.md and skills guide Claude Code in your repositories
    - [x] Added a CLAUDE.md and two maintenance skills to the Terraform repository
    - [x] Created a CLAUDE.md and two skill templates for the dbt repository
    - [x] Created a CLAUDE.md and one skill template for the Prefect repository

Each repository now has:

| Repository | CLAUDE.md | Skills |
|-----------|-----------|--------|
| `terraform/` | Architecture, modules, naming, safety rules | `add-snowflake-user`, `add-data-source` |
| `dbt-transform/` | Model layers, naming, testing, materialisation | `add-source-and-staging`, `add-mart-model` |
| `data-pipelines/` | Three-layer architecture, credentials, retries | `add-dlt-pipeline` |

## What's Next

!!! success
    Your repositories are now set up for AI-assisted development and maintenance. Claude Code can add users, create data sources, build models, and set up pipelines - all following your established conventions.

Continue to [Account Setup](../account-setup/index.md) →
