# Prefect Concepts

On this page, you will:

- [x] Understand flows and tasks
- [x] Learn how deployments work
- [x] Understand work pools and workers
- [x] Learn about blocks for configuration
- [x] Understand schedules and triggers

## Overview

Before setting up Prefect, it's important to understand its core concepts. These concepts apply whether you're using Prefect Cloud or self-hosting.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          PREFECT ARCHITECTURE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Your Code                          Prefect (Cloud or Server)               │
│  ─────────                          ─────────────────────────               │
│  ┌─────────────────┐                ┌─────────────────────────┐             │
│  │     Flows       │  ──deploy──▶   │     Deployments         │             │
│  │   (Python)      │                │   (Config + Schedule)   │             │
│  └─────────────────┘                └───────────┬─────────────┘             │
│                                                 │                           │
│  ┌─────────────────┐                            │ trigger                   │
│  │    Workers      │  ◀──────────────────────── │                           │
│  │  (Executors)    │                            ▼                           │
│  └────────┬────────┘                ┌─────────────────────────┐             │
│           │                         │      Work Pools         │             │
│           │ run                     │   (Job Configuration)   │             │
│           ▼                         └─────────────────────────┘             │
│  ┌─────────────────┐                                                        │
│  │   Flow Runs     │  ──logs/status──▶  UI & API                            │
│  │   (Execution)   │                                                        │
│  └─────────────────┘                                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Flows and Tasks

### Tasks

A **task** is the smallest unit of work in Prefect. It's a Python function decorated with `@task`:

```python
from prefect import task

@task
def extract_data(source: str) -> dict:
    """Extract data from a source."""
    # Your extraction logic here
    return {"records": [...]}

@task
def transform_data(data: dict) -> dict:
    """Transform the extracted data."""
    # Your transformation logic here
    return {"transformed": [...]}

@task
def load_data(data: dict, destination: str) -> int:
    """Load data to destination."""
    # Your loading logic here
    return len(data["transformed"])
```

Tasks provide:

- **Retries** - Automatic retry on failure with configurable backoff
- **Caching** - Skip re-execution if inputs haven't changed
- **Timeouts** - Prevent runaway tasks
- **Logging** - Automatic logging of inputs, outputs, and duration
- **Concurrency** - Control how many instances run simultaneously

### Flows

A **flow** is a container for tasks. It's a Python function decorated with `@flow`:

```python
from prefect import flow

@flow(name="etl-pipeline")
def etl_pipeline(source: str, destination: str):
    """Extract, transform, and load data."""
    # Tasks are called like normal functions
    raw_data = extract_data(source)
    transformed = transform_data(raw_data)
    count = load_data(transformed, destination)

    return f"Loaded {count} records"
```

Flows provide:

- **Orchestration** - Coordinate task execution
- **State management** - Track success, failure, and pending states
- **Observability** - See flow runs in the UI
- **Parameters** - Accept inputs that can vary per run

### Running Flows Locally

You can run any flow locally during development. Here's a complete example file:

```python
# my_flow.py
from prefect import flow, task

@task
def extract_data(source: str) -> dict:
    return {"records": ["a", "b", "c"]}

@task
def transform_data(data: dict) -> dict:
    return {"transformed": [x.upper() for x in data["records"]]}

@task
def load_data(data: dict, destination: str) -> int:
    return len(data["transformed"])

@flow(name="etl-pipeline")
def etl_pipeline(source: str, destination: str):
    raw_data = extract_data(source)
    transformed = transform_data(raw_data)
    count = load_data(transformed, destination)
    return f"Loaded {count} records"

# Run when executed directly
if __name__ == "__main__":
    result = etl_pipeline(
        source="api.example.com",
        destination="snowflake"
    )
    print(result)
```

```sh
python my_flow.py
```

This is one of Prefect's key advantages — your flows are standard Python, so you can develop and test them locally before deploying.

## Deployments

A **deployment** packages a flow with configuration for remote execution:

- **Where** is the code? (Git repository, S3, local path)
- **How** should it run? (Work pool, infrastructure settings)
- **When** should it run? (Schedule, triggers)
- **What** parameters should it use? (Default values)

### Creating a Deployment

Deployments are defined in a `prefect.yaml` file:

```yaml
# prefect.yaml
name: data-pipelines

deployments:
  - name: etl-pipeline-daily
    entrypoint: flows/etl.py:etl_pipeline
    work_pool:
      name: my-work-pool
    parameters:
      source: "api.example.com"
      destination: "snowflake"
    schedules:
      - cron: "0 6 * * *"  # Daily at 6 AM
        timezone: "Europe/London"
```

Deploy with:

```sh
prefect deploy --name etl-pipeline-daily
```

### Deployment Lifecycle

```
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│   Define     │ ──▶  │   Deploy     │ ──▶  │   Trigger    │
│  (YAML/CLI)  │      │  (prefect    │      │  (Schedule,  │
│              │      │   deploy)    │      │   API, UI)   │
└──────────────┘      └──────────────┘      └──────┬───────┘
                                                   │
                                                   ▼
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│   Complete   │ ◀──  │   Execute    │ ◀──  │   Worker     │
│  (Success/   │      │   (Flow      │      │   Picks Up   │
│   Failure)   │      │    Run)      │      │              │
└──────────────┘      └──────────────┘      └──────────────┘
```

## Work Pools and Workers

### Work Pools

A **work pool** defines the infrastructure where flows run. Think of it as a job queue with configuration:

| Work Pool Type | Use Case | Infrastructure |
|----------------|----------|----------------|
| `process` | Local development, simple deployments | Local machine or EC2 |
| `docker` | Containerised flows | Docker daemon |
| `ecs` | AWS serverless | ECS Fargate |
| `kubernetes` | Kubernetes clusters | K8s pods |

Each work pool has a **base job template** that defines default settings for all flows in that pool.

### Workers

A **worker** is a process that polls a work pool for jobs and executes them. Workers:

- Run on your infrastructure (EC2, your laptop, ECS)
- Pick up flow runs from work pools
- Execute flows and report status back to Prefect
- Can be scaled horizontally

```sh
# Start a worker for a process work pool
prefect worker start --pool my-work-pool
```

### Work Pool + Worker Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              PREFECT                                    │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                        Work Pools                               │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │    │
│  │  │ dev-pool    │  │ prod-pool   │  │ ml-pool     │              │    │
│  │  │ (process)   │  │ (ecs)       │  │ (docker)    │              │    │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘              │    │
│  └─────────│────────────────│────────────────│─────────────────────┘    │
│            │                │                │                          │
└────────────│────────────────│────────────────│──────────────────────────┘
             │                │                │
    ┌────────▼────────┐ ┌─────▼─────┐ ┌────────▼────────┐
    │    Worker       │ │  Worker   │ │    Worker       │
    │  (EC2 / Local)  │ │  (ECS)    │ │  (EC2 + Docker) │
    └─────────────────┘ └───────────┘ └─────────────────┘
```

## Blocks

**Blocks** are reusable configuration objects for external systems. They store credentials, connection details, and settings:

```python
from prefect_aws import S3Bucket
from prefect_snowflake import SnowflakeConnector

# Create blocks via code
s3_block = S3Bucket(
    bucket_name="my-data-lake",
    aws_access_key_id="...",
    aws_secret_access_key="..."
)
s3_block.save("data-lake-bucket")

# Use blocks in flows
@task
def load_to_snowflake(data: dict):
    snowflake = SnowflakeConnector.load("warehouse-connection")
    # Use the connection...
```

Common block types:

| Block | Purpose |
|-------|---------|
| `AWS Credentials` | AWS access for S3, Secrets Manager, ECS |
| `Snowflake Connector` | Snowflake connection details |
| `S3 Bucket` | S3 bucket configuration |
| `GitHub` | Repository access for code storage |
| `Secret` | Store sensitive values |

Blocks can be created via:

- **Python code** (as shown above)
- **Prefect UI** (point-and-click)
- **Terraform** (infrastructure as code)

## Schedules and Triggers

### Schedules

Deployments can run on schedules using cron expressions:

```yaml
# prefect.yaml
deployments:
  - name: daily-etl
    schedules:
      - cron: "0 6 * * *"        # Daily at 6 AM
        timezone: "Europe/London"
      - cron: "0 */4 * * *"      # Every 4 hours
        timezone: "UTC"
```

Common cron patterns:

| Pattern | Meaning |
|---------|---------|
| `0 6 * * *` | Daily at 6:00 AM |
| `0 */2 * * *` | Every 2 hours |
| `0 0 * * 1` | Weekly on Monday at midnight |
| `0 0 1 * *` | Monthly on the 1st at midnight |
| `*/15 * * * *` | Every 15 minutes |

### Triggers and Automations

Beyond schedules, Prefect supports event-driven triggers:

- **Automations** - React to events (flow failed, stuck, completed)
- **Webhooks** - Trigger flows from external systems
- **API** - Trigger flows programmatically

Example automation: "If the ETL flow fails, send a Slack notification and retry once."

## State Management

Every flow and task run has a **state** that tracks its lifecycle:

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Pending  │ ─▶ │ Running  │ ─▶ │Completed │    │  Failed  │
└──────────┘    └────┬─────┘    └──────────┘    └──────────┘
                     │                               ▲
                     └───────── on error ────────────┘
```

States enable:

- **Retries** - Automatically retry failed tasks
- **Caching** - Skip tasks with cached results
- **Conditional logic** - Branch based on upstream state
- **Alerting** - Notify on specific state transitions

## Putting It Together

Here's how all the concepts work together in a typical workflow:

1. **Write flows and tasks** in Python
2. **Test locally** by running the flow directly
3. **Create a deployment** with `prefect.yaml`
4. **Deploy** with `prefect deploy`
5. **Start a worker** to execute flow runs
6. **Configure schedules or triggers** in the deployment
7. **Monitor** via the Prefect UI

```python
# flows/etl.py
from prefect import flow, task

@task(retries=3, retry_delay_seconds=60)
def extract():
    ...

@task(cache_key_fn=task_input_hash)
def transform(data):
    ...

@task
def load(data):
    ...

@flow(name="daily-etl", log_prints=True)
def daily_etl():
    data = extract()
    transformed = transform(data)
    load(transformed)

if __name__ == "__main__":
    daily_etl()
```

```yaml
# prefect.yaml
deployments:
  - name: daily-etl
    entrypoint: flows/etl.py:daily_etl
    work_pool:
      name: production
    schedules:
      - cron: "0 6 * * *"
        timezone: "Europe/London"
```

## Summary

You've learned the core Prefect concepts:

- [x] **Tasks** - Units of work with retries, caching, and logging
- [x] **Flows** - Containers that orchestrate tasks
- [x] **Deployments** - Package flows for remote execution
- [x] **Work pools** - Define infrastructure for execution
- [x] **Workers** - Execute flows from work pools
- [x] **Blocks** - Reusable configuration for external systems
- [x] **Schedules** - Cron-based timing for deployments

## What's Next

Now that you understand the concepts, it's time to choose a deployment option. The next page compares Prefect Cloud, Docker Compose, and ECS + RDS to help you decide.

Continue to [Choosing a Deployment](2-choosing-deployment.md) →
