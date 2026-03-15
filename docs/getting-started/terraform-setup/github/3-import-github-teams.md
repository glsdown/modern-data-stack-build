# Add GitHub to Terraform

On this page, you will:

- [x] Import existing teams (data-platform-admins, data-engineers)
- [x] Create a new data-analysts team via Terraform

## Navigate to Your Terraform Directory

```sh
cd ~/projects/data/data-stack-infrastructure/terraform/github
```

!!! note "Working Directory"
  All files noted below are inside this directory. You should replace `~/projects/data/data-stack-infrastructure` with the path to your project folder.

## Import GitHub Teams

Now import the two teams you created manually: `data-platform-admins` and `data-engineers`.

### Add Team Variables

First, define variables for your teams.

Add to `variables.tf`:

```hcl
# Teams
variable "teams" {
  description = "Map of teams to create/manage"
  type = map(object({
    description = string
    privacy     = string
  }))
  default = {}
}
```

Add to `terraform.tfvars`:

```hcl
# Teams
teams = {
  "data-platform-admins" = {
    description = "Full access to data infrastructure"
    privacy     = "closed"
  }
  "data-engineers" = {
    description = "Data Engineers"
    privacy     = "closed"
  }
}
```

### Create Teams Configuration File

Create `teams.tf`:

```hcl
# Teams
# Note: Teams are automatically associated with the organization specified
# in providers.tf (owner = var.github_organization)

resource "github_team" "teams" {
  for_each = var.teams

  name        = each.key
  description = each.value.description
  privacy     = each.value.privacy
}
```

This uses `for_each` to create one team resource per entry in the `teams` variable. The team name (e.g., "data-platform-admins") becomes the map key, which we can use to reference the team later.

### Add Team Import Blocks

Add to `imports.tf`:

```hcl
# Import existing teams
import {
  to = github_team.teams["data-platform-admins"]
  id = "data-platform-admins"
}

import {
  to = github_team.teams["data-engineers"]
  id = "data-engineers"
}
```

!!! note "Team Configuration"
    **Organisation**: Teams are automatically associated with the organisation specified in `providers.tf` (`owner = var.github_organization`). You don't need to specify the organisation for each team.

    **Privacy options**:
    - **closed**: Team members can see who's in the team, but it's not publicly visible
    - **secret**: Only team members know the team exists

### Plan and Apply

```sh
terraform plan
```

Review the plan - you should see 2 imports with no changes.

```sh
terraform apply
```

Expected output:

```
github_team.teams["data-platform-admins"]: Importing... [id=data-platform-admins]
github_team.teams["data-engineers"]: Importing... [id=data-engineers]

Apply complete! Resources: 2 imported, 0 added, 0 changed, 0 destroyed.
```

## Create a New Team with Terraform

Now let's create a new team entirely through Terraform: `data-analysts`.

### Add Data Analysts Team to Variables

Simply add the new team to `terraform.tfvars`:

```hcl
# Teams
teams = {
  "data-platform-admins" = {
    description = "Full access to data infrastructure"
    privacy     = "closed"
  }
  "data-engineers" = {
    description = "Data Engineers"
    privacy     = "closed"
  }
  "data-analysts" = {
    description = "Data Analysts"
    privacy     = "closed"
  }
}
```

!!! note "No Import Block Needed"
    Notice we don't add an `import` block for this team. That's because it doesn't exist yet in GitHub - Terraform will create it from scratch.

!!! tip "No teams.tf Changes Needed"
    Because we're using `for_each` in the `github_team.teams` resource, we don't need to modify `teams.tf` at all. Adding the team to `terraform.tfvars` is sufficient - the `for_each` loop will automatically create a resource for it.

### Plan the Change

```sh
terraform plan
```

Expected output:

```
Terraform will perform the following actions:

  # github_team.teams["data-analysts"] will be created
  + resource "github_team" "teams" {
      + description = "Data Analysts"
      + id          = (known after apply)
      + name        = "data-analysts"
      + privacy     = "closed"
      + slug        = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

### Apply the Change

```sh
terraform apply
```

Expected output:

```
github_team.teams["data-analysts"]: Creating...
github_team.teams["data-analysts"]: Creation complete after 1s [id=data-analysts]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

## Verify in GitHub

Check that everything is managed correctly, go to your organisation's **Teams** page and verify:

- `data-platform-admins` exists
- `data-engineers` exists
- `data-analysts` exists (newly created)

## Commit your work

Make sure to commit your work - remember commit frequently. You need to check you are on the correct branch, which you can do at any time by running `gst` if you haven't set up your command prompt to include the current branch.

## What's Next

You've successfully imported GitHub resources and created new ones with Terraform:

- ✅ Teams managed in code
- ✅ New data-analysts team created
- ✅ All changes version-controlled

Continue to [manage your users](4-github-user-management.md) →
