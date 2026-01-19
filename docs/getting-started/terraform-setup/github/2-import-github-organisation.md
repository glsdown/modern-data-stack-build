# Add GitHub to Terraform

On this page, you will:

- [x] Import GitHub organisation settings

## Navigate to Your Terraform Directory

```sh
cd ~/projects/data/data-stack-infrastructure/terraform/github
```

!!! note "Working Directory"
  All files noted below are inside this directory. You should replace `~/projects/data/data-stack-infrastructure` with the path to your project folder.

## Import GitHub Organisation

First, we'll import your GitHub organisation settings.

!!! warning "Organisation Must Exist"
    The `github_organization_settings` resource manages the *settings* of an existing organisation - it cannot create the organisation itself. You must create the GitHub organisation manually first (which you did in the [GitHub setup guide](../../initial-setup/1-github-setup.md)).

    Terraform will then manage the organisation's settings.

### Create Organisation Settings File

Create `organisation.tf`:

```hcl
# GitHub Organisation Settings
resource "github_organization_settings" "this" {
  # Required configuration
  billing_email     = var.github_billing_email

  # Optional
  name              = var.github_organization_name
  description       = var.github_organization_description

  # Optional but recommended settings
  default_repository_permission                   = "read"  # new members can view all repos
  members_can_create_private_repositories         = true    # team members can create repos
  members_can_create_public_repositories          = false   # prevent accidental public data
  members_can_create_public_pages                 = false
  dependabot_alerts_enabled_for_new_repositories  = true    # enable security scanning
  members_can_create_teams                        = false   # only admins create teams (via Terraform)
}
```

Add the required values to your variables files:

```hcl
# variables.tf
variable "github_billing_email" {
  description = "The billing email address for the GtHub organisation."
  type        = string
}
variable "github_organization_name" {
  description = "The name for the organization."
  type        = string
}
variable "github_organization_description" {
  description = "The description for the organization."
  type        = string
}

# terraform.tfvars
github_billing_email            = "name@your-company.com"
github_organization_name        = "My Company"
github_organization_description = "Description of the organisation"
```

!!! tip "Available options"
  You can see what options are available here by looking at the [docs](https://registry.terraform.io/providers/integrations/github/latest/docs/resources/organization_settings). We've used some basic standards, but you may want to change them, or add additional settings.

### Create Import Configuration File

Firstly, you need to retrieve your organisation ID. To do that, run the following, and copy the response:

```sh
gh api orgs/your-organisation-name --jq '.id'
```

You can press Ctrl + C to exit the command. Add this to your variables files:

```hcl
# variables.tf
variable "github_organization_id" {
  description = "GitHub organisation ID"
  type        = int
}

# terraform.tfvars
github_organization_id = 123456 # Replace with the id retrieved above
```

Now, create `imports.tf` and add the organisation import:

```hcl
# Import block - tells Terraform where to find the existing organisation
import {
  to = github_organization_settings.this
  id = var.github_organization_id
}
```

### Plan the Import

```sh
terraform plan
```

Expected output:

```
github_organization_settings.org: Preparing import... [id=your-org-name]
github_organization_settings.org: Refreshing state... [id=your-org-name]

Terraform will perform the following actions:

  # github_organization_settings.org will be imported
    resource "github_organization_settings" "org" {
        billing_email = "your.email@company.com"
        ...
    }

Plan: 1 to import, 0 to add, 0 to change, 0 to destroy.
```

If you see any differences (indicated by `~` symbols), update your `organisation.tf` to match the required settings.

### Apply the Import

```sh
terraform apply
```

Type `yes` when prompted.

Expected output:

```
github_organization_settings.org: Importing... [id=your-org-name]
github_organization_settings.org: Import complete [id=your-org-name]

Apply complete! Resources: 1 imported, 0 added, 0 changed, 0 destroyed.
```

## Commit your work

Make sure to commit your work - remember commit frequently. You need to check you are on the correct branch, which you can do at any time by running `gst` if you haven't set up your command prompt to include the current branch.

## Troubleshooting

### Error: Resource already exists

If you see:

```
Error: Resource already exists
```

You likely forgot to add an `import` block for an existing resource. Add the import block, run `terraform plan`, then `terraform apply`.

### Error: 404 Not Found

If you see:

```
Error: GET https://api.github.com/orgs/your-org/teams/team-name: 404 Not Found
```

Check:
1. Team name is correct (case-sensitive)
2. `GITHUB_TOKEN` has `admin:org` scope
3. You're authenticated to the correct organisation

### Error: Insufficient permissions

```
Error: PATCH https://api.github.com/orgs/...: 403 Forbidden
```

Your GitHub token needs broader permissions. Regenerate with `admin:org` scope.

## What's Next

You've successfully imported GitHub resources and created new ones with Terraform:

- ✅ Organisation settings managed in code
- ✅ All changes version-controlled
- ✅ Organisational structure is now repeatable and auditable

Continue to [import your teams](3-import-github-teams.md) →
