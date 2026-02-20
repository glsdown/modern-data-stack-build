# BI Tool Landscape

On this page, you will:

- [x] Understand the major categories of BI tools available
- [x] Learn the strengths and limitations of each tool type
- [x] Compare enterprise, modern/dbt-native, and open source options
- [x] Identify which tools align with your budget and requirements

## Overview

The BI tool market is vast and fragmented. Tools range from $0 (open source) to $150+/user/month (enterprise), from no-code drag-and-drop interfaces to code-first analytics platforms, and from general-purpose reporting to dbt-native metric layers.

This page provides a comprehensive landscape overview to help you understand your options before choosing a tool. The next page covers how to evaluate and choose the right tool for your organisation.

## The Four Categories

BI tools fall into four broad categories:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         BI TOOL LANDSCAPE                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Enterprise/Traditional       Modern/dbt-Native                         │
│  ──────────────────────       ──────────────────                        │
│  • Tableau                    • Lightdash                               │
│  • Power BI                   • Omni                                    │
│  • Looker                     • Hex                                     │
│  • Qlik Sense                                                           │
│  • Mode                       Open Source/Community                     │
│                               ────────────────────                      │
│  Cloud Platform Native        • Metabase                                │
│  ──────────────────────       • Apache Superset                         │
│  • Snowsight                  • Redash                                  │
│  • Looker Studio              • Evidence                                │
│  • AWS QuickSight                                                       │
│  • Azure Power BI Service                                               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Enterprise/Traditional BI Tools

These are mature, feature-rich platforms built for large organisations. They offer extensive visualisation libraries, enterprise support, and governance features.

**Characteristics:**
- Decades of development and refinement
- Advanced visualisation capabilities
- Enterprise features (SSO, RBAC, audit logs, governance)
- Large user communities and extensive documentation
- Professional services and training available
- High per-user costs
- No native dbt integration (except Looker)

**Best for:** Large organisations with complex visualisation needs, existing contracts, or requirements for embedded analytics.

### Modern/dbt-Native BI Tools

These tools are built for the modern data stack. They understand dbt projects natively, reading metrics and models directly from your dbt repository.

**Characteristics:**
- Native dbt integration (read dbt YAML, understand lineage)
- Metrics defined in code (version-controlled, testable)
- Modern architecture (cloud-native, API-first)
- Smaller visualisation libraries than enterprise tools
- Lower cost than enterprise (but not always cheap)
- Designed for analytics engineers and data teams

**Best for:** Teams with dbt transformation layers who want metrics as code and version-controlled analytics definitions.

### Open Source/Community BI Tools

These are free, self-hosted tools with active open source communities. You pay only for infrastructure and maintenance.

**Characteristics:**
- Free to use (infrastructure costs only)
- Self-hosted (you control data and deployment)
- Active communities and plugin ecosystems
- No vendor lock-in
- Requires self-hosting expertise
- No native dbt integration (metrics defined in UI)
- Limited enterprise support (paid plans available)

**Best for:** Cost-conscious teams with technical expertise to self-host, or teams wanting full control over their BI infrastructure.

### Cloud Platform Native BI Tools

These are BI and dashboarding capabilities built into cloud data platforms. They're included with your data warehouse subscription.

**Characteristics:**
- Zero setup (already have access)
- No additional cost beyond warehouse compute
- Direct access to warehouse data (no ETL to BI tool)
- Basic visualisations and dashboarding
- SQL-based (no drag-and-drop for non-technical users)
- Limited compared to dedicated BI tools

**Best for:** Quick dashboards, SQL-proficient users, ad-hoc exploration, and teams wanting to avoid additional BI tool costs.

## Enterprise/Traditional Tools

### Tableau

**Overview:** Market-leading visualisation platform owned by Salesforce. Known for powerful, interactive dashboards and extensive visualisation library.

| Feature | Details |
|---------|---------|
| **Pricing** | $70-150/user/month (Creator, Explorer, Viewer tiers) |
| **Deployment** | Cloud (Tableau Online) or self-hosted (Tableau Server) |
| **dbt Integration** | No native integration; use Tableau Catalog or manual setup |
| **Best for** | Complex visualisations, embedded analytics, executive dashboards |
| **Limitations** | Expensive, steep learning curve, no metrics as code |

**Strengths:**
- Industry-leading visualisation capabilities
- Drag-and-drop interface (no SQL required for end users)
- Large community and extensive resources
- Strong performance with large datasets
- Mobile apps and offline capabilities

**Weaknesses:**
- High cost per user
- Metrics and business logic defined in UI (not version-controlled)
- No awareness of dbt models or lineage
- Complex licensing (Creator vs Explorer vs Viewer)

**When to choose Tableau:**
- Your organisation already has a Tableau contract
- You need advanced, interactive visualisations
- You're building embedded analytics for external customers
- Non-technical executives need self-service analytics

### Power BI

**Overview:** Microsoft's BI platform, tightly integrated with the Microsoft ecosystem (Excel, Azure, Office 365).

| Feature | Details |
|---------|---------|
| **Pricing** | $10-20/user/month (Pro, Premium tiers) |
| **Deployment** | Cloud (Power BI Service) or self-hosted (Power BI Report Server) |
| **dbt Integration** | No native integration; manual setup |
| **Best for** | Microsoft-heavy organisations, Excel power users |
| **Limitations** | Windows-centric, no metrics as code, Azure-focused |

**Strengths:**
- Low cost compared to other enterprise tools
- Familiar to Excel users (similar interface)
- Tight integration with Microsoft ecosystem
- Active community and frequent updates
- Good performance with Azure data sources

**Weaknesses:**
- Best experience requires Windows (limited Mac support)
- Metrics defined in Power BI Desktop (not version-controlled)
- No dbt awareness
- Licensing can be confusing (Pro vs Premium vs Embedded)

**When to choose Power BI:**
- Your organisation uses Microsoft 365 and Azure
- You have Excel power users who want self-service BI
- Budget is limited but you still need enterprise features
- You need tight integration with Microsoft products

### Looker

**Overview:** Google's BI platform with a unique LookML modelling layer. Looker uses code to define metrics, making it the most "dbt-like" of the enterprise tools.

| Feature | Details |
|---------|---------|
| **Pricing** | $60-150/user/month (Platform, Standard tiers) |
| **Deployment** | Cloud only (Google Cloud) |
| **dbt Integration** | Partial; can reference dbt models, but separate metric definitions |
| **Best for** | Teams wanting metrics as code, Google Cloud users |
| **Limitations** | Expensive, LookML learning curve, cloud-only |

**Strengths:**
- Metrics defined in LookML (version-controlled, testable)
- Git-based workflow for analytics definitions
- Embedded analytics capabilities
- Strong data governance and access controls
- API-first architecture

**Weaknesses:**
- Expensive (similar pricing to Tableau)
- LookML requires training (different from SQL)
- Metrics layer separate from dbt (maintain two metric definitions)
- Owned by Google (future uncertain post-acquisition)

**When to choose Looker:**
- You value metrics as code but need enterprise features
- Your data platform is Google Cloud
- You need embedded analytics with strong governance
- You're willing to invest in LookML training

### Mode

**Overview:** Analytics platform combining SQL notebooks, visualisations, and collaboration. Popular with data teams for ad-hoc analysis.

| Feature | Details |
|---------|---------|
| **Pricing** | $100+/user/month (Studio tier for full features) |
| **Deployment** | Cloud only |
| **dbt Integration** | Limited; can query dbt models, no native metric integration |
| **Best for** | Data teams doing SQL-based analysis and collaboration |
| **Limitations** | Expensive, SQL required, limited self-service for non-technical users |

**Strengths:**
- SQL notebooks for exploratory analysis
- Good for technical data teams
- Version control for queries and reports
- Python and R support

**Weaknesses:**
- Expensive for what it offers
- Requires SQL knowledge (not self-service for business users)
- Limited visualisation library compared to Tableau
- Not truly a self-service BI tool

**When to choose Mode:**
- Your analysts live in SQL and want collaborative notebooks
- You need ad-hoc analysis more than polished dashboards
- Budget allows for premium tooling

### Qlik Sense

**Overview:** Enterprise BI platform with an associative data model. Known for in-memory analytics and exploration.

| Feature | Details |
|---------|---------|
| **Pricing** | $30-70/user/month (Professional, Business tiers) |
| **Deployment** | Cloud or self-hosted |
| **dbt Integration** | No native integration |
| **Best for** | Organisations with existing Qlik contracts |
| **Limitations** | Smaller community than Tableau/Power BI, proprietary scripting |

**Strengths:**
- Associative data model (explore related data easily)
- Good performance with in-memory analytics
- Self-service interface for business users

**Weaknesses:**
- Smaller market share (less community support)
- Proprietary scripting language (Qlik script)
- No dbt integration

**When to choose Qlik:**
- Your organisation already uses Qlik
- You value the associative exploration model

## Modern/dbt-Native Tools

### Lightdash

**Overview:** Open source BI tool built specifically for dbt. Reads your dbt project directly from GitHub and understands metrics defined in dbt YAML.

| Feature | Details |
|---------|---------|
| **Pricing** | Free (self-hosted) or $2400/month (Cloud, unlimited users) |
| **Deployment** | Self-hosted (ECS, Docker, Kubernetes) or Cloud (managed) |
| **dbt Integration** | Native; reads dbt project, understands metrics and lineage |
| **Best for** | dbt-native teams, cost-conscious, infrastructure-as-code approach |
| **Limitations** | Basic visualisations, smaller community than enterprise tools |

**Strengths:**
- **Free and open source** for self-hosted (only pay infrastructure ~$30/month)
- Native dbt integration (metrics in dbt YAML, not UI)
- Version-controlled analytics (metrics live in Git)
- Understands dbt lineage, tests, and documentation
- Terraform deployment (infrastructure as code)
- Active community and regular updates

**Weaknesses:**
- Basic visualisation library (charts, tables, pivot tables)
- Smaller community than Tableau/Power BI
- Self-hosted requires infrastructure knowledge
- Cloud tier expensive ($2400/month flat rate)

**When to choose Lightdash:**
- You've built a dbt transformation layer
- You want metrics defined in code, version-controlled
- Budget is limited but you can self-host
- Your team values infrastructure as code
- You're comfortable with basic visualisations

!!! tip "Lightdash as the Learning Example"
    This documentation uses Lightdash for hands-on implementation because it's free, dbt-native, and can be deployed with Terraform. The patterns you learn (metrics as code, version-controlled dashboards) apply to other modern tools like Omni as well.

### Omni

**Overview:** Cloud-only BI tool built for dbt. Similar philosophy to Lightdash (metrics in code) but with a more polished interface and higher price.

| Feature | Details |
|---------|---------|
| **Pricing** | $20-50/user/month (3 user minimum = $60/month) |
| **Deployment** | Cloud only (managed SaaS) |
| **dbt Integration** | Native; reads dbt project, understands metrics |
| **Best for** | dbt-native teams wanting polished UX without self-hosting |
| **Limitations** | No self-hosted option, relatively expensive for small teams |

**Strengths:**
- Native dbt integration (metrics in dbt YAML)
- More polished interface than Lightdash
- Cloud-only (no infrastructure to manage)
- Good performance and user experience
- Supports dbt Cloud semantic layer

**Weaknesses:**
- No self-hosted option (must use cloud)
- Expensive for small teams (3 user minimum = $60/month)
- Smaller community and ecosystem than Lightdash
- Newer product (less mature)

**When to choose Omni:**
- You want dbt-native BI without self-hosting
- Budget allows $60-150/month
- You value polished UX over cost savings
- You're using dbt Cloud semantic layer

### Hex

**Overview:** Collaborative notebook platform with SQL, Python, and R support. Combines notebooks, dashboards, and app building.

| Feature | Details |
|---------|---------|
| **Pricing** | $70+/user/month (Team plan) |
| **Deployment** | Cloud only |
| **dbt Integration** | Can query dbt models; no native metric integration |
| **Best for** | Data science teams, Python/SQL collaboration |
| **Limitations** | Expensive, not a traditional BI tool, requires coding |

**Strengths:**
- SQL, Python, and R in collaborative notebooks
- Build interactive apps from notebooks
- Good for data science workflows
- Version control for notebooks

**Weaknesses:**
- Expensive ($70/user/month minimum)
- Not a self-service BI tool for non-technical users
- Requires coding (Python/SQL)
- Limited dbt integration

**When to choose Hex:**
- Your team does heavy Python data science work
- You want notebooks + dashboards in one platform
- Budget allows premium tooling

## Open Source/Community Tools

### Metabase

**Overview:** Popular open source BI tool with a simple, user-friendly interface. Can be self-hosted or used as a cloud service.

| Feature | Details |
|---------|---------|
| **Pricing** | Free (self-hosted) or $85/user/month (Cloud Pro) |
| **Deployment** | Self-hosted (Docker, ECS, Kubernetes) or Cloud |
| **dbt Integration** | None; metrics defined in Metabase UI |
| **Best for** | General-purpose BI, teams without dbt, cost-conscious self-hosting |
| **Limitations** | No dbt integration, metrics in UI (not code) |

**Strengths:**
- **Free and open source** (self-hosted)
- User-friendly interface (easy for non-technical users)
- Active community and extensive documentation
- Mature product (been around since 2015)
- Good mobile experience

**Weaknesses:**
- No dbt integration
- Metrics defined in UI (not version-controlled)
- No awareness of dbt lineage or tests
- Self-hosted infrastructure costs (~$20-30/month)

**When to choose Metabase:**
- You don't have a dbt transformation layer
- You want self-service BI for non-technical users
- Budget is limited and you can self-host
- You prefer mature, stable open source tools

### Apache Superset

**Overview:** Open source data exploration and visualisation platform originally built by Airbnb, now an Apache project.

| Feature | Details |
|---------|---------|
| **Pricing** | Free (self-hosted); paid cloud options via Preset |
| **Deployment** | Self-hosted (Docker, Kubernetes) or Preset Cloud |
| **dbt Integration** | Limited; no native integration |
| **Best for** | Teams wanting advanced visualisations, Apache ecosystem users |
| **Limitations** | Steeper learning curve, requires more infrastructure knowledge |

**Strengths:**
- Free and open source
- More advanced visualisations than Metabase
- SQL Lab for ad-hoc queries
- Active Apache community

**Weaknesses:**
- More complex to set up and maintain than Metabase
- No dbt integration
- Requires Python/data engineering knowledge

**When to choose Superset:**
- You need more visualisation options than Metabase
- Your team is comfortable with Apache projects
- You want free, open source BI

### Redash

**Overview:** Open source BI tool focused on SQL queries and collaboration. Simpler than Superset, more technical than Metabase.

| Feature | Details |
|---------|---------|
| **Pricing** | Free (self-hosted) |
| **Deployment** | Self-hosted (Docker, ECS) |
| **dbt Integration** | None |
| **Best for** | SQL-based teams, query sharing and collaboration |
| **Limitations** | Requires SQL, limited visualisations, not actively maintained |

**Strengths:**
- Free and open source
- SQL-first (good for technical teams)
- Query scheduling and alerts
- Easy to self-host

**Weaknesses:**
- Project less active (acquired by Databricks, future uncertain)
- Limited visualisation library
- Requires SQL knowledge

**When to choose Redash:**
- Your team is SQL-proficient and wants query collaboration
- Budget is extremely limited
- You're aware of the project's maintenance status

### Evidence

**Overview:** Code-first BI tool where dashboards are markdown files with SQL and charts defined in code. Very new but interesting approach.

| Feature | Details |
|---------|---------|
| **Pricing** | Free (self-hosted) or paid cloud (pricing not public) |
| **Deployment** | Self-hosted or Cloud |
| **dbt Integration** | Limited; can query dbt models |
| **Best for** | Teams wanting dashboards as code |
| **Limitations** | Very new, small community, limited features |

**Strengths:**
- Dashboards defined in markdown (version-controlled)
- Code-first approach (appeals to engineers)
- Free for self-hosted

**Weaknesses:**
- Very new product (2021)
- Small community
- Limited compared to mature tools

**When to choose Evidence:**
- You want dashboards fully in code (Git, CI/CD)
- You're comfortable with bleeding-edge tools

## Cloud Platform Native Tools

### Snowsight (Snowflake)

**Overview:** Snowflake's built-in web interface with dashboarding and visualisation capabilities. Included with your Snowflake account.

| Feature | Details |
|---------|---------|
| **Pricing** | $0 (included with Snowflake; uses warehouse compute) |
| **Deployment** | Cloud (part of Snowflake) |
| **dbt Integration** | None; query dbt models directly with SQL |
| **Best for** | Quick dashboards, SQL exploration, analysts with Snowflake access |
| **Limitations** | Basic visualisations, SQL required, no self-service for non-technical users |

**Strengths:**
- **Free** (no additional cost beyond Snowflake compute)
- Zero setup (already have access)
- Native performance (queries run directly in Snowflake)
- SQL worksheets and basic charts
- Snowflake Notebooks (Python notebooks built-in)

**Weaknesses:**
- Basic visualisations only
- Requires SQL knowledge
- No drag-and-drop for business users
- Limited collaboration features

**When to choose Snowsight:**
- You want quick dashboards without additional BI tool costs
- Your users are SQL-proficient
- You need ad-hoc exploration, not polished executive dashboards

!!! tip "Snowsight as Quick Win"
    This documentation covers Snowsight first because it's free and immediate. Build quick dashboards here while evaluating whether you need a dedicated BI tool.

### Looker Studio (Google)

**Overview:** Google's free BI tool (formerly Data Studio). Connects to various data sources and provides drag-and-drop dashboards.

| Feature | Details |
|---------|---------|
| **Pricing** | Free |
| **Deployment** | Cloud (Google) |
| **dbt Integration** | None; connect to BigQuery or other sources |
| **Best for** | Google Workspace users, free basic dashboards |
| **Limitations** | Basic features, best with Google data sources, performance issues with large data |

**Strengths:**
- Completely free
- Easy to use (drag-and-drop)
- Integrates well with Google products (Sheets, Analytics, BigQuery)

**Weaknesses:**
- Basic functionality
- Performance issues with non-Google data sources
- Limited enterprise features

**When to choose Looker Studio:**
- Your data is in Google BigQuery
- Budget is $0 and you need basic dashboards
- Your organisation uses Google Workspace

### AWS QuickSight

**Overview:** AWS's cloud-native BI service. Integrates with AWS data sources and provides ML-powered insights.

| Feature | Details |
|---------|---------|
| **Pricing** | $9-24/user/month |
| **Deployment** | Cloud (AWS) |
| **dbt Integration** | None; connect to Redshift, Athena, S3, etc. |
| **Best for** | AWS-heavy organisations, embedded analytics |
| **Limitations** | Best with AWS sources, smaller ecosystem than Tableau/Power BI |

**Strengths:**
- Low cost for an enterprise-class tool
- Tight AWS integration
- ML-powered insights (anomaly detection, forecasting)
- Pay-per-session pricing option

**Weaknesses:**
- Smaller community than Tableau/Power BI
- Best experience requires AWS data sources
- Limited compared to dedicated BI platforms

**When to choose QuickSight:**
- Your data platform is entirely AWS
- You need low-cost BI with some enterprise features
- You want embedded analytics in AWS applications

## Comparison Summary

### By Cost (Monthly, Self-Hosted Where Applicable)

| Tool | Cost Range | Notes |
|------|-----------|-------|
| **Snowsight** | $0 | Free with Snowflake (uses warehouse compute) |
| **Looker Studio** | $0 | Completely free |
| **Lightdash (self-hosted)** | ~$30 | Infrastructure only (ECS + RDS) |
| **Metabase (self-hosted)** | ~$20-30 | Infrastructure only |
| **Superset (self-hosted)** | ~$25-40 | Infrastructure only |
| **Redash (self-hosted)** | ~$20-30 | Infrastructure only |
| **Evidence (self-hosted)** | ~$15-25 | Infrastructure only |
| **QuickSight** | $9-24/user | AWS cloud service |
| **Power BI** | $10-20/user | Microsoft cloud service |
| **Omni** | $60+/month | 3 user minimum |
| **Looker** | $60-150/user | Google cloud only |
| **Hex** | $70+/user | Team plan minimum |
| **Tableau** | $70-150/user | Creator/Explorer/Viewer tiers |
| **Metabase Cloud** | $85/user | Managed service |
| **Mode** | $100+/user | Studio tier for full features |
| **Lightdash Cloud** | $2400/month | Flat rate, unlimited users |

### By dbt Integration

| Native dbt Integration | Can Query dbt Models | No Integration |
|------------------------|---------------------|----------------|
| **Lightdash** | **Tableau** | **Metabase** |
| **Omni** | **Power BI** | **Superset** |
| | **Looker** (separate metrics) | **Redash** |
| | **Mode** | **QuickSight** |
| | **Hex** | **Looker Studio** |
| | **Snowsight** | **Qlik Sense** |
| | **Evidence** | |

### By Deployment Model

| Cloud Only | Self-Hosted + Cloud | Self-Hosted Only |
|-----------|---------------------|------------------|
| **Looker** | **Lightdash** | (None - all have cloud options) |
| **Omni** | **Tableau** | |
| **Hex** | **Power BI** | |
| **Mode** | **Metabase** | |
| **Looker Studio** | **Superset** (via Preset) | |
| **QuickSight** | **Evidence** | |
| **Snowsight** | | |

## What's Missing From This List?

You might notice some tools not covered:

- **Sisense** - Enterprise BI, similar to Tableau/Qlik (less market share)
- **Domo** - Cloud BI platform (expensive, niche)
- **Sigma** - Spreadsheet-like BI interface (newer, growing)
- **ThoughtSpot** - Search-driven BI (enterprise, expensive)
- **GoodData** - Embedded analytics platform (niche)

These tools exist but are less common in the modern data stack. The tools covered above represent the major players and realistic options for most teams.

## Summary

You've surveyed the BI tool landscape:

- [x] **Enterprise/Traditional** - Tableau, Power BI, Looker (powerful, expensive, no dbt integration)
- [x] **Modern/dbt-Native** - Lightdash, Omni, Hex (metrics as code, dbt-aware)
- [x] **Open Source/Community** - Metabase, Superset, Redash (free, self-hosted, no dbt integration)
- [x] **Cloud Platform Native** - Snowsight, Looker Studio, QuickSight (free or low-cost, basic features)

The right choice depends on your budget, technical capabilities, and whether you've built a dbt transformation layer.

## What's Next

Now that you understand the landscape, learn how to evaluate and choose the right tool for your organisation.

Continue to [Choosing a BI Tool](2-choosing-a-bi-tool.md) →
