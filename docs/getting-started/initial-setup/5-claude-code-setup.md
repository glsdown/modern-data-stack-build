# Claude Code Setup

On this page, you will:

- [x] Understand how CLAUDE.md and skills guide Claude Code in your repositories
- [x] Learn the file structure you'll use across all three repositories

## Overview

As you build repositories throughout this documentation - Terraform, dbt, and Prefect - each one develops its own conventions, module patterns, and safety rules. When Claude Code works in these repositories - adding users, creating pipelines, building models - it needs to follow those conventions precisely. A wrong naming pattern or a missing grant can break downstream access.

**CLAUDE.md** is a project reference file that sits at the root of a repository. It tells Claude Code about the repository's structure, conventions, and safety rules. Think of it as onboarding documentation - but for an AI agent rather than a human.

**Skills** are reusable task templates stored in `.claude/skills/`. Each skill provides step-by-step instructions for a common task, ensuring Claude Code follows the correct procedure every time.

```
your-repository/
├── CLAUDE.md                  ← Project context and conventions
├── .claude/
│   └── skills/
│       ├── add-user/
│       │   └── SKILL.md       ← Step-by-step task instructions
│       └── add-data-source/
│           └── SKILL.md
└── (project files)
```

You'll add a CLAUDE.md and skills to each repository as you build it. The templates and instructions for each are in the [Maintain](../../maintain/index.md) section:

| Repository | When to set up | Skills |
|-----------|---------------|--------|
| `terraform/` | After [Data Warehouse](../../build/data-warehouse/index.md) | `add-snowflake-user`, `add-data-source` |
| `dbt-transform/` | After [Data Transformation](../../build/data-transformation/index.md) | `add-source-and-staging`, `add-mart-model` |
| `data-pipelines/` | After [Orchestration](../../build/orchestration/index.md) | `add-dlt-pipeline` |

## How Skills Work

Skills are invoked by typing `/{skill-name}` in Claude Code. Claude reads the SKILL.md and follows the steps, asking you for the required information along the way.

```
/add-snowflake-user
```

Claude Code will ask which type of user to add, then follow the skill's procedure - choosing the right configuration file, applying the correct role and warehouse, and validating with `terraform plan` before applying.

## Summary

!!! success "What You've Accomplished"
    - [x] Understood how CLAUDE.md and skills guide Claude Code in your repositories
    - [x] Learned the file structure you'll apply to each repository as you build it

## What's Next

Continue to [Hosted Account Setup](../account-setup/index.md) →
