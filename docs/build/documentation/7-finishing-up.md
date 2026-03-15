# Finishing Up

On this page, you will:

- [x] Verify the complete documentation setup end-to-end
- [x] Review the architecture you have built
- [x] Understand costs and maintenance requirements
- [x] Learn how to extend the setup to additional repositories

## Overview

You have built a unified documentation site that aggregates content from all your platform repositories into a single, searchable site. This final page verifies everything works end-to-end and provides guidance for ongoing maintenance.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COMPLETE DOCUMENTATION SETUP                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Source Repositories              Central Repo          Live Site        │
│  ────────────────────             ────────────          ─────────       │
│                                                                         │
│  terraform/docs/                                                        │
│  ├── index.md            ──┐                                            │
│  ├── getting-started.md    │                                            │
│  ├── conventions.md        │   technical-documentation/                 │
│  └── runbooks/             ├──▶ mkdocs.yml              GitHub Pages   │
│                            │    + multirepo plugin  ──▶ Unified site   │
│  data-pipelines/docs/      │    + Material theme        + Search       │
│  ├── index.md            ──┤    + CI/CD workflows       + All repos    │
│  ├── getting-started.md    │                                            │
│  ├── conventions.md        │                            ┌────────────┐ │
│  └── runbooks/             │                            │ dbt Docs   │ │
│                            │                            │ (linked)   │ │
│  dbt-transform/docs/       │                            └────────────┘ │
│  ├── index.md            ──┘                                            │
│  ├── getting-started.md                                                 │
│  ├── conventions.md                                                     │
│  └── runbooks/                                                          │
│                                                                         │
│  CI: Reusable workflow validates each repo's docs on PRs                │
│  CD: Deploys unified site on merge (any repo triggers rebuild)          │
│  Skill: Shared Claude skill writes docs in correct repo                 │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Verification Checklist

### Central Repository

- [ ] `technical-documentation` repository created with MkDocs configuration
- [ ] `mkdocs.yml` has `!import` directives for all source repositories
- [ ] Material theme configured (Raleway font, blue grey / light blue palette, light/dark toggle)
- [ ] `mkdocs-multirepo-plugin` installed and configured
- [ ] Landing page (`docs/index.md`) links to each repository section and dbt docs
- [ ] `data-stack.code-workspace` file includes all repositories

Verify the unified site builds:

```sh
cd ~/src/technical-documentation
uv run mkdocs build --strict
```

### Source Repositories

For each repository (terraform, data-pipelines, dbt-transform):

- [ ] `docs/` directory exists with initial pages (index, getting started, conventions)
- [ ] `docs/runbooks/` directory exists with at least an index page
- [ ] `docs/mkdocs.yml` configured for independent local preview
- [ ] MkDocs dependencies added (`uv add --dev mkdocs mkdocs-material mkdocs-glightbox`)

Verify each repository builds independently:

```sh
cd ~/src/terraform/docs && uv run mkdocs build --strict
cd ~/src/data-pipelines/docs && uv run mkdocs build --strict
cd ~/src/dbt-transform/docs && uv run mkdocs build --strict
```

### CI/CD

- [ ] Reusable CI workflow in `technical-documentation/.github/workflows/docs-ci-reusable.yml`
- [ ] Caller workflow in each source repository: `.github/workflows/docs-ci.yml`
- [ ] Deployment workflow in `technical-documentation/.github/workflows/deploy-docs.yml`
- [ ] `DOCS_GITHUB_TOKEN` secret configured for multirepo plugin access
- [ ] `DOCS_DISPATCH_TOKEN` secret configured for cross-repo triggers
- [ ] GitHub Pages enabled on the `technical-documentation` repository
- [ ] Dispatch trigger added to each source repository's CI workflow

Test the pipeline:

1. Make a documentation change in a source repository
2. Open a pull request - verify CI validates the docs build
3. Merge the pull request - verify the unified site rebuilds and deploys

### Claude Skill

- [ ] `add-documentation-page` skill in `technical-documentation/.claude/skills/`
- [ ] Skill references British English conventions and page templates
- [ ] Skill instructs Claude to write in individual repo's `docs/` directory
- [ ] Workspace file gives Claude access to all repositories

Test the skill:

```
/add-documentation-page Create a runbook for adding a new dlt pipeline
source in the data-pipelines repository
```

Verify Claude writes the file to `data-pipelines/docs/runbooks/` and follows the template.

## Cost Summary

| Component | Monthly Cost | Notes |
|-----------|-------------|-------|
| **MkDocs** | £0 | Open-source |
| **Material for MkDocs** | £0 | Open-source (Insiders optional from $15/month) |
| **mkdocs-multirepo-plugin** | £0 | Open-source |
| **GitHub Pages** | £0 | Free for public repos; included with Team/Enterprise for private |
| **GitHub Actions** | £0 | Within free tier (docs builds are fast) |
| **Total** | **£0** | |

The entire documentation stack is free. GitHub Actions minutes for docs CI/CD are minimal - a typical `mkdocs build` completes in under a minute.

## Maintenance

### Routine Tasks

| Task | Frequency | What to Do |
|------|-----------|-----------|
| Review docs accuracy | Monthly | Read through each repo's docs, verify they match reality |
| Test runbooks | Quarterly | Walk through each runbook to catch stale steps |
| Update conventions | As needed | When patterns change, update conventions page and Claude skill |
| Review CI/CD | Quarterly | Check for plugin updates, GitHub Actions deprecations |

### When Adding a New Repository

To add another repository to the unified site:

1. Add a `docs/` directory with the standard pages (index, getting started, conventions, runbooks)
2. Add a `docs/mkdocs.yml` for local preview
3. Add MkDocs dependencies: `uv add --dev mkdocs mkdocs-material mkdocs-glightbox`
4. Add an `!import` directive to the central `mkdocs.yml`
5. Add the caller workflow `.github/workflows/docs-ci.yml`
6. Add the dispatch trigger to the repository's CI workflow
7. Add the repository to the workspace file
8. Update the Claude skill's write location list

### When Upgrading MkDocs

Update dependencies in the central repository:

```sh
cd ~/src/technical-documentation
uv add --dev mkdocs@latest mkdocs-material@latest
```

Then update each source repository:

```sh
cd ~/src/terraform
uv add --dev mkdocs@latest mkdocs-material@latest
```

The reusable CI workflow ensures all repositories use the same build process, so version mismatches between local preview and the unified site are rare.

## Extending to Additional Repositories

The pattern established here scales to any number of repositories. For example, if you later add a `ml-pipelines` repository:

1. Add `docs/` to the repository following the same structure
2. Add an `!import` to the central `mkdocs.yml`:
   ```yaml
   - ML Pipelines:
       - '!import https://github.com/<your-org>/ml-pipelines?branch=main&docs_dir=docs'
   ```
3. Add CI and dispatch workflows
4. The Claude skill already handles arbitrary repositories - update its write location list

## Summary

- [x] **Unified documentation site** — Single searchable site covering all platform repositories
- [x] **Documentation alongside code** — Source files live in each repository, reviewed in PRs
- [x] **Automated deployment** — CI validates on PRs, CD deploys on merge to any repository
- [x] **Reusable CI** — Shared workflow means one place to update the build process
- [x] **Runbook framework** — Standard template for operational procedures across all repos
- [x] **Claude documentation skill** — Consistent page creation via shared skill in workspace
- [x] **dbt docs integration** — Auto-generated dbt docs linked from the unified site

## What's Next

!!! success
    You have built a complete documentation system for your data platform. Documentation lives alongside code, deploys automatically, and Claude can help keep it current.

Your data platform now has infrastructure, ingestion, transformation, analytics, observability, streaming, and documentation. The next section covers ongoing maintenance - keeping everything running, adding resources, and responding to incidents.

Continue to [Maintain Your Data Stack](../../maintain/index.md) →
