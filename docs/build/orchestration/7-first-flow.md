# Your First Flow

On this page, you will:

- [x] Set up a repository for your data pipelines
- [x] Initialise a Prefect project
- [x] Create and test a flow locally
- [x] Deploy and run your flow

## Overview

Now that you have Prefect set up with work pools and workers, let's create a repository for your flows and deploy your first pipeline.

## Step 1: Create a Repository

Your flows need a Git repository so Prefect can pull and execute them.

### Create the Repository on GitHub

1. Go to [github.com/new](https://github.com/new)
2. Name your repository (e.g., `data-pipelines`)
3. Set visibility to **Private** (recommended for data pipelines)
4. Select **Add a README file**
5. Click **Create repository**

### Clone and Set Up Locally

```sh
# Clone your new repository
cd ~/projects/data
git clone git@github.com:YOUR-ORG/data-pipelines.git
cd data-pipelines

# Initialise project and install dependencies
uv init
uv add prefect prefect-aws prefect-snowflake
```

### Create Project Structure

```sh
# Create directories
mkdir -p flows

# Create __init__.py files
touch flows/__init__.py
```

Your project structure should look like:

```
data-pipelines/
├── flows/
│   └── __init__.py
├── .venv/
├── .gitignore
└── README.md
```

### Add .gitignore

Create `.gitignore`:

```
# Python
.venv/
__pycache__/
*.pyc
.python-version

# Environment
.env
.envrc

# IDE
.vscode/
.idea/

# Prefect
.prefectignore
```

## Step 2: Initialise Prefect Project

Prefect uses `prefect.yaml` to define how flows are deployed. You can create this manually or use `prefect init`.

### Using prefect init

```sh
# Initialise a Prefect project
prefect init
```

When prompted:

1. Select a recipe (choose **git** for GitHub-hosted code)
2. Answer the prompts about your project

This creates a `prefect.yaml` template.

### Or Create Manually

Create `prefect.yaml` in your project root:

```yaml
# Prefect deployment configuration
name: data-pipelines

# Pull section - how to retrieve flow code
pull:
  - prefect.deployments.steps.git_clone:
      repository: https://github.com/YOUR-ORG/data-pipelines.git
      branch: main

# Deployments defined below
deployments: []
```

## Step 3: Create Your First Flow

Create `flows/hello_world.py`:

```python
"""
Simple hello world flow to verify Prefect is working.
"""
from prefect import flow, task, get_run_logger


@task
def say_hello(name: str) -> str:
    """A simple task that says hello."""
    logger = get_run_logger()
    message = f"Hello, {name}!"
    logger.info(message)
    return message


@task
def count_letters(text: str) -> int:
    """Count the letters in a string."""
    logger = get_run_logger()
    count = len(text.replace(" ", ""))
    logger.info(f"'{text}' has {count} letters")
    return count


@flow(name="hello-world", log_prints=True)
def hello_world_flow(name: str = "World") -> dict:
    """
    A simple flow that says hello and counts letters.

    Args:
        name: The name to greet

    Returns:
        dict with greeting and letter count
    """
    greeting = say_hello(name)
    letter_count = count_letters(greeting)

    print(f"Completed! Greeting: {greeting}, Letters: {letter_count}")

    return {
        "greeting": greeting,
        "letter_count": letter_count
    }


# For local testing
if __name__ == "__main__":
    result = hello_world_flow(name="Prefect")
    print(f"Result: {result}")
```

### Test Locally

Run the flow locally to verify it works:

```sh
uv run python flows/hello_world.py
```

Expected output:

```
14:32:01.123 | INFO    | prefect.engine - Created flow run 'copper-fox' for flow 'hello-world'
14:32:01.234 | INFO    | Task run 'say_hello-0' - Hello, Prefect!
14:32:01.345 | INFO    | Task run 'count_letters-0' - 'Hello, Prefect!' has 12 letters
14:32:01.456 | INFO    | Flow run 'copper-fox' - Completed! Greeting: Hello, Prefect!, Letters: 12
14:32:01.567 | INFO    | Flow run 'copper-fox' - Finished in state Completed()
Result: {'greeting': 'Hello, Prefect!', 'letter_count': 12}
```

## Step 4: Create a Deployment

Add your flow to `prefect.yaml`:

```yaml
# Prefect deployment configuration
name: data-pipelines

# Pull section - how to retrieve flow code
pull:
  - prefect.deployments.steps.git_clone:
      repository: https://github.com/YOUR-ORG/data-pipelines.git
      branch: main

# Deployments
deployments:
  - name: hello-world
    entrypoint: flows/hello_world.py:hello_world_flow
    description: "Simple test flow"
    work_pool:
      name: development
    parameters:
      name: "World"
    tags:
      - test
      - example
```

## Step 5: Commit and Push

Prefect needs access to your code via Git. Commit and push your changes:

```sh
git add .
git commit -m "Add hello world flow"
git push
```

## Step 6: Deploy the Flow

Deploy your flow to Prefect Cloud:

```sh
# Deploy all flows defined in prefect.yaml
prefect deploy --all

# Or deploy a specific deployment
prefect deploy --name hello-world
```

Expected output:

```
Successfully created/updated all deployments!

┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━┓
┃ Deployment                 ┃ Status                ┃
┡━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━┩
│ hello-world/hello-world    │ Created               │
└────────────────────────────┴───────────────────────┘
```

## Step 7: Run the Flow

### Ensure Worker is Running

In a separate terminal, start a worker:

```sh
cd ~/projects/data/data-pipelines
prefect worker start --pool development
```

### Trigger a Run

From CLI:

```sh
# Run with default parameters
prefect deployment run hello-world/hello-world

# Run with custom parameters
prefect deployment run hello-world/hello-world --param name=Prefect
```

From the Prefect UI:

1. Open [app.prefect.cloud](https://app.prefect.cloud)
2. Navigate to **Deployments**
3. Click on your deployment
4. Click **Run** → **Quick Run** or **Custom Run**

## Step 8: View Results

### In the UI

The Prefect UI provides:

- **Flow Runs** - List of all runs with status
- **Logs** - Real-time and historical logs
- **Timeline** - Visual task execution timeline
- **Parameters** - Input parameters for each run

### Via CLI

```sh
# List recent flow runs
prefect flow-run ls

# Get details of a specific run
prefect flow-run inspect <flow-run-id>

# View logs
prefect flow-run logs <flow-run-id>
```

## Commit the Project

Commit your project files including the lock file for reproducible installs:

```sh
git add pyproject.toml uv.lock flows/ prefect.yaml .gitignore
git commit -m "Add hello world flow and project setup"
git push
```

## Adding More Deployments

Extend `prefect.yaml` with additional deployments:

```yaml
deployments:
  # Hello World - for testing
  - name: hello-world
    entrypoint: flows/hello_world.py:hello_world_flow
    description: "Simple test flow"
    work_pool:
      name: development
    parameters:
      name: "World"
    tags:
      - test

  # ETL Example - scheduled
  - name: etl-daily
    entrypoint: flows/etl_example.py:etl_flow
    description: "Daily ETL pipeline"
    work_pool:
      name: production
    schedules:
      - cron: "0 6 * * *"
        timezone: "Europe/London"
        active: true
    tags:
      - etl
      - production
```

## Flow Run States

Understanding flow run states:

| State | Meaning |
|-------|---------|
| `Scheduled` | Waiting for scheduled time |
| `Pending` | Waiting for a worker |
| `Running` | Currently executing |
| `Completed` | Finished successfully |
| `Failed` | Finished with error |
| `Cancelled` | Manually cancelled |

## Summary

You've deployed and run your first Prefect flow:

- [x] Created a repository for data pipelines
- [x] Initialised a Prefect project
- [x] Created and tested a flow locally
- [x] Deployed and ran the flow via Prefect Cloud

## What's Next

Now let's integrate your flows with AWS Secrets Manager and Snowflake using Prefect Blocks.

Continue to [Secrets and Blocks](8-secrets-and-blocks.md) →
