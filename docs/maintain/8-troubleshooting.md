# Runbook: Troubleshooting

## Summary

Diagnose and resolve common issues across the data stack. This runbook provides systematic debugging approaches for each layer - Snowflake, dbt, Prefect, dlt, Airbyte, and infrastructure.

## When to Use

- A pipeline, model, or query is failing and the cause is unclear
- An alert fires and the standard runbook doesn't resolve the issue
- Something worked before and now doesn't

## Prerequisites

- [ ] Access: Relevant tool dashboards (Prefect, Snowflake, dbt Cloud)
- [ ] Access: AWS CloudWatch Logs (for ECS-hosted services)
- [ ] Context: Error message, affected component, when it last worked

## Steps

### 1. General Debugging Approach

For any issue, follow this sequence:

1. **Read the error message** - most errors are descriptive
2. **Check when it last worked** - what changed since then?
3. **Check the logs** - Prefect UI, CloudWatch, Snowflake query history
4. **Reproduce locally** (if possible) - run the failing pipeline or model in dev
5. **Fix, test, deploy** - fix in a branch, test locally, create a PR

### 2. Snowflake Issues

#### Authentication Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `Incorrect username or password` | Wrong credentials | Check `.dlt/secrets.toml` or Secrets Manager |
| `JWT token is invalid` | Expired or wrong key pair | Rotate the key (see [Security Hardening](6-security-hardening.md)) |
| `User is disabled` | User account disabled | Re-enable: `ALTER USER <name> SET DISABLED = FALSE;` |
| `IP not allowed` | Network policy blocking | Check network policy assignments and allowed IPs |

#### Permission Errors

```
Insufficient privileges to operate on <object>
```

1. **Check the user's current role:**

    ```sql
    SELECT CURRENT_USER(), CURRENT_ROLE();
    ```

2. **Check what roles are granted:**

    ```sql
    SHOW GRANTS TO USER <username>;
    ```

3. **Check what the role can access:**

    ```sql
    SHOW GRANTS TO ROLE <role_name>;
    ```

4. **Common fix** - the user needs a different role. Either switch roles or update Terraform:

    ```sql
    -- Temporary fix
    USE ROLE <correct_role>;

    -- Permanent fix via Terraform
    -- Add the role to user_additional_roles in users.tf or users.auto.tfvars
    ```

#### Query Timeouts

```
Statement reached its maximum execution time of 300 seconds
```

1. **Check the query profile** in Snowsight → Activity → Query History
2. **Identify the bottleneck** (spilling, full scan, exploding JOIN)
3. **Temporary fix** - increase timeout:

    ```sql
    ALTER SESSION SET STATEMENT_TIMEOUT_IN_SECONDS = 600;
    ```

4. **Permanent fix** - optimise the query (see [Performance Optimisation](4-performance-optimization.md))

#### Warehouse Suspended

```
Warehouse is suspended. Resume warehouse to execute queries.
```

If `AUTO_RESUME = TRUE` (which it should be), this error usually indicates a resource monitor has suspended the warehouse:

```sql
SHOW RESOURCE MONITORS;
-- Check if any monitor has hit its credit limit
```

To resume temporarily:

```sql
USE ROLE SYSADMIN;
ALTER WAREHOUSE <warehouse> RESUME;
```

To increase the credit limit:

```sql
USE ROLE ACCOUNTADMIN;
ALTER RESOURCE MONITOR <monitor> SET CREDIT_QUOTA = <new_limit>;
```

### 3. dbt Issues

#### Model Compilation Errors

```
Compilation Error in model <model_name>
```

1. **Check the error details** - usually a missing ref, source, or macro
2. **Common causes:**
    - Missing source definition in `_sources.yml`
    - Typo in `{{ ref('model_name') }}` or `{{ source('name', 'table') }}`
    - Package not installed: run `dbt deps`

#### Test Failures

```
Failure in test <test_name>
```

1. **Check what data failed:**

    ```sh
    dbt test --select <test_name> --store-failures
    ```

    Failed rows are stored in the `dbt_test__audit` schema.

2. **Query the failures:**

    ```sql
    SELECT * FROM ANALYTICS_DEV.DBT_TEST__AUDIT.<test_name>;
    ```

3. **Common causes:**
    - Source data changed (new NULL values, duplicates)
    - Upstream model bug introduced in a recent PR
    - Test threshold too strict (e.g. freshness, accepted values)

#### Incremental Model Drift

If an incremental model is producing wrong results:

1. **Run a full refresh** to rebuild from scratch:

    ```sh
    dbt run --select <model> --full-refresh
    ```

2. **Compare row counts** before and after
3. **If the issue recurs**, check the `is_incremental()` filter logic

#### Schema Changes

```
Compilation Error: column "new_column" does not exist
```

Source schemas evolve. When upstream tools add or rename columns:

1. **Check what changed** in the source table:

    ```sql
    DESCRIBE TABLE <DATABASE>.<SCHEMA>.<TABLE>;
    ```

2. **Update the staging model** to handle the new schema
3. **Update `_sources.yml`** if table structure changed significantly
4. **Consider `on_schema_change='append_new_columns'`** for incremental models that should absorb new columns automatically

### 4. Prefect Issues

#### Flow Run Failed

1. **Check the flow run logs** in Prefect UI → Flow Runs → click the failed run
2. **Common causes:**

    | Error Pattern | Cause | Fix |
    |--------------|-------|-----|
    | `ModuleNotFoundError` | Missing dependency | Add to `pyproject.toml` with `uv add` |
    | `ConnectionRefusedError` | Source API or database down | Wait and retry |
    | `PermissionError` | Missing credentials or expired token | Check Secrets Manager |
    | `TimeoutError` | Source API slow or unresponsive | Increase timeout or add retries |

3. **Retry the run** if the failure is transient:
    - In Prefect UI, click **Retry** on the failed run
    - Or trigger a new run via CLI:

        ```sh
        prefect deployment run <deployment-name>/production
        ```

#### Worker Not Picking Up Runs

1. **Check worker status** in Prefect UI → Work Pools → select pool → Workers tab
2. **If no workers are online:**
    - **Prefect Cloud**: Check ECS service or the machine running the worker
    - **Self-hosted**: Check the worker container:

        ```sh
        docker compose logs prefect-worker
        ```

3. **Common cause** - the worker lost connection to the Prefect API. Restart it.

#### Deployment Not Found

```
Deployment not found
```

The deployment was deleted or never created. Redeploy:

```sh
cd ~/projects/data/data-pipelines
prefect deploy --all
```

### 5. dlt Issues

#### Schema Evolution Errors

```
Schema has changed
```

dlt detects schema changes automatically. If a source API returns new fields:

1. **Check the schema file** in `.dlt/schemas/`:

    ```sh
    cat .dlt/schemas/<pipeline_name>/schema.json
    ```

2. **If the change is expected**, let dlt evolve the schema automatically (default behaviour)
3. **If the change is unexpected**, investigate the source API for breaking changes

#### Credential Resolution Failures

```
ConfigFieldMissingException: Missing config field
```

dlt resolves credentials in this order: environment variables → `secrets.toml` → `config.toml` → custom providers (e.g. AWS Secrets Manager).

1. **Check `.dlt/secrets.toml`** for local development
2. **Check AWS Secrets Manager** for production
3. **Verify the secret path matches** what the pipeline expects (section name in `@dlt.source(section="...")`)

### 6. Infrastructure Issues

#### Terraform State Lock

```
Error acquiring the state lock
```

Another Terraform operation is running, or a previous operation was interrupted.

1. **Check who holds the lock:**

    ```sh
    aws dynamodb scan \
        --table-name your-org-terraform-locks \
        --profile infrastructure-admin
    ```

2. **If the lock is stale** (the operation is no longer running):

    ```sh
    terraform force-unlock <LOCK_ID>
    ```

    !!! danger "Force Unlock"
        Only force-unlock if you are absolutely certain no other Terraform operation is running. Forcing an unlock while another process is applying can corrupt state.

#### CI/CD Pipeline Failures

1. **Check GitHub Actions logs** for the failing workflow
2. **Common causes:**

    | Error | Fix |
    |-------|-----|
    | OIDC authentication failed | Check IAM role trust policy includes the repository |
    | Secret not found | Verify the secret exists in AWS Secrets Manager |
    | Terraform plan shows drift | Someone made manual changes - import or reconcile |
    | dbt build failed | Check test failures, schema changes |

#### ECS Service Unhealthy

For self-hosted services (Prefect, Airbyte, Lightdash):

1. **Check ECS service events** in AWS Console → ECS → Clusters → Services
2. **Check CloudWatch Logs** for the service's log group
3. **Common causes:**
    - Container failing health check → check application logs
    - Out of memory → increase task definition memory
    - Image pull failure → verify ECR repository or image tag

## Verification

After resolving any issue:

- [ ] The immediate error is resolved
- [ ] Related pipelines and models run successfully
- [ ] No cascading failures in downstream systems
- [ ] Alerts have cleared (Prefect, PagerDuty, Slack)
- [ ] Root cause is understood and documented

## Rollback

Most troubleshooting involves fixing forward rather than rolling back. If a fix introduces new issues:

1. **Revert the PR** that introduced the fix
2. **Restore data** using Snowflake Time Travel if needed
3. **Restart services** that may be in a bad state
4. **Escalate** if the issue is beyond your expertise

## Escalation

- **First contact**: Data Engineering team in #data-eng Slack
- **Snowflake platform issues**: Snowflake Support (Enterprise accounts)
- **AWS infrastructure**: Infrastructure team lead
- **Vendor-specific issues**: Raise a support ticket with the vendor (Prefect, Airbyte, Lightdash)

## See Also

- [Logging and Debugging](../build/observability/10-logging-and-debugging.md) - Centralised logging setup
- [Alerting and Incidents](../build/observability/9-alerting-and-incidents.md) - Alert configuration and incident workflows
- [Performance Optimisation](4-performance-optimization.md) - Query and pipeline performance
- [Disaster Recovery](5-disaster-recovery.md) - Data loss and infrastructure recovery
