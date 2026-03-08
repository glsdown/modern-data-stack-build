# Runbook: Security Hardening

## Summary

Strengthen the security posture of the data stack through key rotation, audit logging, network policies, and access reviews. This runbook covers both routine security maintenance and targeted hardening tasks.

## When to Use

- Scheduled key rotation (recommended every 90 days for service accounts)
- Onboarding or offboarding team members
- Security audit or compliance review
- After a suspected compromise
- Periodic access review (recommended quarterly)

## Prerequisites

- [ ] Access: Snowflake with SECURITYADMIN or ACCOUNTADMIN role
- [ ] Access: AWS with `infrastructure-admin` profile
- [ ] Access: Terraform repository (write access)
- [ ] Context: Which security task is being performed

## Steps

### 1. Rotate Service Account Keys

Service accounts use key-pair authentication. Rotate keys on a regular schedule (every 90 days recommended).

#### Snowflake Key Rotation

Snowflake supports two active public keys (`RSA_PUBLIC_KEY` and `RSA_PUBLIC_KEY_2`), enabling zero-downtime rotation:

1. **Generate a new key pair:**

    ```sh
    openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out new_key.pem -nocrypt
    openssl rsa -in new_key.pem -pubout -out new_key.pub
    ```

2. **Set the new key as the secondary key:**

    ```sql
    USE ROLE SECURITYADMIN;
    ALTER USER SVC_DLT SET RSA_PUBLIC_KEY_2 = '<new public key content>';
    ```

3. **Update AWS Secrets Manager** with the new private key:

    ```sh
    # Read existing secret, update the private_key field
    aws secretsmanager put-secret-value \
        --secret-id "dlt/snowflake-credentials" \
        --secret-string "{
          \"account\": \"YOUR_ACCOUNT\",
          \"user\": \"SVC_DLT\",
          \"private_key\": \"$(cat new_key.pem)\"
        }" \
        --profile infrastructure-admin
    ```

4. **Verify the new key works** by running a test pipeline or query

5. **Promote the new key to primary and remove the old key:**

    ```sql
    USE ROLE SECURITYADMIN;
    -- Copy key 2 to key 1
    ALTER USER SVC_DLT SET RSA_PUBLIC_KEY = '<new public key content>';
    -- Remove key 2
    ALTER USER SVC_DLT UNSET RSA_PUBLIC_KEY_2;
    ```

6. **Delete local key files:**

    ```sh
    rm new_key.pem new_key.pub
    ```

Repeat for each service account: `SVC_DLT`, `SVC_DBT`, `SVC_AIRBYTE`, `SVC_TERRAFORM`, `SVC_LIGHTDASH`, `SVC_KAFKA_CONNECTOR`.

!!! tip "Automate Key Rotation"
    Consider creating a Prefect flow that automates key rotation using the Snowflake Python connector and `boto3` for Secrets Manager updates. Schedule it to run every 90 days.

### 2. Review User Access

Run a quarterly access review to ensure users have appropriate permissions.

#### List All Users and Roles

```sql
USE ROLE SECURITYADMIN;

-- All users with their default roles
SELECT
    name,
    login_name,
    display_name,
    default_role,
    default_warehouse,
    disabled,
    last_success_login,
    created_on
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE deleted_on IS NULL
ORDER BY last_success_login DESC NULLS LAST;
```

#### Find Inactive Users

```sql
-- Users who haven't logged in for 90+ days
SELECT name, display_name, default_role, last_success_login
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE deleted_on IS NULL
  AND disabled = 'false'
  AND (last_success_login IS NULL
       OR last_success_login < DATEADD(day, -90, CURRENT_TIMESTAMP()))
ORDER BY last_success_login ASC NULLS FIRST;
```

For each inactive user, determine whether to:

- **Disable** the user (preserves the account for potential reactivation)
- **Remove** the user via Terraform (permanent)

#### Review Role Grants

```sql
-- All role grants to users
SELECT
    grantee_name AS user_name,
    role AS granted_role,
    granted_by,
    created_on
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_USERS
WHERE deleted_on IS NULL
ORDER BY grantee_name, role;
```

Check that:

- No user has more access than their job function requires
- Service accounts only have the roles documented in their Terraform configuration
- No direct grants exist outside of Terraform-managed role assignments

### 3. Review Network Policies

#### Check Current Policies

```sql
SHOW NETWORK POLICIES;

-- Check which users are assigned to which policy
SELECT *
FROM TABLE(INFORMATION_SCHEMA.POLICY_REFERENCES(
    POLICY_NAME => '<POLICY_NAME>',
    REF_ENTITY_DOMAIN => 'USER'
));
```

#### Ensure All Service Accounts Have Policies

Service accounts should be restricted to known IP ranges:

| Service Account | Expected IPs |
|----------------|-------------|
| `SVC_TERRAFORM` | GitHub Actions runner IPs |
| `SVC_DLT` | Prefect worker / ECS IPs |
| `SVC_DBT` | dbt Cloud IPs or GitHub Actions IPs |
| `SVC_AIRBYTE` | Airbyte Cloud IPs or ECS IPs |
| `SVC_LIGHTDASH` | Lightdash ECS IPs |

Update `snowflake/config/network_policies.auto.tfvars` if any service accounts lack network policy assignments.

### 4. Audit Snowflake Activity

#### Review Login History

```sql
-- Failed login attempts in the last 7 days
SELECT
    user_name,
    client_ip,
    reported_client_type,
    error_code,
    error_message,
    event_timestamp
FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE event_timestamp > DATEADD(day, -7, CURRENT_TIMESTAMP())
  AND is_success = 'NO'
ORDER BY event_timestamp DESC;
```

#### Review Privilege Grants

```sql
-- Recent privilege changes
SELECT
    grant_option,
    grantee_name,
    granted_by,
    privilege,
    table_catalog,
    table_schema,
    name AS object_name,
    created_on
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_ROLES
WHERE created_on > DATEADD(day, -30, CURRENT_TIMESTAMP())
ORDER BY created_on DESC
LIMIT 50;
```

#### Review Sensitive Operations

```sql
-- Account-level changes (roles, users, integrations)
SELECT
    query_text,
    user_name,
    role_name,
    execution_status,
    start_time
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time > DATEADD(day, -30, CURRENT_TIMESTAMP())
  AND (query_text ILIKE '%CREATE USER%'
       OR query_text ILIKE '%ALTER USER%'
       OR query_text ILIKE '%DROP USER%'
       OR query_text ILIKE '%CREATE ROLE%'
       OR query_text ILIKE '%GRANT%ACCOUNTADMIN%'
       OR query_text ILIKE '%CREATE INTEGRATION%')
ORDER BY start_time DESC;
```

### 5. Harden AWS Configuration

#### Review Secrets Manager Access

```sh
# List all secrets
aws secretsmanager list-secrets --profile infrastructure-admin

# Check who accessed secrets recently
aws cloudtrail lookup-events \
    --lookup-attributes AttributeKey=EventName,AttributeValue=GetSecretValue \
    --start-time "$(date -u -v-7d '+%Y-%m-%dT%H:%M:%SZ')" \
    --profile infrastructure-admin
```

#### Review IAM Role Usage

```sh
# Check recent role assumptions
aws cloudtrail lookup-events \
    --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRole \
    --start-time "$(date -u -v-7d '+%Y-%m-%dT%H:%M:%SZ')" \
    --profile infrastructure-admin
```

### 6. Enable MFA for Human Users

Ensure all human Snowflake users have MFA enabled:

```sql
-- Check MFA status
SELECT name, display_name, has_mfa_token
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE deleted_on IS NULL
  AND name NOT LIKE 'SVC_%'
ORDER BY has_mfa_token ASC, name;
```

For users without MFA, require them to enable it at next login:

```sql
USE ROLE SECURITYADMIN;
ALTER USER <USERNAME> SET MINS_TO_BYPASS_MFA = 0;
```

## Verification

- [ ] All service account keys rotated within the last 90 days
- [ ] No inactive users with active access
- [ ] All service accounts have network policies assigned
- [ ] No failed login attempts from unexpected IPs
- [ ] No unexpected privilege grants in the last 30 days
- [ ] All human users have MFA enabled
- [ ] AWS Secrets Manager access logs show only expected access patterns

## Rollback

Security hardening changes can sometimes lock out legitimate access:

1. **Locked out user** - use ACCOUNTADMIN to re-enable or reset credentials
2. **Network policy too restrictive** - modify the policy via Snowflake UI or SQL using an unaffected admin account
3. **Key rotation broke a pipeline** - restore the previous key from the Secrets Manager version history:

    ```sh
    aws secretsmanager list-secret-version-ids \
        --secret-id "dlt/snowflake-credentials" \
        --profile infrastructure-admin
    ```

!!! danger "Network Policy Warning"
    Account-level network policies affect all users. Always ensure at least one admin account can connect from your current IP before applying restrictive policies. Test with user-level policies first.

## Escalation

- **First contact**: Data Engineering team in #data-eng Slack
- **Security incidents**: Security team immediately
- **Locked out of Snowflake**: ACCOUNTADMIN holder, then Snowflake Support
- **Locked out of AWS**: Account root user (last resort)

## See Also

- [Network Policies](../build/data-warehouse/7-network-policies.md) - IP allowlisting configuration
- [SSO Setup](../build/data-warehouse/9-sso-setup.md) - SAML2 integration
- [Alerting and Incidents](../build/observability/9-alerting-and-incidents.md) - Incident response
