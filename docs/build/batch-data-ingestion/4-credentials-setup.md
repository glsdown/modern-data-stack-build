# Credentials Setup

On this page, you will:

- [x] Set secret values in AWS Secrets Manager
- [x] Configure local development credentials
- [x] Verify the VaultDocProvider resolves credentials
- [x] Set up IAM permissions for production access

## Overview

Your dlt pipelines need credentials for data sources (APIs, databases) and destinations (Snowflake, S3). In [Snowflake Infrastructure](3-snowflake-infrastructure.md), you created the secret containers in AWS Secrets Manager and generated the SVC_DLT key pair. Now you'll set the remaining secret values and configure local development.

The credentials flow differs between local development and production:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CREDENTIALS FLOW                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Development (Local)                  Production (Prefect Worker)           │
│  ┌─────────────────────┐              ┌─────────────────────┐               │
│  │ .dlt/secrets.toml   │              │ AWS Secrets Manager │               │
│  │ (your own user)     │              │ (SVC_DLT service    │               │
│  │                     │              │  account)           │               │
│  └──────────┬──────────┘              └──────────┬──────────┘               │
│             │                                    │                          │
│             ▼                                    ▼                          │
│  ┌─────────────────────────────────────────────────────────────────┐        │
│  │              dlt Configuration Resolution                       │        │
│  │  1. Environment variables                                       │        │
│  │  2. .dlt/secrets.toml  ← Local dev wins here                    │        │
│  │  3. .dlt/config.toml                                            │        │
│  │  4. AWSSecretsManagerProvider  ← Production falls through here  │        │
│  └─────────────────────────────────────────────────────────────────┘        │
│             │                                                               │
│             ▼                                                               │
│  ┌─────────────────────────────────────────────────────────────────┐        │
│  │                      dlt Pipeline                               │        │
│  └─────────────────────────────────────────────────────────────────┘        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

In production, there is no `secrets.toml`, so dlt falls through to the `AWSSecretsManagerProvider` you built in [Project Setup](2-project-setup.md). Locally, your `secrets.toml` values take priority — you never need AWS credentials on your development machine.

## Prerequisites

Ensure you have:

- [x] [Snowflake Infrastructure](3-snowflake-infrastructure.md) — Secret containers created, SVC_DLT key pair stored
- [x] Open Exchange Rates account (free tier at [openexchangerates.org](https://openexchangerates.org))
- [x] Clever Cloud PostgreSQL database (free DEV tier)

## Set Secret Values

The secret containers were created in [Snowflake Infrastructure](3-snowflake-infrastructure.md). The Snowflake SVC_DLT credentials are already stored. Now set the remaining values.

### Open Exchange Rates API Key

1. Sign up at [openexchangerates.org](https://openexchangerates.org/signup) (the free tier includes 1,000 requests/month)
2. Copy your App ID from the dashboard

```sh
aws secretsmanager put-secret-value \
    --secret-id "dlt/open-exchange-rates" \
    --secret-string '{"api_key": "YOUR_APP_ID_HERE"}' \
    --profile infrastructure-admin
```

### Clever Cloud PostgreSQL Credentials

1. Sign up at [clever-cloud.com](https://www.clever-cloud.com/)
2. Create a PostgreSQL add-on (free DEV tier)
3. Note the connection details from the add-on dashboard

```sh
aws secretsmanager put-secret-value \
    --secret-id "dlt/clever-cloud-postgres" \
    --secret-string '{
        "host": "xxx.postgresql.clever-cloud.com",
        "port": 5432,
        "database": "xxx",
        "username": "xxx",
        "password": "xxx"
    }' \
    --profile infrastructure-admin
```

!!! warning "Replace Placeholder Values"
    Replace the `xxx` values with your actual Clever Cloud credentials from the add-on dashboard.

### HubSpot API Key (Optional)

If you plan to use dlt for HubSpot ingestion (covered in [HubSpot Pipeline](9-hubspot-pipeline.md)):

```sh
aws secretsmanager put-secret-value \
    --secret-id "dlt/hubspot-api-key" \
    --secret-string '{"api_key": "YOUR_HUBSPOT_API_KEY"}' \
    --profile infrastructure-admin
```

## Store in 1Password (Optional)

For team access and backup, also store credentials in 1Password. When someone needs to rotate a credential, they can copy the full JSON string and paste it directly into the `put-secret-value` command.

| Item Name | Fields |
|-----------|--------|
| Open Exchange Rates API | `api_key` |
| Clever Cloud PostgreSQL | `host`, `port`, `database`, `username`, `password` |
| Snowflake SVC_DLT | `database`, `warehouse`, `role`, `username`, `host`, `private_key` |

## Local Development Setup

For local development, you use your own Snowflake credentials rather than the SVC_DLT service account. This gives you `ANALYTICS_DEVELOPER` permissions for testing without sharing production credentials.

Navigate to your data pipelines repository and update the `.dlt/secrets.toml` you created in [Project Setup](2-project-setup.md):

```sh
cd ~/projects/data/data-pipelines
```

Edit `.dlt/secrets.toml`:

```toml
# DO NOT COMMIT THIS FILE
# Use your own Snowflake credentials for local development.

# -----------------------------------------------------------------------------
# Data Sources
# -----------------------------------------------------------------------------
[sources.open_exchange_rates]
api_key = "your-openexchangerates-app-id"

[sources.clever_cloud]
host = "xxx.postgresql.clever-cloud.com"
port = 5432
database = "xxx"
username = "xxx"
password = "xxx"

# -----------------------------------------------------------------------------
# Destination: Snowflake
# -----------------------------------------------------------------------------
# Use your own credentials, not SVC_DLT.
# ANALYTICS_DEVELOPER has read/write access to the DLT schemas.
[destination.snowflake.credentials]
database = "DLT"
warehouse = "LOADING"
role = "ANALYTICS_DEVELOPER"
username = "YOUR_SNOWFLAKE_USERNAME"
password = "your-password"
host = "orgname-accountname.snowflakecomputing.com"

# -----------------------------------------------------------------------------
# Destination: Filesystem (S3)
# -----------------------------------------------------------------------------
[destination.filesystem]
bucket_url = "s3://your-project-data-lake-prod/dlt"

[destination.filesystem.credentials]
aws_access_key_id = "your-access-key"
aws_secret_access_key = "your-secret-key"
region_name = "eu-west-2"
```

!!! info "Your Own Credentials"
    Using your personal Snowflake account for local development means:

    - You can trace queries in Snowflake's query history to your username
    - You don't need the SVC_DLT private key on your laptop
    - You can use password authentication (simpler than key-pair for development)
    - `ANALYTICS_DEVELOPER` has the necessary read/write permissions via `ANALYTICS_SOURCES_READER`

!!! tip "Snowflake Account Format"
    Use the `orgname-accountname` format for the Snowflake host. You can find this in your Snowflake URL:

    ```
    https://orgname-accountname.snowflakecomputing.com
    ```

    For example, if your URL is `https://myorg-myaccount.snowflakecomputing.com`, set `host = "myorg-myaccount.snowflakecomputing.com"`.

### S3 Bucket URL

The `bucket_url` in `secrets.toml` should point to your S3 data lake bucket with a `dlt` prefix. This keeps dlt output files organised within the bucket. When you build the exchange rates pipeline (which writes to S3), you'll override this with a more specific path.

For local development, you can also use the AWS CLI profile credentials. If `aws_access_key_id` and `aws_secret_access_key` are omitted, dlt falls back to the default AWS credential chain (environment variables → AWS profile → instance profile).

## IAM Permissions for Production

The Prefect worker (or ECS task) needs IAM permissions to read dlt secrets. Add a policy scoped to the `dlt/*` prefix:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "ReadDltSecrets",
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue",
                "secretsmanager:DescribeSecret"
            ],
            "Resource": "arn:aws:secretsmanager:eu-west-2:ACCOUNT_ID:secret:dlt/*"
        }
    ]
}
```

If using ECS, attach this to the task execution role. If using EC2, attach it to the instance profile. The `dlt/*` prefix ensures the worker can only read dlt pipeline secrets, not Terraform or Airbyte secrets.

!!! info "Prefix-Based Access Control"
    The [Secrets Manager Setup](../../getting-started/terraform-setup/aws/6-secrets-manager-setup.md) page explains the full prefix-based access control pattern. Each tool (`dlt/*`, `airbyte/*`, `prefect/*`, `terraform/*`) gets its own read policy scoped to its prefix.

## Verify Secrets

Test that secrets are accessible from the CLI:

```sh
# Test Open Exchange Rates (show first 8 characters of API key)
aws secretsmanager get-secret-value \
    --secret-id "dlt/open-exchange-rates" \
    --query SecretString \
    --output text \
    --profile infrastructure-admin | jq -r '.api_key' | head -c 8

# Test Clever Cloud (show host only)
aws secretsmanager get-secret-value \
    --secret-id "dlt/clever-cloud-postgres" \
    --query SecretString \
    --output text \
    --profile infrastructure-admin | jq -r '.host'

# Test Snowflake (show username only)
aws secretsmanager get-secret-value \
    --secret-id "dlt/snowflake-credentials" \
    --query SecretString \
    --output text \
    --profile infrastructure-admin | jq -r '.username'
```

## Test Local Configuration

Test that your local `secrets.toml` works with dlt:

```python
# test_credentials.py
import dlt

# Test Snowflake destination credentials
pipeline = dlt.pipeline(
    pipeline_name="test_credentials",
    destination="snowflake",
    dataset_name="test",
)

# dlt resolves credentials from secrets.toml automatically
print(f"Pipeline destination: {pipeline.destination}")
print("Credentials loaded successfully!")
```

```sh
python test_credentials.py
rm test_credentials.py
```

### Test the VaultDocProvider

To verify the VaultDocProvider works with AWS (requires AWS credentials configured):

```python
# test_vault_provider.py
from utils.vault_provider import register_aws_secrets, _fetch_secret

# Test direct secret fetch
try:
    secret = _fetch_secret("dlt/open-exchange-rates", "eu-west-2")
    print(f"Open Exchange Rates API key starts with: {secret['api_key'][:8]}...")
    print("VaultDocProvider can reach AWS Secrets Manager!")
except Exception as e:
    print(f"AWS access not configured (expected locally): {e}")

# Test registration (should not error even without AWS)
register_aws_secrets()
print("Provider registered successfully.")
```

```sh
python test_vault_provider.py
rm test_vault_provider.py
```

!!! tip "Expected Behaviour Locally"
    If you don't have AWS credentials configured on your laptop, the VaultDocProvider will fail silently (it returns `None` and dlt moves to the next provider). This is by design — your `secrets.toml` provides the values instead.

## How dlt Resolves Nested Configuration

When dlt encounters nested JSON from an API — for example, the Open Exchange Rates `/latest.json` endpoint returns a `rates` object with currency codes as keys — it flattens the structure into columns:

```json
{
    "base": "USD",
    "rates": {
        "GBP": 0.79,
        "EUR": 0.92,
        "JPY": 149.5
    }
}
```

With `max_table_nesting = 1` (set in `.dlt/config.toml`), dlt creates individual columns for each nested key (`rates__gbp`, `rates__eur`, `rates__jpy`). If the nesting level were higher, dlt would create a separate child table instead. This is important to understand when designing your pipeline schemas.

## Summary

You've configured credentials for all dlt data sources and destinations:

- [x] Open Exchange Rates API key stored in AWS Secrets Manager
- [x] Clever Cloud PostgreSQL credentials stored in AWS Secrets Manager
- [x] Snowflake SVC_DLT key-pair credentials (stored in previous page)
- [x] Local development `secrets.toml` with your own Snowflake credentials
- [x] IAM permissions scoped to `dlt/*` prefix
- [x] VaultDocProvider verified for production use

## What's Next

With credentials in place, you can start building the dlt pipelines. The first pipeline loads currency reference data from the Open Exchange Rates API directly into Snowflake.

Continue to [Currencies Pipeline](5-currencies-pipeline.md) →
