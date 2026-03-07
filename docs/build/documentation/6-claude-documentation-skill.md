# Claude Documentation Skill

On this page, you will:

- [x] Set up a VS Code/Cursor multi-root workspace spanning all repositories
- [x] Create a shared Claude skill for writing documentation pages
- [x] Configure the skill to write docs in the correct repository
- [x] Test the skill by generating a documentation page

## Overview

Throughout this guide, you have used Claude skills to automate repetitive tasks - adding Snowflake users, adding data sources. A documentation skill follows the same pattern: it gives Claude the conventions, templates, and style rules needed to write documentation pages that are consistent with the rest of the site.

The key design decision is where the skill lives. Rather than duplicating it across every repository, the skill lives once in the central `technical-documentation` repository. A VS Code/Cursor multi-root workspace includes all repositories, so Claude can access the skill regardless of which repository's files are being edited.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    WORKSPACE LAYOUT                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  data-stack.code-workspace                                              │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                   │   │
│  │  technical-documentation/                                        │   │
│  │  ├── .claude/skills/add-documentation-page/SKILL.md  ◄── Skill  │   │
│  │  └── mkdocs.yml                                                  │   │
│  │                                                                   │   │
│  │  terraform/                                                      │   │
│  │  ├── docs/  ◄── Claude writes here                               │   │
│  │  └── snowflake/modules/                                          │   │
│  │                                                                   │   │
│  │  data-pipelines/                                                 │   │
│  │  ├── docs/  ◄── Claude writes here                               │   │
│  │  └── sources/                                                    │   │
│  │                                                                   │   │
│  │  dbt-transform/                                                  │   │
│  │  ├── docs/  ◄── Claude writes here                               │   │
│  │  └── models/                                                     │   │
│  │                                                                   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Create the Multi-Root Workspace

### Workspace File

Create `data-stack.code-workspace` in the `technical-documentation` repository:

```json
{
    "folders": [
        {
            "name": "Technical Documentation",
            "path": "."
        },
        {
            "name": "Terraform",
            "path": "../terraform"
        },
        {
            "name": "Data Pipelines",
            "path": "../data-pipelines"
        },
        {
            "name": "dbt Transform",
            "path": "../dbt-transform"
        }
    ],
    "settings": {
        "files.exclude": {
            "**/.git": true,
            "**/__pycache__": true,
            "**/node_modules": true,
            "**/.terraform": true
        }
    }
}
```

!!! info "Relative Paths"
    The workspace file uses relative paths (`../terraform`), assuming all repositories are cloned into the same parent directory. If your repositories are elsewhere, adjust the paths accordingly.

### Open the Workspace

Clone all repositories into the same parent directory:

```sh
cd ~/src
git clone git@github.com:<your-org>/technical-documentation.git
git clone git@github.com:<your-org>/terraform.git
git clone git@github.com:<your-org>/data-pipelines.git
git clone git@github.com:<your-org>/dbt-transform.git
```

Open the workspace in VS Code or Cursor:

```sh
code ~/src/technical-documentation/data-stack.code-workspace
```

Or in Cursor:

```sh
cursor ~/src/technical-documentation/data-stack.code-workspace
```

You should see all four repositories listed in the Explorer sidebar. Claude Code now has access to all repositories and their skills.

## Create the Documentation Skill

### Skill File

Create `.claude/skills/add-documentation-page/SKILL.md` in the `technical-documentation` repository:

```markdown
---
name: add-documentation-page
description: >
  Add a new documentation page to one of the platform repositories
  (terraform, data-pipelines, or dbt-transform). Follows the established
  style, structure, and British English conventions.
allowed-tools: Read, Write, Glob, Grep
---

# Add Documentation Page

This skill creates documentation pages for the data platform repositories.
Pages are written in the individual repository's `docs/` directory, not in
the central technical-documentation repository.

## When to Use

- Adding a new getting started guide, conventions page, or architecture
  decision record to a repository
- Creating a new runbook for an operational procedure
- Documenting a new pattern or convention

## Important: Write Location

**Always write documentation pages in the individual repository's `docs/`
directory.** The technical-documentation repository only contains the unified
site configuration - not the documentation content itself.

- Terraform docs → `terraform/docs/`
- Pipeline docs → `data-pipelines/docs/`
- dbt docs → `dbt-transform/docs/`
- Runbooks → `<repo>/docs/runbooks/`

## Page Structure

Every documentation page follows this structure:

` ``markdown
# Page Title

Brief introduction explaining what this page covers.

## Overview

High-level explanation. Use ASCII diagrams for architecture.

## [Topic Sections]

Detailed content. Explain concepts before instructions. Use admonitions.

## Summary

!!! success "Key Points"
    - [x] Point 1
    - [x] Point 2
` ``

## Style Requirements

### British English

Always use British spellings:
- organise, analyse, optimise, customise, realise
- colour, centre, licence (verb), defence, catalogue

### Voice and Tense

- Second person: "You configure..."
- Active voice: "Configure the provider"
- Present tense: "This creates a role"

### Avoid

- Time estimates ("this takes 5 minutes")
- Subjective difficulty ("this is easy", "simply...")
- Jargon without explanation

### Code Blocks

- Shell: `sh`
- SQL: `sql`
- Terraform: `hcl`
- Python: `python`
- YAML: `yaml`

### Admonitions

- `!!! info` — Background and context
- `!!! tip` — Best practices and recommendations
- `!!! warning` — Critical information and risks
- `!!! danger` — Severe risks and destructive actions
- `!!! success` — Achievements and checklists
- `!!! note` — Additional information

## Runbook Template

When creating a runbook, use this structure:

` ``markdown
# Runbook: [Task Name]

## Summary
One or two sentences.

## When to Use
- Trigger condition 1
- Trigger condition 2

## Prerequisites
- [ ] Access needed
- [ ] Tools needed

## Steps

### 1. First Step
[Instructions]

### 2. Second Step
[Instructions]

## Verification
- [ ] Check 1
- [ ] Check 2

## Rollback
1. Rollback step

## Escalation
- First contact: [team/person]
- Escalation: [next level]
` ``

## After Writing

1. Update the repository's `docs/mkdocs.yml` nav to include the new page
2. Verify the page builds: `cd <repo>/docs && uv run mkdocs build --strict`
3. Check that British English is used throughout
4. Ensure all code blocks have language identifiers
```

### Verify Claude Sees the Skill

With the workspace open, ask Claude:

```
What documentation skills are available?
```

Claude should identify the `add-documentation-page` skill from the `technical-documentation` repository.

## Using the Skill

### Example: Create a Terraform Runbook

Ask Claude to create a new runbook:

```
/add-documentation-page Create a runbook for rotating Snowflake service
account credentials in the terraform repository
```

Claude should:

1. Read existing runbooks and conventions
2. Create the file at `terraform/docs/runbooks/rotate-credentials.md`
3. Follow the runbook template (summary, when to use, prerequisites, steps, verification, rollback, escalation)
4. Use British English throughout
5. Update `terraform/docs/mkdocs.yml` to include the new page in the nav

### Example: Create an Architecture Decision

```
/add-documentation-page Write an ADR for the data-pipelines repository
explaining why we use dlt over custom Python scripts for data ingestion
```

Claude should create the file at `data-pipelines/docs/decisions/adr-001-dlt-over-custom-scripts.md` and follow the ADR template.

### Example: Create a Getting Started Update

```
/add-documentation-page Update the dbt-transform getting started guide
to include the new testing conventions we adopted last sprint
```

Claude reads the existing guide, understands the testing conventions from the codebase, and updates `dbt-transform/docs/getting-started.md`.

## Keeping the Skill Up-to-Date

The skill reflects your current documentation conventions. When conventions change, update the skill:

- **New admonition patterns** — Add examples to the skill
- **New page types** — Add templates (e.g. a "migration guide" template)
- **Style changes** — Update the style requirements section
- **New repositories** — Add the repository to the write location list

Because the skill lives in the central repository, updating it once applies everywhere - any team member opening the workspace gets the latest conventions.

## Summary

!!! success "What You've Accomplished"
    - [x] Created a VS Code/Cursor multi-root workspace spanning all repositories
    - [x] Built a shared Claude skill for consistent documentation across repos
    - [x] Configured the skill to write docs in the correct repository's `docs/` directory
    - [x] Tested the skill by generating documentation pages

## Next Steps

All the pieces are in place: documentation directories in each repository, a unified site with CI/CD, and a Claude skill for consistent page creation. The final page wraps up with a verification checklist and next steps.

Continue to [Finishing Up](7-finishing-up.md) →
