---
name: generate-docs
description: Generate production-ready documentation pages following the project's established style, structure, and British English conventions. Use when creating new documentation pages, guides, or technical content for the modern data stack project.
allowed-tools: Read, Write, Glob, Grep
---

# Generate Documentation

This Skill generates production-ready documentation pages that follow the modern data stack project's established standards.

## When to Use This Skill

Use this Skill when creating any new documentation page:

- Getting started guides (account setup, environment configuration)
- Build guides (infrastructure, data warehouse, pipelines)
- Maintenance guides (operations, troubleshooting)
- Reference documentation (API docs, configuration)
- Conceptual explanations (architecture, design decisions)
- How-to guides (specific tasks, workflows)

## Documentation Philosophy

Before generating any documentation, understand the project's core principles from [CLAUDE.md](../../../CLAUDE.md):

- **Incremental and practical** - Build step by step, real-world solutions
- **Production-grade from day one** - No toy examples or shortcuts
- **Educational** - Explain concepts and reasoning, not just commands
- **British English** - Consistent spelling and terminology
- **Security-first** - Best practices embedded throughout
- **Infrastructure as Code** - Everything version-controlled and reproducible

## Standard Structure

Every documentation page should follow this pattern:

```markdown
# Page Title (H1 - One Per Page)

[Optional: Brief introduction explaining what this page covers]

On this page, you will:

- [x] First thing they'll accomplish
- [x] Second thing they'll accomplish
- [x] Third thing they'll accomplish

## Main Section (H2)

[Content explaining concepts before diving into procedures]

### Subsection (H3)

[Detailed steps or explanations]

!!! info "Additional Context"
    [Background information or explanation]

### Another Subsection (H3)

[More content]

## Next Main Section (H2)

[Continue the logical flow]

## Best Practices / Summary (H2)

!!! success "What You've Accomplished"
    - [x] Achievement 1
    - [x] Achievement 2

## Next Steps (H2)

!!! success
    Brief congratulatory message about what they've achieved.

Your next step is to [logical next action]. This ensures:

- Benefit or reason 1
- Benefit or reason 2

Continue to [Next Page](../path/to/next.md) →
```

## Writing Style Requirements

### Language: British English

Always use British spellings and conventions:

| American | British |
|----------|---------|
| customize | customise |
| organize | organise |
| analyze | analyse |
| realize | realise |
| color | colour |
| center | centre |
| license (verb) | licence (verb) |

### Tone: Educational and Explanatory

- **Explain the "why"** not just the "how"
- Assume readers are intelligent but may be new to concepts
- Provide context before procedures
- Use analogies when helpful
- Avoid jargon without explanation

**Good examples:**
```markdown
✅ "Role-based access control (RBAC) separates permissions from users.
    This means you grant permissions to roles, then assign roles to users.
    When someone leaves, you simply remove their role assignment rather
    than updating permissions across multiple resources."

✅ "Multi-factor authentication (MFA) adds a second verification step
    beyond passwords. Even if someone obtains your password, they cannot
    access your account without the second factor (typically a code from
    your phone)."
```

**Bad examples:**
```markdown
❌ "Obviously, you should use RBAC."
❌ "Simply enable MFA."
❌ "This is easy - just follow these steps."
```

### Voice: Second Person, Active, Present Tense

- Use "you" to address the reader
- Active voice for instructions
- Present tense for descriptions

**Examples:**
```markdown
✅ "You will create a new role"
✅ "Navigate to the IAM console"
✅ "This ensures secure access"

❌ "The user should navigate to the IAM console"
❌ "This will ensure secure access"
❌ "A new role can be created"
```

### Avoid Time Estimates

Never mention how long something takes:

```markdown
❌ "This will take about 5 minutes"
❌ "This is a quick process"
❌ "This should be done in a few hours"

✅ "Follow these steps to enable MFA"
✅ "Complete the configuration"
```

### Avoid Subjective Difficulty

Let readers judge difficulty themselves:

```markdown
❌ "This is simple"
❌ "This is complex"
❌ "Obviously..."
❌ "Simply..."
❌ "Just..."

✅ "Follow these steps"
✅ "This requires several configuration steps"
```

## Content Structure Patterns

### Concept Before Procedure

Always explain concepts before providing instructions:

```markdown
## Understanding Snowflake Warehouses

Snowflake warehouses are compute resources that execute queries. Unlike
traditional databases where storage and compute are coupled, Snowflake
separates them. You can pause warehouses when not in use, paying only
for storage.

### Create Your First Warehouse

Now that you understand what warehouses do, let's create one:

1. Navigate to Admin > Warehouses
2. Click "Create Warehouse"
...
```

### Progressive Disclosure

Start simple, add complexity gradually:

```markdown
## Basic Configuration

[Essential setup that everyone needs]

## Advanced Configuration (Optional)

[Additional options for specific use cases]

!!! tip "When to Use Advanced Features"
    [Guidance on when these are needed]
```

### Security Throughout, Not Afterthought

Integrate security into each section, not as a separate "Security" chapter:

```markdown
## Create Your Admin User

### Enable MFA Immediately

As soon as you create the admin user, enable MFA. This is critical for
accounts with administrative privileges.

[MFA setup steps]

### Set a Safe Default Role

[Default role configuration]
```

## Admonitions (Info Boxes)

Use MkDocs admonition syntax for callouts:

### Info Boxes - Background and Context

```markdown
!!! info "About the Root User"
    The email you provide creates the **root user** - the most powerful
    account in AWS with unrestricted access to all services and billing.
```

Use when:
- Explaining concepts or terminology
- Providing background context
- Offering alternatives
- Describing how something works

### Tips - Helpful Advice

```markdown
!!! tip "Choosing a Region"
    Common choices for UK-based companies:

    - **eu-west-2 (London)** - lowest latency for UK users
    - **eu-west-1 (Ireland)** - larger region, more services
```

Use when:
- Sharing best practices
- Recommending approaches
- Providing shortcuts or efficiency tips
- Suggesting alternatives

### Warnings - Critical Information

```markdown
!!! warning "Store Recovery Codes"
    When setting up MFA, AWS provides recovery codes. Store these securely
    in a password manager. If you lose your MFA device and recovery codes,
    you cannot access your account.
```

Use when:
- Highlighting security risks
- Noting irreversible actions
- Warning about common mistakes
- Emphasizing critical requirements

### Success Boxes - Achievements and Summaries

```markdown
!!! success "Security Checklist"
    - [x] Root user secured with MFA
    - [x] IAM users created
    - [x] Role-based access configured
```

Use when:
- Confirming completion
- Providing validation checklists
- Summarising achievements
- Ending sections with what was accomplished

### Note Boxes - Additional Information

```markdown
!!! note "Future Enhancements"
    You may need to add more roles as your team scales. We'll cover
    better role management patterns in the Terraform section.
```

Use when:
- Noting future considerations
- Referencing other sections
- Providing supplementary information
- Explaining limitations

### Danger Boxes - Severe Risks

```markdown
!!! danger "Use With Caution"
    Account-level policies affect all users. Ensure you have at least one
    admin account that can connect before setting this, or you risk locking
    yourself out.
```

Use when:
- Warning about potential lockouts
- Highlighting destructive or irreversible actions
- Emphasising severe security risks
- Noting actions that could break access

## Tabbed Content

Use Material for MkDocs tabbed content when instructions differ by platform or tool:

```markdown
=== "AWS S3"

    ### Step 1: Create the IAM Role

    First, create an IAM role in AWS that Snowflake will assume.

    \`\`\`hcl
    resource "aws_iam_role" "snowflake_storage" {
      name = "snowflake-storage-access"
      # ...
    }
    \`\`\`

=== "Google Cloud Storage"

    ### Step 1: Create the Snowflake Integration

    For GCS, you create the integration first, then grant access.

    \`\`\`hcl
    module "integration_data_lake" {
      source = "./modules/snowflake_storage_integration"
      # ...
    }
    \`\`\`
```

Use tabs when:
- Instructions differ significantly between cloud providers (AWS, GCP, Azure)
- Steps vary by operating system (macOS, Linux, Windows)
- Configuration differs by identity provider (Okta, Azure AD, Google Workspace)
- Multiple tool options exist for the same task

Guidelines:
- Keep tab names short and descriptive
- Ensure equivalent content coverage across all tabs
- Indent all content within tabs with 4 spaces
- Don't use tabs for minor differences - use inline notes instead

## Code and Commands

### Bash/Shell Commands

Always use `sh` language identifier and include expected output:

```markdown
\`\`\`sh
aws --version
# Expected output: aws-cli/2.x.x Python/3.x.x Darwin/23.x.x
\`\`\`
```

For commands with no output:

```markdown
\`\`\`sh
aws configure --profile default
# You'll be prompted for:
# AWS Access Key ID: [your access key]
# AWS Secret Access Key: [your secret key]
\`\`\`
```

For multi-step sequences:

```markdown
\`\`\`sh
# Back up existing configuration
cp ~/.aws/config ~/.aws/config.backup

# Create new configuration
cat > ~/.aws/config <<EOF
[default]
region = eu-west-2
EOF
\`\`\`
```

### Configuration Files

Specify file path and action (create, edit, append):

```markdown
Edit `~/.aws/config`:

\`\`\`ini
[default]
region = eu-west-2
output = json
\`\`\`
```

Or for new files:

```markdown
Create `~/.config/tool/config.yaml`:

\`\`\`yaml
version: 1
settings:
  enabled: true
\`\`\`
```

### SQL Queries

```markdown
Run the following query (replace `username` with your actual username):

\`\`\`sql
USE ROLE ORGADMIN;
ALTER USER username SET DEFAULT_ROLE = 'PUBLIC';
\`\`\`
```

### Python/Programming Code

Include context and purpose:

```markdown
Create a simple ingestion pipeline with dlt:

\`\`\`python
import dlt
from dlt.sources import rest_api

# Define the source
@dlt.source
def github_api():
    return rest_api({
        "client": {
            "base_url": "https://api.github.com/"
        },
        "resources": ["repos/anthropics/claude-code"]
    })

# Load to Snowflake
pipeline = dlt.pipeline(
    pipeline_name="github_data",
    destination="snowflake",
    dataset_name="raw_github"
)

load_info = pipeline.run(github_api())
print(load_info)
\`\`\`
```

### Terraform Configuration

Include module structure and purpose:

```markdown
Create the Snowflake warehouse resource in `warehouses.tf`:

\`\`\`hcl
resource "snowflake_warehouse" "analytics_wh" {
  name           = "ANALYTICS_WH"
  warehouse_size = "X-Small"
  auto_suspend   = 60  # seconds
  auto_resume    = true

  comment = "Warehouse for analytics queries"
}
\`\`\`
```

## Lists and Formatting

### Numbered Lists - Sequential Steps

Use for procedures that must be followed in order:

```markdown
1. Navigate to **IAM** service
2. Click "Roles" in the left sidebar
3. Click "Create role"
4. Select **"AWS account"** as the trusted entity type
5. Click "Next"
```

### Bullet Points - Non-Sequential Information

Use for options, features, or unordered information:

```markdown
Common choices for UK-based companies:

- **eu-west-2 (London)** - lowest latency for UK users
- **eu-west-1 (Ireland)** - larger region, more services
- **us-east-1 (N. Virginia)** - newest features launch here first
```

### Emphasis

**Bold** for:
- UI element names: Click the **Create role** button
- Critical emphasis: You should **never** commit credentials
- Role/service names: **AdminRole**, **DataEngineerRole**

**Inline code** for:
- File paths: `~/.aws/config`
- Commands: `terraform init`
- Variables: `AWS_PROFILE`
- Configuration values: `eu-west-2`

**Avoid italics** - rarely needed in technical documentation

## Links and References

### Internal Links (Within Documentation)

Use relative paths:

```markdown
See [Database Configuration](../../build/data-warehouse/3-databases.md) for details.

Continue to [Set Up Terraform](../../build/infrastructure-as-code/terraform-fundamentals.md) →
```

Add trailing arrow (`→`) for "next steps" links.

### External Links

Always use descriptive link text, never "here" or "click here":

```markdown
✅ Learn more about [AWS regions](https://aws.amazon.com/about-aws/global-infrastructure/)
✅ Read the [Snowflake password policy documentation](https://docs.snowflake.com/passwords)

❌ Click [here](https://aws.amazon.com/regions/) for more information
❌ Read more [here](https://docs.snowflake.com/passwords)
```

### Code References

When referencing files in a repository, use descriptive inline references rather than linking to external files:

```markdown
See the working example in `snowflake/config/warehouses.tf` in the terraform repository.
```

## Page-Specific Patterns

### Getting Started Pages

Structure:
1. What you'll accomplish (checklist)
2. Prerequisites (if any)
3. Concepts/background
4. Step-by-step procedures
5. Verification/testing
6. Best practices summary
7. Next steps

### Build Guides

Structure:
1. What you'll build
2. Prerequisites
3. Architecture/design explanation
4. Implementation (with code)
5. Testing/validation
6. Troubleshooting common issues
7. Next steps

### Reference Documentation

Structure:
1. Overview and purpose
2. Quick reference (tables, lists)
3. Detailed descriptions
4. Examples
5. API/configuration reference
6. Related resources

### Troubleshooting Guides

Structure:
1. Problem description
2. Common symptoms
3. Diagnostic steps
4. Solutions (ordered by likelihood)
5. Prevention tips
6. When to escalate

## Quality Checklist

Before finalizing any documentation, verify:

**Language and Style:**
- [ ] British English throughout (organise, realise, colour)
- [ ] Second person, active voice
- [ ] No time estimates mentioned
- [ ] No subjective difficulty claims
- [ ] Explains "why" not just "how"

**Structure:**
- [ ] One H1 page title
- [ ] Logical H2/H3 hierarchy
- [ ] Checklist at top (for procedural docs)
- [ ] Concepts explained before procedures
- [ ] Next steps section with link

**Code and Examples:**
- [ ] Code blocks have language identifiers
- [ ] Shell commands show expected output
- [ ] File paths specified for configurations
- [ ] Examples are complete and tested
- [ ] Placeholder values clearly marked

**Links and References:**
- [ ] Internal links use relative paths
- [ ] External links have descriptive text
- [ ] No broken links
- [ ] Trailing arrow on "next steps" links

**Admonitions:**
- [ ] Used appropriately (not overused)
- [ ] Correct type for content (info, tip, warning, danger, success, note)
- [ ] Meaningful titles when needed

**Content Quality:**
- [ ] Accurate and up-to-date
- [ ] Security best practices included
- [ ] Troubleshooting guidance provided
- [ ] Real-world examples, not toy examples

## Generation Process

When generating documentation:

1. **Understand the context**
   - Read [CLAUDE.md](../../../CLAUDE.md) for project philosophy
   - Read similar existing pages for style reference
   - Understand where this page fits in the documentation structure

2. **Research if needed**
   - Review official documentation for the subject
   - Understand current best practices
   - Verify technical accuracy

3. **Plan the structure**
   - Determine page type (getting started, build, reference, etc.)
   - Outline main sections
   - Identify key concepts to explain

4. **Generate content**
   - Follow appropriate structure pattern
   - Use British English
   - Include admonitions appropriately
   - Add code examples with output
   - Explain concepts before procedures

5. **Review against checklist**
   - Verify quality checklist items
   - Check links work
   - Ensure consistent style
   - Validate code examples

6. **Write to correct location**
   - `docs/getting-started/` for initial setup
   - `docs/build/` for implementation guides
   - `docs/maintain/` for operational guides

## Response Format

When invoked:

1. Confirm what you're creating
2. Read relevant reference documentation
3. Generate complete content
4. Write to appropriate location
5. Provide brief summary of what was included
