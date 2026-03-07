# CI/CD Pipeline

On this page, you will:

- [x] Create a reusable CI workflow for validating documentation builds
- [x] Configure each source repository to call the reusable workflow
- [x] Set up continuous deployment to GitHub Pages
- [x] Configure cross-repo triggers so the site rebuilds when any repo changes

## Overview

Documentation CI/CD has two parts:

1. **CI (validation)** — On every pull request, verify the documentation builds without errors. A reusable workflow in the central repo means all repositories share the same CI process.
2. **CD (deployment)** — On every merge to main, deploy the unified site to GitHub Pages. Cross-repo triggers ensure the site stays current when any source repository changes.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DOCUMENTATION CI/CD                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  terraform/              data-pipelines/        dbt-transform/          │
│  ┌──────────────┐       ┌──────────────┐       ┌──────────────┐        │
│  │ PR opened    │       │ PR opened    │       │ PR opened    │        │
│  │     │        │       │     │        │       │     │        │        │
│  │     ▼        │       │     ▼        │       │     ▼        │        │
│  │ docs-ci.yml  │       │ docs-ci.yml  │       │ docs-ci.yml  │        │
│  │ (5 lines)    │       │ (5 lines)    │       │ (5 lines)    │        │
│  │     │        │       │     │        │       │     │        │        │
│  │     └────────┼───────┼─────┘────────┼───────┼─────┘        │        │
│  └──────────────┘       └──────────────┘       └──────────────┘        │
│                    │                                                    │
│                    ▼                                                    │
│           technical-documentation/                                      │
│           ┌──────────────────────────────────┐                         │
│           │ docs-ci-reusable.yml             │  Shared CI logic        │
│           │   uv setup → install → build     │                         │
│           └──────────────────────────────────┘                         │
│                                                                         │
│  On merge to main (any repo):                                          │
│           ┌──────────────────────────────────┐                         │
│           │ deploy-docs.yml                  │  Builds unified site    │
│           │   clone all repos → build →      │  and deploys to         │
│           │   deploy to GitHub Pages         │  GitHub Pages           │
│           └──────────────────────────────────┘                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Reusable CI Workflow

### Why Reusable?

Without a reusable workflow, each repository would have its own copy of the CI logic. When you change the MkDocs version, add a plugin, or adjust the build flags, you would need to update every repository. A reusable workflow centralises the logic - update once, every repository benefits.

### Create the Reusable Workflow

In the `technical-documentation` repository, create `.github/workflows/docs-ci-reusable.yml`:

```yaml
name: Documentation CI (Reusable)

on:
  workflow_call:
    inputs:
      docs_dir:
        description: 'Path to the docs directory containing mkdocs.yml'
        required: false
        default: 'docs'
        type: string
      python_version:
        description: 'Python version to use'
        required: false
        default: '3.13'
        type: string

jobs:
  validate-docs:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v4
        with:
          enable-cache: true

      - name: Set up Python
        run: uv python install ${{ inputs.python_version }}

      - name: Install dependencies
        run: uv sync --dev

      - name: Build documentation (strict)
        working-directory: ${{ inputs.docs_dir }}
        run: uv run mkdocs build --strict
```

This workflow:

- Accepts optional inputs for the docs directory path and Python version
- Uses `uv` for dependency management (matching the project convention)
- Runs `mkdocs build --strict` to catch broken links, missing pages, and configuration errors
- Caches uv dependencies for faster subsequent runs

### Caller Workflows in Source Repositories

Each source repository has a thin workflow that calls the reusable one. Create `.github/workflows/docs-ci.yml` in each repository:

```yaml
name: Documentation CI

on:
  pull_request:
    paths:
      - 'docs/**'
      - 'mkdocs.yml'
      - 'pyproject.toml'

jobs:
  validate:
    uses: <your-org>/technical-documentation/.github/workflows/docs-ci-reusable.yml@main
    with:
      docs_dir: 'docs'
```

That is the entire file. When the CI process needs to change - a new plugin, a different Python version, additional validation steps - update only the reusable workflow in the central repository.

!!! info "Path Filters"
    The `paths` filter ensures CI only runs when documentation-related files change. Adjust the paths to include any files that affect the documentation build (e.g. add `'*.tf'` if you use mkdocs-snippets to include Terraform code in docs).

!!! warning "Private Repositories"
    Reusable workflows across repositories require the calling repository to have access to the reusable workflow. For private repositories, the reusable workflow repository must either:

    - Be in the same organisation, with Actions permissions allowing access
    - Be explicitly shared via repository settings → Actions → General → Allow access from other repositories

    Navigate to the `technical-documentation` repository → Settings → Actions → General → Access, and select **"Accessible from repositories in the organisation"**.

## Continuous Deployment

### GitHub Pages Setup

Before configuring the deployment workflow, enable GitHub Pages on the `technical-documentation` repository:

1. Navigate to Settings → Pages
2. Under **Source**, select **GitHub Actions**

!!! tip "Source: GitHub Actions"
    Selecting "GitHub Actions" as the source (rather than a branch) gives the workflow full control over deployment. This is the recommended approach for MkDocs sites.

### Deployment Workflow

Create `.github/workflows/deploy-docs.yml` in the `technical-documentation` repository:

```yaml
name: Deploy Documentation

on:
  push:
    branches:
      - main
  repository_dispatch:
    types:
      - docs-updated
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v4
        with:
          enable-cache: true

      - name: Set up Python
        run: uv python install 3.13

      - name: Install dependencies
        run: uv sync --dev

      - name: Build unified documentation
        run: uv run mkdocs build --strict
        env:
          GithubAccessToken: ${{ secrets.DOCS_GITHUB_TOKEN }}

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: site/

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

### Key Configuration Details

**Triggers** — The workflow runs on:

- **Push to main**: When the central repo's configuration changes
- **Repository dispatch**: When a source repository triggers a rebuild (covered below)
- **Workflow dispatch**: Manual trigger for ad-hoc rebuilds

**`GithubAccessToken`** — The multirepo plugin needs this token to clone private source repositories during the build. Create a fine-grained personal access token (or GitHub App token) with read access to the source repositories, and add it as a repository secret named `DOCS_GITHUB_TOKEN`.

**Concurrency** — The `concurrency` block ensures only one deployment runs at a time, preventing conflicts when multiple repositories trigger rebuilds simultaneously.

**`cleanup: true` in CI** — For the deployment build, set `cleanup: true` in `mkdocs.yml` or override via environment. The CI environment does not benefit from caching cloned repos between builds.

!!! tip "Overriding cleanup for CI"
    You can set `cleanup: false` in `mkdocs.yml` for local development and override it in CI by setting the `MULTIREPO_CLEANUP` environment variable, or by using a CI-specific `mkdocs.yml` override. Alternatively, keep `cleanup: true` in the committed config and accept slightly slower local rebuilds.

### Create the GitHub Token

1. Navigate to GitHub → Settings → Developer settings → Fine-grained personal access tokens
2. Create a new token with:
   - **Repository access**: Select your source repositories (terraform, data-pipelines, dbt-transform)
   - **Permissions**: Contents → Read-only
3. Copy the token
4. In the `technical-documentation` repository → Settings → Secrets and variables → Actions
5. Add a new secret: `DOCS_GITHUB_TOKEN` with the token value

!!! info "GitHub App Alternative"
    For production setups, consider using a GitHub App instead of a personal access token. GitHub Apps have finer-grained permissions, do not expire with user accounts, and can be managed at the organisation level.

## Cross-Repo Triggers

When documentation changes are merged in a source repository, the unified site should rebuild automatically. Each source repository dispatches an event to the central repository after a successful merge.

### Add Dispatch Step to Source Repositories

Add a job to each source repository's existing CI/CD workflow (or create a new workflow) that fires after docs changes are merged to main:

```yaml
name: Trigger Docs Rebuild

on:
  push:
    branches:
      - main
    paths:
      - 'docs/**'

jobs:
  trigger-rebuild:
    runs-on: ubuntu-latest
    steps:
      - name: Trigger documentation rebuild
        uses: peter-evans/repository-dispatch@v3
        with:
          token: ${{ secrets.DOCS_DISPATCH_TOKEN }}
          repository: <your-org>/technical-documentation
          event-type: docs-updated
          client-payload: '{"repo": "${{ github.repository }}", "sha": "${{ github.sha }}"}'
```

### Create the Dispatch Token

The dispatch step needs a token with permission to trigger workflows in the `technical-documentation` repository:

1. Create a fine-grained personal access token (or use the existing GitHub App)
2. **Repository access**: Select `technical-documentation`
3. **Permissions**: Contents → Read and write (needed for `repository_dispatch`)
4. Add as a secret named `DOCS_DISPATCH_TOKEN` in each source repository

!!! tip "Organisation-Level Secrets"
    If all repositories are in the same GitHub organisation, create `DOCS_DISPATCH_TOKEN` as an organisation-level secret. This avoids adding the same secret to each repository individually.

    Navigate to Organisation → Settings → Secrets and variables → Actions → New organisation secret.

## Complete Workflow Summary

After setup, the documentation lifecycle works as follows:

| Event | What Happens |
|-------|-------------|
| PR opened in source repo | Caller workflow triggers reusable CI → validates docs build |
| PR merged in source repo | Dispatch triggers unified site rebuild → deploys to GitHub Pages |
| PR merged in central repo | Push trigger rebuilds and deploys the unified site |
| Manual trigger | Workflow dispatch rebuilds and deploys on demand |

## Verify the Pipeline

### Test CI

1. In a source repository, create a branch and modify a file in `docs/`
2. Open a pull request
3. Verify the "Documentation CI" check appears and passes

### Test CD

1. Merge the pull request
2. Check the `technical-documentation` repository's Actions tab for a triggered "Deploy Documentation" run
3. After deployment, visit `https://<your-org>.github.io/technical-documentation/` to see the updated site

## Summary

!!! success "What You've Accomplished"
    - [x] Created a reusable CI workflow that all repositories share
    - [x] Configured thin caller workflows in each source repository
    - [x] Set up continuous deployment to GitHub Pages with the multirepo plugin
    - [x] Configured cross-repo triggers so the site stays current
    - [x] Created GitHub tokens for cross-repo access

## Next Steps

The CI/CD pipeline keeps your documentation site building and deploying automatically. The next step is to set up a Claude skill that helps your team write documentation consistently across all repositories.

Continue to [Claude Documentation Skill](6-claude-documentation-skill.md) →
