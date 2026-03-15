# Airbyte Cloud Setup

On this page, you will:

- [x] Create an Airbyte Cloud workspace
- [x] Generate API credentials for Prefect integration
- [x] Store API credentials in AWS Secrets Manager

## Overview

Airbyte Cloud is the managed deployment option. You configure sources and destinations via the web UI, and Airbyte handles infrastructure, scaling, and upgrades.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      AIRBYTE CLOUD SETUP                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Step 1              Step 2              Step 3              Step 4         │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌───────────┐  │
│  │ Create      │     │ Create      │     │ Generate    │     │ Store in  │  │
│  │ Account     │────▶│ Workspace   │────▶│ API Key     │────▶│ Secrets   │  │
│  │             │     │             │     │             │     │ Manager   │  │
│  └─────────────┘     └─────────────┘     └─────────────┘     └───────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Create Account

1. Go to [cloud.airbyte.com](https://cloud.airbyte.com)
2. Sign up with your email or SSO provider
3. Verify your email address

## Create Workspace

A workspace is the top-level container for your connections. Create one per environment (production, staging).

1. After signing in, click **Create Workspace**
2. Name it `production` (or your preferred name)
3. Select your region:
   - **US** (us-east-1) — default
   - **EU** (eu-west-1) — if data residency is a concern

!!! note "Region Selection"
    Choose the region closest to your Snowflake account. If your Snowflake is in `eu-west-2`, the EU region minimises data transfer latency.

## Navigate the UI

The Airbyte Cloud UI has these main sections:

| Section | Purpose |
|---------|---------|
| **Sources** | Configure data extraction endpoints |
| **Destinations** | Configure where data is loaded |
| **Connections** | Link sources to destinations with sync configuration |
| **Settings** | Workspace settings, API keys, billing |

You will configure sources and destinations in [HubSpot Connection](6-hubspot-connection.md). For now, focus on generating API credentials.

## Generate API Credentials

Prefect needs API access to trigger Airbyte syncs programmatically. Generate an API key:

1. Navigate to **Settings** > **API Keys** (or **Developer** section)
2. Click **Generate API Key**
3. Name it `prefect-integration`
4. Copy the generated key immediately — it will not be shown again

The API key provides access to:

- Trigger connection syncs
- Check sync status
- List connections and their configuration

!!! warning "API Key Security"
    The API key grants full access to your Airbyte workspace. Store it securely and never commit it to source control.

## Store Credentials in AWS Secrets Manager

Create the secret containers in your AWS Terraform configuration (`terraform/aws/secrets.tf`). This follows the same pattern as other secrets — the container is managed by Terraform, the value is set manually:

```hcl
# Add to terraform/aws/secrets.tf

# -----------------------------------------------------------------------------
# Airbyte Credentials
# -----------------------------------------------------------------------------
resource "aws_secretsmanager_secret" "airbyte_api_credentials" {
  name        = "airbyte/api-credentials"
  description = "Airbyte Cloud API credentials for Prefect integration"

  tags = {
    Name      = "airbyte/api-credentials"
    ManagedBy = "terraform"
  }
}

resource "aws_secretsmanager_secret" "airbyte_snowflake_credentials" {
  name        = "airbyte/snowflake-credentials"
  description = "Snowflake SVC_AIRBYTE credentials for Airbyte connector"

  tags = {
    Name      = "airbyte/snowflake-credentials"
    ManagedBy = "terraform"
  }
}
```

### Update CI/CD Permissions

The CI/CD role needs access to Airbyte secrets. Update the IAM policy in `oidc.tf` to include the `airbyte/*` prefix:

```hcl
# In terraform/aws/oidc.tf - update the SecretsManagerAccess statement

statement {
  sid     = "SecretsManagerAccess"
  actions = ["secretsmanager:GetSecretValue"]
  resources = [
    "arn:aws:secretsmanager:${var.aws_region}:${var.aws_account_id}:secret:terraform/*",
    "arn:aws:secretsmanager:${var.aws_region}:${var.aws_account_id}:secret:prefect/*",
    "arn:aws:secretsmanager:${var.aws_region}:${var.aws_account_id}:secret:airbyte/*"
  ]
}
```

Deploy the changes via CI/CD, then set the API credential values:

```sh
aws secretsmanager put-secret-value \
    --secret-id "airbyte/api-credentials" \
    --secret-string '{
        "api_key": "YOUR_API_KEY_HERE",
        "workspace_id": "YOUR_WORKSPACE_ID",
        "api_url": "https://api.airbyte.com/v1"
    }' \
    --profile infrastructure-admin
```

!!! tip "Finding Your Workspace ID"
    The workspace ID is visible in the URL when logged into Airbyte Cloud:
    `https://cloud.airbyte.com/workspaces/{workspace_id}/connections`

Verify the secret:

```sh
aws secretsmanager get-secret-value \
    --secret-id "airbyte/api-credentials" \
    --query SecretString \
    --output text | jq -r '.workspace_id'
```

## Verify API Access

Test that the API key works:

```sh
curl -s -H "Authorization: Bearer YOUR_API_KEY" \
    "https://api.airbyte.com/v1/workspaces" | jq '.data[].name'
```

Expected output:

```json
"production"
```

## Billing Setup

Configure billing to avoid service interruption:

1. Navigate to **Settings** > **Billing**
2. Add a payment method
3. Review the current plan (Free or Starter)
4. Upgrade to Starter ($99/month) when ready to go to production

!!! note "Free Tier Limitations"
    The free tier has limited credits and connections. It is suitable for testing but not production workloads. Upgrade to Starter before relying on Airbyte for production data.

## IAM Permissions

Ensure the Prefect worker's IAM role can read the Airbyte secret. Update the IAM policy from [Credentials Setup](../batch-data-ingestion/4-credentials-setup.md) to include the new secret path:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue",
                "secretsmanager:DescribeSecret"
            ],
            "Resource": [
                "arn:aws:secretsmanager:eu-west-2:*:secret:dlt/*",
                "arn:aws:secretsmanager:eu-west-2:*:secret:airbyte/*"
            ]
        }
    ]
}
```

## Summary

You've set up Airbyte Cloud:

- [x] Created an Airbyte Cloud account and workspace
- [x] Generated API credentials for Prefect integration
- [x] Stored credentials in AWS Secrets Manager
- [x] Verified API access

## What's Next

If you're using Airbyte Cloud, skip to [Snowflake Infrastructure](5-snowflake-infrastructure.md) to set up the Snowflake database and service account.

If you want to self-host instead, continue to [Self-Hosted Setup](4-self-hosted-setup.md) →
