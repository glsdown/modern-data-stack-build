# Prefect Orchestration

On this page, you will:

- [x] Trigger dbt runs from Prefect after ingestion flows complete
- [x] Handle dependency ordering between ingestion and transformation
- [x] Configure error handling and alerting for dbt failures

## Overview

Prefect orchestrates the end-to-end data pipeline: ingestion flows run first, and when they complete successfully, the dbt transformation run is triggered. This keeps the full pipeline dependency graph visible in one place.

```
Prefect (data-pipelines repo)
├── daily_pipeline flow
│   ├── Task: run DLT exchange rates pipeline
│   ├── Task: trigger Airbyte HubSpot sync (wait for completion)
│   └── Task: trigger dbt run (after ingestion tasks succeed)
│
└── weekly_pipeline flow
    ├── Task: run DLT currencies pipeline
    └── Task: trigger dbt full refresh
```

The dbt code lives in the separate `dbt-transform` repository. Prefect pulls the latest version at runtime and runs it.

=== "dbt Core"

    ## dbt Core Integration

    For dbt Core, Prefect clones the `dbt-transform` repository at runtime and uses `DbtCoreOperation` from `prefect-dbt` to run dbt commands.

    ### Install Dependencies

    Add `prefect-dbt` to the `data-pipelines` repository:

    ```sh
    # requirements.txt
    prefect-dbt[snowflake]>=0.6.0
    ```

    ### The Transformation Flow

    Create `pipelines/transform/dbt_flow.py` in the `data-pipelines` repository:

    ```python
    import os
    import subprocess
    import tempfile
    from pathlib import Path

    import boto3
    from prefect import flow, task, get_run_logger
    from prefect.blocks.system import Secret
    from prefect_dbt.cli.commands import DbtCoreOperation


    @task(retries=1, retry_delay_seconds=30)
    def clone_dbt_repo(target_dir: Path) -> None:
        """Clone the dbt-transform repository at the latest main branch commit."""
        logger = get_run_logger()

        # Fetch GitHub token from Secrets Manager
        client = boto3.client("secretsmanager", region_name="eu-west-1")
        secret = client.get_secret_value(SecretId="terraform/github-token")
        github_token = secret["SecretString"]

        repo_url = f"https://{github_token}@github.com/your-org/dbt-transform.git"

        logger.info(f"Cloning dbt-transform repository to {target_dir}")
        result = subprocess.run(
            ["git", "clone", "--depth", "1", "--branch", "main", repo_url, str(target_dir)],
            capture_output=True,
            text=True,
        )
        if result.returncode != 0:
            raise RuntimeError(f"Git clone failed: {result.stderr}")

        logger.info("Repository cloned successfully")


    @task(retries=0)
    def write_dbt_profiles(project_dir: Path, credentials: dict) -> Path:
        """Write a temporary profiles.yml for this run."""
        profiles_dir = project_dir / ".profiles"
        profiles_dir.mkdir(exist_ok=True)
        profiles_path = profiles_dir / "profiles.yml"

        # Write private key to a temp file
        key_path = project_dir / ".profiles" / "snowflake_key.pem"
        key_path.write_text(credentials["private_key"])
        key_path.chmod(0o600)

        profiles_content = f"""
    dbt_transform:
      target: prod
      outputs:
        prod:
          type: snowflake
          account: "{credentials['account']}"
          user: "{credentials['user']}"
          private_key_path: "{key_path}"
          role: "{credentials['role']}"
          warehouse: "{credentials['warehouse']}"
          database: "{credentials['database']}"
          schema: "{credentials['schema']}"
          threads: 8
    """
        profiles_path.write_text(profiles_content.strip())
        return profiles_dir


    @task(retries=1, retry_delay_seconds=60)
    def fetch_dbt_manifest(project_dir: Path) -> None:
        """Fetch the production manifest.json from S3 for state-based operations."""
        logger = get_run_logger()
        artifacts_dir = project_dir / ".artifacts"
        artifacts_dir.mkdir(exist_ok=True)

        s3 = boto3.client("s3", region_name="eu-west-1")
        try:
            s3.download_file(
                "your-org-dbt-artifacts",
                "artifacts/manifest.json",
                str(artifacts_dir / "manifest.json"),
            )
            logger.info("Production manifest fetched successfully")
        except Exception as e:
            logger.warning(f"Could not fetch manifest (first run?): {e}")


    @task(retries=2, retry_delay_seconds=60)
    def run_dbt(project_dir: Path, profiles_dir: Path, commands: list[str]) -> None:
        """Run dbt commands using DbtCoreOperation."""
        logger = get_run_logger()
        logger.info(f"Running dbt commands: {commands}")

        with DbtCoreOperation(
            commands=commands,
            project_dir=str(project_dir),
            profiles_dir=str(profiles_dir),
            overwrite_profiles=False,
        ) as dbt_op:
            dbt_op.run()


    @flow(name="dbt-transformation", log_prints=True)
    def dbt_transformation_flow(full_refresh: bool = False) -> None:
        """
        Run dbt transformations.

        Args:
            full_refresh: If True, run with --full-refresh flag (for weekly runs).
        """
        logger = get_run_logger()

        # Fetch dbt credentials from Secrets Manager
        client = boto3.client("secretsmanager", region_name="eu-west-1")
        secret = client.get_secret_value(SecretId="dbt/snowflake-credentials")
        import json
        credentials = json.loads(secret["SecretString"])

        with tempfile.TemporaryDirectory() as tmp_dir:
            project_dir = Path(tmp_dir) / "dbt-transform"

            # Clone the dbt repo
            clone_dbt_repo(project_dir)

            # Install dbt packages
            subprocess.run(
                ["dbt", "deps", "--project-dir", str(project_dir)],
                check=True,
            )

            # Write credentials
            profiles_dir = write_dbt_profiles(project_dir, credentials)

            # Fetch production manifest for deferral
            fetch_dbt_manifest(project_dir)

            # Check source freshness (warn, don't fail)
            try:
                run_dbt(project_dir, profiles_dir, ["dbt source freshness"])
            except Exception as e:
                logger.warning(f"Source freshness check failed: {e}")

            # Build dbt models
            build_command = "dbt build"
            if full_refresh:
                build_command += " --full-refresh"
            run_dbt(project_dir, profiles_dir, [build_command])

            logger.info("dbt transformation completed successfully")
    ```

    ### Integrate with the Daily Pipeline

    Update `pipelines/daily_pipeline.py` to call the transformation flow after ingestion:

    ```python
    from prefect import flow
    from pipelines.ingest.dlt_flow import exchange_rates_flow
    from pipelines.ingest.airbyte_flow import hubspot_airbyte_flow
    from pipelines.transform.dbt_flow import dbt_transformation_flow


    @flow(name="daily-pipeline", log_prints=True)
    def daily_pipeline() -> None:
        """
        Full daily pipeline: ingestion → transformation.
        Ingestion flows run in parallel; transformation runs after both complete.
        """
        # Run ingestion flows concurrently
        exchange_rates = exchange_rates_flow(return_state=True)
        hubspot = hubspot_airbyte_flow(return_state=True)

        # Wait for both ingestion flows to complete successfully
        if exchange_rates.is_failed() or hubspot.is_failed():
            raise RuntimeError("One or more ingestion flows failed — skipping transformation")

        # Run transformation after ingestion
        dbt_transformation_flow()
    ```

    ### Deploy the Flow

    Update `prefect.yaml` in the `data-pipelines` repository:

    ```yaml
    deployments:
      - name: daily-pipeline
        flow: pipelines/daily_pipeline.py:daily_pipeline
        schedule:
          cron: "0 6 * * *"   # 06:00 UTC daily
          timezone: "UTC"
        work_pool:
          name: your-work-pool
        parameters: {}

      - name: weekly-full-refresh
        flow: pipelines/transform/dbt_flow.py:dbt_transformation_flow
        schedule:
          cron: "0 2 * * 0"  # 02:00 UTC every Sunday
          timezone: "UTC"
        work_pool:
          name: your-work-pool
        parameters:
          full_refresh: true
    ```

    Deploy:

    ```sh
    prefect deploy --all
    ```

=== "dbt Cloud"

    ## dbt Cloud Integration

    For dbt Cloud, Prefect triggers a dbt Cloud job via the API and polls for completion. This keeps scheduling in Prefect while delegating the actual dbt run to dbt Cloud.

    ### Install Dependencies

    ```sh
    # requirements.txt
    prefect-dbt[cloud]>=0.6.0
    ```

    ### Configure the dbt Cloud Block

    Create a Prefect `DbtCloudCredentials` block with the API token:

    ```python
    from prefect_dbt.cloud import DbtCloudCredentials

    # Run once to create the block (or use the Prefect UI)
    DbtCloudCredentials(
        api_key="YOUR_SERVICE_TOKEN",
        account_id=YOUR_ACCOUNT_ID,
    ).save("dbt-cloud-credentials")
    ```

    Or create via the Prefect UI: **Blocks** → **New Block** → **dbt Cloud Credentials**.

    Store the service token in AWS Secrets Manager (`dbt-cloud/api-credentials`) and create the block programmatically in your CI/CD:

    ```python
    # scripts/create_prefect_blocks.py
    import boto3
    import json
    from prefect_dbt.cloud import DbtCloudCredentials

    client = boto3.client("secretsmanager", region_name="eu-west-1")
    secret = json.loads(
        client.get_secret_value(SecretId="dbt-cloud/api-credentials")["SecretString"]
    )

    DbtCloudCredentials(
        api_key=secret["api_token"],
        account_id=int(secret["account_id"]),
    ).save("dbt-cloud-credentials", overwrite=True)
    ```

    ### The Transformation Flow

    Create `pipelines/transform/dbt_cloud_flow.py`:

    ```python
    from prefect import flow, task, get_run_logger
    from prefect_dbt.cloud import DbtCloudJob
    from prefect_dbt.cloud.credentials import DbtCloudCredentials


    @task(retries=2, retry_delay_seconds=60)
    def trigger_dbt_cloud_job(job_id: int, full_refresh: bool = False) -> dict:
        """Trigger a dbt Cloud job and wait for it to complete."""
        logger = get_run_logger()

        credentials = DbtCloudCredentials.load("dbt-cloud-credentials")

        steps_override = None
        if full_refresh:
            steps_override = ["dbt build --full-refresh"]

        logger.info(f"Triggering dbt Cloud job {job_id}")
        job = DbtCloudJob(
            dbt_cloud_credentials=credentials,
            job_id=job_id,
            steps_override=steps_override,
        )

        run = job.trigger()
        run.wait_for_completion()

        artifact_urls = run.get_artifact_link("manifest.json")
        logger.info(f"dbt Cloud job completed. Artifacts: {artifact_urls}")

        return run.get_run_results_artifact()


    @flow(name="dbt-cloud-transformation", log_prints=True)
    def dbt_cloud_transformation_flow(full_refresh: bool = False) -> None:
        """
        Trigger a dbt Cloud job and wait for completion.

        Args:
            full_refresh: If True, override job commands with --full-refresh.
        """
        import json
        import boto3

        client = boto3.client("secretsmanager", region_name="eu-west-1")
        secret = json.loads(
            client.get_secret_value(SecretId="dbt-cloud/api-credentials")["SecretString"]
        )
        job_id = int(secret["job_id_production"])

        trigger_dbt_cloud_job(job_id=job_id, full_refresh=full_refresh)
    ```

    ### Integrate with the Daily Pipeline

    ```python
    from prefect import flow
    from pipelines.ingest.dlt_flow import exchange_rates_flow
    from pipelines.ingest.airbyte_flow import hubspot_airbyte_flow
    from pipelines.transform.dbt_cloud_flow import dbt_cloud_transformation_flow


    @flow(name="daily-pipeline", log_prints=True)
    def daily_pipeline() -> None:
        """Full daily pipeline: ingestion → dbt Cloud transformation."""
        exchange_rates = exchange_rates_flow(return_state=True)
        hubspot = hubspot_airbyte_flow(return_state=True)

        if exchange_rates.is_failed() or hubspot.is_failed():
            raise RuntimeError("Ingestion failed — skipping transformation")

        dbt_cloud_transformation_flow()
    ```

    ### Scheduling Options

    You can schedule the dbt Cloud job directly in dbt Cloud (and not include it in the Prefect daily pipeline), or trigger it from Prefect for dependency ordering. If ingestion timing is variable, triggering from Prefect ensures dbt always runs after fresh data arrives.

## Alerting

Both approaches use Prefect's existing alerting (configured in the [Orchestration Alerting](../orchestration/10-alerting.md) page). A failed dbt run will trigger the same Slack or PagerDuty notifications as a failed ingestion flow.

For dbt-specific context in alerts, add a `on_failure` hook to the flow:

```python
from prefect import flow
from prefect.blocks.notifications import SlackWebhook


def notify_dbt_failure(flow, flow_run, state):
    slack = SlackWebhook.load("slack-webhook")
    slack.notify(
        f"❌ dbt run failed in flow `{flow.name}`\n"
        f"Run ID: {flow_run.id}\n"
        f"Error: {state.message}"
    )


@flow(
    name="dbt-transformation",
    on_failure=[notify_dbt_failure],
    log_prints=True,
)
def dbt_transformation_flow(full_refresh: bool = False) -> None:
    ...
```

## Summary

You've connected Prefect to dbt:

=== "dbt Core"

    - [x] Prefect clones `dbt-transform` at runtime using the GitHub token from Secrets Manager
    - [x] `DbtCoreOperation` runs dbt commands in the cloned project directory
    - [x] Production manifest fetched from S3 before each run for state-based operations
    - [x] Daily pipeline chains ingestion → transformation with dependency ordering
    - [x] Weekly full-refresh deployment runs on Sundays

=== "dbt Cloud"

    - [x] Prefect triggers dbt Cloud jobs via API using `DbtCloudJob`
    - [x] Job completion is polled — Prefect waits before marking the flow as complete
    - [x] dbt Cloud credentials stored in AWS Secrets Manager and loaded via Prefect blocks
    - [x] Daily pipeline chains ingestion → dbt Cloud trigger with dependency ordering

## What's Next

Verify the full pipeline end-to-end and review the complete architecture.

Continue to [Finishing Up](12-finishing-up.md) →
