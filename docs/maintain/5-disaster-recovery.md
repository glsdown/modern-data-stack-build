# Runbook: Disaster Recovery

## Summary

Recover from data loss, infrastructure failures, and service outages across the data stack. This covers Snowflake Time Travel, Terraform state recovery, pipeline failover, and backup strategies.

## When to Use

- Data was accidentally deleted or corrupted in Snowflake
- A Terraform apply destroyed resources unexpectedly
- Terraform state file is corrupted or lost
- A critical pipeline has been failing for an extended period
- An AWS service outage affects the data platform
- Snowflake account access is compromised

## Prerequisites

- [ ] Access: Snowflake with ACCOUNTADMIN role
- [ ] Access: AWS with `infrastructure-admin` profile
- [ ] Access: Terraform state bucket (S3) and lock table (DynamoDB)
- [ ] Context: What was lost, when it happened, and the impact scope

## Steps

### 1. Assess the Situation

Before taking any action, determine:

| Question | Why It Matters |
|----------|---------------|
| **What was affected?** | Scopes the recovery effort |
| **When did it happen?** | Determines Time Travel availability |
| **Is the issue ongoing?** | May need to stop further damage first |
| **Who is impacted?** | Determines urgency and communication needs |
| **Is data recoverable?** | Time Travel, backups, or source re-ingestion |

!!! danger "Stop the Bleeding First"
    If an automated process is causing ongoing damage (e.g. a misconfigured pipeline overwriting data), disable it before attempting recovery. Pause the Prefect deployment, revert the Terraform change, or suspend the Snowflake warehouse.

### 2. Recover Snowflake Data

#### Using Time Travel

Snowflake retains historical data for the retention period (1 day Standard, up to 90 days Enterprise):

**Restore a dropped table:**

```sql
USE ROLE SYSADMIN;
UNDROP TABLE <DATABASE>.<SCHEMA>.<TABLE>;
```

**Restore a table to a point in time:**

```sql
-- Restore to 1 hour ago
CREATE OR REPLACE TABLE <DATABASE>.<SCHEMA>.<TABLE>
  CLONE <DATABASE>.<SCHEMA>.<TABLE> AT (OFFSET => -3600);

-- Restore to a specific timestamp
CREATE OR REPLACE TABLE <DATABASE>.<SCHEMA>.<TABLE>
  CLONE <DATABASE>.<SCHEMA>.<TABLE> AT (TIMESTAMP => '2026-03-01 12:00:00'::TIMESTAMP);

-- Restore to before a specific query
CREATE OR REPLACE TABLE <DATABASE>.<SCHEMA>.<TABLE>
  CLONE <DATABASE>.<SCHEMA>.<TABLE> BEFORE (STATEMENT => '<query_id>');
```

**Restore an entire schema:**

```sql
CREATE OR REPLACE SCHEMA <DATABASE>.<SCHEMA>
  CLONE <DATABASE>.<SCHEMA> AT (OFFSET => -3600);
```

**Restore an entire database:**

```sql
CREATE OR REPLACE DATABASE <DATABASE>
  CLONE <DATABASE> AT (OFFSET => -3600);
```

!!! warning "Time Travel Limits"
    Time Travel only works within the retention period. After that, data enters Fail-safe (7 days, Snowflake support access only). Beyond Fail-safe, data is unrecoverable from Snowflake.

#### Using Clones as Backups

For critical operations (large backfills, schema migrations), create a backup clone first:

```sql
-- Create a backup before a risky operation
CREATE DATABASE <DATABASE>_BACKUP CLONE <DATABASE>;

-- If something goes wrong, swap back
ALTER DATABASE <DATABASE> RENAME TO <DATABASE>_BROKEN;
ALTER DATABASE <DATABASE>_BACKUP RENAME TO <DATABASE>;

-- Clean up after confirming recovery
DROP DATABASE <DATABASE>_BROKEN;
```

Zero-copy clones are free (they share storage until data diverges).

### 3. Recover Terraform State

#### Corrupted State File

S3 versioning is enabled on the Terraform state bucket. Restore a previous version:

```sh
# List state file versions
aws s3api list-object-versions \
    --bucket your-org-terraform-state \
    --prefix snowflake/terraform.tfstate \
    --profile infrastructure-admin

# Download a specific version
aws s3api get-object \
    --bucket your-org-terraform-state \
    --key snowflake/terraform.tfstate \
    --version-id <VERSION_ID> \
    restored-state.tfstate \
    --profile infrastructure-admin

# Upload the restored state
aws s3 cp restored-state.tfstate \
    s3://your-org-terraform-state/snowflake/terraform.tfstate \
    --profile infrastructure-admin
```

!!! danger "DynamoDB Lock"
    If a Terraform operation was interrupted, the DynamoDB lock may be stuck. Check and remove it:

    ```sh
    aws dynamodb scan \
        --table-name your-org-terraform-locks \
        --profile infrastructure-admin
    ```

    If a stale lock exists, delete it via the AWS Console or CLI. Only do this if you are certain no other Terraform operation is running.

#### Destroyed Resources

If `terraform apply` destroyed resources unexpectedly:

1. **Check the Terraform plan output** to understand what was destroyed
2. **Revert the PR** that caused the destruction
3. **Re-apply** to recreate the resources from the reverted configuration
4. **Re-import** any resources that can't be recreated automatically (e.g. `terraform import`)
5. **Restore data** using Time Travel if databases or schemas were dropped

### 4. Recover Pipeline State

#### Prefect Flow Failures

If a critical pipeline has been failing:

1. **Check the flow run logs** in Prefect UI for the root cause
2. **Fix the underlying issue** (credentials, source availability, schema change)
3. **Determine the data gap** - what date range was missed
4. **Run a backfill** to fill the gap (see [Backfills](3-backfills.md))
5. **Verify downstream models** are correct after the backfill

#### dlt Pipeline State

dlt tracks pipeline state (last loaded record, incremental cursors) in its state store. If this is corrupted:

```sh
# Reset pipeline state (forces full reload on next run)
cd ~/projects/data/data-pipelines
python -c "
import dlt
pipeline = dlt.pipeline(pipeline_name='<pipeline_name>')
pipeline.drop()
"
```

Then run a full reload or backfill.

### 5. Recover from AWS Outages

If an AWS service outage affects the platform:

| Affected Service | Impact | Mitigation |
|-----------------|--------|-----------|
| **S3** | Data lake unavailable, Snowpipe paused | Wait for recovery; Snowpipe auto-catches up |
| **Secrets Manager** | Pipelines can't authenticate | Use cached credentials if available; wait for recovery |
| **ECS** | Self-hosted services down (Prefect, Airbyte, Lightdash) | Wait for recovery; consider cloud alternatives |
| **DynamoDB** | Terraform state locking unavailable | Wait; do not run Terraform without locking |

Monitor AWS service health at [status.aws.amazon.com](https://status.aws.amazon.com/).

### 6. Recover from Account Compromise

If a Snowflake or AWS account may be compromised:

1. **Rotate all service account credentials immediately:**

    ```sh
    # Generate new key pairs for each service account
    openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out new_key.pem -nocrypt
    openssl rsa -in new_key.pem -pubout -out new_key.pub
    ```

2. **Update Snowflake public keys:**

    ```sql
    USE ROLE SECURITYADMIN;
    ALTER USER SVC_DLT SET RSA_PUBLIC_KEY = '<new public key>';
    ALTER USER SVC_DBT SET RSA_PUBLIC_KEY = '<new public key>';
    ALTER USER SVC_AIRBYTE SET RSA_PUBLIC_KEY = '<new public key>';
    ```

3. **Update AWS Secrets Manager** with the new private keys
4. **Review Snowflake access history** for suspicious activity:

    ```sql
    SELECT *
    FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
    WHERE event_timestamp > DATEADD(day, -7, CURRENT_TIMESTAMP())
    ORDER BY event_timestamp DESC;
    ```

5. **Enable network policies** to restrict access if not already in place
6. **Report the incident** to your security team

## Verification

- [ ] Affected data restored to the correct state
- [ ] All pipelines running successfully
- [ ] dbt tests pass on affected models
- [ ] Dashboards showing correct data
- [ ] No residual errors in Prefect, dbt, or Snowflake logs
- [ ] Credentials rotated (if compromise was involved)

## Rollback

Recovery operations are themselves at risk of causing issues:

1. **Before restoring** - create a clone of the current state as a safety net
2. **After restoring** - verify data integrity before deleting backups
3. **Keep clones for 24-48 hours** after recovery to ensure no secondary issues emerge

## Escalation

- **First contact**: Data Engineering team in #data-eng Slack
- **Escalation**: Infrastructure team lead
- **Security incidents**: Security team immediately, then infrastructure team lead
- **Snowflake platform issues**: [Snowflake Support](https://community.snowflake.com/s/support) (Enterprise accounts)

## See Also

- [Backfills](3-backfills.md) - Historical data reprocessing
- [Security Hardening](6-security-hardening.md) - Key rotation and audit logging
- [Alerting and Incidents](../build/observability/9-alerting-and-incidents.md) - Incident response workflows
