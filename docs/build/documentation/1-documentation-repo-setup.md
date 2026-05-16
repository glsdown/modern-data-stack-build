# Documentation Repository Setup

On this page, you will:

- [x] Create the `technical-documentation` repository
- [x] Set up MkDocs Material with the multirepo plugin
- [x] Configure the theme to match your existing documentation style
- [x] Build and preview the unified site locally

## Overview

The `technical-documentation` repository is the central hub for your unified documentation site. It does not contain much documentation itself - instead, it owns the site configuration, theme, and the `mkdocs-multirepo-plugin` setup that pulls documentation from your other repositories at build time.

```
technical-documentation/
├── mkdocs.yml                        # Unified navigation, theme, plugin config
├── pyproject.toml                    # Python dependencies (MkDocs + plugins)
├── docs/
│   └── index.md                      # Landing page for the unified site
├── .claude/
│   └── skills/
│       └── add-documentation-page/   # Shared Claude skill (covered later)
├── .github/
│   └── workflows/
│       ├── deploy-docs.yml           # CD: deploy to GitHub Pages
│       └── docs-ci-reusable.yml      # Reusable CI (called by other repos)
└── data-stack.code-workspace         # VS Code/Cursor multi-root workspace
```

## Create the Repository

Create a new repository in your GitHub organisation:

1. Navigate to your GitHub organisation
2. Click **New repository**
3. Name it `technical-documentation`
4. Set visibility to match your other repositories (private for most teams)
5. Initialise with a README
6. Clone the repository locally:

```sh
cd ~/src
git clone git@github.com:<your-org>/technical-documentation.git
cd technical-documentation
```

## Initialise the Python Project

Use `uv` to set up the Python project and install MkDocs with all required plugins:

```sh
uv init
```

Add the MkDocs dependencies:

```sh
uv add --dev \
    mkdocs \
    mkdocs-material \
    mkdocs-git-revision-date-localized-plugin \
    mkdocs-glightbox \
    mkdocs-multirepo-plugin
```

This installs:

| Package | Purpose |
|---------|---------|
| `mkdocs` | Static site generator for documentation |
| `mkdocs-material` | Material Design theme with rich features |
| `mkdocs-git-revision-date-localized-plugin` | Shows creation and modification dates on pages |
| `mkdocs-glightbox` | Image lightbox for full-screen viewing |
| `mkdocs-multirepo-plugin` | Imports docs from multiple repositories at build time |

## Create the MkDocs Configuration

Create `mkdocs.yml` at the repository root. This configuration matches the style of the guide you are reading now - the same colour scheme, font, and extensions:

```yaml
site_name: Data Platform Documentation
site_url: https://<your-org>.github.io/technical-documentation
repo_url: https://github.com/<your-org>/technical-documentation
nav:
  - Home: index.md
  - Infrastructure:
      - '!import https://github.com/<your-org>/terraform?branch=main&docs_dir=docs'
  - Data Pipelines:
      - '!import https://github.com/<your-org>/data-pipelines?branch=main&docs_dir=docs'
  - Data Transformation:
      - '!import https://github.com/<your-org>/dbt-transform?branch=main&docs_dir=docs'
theme:
  name: material
  font:
    text: Raleway
  favicon: images/favicon.png
  icon:
    logo: fontawesome/solid/cubes-stacked
  palette:
    # Palette toggle for light mode
    - media: "(prefers-color-scheme: light)"
      scheme: default
      primary: blue grey
      toggle:
        icon: material/lightbulb
        name: Switch to dark mode

    # Palette toggle for dark mode
    - media: "(prefers-color-scheme: dark)"
      scheme: slate
      primary: light blue
      toggle:
        icon: material/lightbulb-outline
        name: Switch to light mode
  features:
    - navigation.instant
    - navigation.tracking
    - navigation.indexes
    - toc.follow
    - search.highlight
    - search.suggest
plugins:
  - search
  - glightbox
  - multirepo:
      cleanup: false  # Set to true in CI, false for local development
      keep_docs_dir: true
  - git-revision-date-localized:
      type: date
      enable_creation_date: true
      fallback_to_build_date: true
markdown_extensions:
  - admonition
  - attr_list
  - md_in_html
  - pymdownx.highlight:
      anchor_linenums: true
  - pymdownx.inlinehilite
  - pymdownx.snippets
  - pymdownx.superfences:
      custom_fences:
        - name: mermaid
          class: mermaid
          format: !!python/name:pymdownx.superfences.fence_code_format
  - pymdownx.tasklist:
      custom_checkbox: true
      clickable_checkbox: true
  - pymdownx.tabbed:
      alternate_style: true
  - pymdownx.keys
extra:
  social:
    - icon: fontawesome/brands/github
      link: https://github.com/<your-org>
  generator: false
```

### Key Configuration Details

**Navigation with `!import`** — Each `!import` directive tells the multirepo plugin to clone that repository and pull in its `docs/` directory. The imported repository must have a `mkdocs.yml` file (in `docs/` or the root) with a `nav` section that defines its page structure.

**`cleanup: false`** — During local development with `mkdocs serve`, set this to `false` so the plugin does not remove the cloned repositories between rebuilds. In CI/CD, set it to `true` to keep the build environment clean.

**Theme and extensions** — These match the configuration used throughout this guide. Your documentation site will have the same look and feel: Raleway font, blue grey / light blue colour scheme, light/dark mode toggle, admonitions, tabbed content, and Mermaid diagrams.

!!! tip "Private Repositories"
    If your repositories are private, the multirepo plugin needs authentication to clone them. For local development, your existing SSH key or Git credentials handle this automatically. For CI/CD, you will configure a GitHub token - covered in [CI/CD Pipeline](5-ci-cd-pipeline.md).

## Create the Landing Page

Create `docs/index.md`:

```markdown
# Data Platform Documentation

Welcome to the data platform documentation. This site brings together documentation
from all platform repositories into a single, searchable location.

## Repositories

| Repository | Description | Docs |
|------------|-------------|------|
| **Infrastructure** | Terraform modules for AWS, Snowflake, and GitHub | [View docs](terraform/) |
| **Data Pipelines** | dlt pipelines and Prefect orchestration | [View docs](data-pipelines/) |
| **Data Transformation** | dbt models, tests, and macros | [View docs](dbt-transform/) |

## Additional Resources

- **[dbt Docs](https://your-dbt-docs-url)** - Auto-generated model documentation,
  lineage graph, and data dictionary (hosted separately)
- **[Prefect Dashboard](https://app.prefect.cloud)** - Pipeline monitoring and
  flow run history
- **[Snowflake Console](https://app.snowflake.com)** - Data warehouse management

## About This Site

This site is built with [MkDocs Material](https://squidfundamentals.com/mkdocs-material/)
and the [multirepo plugin](https://github.com/jdoiro3/mkdocs-multirepo-plugin).
Documentation source files live alongside code in each repository. Changes to
documentation are reviewed in pull requests alongside code changes.
```

Create the images directory for the favicon:

```sh
mkdir -p docs/images
```

!!! info "Favicon"
    Copy your organisation's favicon to `docs/images/favicon.png`, or remove the `favicon` line from `mkdocs.yml` to use the default Material theme icon.

## Local Development

### Preview the Site

Start the local development server:

```sh
uv run mkdocs serve --livereload
```

This clones each imported repository into a temporary directory and builds the unified site. The first run takes longer as it clones the repositories. Subsequent rebuilds are faster because `cleanup: false` preserves the cloned content.

Open `http://127.0.0.1:8000` in your browser to preview the site.

!!! warning "First Build"
    The first `mkdocs serve` will fail if the imported repositories do not yet have `docs/` directories with a `mkdocs.yml` containing a `nav` section. That is expected - you will set those up in the [next page](2-repository-integration.md). For now, you can comment out the `!import` lines in the nav to verify the base site builds.

### Preview Individual Repos

Each repository also has its own minimal `mkdocs.yml` for independent preview. When working on documentation for a specific repository, you can preview just that repository's docs without cloning the others:

```sh
# From within the terraform repository
cd ~/src/terraform
uv run mkdocs serve --livereload
```

This is faster and does not require network access to clone other repositories.

## Verify the Build

Run a strict build to catch any issues:

```sh
uv run mkdocs build --strict
```

The `--strict` flag treats warnings as errors, catching broken links, missing pages, and invalid configuration before deployment.

## Summary

!!! success "What You've Accomplished"
    - [x] Created the `technical-documentation` repository
    - [x] Installed MkDocs Material with the multirepo plugin
    - [x] Configured the theme to match your existing documentation style
    - [x] Created the landing page for the unified site
    - [x] Verified local development workflow

## What's Next

The central repository is ready, but the imported repositories do not have `docs/` directories yet. The next step is to add documentation directories and initial content to each of your source repositories.

Continue to [Repository Integration](2-repository-integration.md) →
