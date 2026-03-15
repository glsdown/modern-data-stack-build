# Prefect Monitoring

On this page, you will:

- [x] Understand Prefect flow and task states
- [x] Configure Slack notifications for pipeline failures
- [x] Set up automation rules for retries and alerts
- [x] Create custom metrics for pipeline observability
- [x] Monitor pipeline performance and identify bottlenecks

## Overview

Prefect provides built-in observability for data pipelines. Every flow run, task execution, and state transition is tracked, logged, and available for monitoring.

Prefect monitoring answers:
- **"Did my pipeline run successfully?"** (flow states)
- **"Why did this task fail?"** (logs and error tracking)
- **"How long does this pipeline take?"** (duration metrics)
- **"Is this pipeline getting slower?"** (performance trends)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     PREFECT MONITORING STACK                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Flow Execution         Monitoring             Alerts                  │
│  ───────────────        ──────────             ──────                  │
│                                                                         │
│  ┌──────────────┐      ┌──────────────┐       ┌──────────────┐        │
│  │ dbt daily    │─────▶│ Prefect      │──────▶│ Slack        │        │
│  │ pipeline     │      │ Cloud UI     │       │ #data-alerts │        │
│  │              │      │ • States     │       └──────────────┘        │
│  │ Tasks:       │      │ • Logs       │              │                │
│  │ • Extract    │      │ • Duration   │       ┌──────▼───────┐        │
│  │ • Transform  │      │ • Retries    │       │ PagerDuty    │        │
│  │ • Load       │      └──────────────┘       │ (Critical)   │        │
│  └──────────────┘              │              └──────────────┘        │
│         │                      │                                       │
│         │                      ▼                                       │
│         │              ┌──────────────┐                                │
│         └─────────────▶│ Automation   │                                │
│                        │ Rules        │                                │
│                        │ • Retry 3x   │                                │
│                        │ • Alert if   │                                │
│                        │   failed     │                                │
│                        └──────────────┘                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Flow and Task States

Prefect tracks the state of every flow run and task run.

### Flow States

| State | Description | When It Occurs |
|-------|-------------|----------------|
| **Scheduled** | Flow run is scheduled but not started | After deployment with schedule |
| **Pending** | Flow run is waiting for available resources | Queue depth or worker availability |
| **Running** | Flow is currently executing | Tasks are running |
| **Completed** | Flow finished successfully | All tasks completed |
| **Failed** | Flow failed | Any task failed without retry recovery |
| **Crashed** | Flow crashed unexpectedly | Worker died, infrastructure issue |
| **Cancelled** | Flow was manually cancelled | User cancellation via UI/API |

### Task States

| State | Description | Action Required |
|-------|-------------|-----------------|
| **Pending** | Task waiting to start | Normal — will start soon |
| **Running** | Task is executing | Normal — monitor progress |
| **Completed** | Task finished successfully | None — success |
| **Failed** | Task failed | **Investigate** — check logs |
| **Retrying** | Task failed, retrying | Monitor — may resolve automatically |
| **Cached** | Task result cached, skipped | Normal — efficiency feature |
| **Cancelled** | Task cancelled | Verify if intentional |

### State Transitions

```
Scheduled → Pending → Running → Completed ✓
                          │
                          ├─▶ Failed → Retrying → Running → Completed ✓
                          │                    └─▶ Failed ✗
                          │
                          └─▶ Crashed ✗ (infrastructure issue)
```

### View States in Prefect Cloud

1. Log into [Prefect Cloud](https://app.prefect.cloud)
2. Navigate to **Flow Runs**
3. Filter by state:
   - ✓ Green: Completed
   - ✗ Red: Failed
   - ⟳ Yellow: Retrying
   - ⏸ Grey: Scheduled/Pending

Click on any flow run to see:
- **Timeline** — visual representation of task execution
- **Logs** — stdout/stderr from each task
- **State history** — transitions over time
- **Parameters** — inputs passed to the flow

## Configure Slack Notifications

Send alerts to Slack when pipelines fail.

### Step 1: Create Slack Webhook

1. Navigate to [Slack API](https://api.slack.com/apps)
2. Click **Create New App** → **From scratch**
3. App name: **Prefect Alerts**
4. Workspace: Your workspace
5. Navigate to **Incoming Webhooks** → Enable
6. Click **Add New Webhook to Workspace**
7. Select channel: `#data-alerts`
8. Copy webhook URL: `https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXX`

### Step 2: Store Webhook in Prefect

```sh
# Store as Prefect Secret
prefect block register -m prefect_slack

prefect block create webhook \
    --name "slack-data-alerts" \
    --url "https://hooks.slack.com/services/..."
```

Or create via Python:

```python
from prefect_slack import SlackWebhook

slack_webhook = SlackWebhook(
    url="https://hooks.slack.com/services/..."
)
slack_webhook.save("slack-data-alerts")
```

### Step 3: Send Notifications from Flows

```python
from prefect import flow, task
from prefect_slack import SlackWebhook
from datetime import datetime

@flow(name="dbt-daily-pipeline")
def dbt_pipeline():
    slack_webhook = SlackWebhook.load("slack-data-alerts")

    try:
        # Run your pipeline tasks
        run_dbt_build()
        run_dbt_tests()

        # Success notification
        slack_webhook.notify(
            text=f"✅ dbt pipeline completed successfully at {datetime.now()}"
        )

    except Exception as e:
        # Failure notification
        slack_webhook.notify(
            text=f"❌ dbt pipeline failed: {str(e)}\nCheck logs: https://app.prefect.cloud"
        )
        raise  # Re-raise to mark flow as failed
```

### Step 4: Customise Notification Format

Send richer notifications with Slack blocks:

```python
slack_webhook.notify(
    blocks=[
        {
            "type": "header",
            "text": {
                "type": "plain_text",
                "text": "❌ Pipeline Failure Alert"
            }
        },
        {
            "type": "section",
            "text": {
                "type": "mrkdwn",
                "text": f"*Flow:* dbt-daily-pipeline\n*Status:* Failed\n*Time:* {datetime.now()}"
            }
        },
        {
            "type": "section",
            "text": {
                "type": "mrkdwn",
                "text": f"*Error:* {str(error)}"
            }
        },
        {
            "type": "actions",
            "elements": [
                {
                    "type": "button",
                    "text": {
                        "type": "plain_text",
                        "text": "View Logs"
                    },
                    "url": "https://app.prefect.cloud/flow-runs/..."
                }
            ]
        }
    ]
)
```

## Automation Rules

Prefect automations allow you to define rules that trigger actions based on flow states.

### Create Automation via UI

1. In Prefect Cloud, navigate to **Automations**
2. Click **Create Automation**
3. Define trigger and action

### Example Automations

#### 1. Retry Failed Flows

**Trigger:** Flow run enters `Failed` state
**Action:** Retry the flow run (max 3 times)

**Configuration:**

```yaml
trigger:
  match:
    prefect.resource.name: "dbt-daily-pipeline"
  expect:
    - "prefect.flow-run.Failed"

actions:
  - type: retry-flow-run
    max_retries: 3
    retry_delay: 300  # 5 minutes
```

Create via Python API:

```python
from prefect import get_client
from prefect.events import Automation, EventTrigger, RetryFlowRun

async def create_retry_automation():
    async with get_client() as client:
        automation = Automation(
            name="Retry failed dbt pipeline",
            trigger=EventTrigger(
                match={"prefect.resource.name": "dbt-daily-pipeline"},
                expect=["prefect.flow-run.Failed"]
            ),
            actions=[
                RetryFlowRun(
                    max_retries=3,
                    retry_delay_seconds=300
                )
            ]
        )
        await client.create_automation(automation)
```

#### 2. Alert on Repeated Failures

**Trigger:** Flow run fails 3 times in a row
**Action:** Send Slack alert + page on-call engineer

**Configuration:**

```python
automation = Automation(
    name="Alert on repeated dbt failures",
    trigger=EventTrigger(
        match={"prefect.resource.name": "dbt-daily-pipeline"},
        expect=["prefect.flow-run.Failed"],
        threshold=3,  # Trigger after 3 failures
        within=3600   # Within 1 hour
    ),
    actions=[
        SendSlackNotification(
            webhook="slack-data-alerts",
            message="🚨 dbt pipeline has failed 3 times in the last hour. Immediate attention required."
        ),
        SendPagerDutyAlert(
            severity="high"
        )
    ]
)
```

#### 3. Scale Resources on Long-Running Flows

**Trigger:** Flow run duration > 30 minutes
**Action:** Scale up worker pool

**Use case:** Automatically allocate more resources if pipeline is slow.

```python
automation = Automation(
    name="Scale up for slow pipelines",
    trigger=EventTrigger(
        match={"prefect.resource.name": "dbt-daily-pipeline"},
        expect=["prefect.flow-run.Running"],
        posture="Proactive",
        threshold_duration=1800  # 30 minutes
    ),
    actions=[
        ScaleWorkPool(
            work_pool="data-pipelines",
            target_concurrency=10
        )
    ]
)
```

#### 4. Pause Downstream Flows on Failure

**Trigger:** Upstream flow fails
**Action:** Pause downstream flows

**Use case:** If dbt fails, pause Lightdash dashboard refresh (no point refreshing if data is stale).

```python
automation = Automation(
    name="Pause downstream on dbt failure",
    trigger=EventTrigger(
        match={"prefect.resource.name": "dbt-daily-pipeline"},
        expect=["prefect.flow-run.Failed"]
    ),
    actions=[
        PauseDeployment(
            deployment_name="lightdash-refresh"
        ),
        SendSlackNotification(
            message="⏸ Lightdash refresh paused due to dbt failure. Fix dbt, then manually resume."
        )
    ]
)
```

## Custom Metrics

Track custom metrics to monitor pipeline performance beyond default Prefect metrics.

### Built-in Metrics

Prefect Cloud automatically tracks:
- **Flow run duration** — total time from start to completion
- **Task run duration** — time per task
- **Success rate** — percentage of successful runs
- **Concurrency** — number of concurrent flow runs

View in **Prefect Cloud** → **Insights** → **Metrics**

### Custom Metrics with Logging

Log custom metrics within tasks:

```python
from prefect import task, get_run_logger

@task
def extract_data():
    logger = get_run_logger()

    # Your extraction logic
    row_count = fetch_data_from_api()

    # Log custom metric
    logger.info(f"Extracted {row_count} rows from API")
    logger.info(f"METRIC: rows_extracted={row_count}")

    return row_count
```

These logs are searchable in Prefect Cloud:
1. Navigate to **Flow Runs** → Select run → **Logs**
2. Filter logs by `METRIC:` prefix
3. Export to external monitoring (Datadog, CloudWatch)

### Custom Metrics with Prefect Blocks

Store metrics in Prefect KV store for trend analysis:

```python
from prefect.blocks.system import JSON
from datetime import datetime

@task
def record_pipeline_metrics(row_count: int, duration: float):
    # Load existing metrics
    try:
        metrics_block = JSON.load("pipeline-metrics")
        metrics = metrics_block.value
    except:
        metrics = []

    # Append new metric
    metrics.append({
        "timestamp": datetime.now().isoformat(),
        "row_count": row_count,
        "duration_seconds": duration
    })

    # Save updated metrics
    metrics_block = JSON(value=metrics)
    metrics_block.save("pipeline-metrics", overwrite=True)
```

Query metrics later:

```python
metrics = JSON.load("pipeline-metrics").value
recent_metrics = [m for m in metrics if m['timestamp'] > '2026-02-01']
avg_duration = sum(m['duration_seconds'] for m in recent_metrics) / len(recent_metrics)
print(f"Average duration: {avg_duration:.2f} seconds")
```

### Send Metrics to CloudWatch

For enterprise monitoring, send metrics to AWS CloudWatch:

```python
import boto3
from prefect import task

@task
def send_cloudwatch_metric(metric_name: str, value: float):
    cloudwatch = boto3.client('cloudwatch', region_name='eu-west-2')

    cloudwatch.put_metric_data(
        Namespace='DataPipelines',
        MetricData=[
            {
                'MetricName': metric_name,
                'Value': value,
                'Unit': 'Count',
                'Timestamp': datetime.now()
            }
        ]
    )
```

Usage in flow:

```python
@flow
def dbt_pipeline():
    start_time = datetime.now()

    # Run dbt
    row_count = run_dbt_build()

    # Calculate duration
    duration = (datetime.now() - start_time).total_seconds()

    # Send metrics
    send_cloudwatch_metric("dbt_row_count", row_count)
    send_cloudwatch_metric("dbt_duration_seconds", duration)
```

View metrics in AWS CloudWatch:
- Navigate to **CloudWatch** → **Metrics** → **DataPipelines**
- Create dashboards for trend visualisation
- Set CloudWatch alarms for abnormal metrics

## Monitor Pipeline Performance

### Identify Bottlenecks

1. Navigate to **Flow Runs** → Select a completed run
2. View **Timeline** — visualises task execution
3. Identify longest-running tasks:
   - Hover over tasks to see duration
   - Look for tasks that take >50% of total time

**Example:**

```
Timeline:
[extract_data]────────────────────────── 5m (bottleneck!)
    [transform_data]─────── 30s
        [load_data]──── 15s
```

**Action:** Optimise `extract_data` task (parallel fetching, batch size tuning).

### Track Duration Trends

1. Navigate to **Insights** → **Flow Run Duration**
2. View duration trends over time
3. Identify increasing trends:
   - Is the pipeline getting slower?
   - Are specific days slower (e.g., month-end spikes)?

**Example chart:**

```
Duration (minutes)
60 │                                               ╱─
50 │                                           ╱───
40 │                                       ╱───
30 │                               ╱───────
20 │───────────────────────────────
   └────────────────────────────────────────────────
   Jan 1          Jan 15          Feb 1    Today
```

**Finding:** Pipeline duration increasing 2x per month → investigate data volume growth.

### Analyse Task Retry Patterns

If tasks frequently retry:

1. Navigate to **Flow Runs** → Filter by **Retrying** state
2. Identify which tasks retry most often
3. Check logs for error patterns

**Common retry causes:**
- API rate limits → Add exponential backoff
- Network timeouts → Increase timeout threshold
- Transient database locks → Add retry with delay

### Set Performance Baselines

Define SLOs (Service Level Objectives) for pipelines:

```python
# Expected performance
DBT_PIPELINE_SLO = {
    "max_duration_minutes": 30,
    "max_row_count": 1_000_000,
    "min_success_rate_pct": 99
}

@flow
def dbt_pipeline():
    start_time = datetime.now()

    # Run pipeline
    row_count = run_dbt_build()

    # Check SLO
    duration = (datetime.now() - start_time).total_seconds() / 60

    if duration > DBT_PIPELINE_SLO["max_duration_minutes"]:
        slack_webhook.notify(
            text=f"⚠️ SLO violation: dbt pipeline took {duration:.1f} minutes (SLO: {DBT_PIPELINE_SLO['max_duration_minutes']} minutes)"
        )
```

## Monitoring Best Practices

### 1. Set Alerts for Critical Failures

**Critical pipelines** (daily dbt run, revenue data sync):
- Alert immediately via Slack + PagerDuty
- Auto-retry 3 times with 5-minute delays
- Escalate to on-call if retries fail

**Non-critical pipelines** (weekly analytics refresh):
- Log failure, no immediate alert
- Review failures weekly
- Alert only if repeated failures

### 2. Monitor Success Rate Over Time

Track success rate as a KPI:

```sql
-- Query Prefect database (if self-hosted) or export logs
SELECT
    DATE_TRUNC('day', start_time) AS run_date,
    COUNT(*) AS total_runs,
    SUM(CASE WHEN state = 'Completed' THEN 1 ELSE 0 END) AS successful_runs,
    (successful_runs * 100.0 / total_runs) AS success_rate_pct
FROM flow_runs
WHERE flow_name = 'dbt-daily-pipeline'
  AND start_time >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY run_date
ORDER BY run_date;
```

**Target SLO:** ≥ 99% success rate over 30 days.

### 3. Review Logs Regularly

Schedule weekly reviews:
- Check failed flows for patterns
- Review retry logs for recurring issues
- Identify slow tasks for optimisation

### 4. Use Tags for Filtering

Tag flows by criticality, team, or data domain:

```python
@flow(
    name="dbt-daily-pipeline",
    tags=["critical", "dbt", "daily", "data-engineering"]
)
def dbt_pipeline():
    ...
```

Filter in Prefect Cloud:
- **Critical flows** → monitor closely
- **Team: data-engineering** → team-specific dashboards
- **Daily** → ensure all daily flows succeeded

### 5. Archive Old Flow Runs

Prefect Cloud Free tier retains 7 days of flow run history. For longer retention:

**Option A:** Upgrade to Prefect Cloud Pro ($450/month) for 30-day retention

**Option B:** Export logs to S3/CloudWatch:

```python
@flow(on_completion=[export_logs_to_s3])
def dbt_pipeline():
    ...

def export_logs_to_s3(flow, flow_run, state):
    """Export flow run logs to S3 for long-term storage"""
    import boto3

    s3 = boto3.client('s3')
    logs = get_flow_run_logs(flow_run.id)

    s3.put_object(
        Bucket='data-pipeline-logs',
        Key=f"flow-runs/{flow_run.id}.json",
        Body=json.dumps(logs)
    )
```

## Summary

You've configured Prefect monitoring:

- [x] **Flow and task states** — understand Scheduled, Running, Completed, Failed, Retrying
- [x] **Slack notifications** — alerts sent to `#data-alerts` on pipeline failures
- [x] **Automation rules** — auto-retry failed flows, alert on repeated failures
- [x] **Custom metrics** — track row counts, durations, and business KPIs
- [x] **Performance monitoring** — identify bottlenecks, track duration trends, set SLOs

Prefect provides comprehensive pipeline observability out of the box. Next, monitor Snowflake query performance and warehouse utilisation.

## What's Next

Monitor Snowflake compute costs and query performance.

Continue to [Snowflake Monitoring](7-snowflake-monitoring.md) →
