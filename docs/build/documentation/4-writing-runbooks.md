# Writing Runbooks

On this page, you will:

- [x] Understand what a runbook is and why you need one
- [x] Learn the standard runbook structure
- [x] Write example runbooks for each repository
- [x] Establish a process for keeping runbooks current

## Overview

A runbook is a step-by-step procedure for performing a specific operational task. Runbooks turn tribal knowledge into documented, repeatable processes. When an alert fires at 2am, the on-call engineer should not need to reverse-engineer the fix from first principles - they follow the runbook.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    RUNBOOK CATEGORIES                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ROUTINE OPERATIONS         INCIDENT RESPONSE         MAINTENANCE       │
│  ──────────────────         ─────────────────         ───────────       │
│                                                                         │
│  Add a new user             Failed Terraform deploy   Rotate creds     │
│  Add a data source          Pipeline backfill needed   Upgrade dbt      │
│  Add a dbt model            Snowflake credit spike    Patch Prefect    │
│  Grant role access          Data quality alert         Review costs     │
│  Configure new pipeline     Stale data detected       Prune old data   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## When to Write a Runbook

Write a runbook when:

- A task involves more than three steps
- Someone other than the original author may need to perform it
- The task involves production systems where mistakes have consequences
- You find yourself explaining the same process to multiple people
- An incident occurs and you want to be prepared for next time

You do not need a runbook for tasks that are self-explanatory from the code (e.g. "run `terraform plan`") or tasks that have only one step.

## Runbook Structure

Every runbook follows the same structure. Consistency makes runbooks faster to use under pressure - engineers know exactly where to find each piece of information.

### Template

```markdown
# Runbook: [Task Name]

## Summary

One or two sentences explaining what this runbook covers and when to use it.

## When to Use

- Trigger condition 1 (e.g. "A new team member needs Snowflake access")
- Trigger condition 2 (e.g. "PagerDuty alert: snowflake_user_access_denied")

## Prerequisites

- [ ] Access: [what access is needed, e.g. "Terraform repository write access"]
- [ ] Tools: [what tools are needed, e.g. "AWS CLI configured with infrastructure-admin profile"]
- [ ] Context: [any information needed, e.g. "New user's email and role requirements"]

## Steps

### 1. [First Step]

[Detailed instructions with commands and expected output]

### 2. [Second Step]

[Continue with each step]

### 3. [Third Step]

[And so on]

## Verification

How to confirm the task completed successfully:

- [ ] Check 1
- [ ] Check 2

## Rollback

If something goes wrong, how to undo the changes:

1. [Rollback step 1]
2. [Rollback step 2]

## Escalation

If the runbook does not resolve the issue:

- **First contact**: [team or person, e.g. "Data Engineering team in #data-eng Slack"]
- **Escalation**: [next level, e.g. "Infrastructure team lead"]
```

## Example Runbooks

### Terraform: Adding a New Snowflake User

This example demonstrates a complete runbook for one of the most common operational tasks.

```markdown
# Runbook: Add a New Snowflake User

## Summary

Add a new team member to Snowflake via Terraform, granting them appropriate
roles and warehouse access.

## When to Use

- A new team member joins and needs Snowflake access
- An existing user needs a different access level

## Prerequisites

- [ ] Access: Write access to the terraform repository
- [ ] Tools: Terraform CLI, AWS CLI with `infrastructure-admin` profile
- [ ] Context: User's name, email, desired role (developer or admin)

## Steps

### 1. Determine the User Category

| Role | Default Role | Default Warehouse | Example |
|------|--------------|-------------------|---------|
| Admin | SYSADMIN | DEVELOPER | JBLOGGS_ADMIN |
| Developer | ANALYTICS_DEVELOPER | DEVELOPER | JBLOGGS |

### 2. Add to users.auto.tfvars

Edit `snowflake/config/users.auto.tfvars` and add the user to the
appropriate list:

` ``hcl
developer_user_list = {
  # ... existing users ...
  JBLOGGS = {
    email        = "jane.bloggs@company.com"
    first_name   = "Jane"
    last_name    = "Bloggs"
    display_name = "Jane Bloggs"
  }
}
` ``

### 3. Run Terraform Plan

` ``sh
cd snowflake/config
terraform plan
# Verify: should show new user resource being created
` ``

### 4. Create Pull Request

` ``sh
git checkout -b add-user-jbloggs
git add snowflake/config/users.auto.tfvars
git commit -m "Add Jane Bloggs as Snowflake developer"
git push -u origin add-user-jbloggs
# Create PR via GitHub
` ``

### 5. After Merge

CI/CD runs `terraform apply` automatically. The user is created with a
temporary password.

### 6. Provide Credentials

1. Set the user's initial password via Snowflake UI (as USERADMIN)
2. Send credentials via 1Password secure sharing
3. Ask user to change password and enable MFA on first login

## Verification

- [ ] User appears in Snowflake: `SHOW USERS LIKE 'JBLOGGS';`
- [ ] User has correct role: `SHOW GRANTS TO USER JBLOGGS;`
- [ ] User can log in and run a test query

## Rollback

1. Revert the PR (GitHub → PR → Revert)
2. Merge the revert PR — CI/CD runs `terraform apply` to remove the user
3. Or manually: `DROP USER JBLOGGS;` (as USERADMIN)

## Escalation

- **First contact**: Data Engineering team in #data-eng Slack
- **Escalation**: Infrastructure team lead
```

!!! tip "Claude Skills and Runbooks"
    If you have the `add-snowflake-user` Claude skill configured (covered in the [Claude Code Setup](../../getting-started/initial-setup/5-claude-code-setup.md) page), Claude can perform steps 2-4 automatically. The runbook still documents the full process for cases where Claude is not available or the task needs manual review.

### Data Pipelines: Responding to a Failed Pipeline

```markdown
# Runbook: Respond to a Failed Pipeline

## Summary

Diagnose and resolve a dlt pipeline failure reported by Prefect alerting.

## When to Use

- Prefect sends a Slack/PagerDuty alert for a failed flow run
- Data freshness SLA is at risk

## Prerequisites

- [ ] Access: Prefect Cloud dashboard or self-hosted UI
- [ ] Access: AWS CloudWatch Logs (for ECS-hosted workers)
- [ ] Access: Snowflake (to verify data state)

## Steps

### 1. Check the Flow Run in Prefect

1. Open Prefect UI → Flow Runs
2. Find the failed run and click into it
3. Read the error message and traceback

### 2. Identify the Failure Category

| Error Pattern | Likely Cause | Action |
|---------------|-------------|--------|
| Connection refused / timeout | Source API down | Wait and retry |
| Authentication error | Credentials expired | Rotate credentials |
| Schema change | Source schema evolved | Update dlt schema |
| Rate limit exceeded | Too many API calls | Increase backoff |
| Snowflake error | Warehouse suspended / permissions | Check Snowflake |

### 3. Retry the Flow Run

If the failure is transient (network, rate limit):

1. In Prefect UI, click **Retry** on the failed run
2. Monitor the retry

### 4. Fix and Redeploy (if code change needed)

1. Fix the issue in the pipeline code
2. Test locally: `python -m pipelines.<pipeline_name>`
3. Create PR, merge, deploy via CI/CD
4. Trigger a manual run in Prefect

## Verification

- [ ] Flow run completes successfully in Prefect
- [ ] Data appears in Snowflake with expected row counts
- [ ] No downstream dbt test failures

## Rollback

If the fix introduces new issues:

1. Revert the PR
2. Trigger the previous pipeline version via Prefect

## Escalation

- **First contact**: Data Engineering team in #data-eng Slack
- **Escalation**: On-call data engineer
```

### dbt: Running a Full Refresh

```markdown
# Runbook: Run a Full Refresh for a dbt Model

## Summary

Rebuild a dbt incremental model from scratch, replacing all data.

## When to Use

- Incremental logic has changed and historical data needs reprocessing
- Source data was corrected retroactively
- Model has accumulated errors from incremental loads

## Prerequisites

- [ ] Access: dbt CLI configured with Snowflake credentials
- [ ] Context: Model name and reason for full refresh
- [ ] Timing: Schedule during low-usage window (full refresh uses more compute)

## Steps

### 1. Verify the Model is Incremental

` ``sh
dbt ls --select model_name --output json | grep materialized
# Should show: "materialized": "incremental"
` ``

### 2. Run the Full Refresh

` ``sh
dbt run --select model_name --full-refresh
` ``

### 3. Run Tests

` ``sh
dbt test --select model_name
` ``

### 4. Verify Row Counts

` ``sql
SELECT COUNT(*) FROM ANALYTICS.MARTS.MODEL_NAME;
-- Compare with expected count
` ``

## Verification

- [ ] Model rebuilt successfully (no dbt errors)
- [ ] All tests pass
- [ ] Row count is within expected range
- [ ] Downstream models still function correctly

## Rollback

Incremental models retain historical data via Snowflake Time Travel:

` ``sql
-- Restore to state before full refresh (within retention period)
CREATE OR REPLACE TABLE ANALYTICS.MARTS.MODEL_NAME
  CLONE ANALYTICS.MARTS.MODEL_NAME AT (OFFSET => -3600);
` ``

## Escalation

- **First contact**: Analytics Engineering team in #analytics Slack
- **Escalation**: dbt project owner
```

## Including Runbooks in the Docs Site

Runbooks live in each repository's `docs/runbooks/` directory. The `index.md` page in that directory serves as a table of contents:

```markdown
# Runbooks

Operational procedures for the Terraform infrastructure repository.

| Runbook | When to Use |
|---------|------------|
| [Add a Snowflake User](add-snowflake-user.md) | New team member needs access |
| [Rotate Credentials](rotate-credentials.md) | Scheduled rotation or compromise |
| [Failed Deployment](failed-deployment.md) | CI/CD terraform apply fails |
| [Add a Data Source](add-data-source.md) | New ingestion source needed |
```

Update each repository's `docs/mkdocs.yml` nav to include runbook pages as they are written.

## Keeping Runbooks Current

Runbooks go stale quickly if not maintained. Several practices help:

### Test Runbooks Periodically

Run through each runbook at least once per quarter, even if there is no active need. This catches:

- Commands that no longer work due to tool updates
- Steps that assume configuration that has changed
- Links to dashboards or tools that have moved

### Link Runbooks to Alerts

When creating Prefect automations, PagerDuty alerts, or Slack notifications, include a link to the relevant runbook:

```python
# In Prefect automation
send_slack_notification(
    message=f"Pipeline {flow_name} failed. "
    f"Runbook: https://<your-org>.github.io/technical-documentation/"
    f"data-pipelines/runbooks/failed-pipeline/"
)
```

### Update After Incidents

After every incident, review the runbook used:

- Did the steps work as documented?
- Were any steps missing?
- Did the verification steps catch the resolution?
- Does the escalation path need updating?

Update the runbook in the same PR as any code fixes from the incident.

## Summary

!!! success "What You've Accomplished"
    - [x] Understand when and why to write runbooks
    - [x] Know the standard runbook structure (summary, when to use, prerequisites, steps, verification, rollback, escalation)
    - [x] Have example runbooks for Terraform, data pipelines, and dbt
    - [x] Know how to organise and maintain runbooks

## Next Steps

With documentation content in place, the next step is to set up CI/CD - validating documentation builds on pull requests and deploying the unified site to GitHub Pages automatically.

Continue to [CI/CD Pipeline](5-ci-cd-pipeline.md) →
