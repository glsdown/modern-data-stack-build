# Secrets and Blocks

On this page, you will:

- [x] Understand Prefect Blocks for configuration management
- [x] Create blocks via Terraform
- [x] Integrate with AWS Secrets Manager
- [x] Connect to your S3 data lake buckets

## Overview

Prefect **Blocks** are reusable configuration objects that store connection details, credentials, and settings for external systems. They keep sensitive data out of your code and make flows portable across environments.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          PREFECT BLOCKS                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐              │
│  │ AWS Credentials │  │   Snowflake     │  │    S3 Bucket    │              │
│  │                 │  │   Connector     │  │                 │              │
│  │  (IAM Role)     │  │  account        │  │  bucket_name    │              │
│  │                 │  │  user           │  │  credentials    │              │
│  │                 │  │  password       │  │                 │              │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘              │
│           │                    │                    │                       │
│           └────────────────────┼────────────────────┘                       │
│                                │                                            │
│                                ▼                                            │
│                    ┌─────────────────────┐                                  │
│                    │     Your Flows      │                                  │
│                    │  block = Block.load │                                  │
│                    └─────────────────────┘                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Creating Blocks via Terraform

Blocks can be managed via Terraform using the Prefect provider, keeping your infrastructure as code.

### Add Blocks to Prefect Terraform

Update your Prefect Terraform configuration to include blocks. Add `blocks.tf` to `terraform/prefect/config/`:

```hcl
# =============================================================================
# Prefect Blocks
# =============================================================================
# Blocks for connecting to external systems.

# -----------------------------------------------------------------------------
# Data Sources - Reference AWS Resources
# -----------------------------------------------------------------------------
# Reference the S3 data lake buckets created in AWS Terraform

data "terraform_remote_state" "aws" {
  backend = "s3"
  config = {
    bucket = "your-terraform-state-bucket"
    key    = "aws/terraform.tfstate"
    region = "eu-west-2"
  }
}

# -----------------------------------------------------------------------------
# AWS Credentials Block
# -----------------------------------------------------------------------------
# Uses IAM roles - no access keys needed when running on AWS
resource "prefect_block" "aws_credentials" {
  name      = "data-platform"
  type_slug = "aws-credentials"

  data = jsonencode({
    region_name = var.aws_region
    # When running on ECS/EC2 with IAM roles, credentials are automatic
    # No access keys stored in the block
  })
}

# -----------------------------------------------------------------------------
# S3 Data Lake Blocks
# -----------------------------------------------------------------------------
# Create blocks for each data lake bucket

resource "prefect_block" "s3_data_lake_dev" {
  name      = "data-lake-dev"
  type_slug = "s3-bucket"

  data = jsonencode({
    bucket_name       = data.terraform_remote_state.aws.outputs.data_lake_buckets["dev"].id
    credentials_block = prefect_block.aws_credentials.id
  })
}

resource "prefect_block" "s3_data_lake_staging" {
  name      = "data-lake-staging"
  type_slug = "s3-bucket"

  data = jsonencode({
    bucket_name       = data.terraform_remote_state.aws.outputs.data_lake_buckets["staging"].id
    credentials_block = prefect_block.aws_credentials.id
  })
}

resource "prefect_block" "s3_data_lake_prod" {
  name      = "data-lake-prod"
  type_slug = "s3-bucket"

  data = jsonencode({
    bucket_name       = data.terraform_remote_state.aws.outputs.data_lake_buckets["prod"].id
    credentials_block = prefect_block.aws_credentials.id
  })
}

# -----------------------------------------------------------------------------
# Outputs
# -----------------------------------------------------------------------------
output "block_aws_credentials" {
  description = "AWS credentials block name"
  value       = prefect_block.aws_credentials.name
}

output "block_s3_data_lake_dev" {
  description = "S3 data lake dev block name"
  value       = prefect_block.s3_data_lake_dev.name
}

output "block_s3_data_lake_prod" {
  description = "S3 data lake prod block name"
  value       = prefect_block.s3_data_lake_prod.name
}
```

### Deploy the Blocks

Commit and push to deploy via CI/CD:

```sh
git add terraform/prefect/config/blocks.tf
git commit -m "Add Prefect blocks for AWS and S3"
git push
```

### Verify Blocks

After CI/CD completes:

```sh
# List all blocks
prefect block ls

# Check specific block types
prefect block ls --type s3-bucket
```

## Using Blocks in Flows

### S3 Block

```python
from prefect import flow, task
from prefect_aws import S3Bucket
import json


@task
def upload_to_s3(data: dict, key: str):
    """Upload data to S3."""
    s3 = S3Bucket.load("data-lake-dev")
    s3.write_path(
        path=key,
        content=json.dumps(data).encode()
    )


@task
def download_from_s3(key: str) -> dict:
    """Download data from S3."""
    s3 = S3Bucket.load("data-lake-dev")
    content = s3.read_path(path=key)
    return json.loads(content)


@flow
def s3_flow():
    # Upload data
    data = {"key": "value", "timestamp": "2024-01-15T10:30:00Z"}
    upload_to_s3(data, "events/2024/01/15/event_001.json")

    # Download data
    retrieved = download_from_s3("events/2024/01/15/event_001.json")
    print(f"Retrieved: {retrieved}")
```

### AWS Credentials Block

```python
from prefect import task
from prefect_aws import AwsCredentials
import boto3


@task
def get_secret(secret_name: str) -> dict:
    """Fetch a secret from AWS Secrets Manager."""
    aws_creds = AwsCredentials.load("data-platform")

    client = boto3.client(
        "secretsmanager",
        region_name=aws_creds.region_name,
        # Credentials are automatic when using IAM roles
    )

    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response["SecretString"])
```

## AWS Secrets Manager Integration

For secrets like database passwords, fetch from AWS Secrets Manager at runtime rather than storing in blocks.

### Pattern: Fetch Secrets in Flows

```python
import json
import boto3
from prefect import task, flow, get_run_logger


@task
def get_secret(secret_name: str, region: str = "eu-west-2") -> dict:
    """Fetch a secret from AWS Secrets Manager."""
    logger = get_run_logger()

    # Uses IAM role credentials automatically on ECS/EC2
    client = boto3.client("secretsmanager", region_name=region)

    response = client.get_secret_value(SecretId=secret_name)
    secret_value = response["SecretString"]

    logger.info(f"Retrieved secret: {secret_name}")

    try:
        return json.loads(secret_value)
    except json.JSONDecodeError:
        return {"value": secret_value}


@flow
def flow_with_secrets():
    # Fetch Snowflake credentials at runtime
    snowflake_creds = get_secret("terraform/snowflake-credentials")

    # Use the credentials
    account = snowflake_creds["account"]
    user = snowflake_creds["user"]
    # ...
```

### Reusable Secrets Module

Create a reusable module in your data-pipelines repository:

```python
# utils/secrets.py
import json
import boto3
from functools import lru_cache


@lru_cache(maxsize=32)
def get_secret(secret_name: str, region: str = "eu-west-2") -> dict:
    """
    Fetch a secret from AWS Secrets Manager.

    Uses caching to avoid repeated API calls within a flow run.
    """
    client = boto3.client("secretsmanager", region_name=region)
    response = client.get_secret_value(SecretId=secret_name)

    try:
        return json.loads(response["SecretString"])
    except json.JSONDecodeError:
        return {"value": response["SecretString"]}
```

## Snowflake Block (Optional)

If you want to create a Snowflake connector block, add to `blocks.tf`:

```hcl
# Fetch Snowflake credentials from Secrets Manager
data "aws_secretsmanager_secret_version" "snowflake" {
  secret_id = "terraform/snowflake-credentials"
}

locals {
  snowflake_creds = jsondecode(data.aws_secretsmanager_secret_version.snowflake.secret_string)
}

resource "prefect_block" "snowflake_connector" {
  name      = "snowflake-analytics"
  type_slug = "snowflake-connector"

  data = jsonencode({
    account   = local.snowflake_creds.account
    user      = local.snowflake_creds.user
    password  = local.snowflake_creds.password
    role      = "ANALYTICS_TRANSFORMER"
    warehouse = "TRANSFORMING"
    database  = "ANALYTICS"
  })
}
```

!!! warning "Secrets in Terraform State"
    When fetching secrets in Terraform, they appear in state. For production, consider fetching credentials at flow runtime instead of storing them in blocks.

## Using Snowflake Block in Flows

```python
from prefect import flow, task
from prefect_snowflake import SnowflakeConnector


@task
def query_snowflake(query: str) -> list[dict]:
    """Execute a query against Snowflake."""
    connector = SnowflakeConnector.load("snowflake-analytics")

    with connector.get_connection() as conn:
        cursor = conn.cursor()
        cursor.execute(query)
        columns = [col[0] for col in cursor.description]
        rows = cursor.fetchall()

    return [dict(zip(columns, row)) for row in rows]


@flow
def snowflake_flow():
    results = query_snowflake("SELECT * FROM analytics.reporting.summary LIMIT 10")
    print(f"Got {len(results)} rows")
```

## Managing Blocks

### List Blocks

```sh
# List all blocks
prefect block ls

# List blocks of a specific type
prefect block ls --type s3-bucket
```

### Delete Blocks

```sh
prefect block delete s3-bucket/data-lake-dev
```

## Best Practices

### 1. Use Terraform for Block Management

Managing blocks via Terraform ensures:

- Blocks are version controlled
- Changes are reviewed in PRs
- Consistent across environments

### 2. Use IAM Roles Instead of Access Keys

```python
# Good: Uses IAM role automatically
aws_creds = AwsCredentials(region_name="eu-west-2")

# Avoid: Storing access keys
aws_creds = AwsCredentials(
    aws_access_key_id="AKIA...",  # Don't do this
    aws_secret_access_key="...",
)
```

### 3. Fetch Sensitive Data at Runtime

```python
# Good: Fetch from Secrets Manager at runtime
creds = get_secret("snowflake/credentials")

# Avoid: Hard-coding credentials
password = "my-secret-password"  # Don't do this!
```

### 4. Create Environment-Specific Blocks

Use naming conventions to distinguish environments:

```
data-lake-dev
data-lake-staging
data-lake-prod
```

## Summary

You've learned how to manage secrets and configuration with Prefect Blocks:

- [x] Created blocks via Terraform
- [x] Connected to S3 data lake buckets
- [x] Integrated with AWS Secrets Manager
- [x] Used blocks in flows

## What's Next

You've completed the core Prefect setup. The final page summarises what you've built and outlines next steps.

Continue to [Finishing Up](11-finishing-up.md) →
