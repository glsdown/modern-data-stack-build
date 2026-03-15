# Alerting and Incidents

On this page, you will:

- [x] Design effective alert routing for different severity levels
- [x] Create runbooks for common incident scenarios
- [x] Set up on-call rotations and escalation policies
- [x] Implement incident response workflows
- [x] Learn post-incident review practices

## Overview

Alerts notify you when something goes wrong. Incident response is how you fix it. Together, they ensure data platform reliability.

Effective alerting requires:
1. **Right alerts** — actionable, not noisy
2. **Right people** — route to teams who can fix the issue
3. **Right context** — enough information to debug quickly
4. **Right escalation** — ensure critical issues get attention

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ALERTING AND INCIDENT WORKFLOW                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Issue Detected      Alert Routing         Incident Response           │
│  ──────────────      ──────────────        ─────────────────           │
│                                                                         │
│  ┌──────────────┐   ┌──────────────┐      ┌──────────────┐            │
│  │ dbt test     │──▶│ SEV1         │─────▶│ PagerDuty    │            │
│  │ failed       │   │ (Critical)   │      │ page on-call │            │
│  └──────────────┘   └──────────────┘      └──────────────┘            │
│                             │                      │                   │
│  ┌──────────────┐           │              ┌───────▼──────┐            │
│  │ Anomaly      │──▶┌───────▼───────┐      │ Run runbook  │            │
│  │ detected     │   │ SEV2          │─────▶│ Fix issue    │            │
│  └──────────────┘   │ (High)        │      │ Document     │            │
│                     └───────────────┘      └──────────────┘            │
│  ┌──────────────┐           │                                          │
│  │ Slow query   │──▶┌───────▼───────┐      ┌──────────────┐            │
│  │ detected     │   │ SEV3          │─────▶│ Log for      │            │
│  └──────────────┘   │ (Medium)      │      │ weekly review│            │
│                     └───────────────┘      └──────────────┘            │
│                                                                         │
│  Post-Incident:                                                         │
│  ┌────────────────────────────────────────────────────────────┐        │
│  │ Post-mortem → Root cause → Prevention → Update runbooks    │        │
│  └────────────────────────────────────────────────────────────┘        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Alert Routing Strategy

Different severity levels require different routing and response times.

### Alert Severity Levels (Recap)

| Severity | Description | Response Time | Routing |
|----------|-------------|---------------|---------|
| **SEV1** | Data outage, critical pipeline failure | < 2 hours | PagerDuty + Slack + Email |
| **SEV2** | Quality degraded, non-critical failures | < 4 hours (business hours) | Slack + Email |
| **SEV3** | Minor issues, edge cases | < 1 week | Slack (no ping) or log only |
| **SEV4** | Cosmetic, documentation | No deadline | Backlog ticket |

### Routing Table

| Alert Source | Condition | Severity | Destination |
|--------------|-----------|----------|-------------|
| dbt | Full run failed | SEV1 | PagerDuty + Slack `#data-critical` |
| dbt | ≥3 tests failed | SEV2 | Slack `#data-alerts` |
| dbt | 1-2 tests failed | SEV3 | Slack `#data-alerts` (no ping) |
| Prefect | Flow failed after 3 retries | SEV1 | PagerDuty + Slack `#data-critical` |
| Prefect | Flow retrying | SEV3 | Log only (no alert) |
| Elementary | Volume anomaly >50% drop | SEV2 | Slack `#data-alerts` |
| Elementary | Schema change detected | SEV3 | Slack `#data-alerts` (no ping) |
| Snowflake Resource Monitor | 100% of budget used | SEV1 | PagerDuty + Email |
| Snowflake Resource Monitor | 90% of budget used | SEV2 | Email |
| AWS Budget | 100% of budget used | SEV2 | Email |
| Lightdash | Query timeout | SEV3 | Log only |

## Alert Channels

### Slack Channels

Create separate channels for different severity levels:

**#data-critical** (SEV1 only):
- Mentions `@data-oncall` (on-call engineer)
- Notifications enabled 24/7
- Only critical alerts (pipeline failures, outages)

**#data-alerts** (SEV2-SEV3):
- No `@channel` or `@here` mentions
- Notifications during business hours only
- Quality issues, anomalies, warnings

**#data-logs** (SEV4, informational):
- No notifications
- Audit trail, successful runs, metrics

### PagerDuty

Use PagerDuty for SEV1 incidents that require immediate response.

#### Create PagerDuty Service

1. Log into PagerDuty
2. Navigate to **Services** → **New Service**
3. Name: **Data Platform Critical Alerts**
4. Escalation policy: **Data Engineering On-Call**
5. Integration: **Events API v2**
6. Copy integration key: `R0XXXXXXXXXXXXXXXXXXXXXXXX`

#### Store Integration Key in Prefect

```python
from prefect.blocks.system import Secret

secret = Secret(value="R0XXXXXXXXXXXXXXXXXXXXXXXX")
secret.save("pagerduty-integration-key")
```

#### Send Alert from Prefect

```python
from prefect import flow, task
from prefect.blocks.system import Secret
import requests

@task
def send_pagerduty_alert(summary: str, severity: str = "critical"):
    integration_key = Secret.load("pagerduty-integration-key").get()

    payload = {
        "routing_key": integration_key,
        "event_action": "trigger",
        "payload": {
            "summary": summary,
            "severity": severity,
            "source": "Prefect",
            "custom_details": {
                "flow_run_url": "https://app.prefect.cloud/..."
            }
        }
    }

    response = requests.post(
        "https://events.pagerduty.com/v2/enqueue",
        json=payload
    )
    response.raise_for_status()

@flow
def dbt_pipeline():
    try:
        run_dbt_build()
    except Exception as e:
        # SEV1: Send PagerDuty alert
        send_pagerduty_alert(
            summary=f"🚨 dbt pipeline failed: {str(e)}",
            severity="critical"
        )
        raise
```

### Email

Use email for non-urgent alerts (SEV2-SEV3) or when Slack is unavailable.

#### Configure SMTP in Prefect

```python
from prefect_email import EmailServerCredentials

email_server = EmailServerCredentials(
    username="alerts@yourcompany.com",
    password="your-app-password",  # Use app password, not account password
    smtp_server="smtp.gmail.com",
    smtp_port=587,
    smtp_type="STARTTLS"
)
email_server.save("email-server")
```

#### Send Email Alert

```python
from prefect_email import email_send_message

@task
def send_email_alert(subject: str, body: str):
    email_send_message(
        email_server_credentials=EmailServerCredentials.load("email-server"),
        subject=subject,
        msg=body,
        email_to="data-engineering@yourcompany.com"
    )
```

## Runbooks

Runbooks are step-by-step guides for resolving common incidents. They reduce mean time to resolution (MTTR) by providing tested procedures.

### Runbook Template

```markdown
# Runbook: [Incident Type]

## Symptoms
- [What the user or monitoring system observes]

## Severity
- [SEV1/SEV2/SEV3]

## Diagnosis
1. [Step to confirm the issue]
2. [Step to identify root cause]

## Resolution
1. [Step to fix the issue]
2. [Step to verify fix]

## Prevention
- [How to prevent this in future]

## Escalation
- If unable to resolve within [X hours], escalate to [person/team]
```

### Example Runbook: dbt Daily Run Failed

```markdown
# Runbook: dbt Daily Run Failed

## Symptoms
- Prefect flow "dbt-daily-pipeline" status: Failed
- Slack alert in #data-critical: "🚨 dbt pipeline failed"
- Dashboards showing stale data (timestamp > 12 hours old)

## Severity
- **SEV1** (Critical) — all dashboards stale

## Diagnosis

### Step 1: Check Prefect Logs
1. Navigate to Prefect Cloud: https://app.prefect.cloud
2. Find the failed flow run
3. Click on the flow run → **Logs**
4. Identify which task failed

### Step 2: Identify Error Type

**If error contains "Warehouse suspended":**
- Snowflake Resource Monitor suspended warehouse
- Go to **Resolution: Resource Monitor**

**If error contains "Test failure":**
- dbt tests failed
- Go to **Resolution: Test Failure**

**If error contains "Compilation error":**
- dbt model SQL is invalid
- Go to **Resolution: Compilation Error**

**If error contains "Connection timeout":**
- Network issue or Snowflake outage
- Go to **Resolution: Connection Issue**

## Resolution

### Resolution: Resource Monitor

**Cause:** Snowflake Resource Monitor suspended warehouse due to budget limit.

**Fix:**
1. Log into Snowflake as ACCOUNTADMIN
2. Check Resource Monitor status:
   ```sql
   SHOW RESOURCE MONITORS;
   ```
3. If suspended, temporarily increase quota or resume warehouse:
   ```sql
   ALTER RESOURCE MONITOR monthly_budget SET CREDIT_QUOTA = 600;  -- Increase from 500
   -- OR
   ALTER WAREHOUSE TRANSFORMING RESUME;
   ```
4. Re-run dbt pipeline in Prefect
5. Verify dashboards updated

**Follow-up:** Review why budget was exceeded (see Runbook: Budget Exceeded).

### Resolution: Test Failure

**Cause:** dbt tests failed, indicating data quality issue.

**Fix:**
1. Identify failed tests:
   ```sh
   cd ~/projects/dbt/dbt-transform
   dbt test --select result:fail
   ```
2. Review test output for which model and test failed
3. Query the table in Snowflake to inspect bad data:
   ```sql
   SELECT * FROM analytics.marts.fct_orders
   WHERE order_total < 0;  -- Example: test expects order_total >= 0
   ```
4. **Decision:**
   - If data quality issue is in source: Fix source, re-ingest, re-run dbt
   - If dbt logic is wrong: Fix dbt model, re-run
   - If test is too strict: Relax test threshold, re-run

**Temporary workaround (if urgent):**
- Comment out failing test in `schema.yml`
- Re-run dbt (tests will pass)
- Create ticket to fix root cause later

### Resolution: Compilation Error

**Cause:** Invalid SQL in dbt model.

**Fix:**
1. Check error message for model name and line number
2. Open the model file:
   ```sh
   vim ~/projects/dbt/dbt-transform/models/marts/fct_orders.sql
   ```
3. Fix SQL syntax error
4. Test locally:
   ```sh
   dbt run --select fct_orders
   ```
5. Commit fix and re-deploy

**Common causes:**
- Typo in column name
- Missing comma in SELECT clause
- Referencing a model that doesn't exist

### Resolution: Connection Issue

**Cause:** Network timeout or Snowflake service issue.

**Fix:**
1. Check Snowflake status page: https://status.snowflake.com/
2. If Snowflake outage: Wait for resolution, no action needed
3. If no outage, check network connectivity from Prefect worker:
   ```sh
   # SSH to Prefect worker (if self-hosted)
   ping your-account.snowflakecomputing.com
   ```
4. If connectivity issue, restart Prefect worker or check VPC routing
5. Re-run dbt pipeline

## Prevention

- **Resource Monitor:** Set budget with buffer (e.g., if average usage is 400 credits, set quota to 500)
- **Test Failures:** Add Elementary anomaly detection to catch data quality issues before dbt runs
- **Compilation Errors:** Add pre-commit hook to run `dbt compile` before pushing code
- **Connection Issues:** Use Snowflake private link for reliable connectivity

## Escalation

- If unable to resolve within 2 hours, escalate to:
  - Slack: Mention `@data-lead` in #data-critical
  - Email: data-lead@yourcompany.com
  - Phone: [On-call phone number]
```

### Example Runbook: Dashboard Shows Wrong Data

```markdown
# Runbook: Dashboard Shows Wrong Data

## Symptoms
- User reports: "Revenue dashboard shows $0 for all customers today"
- Lightdash dashboard: Values are unexpectedly low or zero
- No pipeline failure alerts

## Severity
- **SEV2** (High) — data quality issue, not a pipeline outage

## Diagnosis

### Step 1: Check Data Freshness
1. Query the table in Snowflake:
   ```sql
   SELECT MAX(loaded_at) AS last_updated
   FROM analytics.marts.fct_revenue;
   ```
2. If `last_updated` > 12 hours ago: Data is stale (see Runbook: dbt Daily Run Failed)
3. If `last_updated` is recent: Data loaded but values are wrong

### Step 2: Check Row Counts
```sql
SELECT
    DATE(order_date) AS day,
    COUNT(*) AS row_count
FROM analytics.marts.fct_revenue
WHERE order_date >= CURRENT_DATE() - INTERVAL '7 days'
GROUP BY day
ORDER BY day;
```

If today's row count is 0 or significantly lower: Upstream issue.

### Step 3: Trace Lineage

1. Navigate to OpenMetadata
2. Search for `fct_revenue`
3. Click **Lineage** tab
4. Identify upstream dependencies (e.g., `stg_orders`, `stg_exchange_rates`)
5. Check each upstream table for row counts and freshness

## Resolution

**If upstream table is stale:**
1. Identify which pipeline loads that table (dlt, Airbyte, Snowpipe)
2. Check Prefect for failed flow runs
3. Re-run the pipeline
4. dbt will automatically re-run via Prefect automation

**If upstream table has zero rows:**
1. Check source system (API, database)
2. Verify API credentials are valid
3. Check for API rate limits or outages
4. Re-run extraction pipeline

**If data is present but values are wrong:**
1. Review dbt SQL logic in OpenMetadata (SQL tab)
2. Check for recent code changes (git log)
3. Query intermediate tables to isolate where values become wrong
4. Fix dbt model and re-run

## Prevention

- Add elementary freshness tests to catch stale data:
  ```yaml
  tests:
    - elementary.freshness_anomalies:
        timestamp_column: "loaded_at"
  ```
- Add row count tests:
  ```yaml
  tests:
    - dbt_expectations.expect_table_row_count_to_be_between:
        min_value: 100
        max_value: 10000
  ```

## Escalation

- If unable to identify root cause within 4 hours, escalate to data platform team lead
```

### Store Runbooks

Create a runbooks directory in your documentation:

```
documentation/docs/runbooks/
├── dbt-pipeline-failed.md
├── dashboard-wrong-data.md
├── snowflake-budget-exceeded.md
├── airbyte-sync-failed.md
└── lightdash-unavailable.md
```

Link runbooks in alerts:

```python
slack_webhook.notify(
    text=f"🚨 dbt pipeline failed. See runbook: https://docs.yourcompany.com/runbooks/dbt-pipeline-failed/"
)
```

## On-Call Rotations

For 24/7 support, establish on-call rotations.

### Define On-Call Schedule

**Example rotation:**
- **Primary on-call:** 1 week rotation among 4 data engineers
- **Secondary on-call (escalation):** Data platform lead
- **Coverage:** 24/7 for SEV1, business hours only for SEV2-SEV3

**Schedule in PagerDuty:**
1. Navigate to **Schedules** → **New Schedule**
2. Name: **Data Engineering On-Call**
3. Time zone: Europe/London
4. Rotation type: Weekly
5. Participants: Alice, Bob, Carol, Dave
6. Handoff time: Monday 9:00 AM

### Escalation Policy

Define how alerts escalate if not acknowledged:

**Level 1:** Primary on-call
- Alert sent immediately
- If not acknowledged within 15 minutes → escalate to Level 2

**Level 2:** Secondary on-call (team lead)
- Alert sent
- If not acknowledged within 15 minutes → escalate to Level 3

**Level 3:** Engineering manager
- Alert sent
- Final escalation

**Configure in PagerDuty:**
1. Navigate to **Escalation Policies** → **New Policy**
2. Name: **Data Engineering Escalation**
3. Level 1: Notify **Data Engineering On-Call** schedule
4. Escalate after 15 minutes if not acknowledged
5. Level 2: Notify **Data Platform Lead**
6. Escalate after 15 minutes if not acknowledged
7. Level 3: Notify **Engineering Manager**

### On-Call Best Practices

1. **Acknowledge alerts immediately** — even if you can't fix right away, acknowledging stops escalation
2. **Use runbooks** — don't reinvent solutions, follow tested procedures
3. **Document new issues** — if an incident isn't covered by a runbook, create one after resolving
4. **Handoff clearly** — at end of on-call shift, brief next person on any ongoing incidents
5. **Balance workload** — if one person gets paged excessively, investigate why (bad alerts? infrastructure issues?)

## Incident Response Workflow

### Step 1: Acknowledge

- Acknowledge alert in PagerDuty (stops escalation)
- Post in Slack `#data-critical`: "I'm investigating [incident]"

### Step 2: Assess Severity

- Is this truly SEV1 (outage) or can it be downgraded to SEV2 (quality issue)?
- Adjust severity in PagerDuty if needed

### Step 3: Follow Runbook

- Identify matching runbook
- Follow diagnostic and resolution steps
- Document any deviations from runbook

### Step 4: Communicate Status

**For SEV1:**
- Update stakeholders every 30 minutes:
  - "Investigating root cause..."
  - "Fix deployed, monitoring..."
  - "Incident resolved"

**For SEV2:**
- Single update when resolved

### Step 5: Resolve

- Mark incident as resolved in PagerDuty
- Post resolution message in Slack
- Update any affected dashboards or documentation

### Step 6: Post-Incident Review

**For SEV1 incidents:** Always conduct a post-incident review.

**For SEV2 incidents:** Review if recurrent or caused by systemic issue.

## Post-Incident Reviews

Post-incident reviews (also called post-mortems) focus on learning, not blame.

### Template

```markdown
# Post-Incident Review: [Incident Title]

**Date:** 2026-02-20
**Severity:** SEV1
**Duration:** 2 hours 15 minutes (02:15 - 04:30 UTC)
**Responders:** Alice (primary), Bob (assisted)

## Summary

Brief description of what happened.

## Timeline

- **02:15** — Alert received: dbt pipeline failed
- **02:20** — Alice acknowledged, began investigation
- **02:35** — Root cause identified: Snowflake Resource Monitor suspended warehouse
- **02:40** — Warehouse resumed, dbt pipeline restarted
- **04:15** — dbt pipeline completed successfully
- **04:30** — Dashboards updated, incident resolved

## Root Cause

Snowflake Resource Monitor suspended the TRANSFORMING warehouse after hitting 100% of monthly budget (500 credits). Budget was set conservatively based on initial estimates, but actual usage has grown 30% month-over-month.

## Impact

- **Dashboards stale for 12 hours** (6pm previous day to 6am today)
- **No user-facing reports affected** (users don't access dashboards overnight)
- **No data loss**

## What Went Well

- Alert fired correctly via PagerDuty
- On-call engineer acknowledged within 5 minutes
- Runbook accurately described diagnosis and resolution
- Fix deployed quickly (20 minutes from acknowledgement to resolution)

## What Could Be Improved

- **Budget monitoring:** Resource Monitor budget should have triggered alerts at 90% (it did, but we didn't act on it)
- **Proactive capacity planning:** We should have reviewed credit usage trends monthly and adjusted budget proactively
- **Graceful degradation:** Suspending the warehouse immediately is harsh; consider SUSPEND instead of SUSPEND_IMMEDIATE to allow running queries to finish

## Action Items

| Action | Owner | Deadline | Status |
|--------|-------|----------|--------|
| Increase Resource Monitor quota to 600 credits | Alice | 2026-02-21 | ✅ Done |
| Create monthly cost review meeting | Bob | 2026-02-25 | 🟡 In Progress |
| Change Resource Monitor to SUSPEND (not SUSPEND_IMMEDIATE) at 100% | Alice | 2026-02-22 | ✅ Done |
| Add 75% budget alert to #data-alerts (not just email) | Alice | 2026-02-23 | ✅ Done |
| Document capacity planning process | Bob | 2026-03-01 | ⬜ To Do |

## Lessons Learned

1. **Proactive monitoring is cheaper than reactive fixes** — 5 minutes to review a budget alert vs 2 hours to resolve an outage
2. **Resource Monitor thresholds matter** — SUSPEND allows graceful shutdown, SUSPEND_IMMEDIATE is too aggressive for most cases
3. **Budget growth should be anticipated** — 30% month-over-month growth is predictable, budget should have been increased preemptively
```

### Conduct Review Meeting

1. Schedule within 48 hours of incident
2. Invite responders and stakeholders
3. Review timeline and root cause
4. Discuss what went well and what could improve
5. Define action items with owners and deadlines
6. **No blame** — focus on systemic improvements

### Share Lessons Learned

Post summary in:
- Slack `#data-engineering`
- Team wiki or documentation
- Engineering all-hands (for major incidents)

## Summary

You've established a comprehensive alerting and incident response system:

- [x] **Alert routing** — SEV1 to PagerDuty, SEV2-SEV3 to Slack, severity-based escalation
- [x] **Runbooks** — step-by-step guides for common incidents
- [x] **On-call rotations** — fair distribution of on-call responsibility
- [x] **Incident workflow** — acknowledge, assess, diagnose, resolve, review
- [x] **Post-incident reviews** — learn from incidents and prevent recurrence

Effective incident response minimises downtime and builds a culture of continuous improvement.

## What's Next

Centralise logging for easier debugging and troubleshooting.

Continue to [Logging and Debugging](10-logging-and-debugging.md) →
