# Repository Integration

On this page, you will:

- [x] Add a `docs/` directory to each source repository
- [x] Create initial documentation pages covering architecture, getting started, and conventions
- [x] Configure each repo for independent local preview
- [x] Verify the unified site imports from all repositories

## Overview

Each repository gets a `docs/` directory with documentation focused on **how to use the repository** - its architecture, getting started guide, conventions, and runbooks. This is distinct from resource-level documentation (individual module READMEs, model descriptions, pipeline docstrings) which stays in-place alongside the code.

```
terraform/                          data-pipelines/
├── github/                         ├── sources/
├── aws/                            ├── pipelines/
├── snowflake/                      ├── flows/
├── docs/          ◄─ NEW          ├── docs/          ◄─ NEW
│   ├── mkdocs.yml                  │   ├── mkdocs.yml
│   ├── index.md                    │   ├── index.md
│   ├── getting-started.md          │   ├── getting-started.md
│   ├── conventions.md              │   ├── conventions.md
│   ├── modules.md                  │   └── runbooks/
│   └── runbooks/                   │       └── index.md
│       └── index.md
└── ...                             └── ...

dbt-transform/
├── models/
├── tests/
├── docs/              ◄─ NEW
│   ├── mkdocs.yml
│   ├── index.md
│   ├── getting-started.md
│   ├── conventions.md
│   └── runbooks/
│       └── index.md
└── ...
```

## Terraform Repository

### Add the Documentation Directory

```sh
cd ~/src/terraform
mkdir -p docs/runbooks
```

### Add MkDocs Dependencies

If the repository does not already have a `pyproject.toml`:

```sh
uv init
```

Add the MkDocs dependencies for local preview:

```sh
uv add --dev mkdocs mkdocs-material mkdocs-glightbox
```

!!! tip "Minimal Dependencies"
    Each source repository only needs `mkdocs` and `mkdocs-material` for local preview. The multirepo plugin and git-revision-date plugin are only needed in the central `technical-documentation` repository.

### Create the Local MkDocs Configuration

Create `docs/mkdocs.yml` for independent local preview:

```yaml
site_name: Infrastructure Documentation
docs_dir: .
nav:
  - Overview: index.md
  - Getting Started: getting-started.md
  - Conventions: conventions.md
  - Modules: modules.md
  - Runbooks:
      - runbooks/index.md
theme:
  name: material
  font:
    text: Raleway
  palette:
    - media: "(prefers-color-scheme: light)"
      scheme: default
      primary: blue grey
      toggle:
        icon: material/lightbulb
        name: Switch to dark mode
    - media: "(prefers-color-scheme: dark)"
      scheme: slate
      primary: light blue
      toggle:
        icon: material/lightbulb-outline
        name: Switch to light mode
  features:
    - navigation.instant
    - navigation.tracking
    - toc.follow
plugins:
  - search
  - glightbox
markdown_extensions:
  - admonition
  - attr_list
  - pymdownx.highlight:
      anchor_linenums: true
  - pymdownx.inlinehilite
  - pymdownx.snippets
  - pymdownx.superfences:
      custom_fences:
        - name: mermaid
          class: mermaid
          format: !!python/name:pymdownx.superfences.fence_code_format
  - pymdownx.tasklist:
      custom_checkbox: true
      clickable_checkbox: true
  - pymdownx.tabbed:
      alternate_style: true
  - pymdownx.keys
```

!!! info "Shared Theme Configuration"
    The theme, palette, and markdown extensions match the central repository. This ensures that pages look the same whether previewed locally or on the unified site. If you change the theme in the central repository, update each source repository's `mkdocs.yml` to match.

### Create Initial Pages

Create `docs/index.md`:

```markdown
# Infrastructure Documentation

This repository manages all infrastructure as code for the data platform using
Terraform. It provisions and configures resources across GitHub, AWS, and Snowflake.

## Repository Structure

` `` (use triple backticks)
terraform/
├── github/          GitHub organisation, teams, users
├── aws/             S3, IAM, VPC, Secrets Manager
│   ├── config/      Resource definitions
│   └── modules/     Reusable modules (s3_bucket, vpc)
└── snowflake/       Warehouses, databases, roles, users
    ├── config/      Resource definitions and tfvars
    └── modules/     Reusable modules (database, user, warehouse, etc.)
` ``

## Architecture

The repository uses **multiple Terraform state files** - one per provider (GitHub,
AWS, Snowflake) - each with its own backend configuration. This separation means
a change to Snowflake resources does not require locking the AWS state file.

### Provider Model

Snowflake uses **multiple provider aliases** to operate with different admin roles:

- `snowflake.sys_admin` — Creates warehouses and databases
- `snowflake.security_admin` — Manages role grants
- `snowflake.user_admin` — Creates users and roles
- `snowflake.account_admin` — Account-level settings (rarely used)

## Quick Links

- [Getting Started](getting-started.md) — Set up your local environment
- [Conventions](conventions.md) — Naming patterns and module standards
- [Runbooks](runbooks/index.md) — Operational procedures
```

Create `docs/getting-started.md`:

```markdown
# Getting Started

This guide covers setting up your local environment to work with the Terraform
repository.

## Prerequisites

- Terraform 1.6+ installed
- AWS CLI configured with `infrastructure-admin` and `data-engineer` profiles
- Snowflake account with admin access (for initial setup)
- 1Password CLI for secrets management
- pre-commit installed (`brew install pre-commit`)
- terraform-docs installed (`brew install terraform-docs`)

## Local Setup

1. Clone the repository
2. Install pre-commit hooks:

    ` ``sh
    pre-commit install --hook-type pre-push
    ` ``

3. Navigate to the provider directory you need to work with (e.g. `snowflake/config/`)
4. Run `terraform init` to initialise the backend and download providers
5. Run `terraform plan` to preview changes
6. Create a pull request — CI/CD handles `terraform apply`

## Development Workflow

All Terraform changes follow the same process:

1. Create a feature branch
2. Make changes to `.tf` and `.auto.tfvars` files
3. Run `terraform plan` locally to verify
4. Open a pull request
5. CI validates the plan (and pre-commit hooks run terraform-docs to update module READMEs)
6. After approval and merge, CD runs `terraform apply`

!!! warning "Never Apply Locally"
    All `terraform apply` operations run through CI/CD. Local `terraform plan`
    is safe and encouraged for validation, but applying changes locally risks
    state corruption and bypasses the review process.

## Pre-commit Hooks

The repository uses pre-commit hooks that run on `git push`:

| Hook | Purpose |
|------|---------|
| `terraform_fmt` | Auto-format Terraform code |
| `terraform_validate` | Validate syntax |
| `terraform_docs` | Generate module documentation |
| `terraform_tflint` | Lint Terraform code |
| `terraform_checkov` | Security scanning |

Run hooks manually at any time:

` ``sh
pre-commit run --all-files
` ``
```

Create `docs/conventions.md`:

```markdown
# Conventions

Naming patterns, module standards, and configuration approach used across this
repository.

## Naming Conventions

| Resource Type | Convention | Example |
|---------------|-----------|---------|
| Snowflake objects | `UPPER_CASE` | `ANALYTICS`, `SVC_DLT`, `DEVELOPER` |
| Terraform resources | `snake_case` | `snowflake_warehouse.loading` |
| Terraform modules | `snake_case` | `module "database_dlt"` |
| Service accounts | `SVC_` prefix | `SVC_TERRAFORM`, `SVC_DBT` |
| Variable files | `*.auto.tfvars` | `users.auto.tfvars` |

## Service Account Naming

All service accounts use the `SVC_` prefix:

| Account | Purpose |
|---------|---------|
| `SVC_TERRAFORM` | CI/CD Terraform operations |
| `SVC_DLT` | dlt pipeline loading |
| `SVC_AIRBYTE` | Airbyte SaaS ingestion |
| `SVC_DBT` | dbt transformations |
| `SVC_LIGHTDASH` | BI tool access |
| `SVC_KAFKA_CONNECTOR` | Kafka Connect to Snowflake |

## Database Naming

Databases are named after the loader tool that writes to them:

- `DLT` — dlt batch pipelines
- `SNOWPIPE` — Snowpipe auto-ingest
- `AIRBYTE` — Airbyte connections
- `STREAMING` — Kafka Connect
- `ANALYTICS` — dbt transformations
- `ANALYTICS_DEV` — dbt development
- `ADMIN` — Administrative objects (resource monitors, tasks)

## Module Patterns

Each Terraform module creates a complete resource with all associated permissions.
For example, `snowflake_database` creates the database plus `DB_READER` and
`DB_WRITER` database roles with appropriate grants.

Modules accept multiple Snowflake provider aliases:

` ``hcl
module "database_dlt" {
  source = "./modules/snowflake_database"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  database_name    = "DLT"
  database_comment = "Data loaded by dlt pipelines."
  # ...
}
` ``

## Variable Files

Configuration values live in `.auto.tfvars` files, which Terraform loads
automatically. Use descriptive filenames:

- `users.auto.tfvars` — User definitions
- `network_policies.auto.tfvars` — IP allowlists
- `warehouses.auto.tfvars` — Warehouse configurations

## Documentation Generation

Module documentation is auto-generated using [terraform-docs](https://terraform-docs.io/).

### How It Works

The pre-commit hook `terraform_docs` runs on every `git push` and generates a
`README.md` in each module directory containing:

- **Inputs** — All variables with type, description, default, and whether required
- **Outputs** — All outputs with description
- **Resources** — Terraform resources managed by the module
- **Providers** — Required provider versions

### README Markers

Each module's `README.md` must include injection markers. terraform-docs inserts
generated content between these markers, preserving any hand-written content above
or below:

` ``markdown
# Module: Snowflake Database

Creates a Snowflake database with DB_READER and DB_WRITER database roles.

<!-- BEGIN_TF_DOCS -->
<!-- END_TF_DOCS -->
` ``

### Variable and Output Descriptions

All variables and outputs **must** include a `description` field. terraform-docs
uses these to generate the documentation:

` ``hcl
variable "database_name" {
  description = "Name of the Snowflake database to create."
  type        = string
}

output "database_name" {
  description = "The name of the created database."
  value       = snowflake_database.this.name
}
` ``

### Configuration

The `.terraform-docs.yml` file at the repository root controls formatting:

- **Formatter**: `markdown table` — clean table layout
- **Sort**: By `required` — required variables appear first
- **Mode**: `inject` — inserts between markers rather than replacing the whole file

Run terraform-docs manually for a specific module:

` ``sh
terraform-docs markdown table snowflake/modules/snowflake_database/
` ``
```

Create `docs/runbooks/index.md`:

```markdown
# Runbooks

Operational procedures for common tasks and incident response.

## Available Runbooks

| Runbook | When to Use |
|---------|------------|
| Adding a new Snowflake user | New team member needs access |
| Rotating service account credentials | Scheduled rotation or key compromise |
| Responding to a failed Terraform deployment | CI/CD `terraform apply` fails |
| Adding a new data source | New ingestion source needs database, service account, and schemas |

!!! info "Platform-Level Runbooks"
    For comprehensive runbooks covering the full data stack (not just Terraform), see the Maintain section on the platform documentation site. These runbooks cover end-to-end procedures including pipeline setup, dbt configuration, and cross-tool coordination.

See [Writing Runbooks](../../../../build/documentation/4-writing-runbooks.md) in
the platform guide for the runbook template and structure.
```

### Create the Module Reference Page

Create `docs/modules.md` to document all Terraform modules. This page lists each module with its purpose, key inputs, and shows how modules relate to each other.

??? example "docs/modules.md (click to expand)"

    ```markdown
    --8<-- "build/documentation/examples/terraform-modules.md"
    ```

!!! tip "Auto-Generated Module Documentation"
    The pre-commit hook `terraform_docs` generates a `README.md` in each module directory with inputs, outputs, and resources tables. Configure formatting in `.terraform-docs.yml` at the repository root. These READMEs complement the `docs/modules.md` page — the READMEs provide raw reference alongside the code, the docs page provides context and usage guidance for the unified site.

## Data Pipelines Repository

### Add the Documentation Directory

```sh
cd ~/src/data-pipelines
mkdir -p docs/runbooks
```

Add MkDocs dependencies:

```sh
uv add --dev mkdocs mkdocs-material mkdocs-glightbox
```

### Create the Local MkDocs Configuration

Create `docs/mkdocs.yml` with the same theme configuration as the Terraform repository (see above), updating:

```yaml
site_name: Data Pipelines Documentation
```

### Create Initial Pages

Create `docs/index.md`:

```markdown
# Data Pipelines Documentation

This repository manages data ingestion pipelines using dlt (data load tool) and
orchestration with Prefect.

## Repository Structure

` `` (use triple backticks)
data-pipelines/
├── sources/          dlt source definitions
├── pipelines/        dlt pipeline configurations
├── flows/            Prefect flow definitions
├── tests/            Pipeline tests
└── docs/             Documentation (you are here)
` ``

## Architecture

Pipelines follow a consistent pattern:

1. **Source** — Defines how to extract data (API, database, file)
2. **Pipeline** — Configures the source, destination, and write disposition
3. **Flow** — Wraps the pipeline in a Prefect flow with scheduling and error handling

## Quick Links

- [Getting Started](getting-started.md) — Set up your local environment
- [Conventions](conventions.md) — Pipeline patterns and configuration approach
- [Runbooks](runbooks/index.md) — Operational procedures
```

Create `docs/getting-started.md` and `docs/conventions.md` following the same pattern as the Terraform repository, tailored to pipeline development:

- **Getting started**: How to set up the local environment, run a pipeline locally, configure credentials via `.dlt/secrets.toml`
- **Conventions**: Pipeline naming, source structure, configuration patterns, testing approach

Create `docs/runbooks/index.md` with placeholder content listing planned runbooks (adding a new source, backfilling data, debugging a failed pipeline).

## dbt Transform Repository

### Add the Documentation Directory

```sh
cd ~/src/dbt-transform
mkdir -p docs/runbooks
```

Add MkDocs dependencies:

```sh
uv add --dev mkdocs mkdocs-material mkdocs-glightbox
```

### Create the Local MkDocs Configuration

Create `docs/mkdocs.yml` with the same theme configuration, updating:

```yaml
site_name: dbt Transform Documentation
```

### Create Initial Pages

Create `docs/index.md`:

```markdown
# dbt Transform Documentation

This repository manages data transformation using dbt (data build tool). Models
transform raw data from ingestion sources into analytics-ready tables.

## Repository Structure

` `` (use triple backticks)
dbt-transform/
├── models/
│   ├── staging/         Clean raw data (views)
│   ├── intermediate/    Business logic (ephemeral)
│   ├── marts/           Analytics tables (tables/incremental)
│   └── reporting/       BI-facing subset (views)
├── tests/               Custom singular tests
├── macros/              Reusable SQL macros
└── docs/                Documentation (you are here)
` ``

## Auto-Generated dbt Docs

This repository also generates a dbt docs site with a lineage graph and model
explorer. The auto-generated site is hosted separately:

- **dbt Core**: Published to CloudFront via CI/CD
- **dbt Cloud**: Built-in docs at your dbt Cloud URL

The auto-generated docs complement this documentation — they cover individual model
details, column descriptions, and lineage. This site covers how to use the repository
as a whole.

## Quick Links

- [Getting Started](getting-started.md) — Set up your local environment
- [Conventions](conventions.md) — Naming patterns and testing strategy
- [Runbooks](runbooks/index.md) — Operational procedures
```

Create `docs/getting-started.md` and `docs/conventions.md` tailored to dbt development:

- **Getting started**: How to set up locally, run `dbt build`, understand model layers, connect to Snowflake
- **Conventions**: Model naming (`stg_`, `int_`, `fct_`, `dim_`), materialisation choices, testing strategy, YAML structure

Create `docs/runbooks/index.md` with placeholder content listing planned runbooks (adding a new model, running a full refresh, debugging test failures).

## Verify Independent Builds

Each repository can now build its own documentation independently:

```sh
# From within each repo's docs directory
cd ~/src/terraform/docs && uv run mkdocs build --strict
cd ~/src/data-pipelines/docs && uv run mkdocs build --strict
cd ~/src/dbt-transform/docs && uv run mkdocs build --strict
```

!!! tip "Local Preview"
    To preview a single repository's docs during development:

    ```sh
    cd ~/src/terraform/docs
    uv run mkdocs serve
    ```

    This is faster than building the unified site and does not require network access.

## Verify the Unified Build

Return to the central documentation repository and build the unified site:

```sh
cd ~/src/technical-documentation
uv run mkdocs build --strict
```

The multirepo plugin clones each source repository and imports its `docs/` directory. The resulting site includes all repositories with unified search.

!!! warning "Git Authentication"
    The multirepo plugin clones repositories using Git. For private repositories, ensure your SSH key or Git credentials are configured. The plugin uses the same authentication as `git clone`.

## Summary

!!! success "What You've Accomplished"
    - [x] Added `docs/` directory to the Terraform repository with architecture, getting started, and conventions pages
    - [x] Added `docs/` directory to the data pipelines repository with initial content
    - [x] Added `docs/` directory to the dbt-transform repository with initial content and link to auto-generated dbt docs
    - [x] Configured each repository for independent local preview
    - [x] Verified the unified site builds with all repositories imported

## Next Steps

Your repositories now have documentation directories with initial structure. The next step is to understand what makes effective repository documentation and how to write content that stays useful over time.

Continue to [Writing Documentation](3-writing-documentation.md) →
