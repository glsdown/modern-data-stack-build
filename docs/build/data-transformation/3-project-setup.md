# Project Setup

On this page, you will:

- [x] Create the `dbt-transform` GitHub repository with README, PR template, and CODEOWNERS
- [x] Install dbt locally and configure credentials
- [x] Initialise the project with `dbt init` and scaffold the directory structure
- [x] Configure `dbt_project.yml` with custom schema naming
- [x] Add required packages to `packages.yml`
- [x] Configure sqlfluff and yamllint with pre-commit hooks

## Overview

The dbt project lives in its own repository, separate from `data-pipelines`. This keeps analytics engineering work — model development, schema changes, test coverage — independent from Prefect infrastructure.

```
GitHub Organisation
├── terraform/               ← Infrastructure as code
├── data-pipelines/          ← Prefect flows
└── dbt-transform/           ← dbt project  ← you are here
```

## Create the Repository

In your GitHub organisation, create a new repository named `dbt-transform` through the GitHub UI:

- **Name**: `dbt-transform`
- **Visibility**: Private
- **Initialise with**: README

### README

Replace the generated `README.md` with a developer setup guide. This is the first thing a new team member reads when onboarding to the project:

````markdown
# dbt-transform

dbt transformation project for the [Company] data platform.

## Prerequisites

- [direnv](https://direnv.net/) installed globally — see the [local environment guide](https://docs.your-company.com/getting-started/local-environment/)
- [uv](https://docs.astral.sh/uv/) installed globally
- Access to Snowflake (SSO via your company identity provider)

## Getting Started

1. Clone the repository:

   ```sh
   git clone git@github.com:your-org/dbt-transform.git
   cd dbt-transform
   ```

2. Install Python dependencies:

   ```sh
   uv sync
   ```

3. Set up your local environment:

   ```sh
   cp .envrc.example .envrc
   # Edit .envrc with your Snowflake credentials and personal schema prefix
   direnv allow
   ```

4. Install dbt packages and verify the connection:

   ```sh
   dbt deps
   dbt debug
   ```

   `dbt debug` should report `All checks passed!`.

5. Install pre-commit hooks:

   ```sh
   pre-commit install
   ```

## Common Commands

```sh
dbt build                              # build all models and run all tests
dbt build --select staging             # build all staging models
dbt build --select my_model+           # build a model and all downstream
dbt build --select +my_model           # build a model and all upstream
dbt test --select my_model             # run tests for a specific model
dbt source freshness                   # check whether sources are up to date
```

## Quality Checks

Run project structure checks and custom rules:

```sh
dbt build --selector ci_quality_checks
```

## Project Structure

Models are organised into four layers:

| Layer | Folder | Prefix | Purpose |
|-------|--------|--------|---------|
| Staging | `models/staging/` | `stg_` | Clean and rename raw source data |
| Intermediate | `models/intermediate/` | `int_` | Business logic joins and transformations |
| Marts | `models/marts/` | `fct_`, `dim_` | Final analytics-ready tables |
| Utilities | `models/utilities/` | — | Date spines, shared lookup tables |
````

### PR Template

Create `.github/pull_request_template.md`:

```markdown
## Summary

<!-- What does this PR do? -->

## Changes

-

## Test Plan

- [ ] `dbt build --select state:modified+` runs without errors
- [ ] All new models have YAML documentation
- [ ] New tests pass locally
- [ ] `dbt build --selector ci_quality_checks` passes

## Notes

<!-- Any context reviewers need to know? Breaking changes? Deprecations? -->
```

### CODEOWNERS

Create `.github/CODEOWNERS`:

```
# Global owners
*       @your-org/data-platform-admins   @your-org/data-engineers
```

### Branch Protection

Go to **Settings** → **Branches** → **Add branch protection rule** for `main`:

✅ **Require a pull request before merging**
- Required approvals: **1**
- Require review from Code Owners
- Dismiss stale pull request approvals when new commits are pushed

✅ **Require status checks to pass before merging**
- Require branches to be up to date before merging
- Add the CI workflow check once CI is configured (see [dbt Core Deployment](9-dbt-core-deployment.md))

✅ **Require conversation resolution before merging**

✅ **Do not allow bypassing the above settings**

### Merge Settings

Go to **Settings** → **General** → **Pull Requests**:

- ❌ Uncheck "Allow merge commits"
- ❌ Uncheck "Allow rebase merging"
- ✅ Check "Allow squash merging"
- ✅ Enable "Always suggest updating pull request branches"
- ✅ Enable "Automatically delete head branches"

These are the same settings used across all repositories in this project. See [GitHub Setup](../../getting-started/initial-setup/1-github-setup.md) for the rationale.

## Set Up dbt

### Install

Install dbt and the Snowflake adapter into the project:

```sh
uv add dbt-snowflake
uv sync
```

`uv add` installs `dbt-snowflake` (which includes `dbt-core`) and records it in `pyproject.toml`. `uv sync` creates the `.venv` virtual environment.

### Configure direnv

[direnv](https://direnv.net/) activates the virtual environment and exports Snowflake credentials automatically when you `cd` into the project. Install it if you have not already — see [Local Environment](../../getting-started/initial-setup/2-local-environment.md).

Commit `.envrc.example` to the repository as a template for the team:

```sh
# .envrc.example — commit this file
# Copy to .envrc and fill in your values (never commit .envrc)

# Auto-activate the Python virtual environment
source .venv/bin/activate

# Snowflake credentials
export SNOWFLAKE_ACCOUNT="your-account.snowflakecomputing.com"
export SNOWFLAKE_USER="your.name@company.com"

# Your personal development schema prefix
# Schemas will be named e.g. DBT_JBLOGGS_STAGING, DBT_JBLOGGS_MARTS
export DBT_TARGET_SCHEMA="dbt_yourname"
```

Each developer copies `.envrc.example` to `.envrc` and fills in their own values:

```sh
cp .envrc.example .envrc
# Edit .envrc with your values
direnv allow
```

With `direnv allow`, the virtual environment activates automatically and `dbt` is available directly — no `uv run` prefix needed.

### Configure `profiles.yml`

`profiles.yml` references environment variables for all per-developer values, so it contains no secrets and **can be committed to the repository**.

Create `profiles.yml` in the project root:

```yaml
dbt_transform:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      authenticator: externalbrowser  # SSO login
      role: ANALYTICS_DEVELOPER
      warehouse: DEVELOPER
      database: ANALYTICS_DEV
      schema: "{{ env_var('DBT_TARGET_SCHEMA') }}"
      threads: 4

    prod:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: SVC_DBT
      private_key_path: "{{ env_var('SNOWFLAKE_PRIVATE_KEY_PATH') }}"
      private_key_passphrase: "{{ env_var('SNOWFLAKE_PRIVATE_KEY_PASSPHRASE', '') }}"
      role: SVC_DBT
      warehouse: TRANSFORMING
      database: ANALYTICS
      schema: ANALYTICS
      threads: 8
```

!!! tip "Developer authentication"
    Developers connect using SSO (`externalbrowser`) with their personal Snowflake user and the `ANALYTICS_DEVELOPER` role. The `SVC_DBT` service account with key-pair authentication is only used in CI/CD and production runs.

## Project Structure

Clone the new repository and initialise the dbt project:

```sh
git clone git@github.com:your-org/dbt-transform.git
cd dbt-transform
```

Use `dbt init` to scaffold the standard project structure:

```sh
dbt init . --skip-profile-setup
```

`dbt init` creates the standard directory layout and a `dbt_project.yml`. Remove the example models it generates and create the source-specific directories for this project:

```sh
# Remove generated examples
rm -rf models/example

# Create source-specific staging directories
mkdir -p models/staging/dlt
mkdir -p models/staging/snowpipe
mkdir -p models/staging/airbyte

# Create remaining directories
mkdir -p models/intermediate
mkdir -p models/utilities/dbt_project_evaluator
mkdir -p models/marts/core
mkdir -p models/marts/crm
mkdir -p models/marts/reporting
mkdir -p .github/workflows
```

The final layout:

```
dbt-transform/
├── .github/
│   ├── CODEOWNERS
│   ├── pull_request_template.md
│   └── workflows/
│       ├── ci.yml          ← slim CI on pull requests
│       └── deploy.yml      ← production run on merge to main
├── analyses/               ← ad-hoc analyses (not materialised)
├── macros/
│   └── generate_schema_name.sql
├── models/
│   ├── staging/
│   │   ├── dlt/
│   │   │   └── sources.yml
│   │   ├── snowpipe/
│   │   │   └── sources.yml
│   │   └── airbyte/
│   │       └── sources.yml
│   ├── intermediate/
│   ├── utilities/
│   └── marts/
│       ├── core/
│       ├── crm/
│       └── reporting/
├── seeds/
├── .envrc.example
├── .gitignore
├── .pre-commit-config.yaml
├── .sqlfluff
├── .sqlfluffignore
├── .yamllint
├── dbt_project.yml
├── packages.yml
├── profiles.yml            ← safe to commit — uses env vars only
├── README.md
└── selectors.yml
```

## Configure `.gitignore`

Create `.gitignore` to exclude generated files and credentials:

```gitignore
# dbt generated files
target/
dbt_packages/
logs/

# Local environment variables (never commit — contains personal credentials)
.envrc

# State artifacts (fetched at runtime, not stored in repo)
.artifacts/

# Python
__pycache__/
*.pyc
.venv/
```

!!! warning "Never commit `.envrc`"
    The `.envrc` file contains your personal Snowflake credentials. Always keep it in `.gitignore`. Commit `.envrc.example` instead so other developers have a template. In CI/CD, credentials are injected via GitHub Actions secrets — not a `.envrc` file.

## Configure `dbt_project.yml`

Replace the `dbt_project.yml` generated by `dbt init`:

```yaml
name: dbt_transform
version: '1.0.0'
config-version: 2

# Profile name — matches the profile in profiles.yml
profile: dbt_transform

# Paths
model-paths: ["models"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
analysis-paths: ["analyses"]
target-path: "target"

clean-targets:
  # These are the folders whose contents are deleted when `dbt clean` is run
  - "target"
  - "dbt_packages"

# Model configurations
models:
  dbt_transform:
    staging:
      +materialized: view
      +schema: staging
      dlt:
        +tags: ["staging", "dlt"]
      snowpipe:
        +tags: ["staging", "snowpipe"]
      airbyte:
        +tags: ["staging", "airbyte"]
    intermediate:
      +materialized: ephemeral
      +schema: intermediate
      +tags: ["intermediate"]
    utilities:
      +materialized: table
      +schema: utilities
      +tags: ["utilities"]
    marts:
      +materialized: table
      +schema: marts
      +tags: ["marts"]
      reporting:
        +schema: reporting
        +tags: ["reporting"]

# Seed configurations
seeds:
  dbt_transform:
    +schema: seeds

# Snapshot configurations
snapshots:
  dbt_transform:
    +schema: snapshots
    +strategy: timestamp
    +updated_at: updated_at
```

## Add the Custom Schema Macro

By default, dbt appends a custom schema to the target schema. This project overrides that behaviour: in production, only the custom schema name is used, giving clean schema names. In all other environments (dev, CI), the target schema is prepended so schemas are clearly scoped to each developer or PR.

Create `macros/generate_schema_name.sql`:

```sql
{% macro generate_schema_name(custom_schema_name, node) -%}

    {%- if custom_schema_name is none -%}

        {{ target.schema | trim }}

    {%- elif target.name == 'prod' -%}

        {{ custom_schema_name | trim | upper }}

    {%- else -%}

        {{ target.schema | trim }}_{{ custom_schema_name | trim | upper }}

    {%- endif -%}

{%- endmacro %}
```

With this macro:

| Environment | Target Name | Database | Target Schema | Model Config | Resulting Schema |
|-------------|------------|----------|--------------|--------------|-----------------|
| prod | `prod` | `ANALYTICS` | `ANALYTICS` | `+schema: staging` | `ANALYTICS.STAGING` |
| prod | `prod` | `ANALYTICS` | `ANALYTICS` | `+schema: reporting` | `ANALYTICS.REPORTING` |
| dev | `dev` | `ANALYTICS_DEV` | `dbt_jbloggs` | `+schema: staging` | `ANALYTICS_DEV.DBT_JBLOGGS_STAGING` |
| dev | `dev` | `ANALYTICS_DEV` | `dbt_jbloggs` | `+schema: reporting` | `ANALYTICS_DEV.DBT_JBLOGGS_REPORTING` |
| CI | `ci` | `ANALYTICS_DEV` | `CI_PR_123` | `+schema: staging` | `ANALYTICS_DEV.CI_PR_123_STAGING` |

!!! note "REPORTING schema in dev"
    In production, dbt writes to `ANALYTICS.REPORTING` — the Terraform-managed schema with BI tool access. In development and CI, dbt writes to scoped schemas such as `ANALYTICS_DEV.DBT_JBLOGGS_REPORTING` or `ANALYTICS_DEV.CI_PR_123_REPORTING`. These are created dynamically by dbt and do not affect the Terraform-managed production schema.

## Configure `packages.yml`

Create `packages.yml` at the repository root:

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: [">=1.3.0", "<2.0.0"]

  - package: metaplane/dbt_expectations
    version: [">=0.10.0", "<1.0.0"]

  - package: dbt-labs/dbt_project_evaluator
    version: [">=1.2.0", "<2.0.0"]
```

Install packages:

```sh
dbt deps
```

This creates a `dbt_packages/` directory (excluded by `.gitignore`). Always run `dbt deps` after cloning the repo or updating `packages.yml`.

### Package Overview

**dbt-utils** provides utility macros used throughout the project:

```sql
-- Generate a surrogate key from multiple columns
{{ dbt_utils.generate_surrogate_key(['date', 'currency']) }}

-- Date spine for building a date dimension
{{ dbt_utils.date_spine(
    datepart="day",
    start_date="cast('2020-01-01' as date)",
    end_date="current_date"
) }}

-- Union all tables matching a pattern
{{ dbt_utils.union_relations(
    relations=[ref('stg_dlt__exchange_rates'), ref('stg_snowpipe__exchange_rates')]
) }}
```

**dbt_expectations** adds column-level tests beyond the four built-in generic tests:

```yaml
columns:
  - name: rate
    tests:
      - dbt_expectations.expect_column_values_to_be_between:
          min_value: 0
          strictly: true
      - dbt_expectations.expect_column_values_to_not_be_null
```

**dbt_project_evaluator** enforces project structure rules in CI. Full configuration is covered in [Testing and Documentation](8-testing-and-documentation.md#dbt_project_evaluator).

## Linting and Formatting

Two tools enforce code quality across SQL and YAML files:

- **sqlfluff** — lints and auto-fixes SQL, using the dbt templater so it understands `{{ ref() }}` and `{{ source() }}` expressions
- **yamllint** — lints YAML files including `dbt_project.yml`, `packages.yml`, and schema files

Both are run automatically on commit via [pre-commit](https://pre-commit.com/). pre-commit itself is already installed globally — see [Local Environment](../../getting-started/initial-setup/2-local-environment.md). This section adds sqlfluff and yamllint as project-level dev dependencies.

### Install Dev Tools

```sh
uv add --dev sqlfluff yamllint
```

These are added as dev dependencies in `pyproject.toml` and are available after `uv sync`.

### Configure sqlfluff

Create `.sqlfluff` in the project root:

```ini
[sqlfluff]
templater = dbt
dialect = snowflake

[sqlfluff:templater:dbt]
project_dir = .

[sqlfluff:rules:capitalisation.keywords]
capitalisation_policy = upper

[sqlfluff:rules:capitalisation.identifiers]
capitalisation_policy = upper

[sqlfluff:rules:capitalisation.functions]
capitalisation_policy = upper

[sqlfluff:rules:capitalisation.literals]
capitalisation_policy = upper
```

Create `.sqlfluffignore` to exclude generated directories:

```
dbt_packages/
target/
```

### Configure yamllint

Create `.yamllint` in the project root:

```yaml
extends: default
rules:
  line-length:
    max: 120
    level: warning
  truthy:
    allowed-values: ['true', 'false']
```

### Configure pre-commit

Create `.pre-commit-config.yaml` in the project root:

```yaml
repos:
  - repo: https://github.com/sqlfluff/sqlfluff
    rev: 3.4.0
    hooks:
      - id: sqlfluff-lint
      - id: sqlfluff-fix
        args: [--force]

  - repo: https://github.com/adrienverge/yamllint
    rev: v1.35.1
    hooks:
      - id: yamllint
```

!!! tip "Keep hook versions up to date"
    Run `pre-commit autoupdate` periodically to bump the `rev` values to the latest releases.

Install the hooks into your local git repository:

```sh
pre-commit install
```

From this point on, sqlfluff and yamllint run automatically on every `git commit`. Files that fail linting are flagged before the commit proceeds; sqlfluff will also attempt to auto-fix fixable issues.

### Run Manually

To lint all files at any time without committing:

```sh
pre-commit run --all-files
```

To lint or fix SQL files directly:

```sh
# Lint all models
sqlfluff lint models/

# Fix all models
sqlfluff fix models/
```

## Verify the Setup

With dbt installed and credentials configured, verify the project:

```sh
dbt deps    # install packages
dbt debug   # check connection and project configuration
```

Expected output from `dbt debug`:

```
dbt version: 1.8.x
python version: 3.11.x
...
Connection test: [OK connection ok]

All checks passed!
```

## Commit the Initial Structure

```sh
git add .
git commit -m "Initial dbt project structure

- README with developer setup guide
- .github/pull_request_template.md
- .github/CODEOWNERS
- dbt_project.yml with custom schema naming
- packages.yml (dbt-utils, dbt_expectations, dbt_project_evaluator)
- profiles.yml using env vars (safe to commit — no secrets)
- .envrc.example template for local developer setup
- Custom generate_schema_name macro
- .sqlfluff, .sqlfluffignore, .yamllint, .pre-commit-config.yaml
- .gitignore excluding .envrc and generated files"
git push
```

## Summary

You've set up the `dbt-transform` repository:

- [x] Created the repository with README, PR template, CODEOWNERS, and branch protection
- [x] Installed dbt locally and configured credentials via direnv and `profiles.yml`
- [x] Initialised the project with `dbt init`
- [x] Configured `dbt_project.yml` with custom schema naming
- [x] Added packages: dbt-utils, dbt_expectations, dbt_project_evaluator
- [x] Added the custom schema macro for clean schema names
- [x] Configured sqlfluff and yamllint with pre-commit hooks

## What's Next

Create the Snowflake service account that dbt will use for production runs.

Continue to [Snowflake Infrastructure](4-snowflake-infrastructure.md) →
