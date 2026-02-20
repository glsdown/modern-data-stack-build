# Choosing a BI Tool

On this page, you will:

- [x] Learn the key criteria for evaluating BI tools
- [x] Understand how your budget, team, and requirements narrow your options
- [x] Use a decision framework to choose the right tool
- [x] Know when to start simple and when to invest in enterprise tools

## Overview

With dozens of BI tools available, choosing the right one requires evaluating your organisation's specific needs, constraints, and capabilities. This page provides a structured framework to narrow your options and make an informed decision.

The goal is not to find the "best" BI tool — there isn't one. The goal is to find the **best fit** for your situation right now, with the flexibility to evolve as your needs change.

## The Key Questions

Start by answering these five questions. Your answers will eliminate most tools immediately and point you toward 2-3 finalists.

### 1. Do You Have a dbt Transformation Layer?

This is the first decision point.

**If YES (you have dbt models):**
- Consider **dbt-native tools** (Lightdash, Omni)
- These tools read your dbt project and understand metrics defined in code
- You get version-controlled metrics, testable business logic, and dbt lineage awareness
- Alternative: use general-purpose tools (Metabase, Tableau) but lose metrics-as-code benefits

**If NO (no dbt, or loading raw data directly to BI tool):**
- Focus on **general-purpose tools** (Metabase, Tableau, Power BI, Superset)
- dbt-native tools add no value without a dbt project
- Metrics will be defined in the BI tool UI (not code)

!!! tip "dbt Changes the Decision"
    If you've built the transformation layer in this documentation, you **have dbt**. Lightdash and Omni will leverage your existing dbt work. General-purpose tools will ignore it.

### 2. What Is Your Monthly Budget?

Budget determines whether enterprise, modern, open source, or free tools are realistic.

| Budget Range | Realistic Options |
|--------------|-------------------|
| **$0-50/month** | Snowsight (free), Looker Studio (free), Lightdash self-hosted (~$30), Metabase self-hosted (~$25), Superset self-hosted (~$30) |
| **$50-500/month** | Omni (3-10 users), QuickSight (few users), Power BI (few users) |
| **$500-2000/month** | Power BI (10-50 users), Metabase Cloud (10-20 users), Lightdash Cloud ($2400 unlimited) |
| **$2000+/month** | Tableau (10+ users), Looker (10+ users), Mode (10+ users), Hex (10+ users) |

!!! warning "Hidden Costs"
    Self-hosted tools are "free" but require infrastructure (~$20-40/month AWS costs) and engineering time for setup, upgrades, and maintenance. Factor in at least 4-8 hours/month for maintenance.

### 3. Can You Self-Host, or Do You Need Cloud SaaS?

Self-hosting requires infrastructure knowledge and ongoing maintenance. Cloud SaaS is easier but more expensive and offers less control.

**Choose self-hosted if:**
- You have DevOps/infrastructure expertise on your team
- You want full control over data and deployment
- Budget is limited but you have engineering time
- You're comfortable with Docker, ECS, or Kubernetes
- You manage other self-hosted services (Prefect, Airflow, etc.)

**Choose cloud SaaS if:**
- You want zero infrastructure management
- Your team is small and engineering time is limited
- Budget allows for SaaS pricing
- You prefer vendor-managed updates and support
- Compliance allows external BI tool hosting

!!! info "Data Residency"
    Even cloud BI tools typically don't copy your entire warehouse. They query on-demand or cache aggregated results. Check each tool's architecture if data residency is a concern.

### 4. Who Are Your Primary Users?

Different tools serve different user types. Know your audience.

| User Type | Needs | Best Tools |
|-----------|-------|-----------|
| **Analysts (SQL-proficient)** | SQL queries, ad-hoc exploration, basic charts | Snowsight, Mode, Redash, Metabase |
| **Business users (non-technical)** | Drag-and-drop, self-service, no SQL | Tableau, Power BI, Metabase, Looker |
| **Data scientists** | Python/R notebooks, ML, exploratory analysis | Hex, Snowflake Notebooks, Jupyter |
| **Executives** | Polished dashboards, mobile access, scheduled reports | Tableau, Power BI, Looker, Lightdash |
| **Analytics engineers** | Metrics as code, dbt integration, version control | Lightdash, Omni |

**Mixed teams?** You may need multiple tools:
- Snowsight or notebooks for analysts doing ad-hoc SQL/Python work
- Lightdash or Tableau for business users and executives consuming dashboards

### 5. What Visualisation Complexity Do You Need?

Be honest about what you actually need, not what you might want someday.

**Basic needs** (most teams):
- Line charts, bar charts, pie charts
- Tables and pivot tables
- Filters and drill-down
- Scheduled dashboards

**→ Lightdash, Metabase, Omni, Power BI, Snowsight**

**Advanced needs** (large organisations, embedded analytics):
- Custom visualisations and D3.js charts
- Geospatial analysis and maps
- Embedded dashboards in external applications
- Complex interactivity and animations

**→ Tableau, Looker, Power BI (premium), Superset**

!!! tip "Start Simple"
    Most teams overestimate their visualisation needs. Start with a simpler tool (Lightdash, Metabase) and upgrade to Tableau/Looker later if you hit limitations. The reverse migration (Tableau → Lightdash) is much harder.

## Decision Framework

Use this flowchart to narrow your options:

```
Do you have a dbt transformation layer?
│
├─ YES → Do you want metrics as code (version-controlled)?
│   │
│   ├─ YES → Budget?
│   │   ├─ $0-100/month → Lightdash (self-hosted)
│   │   ├─ $100-500/month → Omni (cloud)
│   │   └─ $2400+/month → Lightdash Cloud or Looker
│   │
│   └─ NO → Treat as general-purpose BI (see below)
│
└─ NO → Can you self-host?
    │
    ├─ YES → Budget?
    │   ├─ $0-50/month → Snowsight (free) or Metabase (self-hosted)
    │   └─ $50+/month → Superset (self-hosted) for advanced viz
    │
    └─ NO → Budget?
        ├─ $0 → Looker Studio (Google) or Snowsight
        ├─ $10-500/month → Power BI or QuickSight
        ├─ $500-2000/month → Power BI (larger team) or Metabase Cloud
        └─ $2000+/month → Tableau or Looker (enterprise features)
```

## Evaluation Criteria Deep Dive

### Cost Model

Understand how the tool charges and what your actual cost will be:

| Model | Examples | Watch Out For |
|-------|----------|---------------|
| **Per-user/month** | Tableau, Power BI, Mode, Hex | Costs scale linearly with users; viewer tiers may be cheaper |
| **Flat rate** | Lightdash Cloud ($2400/month) | Good for large teams, expensive for small teams |
| **User tiers** | Tableau (Creator/Explorer/Viewer) | Understand which tier each user type needs |
| **Usage-based** | QuickSight (pay-per-session option) | Unpredictable costs if usage spikes |
| **Infrastructure-only** | Lightdash, Metabase (self-hosted) | Infrastructure ($20-40/month) + engineering time |
| **Free** | Snowsight, Looker Studio | Warehouse compute costs (Snowsight) or performance limits |

### Integration and Data Sources

Ensure the tool connects to your data warehouse and understands your data model:

| Tool | Snowflake Support | dbt Integration | Other Sources |
|------|-------------------|-----------------|---------------|
| **Lightdash** | ✅ Native | ✅ Native (reads dbt YAML) | Limited (dbt sources only) |
| **Omni** | ✅ Native | ✅ Native (reads dbt YAML) | PostgreSQL, BigQuery, Redshift |
| **Metabase** | ✅ Native | ❌ None | 30+ databases |
| **Tableau** | ✅ Native | ❌ None | 100+ connectors |
| **Power BI** | ✅ Native | ❌ None | 100+ connectors |
| **Looker** | ✅ Native | ⚠️ Partial (separate LookML) | Most SQL databases |
| **Superset** | ✅ Native | ❌ None | Most SQL databases |
| **Snowsight** | ✅ Native (built-in) | ❌ None | Snowflake only |

### Governance and Security

Enterprise organisations need robust access controls and audit logs:

| Feature | Enterprise (Tableau/Looker) | Modern (Lightdash/Omni) | Open Source (Metabase) |
|---------|----------------------------|------------------------|----------------------|
| **SSO (SAML/OKTA)** | ✅ Standard | ✅ Paid tiers | ✅ Enterprise edition |
| **Row-level security** | ✅ Advanced | ⚠️ Basic | ⚠️ Basic |
| **Audit logs** | ✅ Detailed | ⚠️ Basic | ⚠️ Basic |
| **Role-based access** | ✅ Granular | ✅ Standard | ✅ Standard |
| **Data lineage** | ⚠️ Via plugins | ✅ dbt-native | ❌ None |
| **Certified datasets** | ✅ Yes | ⚠️ Via dbt | ❌ None |

### Developer Experience

For analytics engineers and data teams, developer experience matters:

| Feature | Lightdash | Omni | Looker | Metabase | Tableau |
|---------|-----------|------|--------|----------|---------|
| **Metrics as code** | ✅ dbt YAML | ✅ dbt YAML | ✅ LookML | ❌ UI | ❌ UI |
| **Version control** | ✅ Git | ✅ Git | ✅ Git | ❌ UI | ❌ UI |
| **CI/CD** | ✅ dbt CI | ✅ dbt CI | ✅ LookML CI | ⚠️ API-based | ⚠️ API-based |
| **Testable logic** | ✅ dbt tests | ✅ dbt tests | ⚠️ LookML tests | ❌ No | ❌ No |
| **API access** | ✅ REST API | ✅ REST API | ✅ REST API | ✅ REST API | ✅ REST API |
| **Terraform provider** | ⚠️ Community | ❌ No | ✅ Official | ⚠️ Community | ✅ Official |

### Learning Curve and Support

Consider how quickly your team can get productive:

| Tool | Learning Curve | Documentation | Community Size | Professional Services |
|------|---------------|---------------|----------------|----------------------|
| **Snowsight** | Very low | Excellent (Snowflake docs) | Large | Via Snowflake |
| **Metabase** | Low | Good | Large | Limited |
| **Power BI** | Medium | Excellent (Microsoft docs) | Very large | Extensive |
| **Lightdash** | Medium | Good | Growing | Community support |
| **Tableau** | High | Excellent | Very large | Extensive |
| **Looker** | High (LookML) | Good | Large | Extensive (Google) |
| **Superset** | Medium-high | Fair | Medium | Community support |

## Common Scenarios and Recommendations

### Scenario 1: Startup with dbt, Budget $0-100/month

**Your situation:**
- Built dbt transformation layer following this documentation
- 2-5 person data team
- Budget is very limited
- Team has DevOps skills (already self-hosting Prefect)

**Recommendation: Lightdash (self-hosted)**

Why:
- Free except infrastructure (~$30-40/month)
- Native dbt integration (use your existing metrics)
- Team already manages self-hosted infrastructure
- Can upgrade to Lightdash Cloud ($2400/month) later if needed

**Alternative: Snowsight (free quick win)**
- Use while evaluating Lightdash
- Good for SQL-proficient analysts
- Zero additional cost

### Scenario 2: Mid-Size Company, Budget $500-2000/month, Microsoft-Heavy

**Your situation:**
- 20-50 employees using Office 365 and Azure
- Mix of technical and non-technical users
- Budget allows $500-2000/month
- No dbt transformation layer (raw data → BI tool)

**Recommendation: Power BI**

Why:
- Low cost ($10-20/user/month)
- Familiar to Excel users
- Tight Office 365 integration
- Self-service for business users
- Within budget for 50 users (~$1000/month)

**Alternative: Metabase Cloud**
- If you don't use Microsoft ecosystem
- Easier for non-technical users than Power BI

### Scenario 3: Enterprise, Budget $5000+/month, Complex Viz Needs

**Your situation:**
- 100+ employees across multiple departments
- Need embedded analytics in customer-facing applications
- Complex visualisation requirements
- Budget allows enterprise pricing

**Recommendation: Tableau or Looker**

Why:
- Industry-leading visualisation capabilities
- Embedded analytics support
- Enterprise features (SSO, governance, audit logs)
- Large community and professional services

**Tableau if:** Best-in-class visualisations, polished dashboards
**Looker if:** You want metrics as code (LookML) and Google Cloud integration

### Scenario 4: Data Team with dbt, Budget $100-500/month

**Your situation:**
- 5-15 person data/analytics team
- Built dbt transformation layer
- Want metrics as code but can't self-host
- Budget allows a few hundred per month

**Recommendation: Omni**

Why:
- Native dbt integration (metrics in dbt YAML)
- Cloud-hosted (no infrastructure management)
- More polished UX than Lightdash
- $60-300/month for 3-15 users

**Alternative: Lightdash self-hosted**
- If you can manage infrastructure
- Save money (~$30/month vs $60-300/month)

### Scenario 5: Analysts Only, SQL-Proficient Team

**Your situation:**
- Data analysts who live in SQL
- No need for self-service business user tool
- Want ad-hoc exploration and basic dashboards

**Recommendation: Snowsight**

Why:
- Free (included with Snowflake)
- Native performance (queries run in warehouse)
- SQL worksheets + basic charts
- Snowflake Notebooks for Python work

**Alternative: Redash or Mode**
- If you need more collaboration features
- Redash free (self-hosted), Mode expensive ($100+/user)

## When to Start Simple and Upgrade Later

Most teams should **start simple** and upgrade when they hit real limitations, not hypothetical ones.

### Start Simple If:

- Your team is small (< 10 people)
- You're early in your data journey
- Budget is limited
- You don't have complex visualisation needs yet
- You're not sure what your long-term BI needs are

**Start with:** Snowsight (free) or Lightdash self-hosted (~$30/month)

### Upgrade to Enterprise When:

- You have 50+ business users consuming dashboards
- You need embedded analytics in external applications
- Compliance requires detailed audit logs and governance
- You hit visualisation limitations (need custom D3.js charts, advanced maps)
- You have budget ($5000+/month) and executive support

**Upgrade to:** Tableau, Looker, or Power BI (if Microsoft-heavy)

!!! success "Migration Path"
    Snowsight → Lightdash → Tableau is a common evolution. You learn what your organisation needs at each stage before committing to expensive enterprise tools.

## Red Flags and Anti-Patterns

Avoid these common mistakes when choosing a BI tool:

**❌ Choosing based on "what if" scenarios**
- "What if we need geospatial analysis in 3 years?"
- Choose for your needs today, not hypothetical future requirements

**❌ Ignoring total cost of ownership**
- Self-hosted tools are "free" but require engineering time
- Factor in setup (8-16 hours), maintenance (4-8 hours/month), upgrades

**❌ Not involving actual users in evaluation**
- If business users will consume dashboards, get their feedback
- Don't choose a technical tool (Mode, Redash) for non-technical users

**❌ Choosing enterprise tools too early**
- Tableau for a 3-person startup is overkill
- Start simple and upgrade when you hit limits

**❌ Ignoring dbt integration if you have dbt**
- If you've built dbt models, use a dbt-native tool
- Otherwise you're maintaining metrics in two places (dbt + BI tool)

**❌ Over-indexing on visualisation library**
- Most teams use line charts, bar charts, and tables 95% of the time
- Don't pay for Tableau if you only need basic charts

## Evaluation Checklist

When evaluating finalists, test these scenarios:

- [ ] Connect to your Snowflake warehouse
- [ ] Query one of your dbt models (if applicable)
- [ ] Create a simple dashboard (line chart, bar chart, filter)
- [ ] Set up user access controls (roles, permissions)
- [ ] Schedule a dashboard to email stakeholders
- [ ] (Self-hosted) Deploy to your infrastructure and test upgrades
- [ ] (dbt-native) Add a metric to dbt YAML and see it appear in BI tool
- [ ] Check documentation and community support quality
- [ ] Calculate total cost for your expected user count
- [ ] Test mobile experience (if relevant)

## Summary

You've learned how to choose a BI tool:

- [x] **Five key questions** narrow your options significantly (dbt?, budget?, self-host?, users?, viz complexity?)
- [x] **Decision framework** guides you from questions to specific tool recommendations
- [x] **Evaluation criteria** cover cost, integrations, governance, developer experience, learning curve
- [x] **Common scenarios** show realistic recommendations for startups, mid-size companies, enterprises
- [x] **Start simple** with Snowsight or Lightdash, upgrade to enterprise tools when you hit real limitations

The right tool depends on your specific situation. There's no universally "best" BI tool — only the best fit for you right now.

## What's Next

Now that you understand how to evaluate tools, learn about the trade-offs between SaaS and self-hosted deployments.

Continue to [Deployment Options](3-deployment-options.md) →
