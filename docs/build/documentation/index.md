# Documentation

You have built infrastructure, pipelines, transformations, and analytics across multiple repositories. This section covers how to create a **unified documentation site** that pulls content from all your repositories into a single, searchable site - while keeping each repository's documentation alongside its code.

On this page, you will:

- [x] Understand why documentation belongs alongside code
- [x] Learn the multi-repo documentation architecture
- [x] See how `mkdocs-multirepo-plugin` unifies docs from separate repositories
- [x] Understand what belongs on the docs site versus in-place in code

## Overview

Your data stack spans three repositories, each with its own conventions, patterns, and operational procedures. New team members need to understand how each repository works. On-call engineers need runbooks when things go wrong. And as your stack evolves, documentation must evolve with it.

The challenge is keeping documentation close to the code it describes - so it stays current - while also having a single place to search across everything.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DOCUMENTATION ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  terraform/              data-pipelines/        dbt-transform/          │
│  ┌──────────────┐       ┌──────────────┐       ┌──────────────┐        │
│  │ terraform/   │       │ sources/     │       │ models/      │        │
│  │ modules/     │       │ pipelines/   │       │ tests/       │        │
│  │ docs/ ◄──────┤       │ docs/ ◄──────┤       │ docs/ ◄──────┤        │
│  │  ├ index.md  │       │  ├ index.md  │       │  ├ index.md  │        │
│  │  ├ getting-  │       │  ├ getting-  │       │  ├ getting-  │        │
│  │  │ started   │       │  │ started   │       │  │ started   │        │
│  │  ├ conven-   │       │  ├ conven-   │       │  ├ conven-   │        │
│  │  │ tions     │       │  │ tions     │       │  │ tions     │        │
│  │  └ runbooks/ │       │  └ runbooks/ │       │  └ runbooks/ │        │
│  └──────────────┘       └──────────────┘       └──────────────┘        │
│         │                       │                       │               │
│         └───────────┬───────────┘───────────────────────┘               │
│                     ▼                                                   │
│           technical-documentation/                                      │
│           ┌─────────────────────┐                                      │
│           │ mkdocs.yml          │  multirepo plugin imports             │
│           │ + multirepo plugin  │  docs from each repo at              │
│           │ + Material theme    │  build time                          │
│           └────────┬────────────┘                                      │
│                    ▼                                                    │
│           ┌─────────────────────┐                                      │
│           │  GitHub Pages       │  Single unified site                 │
│           │  ┌───────────────┐  │  with cross-repo search             │
│           │  │ Unified Site  │  │                                      │
│           │  │ + Search      │  │  ┌──────────────┐                   │
│           │  │ + All repos   │──┼─▶│ dbt Docs     │ (linked,          │
│           │  └───────────────┘  │  │ (CloudFront  │  not embedded)    │
│           └─────────────────────┘  │  or dbt Cloud)│                   │
│                                    └──────────────┘                    │
└─────────────────────────────────────────────────────────────────────────┘
```

## Why Documentation Alongside Code

Keeping documentation in the same repository as the code it describes has several advantages:

### Claude Can Keep Docs Up-to-Date

When Claude modifies a Terraform module, it can update the corresponding documentation in the same pull request. The documentation and code change together, in one review.

### Code Review Applies to Documentation

Documentation changes go through the same pull request process as code. Reviewers see both the code change and its documentation impact, catching inconsistencies before they reach production.

### Version-Controlled, Single Source of Truth

Documentation is versioned alongside the code. You can see what the documentation said at any point in history, and `git blame` shows who changed what and when.

### New Team Members Find Docs Where They Expect Them

When someone clones a repository, the documentation is right there. They do not need to know about a separate wiki, Confluence space, or shared drive.

## The Multi-Repo Challenge

Your data stack uses three repositories:

| Repository | Purpose | Documentation Covers |
|------------|---------|---------------------|
| **terraform** | Infrastructure as code | Architecture, module patterns, conventions, runbooks |
| **data-pipelines** | Ingestion and orchestration | Pipeline architecture, getting started, conventions, runbooks |
| **dbt-transform** | Data transformation | Model layers, testing strategy, conventions, runbooks |

Each repository has its own `docs/` directory with documentation specific to that codebase. But engineers need to search across all three - for example, finding all runbooks related to Snowflake, regardless of which repository they live in.

### The Solution: `mkdocs-multirepo-plugin`

The [`mkdocs-multirepo-plugin`](https://github.com/jdoiro3/mkdocs-multirepo-plugin) solves this by importing documentation from multiple repositories at build time. A central `technical-documentation` repository owns the unified site configuration, theme, and shared content. At build time, the plugin clones each source repository and pulls in its `docs/` directory, producing a single site with unified search.

Each repository remains independent - developers edit documentation alongside their code, review it in pull requests, and preview it locally. The central repository simply aggregates everything into one site.

## What Belongs Where

Not all documentation belongs on the docs site. The distinction is between **repo-level** documentation (how to use the repository as a whole) and **resource-level** documentation (details about specific modules, pipelines, or models).

### On the Docs Site (Repo-Level)

- **Architecture** - How the repository is structured, design decisions, provider patterns
- **Getting started** - How to set up locally, run commands, understand the workflow
- **Conventions** - Naming patterns, configuration approach, coding standards
- **Runbooks** - Operational procedures for common tasks and incident response

### In-Place in the Code (Resource-Level)

- **Terraform** - Module READMEs, variable descriptions, inline comments
- **Data pipelines** - Source docstrings, pipeline README files in each directory
- **dbt** - Model descriptions in YAML files, doc blocks, auto-generated dbt docs site

!!! info "dbt Docs"
    The auto-generated dbt docs site (with its lineage graph and model explorer) remains separately hosted - on CloudFront for dbt Core, or built into dbt Cloud. The unified docs site links to it rather than embedding it, since the dbt docs UI is specialised for exploring model relationships.

## What You'll Build

This section walks through the complete setup:

| Page | What You'll Learn |
|------|------------------|
| [Documentation Repo Setup](1-documentation-repo-setup.md) | Create the central `technical-documentation` repo with MkDocs Material and the multirepo plugin |
| [Repository Integration](2-repository-integration.md) | Add `docs/` directories to each repo with initial content |
| [Writing Documentation](3-writing-documentation.md) | What to document and how to structure repo-level docs |
| [Writing Runbooks](4-writing-runbooks.md) | Runbook structure, examples, and maintenance |
| [CI/CD Pipeline](5-ci-cd-pipeline.md) | Reusable CI for validation, CD for GitHub Pages deployment |
| [Claude Documentation Skill](6-claude-documentation-skill.md) | Shared Claude skill for consistent documentation across repos |
| [Finishing Up](7-finishing-up.md) | Verification, architecture summary, and next steps |

## Prerequisites

Before starting this section, ensure you have completed:

- [x] [GitHub Setup](../../getting-started/initial-setup/1-github-setup.md) - GitHub organisation and repositories
- [x] [Terraform Setup](../../getting-started/terraform-setup/index.md) - At least the basic Terraform configuration
- [x] [CI/CD Deployment](../../getting-started/terraform-setup/4-terraform-deployment.md) - GitHub Actions fundamentals

## Cost Considerations

| Tool | Cost | Notes |
|------|------|-------|
| **MkDocs** | Free | Open-source static site generator |
| **Material for MkDocs** | Free | Open-source theme (Insiders edition optional, from $15/month) |
| **mkdocs-multirepo-plugin** | Free | Open-source plugin |
| **GitHub Pages** | Free | For public repositories; included with GitHub Team for private |
| **GitHub Actions** | Free tier | 2,000 mins/month on free plan, 3,000 on Team |

The entire documentation stack costs nothing beyond what you are already paying for GitHub.

Continue to [Documentation Repo Setup](1-documentation-repo-setup.md) →
