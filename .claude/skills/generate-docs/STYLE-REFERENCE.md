# Style Reference

Quick reference for documentation style decisions in the modern data stack project.

## British vs American English

| American | British | Example Usage |
|----------|---------|---------------|
| customize | customise | "Customise your Terraform configuration" |
| organize | organise | "Organise your files into modules" |
| analyze | analyse | "Analyse the query performance" |
| realize | realise | "You'll realise the benefits of..." |
| optimize | optimise | "Optimise warehouse sizing" |
| authorize | authorise | "Authorise users to access data" |
| color | colour | "Choose a colour for the role" |
| favor | favour | "We favour explicit configuration" |
| center | centre | "The data centre is located in..." |
| license (v) | licence (v) | "You must licence the software" |
| license (n) | license (n) | "The Apache 2.0 license" |
| defense | defence | "Defence against SQL injection" |
| catalog | catalogue | "The data catalogue shows..." |
| meter | metre | "Per-metre pricing" |
| liter | litre | "Per-litre storage costs" |

## Voice and Tense

| Use | Example |
|-----|---------|
| Second person | "You will create a role" |
| Active voice | "Configure the warehouse" not "The warehouse should be configured" |
| Present tense | "This ensures security" not "This will ensure security" |
| Imperative for instructions | "Navigate to IAM" not "You should navigate to IAM" |

## Admonition Types

| Type | Purpose | Example Title |
|------|---------|---------------|
| `!!! info` | Background, context, explanations | "About Snowflake Editions", "Why Use RBAC" |
| `!!! tip` | Best practices, recommendations | "Choosing a Region", "Keyboard Shortcuts" |
| `!!! warning` | Critical info, risks, irreversible actions | "Store Recovery Codes", "Cannot Be Undone" |
| `!!! danger` | Severe risks, potential lockouts, destructive actions | "Use With Caution", "Risk of Lockout" |
| `!!! success` | Achievements, completions, checklists | "What You've Accomplished", "Security Checklist" |
| `!!! note` | Additional info, future considerations | "Future Enhancements", "For Reference" |

## Code Block Languages

| Content Type | Language Identifier | Example |
|--------------|-------------------|---------|
| Shell/Bash commands | `sh` | ````md ```sh```` |
| SQL queries | `sql` | ````md ```sql```` |
| Python code | `python` | ````md ```python```` |
| Terraform | `hcl` | ````md ```hcl```` |
| YAML config | `yaml` | ````md ```yaml```` |
| JSON | `json` | ````md ```json```` |
| INI config | `ini` | ````md ```ini```` |
| Markdown examples | `markdown` | ````md ```markdown```` |
| Plain text output | `text` or none | ````md ```text```` |

## Heading Hierarchy

| Level | Usage | Example |
|-------|-------|---------|
| H1 (`#`) | Page title only (one per page) | `# Create an AWS Account` |
| H2 (`##`) | Main sections | `## Secure the Root User` |
| H3 (`###`) | Subsections | `### Enable MFA` |
| H4+ | Avoid if possible, restructure content | Don't use |

## Emphasis

| Style | Usage | Example |
|-------|-------|---------|
| **Bold** | UI elements, critical emphasis, names | Click **Create role**, **never** commit credentials |
| `inline code` | File paths, commands, variables, values | `~/.aws/config`, `terraform init`, `AWS_PROFILE` |
| _Italics_ | Avoid in technical docs | Rarely needed |

## Link Patterns

| Type | Pattern | Example |
|------|---------|---------|
| Internal | Relative path | `[Terraform Setup](../../build/infrastructure-as-code/terraform.md)` |
| Next steps | Relative path with arrow | `Continue to [Next Page](./next.md) →` |
| External | Full URL with descriptive text | `[AWS IAM best practices](https://docs.aws.amazon.com/iam/best-practices)` |
| Code reference | Relative path to repo file | `[warehouses.tf](../../../repositories/terraform/snowflake/config/warehouses.tf)` |

## List Types

| Use For | Example |
|---------|---------|
| Sequential steps | `1. Navigate to IAM`<br>`2. Click Roles` |
| Options/features | `- Option A`<br>`- Option B` |
| Checklists | `- [x] Task complete`<br>`- [ ] Task pending` |

## Tabbed Content

Use Material for MkDocs tabbed content for platform-specific instructions:

```markdown
=== "AWS S3"

    Content for AWS users goes here.

    - Indented with 4 spaces
    - Can include code blocks, lists, etc.

=== "Google Cloud Storage"

    Content for GCP users goes here.
```

| Use For | Example Tabs |
|---------|--------------|
| Cloud provider variants | "AWS S3", "Google Cloud Storage", "Azure Blob" |
| Operating system instructions | "macOS", "Linux", "Windows" |
| Identity provider setup | "Okta", "Azure AD", "Google Workspace" |
| Tool alternatives | "Homebrew", "apt", "Manual Install" |

Guidelines:
- Use when content differs significantly between options
- Keep tab names short and descriptive
- Ensure all tabs have equivalent content coverage
- Indent content with 4 spaces under each tab

## Common Patterns

### File Operation Instructions

| Action | Pattern |
|--------|---------|
| Create file | "Create `~/.aws/config`:" |
| Edit file | "Edit `~/.aws/config`:" |
| Append to file | "Add to `~/.zshrc`:" |
| Read file | "View `~/.aws/config`:" |

### Command Output

```markdown
\`\`\`sh
command --to-run
# Expected output: expected result
\`\`\`
```

Or for JSON/structured output:

```markdown
\`\`\`sh
aws sts get-caller-identity
\`\`\`

Expected output:

\`\`\`json
{
  "UserId": "AIDAEXAMPLE",
  "Account": "123456789012"
}
\`\`\`
```

### Placeholders

Use descriptive placeholders in angle brackets or square brackets:

```markdown
Replace `ACCOUNT_ID` with your 12-digit account ID

git clone git@github.com:<your-org>/<repo-name>.git

aws iam create-role --role-name [role-name]
```

## Phrases to Avoid

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| "Obviously..." | Assumes knowledge | Just explain it |
| "Simply..." | Minimises difficulty | "Follow these steps" |
| "Just..." | Minimises difficulty | "Configure..." |
| "This is easy" | Subjective | Remove entirely |
| "This is complex" | Subjective | "This requires several steps" |
| "This takes 5 minutes" | Time estimate | Remove entirely |
| "Click here" | Non-descriptive link | Descriptive link text |
| "As mentioned above" | Vague reference | Link to specific section |

## Consistent Terminology

Use these terms consistently:

| Concept | Preferred Term | Not |
|---------|---------------|-----|
| Command line | terminal, CLI | console, command prompt |
| Version control | Git, version control | source control |
| Environment variable | environment variable | env var, environment var |
| Configuration file | configuration file, config file | conf file, settings file |
| Directory | directory | folder (except for GUI contexts) |
| Repository | repository, repo | project |

## Numbers and Units

| Type | Format | Example |
|------|--------|---------|
| Small numbers | Spell out 1-10, digits for 11+ | "three roles", "12 users" |
| Large numbers | Use digits with separators | "10,000 records", "1.5 million" |
| Storage | Use powers of 2 | "1 GiB" not "1 GB" (unless service uses GB) |
| Money | Use appropriate currency symbol | "£100", "$50", "€75" |
| Percentages | Digit with % symbol | "80%", "99.9%" |

## Common Abbreviations

| Abbreviation | First Use | Subsequent |
|--------------|-----------|------------|
| MFA | multi-factor authentication (MFA) | MFA |
| RBAC | role-based access control (RBAC) | RBAC |
| CLI | command-line interface (CLI) | CLI |
| API | application programming interface (API) | API |
| IAM | Identity and Access Management (IAM) | IAM |
| SSO | single sign-on (SSO) | SSO |
| SAML | Security Assertion Markup Language (SAML) | SAML |

## Version References

| Context | Format | Example |
|---------|--------|---------|
| Specific version | Full version number | "Terraform 1.6.0" |
| Version range | Comparison or range | "Python 3.9 or later" |
| Approximate version | X.x.x format | "aws-cli/2.x.x" |
| Latest | "latest version" not "newest" | "Install the latest version" |

## Date and Time

| Context | Format | Example |
|---------|--------|---------|
| Full date | ISO 8601 | "2026-01-17" |
| Month-year | Full month name | "January 2026" |
| Time | 24-hour format | "14:30 UTC" |
| Duration | Explicit units | "60 seconds" not "60 secs" |
| Frequency | Full words | "daily" not "once per day" |

## Example Documentation Snippets

### Opening Checklist

```markdown
# Create an AWS Account

On this page, you will:

- [x] Create a new AWS account
- [x] Secure the root user with MFA
- [x] Create your personal IAM user
- [x] Configure AWS CLI for local development
```

### Concept Explanation

```markdown
## Understanding Snowflake Warehouses

Snowflake warehouses are compute resources that execute queries. Unlike traditional
databases where storage and compute are coupled, Snowflake separates them completely.

This separation means you can:

- Pause warehouses when not in use (paying only for storage)
- Run multiple warehouses simultaneously without contention
- Size each warehouse independently based on workload
```

### Procedural Steps

```markdown
### Create the AdminRole

1. Navigate to **IAM** service (search "IAM" in the top search bar)
2. Click "Roles" in the left sidebar
3. Click "Create role"
4. Select **"AWS account"** as the trusted entity type
5. Click "Next"
```

### Command with Output

```markdown
Test your configuration:

\`\`\`sh
aws sts get-caller-identity
\`\`\`

Expected output:

\`\`\`json
{
    "UserId": "AIDAEXAMPLEUSERID",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/jane.bloggs"
}
\`\`\`

Notice the ARN shows your user identity.
```

### Info Box

```markdown
!!! info "About the Root User"
    The email you provide creates the **root user** - the most powerful account
    in AWS with unrestricted access to all services and billing. You should rarely
    use this account for day-to-day operations.
```

### Warning

```markdown
!!! warning "Store Recovery Codes"
    When setting up MFA, AWS provides recovery codes. Store these securely in a
    password manager or encrypted document. If you lose your MFA device and
    recovery codes, you cannot access your account.
```

### Next Steps

```markdown
## Next Steps

!!! success
    You now have a secure AWS account with proper IAM configuration and CLI access!

Your next step is to configure Terraform to manage AWS infrastructure as code. This ensures
all future resources are:

- Version controlled
- Reproducible
- Peer reviewed
- Documented

Continue to [Terraform Fundamentals](../../build/infrastructure-as-code/terraform-fundamentals.md) →
```
