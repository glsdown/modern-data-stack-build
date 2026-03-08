# dbt Core vs dbt Cloud

On this page, you will:

- [x] Compare dbt Core and dbt Cloud across cost, features, and operational overhead
- [x] Understand how each option handles CI/CD, deferral, and documentation
- [x] Choose the right deployment for your team

## Overview

dbt is available in two forms:

1. **dbt Core** — the open-source command-line tool, free to use, self-managed
2. **dbt Cloud** — a managed platform built on top of dbt Core, with an IDE, scheduler, and hosted docs site

Both run the same underlying transformation engine. The difference is what comes bundled with the platform vs what you build yourself.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          DEPLOYMENT OPTIONS                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Option 1: dbt Core                    Option 2: dbt Cloud                  │
│  ──────────────────                    ───────────────────                  │
│                                                                             │
│  ┌─────────────────────────┐             ┌───────────────────────┐          │
│  │  You manage:            │             │  dbt Labs manages:    │          │
│  │  • CI/CD (Actions)      │             │  • IDE                │          │
│  │  • Scheduling (Prefect) │             │  • Scheduler          │          │
│  │  • Docs hosting (S3)    │             │  • Docs site          │          │
│  │  • State artifacts      │             │  • State management   │          │
│  │  • deferral setup       │             │  • Deferral           │          │
│  └─────────────────────────┘             └───────────────────────┘          │
│                                                                             │
│  Cost: Free                            Cost: Free (1 seat) / $100/seat/mo   │
│  Setup: GitHub Actions + S3            Setup: Account + environments        │
│  Maintenance: Ongoing                  Maintenance: Minimal                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## dbt Core

### What It Is

dbt Core is the open-source Python package you install locally and in CI/CD. You write models in your editor, run them with the `dbt` CLI, and manage everything yourself.

```sh
uv add dbt-snowflake

dbt run           # run all models
dbt test          # run all tests
dbt docs generate # generate docs
dbt docs serve    # serve docs locally
```

### Cost

dbt Core is free and open source. The only costs are infrastructure you choose to run:

| Component | Purpose | Monthly Cost |
|-----------|---------|-------------|
| S3 bucket (artifacts) | Store `manifest.json` for state-based operations | < $0.01 |
| CloudFront + S3 (docs) | Host the generated docs site | ~$1-5 |
| GitHub Actions minutes | CI/CD runs | Included in most tiers |
| **Total additional** | | **~$1-5/month** |

### Developer Workflow

Developers work locally:

```sh
# Clone the dbt-transform repo
git clone git@github.com:your-org/dbt-transform.git
cd dbt-transform

# Install dependencies and activate venv via direnv
uv add dbt-snowflake
dbt deps

# Run models against dev environment (dev is the default target)
dbt run

# Run with deferral to production (see page 9)
dbt run --defer --state .artifacts/
```

### CI/CD

CI/CD uses GitHub Actions:

- **Pull requests**: Run [slim CI](https://docs.getdbt.com/docs/deploy/continuous-integration) (`dbt build --select state:modified+`) using production artifacts to defer unchanged upstream models
- **Merge to main**: Full production run, upload new `manifest.json` to S3, deploy updated docs site

### Production Deferral

dbt Core v1.6+ supports the `--defer` flag natively. When you run with `--defer --state .artifacts/`:

- Models that have not changed are resolved from the **production** database instead of being rebuilt in dev
- Only changed models (and their downstream dependencies) are built in the dev environment
- This means a developer can run `dbt build --select state:modified+ --defer --state .artifacts/` and test their changes against production data without building the entire project

The `manifest.json` (the production state file) is stored in S3 and fetched before each dev/CI run:

```sh
aws s3 cp --profile data-engineer s3://your-bucket/dbt-artifacts/manifest.json .artifacts/manifest.json
dbt build --select state:modified+ --defer --state .artifacts/
```

### Docs Hosting

After a production run, `dbt docs generate` produces static HTML. A GitHub Actions step deploys this to S3 + CloudFront, giving analysts a browsable docs site at a URL you control.

```sh
dbt docs generate
aws s3 sync --profile data-engineer ./target/ s3://your-bucket/dbt-docs/ --delete
```

### Advantages

- **Free**: No subscription cost
- **Full control**: CI/CD, scheduling, and docs exactly as you want them
- **No vendor dependency**: Works the same regardless of what platform you're on
- **Prefect integration**: Scheduling stays in Prefect alongside ingestion flows

### Disadvantages

- **More to build**: CI/CD, docs hosting, and deferral setup require initial investment
- **No browser IDE**: Developers need local setup (Python, git, dbt CLI)
- **No semantic layer**: MetricFlow / semantic layer is a dbt Cloud feature

---

## dbt Cloud

### What It Is

dbt Cloud is a managed platform that wraps dbt Core with:

- A browser-based IDE with SQL autocompletion and lineage graph
- A built-in scheduler for running jobs on a schedule or via API
- A hosted docs site generated automatically after each job run
- Native environment management (development → staging → production)
- Built-in state management and deferral (no S3 artifacts to manage)

### Pricing

| Tier | Monthly Cost | Seats | Key Features |
|------|-------------|-------|-------------|
| **Developer** | Free | 1 | 1 project, browser IDE, 1 job |
| **Team** | $100/seat/month | 2+ | Multiple projects, SSO, audit logs, semantic layer |
| **Enterprise** | Custom | Custom | PrivateLink, custom SLAs, advanced RBAC |

[getdbt.com/pricing](https://www.getdbt.com/pricing/)

!!! tip "Start with Developer (Free)"
    If you're a solo analytics engineer, the Developer tier covers everything you need: browser IDE, a production environment, and scheduled jobs. Upgrade to Team when you need multi-seat collaboration or the semantic layer.

### Developer Workflow

Developers work in the dbt Cloud IDE or locally with the [dbt Cloud CLI](https://docs.getdbt.com/docs/cloud/cloud-cli-installation):

```sh
# Install dbt Cloud CLI via Homebrew
brew tap dbt-labs/dbt-cli && brew install dbt

# Configure: download dbt_cloud.yml from dbt Cloud UI
# Account Settings → CLI → Download CLI configuration file → save to ~/.dbt/dbt_cloud.yml

# Run models — automatically defers to production
dbt run --select stg_dlt__exchange_rates+
```

The dbt Cloud CLI handles deferral automatically. There is no need to fetch `manifest.json` from S3 — dbt Cloud manages this for you.

### Environments

dbt Cloud uses environments to separate development from production:

| Environment | Purpose | Who uses it |
|-------------|---------|------------|
| **Development** | Each developer's personal sandbox | Individual developers |
| **Production** | Source of truth; runs on schedule | CI/CD jobs |

Each environment is configured with:
- A Snowflake target database (`ANALYTICS_DEV` for dev, `ANALYTICS` for prod) and a schema prefix (`ANALYTICS_DEV` for dev — the custom macro uses this to produce schemas like `ANALYTICS_DEV_STAGING`)
- A service account (SVC_DBT credentials from AWS Secrets Manager)
- A dbt version

### Jobs

A **job** is a configured set of dbt commands that runs on a schedule or via API trigger:

| Job | Commands | Schedule |
|-----|---------|---------|
| **Daily production run** | `dbt build` | Daily at 08:00 UTC (or triggered by Prefect) |
| **Weekly full refresh** | `dbt build --full-refresh` | Sundays at 02:00 UTC |
| **Slim CI** | `dbt build --select state:modified+` | On pull request |

### Triggering from Prefect

Rather than scheduling jobs in dbt Cloud, you can trigger them via the dbt Cloud API from Prefect, keeping all orchestration in one place.

Alternatively, let dbt Cloud handle the schedule independently — useful if analytics engineers want control over when transformations run.

### Docs and Lineage

After each job run, dbt Cloud automatically:

- Generates docs from model descriptions and `sources.yml`
- Hosts the docs site at a URL you share with your team
- Shows the full DAG in the lineage graph
- Tracks model performance over time (run duration, row counts)

### Semantic Layer (Team/Enterprise)

The dbt Cloud semantic layer (powered by MetricFlow) lets you define metrics in dbt and query them from BI tools that support the protocol. Supported tools include Tableau, Power BI, Omni, Mode, and others.

### Advantages

- **Minimal setup**: No S3 artifacts, no docs hosting, no deferral configuration
- **Browser IDE**: Analytics engineers can work without local Python setup
- **Automatic deferral**: dbt Cloud CLI handles dev-to-prod deferral natively
- **Semantic layer**: Available on Team/Enterprise for governed metric definitions
- **Built-in monitoring**: Job history, model timing, and error details in the UI

### Disadvantages

- **Cost**: $100/seat/month for Team tier
- **Scheduler duplication**: If you use Prefect for ingestion, you now have two schedulers
- **Vendor dependency**: Relies on dbt Labs' platform availability

---

## Decision Matrix

| Factor | dbt Core | dbt Cloud |
|--------|:--------:|:---------:|
| Cost | Free | Free (1 seat) / $100+/seat/mo |
| Browser IDE | No | Yes |
| Scheduling | Prefect | dbt Cloud or Prefect |
| Docs hosting | Self-managed (S3) | Managed |
| Deferral | Manual (S3 artifacts) | Automatic |
| Semantic layer | No | Team/Enterprise |
| CI/CD setup effort | Moderate | Minimal |
| Vendor dependency | None | dbt Labs |
| Best for | Code-first teams with Prefect | Teams wanting managed platform |

### Recommendation

**Use dbt Core** if:

- You already have Prefect and want scheduling in one place
- Your team is comfortable with GitHub Actions and the CLI
- You want zero additional subscription cost
- You don't need the semantic layer or browser IDE

**Use dbt Cloud** if:

- Analytics engineers need a browser IDE (no local Python setup)
- You want automatic deferral without S3 artifact management
- You need the semantic layer (and use a supported BI tool)
- You want built-in docs hosting and model performance monitoring

!!! tip "Migration Path"
    You can start with dbt Core and migrate to dbt Cloud later. The dbt project files (`models/`, `dbt_project.yml`, `packages.yml`) are identical — only the CI/CD configuration and profile changes. The Snowflake infrastructure and model code move across unchanged.

## This Guide's Approach

Both options are covered in full:

- [dbt Core Deployment](9-dbt-core-deployment.md) covers GitHub Actions CI/CD, S3 state artifacts, and docs hosting
- [dbt Cloud Setup](10-dbt-cloud-setup.md) covers environments, jobs, and the browser IDE

The model pages (staging, intermediate, marts, testing) apply equally to both options. The Prefect orchestration page covers both integration patterns.

## Summary

You've compared the deployment options:

- [x] dbt Core: free, self-managed CI/CD and docs, deferral via S3, scheduling in Prefect
- [x] dbt Cloud: managed platform, browser IDE, automatic deferral, semantic layer on paid tier
- [x] Chosen your deployment approach

## What's Next

Set up the `dbt-transform` repository with the project structure and required packages.

Continue to [Project Setup](3-project-setup.md) →
