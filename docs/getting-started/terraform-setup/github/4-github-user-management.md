# Manage GitHub Users and Team Membership

On this page, you will:

- [x] Understand GitHub organisation membership vs team membership
- [x] Add organisation members via Terraform
- [x] Assign members to teams with validation
- [x] Learn about resource dependencies
- [x] Use dynamic references between resources
- [x] Set up CODEOWNERS for membership approval
- [x] Verify membership in GitHub

## Understanding GitHub Membership

GitHub has two levels of membership:

1. **Organisation Membership**: Users who are members of your GitHub organisation
2. **Team Membership**: Organisation members assigned to specific teams

You must first add users as organisation members before you can assign them to teams. This creates a **dependency** that Terraform needs to understand.

## Organisation Members vs Team Members

```
┌─────────────────────────────────────────────────────┐
│ GitHub Organisation: your-org                       │
│                                                     │
│  Organisation Members (GitHub usernames):           │
│  - your-admin-username (admin)                      │
│  - your-engineer-username (member)                  │
│  - your-analyst-username (member)                   │
│                                                     │
│  ┌────────────────────────────────────────┐         │
│  │ Team: data-platform-admins             │         │
│  │ Members: your-admin-username           │         │
│  └────────────────────────────────────────┘         │
│                                                     │
│  ┌────────────────────────────────────────┐         │
│  │ Team: data-engineers                   │         │
│  │ Members: your-admin-username,          │         │
│  │          your-engineer-username        │         │
│  └────────────────────────────────────────┘         │
│                                                     │
│  ┌────────────────────────────────────────┐         │
│  │ Team: data-analysts                    │         │
│  │ Members: your-analyst-username         │         │
│  └────────────────────────────────────────┘         │
│                                                     │
└─────────────────────────────────────────────────────┘
```

## Add Organisation Members

First, define variables for your organisation members.

### Update Variables

Add to `variables.tf`:

```hcl
# Organisation Members
variable "organization_members" {
  description = "Map of organisation members with their roles"
  type = map(object({
    role = string # "admin" or "member"
  }))
  default = {}
}
```

This uses a **map of objects** structure, which allows you to define multiple members with their properties in a clean way.

### Update Variable Values

Create a new file `members.auto.tfvars` for organisation membership configuration:

```hcl
# Organisation Members
# Replace these example usernames with actual GitHub usernames from your organisation
organization_members = {
  "actual-github-username-1" = {
    role = "admin"  # Organisation admin
  }
  "actual-github-username-2" = {
    role = "member"  # Regular member
  }
  "actual-github-username-3" = {
    role = "member"  # Regular member
  }
}
```

!!! warning "Use Real GitHub Usernames"
    **IMPORTANT**: Replace `actual-github-username-1`, `actual-github-username-2`, etc. with the real GitHub usernames of people in your organisation. These usernames must:

    - Be existing GitHub accounts
    - Match exactly (case-sensitive)
    - Be the username, not email address

    You can find a user's GitHub username on their GitHub profile at `https://github.com/username`.

!!! tip "Auto-Loaded tfvars File"
    We use `members.auto.tfvars` (note the `.auto.` part) because:

    - Terraform automatically loads `*.auto.tfvars` files
    - You can add it to CODEOWNERS requiring admin team approval for changes
    - It keeps sensitive membership data separate from general configuration
    - Different team members can manage different tfvars files

!!! note "Automatic Loading"
    Terraform automatically loads files in this order:

    1. `terraform.tfvars` (if present)
    2. `*.auto.tfvars` (in alphabetical order)
    3. Any `-var-file` flags you specify

    By naming it `members.auto.tfvars`, you never need to remember the `-var-file` flag!

### Create Organisation Members Resource

Create a new file `members.tf`:

```hcl
# Organisation Members
# Add users to the organisation before assigning them to teams

resource "github_membership" "members" {
  for_each = var.organization_members

  username = each.key
  role     = each.value.role
}
```

This uses the same `for_each` pattern we introduced in section 3 for teams. It creates one membership resource per member, using the GitHub username as the resource key (e.g., `github_membership.members["your-admin-username"]`).

## Assign Members to Teams

Now we'll assign organisation members to teams. This requires referencing both the team resources and the membership resources.

### Define Team Memberships

Add to `variables.tf`:

```hcl
# Team Memberships
variable "team_memberships" {
  description = "Map of team memberships"
  type = map(object({
    team_members = list(string)
  }))
  default = {}
}
```

### Update Variable Values

Add team membership configuration to the same `members.auto.tfvars` file:

```hcl
# Organisation Members
# Replace these example usernames with actual GitHub usernames from your organisation
organization_members = {
  "actual-github-username-1" = {
    role = "admin"
  }
  "actual-github-username-2" = {
    role = "member"
  }
  "actual-github-username-3" = {
    role = "member"
  }
}

# Team Memberships
# Assign organisation members to teams
# IMPORTANT: All usernames here MUST also be in organization_members above
team_memberships = {
  "data-platform-admins" = {
    team_members = ["actual-github-username-1"]  # Admin user
  }
  "data-engineers" = {
    team_members = ["actual-github-username-1", "actual-github-username-2"]
  }
  "data-analysts" = {
    team_members = ["actual-github-username-3"]
  }
}
```

!!! warning "Team Members Must Be Organisation Members"
    Every username in `team_members` must also exist in `organization_members`. If you try to add someone to a team who isn't an organisation member, Terraform will fail. We'll add validation to catch this error early.

!!! tip "Complete members.auto.tfvars File"
    Your complete `members.auto.tfvars` file should now contain both organisation members and team memberships as shown above.

### Create Team Membership Resources

Add to `teams.tf`:

```hcl
# Team Memberships
# Assign organisation members to teams

locals {
  # Flatten team memberships into a list of (team, member) pairs
  team_member_pairs = flatten([
    for team_name, config in var.team_memberships : [
      for member in config.team_members : {
        team   = team_name
        member = member
      }
    ]
  ])

  # Get all unique team members across all teams
  all_team_members = toset(flatten([
    for team, config in var.team_memberships : config.team_members
  ]))

  # Validate: all team members must be organisation members
  invalid_team_members = setsubtract(
    local.all_team_members,
    keys(var.organization_members)
  )
}

# Validation: Ensure all team members are organisation members
resource "terraform_data" "validate_team_members" {
  lifecycle {
    precondition {
      condition     = length(local.invalid_team_members) == 0
      error_message = "The following users are assigned to teams but not in organization_members: ${join(", ", local.invalid_team_members)}. Add them to organization_members in members.auto.tfvars first."
    }
  }
}

resource "github_team_membership" "memberships" {
  for_each = {
    for pair in local.team_member_pairs :
    "${pair.team}-${pair.member}" => pair
  }

  team_id  = github_team.teams[each.value.team].id
  username = each.value.member
  role     = "member"

  # Ensure the user is an organisation member first
  depends_on = [github_membership.members, terraform_data.validate_team_members]
}
```

This configuration adds:

1. **Validation check**: `terraform_data.validate_team_members` ensures every user in `team_memberships` exists in `organization_members`
2. **Clear error messages**: If validation fails, you'll see exactly which usernames are missing
3. **Early failure**: Terraform will catch the error during plan, before trying to create resources

This is more complex, so let's break it down:

### Understanding the Flattening Logic

**Step 1: Input Data Structure**

From your `members.tfvars`:
```hcl
team_memberships = {
  "data-engineers" = {
    team_members = ["your-admin-username", "your-engineer-username"]
  }
  "data-analysts" = {
    team_members = ["your-analyst-username"]
  }
}
```

**Step 2: Flatten to Pairs**

The `local.team_member_pairs` transforms this into:
```hcl
[
  { team = "data-engineers", member = "your-admin-username" },
  { team = "data-engineers", member = "your-engineer-username" },
  { team = "data-analysts", member = "your-analyst-username" }
]
```

**Step 3: Create Unique Resource Keys**

The `for_each` then creates:
```
github_team_membership.memberships["data-engineers-your-admin-username"]
github_team_membership.memberships["data-engineers-your-engineer-username"]
github_team_membership.memberships["data-analysts-your-analyst-username"]
```

### Understanding Resource Dependencies

Notice these important connections:

1. **Explicit Dependency**: `depends_on = [github_membership.members]`
   - Ensures users are organisation members before adding them to teams
   - Terraform waits for all memberships to be created first

2. **Implicit Dependency**: `team_id = github_team.teams[each.value.team].id`
   - References the team resource directly
   - Terraform knows the team must exist before creating the membership
   - Uses dynamic reference to get the team ID

### How Dynamic References Work

The expression `github_team.teams[each.value.team].id` works like this:

1. `github_team.teams` - refers to the `github_team` resource with `for_each` (created in section 3)
2. `[each.value.team]` - looks up the specific team by its GitHub team name from tfvars (e.g., "data-engineers")
3. `.id` - gets the team's ID attribute

**Important**: The `each.value.team` is the GitHub team name (e.g., "data-engineers") which matches the key in the `teams` variable from `terraform.tfvars`. Because we used `for_each = var.teams` with the team name as the key in section 3, we can now reference teams by their GitHub name: `github_team.teams["data-engineers"]`.

This is much better than hardcoding team IDs because:
- Works automatically when teams are created
- No manual lookup required
- Changes propagate automatically
- The team name in tfvars directly maps to the resource reference

## Plan and Apply

### Review the Plan

```sh
terraform plan
```

!!! tip "Auto-Loading of tfvars Files"
    Because we named the file `members.auto.tfvars`, Terraform automatically loads it - no `-var-file` flag needed! Terraform loads files in this order:

    1. `terraform.tfvars` (general configuration)
    2. `members.auto.tfvars` (membership configuration)
    3. Any `-var-file` flags you specify (for additional overrides)

You should see Terraform planning to:
1. Validate that all team members are organisation members
2. Create organisation memberships for your users
3. Create team memberships linking users to teams

Expected output (using your actual usernames):
```
Terraform will perform the following actions:

  # terraform_data.validate_team_members will be created
  + resource "terraform_data" "validate_team_members" {
      + id = (known after apply)
    }

  # github_membership.members["actual-github-username-1"] will be created
  + resource "github_membership" "members" {
      + role     = "admin"
      + username = "actual-github-username-1"
    }

  # github_membership.members["actual-github-username-2"] will be created
  + resource "github_membership" "members" {
      + role     = "member"
      + username = "actual-github-username-2"
    }

  # github_membership.members["actual-github-username-3"] will be created
  + resource "github_membership" "members" {
      + role     = "member"
      + username = "actual-github-username-3"
    }

  # github_team_membership.memberships["data-platform-admins-actual-github-username-1"] will be created
  + resource "github_team_membership" "memberships" {
      + team_id  = (known after apply)
      + username = "actual-github-username-1"
      + role     = "member"
    }

  # github_team_membership.memberships["data-engineers-actual-github-username-1"] will be created
  + resource "github_team_membership" "memberships" {
      + team_id  = (known after apply)
      + username = "actual-github-username-1"
      + role     = "member"
    }

  # github_team_membership.memberships["data-engineers-actual-github-username-2"] will be created
  + resource "github_team_membership" "memberships" {
      + team_id  = (known after apply)
      + username = "actual-github-username-2"
      + role     = "member"
    }

  # github_team_membership.memberships["data-analysts-actual-github-username-3"] will be created
  + resource "github_team_membership" "memberships" {
      + team_id  = (known after apply)
      + username = "actual-github-username-3"
      + role     = "member"
    }

Plan: 7 to add, 0 to change, 0 to destroy.
```

Notice:
- The validation resource is created first
- Organisation memberships will be created next (due to `depends_on`)
- Team memberships show `(known after apply)` for team_id because teams were created in section 3
- Resource names include your actual GitHub usernames

### Apply the Changes

```sh
terraform apply
```

Type `yes` when prompted.

## Set Up CODEOWNERS for Membership Approval

To ensure that only admins can approve changes to organisation and team membership, add the `members.auto.tfvars` file to your CODEOWNERS configuration.

Update `.github/CODEOWNERS` in your repository root by adding the following to the bottom:

```
# Terraform Membership Configuration
# Only data-platform-admins team can approve membership changes
terraform/github/members.auto.tfvars @your-org/data-platform-admins
```

Replace `@your-org/data-platform-admins` with your actual organisation name and team.

!!! tip "CODEOWNERS Protection"
    With this in place:

    - Pull requests that modify `members.auto.tfvars` require approval from the data-platform-admins team
    - General configuration in `terraform.tfvars` can be reviewed by anyone
    - Sensitive membership changes have an additional approval gate
    - You can track who approved membership changes in PR history

!!! note "Commit members.auto.tfvars to Git"
    Unlike other `.tfvars` files, we **DO** want to commit `members.auto.tfvars` to version control because:

    - It doesn't contain secrets (just GitHub usernames which are public)
    - We want to track membership changes in Git history
    - CODEOWNERS protection requires the file to be in the repository
    - The `.auto.tfvars` extension makes Terraform load it automatically

    The `.gitignore` in section 3 excludes `*.tfvars` but includes `terraform.tfvars`. We need to also allow `*.auto.tfvars` files.

## Update .gitignore

The `.gitignore` configuration from section 3 currently has:

```gitignore
# Exclude all tfvars files which may contain sensitive data
*.tfvars
*.tfvars.json

# BUT include terraform.tfvars since it only has non-sensitive config
!terraform.tfvars
```

We need to also allow `*.auto.tfvars` files. Update the `.gitignore` in your repository root:

```gitignore
# Exclude all tfvars files which may contain sensitive data
*.tfvars
*.tfvars.json

# BUT include terraform.tfvars and *.auto.tfvars since they only have non-sensitive config
!terraform.tfvars
!*.auto.tfvars
```

This allows `members.auto.tfvars` (and any other `*.auto.tfvars` files) to be committed to Git whilst still excluding other tfvars files that might contain secrets.

## Verify in GitHub

### Check Organisation Members

1. Go to your organisation's People page: `https://github.com/orgs/YOUR-ORG/people`
2. Verify all your users are listed
3. Check that users with `role = "admin"` have the "Owner" badge

### Check Team Memberships

1. Go to Teams page: `https://github.com/orgs/YOUR-ORG/teams`
2. Click on each team to verify the members match your `members.tfvars` configuration:
   - **data-platform-admins**: Should show your admin users
   - **data-engineers**: Should show your engineering team members
   - **data-analysts**: Should show your analyst team members

## Understanding What You've Built

You've now seen several important Terraform concepts in action:

### 1. For_each Loops

Used to create multiple similar resources:
```hcl
resource "github_membership" "members" {
  for_each = var.organization_members
  # Creates one resource per member
}
```

### 2. Dynamic References

Linking resources together:
```hcl
team_id = github_team.teams[each.value.team].id
# References another resource's attribute
```

### 3. Resource Dependencies

Terraform understands the order things must be created:
- **Implicit**: When you reference a resource attribute (`team_id = github_team.teams[...].id`)
- **Explicit**: When you use `depends_on = [...]`

### 4. Data Transformation

Flattening nested structures:
```hcl
locals {
  team_member_pairs = flatten([...])
}
```

This transforms a nested map into a flat list that `for_each` can use.

## Common Patterns Explained

### Pattern: Map of Objects for Configuration

```hcl
variable "organization_members" {
  type = map(object({
    role = string
  }))
}
```

This pattern lets you define configuration as data:
- Easy to read and maintain
- Can be extended with more fields later
- Works well with `for_each`

### Pattern: Flatten for Many-to-Many Relationships

Teams have many members, members belong to many teams. The flatten pattern handles this:

```hcl
locals {
  team_member_pairs = flatten([
    for team, config in var.team_memberships : [
      for member in config.team_members : {
        team   = team
        member = member
      }
    ]
  ])
}
```

This creates one resource per relationship, with a unique key.

### Pattern: Validation with Preconditions

You can validate data before Terraform creates resources using lifecycle preconditions:

```hcl
locals {
  # Collect all team members
  all_team_members = toset(flatten([
    for team, config in var.team_memberships : config.team_members
  ]))

  # Find members assigned to teams who aren't in the organisation
  invalid_team_members = setsubtract(
    local.all_team_members,
    keys(var.organization_members)
  )
}

resource "terraform_data" "validate_team_members" {
  lifecycle {
    precondition {
      condition     = length(local.invalid_team_members) == 0
      error_message = "Team members must be organisation members first: ${join(", ", local.invalid_team_members)}"
    }
  }
}
```

This pattern:
- Catches errors during `terraform plan` before attempting to create resources
- Provides clear, actionable error messages
- Uses set operations (`setsubtract`) to find invalid data
- Prevents runtime failures with early validation

## Troubleshooting

### Error: Team members not in organisation

```
Error: Precondition failed
│
│   on teams.tf line 24, in resource "terraform_data" "validate_team_members":
│   24:       condition     = length(local.invalid_team_members) == 0
│
│ The following users are assigned to teams but not in organization_members:
│ some-username. Add them to organization_members in members.auto.tfvars first.
```

This validation error means you've added a user to a team without adding them to the organisation first.

**Fix**: Add the missing username to the `organization_members` section in `members.auto.tfvars`:

```hcl
organization_members = {
  "some-username" = {
    role = "member"
  }
  # ... other members
}
```

### Error: User not found

```
Error: GET https://api.github.com/users/some-username: 404 Not Found
```

The GitHub username doesn't exist. Check:
1. Spelling is correct (GitHub usernames are case-sensitive)
2. User has a GitHub account
3. Username hasn't changed
4. You're using the username, not email address

### Error: Must be an organisation member

```
Error: User must be a member of the organization
```

This happens if team membership is created before organisation membership. Check:
- `depends_on = [github_membership.members]` is present in team membership resource
- Run `terraform apply` again (dependencies should fix it on retry)

### Members Not Showing in Team

If members don't appear:
1. Check they accepted the organisation invitation
2. Verify the team name matches exactly (case-sensitive)
3. Run `terraform refresh` to sync state with GitHub

## Best Practices

### 1. Always Use depends_on for Team Memberships

```hcl
resource "github_team_membership" "memberships" {
  # ...
  depends_on = [github_membership.members]
}
```

This prevents race conditions where Terraform tries to add someone to a team before they're an organisation member.

### 2. Use Meaningful Keys in for_each

```hcl
for_each = {
  for pair in local.team_member_pairs :
  "${pair.team}-${pair.member}" => pair  # Good: "data-engineers-alice"
}
```

Not:
```hcl
for_each = {
  for idx, pair in local.team_member_pairs :
  idx => pair  # Bad: "0", "1", "2"
}
```

Meaningful keys make it clear which resource is which in `terraform plan` output.

### 3. Keep Team and Member Definitions Together

Put team memberships in the same file as team definitions (`teams.tf`) so it's easy to see the full team structure.

## Commit your work

Make sure to commit your work - remember commit frequently. You need to check you are on the correct branch, which you can do at any time by running `gst` if you haven't set up your command prompt to include the current branch.

## What's Next

You've now got a complete GitHub organisation structure managed in Terraform:

- ✅ Organisation settings and policies
- ✅ Teams with clear purposes
- ✅ Organisation members with roles
- ✅ Team memberships linking users to teams
- ✅ Dynamic resource references
- ✅ Proper dependency management

Next, you'll clean up the import blocks and finalise the Terraform configuration.

Continue to [Finishing Up](5-finishing-up.md) →
