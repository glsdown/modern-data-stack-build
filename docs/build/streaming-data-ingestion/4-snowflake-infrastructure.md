# Snowflake Infrastructure

On this page, you will:

- [x] Create STREAMING database via Terraform using the database module
- [x] Create SVC_KAFKA_CONNECTOR service account with `user_create_dedicated_role = true`
- [x] Configure key-pair authentication for the service account
- [x] Grant STREAMING_DB_WRITER role to SVC_KAFKA_CONNECTOR
- [x] Add credentials to AWS Secrets Manager
- [x] Test authentication

## Overview

Streaming data from Kafka requires dedicated Snowflake infrastructure. You'll create:

1. **STREAMING database** — Stores events from Kafka topics
2. **SVC_KAFKA_CONNECTOR service account** — Authenticates Kafka Connect
3. **Key-pair authentication** — Secure, passwordless auth for automation
4. **STREAMING_DB_WRITER role** — Write permissions for the connector

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SNOWFLAKE STREAMING INFRASTRUCTURE                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Kafka Connect (Confluent Cloud)                                        │
│  ────────────────────────────                                           │
│  ┌────────────────────────────────────────────────────┐                │
│  │ Snowflake Sink Connector                           │                │
│  │ • Reads from Kafka topics                          │                │
│  │ • Authenticates as SVC_KAFKA_CONNECTOR (key-pair)  │                │
│  │ • Writes to STREAMING database                     │                │
│  └────────────────────────────────────────────────────┘                │
│                         │                                               │
│                         ▼                                               │
│  Snowflake                                                              │
│  ─────────                                                              │
│  ┌────────────────────────────────────────────────────┐                │
│  │ STREAMING database                                 │                │
│  │ • STREAMING_DB_READER (read-only)                  │                │
│  │ • STREAMING_DB_WRITER (insert/update)              │                │
│  └────────────────────────────────────────────────────┘                │
│                         │                                               │
│  ┌────────────────────────────────────────────────────┐                │
│  │ SVC_KAFKA_CONNECTOR service account                │                │
│  │ • Type: SERVICE                                    │                │
│  │ • Auth: Key-pair (no password)                     │                │
│  │ • Dedicated role: SVC_KAFKA_CONNECTOR              │                │
│  │ • Grants: STREAMING_DB_WRITER                      │                │
│  └────────────────────────────────────────────────────┘                │
│                         │                                               │
│  ┌────────────────────────────────────────────────────┐                │
│  │ ANALYTICS_SOURCES_READER (account role)            │                │
│  │ • Granted: STREAMING_DB_READER                     │                │
│  │ • Inherited by: ANALYTICS_DEVELOPER                │                │
│  └────────────────────────────────────────────────────┘                │
│                                                                         │
│  Pattern: Service account has dedicated role + writer access            │
│  Analysts get reader access via ANALYTICS_SOURCES_READER                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Create STREAMING Database

Use the existing `snowflake_database` Terraform module to create the STREAMING database.

### Add to databases.tf

Edit `~/src/terraform/snowflake/config/databases.tf` and add:

```hcl
# -----------------------------------------------------------------------------
# STREAMING Database (Kafka Events)
# -----------------------------------------------------------------------------
module "streaming_database" {
  source = "../modules/snowflake_database"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  database_name    = "STREAMING"
  database_comment = "Real-time events from Kafka topics via Snowpipe Streaming"

  # Data retention (7 days for streaming events - adjust based on your needs)
  data_retention_time_in_days = 7

  # Automatic database roles
  create_reader_role = true  # STREAMING_DB_READER
  create_writer_role = true  # STREAMING_DB_WRITER
}
```

### Why 7 Days Retention?

Streaming data is typically processed quickly and doesn't need long retention in raw form:

- **7 days** — Sufficient for debugging, reprocessing recent events
- **Longer retention** — Store in S3 or transformed tables in ANALYTICS
- **Cost savings** — Lower storage costs for high-volume event streams

Adjust based on your requirements:
- **1 day** — Very high volume, process immediately
- **30 days** — Compliance, audit trails
- **0 days** — Inherit account default (typically 1 day for transient data)

### Apply Terraform

```sh
cd ~/src/terraform/snowflake/config

# Set AWS profile for Secrets Manager access
export AWS_PROFILE=infrastructure-admin

# Initialise (if not already done)
terraform init

# Plan
terraform plan -out=tfplan

# Review changes (should show STREAMING database + 2 database roles)

# Apply
terraform apply tfplan
```

**Expected output:**
```
module.streaming_database.snowflake_database.this: Creating...
module.streaming_database.snowflake_database_role.reader: Creating...
module.streaming_database.snowflake_database_role.writer: Creating...
module.streaming_database.snowflake_grant_database_role.reader_to_sysadmin: Creating...
module.streaming_database.snowflake_grant_database_role.writer_to_sysadmin: Creating...

Apply complete! Resources: 5 added, 0 changed, 0 destroyed.
```

### Verify in Snowflake

```sql
-- Show databases
SHOW DATABASES LIKE 'STREAMING';

-- Show database roles
SHOW DATABASE ROLES IN DATABASE STREAMING;

-- Result:
-- STREAMING_DB_READER
-- STREAMING_DB_WRITER
```

## Create SVC_KAFKA_CONNECTOR Service Account

Service accounts use the `user_create_dedicated_role = true` pattern, which creates a role matching the user name.

### Generate Key Pair

Snowflake service accounts use **key-pair authentication** (not passwords) for security.

Generate an RSA key pair:

```sh
# Create directory for keys
mkdir -p ~/.ssh/snowflake

# Generate private key (4096-bit RSA)
openssl genrsa -out ~/.ssh/snowflake/svc_kafka_connector_key.pem 4096

# Generate public key
openssl rsa -in ~/.ssh/snowflake/svc_kafka_connector_key.pem \
    -pubout -out ~/.ssh/snowflake/svc_kafka_connector_key.pub

# Secure private key
chmod 600 ~/.ssh/snowflake/svc_kafka_connector_key.pem

# Extract public key value (without headers) for Snowflake
grep -v "BEGIN PUBLIC KEY" ~/.ssh/snowflake/svc_kafka_connector_key.pub | \
    grep -v "END PUBLIC KEY" | \
    tr -d '\n' > ~/.ssh/snowflake/svc_kafka_connector_key_value.txt

# Display public key value
cat ~/.ssh/snowflake/svc_kafka_connector_key_value.txt
```

**Expected output:**
```
MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEA1234567890abcdef...
```

!!! warning "Protect Private Keys"
    The private key (`svc_kafka_connector_key.pem`) is sensitive. Never commit it to Git or share it. Store securely and restrict file permissions to 600.

### Add Service Account to users.tf

Edit `~/src/terraform/snowflake/config/users.tf` and add:

```hcl
# -----------------------------------------------------------------------------
# Service Accounts
# -----------------------------------------------------------------------------

# SVC_KAFKA_CONNECTOR - Kafka Connect writes to Snowflake
module "svc_kafka_connector" {
  source = "../modules/snowflake_user"

  providers = {
    snowflake.account_admin = snowflake.account_admin
    snowflake.security_admin = snowflake.security_admin
    snowflake.sys_admin      = snowflake.sys_admin
  }

  username = "SVC_KAFKA_CONNECTOR"
  comment  = "Service account for Kafka Connect to write streaming events"

  # Service account configuration
  user_type                  = "SERVICE"
  user_create_dedicated_role = true  # Creates SVC_KAFKA_CONNECTOR role

  # Key-pair authentication (no password)
  rsa_public_key = file("~/.ssh/snowflake/svc_kafka_connector_key_value.txt")

  # Default warehouse
  default_warehouse = "INGEST_WH"

  # Dedicated warehouse for this service account
  create_dedicated_warehouse = false  # Use shared INGEST_WH

  # Roles to grant
  granted_roles = [
    module.streaming_database.writer_role_name,  # STREAMING_DB_WRITER
  ]

  # Network policy (optional - restrict IPs if needed)
  network_policy_name = null  # Allow from anywhere for Confluent Cloud
}
```

### Why user_create_dedicated_role = true?

This pattern creates a role matching the service account name (`SVC_KAFKA_CONNECTOR`):

- **Dedicated role** — Each service account has its own role
- **Audit trail** — Query history shows which service account made changes
- **Least privilege** — Grant only necessary roles to the dedicated role
- **Consistency** — All service accounts follow the same pattern

### Apply Terraform

```sh
terraform plan -out=tfplan

# Review changes:
# + SVC_KAFKA_CONNECTOR user
# + SVC_KAFKA_CONNECTOR role (dedicated)
# + Grant STREAMING_DB_WRITER to SVC_KAFKA_CONNECTOR
# + Grant SVC_KAFKA_CONNECTOR to SYSADMIN

terraform apply tfplan
```

**Expected output:**
```
module.svc_kafka_connector.snowflake_user.this: Creating...
module.svc_kafka_connector.snowflake_role.dedicated: Creating...
module.svc_kafka_connector.snowflake_grant_account_role.dedicated_to_user: Creating...
module.svc_kafka_connector.snowflake_grant_account_role.granted_roles[0]: Creating...

Apply complete! Resources: 4 added, 0 changed, 0 destroyed.
```

### Verify in Snowflake

```sql
-- Show service account
SHOW USERS LIKE 'SVC_KAFKA_CONNECTOR';

-- Show dedicated role
SHOW ROLES LIKE 'SVC_KAFKA_CONNECTOR';

-- Show role grants
SHOW GRANTS TO ROLE SVC_KAFKA_CONNECTOR;

-- Result should include:
-- STREAMING_DB_WRITER (database role)
```

## Grant Reader Access to Analysts

Analysts need read-only access to streaming data. Grant `STREAMING_DB_READER` to `ANALYTICS_SOURCES_READER`.

### Update functional_roles.tf

Edit `~/src/terraform/snowflake/config/functional_roles.tf`:

```hcl
# ANALYTICS_SOURCES_READER - Read access to all data sources
module "analytics_sources_reader_role" {
  source = "../modules/snowflake_role"

  providers = {
    snowflake.security_admin = snowflake.security_admin
    snowflake.sys_admin      = snowflake.sys_admin
  }

  role_name    = "ANALYTICS_SOURCES_READER"
  role_comment = "Read access to all data sources (DLT, AIRBYTE, STREAMING)"

  granted_roles = [
    module.dlt_database.reader_role_name,        # DLT_DB_READER
    module.airbyte_database.reader_role_name,    # AIRBYTE_DB_READER
    module.streaming_database.reader_role_name,  # STREAMING_DB_READER (new)
  ]

  granted_to_roles = ["ANALYTICS_DEVELOPER"]
}
```

### Apply Terraform

```sh
terraform plan -out=tfplan

# Review: Should show grant of STREAMING_DB_READER to ANALYTICS_SOURCES_READER

terraform apply tfplan
```

**Expected output:**
```
module.analytics_sources_reader_role.snowflake_grant_account_role.granted_roles[2]: Creating...

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

### Verify Access

```sql
-- Login as analyst
USE ROLE ANALYTICS_DEVELOPER;

-- Should be able to read (but not write)
USE DATABASE STREAMING;
SHOW SCHEMAS;

-- Try to create schema (should fail)
CREATE SCHEMA TEST;
-- Error: Insufficient privileges
```

## Store Credentials in AWS Secrets Manager

Store the Snowflake service account credentials for Kafka Connect to use.

### Create Secret

```sh
# Set AWS profile
export AWS_PROFILE=data-engineer

# Read private key
PRIVATE_KEY=$(cat ~/.ssh/snowflake/svc_kafka_connector_key.pem | tr -d '\n')

# Create secret
aws secretsmanager create-secret \
    --name snowflake/svc-kafka-connector \
    --description "Snowflake credentials for Kafka Connect Snowflake Sink Connector" \
    --secret-string "{
        \"account\": \"your-account.eu-west-2.aws\",
        \"user\": \"SVC_KAFKA_CONNECTOR\",
        \"private_key\": \"$PRIVATE_KEY\",
        \"database\": \"STREAMING\",
        \"schema\": \"PUBLIC\",
        \"warehouse\": \"INGEST_WH\",
        \"role\": \"SVC_KAFKA_CONNECTOR\"
    }" \
    --region eu-west-2
```

!!! tip "Account Identifier"
    Find your Snowflake account identifier:
    ```sql
    SELECT CURRENT_ACCOUNT();
    ```
    Format: `your-account.eu-west-2.aws` (includes region and cloud)

**Expected output:**
```json
{
    "ARN": "arn:aws:secretsmanager:eu-west-2:123456789012:secret:snowflake/svc-kafka-connector-abc123",
    "Name": "snowflake/svc-kafka-connector",
    "VersionId": "a1b2c3d4-5e6f-7a8b-9c0d-1e2f3a4b5c6d"
}
```

### Retrieve Secret (Test)

```sh
aws secretsmanager get-secret-value \
    --secret-id snowflake/svc-kafka-connector \
    --region eu-west-2 \
    --query SecretString \
    --output text | jq .
```

**Expected output:**
```json
{
  "account": "your-account.eu-west-2.aws",
  "user": "SVC_KAFKA_CONNECTOR",
  "private_key": "-----BEGIN RSA PRIVATE KEY-----\n...",
  "database": "STREAMING",
  "schema": "PUBLIC",
  "warehouse": "INGEST_WH",
  "role": "SVC_KAFKA_CONNECTOR"
}
```

## Test Authentication

Verify the service account can authenticate using the private key.

### Python Test Script

Create `test_snowflake_auth.py`:

```python
#!/usr/bin/env python3
"""
Test Snowflake key-pair authentication for SVC_KAFKA_CONNECTOR.
"""
import json
import boto3
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.backends import default_backend
import snowflake.connector


def get_snowflake_credentials():
    """Retrieve Snowflake credentials from AWS Secrets Manager."""
    session = boto3.session.Session()
    client = session.client(
        service_name='secretsmanager',
        region_name='eu-west-2'
    )

    response = client.get_secret_value(SecretId='snowflake/svc-kafka-connector')
    return json.loads(response['SecretString'])


def get_private_key_bytes(private_key_pem):
    """Convert PEM private key to DER bytes for Snowflake connector."""
    private_key = serialization.load_pem_private_key(
        private_key_pem.encode('utf-8'),
        password=None,
        backend=default_backend()
    )

    return private_key.private_bytes(
        encoding=serialization.Encoding.DER,
        format=serialization.PrivateFormat.PKCS8,
        encryption_algorithm=serialization.NoEncryption()
    )


def main():
    print("🔗 Connecting to Snowflake...")

    # Get credentials from Secrets Manager
    creds = get_snowflake_credentials()

    # Convert private key to bytes
    private_key_bytes = get_private_key_bytes(creds['private_key'])

    # Connect to Snowflake
    conn = snowflake.connector.connect(
        account=creds['account'],
        user=creds['user'],
        private_key=private_key_bytes,
        warehouse=creds['warehouse'],
        database=creds['database'],
        schema=creds['schema'],
        role=creds['role']
    )

    # Test query
    cursor = conn.cursor()
    cursor.execute("SELECT CURRENT_USER(), CURRENT_ROLE(), CURRENT_DATABASE()")
    result = cursor.fetchone()

    print(f"✅ Connected successfully!")
    print(f"   User: {result[0]}")
    print(f"   Role: {result[1]}")
    print(f"   Database: {result[2]}")

    # Test write permission
    cursor.execute("CREATE SCHEMA IF NOT EXISTS TEST_SCHEMA")
    print(f"✅ Write permission verified (created TEST_SCHEMA)")

    # Clean up
    cursor.execute("DROP SCHEMA IF EXISTS TEST_SCHEMA")
    cursor.close()
    conn.close()

    print("✅ Authentication test complete!")


if __name__ == '__main__':
    main()
```

### Install Dependencies

```sh
uv add snowflake-connector-python cryptography boto3
```

### Run Test

```sh
python test_snowflake_auth.py
```

**Expected output:**
```
🔗 Connecting to Snowflake...
✅ Connected successfully!
   User: SVC_KAFKA_CONNECTOR
   Role: SVC_KAFKA_CONNECTOR
   Database: STREAMING
✅ Write permission verified (created TEST_SCHEMA)
✅ Authentication test complete!
```

### Troubleshooting

**Error: `Invalid private key`**

Check:
- Private key file is correctly formatted (PEM format)
- No extra whitespace or line breaks in the key
- Key was generated with 2048-bit or 4096-bit RSA

**Error: `User does not exist`**

Service account wasn't created. Re-run Terraform apply.

**Error: `Insufficient privileges`**

Service account doesn't have STREAMING_DB_WRITER granted. Check grants:
```sql
SHOW GRANTS TO ROLE SVC_KAFKA_CONNECTOR;
```

## Summary

You've configured Snowflake infrastructure for streaming:

- [x] **STREAMING database created** — with 7-day retention for event data
- [x] **Database roles created** — STREAMING_DB_READER and STREAMING_DB_WRITER
- [x] **SVC_KAFKA_CONNECTOR service account** — with dedicated role and key-pair auth
- [x] **Writer permissions granted** — service account can write to STREAMING
- [x] **Reader access for analysts** — via ANALYTICS_SOURCES_READER
- [x] **Credentials stored** — in AWS Secrets Manager for Kafka Connect
- [x] **Authentication tested** — Python script verified key-pair auth works

Snowflake is ready to receive events from Kafka. Next, deploy the Snowflake Sink Connector.

## What's Next

Deploy the Snowflake Kafka Connector to stream events from Kafka topics into Snowflake tables.

Continue to [Kafka Connect Snowflake](5-kafka-connect-snowflake.md) →
