# Runbook: Adding Data Sources

## Summary

Add a new data source to the platform - including Snowflake infrastructure (database, service account, schemas), ingestion pipeline, dbt staging models, and orchestration. This runbook covers the end-to-end process across all three repositories.

## When to Use

- A new SaaS tool, API, or database needs to be ingested
- A new category of data requires its own loader database
- A new team wants to bring data into the warehouse

## Prerequisites

- [ ] Access: Write access to `terraform`, `data-pipelines`, and `dbt-transform` repositories
- [ ] Tools: Terraform CLI, AWS CLI (`infrastructure-admin` profile), dlt or Airbyte configured
- [ ] Context: Source system name, expected schemas, ingestion method (dlt, Airbyte, Snowpipe, or streaming)

## Steps

### 1. Choose the Ingestion Method

| Method | Best For | Repository | Destination Database |
|--------|----------|-----------|---------------------|
| **dlt** | APIs, databases, 1-2 simple sources | data-pipelines | `DLT` |
| **Airbyte** | Complex SaaS (5+ sources), non-engineer config | Airbyte Cloud/self-hosted | `AIRBYTE` |
| **Snowpipe** | File-based (S3 auto-ingest) | terraform (pipe config) | `SNOWPIPE` |
| **Kafka Connect** | Real-time event streams | Confluent Cloud | `STREAMING` |

If the source fits into an existing loader database (e.g. a new API via dlt goes into the `DLT` database), skip to Step 3 - you only need a new schema, not a new database.

If the source requires a new loader tool (e.g. adding Fivetran), follow all steps.

### 2. Create Snowflake Infrastructure (New Loader Only)

!!! tip "Claude Code Automation"
    The `add-data-source` Claude skill in the terraform repository automates steps 2a-2d. See [Claude Code Setup](../getting-started/initial-setup/5-claude-code-setup.md).

#### 2a. Create the Database

Add to `snowflake/config/databases.tf`:

```hcl
module "database_<loader>" {
  source = "./modules/snowflake_database"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  database_name    = "<LOADER>"
  database_comment = "Raw data loaded by <loader>."

  grant_reader_to_account_roles = [
    module.role_analytics_sources_reader.role_name,
  ]
  grant_writer_to_account_roles = [
    module.user_svc_<loader>.user_default_role,
  ]
}
```

The database module creates `<LOADER>_DB_READER` and `<LOADER>_DB_WRITER` database roles. Granting the reader to `ANALYTICS_SOURCES_READER` maintains the reader access chain so developers and transformers can query the data.

#### 2b. Create the Service Account

Add to `snowflake/config/users.tf`:

```hcl
module "user_svc_<loader>" {
  source = "./modules/snowflake_user"

  providers = {
    snowflake.security_admin = snowflake.security_admin
    snowflake.user_admin     = snowflake.user_admin
  }

  user_name                  = "SVC_<LOADER>"
  user_display_name          = "<Loader> Service Account"
  user_comment               = "Service account for <loader> data loading."
  user_is_service_account    = true
  user_create_dedicated_role = true
  user_default_warehouse     = module.warehouse_loading.warehouse_name

  user_additional_roles = []
}
```

#### 2c. Add AWS Secrets Manager Container

Add to `aws/config/secrets.tf`:

```hcl
resource "aws_secretsmanager_secret" "<loader>_snowflake_credentials" {
  name        = "<loader>/snowflake-credentials"
  description = "Snowflake credentials for SVC_<LOADER>."
}
```

#### 2d. Validate and Deploy

```sh
cd snowflake/config && terraform plan
cd ../../aws/config && terraform plan
```

Create a PR, get approval, and merge. CI/CD applies the changes.

After merge, generate a key pair and store credentials in Secrets Manager (see [Adding Users](1-adding-users.md) step 6 for the key pair process).

### 3. Create Schemas

Add to `snowflake/config/schemas.tf` for each data source within the database:

```hcl
module "schema_<loader>_<source>" {
  source = "./modules/snowflake_schema"

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }

  database_name  = module.database_<loader>.database_name
  schema_name    = "<SOURCE>"
  schema_comment = "Data from <source> loaded by <loader>."
}
```

Repeat for each source (e.g. `HUBSPOT`, `STRIPE`, `SALESFORCE`). Create a PR and merge.

### 4. Build the Ingestion Pipeline

=== "dlt Pipeline"

    In the `data-pipelines` repository:

    1. **Create the source** in `sources/<source_name>/`:

        ```python
        import dlt

        @dlt.source(section="<source_name>")
        def <source_name>():
            @dlt.resource(write_disposition="replace")
            def <table_name>():
                # Fetch data from the API
                yield from api_client.get_data()

            return <table_name>
        ```

    2. **Create the pipeline** in `pipelines/<source_name>.py`:

        ```python
        import dlt
        from sources.<source_name> import <source_name>

        def run():
            pipeline = dlt.pipeline(
                pipeline_name="<source_name>",
                destination="snowflake",
                dataset_name="<SOURCE>",
            )
            load_info = pipeline.run(<source_name>())
            return load_info
        ```

    3. **Add credentials** to `.dlt/secrets.toml` (local) and AWS Secrets Manager (production)

    4. **Test locally with DuckDB first**:

        ```sh
        # Temporarily change destination to duckdb for testing
        python -m pipelines.<source_name>
        ```

=== "Airbyte Connection"

    In Airbyte Cloud or self-hosted UI:

    1. Add the source connector and configure credentials
    2. Add the Snowflake destination (using `SVC_AIRBYTE` credentials)
    3. Set the destination namespace to `<SOURCE>` within the `AIRBYTE` database
    4. Configure sync frequency and select streams
    5. Run an initial sync and verify data appears

    See [HubSpot Connection](../build/saas-ingestion/6-hubspot-connection.md) for a worked example.

=== "Snowpipe"

    1. Add a storage integration in `snowflake/config/storage_integrations.tf` (if not already present)
    2. Create the pipe configuration using the `snowflake_snowpipe` module
    3. Configure S3 event notifications to trigger the pipe
    4. Write a pipeline that uploads files to the S3 staging location

    See [Exchange Rates to S3](../build/batch-data-ingestion/6-exchange-rates-to-s3.md) for a worked example.

### 5. Create Prefect Flow (if Using dlt)

Create `flows/<source_name>.py`:

```python
"""Prefect flow for <source_name> pipeline."""

from prefect import flow, task, get_run_logger
from prefect.tasks import exponential_backoff


@task(
    name="load-<source-name>",
    retries=3,
    retry_delay_seconds=exponential_backoff(backoff_factor=10),
    retry_jitter_factor=0.5,
)
def load_task():
    """Load <source_name> data to Snowflake."""
    logger = get_run_logger()
    logger.info("Starting <source_name> load")

    from pipelines.<source_name> import run

    load_info = run()
    logger.info(f"Load completed: {load_info}")
    return load_info


@flow(name="<source-name>-daily", log_prints=True)
def daily_flow():
    """Daily <source_name> load."""
    load_info = load_task()
    print(f"<Source> loaded: {load_info}")
    return {"status": "success", "load_info": str(load_info)}


if __name__ == "__main__":
    daily_flow()
```

Add the deployment to `prefect.yaml`:

```yaml
deployments:
  # ... existing deployments ...
  - name: <source-name>-daily
    entrypoint: flows/<source_name>.py:daily_flow
    description: "Daily <source_name> load"
    work_pool:
      name: production
    schedules:
      - cron: "0 7 * * *"
        timezone: "UTC"
        active: true
    tags:
      - dlt
      - <source-name>
      - daily
```

Deploy:

```sh
prefect deploy --all
```

### 6. Add dbt Source and Staging Model

In the `dbt-transform` repository:

1. **Create or update `_sources.yml`** in `models/staging/<loader>/`:

    ```yaml
    version: 2

    sources:
      - name: <loader>_<source>
        database: <LOADER>
        schema: <SOURCE>
        loaded_at_field: _dlt_load_time  # or _airbyte_extracted_at for Airbyte
        freshness:
          warn_after: { count: 24, period: hour }
          error_after: { count: 48, period: hour }
        tables:
          - name: <table_name>
            description: "<Description of the table>"
    ```

2. **Create the staging model** `models/staging/<loader>/stg_<loader>__<table>.sql`:

    ```sql
    with source as (

        select * from {{ source('<loader>_<source>', '<table_name>') }}

    ),

    renamed as (

        select
            -- identifiers
            id as <entity>_id,

            -- dimensions
            column_a,
            column_b,

            -- timestamps
            cast(created_at as timestamp_ntz) as created_at

        from source

    )

    select * from renamed
    ```

3. **Create the model YAML** `models/staging/<loader>/stg_<loader>__<table>.yml`:

    ```yaml
    version: 2

    models:
      - name: stg_<loader>__<table>
        description: >
          <Description>. One row per <entity>.
        columns:
          - name: <entity>_id
            description: Primary key.
            tests:
              - unique
              - not_null
    ```

4. **Run and test**:

    ```sh
    dbt build --select stg_<loader>__<table>
    ```

## Verification

- [ ] Snowflake database exists (if new loader):

    ```sql
    SHOW DATABASES LIKE '<LOADER>';
    ```

- [ ] Schema exists:

    ```sql
    SHOW SCHEMAS IN DATABASE <LOADER>;
    ```

- [ ] Reader access chain works:

    ```sql
    USE ROLE ANALYTICS_DEVELOPER;
    SELECT * FROM <LOADER>.<SOURCE>.<TABLE> LIMIT 10;
    ```

- [ ] Pipeline runs successfully (Prefect UI or manual run)
- [ ] dbt staging model builds and tests pass:

    ```sh
    dbt build --select stg_<loader>__<table>
    ```

- [ ] Source freshness check passes:

    ```sh
    dbt source freshness --select source:<loader>_<source>
    ```

## Rollback

If something goes wrong at any stage:

1. **Terraform changes** - Revert the PR in the terraform repository. CI/CD runs `terraform apply` to remove the resources
2. **Pipeline code** - Revert the PR in the data-pipelines repository
3. **dbt models** - Revert the PR in the dbt-transform repository. Run `dbt run` to clean up any partially created models
4. **Snowflake data** (manual cleanup):

    ```sql
    USE ROLE SYSADMIN;
    DROP SCHEMA IF EXISTS <LOADER>.<SOURCE> CASCADE;
    -- Only if removing the entire loader:
    DROP DATABASE IF EXISTS <LOADER>;
    ```

## Escalation

- **First contact**: Data Engineering team in #data-eng Slack
- **Escalation**: Infrastructure team lead (for Terraform), pipeline team lead (for ingestion)

## See Also

- [Databases](../build/data-warehouse/3-databases.md) - Database module patterns
- [Schemas](../build/data-warehouse/6-schemas.md) - Schema module patterns
- [Batch Data Ingestion](../build/batch-data-ingestion/index.md) - Full dlt pipeline guide
- [SaaS Ingestion](../build/saas-ingestion/index.md) - Full Airbyte guide
- [Sources and Staging](../build/data-transformation/5-sources-and-staging.md) - dbt staging patterns
