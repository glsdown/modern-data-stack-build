# Snowflake Infrastructure for Lightdash

On this page, you will:

- [x] Create a `SVC_LIGHTDASH` service account in Snowflake using Terraform
- [x] Configure key-pair authentication for secure, passwordless access
- [x] Grant the `ANALYTICS_REPORTER` role (read-only access to the `REPORTING` schema)
- [x] Store credentials in AWS Secrets Manager for Lightdash to use
- [x] Understand why Lightdash gets read-only access, not full warehouse access

## Overview

Lightdash needs a Snowflake service account to query your data warehouse. Following the principle of least privilege, this account has **read-only access to the `REPORTING` schema only** — the curated subset of analytics-ready models published by dbt.

This ensures Lightdash cannot:
- Modify data in the warehouse
- Access raw or intermediate dbt models
- Query staging or source data
- Create or drop tables

The `SVC_LIGHTDASH` account uses key-pair authentication (no password) and is granted the `ANALYTICS_REPORTER` role, which you created in the [Data Warehouse section](../data-warehouse/4-functional-roles.md).

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    LIGHTDASH SNOWFLAKE ACCESS                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Service Account              Role                   Access             │
│  ───────────────              ────                   ──────             │
│                                                                         │
│  SVC_LIGHTDASH  ────────▶  SVC_LIGHTDASH  ──────▶  ANALYTICS_REPORTER  │
│  (user)                    (dedicated role)        (functional role)    │
│  • Key-pair auth                                                        │
│  • No password                                     └─▶ READ on          │
│                                                         ANALYTICS.       │
│                                                         REPORTING        │
│                                                                         │
│  Lightdash queries REPORTING schema only.                              │
│  Cannot access STAGING, INTERMEDIATE, MARTS.                           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Prerequisites

Before proceeding, ensure you have:

- [x] Completed [Data Warehouse](../data-warehouse/index.md) — `ANALYTICS` database, `REPORTING` schema, `ANALYTICS_REPORTER` role exist
- [x] Completed [Data Transformation](../data-transformation/index.md) — dbt models published to `REPORTING` schema
- [x] Terraform configured for Snowflake (from [Snowflake Infrastructure](../data-warehouse/1-project-structure.md))
- [x] AWS CLI configured with `infrastructure-admin` profile

## Generate Key Pair

Lightdash authenticates to Snowflake using key-pair authentication. Generate an RSA key pair on your local machine:

```sh
# Generate private key (4096-bit RSA, encrypted with passphrase)
openssl genrsa -out svc_lightdash_rsa_key.pem 4096

# Generate public key from private key
openssl rsa -in svc_lightdash_rsa_key.pem -pubout -out svc_lightdash_rsa_key.pub

# Display the public key (you'll add this to Terraform)
cat svc_lightdash_rsa_key.pub
```

Store both keys securely:

1. **Private key** → 1Password (secure note: "Snowflake SVC_LIGHTDASH Private Key")
2. **Public key** → Terraform variable (see below)

!!! warning "Never Commit Private Keys"
    The private key (`svc_lightdash_rsa_key.pem`) must never be committed to Git. Store it in 1Password and pass it to Lightdash via environment variables or secrets.

## Terraform Configuration

### Add the Service User Module

Create a new Terraform file for the Lightdash service account:

```hcl
# terraform/snowflake/service-accounts/lightdash.tf
module "service_user_lightdash" {
  source = "../modules/snowflake_service_user"

  username    = "SVC_LIGHTDASH"
  comment     = "Service account for Lightdash BI tool (read-only access to REPORTING schema)"
  email       = "data-platform@yourcompany.com"
  rsa_public_key = var.svc_lightdash_public_key

  user_create_dedicated_role = true
  dedicated_role_grants = [
    "ANALYTICS_REPORTER"
  ]

  default_warehouse = "REPORTING"
  default_namespace = "ANALYTICS.REPORTING"
  default_role      = "SVC_LIGHTDASH"

  tags = {
    Service   = "lightdash"
    ManagedBy = "terraform"
  }
}
```

### Add Variable for Public Key

Add the public key variable to your Terraform variables:

```hcl
# terraform/snowflake/variables.tf
variable "svc_lightdash_public_key" {
  description = "RSA public key for SVC_LIGHTDASH service account"
  type        = string
  sensitive   = true
}
```

### Pass the Public Key

Pass the public key via environment variable (recommended for CI/CD):

```sh
# Extract the public key content (remove header/footer)
export TF_VAR_svc_lightdash_public_key=$(cat svc_lightdash_rsa_key.pub | \
    grep -v "BEGIN PUBLIC KEY" | \
    grep -v "END PUBLIC KEY" | \
    tr -d '\n')

echo $TF_VAR_svc_lightdash_public_key
# Should output: MIICIjANBgkqhkiG9w0BAQ...rest of key...
```

Alternatively, create a `terraform.tfvars` file (do not commit this file):

```hcl
# terraform/snowflake/terraform.tfvars (DO NOT COMMIT)
svc_lightdash_public_key = "MIICIjANBgkqhkiG9w0BAQ...your public key..."
```

Add `terraform.tfvars` to `.gitignore`:

```gitignore
# terraform/snowflake/.gitignore
terraform.tfvars
*.pem
*.pub
```

## Apply Terraform

Run Terraform to create the service account:

```sh
cd terraform/snowflake
terraform plan

# Review the plan — should create:
# - snowflake_user.svc_lightdash
# - snowflake_role.svc_lightdash (dedicated role)
# - snowflake_role_grants.svc_lightdash_grants (grants ANALYTICS_REPORTER)

terraform apply
```

Expected output:

```
Apply complete! Resources: 3 added, 0 changed, 0 destroyed.

Outputs:

svc_lightdash_public_key_fingerprint = "SHA256:a1b2c3d4e5f6..."
```

## Verify the Account

Test the service account authentication:

```sql
-- In Snowsight, logged in as USERADMIN or ACCOUNTADMIN
USE ROLE USERADMIN;

-- Show the user and its properties
SHOW USERS LIKE 'SVC_LIGHTDASH';

-- Verify RSA public key is set
DESC USER SVC_LIGHTDASH;
-- Look for RSA_PUBLIC_KEY_FP (fingerprint)

-- Verify role grants
SHOW GRANTS TO USER SVC_LIGHTDASH;
-- Should show: granted role SVC_LIGHTDASH

-- Verify dedicated role grants
USE ROLE SECURITYADMIN;
SHOW GRANTS TO ROLE SVC_LIGHTDASH;
-- Should show: granted role ANALYTICS_REPORTER
```

### Test Read Access

Verify `SVC_LIGHTDASH` can query the `REPORTING` schema:

```sql
-- Switch to SVC_LIGHTDASH role
USE ROLE SVC_LIGHTDASH;

-- Should succeed: query REPORTING schema
SELECT COUNT(*) FROM analytics.reporting.fct_exchange_rates;

-- Should fail: MARTS schema not accessible to ANALYTICS_REPORTER
SELECT COUNT(*) FROM analytics.marts.fct_exchange_rates;
-- Error: SQL access control error: Insufficient privileges
```

!!! success "Least Privilege Working"
    The error on the `MARTS` query confirms that `SVC_LIGHTDASH` has read-only access to `REPORTING` only, as intended.

## Store Credentials in AWS Secrets Manager

Lightdash (whether self-hosted or cloud) needs the private key to authenticate. Store it in AWS Secrets Manager:

```sh
# Read private key content
PRIVATE_KEY=$(cat svc_lightdash_rsa_key.pem)

# Create secret in AWS Secrets Manager
aws secretsmanager create-secret \
    --name "lightdash/snowflake-credentials" \
    --description "Snowflake credentials for Lightdash BI tool" \
    --secret-string "$(jq -n \
        --arg account "your-account.snowflakecomputing.com" \
        --arg user "SVC_LIGHTDASH" \
        --arg role "SVC_LIGHTDASH" \
        --arg warehouse "REPORTING" \
        --arg database "ANALYTICS" \
        --arg schema "REPORTING" \
        --arg private_key "$PRIVATE_KEY" \
        '{
            account: $account,
            user: $user,
            role: $role,
            warehouse: $warehouse,
            database: $database,
            schema: $schema,
            private_key: $private_key
        }')" \
    --profile infrastructure-admin
```

Verify the secret:

```sh
aws secretsmanager get-secret-value \
    --secret-id "lightdash/snowflake-credentials" \
    --query SecretString \
    --output text \
    --profile infrastructure-admin | jq .
```

Expected output:

```json
{
  "account": "your-account.snowflakecomputing.com",
  "user": "SVC_LIGHTDASH",
  "role": "SVC_LIGHTDASH",
  "warehouse": "REPORTING",
  "database": "ANALYTICS",
  "schema": "REPORTING",
  "private_key": "-----BEGIN RSA PRIVATE KEY-----\n..."
}
```

!!! tip "Terraform AWS Secrets Manager"
    You can also create secrets via Terraform using the `aws_secretsmanager_secret` resource. This keeps all infrastructure in code.

## Why ANALYTICS_REPORTER (Not ANALYTICS_DEVELOPER)?

Lightdash gets the `ANALYTICS_REPORTER` role, which has **read-only access to the `REPORTING` schema**. This is intentional:

| Role | Access | Use Case |
|------|--------|----------|
| **ANALYTICS_TRANSFORMER** | Read/write on `STAGING`, `INTERMEDIATE`, `MARTS`; write to `REPORTING` | dbt (SVC_DBT) — transforms and publishes models |
| **ANALYTICS_DEVELOPER** | Read on all schemas in `ANALYTICS` and `ANALYTICS_DEV`; write to `ANALYTICS_DEV` | Analytics engineers developing dbt models |
| **ANALYTICS_REPORTER** | Read-only on `REPORTING` schema | BI tools (Lightdash, Tableau) — query published models only |

**Why not give Lightdash access to `MARTS`?**

The `REPORTING` schema is a **curated subset** of `MARTS` — only the models you want business users to query. This allows you to:
- Publish stable, documented, production-ready models to `REPORTING`
- Keep experimental or internal models in `MARTS` (not exposed to BI tools)
- Control what end users see without giving access to all dbt models

**Example `REPORTING` schema structure:**

```sql
-- Published to REPORTING (BI-ready)
CREATE VIEW analytics.reporting.fct_exchange_rates AS
SELECT * FROM analytics.marts.fct_exchange_rates;

CREATE VIEW analytics.reporting.dim_products AS
SELECT * FROM analytics.marts.dim_products;

-- NOT published to REPORTING (internal dbt model)
-- analytics.marts.int_exchange_rates__unioned
-- (Intermediate model, not BI-ready)
```

This pattern is covered in [dbt Mart Models](../data-transformation/7-mart-models.md).

## Alternative: ACCOUNTADMIN for Testing

For initial testing, you might grant broader access temporarily:

```sql
-- TEMPORARY: Grant ANALYTICS_DEVELOPER for testing Lightdash setup
USE ROLE SECURITYADMIN;
GRANT ROLE ANALYTICS_DEVELOPER TO ROLE SVC_LIGHTDASH;

-- After testing, revoke and keep only ANALYTICS_REPORTER
REVOKE ROLE ANALYTICS_DEVELOPER FROM ROLE SVC_LIGHTDASH;
```

!!! danger "Do Not Use ACCOUNTADMIN in Production"
    Never grant `ACCOUNTADMIN` or `SYSADMIN` to service accounts. These roles have unrestricted access and bypass all security controls.

## Cost Considerations

The `SVC_LIGHTDASH` account uses the `REPORTING` warehouse (X-Small). Warehouse compute is billed when queries run.

**Expected costs:**
- **Warehouse size**: X-Small = $2/hour
- **Auto-suspend**: 60 seconds (warehouse pauses when idle)
- **Typical query**: 2-5 seconds
- **Dashboard load**: 5-10 queries (Lightdash fetches data for each chart)

**Monthly estimate:**
- 100 dashboard views/day (5 queries each = 500 queries/day)
- Average query duration: 3 seconds
- Daily compute: 500 queries × 3 seconds = 1500 seconds = 25 minutes = $0.83/day
- **Monthly: ~$25 in Snowflake compute**

This is separate from Lightdash infrastructure costs (~$30/month for self-hosted or $2400/month for cloud).

!!! tip "Optimise Query Performance"
    Fast queries reduce costs. Ensure your dbt models are:
    - Materialised as tables (not views) in `REPORTING` for complex aggregations
    - Indexed or clustered on commonly filtered columns (if using Snowflake Enterprise)
    - Pre-aggregated where possible (avoid `COUNT(*)` on raw tables)

## Summary

You've configured Snowflake for Lightdash:

- [x] Created `SVC_LIGHTDASH` service account with key-pair authentication
- [x] Created `SVC_LIGHTDASH` dedicated role and granted `ANALYTICS_REPORTER`
- [x] Verified read-only access to `REPORTING` schema (cannot access `MARTS`, `STAGING`, or modify data)
- [x] Stored credentials in AWS Secrets Manager
- [x] Understood the `REPORTING` schema design (curated subset of `MARTS` for BI tools)

The `SVC_LIGHTDASH` account is now ready for Lightdash to connect and query your dbt models.

## What's Next

With Snowflake infrastructure ready, set up Lightdash Cloud for a quick, managed BI experience.

Continue to [Lightdash Setup](6-lightdash-setup.md) →
