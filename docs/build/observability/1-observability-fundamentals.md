# Observability Fundamentals

On this page, you will:

- [x] Understand the five dimensions of data quality
- [x] Learn SLIs, SLOs, and SLAs for data pipelines
- [x] Distinguish between observability, monitoring, and testing
- [x] Understand incident severity levels
- [x] Know when to alert, log, or ignore issues

## Overview

Before building observability infrastructure, you need to understand the fundamentals: what makes data "good quality", how to measure pipeline reliability, and when to take action on issues.

This page covers the core concepts that underpin all observability practices in data engineering.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    OBSERVABILITY FUNDAMENTALS                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Data Quality Dimensions          Reliability Metrics                  │
│  ───────────────────────          ───────────────────                  │
│                                                                         │
│  • Accuracy                       • SLI (Indicator)                    │
│  • Completeness                     "dbt run success rate"             │
│  • Consistency                    • SLO (Objective)                    │
│  • Timeliness                       "99% of runs succeed"              │
│  • Validity                       • SLA (Agreement)                    │
│                                     "We guarantee 99% uptime"          │
│                                                                         │
│  Incident Severity                Action Thresholds                    │
│  ─────────────────                ──────────────────                   │
│                                                                         │
│  • SEV1: Data outage              Alert → Immediate page-out           │
│  • SEV2: Quality degraded         Alert → Next business day           │
│  • SEV3: Minor issue              Log → Review weekly                 │
│  • SEV4: Cosmetic                 Log → Review monthly                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## The Five Dimensions of Data Quality

Data quality is multi-dimensional. A dataset can be accurate but late, or complete but inconsistent. Understanding these dimensions helps you design appropriate tests and monitors.

### 1. Accuracy

**Definition:** Data correctly represents the real-world entity or event it describes.

**Examples:**
- ✅ Good: Customer email is `alice@example.com` (matches reality)
- ❌ Bad: Customer email is `bob@example.com` (wrong person)

**How to measure:**
- Compare data to authoritative source (e.g., manual audit sample)
- Check calculations match expected results (e.g., `revenue = quantity * price`)
- Validate against business rules (e.g., discounts never exceed 100%)

**dbt test example:**
```yaml
# Ensure calculated revenue matches sum of line items
- name: order_total_accuracy
  test: dbt_utils.expression_is_true
  config:
    expression: "abs(order_total - (quantity * unit_price)) < 0.01"
```

### 2. Completeness

**Definition:** All expected data is present; no missing values or records.

**Examples:**
- ✅ Good: All 1000 orders from yesterday loaded into warehouse
- ❌ Bad: Only 950 orders loaded (50 missing)

**How to measure:**
- Row count matches source system
- No unexpected NULL values in required columns
- All time periods have data (no gaps in daily data)

**dbt test example:**
```yaml
# Ensure critical columns have no nulls
- name: customer_id
  tests:
    - not_null
    - dbt_expectations.expect_column_values_to_not_be_null:
        row_condition: "order_status != 'cancelled'"
```

### 3. Consistency

**Definition:** Data is uniform across systems and over time; no contradictions.

**Examples:**
- ✅ Good: Customer name is "Alice Smith" in both CRM and orders table
- ❌ Bad: Customer name is "Alice Smith" in CRM, "A. Smith" in orders

**How to measure:**
- Cross-system comparisons (CRM vs warehouse)
- Referential integrity (foreign keys exist)
- Temporal consistency (values don't change illogically)

**dbt test example:**
```yaml
# Ensure foreign keys are valid
- name: customer_id
  tests:
    - relationships:
        to: ref('dim_customers')
        field: customer_id
```

### 4. Timeliness

**Definition:** Data is available when needed; minimal lag between event and availability.

**Examples:**
- ✅ Good: Yesterday's orders available by 6am today
- ❌ Bad: Yesterday's orders available at 2pm today (8 hour delay)

**How to measure:**
- Data freshness (max age of data)
- Pipeline SLAs (ingestion completed within X hours)
- Time-to-availability metrics

**dbt test example:**
```yaml
# dbt source freshness check
sources:
  - name: raw_orders
    freshness:
      warn_after: {count: 6, period: hour}
      error_after: {count: 12, period: hour}
```

### 5. Validity

**Definition:** Data conforms to defined formats, types, and rules.

**Examples:**
- ✅ Good: Email is `user@domain.com` (valid format)
- ❌ Bad: Email is `not_an_email` (invalid format)

**How to measure:**
- Type checking (dates are dates, integers are integers)
- Format validation (emails, phone numbers, postcodes)
- Range validation (age between 0 and 120)

**dbt test example:**
```yaml
# Ensure values are within expected ranges
- name: order_total
  tests:
    - dbt_expectations.expect_column_values_to_be_between:
        min_value: 0
        max_value: 1000000
```

## SLIs, SLOs, and SLAs

These metrics help you measure and communicate pipeline reliability.

### SLI (Service Level Indicator)

**Definition:** A quantifiable measure of service quality.

**Data pipeline examples:**
- dbt run success rate: `successful_runs / total_runs`
- Data freshness: `max(loaded_at) - current_time`
- Pipeline duration: `time(end) - time(start)`
- Data quality: `passed_tests / total_tests`

**How to calculate:**
```sql
-- dbt run success rate over last 30 days
SELECT
    COUNT(CASE WHEN status = 'success' THEN 1 END) * 100.0 / COUNT(*) AS success_rate_pct
FROM dbt_run_history
WHERE run_date >= CURRENT_DATE() - INTERVAL '30 days';
```

### SLO (Service Level Objective)

**Definition:** Target value or range for an SLI. Your internal goal.

**Data pipeline examples:**
- dbt run success rate ≥ 99% (99% of runs succeed)
- Data freshness ≤ 4 hours (data available within 4 hours of event)
- Pipeline duration ≤ 30 minutes (95th percentile)
- Data quality ≥ 95% (95% of tests pass)

**SLOs are:**
- **Internal targets** (not customer-facing promises)
- **Realistic** (based on historical performance)
- **Measurable** (can be calculated from SLIs)
- **Time-bound** (measured over a period, e.g., 30 days)

**Example SLO definition:**
```yaml
# SLOs for the dbt daily run
slos:
  - name: dbt_daily_run_success_rate
    sli: successful_dbt_runs / total_dbt_runs
    target: 0.99  # 99%
    period: 30_days

  - name: dbt_daily_run_duration
    sli: p95(dbt_run_duration_minutes)
    target: 30  # 30 minutes
    period: 30_days
```

### SLA (Service Level Agreement)

**Definition:** A contractual commitment to customers. What you **promise**.

**Data pipeline examples:**
- "Marketing dashboard updated by 8am daily" (customer: marketing team)
- "Financial reports available within 24 hours of month-end" (customer: finance team)
- "Customer data deleted within 30 days of request" (customer: GDPR compliance)

**SLAs are:**
- **External commitments** (to stakeholders, customers, regulators)
- **Contractual** (may have penalties if violated)
- **Stricter than SLOs** (SLO = 99%, SLA = 95% to leave error budget)

**Example SLA:**
```markdown
## Marketing Dashboard SLA

**Commitment:** Marketing KPI dashboard refreshed by 08:00 UTC daily.

**Scope:** Includes metrics from Airbyte (HubSpot), dbt (fct_contacts), Lightdash (dashboard)

**Target:** 95% of days (28.5 days per month)

**Penalties:** None (internal SLA)

**Escalation:** If not met, page on-call data engineer.
```

### Relationship Between SLI, SLO, SLA

```
SLI (Measurement)
    ↓
SLO (Internal Target)
    ↓
SLA (External Promise)

Example:
SLI: dbt run success rate = 99.2% (last 30 days)
SLO: Target ≥ 99%
SLA: Promise to marketing team: "Dashboard updated 95% of days"
```

**Error Budget:**
If your SLO is 99%, you have a 1% error budget (3.6 hours/month of downtime allowable).

## Observability vs Monitoring vs Testing

These concepts overlap but have distinct purposes.

### Testing

**Definition:** Validate data against known rules at a specific point in time.

**Characteristics:**
- **Proactive** — define rules upfront
- **Binary** — pass or fail
- **Known rules** — test what you know to check

**Examples:**
- dbt test: `customer_id` must be unique
- dbt test: `order_total` must be ≥ 0
- Python test: `assert len(df) > 0`

**When to use:** You know what "good" looks like and can define explicit rules.

### Monitoring

**Definition:** Track metrics over time and alert when thresholds are crossed.

**Characteristics:**
- **Reactive** — respond to threshold violations
- **Time-series** — track trends, not single points
- **Known thresholds** — alert when metric exceeds/drops below limit

**Examples:**
- Alert if dbt run duration > 60 minutes
- Alert if row count drops 50% from 7-day average
- Alert if Snowflake credit usage > $100/day

**When to use:** You know what metrics matter and can define "normal" ranges.

### Observability

**Definition:** Understand system state from outputs; explore unknowns with full context.

**Characteristics:**
- **Exploratory** — ask new questions after incidents
- **Context-rich** — combines logs, metrics, lineage, metadata
- **Unknown unknowns** — investigate issues you didn't anticipate

**Examples:**
- "Why did revenue drop 20% last Tuesday?" → Use lineage to trace upstream
- "Which dashboard queries are slowest?" → Analyse query logs + BI tool metadata
- "What broke when we changed this dbt model?" → Impact analysis via lineage

**When to use:** Debugging complex issues, root cause analysis, understanding cascading failures.

### Comparison Table

| | Testing | Monitoring | Observability |
|--|---------|-----------|---------------|
| **Purpose** | Validate known rules | Track known metrics | Explore unknowns |
| **Timing** | Point-in-time | Continuous | Post-incident |
| **Output** | Pass/Fail | Metrics, alerts | Insights, root causes |
| **Tools** | dbt tests, Great Expectations | Prefect, Elementary, Datadog | OpenMetadata, logs, lineage |
| **Example** | "Is customer_id unique?" | "Is row count stable?" | "Why did the pipeline fail?" |

**You need all three:**
- **Tests** catch known bad data
- **Monitoring** alerts you to anomalies
- **Observability** helps you debug and learn

## Incident Severity Levels

Not all issues are equal. Classify incidents by severity to prioritize response.

### SEV1 (Critical) - Data Outage

**Definition:** Core data pipelines broken; critical dashboards unavailable.

**Examples:**
- dbt daily run failed; all dashboards stale
- Snowflake warehouse suspended due to credit limit
- Airbyte sync broken for primary revenue data source

**Response:**
- **Immediate page-out** (call on-call engineer)
- **Drop everything** to resolve
- **Target resolution:** < 2 hours
- **Post-incident review:** Always

**Alert channel:** PagerDuty (page), Slack (urgent), Email

### SEV2 (High) - Quality Degraded

**Definition:** Data available but quality issues; non-critical dashboards affected.

**Examples:**
- dbt test failures (data loaded but may be incorrect)
- Anomaly detected: row count 30% lower than expected
- Secondary data source (e.g., product catalogue) stale

**Response:**
- **Alert on-call during business hours**
- **Investigate within 4 hours**
- **Target resolution:** < 1 business day
- **Post-incident review:** If recurrent

**Alert channel:** Slack (non-urgent), Email

### SEV3 (Medium) - Minor Issue

**Definition:** Edge case failure; minimal user impact.

**Examples:**
- Single dbt test failed on a rarely-used model
- Log warnings (not errors)
- Performance degradation (queries 2x slower than normal, but still < 10 seconds)

**Response:**
- **Log for review**
- **Investigate weekly**
- **Target resolution:** < 1 week
- **Post-incident review:** No

**Alert channel:** Log only (no Slack/email)

### SEV4 (Low) - Cosmetic

**Definition:** Documentation, cosmetic issues; no functional impact.

**Examples:**
- dbt model missing description
- Dashboard title typo
- Log noise (non-actionable warnings)

**Response:**
- **Backlog ticket**
- **Fix opportunistically**
- **Target resolution:** No deadline
- **Post-incident review:** No

**Alert channel:** None (create Jira/Linear ticket)

## When to Alert, Log, or Ignore

**Alert fatigue** is real. Too many alerts → ignored alerts → missed critical issues.

### Alert (Send to Slack/PagerDuty/Email)

**When:**
- SEV1 or SEV2 incidents
- Immediate action required
- User-facing impact (dashboards broken, data stale)
- SLA at risk

**Examples:**
- dbt run failed
- Data freshness > 12 hours (SLA is 6 hours)
- Snowflake Resource Monitor hit 90% of budget
- More than 3 dbt tests failed

**Alert criteria:**
- **Actionable** — someone can fix it now
- **Urgent** — needs attention within hours
- **User-facing** — affects stakeholders

### Log (Write to CloudWatch/File; Review Periodically)

**When:**
- SEV3 or SEV4 incidents
- Informational (not actionable)
- Debugging context (for future incidents)
- Performance metrics (for trend analysis)

**Examples:**
- dbt test passed (log for audit trail)
- Query took 5 seconds (within normal range)
- Prefect task retried once and succeeded
- Row count within expected range but at the low end

**Log criteria:**
- **Not urgent** — can review later
- **Informational** — good to know, not actionable
- **Trending** — useful for pattern detection

### Ignore (Don't Alert or Log)

**When:**
- Expected behavior
- Transient issues that auto-resolve
- Noise (e.g., connection retries that succeed)

**Examples:**
- Snowflake warehouse auto-suspended after idle time (expected)
- Prefect task retried once and succeeded (handled automatically)
- Log message "Query compiled successfully" (too noisy, not useful)

**Ignore criteria:**
- **Expected** — normal system behavior
- **Auto-resolved** — transient, self-healing
- **Too frequent** — would create noise

### Decision Tree

```
Issue detected
    │
    ├─ User-facing impact? ─── YES ──▶ Alert (SEV1/SEV2)
    │                           NO
    │                            ↓
    ├─ Actionable now? ──────── YES ──▶ Alert (SEV2/SEV3)
    │                           NO
    │                            ↓
    ├─ Useful for debugging? ── YES ──▶ Log (SEV3/SEV4)
    │                           NO
    │                            ↓
    └─ Ignore (noise, expected behavior)
```

## Summary

You've learned observability fundamentals:

- [x] **Five data quality dimensions** — Accuracy, Completeness, Consistency, Timeliness, Validity
- [x] **SLIs, SLOs, SLAs** — Measure reliability (SLI), set targets (SLO), promise uptime (SLA)
- [x] **Observability vs Monitoring vs Testing** — Explore (observability), track (monitoring), validate (testing)
- [x] **Incident severity** — SEV1 (outage), SEV2 (degraded), SEV3 (minor), SEV4 (cosmetic)
- [x] **Alert thresholds** — Alert (urgent), Log (review later), Ignore (noise)

These concepts underpin all observability tools and practices. You'll apply them throughout this section when configuring dbt tests, Elementary anomaly detection, and Prefect monitoring.

## What's Next

Apply these fundamentals to data quality testing with dbt.

Continue to [Data Quality with dbt](2-data-quality-with-dbt.md) →
