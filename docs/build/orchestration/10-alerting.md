# Alerting and Notifications

On this page, you will:

- [x] Set up Slack webhook notifications
- [x] Configure PagerDuty integration (optional)
- [x] Create Prefect automations for failure alerts
- [x] Understand alerting best practices

## Overview

Alerting ensures you know when pipelines fail so you can respond quickly. Prefect provides two mechanisms for notifications:

1. **Flow-level hooks**: Code-based notifications attached to specific flows
2. **Automations**: Platform-level rules that trigger actions based on events

For production systems, automations are recommended because they're centralised and don't require code changes.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ALERTING ARCHITECTURE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                      Prefect Cloud                                  │    │
│  │                                                                     │    │
│  │  Flow Run Failed ──────▶ Automation Trigger ──────▶ Actions         │    │
│  │                                                                     │    │
│  │                                              ┌─────────────────┐    │    │
│  │                                              │  Slack Webhook  │    │    │
│  │                                              └────────┬────────┘    │    │
│  │                                                       │             │    │
│  │                                              ┌────────▼────────┐    │    │
│  │                                              │   PagerDuty     │    │    │
│  │                                              │   (optional)    │    │    │
│  │                                              └─────────────────┘    │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Prerequisites

- [x] [Prefect Setup](index.md) - Prefect Cloud or self-hosted configured
- [x] Slack workspace with permission to add apps
- [x] (Optional) PagerDuty account

## Slack Webhook Setup

### Create a Slack App

1. Go to [api.slack.com/apps](https://api.slack.com/apps)
2. Click **Create New App** → **From scratch**
3. Name it (e.g., "Prefect Alerts") and select your workspace
4. Click **Create App**

### Enable Incoming Webhooks

1. In the left sidebar, click **Incoming Webhooks**
2. Toggle **Activate Incoming Webhooks** to On
3. Click **Add New Webhook to Workspace**
4. Select the channel for alerts (e.g., `#data-alerts`)
5. Click **Allow**

Copy the webhook URL - you'll need it for Prefect. Make sure to store it in your password manager.

### Create Slack Block in Prefect

Create a Prefect Block to store the webhook URL:

=== "Terraform"

    Add to `terraform/prefect/blocks.tf`:

    ```hcl
    # =============================================================================
    # Notification Blocks
    # =============================================================================

    resource "prefect_block" "slack_alerts" {
      name      = "alerts"
      type_slug = "slack-webhook"

      data = jsonencode({
        url = var.slack_webhook_url
      })
    }
    ```

    Add to `terraform/prefect/variables.tf`:

    ```hcl
    variable "slack_webhook_url" {
      description = "Slack webhook URL for pipeline failure alerts"
      type        = string
      sensitive   = true
    }
    ```

    **Store the webhook URL in AWS Secrets Manager:**

    ```sh
    aws secretsmanager create-secret \
        --name "prefect/slack-webhook-url" \
        --description "Slack webhook URL for Prefect pipeline alerts" \
        --secret-string "https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX" \
        --profile admin
    ```

    !!! tip "Avoid Shell History"
        To keep the URL out of your shell history, pipe it from 1Password:

        ```sh
        aws secretsmanager create-secret \
            --name "prefect/slack-webhook-url" \
            --description "Slack webhook URL for Prefect pipeline alerts" \
            --secret-string "$(op item get 'Slack Webhook - Prefect Alerts' --fields credential)" \
            --profile admin
        ```

    **Update your GitHub Actions workflows** to retrieve the secret. In `.github/workflows/terraform_ci.yml` and `.github/workflows/terraform_apply.yml`, add to the `secret-ids` in the Prefect plan/apply jobs:

    ```yaml
    - name: Get secrets from AWS Secrets Manager
      uses: aws-actions/aws-secretsmanager-get-secrets@v2
      with:
        secret-ids: |
          TF_VAR_SLACK_WEBHOOK_URL, prefect/slack-webhook-url
        parse-json-secrets: false
    ```

    This sets `TF_VAR_SLACK_WEBHOOK_URL` as an environment variable, which Terraform automatically uses for the `slack_webhook_url` variable.

=== "CLI"

    ```sh
    prefect block register -m prefect.blocks.notifications

    prefect block create slack-webhook/alerts \
        --url "https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX"
    ```

=== "Python"

    ```python
    from prefect.blocks.notifications import SlackWebhook

    slack = SlackWebhook(url="https://hooks.slack.com/services/...")
    slack.save("alerts")
    ```

### Test the Webhook

```python
from prefect.blocks.notifications import SlackWebhook

slack = SlackWebhook.load("alerts")
slack.notify("Test notification from Prefect!")
```

You should see the message in your Slack channel.

## Prefect Automations

Automations are event-driven rules configured in Prefect Cloud. They're more flexible than flow hooks because:

- They work across all flows without code changes
- They can be managed centrally by platform admins
- They support complex conditions and multiple actions

### Create a Failure Automation

1. In Prefect Cloud, navigate to **Automations**
2. Click **Create Automation**
3. Configure the trigger:

   | Field | Value |
   |-------|-------|
   | Trigger type | Flow run state change |
   | Flow run state | Failed, Crashed |
   | Tags | `dlt` (to match all dlt flows) |

4. Configure the action:

   | Field | Value |
   |-------|-------|
   | Action type | Send notification |
   | Block | slack-webhook/alerts |
   | Message | See template below |

### Message Template

Use this template for informative Slack messages:

```
:x: *Pipeline Failed*

*Flow*: {{ flow.name }}
*Deployment*: {{ deployment.name }}
*Run ID*: {{ flow_run.id }}
*State*: {{ flow_run.state.name }}
*Error*: {{ flow_run.state.message }}

<{{ flow_run_url }}|View in Prefect Cloud>
```

### Terraform Automation

Add to `terraform/prefect/automations.tf`:

```hcl
resource "prefect_automation" "dlt_failure_alert" {
  name        = "DLT Pipeline Failure Alert"
  description = "Alert on any dlt pipeline failure via Slack"
  enabled     = true

  trigger = {
    type    = "event"
    posture = "Reactive"

    expect = ["prefect.flow-run.Failed", "prefect.flow-run.Crashed"]

    match_related = {
      "prefect.resource.role"    = "flow-run"
      "prefect.tag"              = "dlt"
    }
  }

  actions = [
    {
      type     = "send-notification"
      block_id = prefect_block.slack_alerts.id
      subject  = "Pipeline Failed: {{ flow.name }}"
      body     = <<-EOT
        :x: *Pipeline Failed*

        *Flow*: {{ flow.name }}
        *State*: {{ flow_run.state.name }}
        *Error*: {{ flow_run.state.message }}
      EOT
    }
  ]
}
```

## PagerDuty Integration (Optional)

PagerDuty provides advanced incident management for production systems. Use it when you need:

- On-call rotations with schedule management
- Escalation policies (alert team lead if engineer doesn't respond)
- Phone/SMS alerts for critical failures
- Incident tracking and post-mortems

### PagerDuty Setup

1. Create a PagerDuty account at [pagerduty.com](https://www.pagerduty.com/)
2. Create a **Service** for data pipelines:
   - Navigate to **Services** → **Service Directory** → **New Service**
   - Name: "Data Pipelines"
   - Integration: Select **Events API V2**
3. Copy the **Integration Key**

### Create PagerDuty Block

=== "Terraform"

    Add to `terraform/prefect/blocks.tf`:

    ```hcl
    resource "prefect_block" "pagerduty_data_pipelines" {
      name      = "data-pipelines"
      type_slug = "pager-duty-webhook"

      data = jsonencode({
        integration_key = var.pagerduty_integration_key
        api_key         = var.pagerduty_api_key
      })
    }
    ```

    Add to `terraform/prefect/variables.tf`:

    ```hcl
    variable "pagerduty_integration_key" {
      description = "PagerDuty Events API v2 integration key for data pipeline alerts"
      type        = string
      sensitive   = true
      default     = ""
    }

    variable "pagerduty_api_key" {
      description = "PagerDuty API key for incident management"
      type        = string
      sensitive   = true
      default     = ""
    }
    ```

    **Store the credentials in AWS Secrets Manager:**

    ```sh
    aws secretsmanager create-secret \
        --name "prefect/pagerduty-integration-key" \
        --description "PagerDuty integration key for Prefect pipeline alerts" \
        --secret-string "YOUR_INTEGRATION_KEY" \
        --profile admin
    ```

    **Update your GitHub Actions workflows** to retrieve the secret. In `.github/workflows/terraform_ci.yml` and `.github/workflows/terraform_apply.yml`, add to the `secret-ids` in the Prefect plan/apply jobs:

    ```yaml
    - name: Get secrets from AWS Secrets Manager
      uses: aws-actions/aws-secretsmanager-get-secrets@v2
      with:
        secret-ids: |
          TF_VAR_PAGERDUTY_INTEGRATION_KEY, prefect/pagerduty-integration-key
        parse-json-secrets: false
    ```

=== "CLI"

    ```sh
    prefect block register -m prefect.blocks.notifications

    prefect block create pager-duty-webhook/data-pipelines \
        --integration-key "your-integration-key"
    ```

=== "Python"

    ```python
    from prefect.blocks.notifications import PagerDutyWebHook

    pagerduty = PagerDutyWebHook(
        integration_key="your-integration-key",
        api_key="your-api-key",  # Optional, for API access
    )
    pagerduty.save("data-pipelines")
    ```

### Add PagerDuty Automation (Terraform)

For critical failures, add an automation that pages via PagerDuty. Add to `terraform/prefect/automations.tf`:

```hcl
resource "prefect_automation" "dlt_critical_alert" {
  name        = "DLT Pipeline Critical Alert (PagerDuty)"
  description = "Page on-call engineer after 3 consecutive dlt pipeline failures"
  enabled     = true

  trigger = {
    type    = "event"
    posture = "Reactive"

    expect = ["prefect.flow-run.Failed", "prefect.flow-run.Crashed"]

    match_related = {
      "prefect.resource.role" = "flow-run"
      "prefect.tag"           = "dlt"
    }

    # Only trigger after 3 consecutive failures
    threshold = 3
    within    = 86400  # 24 hours
  }

  actions = [
    {
      type     = "send-notification"
      block_id = prefect_block.pagerduty_data_pipelines.id
      subject  = "CRITICAL: {{ flow.name }} failed 3 times"
      body     = "Flow {{ flow.name }} has failed 3 times in 24 hours. State: {{ flow_run.state.message }}"
    }
  ]
}
```

### Tiered Alerting

Configure different actions based on severity:

| Severity | Condition | Action |
|----------|-----------|--------|
| Warning | Single failure | Slack notification |
| Critical | 3+ consecutive failures | Slack + PagerDuty |
| P1 | All pipelines down | PagerDuty with immediate escalation |

Create multiple automations with different triggers:

**Warning Automation**:
- Trigger: Any flow failure
- Action: Slack notification

**Critical Automation**:
- Trigger: 3 consecutive failures of same flow
- Action: Slack + PagerDuty

### Cost Considerations

| PagerDuty Tier | Monthly Cost | Features |
|----------------|-------------|----------|
| Free | $0 (≤5 users) | Basic alerting, email/push |
| Professional | $21/user | Phone/SMS, schedules, escalations |
| Business | $41/user | Multiple teams, analytics |

For most small data teams, the free tier or Slack-only is sufficient.

## Flow-Level Notifications

For specific flows that need custom notification logic, use flow hooks:

```python
from prefect import flow, get_run_logger
from prefect.blocks.notifications import SlackWebhook


def notify_on_failure(flow, flow_run, state):
    """Send detailed failure notification."""
    logger = get_run_logger()
    logger.error(f"Flow {flow.name} failed: {state.message}")

    slack = SlackWebhook.load("alerts")
    slack.notify(
        f":x: *{flow.name}* failed!\n"
        f"Error: {state.message}\n"
        f"Run ID: `{flow_run.id}`"
    )


def notify_on_completion(flow, flow_run, state):
    """Send completion notification (optional)."""
    # Only for critical flows
    if flow.name in ["products-daily", "exchange-rates-daily"]:
        slack = SlackWebhook.load("alerts")
        slack.notify(f":white_check_mark: *{flow.name}* completed successfully")


@flow(
    name="exchange-rates-daily",
    on_failure=[notify_on_failure],
    on_crashed=[notify_on_failure],
    # on_completion=[notify_on_completion],  # Optional
)
def exchange_rates_daily_flow():
    # ... flow logic
    pass
```

## Best Practices

### 1. Don't Alert on Everything

Alert fatigue reduces effectiveness. Only alert on:

- **Failures**: Pipeline failures that need attention
- **SLA breaches**: Data not refreshed by expected time
- **Anomalies**: Unusual patterns (e.g., 0 rows loaded)

Don't alert on:

- Successful runs (use dashboards instead)
- Retries that succeed
- Expected maintenance windows

### 2. Include Actionable Information

Every alert should help the responder understand:

- **What** failed
- **When** it failed
- **Where** to investigate (link to logs)
- **How** to start troubleshooting

### 3. Set Up Escalation

For critical pipelines, configure escalation:

1. **Immediate**: Slack notification
2. **After 15 mins**: Page on-call engineer
3. **After 30 mins**: Escalate to team lead
4. **After 1 hour**: Escalate to manager

### 4. Separate Channels by Severity

| Channel | Purpose |
|---------|---------|
| `#data-alerts` | All pipeline notifications |
| `#data-alerts-critical` | Failures requiring immediate action |
| `#data-info` | Success notifications, metrics (optional) |

### 5. Test Your Alerts

Regularly test that alerts work:

```python
@flow(name="test-alert")
def test_alert_flow():
    """Flow that intentionally fails to test alerting."""
    raise Exception("This is a test failure - ignore")
```

Run this periodically to verify Slack/PagerDuty integration.

## Monitoring Dashboard

Beyond alerts, consider a monitoring dashboard showing:

- Recent flow runs and their states
- Success rate over time
- Average run duration
- Data freshness (when was data last loaded)

Prefect Cloud provides these in the UI. For custom dashboards, use the Prefect API:

```python
from prefect.client import get_client

async def get_recent_failures():
    async with get_client() as client:
        runs = await client.read_flow_runs(
            flow_run_filter={
                "state": {"type": {"any_": ["FAILED", "CRASHED"]}},
                "start_time": {"after_": "2026-01-01T00:00:00Z"},
            },
            limit=10,
        )
        return runs
```

## Summary

You've configured alerting for your data pipelines:

- [x] Created Slack webhook block for notifications
- [x] Set up Prefect automations for failure alerts
- [x] Understood PagerDuty integration options
- [x] Learned alerting best practices

## What's Next

With alerting configured, your orchestration layer is complete. Continue to build data pipelines in the [Batch Data Ingestion](../batch-data-ingestion/index.md) section.
