# Writing Documentation

On this page, you will:

- [x] Understand what belongs on the docs site versus in-place in code
- [x] Learn the page structure for repository-level documentation
- [x] Write an architecture decision record (ADR)
- [x] Establish a process for keeping documentation current

## Overview

The documentation site covers **repo-level** content - how to use the repository as a whole. Individual resources (specific Terraform modules, dlt pipelines, dbt models) are documented in-place alongside the code. This separation keeps resource-level docs close to what they describe, while the docs site provides the broader context that new team members and on-call engineers need.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   DOCUMENTATION LAYERS                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  DOCS SITE (repo-level)              IN-PLACE (resource-level)          │
│  ─────────────────────               ────────────────────────           │
│                                                                         │
│  Architecture decisions              Module READMEs                     │
│  Getting started guides              Variable descriptions              │
│  Naming conventions                  Inline comments                    │
│  Configuration patterns              Source docstrings                  │
│  Operational runbooks                Pipeline README files              │
│  Onboarding checklists              Model descriptions (YAML)          │
│  Design rationale                    Doc blocks (dbt)                   │
│                                      Auto-generated dbt docs            │
│                                                                         │
│  "How do I use this repo?"          "What does this specific            │
│  "Why was it built this way?"        resource do?"                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## What Belongs on the Docs Site

### Architecture Documentation

Explain how the repository is structured and why. New team members need to understand the overall design before diving into individual resources.

**Good examples:**

- Why the Terraform repository uses separate state files per provider
- Why dbt models use a four-layer structure (staging → intermediate → marts → reporting)
- Why data pipelines separate sources, pipelines, and flows into different directories
- Why Snowflake uses multiple provider aliases for different admin roles

**Bad examples (too granular for the docs site):**

- What the `snowflake_warehouse` module's `auto_suspend` parameter does (belongs in the module README)
- How the `stg_hubspot__contacts` model transforms raw data (belongs in the model YAML description)

### Getting Started Guides

Step-by-step instructions for someone new to the repository. Cover:

1. **Prerequisites** - Tools, access, and accounts needed
2. **Local setup** - Clone, install dependencies, configure credentials
3. **First task** - A simple, safe task to verify the setup works
4. **Development workflow** - Branch, change, test, PR, deploy

### Conventions

Document patterns that are not obvious from reading the code alone:

- Naming conventions (service accounts, databases, roles, models)
- Configuration patterns (where values live, how they are structured)
- Testing approach (what to test, how to run tests, coverage expectations)
- Code organisation (directory structure, file naming, module boundaries)

### Runbooks

Operational procedures for common tasks and incident response. Covered in detail on the [next page](4-writing-runbooks.md).

## What Belongs In-Place in the Code

### Terraform: Module Documentation

Each Terraform module should have a `README.md` in its directory with:

- **Purpose** - What the module creates
- **Inputs** - Variable table (name, type, description, default)
- **Outputs** - Output table (name, description)
- **Usage example** - How to call the module

```markdown
# Snowflake Database Module

Creates a Snowflake database with associated `DB_READER` and `DB_WRITER`
database roles and grants.

## Inputs

| Name | Type | Description | Default |
|------|------|-------------|---------|
| `name` | `string` | Database name | - |
| `data_retention_time_in_days` | `number` | Time Travel retention | `1` |
| `reader_roles` | `list(string)` | Roles granted DB_READER | `[]` |
| `writer_roles` | `list(string)` | Roles granted DB_WRITER | `[]` |

## Usage

` ``hcl
module "analytics_db" {
  source = "../modules/snowflake_database"

  name                         = "ANALYTICS"
  data_retention_time_in_days  = 7
  reader_roles                 = ["ANALYTICS_REPORTER"]
  writer_roles                 = ["ANALYTICS_TRANSFORMER"]

  providers = {
    snowflake.sys_admin      = snowflake.sys_admin
    snowflake.security_admin = snowflake.security_admin
  }
}
` ``
```

!!! tip "terraform-docs"
    Consider using [`terraform-docs`](https://terraform-docs.io/) to auto-generate module documentation from your Terraform code. It creates consistent README files from variable and output blocks.

### Data Pipelines: Source and Pipeline Documentation

Document sources and pipelines using docstrings and README files in each directory:

```python
@dlt.source(section="open_exchange_rates")
def exchange_rates(
    api_key: str = dlt.secrets.value,
    base_currency: str = "GBP",
) -> Iterator[DltResource]:
    """Extract exchange rates from Open Exchange Rates API.

    Fetches historical exchange rates for the given base currency.
    Rates are loaded incrementally based on the date field.

    Args:
        api_key: Open Exchange Rates API key (from secrets)
        base_currency: Base currency code (default: GBP)
    """
```

### dbt: Model Descriptions and Doc Blocks

Document models in their YAML files and use doc blocks for reusable descriptions:

```yaml
models:
  - name: stg_hubspot__contacts
    description: >
      Staged HubSpot contacts with standardised column names and types.
      Deduplicates on contact_id, keeping the most recent record.
    columns:
      - name: contact_id
        description: Unique identifier for the contact in HubSpot
        data_tests:
          - unique
          - not_null
```

The auto-generated dbt docs site renders these descriptions with the lineage graph, making them discoverable without the docs site.

## Page Structure for Docs Site Pages

Follow a consistent structure across all repository docs:

```markdown
# Page Title

Brief introduction explaining what this page covers and why it matters.

## Overview

High-level explanation of the topic. Use ASCII diagrams for architecture.

## [Topic Sections]

Detailed content broken into logical sections. Explain concepts before
providing instructions. Use admonitions for tips, warnings, and notes.

## Summary

!!! success "Key Points"
    - [x] Point 1
    - [x] Point 2
    - [x] Point 3
```

### Style Guide

Follow the same conventions used throughout this guide:

- **British English** - organise, analyse, colour, centre
- **Second person** - "You configure...", not "Users configure..."
- **Active voice** - "Configure the provider", not "The provider should be configured"
- **Present tense** - "This creates a role", not "This will create a role"
- **No time estimates** - never mention how long something takes
- **Explain why before how** - context before instructions

## Architecture Decision Records

Architecture decision records (ADRs) capture the reasoning behind significant design choices. When someone asks "why did we do it this way?", the ADR provides the answer - even if the original decision-makers have moved on.

### ADR Template

Create ADRs in a `docs/decisions/` directory:

```markdown
# ADR-001: Separate Terraform State Per Provider

## Status

Accepted

## Context

The data platform uses three Terraform providers (GitHub, AWS, Snowflake).
We need to decide whether to use a single state file or separate state
files per provider.

## Decision

Use separate state files - one per provider directory (github/, aws/,
snowflake/). Each has its own backend.tf and can be planned and applied
independently.

## Consequences

**Benefits:**
- A Snowflake change does not lock the AWS state file
- CI/CD can run provider-specific plans in parallel
- Blast radius of state corruption is limited to one provider

**Trade-offs:**
- Cross-provider references require data sources or hardcoded values
- More backend configurations to maintain
- Developers need to know which directory to work in

## Alternatives Considered

- **Single state file**: Simpler setup but creates bottleneck for team
- **Workspaces**: Adds complexity without solving the lock contention issue
```

### When to Write an ADR

Write an ADR when a decision:

- Affects the repository's structure or architecture
- Was debated between multiple valid approaches
- Is likely to be questioned by future team members
- Would be costly to reverse

You do not need an ADR for every small decision. Use judgement - if someone new would naturally ask "why?", document the answer.

## Keeping Documentation Current

Documentation that falls out of date is worse than no documentation - it actively misleads. Several practices help keep docs current:

### Review Docs in Pull Requests

When reviewing code changes, check whether the documentation still reflects reality. If a PR changes a convention or introduces a new pattern, the documentation should be updated in the same PR.

### Link to Code, Not Copies

Where possible, link to the actual code rather than copying it into documentation. Copied code becomes stale; links always point to the current version:

```markdown
<!-- Good: links to actual file -->
See the [warehouse module](../../snowflake/modules/snowflake_warehouse/) for
the current implementation.

<!-- Bad: copy that will go stale -->
The warehouse module uses the following configuration:
(pasted code that may no longer match reality)
```

### Use Claude to Update Docs

When working in the VS Code/Cursor workspace with the documentation skill configured, Claude can update documentation as part of code changes. When you modify a convention or add a new pattern, ask Claude to update the relevant documentation pages.

### Schedule Documentation Reviews

Set a recurring calendar event (monthly or quarterly) to review the docs site. Check that:

- Getting started guides still work for a fresh setup
- Conventions pages match actual practice
- Runbooks reference current tools and processes
- Architecture decisions are still current

## Summary

!!! success "What You've Accomplished"
    - [x] Understand the distinction between docs site content and in-place documentation
    - [x] Know how to structure pages for the repository documentation site
    - [x] Can write architecture decision records to capture design rationale
    - [x] Have a process for keeping documentation current

## Next Steps

The docs site also needs runbooks - operational procedures that on-call engineers follow when responding to incidents or performing routine maintenance tasks.

Continue to [Writing Runbooks](4-writing-runbooks.md) →
