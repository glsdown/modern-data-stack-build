# Project Setup

On this page, you will:

- [x] Expand the data pipelines repository with dlt structure
- [x] Set up the project structure for dlt sources and pipelines
- [x] Configure development dependencies with uv
- [x] Build a custom secrets provider for AWS Secrets Manager

## Overview

Your data pipelines live in the `data-pipelines` repository you created in the [Orchestration](../orchestration/7-first-flow.md) section. Now you'll expand it with dlt sources, pipelines, and utilities alongside the existing Prefect flows. This structure follows the [dlt deployment guide](https://dlthub.com/docs/walkthroughs/deploy-a-pipeline/deploy-with-prefect) recommendation of keeping dlt and Prefect code together.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     data-pipelines Repository                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  sources/                  dlt source definitions                           │
│  ├── exchange_rates/       API extraction logic                             │
│  ├── currencies/           API extraction logic                             │
│  └── products/             Database extraction logic                        │
│                                                                             │
│  pipelines/                dlt pipeline configurations                      │
│  ├── exchange_rates.py     Pipeline: API → S3 → Snowpipe → Snowflake       │
│  ├── currencies.py         Pipeline: API → Snowflake                        │
│  └── products.py           Pipeline: PostgreSQL → Snowflake                 │
│                                                                             │
│  flows/                    Prefect flow definitions                         │
│  ├── exchange_rates.py     Scheduled flow for exchange rates                │
│  ├── currencies.py         Scheduled flow for currencies                    │
│  └── products.py           Scheduled flow for products                      │
│                                                                             │
│  utils/                    Shared utilities                                 │
│  └── vault_provider.py     AWS Secrets Manager provider for dlt             │
│                                                                             │
│  prefect.yaml              Prefect deployment configuration                 │
│  pyproject.toml            Project dependencies (managed by uv)             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Use the Existing Repository

You already created the `data-pipelines` repository and deployed your first flow in [Your First Flow](../orchestration/7-first-flow.md). Navigate to it now:

```sh
cd data-pipelines
```

The repository currently has a basic `flows/` directory and `prefect.yaml`. You'll expand it with dlt sources, pipelines, and utilities.

## Project Structure

Create the directory structure:

```sh
mkdir -p sources/exchange_rates sources/currencies sources/products
mkdir -p pipelines flows utils
touch sources/__init__.py sources/exchange_rates/__init__.py
touch sources/currencies/__init__.py sources/products/__init__.py
touch pipelines/__init__.py flows/__init__.py utils/__init__.py
```

Your structure should look like:

```
data-pipelines/
├── sources/
│   ├── __init__.py
│   ├── exchange_rates/
│   │   ├── __init__.py
│   │   └── source.py
│   ├── currencies/
│   │   ├── __init__.py
│   │   └── source.py
│   └── products/
│       ├── __init__.py
│       └── source.py
├── pipelines/
│   ├── __init__.py
│   ├── exchange_rates.py
│   ├── currencies.py
│   └── products.py
├── flows/
│   ├── __init__.py
│   ├── exchange_rates.py
│   ├── currencies.py
│   └── products.py
├── utils/
│   ├── __init__.py
│   └── vault_provider.py
├── .dlt/
│   ├── config.toml
│   └── secrets.toml.example
├── prefect.yaml
├── pyproject.toml
├── uv.lock
├── .gitignore
└── README.md
```

## Dependencies

This project uses [uv](https://docs.astral.sh/uv/) for dependency management. If you followed the orchestration section, uv is already installed and a `pyproject.toml` exists. Add the dlt dependencies:

```sh
# dlt with Snowflake and S3 (filesystem) destinations
uv add "dlt[snowflake]" "dlt[filesystem]"

# dlt SQL database source (for PostgreSQL extraction)
uv add "dlt[sql_database]"

# Prefect orchestration (already installed if you followed the orchestration section)
uv add prefect prefect-aws prefect-snowflake

# AWS SDK for Secrets Manager
uv add boto3

# PostgreSQL driver
uv add psycopg2-binary
```

!!! tip "Why `uv add` Instead of `pip install`?"
    `uv add` updates your `pyproject.toml` and `uv.lock` files, ensuring reproducible installs. The lock file pins exact versions so every environment — local, CI/CD, and production — uses the same dependency tree. You never need to activate a virtual environment manually; `uv run` handles it automatically.

For local testing with DuckDB (a lightweight in-process database):

```sh
uv add --dev "dlt[duckdb]"
```

The `--dev` flag adds DuckDB as a development-only dependency. It won't be installed in production.

Your `pyproject.toml` dependencies section should look similar to:

```toml
[project]
name = "data-pipelines"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "boto3>=1.34.0",
    "dlt[filesystem]>=0.4.0",
    "dlt[snowflake]>=0.4.0",
    "dlt[sql_database]>=0.4.0",
    "prefect>=3.0.0",
    "prefect-aws>=0.5.0",
    "prefect-snowflake>=0.3.0",
    "psycopg2-binary>=2.9.0",
]

[dependency-groups]
dev = [
    "dlt[duckdb]>=0.4.0",
]
```

## dlt Configuration

Create the dlt configuration directory and files:

```sh
mkdir -p .dlt
```

Create `.dlt/config.toml` for non-sensitive configuration:

```toml
[runtime]
log_level = "INFO"

[normalize]
# Flatten nested JSON structures
max_table_nesting = 1
```

Create `.dlt/secrets.toml.example` as a template (the actual `secrets.toml` should not be committed):

```toml
# Copy this file to secrets.toml and fill in your values.
# DO NOT commit secrets.toml to version control.
#
# For local development, use your own Snowflake credentials
# (not the service account). This gives you ANALYTICS_DEVELOPER
# permissions for testing.

[destination.snowflake.credentials]
database = "DLT"
warehouse = "LOADING"
role = "ANALYTICS_DEVELOPER"
username = "YOUR_SNOWFLAKE_USERNAME"
password = "your-password"
host = "orgname-accountname.snowflakecomputing.com"

[destination.filesystem]
bucket_url = "s3://your-data-lake-bucket/dlt"

[destination.filesystem.credentials]
aws_access_key_id = "your-access-key"
aws_secret_access_key = "your-secret-key"
region_name = "eu-west-2"

[sources.open_exchange_rates]
api_key = "your-api-key-here"

[sources.clever_cloud]
host = "xxx.postgresql.clever-cloud.com"
port = 5432
database = "xxx"
username = "xxx"
password = "xxx"
```

!!! info "Local Credentials vs Production"
    For local development, use your own Snowflake username and the `ANALYTICS_DEVELOPER` role. This gives you read/write access to the schemas you need for testing without sharing the service account credentials. In production, the pipeline runs as `SVC_DLT` with its dedicated role — credentials are fetched automatically from AWS Secrets Manager via the vault provider you'll build next.

### Environment Variable Naming

dlt resolves configuration from multiple sources in priority order: environment variables, `secrets.toml`, `config.toml`, and custom providers. Environment variables follow a specific naming convention — sections are separated by double underscores and names are capitalised:

| Config Path | Environment Variable |
|-------------|---------------------|
| `destination.snowflake.credentials.database` | `DESTINATION__SNOWFLAKE__CREDENTIALS__DATABASE` |
| `destination.snowflake.credentials.password` | `DESTINATION__SNOWFLAKE__CREDENTIALS__PASSWORD` |
| `destination.filesystem.bucket_url` | `DESTINATION__FILESYSTEM__BUCKET_URL` |
| `sources.open_exchange_rates.api_key` | `SOURCES__OPEN_EXCHANGE_RATES__API_KEY` |

This convention is useful in CI/CD and container environments where you set configuration via environment variables rather than files.

## AWS Secrets Manager Provider

In production, your pipelines run on ephemeral infrastructure (Prefect workers, ECS tasks) where local `secrets.toml` files don't exist. Rather than setting dozens of environment variables, you can integrate dlt directly with AWS Secrets Manager using a custom configuration provider.

dlt's configuration system supports custom providers that plug into its resolution chain. When dlt needs a configuration value (like `destination.snowflake.credentials.password`), it queries each registered provider in order until one returns a value.

Create `utils/vault_provider.py`:

```python
"""Custom dlt configuration provider for AWS Secrets Manager."""

import json
import logging
from functools import lru_cache
from typing import Any

import boto3
from botocore.exceptions import ClientError
from dlt.common.configuration.providers import ConfigProvider

logger = logging.getLogger(__name__)

# Maps dlt config paths to AWS Secrets Manager secret names.
# When dlt resolves a config value, it passes the section path
# (e.g. "destination", "snowflake", "credentials") and the key
# (e.g. "password"). This map matches section paths to secrets.
SECRET_MAP = {
    "destination.snowflake.credentials": "dlt/snowflake-credentials",
    "destination.filesystem.credentials": "dlt/s3-credentials",
    "sources.open_exchange_rates": "dlt/open-exchange-rates",
    "sources.clever_cloud": "dlt/clever-cloud-postgres",
    "sources.hubspot": "dlt/hubspot-api-key",
}


@lru_cache(maxsize=32)
def _fetch_secret(secret_name: str, region: str) -> dict[str, Any]:
    """Fetch and cache a secret from AWS Secrets Manager.

    Results are cached for the lifetime of the process to avoid
    repeated API calls within a single pipeline run.
    """
    client = boto3.client("secretsmanager", region_name=region)
    response = client.get_secret_value(SecretId=secret_name)
    secret_string = response["SecretString"]
    try:
        return json.loads(secret_string)
    except json.JSONDecodeError:
        return {"value": secret_string}


class AWSSecretsManagerProvider(ConfigProvider):
    """Resolves dlt configuration values from AWS Secrets Manager.

    This provider maps dlt's dotted config paths to secret names
    in AWS Secrets Manager. For example, when dlt looks up
    ``destination.snowflake.credentials.password``, this provider:

    1. Joins the sections into ``destination.snowflake.credentials``
    2. Looks up the secret name in SECRET_MAP → ``dlt/snowflake-credentials``
    3. Fetches the secret JSON from AWS
    4. Returns the ``password`` field from that JSON
    """

    def __init__(self, region: str = "eu-west-2"):
        self.region = region
        super().__init__()

    @property
    def name(self) -> str:
        return "AWS Secrets Manager"

    @property
    def supports_secrets(self) -> bool:
        return True

    def get_value(
        self, key: str, hint: Any = None, pipeline_name: str = None, *sections: str
    ) -> tuple[Any, str]:
        """Look up a configuration value in AWS Secrets Manager.

        Args:
            key: The config key (e.g. "password", "api_key")
            hint: Type hint for the value
            pipeline_name: Current pipeline name (unused)
            *sections: Config path sections (e.g. "destination", "snowflake", "credentials")

        Returns:
            Tuple of (value, secret_name) or (None, None) if not found
        """
        full_path = ".".join(sections)
        secret_name = SECRET_MAP.get(full_path)
        if not secret_name:
            return None, None

        try:
            secret_data = _fetch_secret(secret_name, self.region)
        except ClientError:
            logger.debug("Secret '%s' not found in AWS Secrets Manager", secret_name)
            return None, None

        value = secret_data.get(key)
        if value is not None:
            return value, secret_name

        return None, None


def register_aws_secrets(region: str = "eu-west-2") -> None:
    """Register the AWS Secrets Manager provider with dlt.

    Call this at the start of any pipeline that should resolve
    secrets from AWS. The provider is added to the end of the
    chain, so local secrets.toml and environment variables still
    take priority (useful for development overrides).
    """
    from dlt.common.configuration.container import Container
    from dlt.common.configuration.providers import ConfigProvidersContext

    provider = AWSSecretsManagerProvider(region=region)
    ctx = Container()[ConfigProvidersContext]
    ctx.providers.append(provider)
```

### How It Works

When a pipeline calls `register_aws_secrets()`, the provider is added to dlt's configuration resolution chain:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    dlt Configuration Resolution Order                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. Environment variables      DESTINATION__SNOWFLAKE__CREDENTIALS__PASSWORD│
│  2. secrets.toml               [destination.snowflake.credentials]          │
│  3. config.toml                [runtime] log_level                          │
│  4. AWS Secrets Manager ★      dlt/snowflake-credentials → {"password":..} │
│                                                                             │
│  dlt queries each provider in order. The first to return a value wins.     │
│  This means local secrets.toml always overrides AWS for development.       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

In production (no `secrets.toml` present), dlt falls through to the AWS provider. In local development, your `secrets.toml` values take priority — you never need to configure AWS credentials on your laptop.

### Using the Provider

In your pipeline code, register the provider before creating the pipeline:

```python
from utils.vault_provider import register_aws_secrets

# Register AWS Secrets Manager (no-op locally if secrets.toml exists)
register_aws_secrets()

pipeline = dlt.pipeline(
    pipeline_name="currencies",
    destination="snowflake",
    dataset_name="open_exchange_rates",
)
```

The secret JSON stored in AWS Secrets Manager should match the dlt configuration keys. For example, the `dlt/snowflake-credentials` secret:

```json
{
    "database": "DLT",
    "warehouse": "LOADING",
    "role": "SVC_DLT",
    "username": "SVC_DLT",
    "password": "the-password",
    "host": "orgname-accountname.snowflakecomputing.com"
}
```

You'll create these secrets in [Snowflake Infrastructure](3-snowflake-infrastructure.md).

## Prefect Configuration

Create `prefect.yaml` for deployment configuration:

```yaml
name: data-pipelines

# How Prefect retrieves flow code
pull:
  - prefect.deployments.steps.git_clone:
      repository: https://github.com/YOUR-ORG/data-pipelines.git
      branch: main

# Deployments are defined in later pages
deployments: []
```

We'll add deployments in the [Prefect Orchestration](8-prefect-orchestration.md) page.

## Git Configuration

Create `.gitignore`:

```gitignore
# Python
__pycache__/
*.py[cod]
*$py.class
.venv/
venv/
*.egg-info/

# dlt
.dlt/secrets.toml
.dlt/pipeline_*/

# Environment
.env
.envrc

# IDE
.idea/
.vscode/
*.swp

# OS
.DS_Store
Thumbs.db
```

## Local Development

For local development, create a `.dlt/secrets.toml` file with your credentials (this file is gitignored):

```sh
cp .dlt/secrets.toml.example .dlt/secrets.toml
# Edit .dlt/secrets.toml with your actual credentials
```

Test that dlt is working with a local DuckDB pipeline:

```python
# test_dlt.py
import dlt


@dlt.resource
def test_data():
    yield {"id": 1, "name": "test"}
    yield {"id": 2, "name": "another test"}


pipeline = dlt.pipeline(
    pipeline_name="test",
    destination="duckdb",
    dataset_name="test_data",
)

load_info = pipeline.run(test_data())
print(load_info)
print("dlt is working!")
```

```sh
python test_dlt.py
```

Expected output:

```
Pipeline test completed in 0.XX seconds
1 load package(s) were loaded to destination duckdb and target schema test_data
The duckdb destination used duckdb:////...data-pipelines/test.duckdb location to store data
dlt is working!
```

Clean up the test file and DuckDB database:

```sh
rm test_dlt.py test.duckdb
```

!!! tip "Switching Destinations"
    During development, you can switch between DuckDB (local) and Snowflake by changing the `destination` parameter in your pipeline code — or by setting the `DESTINATION__NAME` environment variable. The rest of the code stays the same. This makes it easy to develop and test locally before deploying to production.

## Initial Commit

Commit the project structure:

```sh
git add .
git commit -m "Add dlt project structure and dependencies"
git push origin main
```

## Summary

You've expanded the data pipelines repository:

- [x] Extended repository with dlt + Prefect structure
- [x] Source, pipeline, and flow directories
- [x] Dependencies managed with uv and `pyproject.toml`
- [x] dlt configuration files with environment variable naming
- [x] AWS Secrets Manager provider for production credentials
- [x] DuckDB for local testing

## What's Next

Before building pipelines, you need to set up the Snowflake infrastructure — databases, roles, and the dlt service account.

Continue to [Snowflake Infrastructure](3-snowflake-infrastructure.md) →
