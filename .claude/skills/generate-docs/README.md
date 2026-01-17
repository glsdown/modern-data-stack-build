# Generate Documentation Skill

This Claude Code skill generates production-ready documentation pages following the modern data stack project's established style, structure, and conventions.

## Purpose

Create consistent, high-quality documentation for any part of the project:

- Getting started guides
- Build and implementation guides
- Reference documentation
- Troubleshooting guides
- Conceptual explanations

## What This Skill Does

When generating documentation, the skill ensures:

- **British English** throughout
- **Educational tone** - explains concepts, not just procedures
- **Consistent structure** - appropriate patterns for doc type
- **Production-ready** - security, best practices, real-world examples
- **Proper formatting** - admonitions, code blocks, links
- **Quality standards** - complete checklist verification

## How to Use

### Automatic Invocation

Claude will suggest this skill when you request documentation:

```
Create a guide for setting up Terraform remote state
```

```
Write documentation for configuring Snowflake warehouses
```

```
I need reference docs for our dlt pipeline patterns
```

### Manual Invocation

```
/generate-docs for Prefect workflow orchestration setup
```

## What Gets Generated

Documentation follows project standards:

- British English spelling (organise, realise, colour)
- Educational explanations (why, not just how)
- Security best practices embedded
- Code examples with expected output
- Logical structure and navigation
- Next steps clearly defined

## Key Files

- **SKILL.md** - Main skill definition and complete documentation standards
- **STYLE-REFERENCE.md** - Quick reference for style decisions
- **README.md** - This file

## Documentation Types

The skill adapts structure based on content type:

### Getting Started Pages
- What you'll accomplish (checklist)
- Prerequisites
- Concepts and background
- Step-by-step procedures
- Verification and testing
- Best practices summary
- Next steps

### Build Guides
- What you'll build
- Prerequisites
- Architecture/design explanation
- Implementation with code
- Testing and validation
- Troubleshooting
- Next steps

### Reference Documentation
- Overview and purpose
- Quick reference tables
- Detailed descriptions
- Examples and usage
- API/configuration reference
- Related resources

### Troubleshooting Guides
- Problem description
- Common symptoms
- Diagnostic steps
- Solutions (ordered by likelihood)
- Prevention tips

## Style Standards

### Language
- British English throughout
- Second person, active voice, present tense
- No time estimates
- No subjective difficulty

### Structure
- One H1 (page title)
- Logical H2/H3 hierarchy
- Concepts before procedures
- Progressive disclosure

### Code
- Language identifiers on code blocks (`sh`, `sql`, `python`, `hcl`)
- Expected output shown
- File paths specified
- Complete, tested examples

### Links
- Internal: relative paths
- External: descriptive text (not "here" or "click here")
- Next steps: trailing arrow →

### Admonitions
- `info` - Background and context
- `tip` - Best practices and recommendations
- `warning` - Critical information and risks
- `success` - Achievements and checklists
- `note` - Additional information

## Examples

### Existing Documentation

Reference examples in the project:

- [docs/getting-started/account-setup/snowflake.md](../../../docs/getting-started/account-setup/snowflake.md)
- [docs/getting-started/account-setup/aws.md](../../../docs/getting-started/account-setup/aws.md)

### Usage Scenarios

**Creating account setup:**
```
Create documentation for GCP account setup
```

**Creating build guide:**
```
Write a guide for setting up Snowflake databases with Terraform
```

**Creating reference docs:**
```
Document our standard dbt project structure
```

**Updating existing docs:**
```
Add a section on warehouse auto-suspend configuration to the Snowflake warehouses guide
```

## Quality Standards

Every generated page is verified against:

**Language and Style:**
- [ ] British English throughout
- [ ] Second person, active voice
- [ ] No time estimates
- [ ] No subjective difficulty
- [ ] Explains why, not just how

**Structure:**
- [ ] One H1 page title
- [ ] Logical H2/H3 hierarchy
- [ ] Concepts before procedures
- [ ] Next steps with link

**Code and Examples:**
- [ ] Language identifiers on code blocks
- [ ] Expected output shown
- [ ] File paths specified
- [ ] Complete examples

**Links:**
- [ ] Internal links use relative paths
- [ ] External links have descriptive text
- [ ] No broken links

**Content:**
- [ ] Accurate and current
- [ ] Security best practices
- [ ] Real-world examples
- [ ] Troubleshooting guidance

## Output Location

Documentation is written to appropriate location:

```
docs/
├── getting-started/     # Initial setup and onboarding
├── build/               # Implementation guides
│   ├── infrastructure-as-code/
│   ├── data-warehouse/
│   ├── api-ingestion/
│   └── ...
└── maintain/            # Operations and troubleshooting
```

## Project Context

This skill follows the modern data stack project principles:

- **Incremental build** - Step-by-step progression
- **Production-grade** - No toy examples
- **Infrastructure as Code** - Version controlled
- **Security-first** - Best practices embedded
- **Educational** - Explain concepts and reasoning

See [CLAUDE.md](../../../CLAUDE.md) for full project philosophy.

## Quick Reference

Common style decisions:

| Category | Standard |
|----------|----------|
| Language | British English (organise, realise, colour) |
| Voice | Second person, active, present tense |
| Shell blocks | `sh` language identifier |
| File references | Inline code: `~/.aws/config` |
| Links | Descriptive text, relative paths internally |
| Admonitions | info, tip, warning, success, note |
| Headings | H1 once, H2 for sections, H3 for subsections |

See [STYLE-REFERENCE.md](STYLE-REFERENCE.md) for complete style guide.

## Maintenance

To update documentation standards:

1. Update SKILL.md for fundamental changes
2. Update STYLE-REFERENCE.md for specific style rules
3. Update this README for new capabilities
4. Regenerate existing docs if needed for consistency

## Version

Current version: 1.0.0
Last updated: 2026-01-17
