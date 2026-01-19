# Add GitHub to Terraform

In this section, you will:

- [x] Understand Terraform import workflow
- [x] Set up `.envrc` file for managing environment variables
- [x] Set up the GitHub provider and backend
- [x] Import GitHub organisation settings
- [x] Import existing teams (data-platform-admins, data-engineers)
- [x] Create a new data-analysts team via Terraform
- [x] Manage organisation and team memberships
- [x] Plan and verify changes

## The GitHub Provider

A provider in terraform is a plugin that allows it to interact with various cloud services. 

> The GitHub provider is used to interact with GitHub resources. The provider allows you to manage your GitHub organisation's members and teams easily. It needs to be configured with the proper credentials before it can be used.

You can find out more about the GitHub provider in their [docs](https://registry.terraform.io/providers/integrations/github/latest/docs), along with the range of resources that can be managed with it.

## What Should We Manage in Terraform?

For GitHub, we'll focus on managing:

- **Organisation settings**: Policies, security settings, member permissions
- **Teams**: Team creation, descriptions, and privacy settings
- **Team membership**: Who belongs to which teams

For now, we'll not manage in Terraform:

- **Repositories**: Managing repos in Terraform creates tight coupling and makes it difficult to delete repositories. Create repositories through the GitHub UI or CLI as needed.
- **Branch protection rules**: These are repository-specific and often change frequently. Better managed per-repository through GitHub's UI or via repository-specific `.github` workflows.

!!! note "Pragmatic Approach"
    The goal is to manage organisational structure and access control in Terraform, whilst leaving day-to-day repository operations flexible. This strikes a balance between infrastructure-as-code benefits and operational practicality.

## Set Up GitHub Authentication

Before working with the GitHub provider, set your GitHub token.

### Create a GitHub Personal Access Token

1. Go to [github.com/settings/tokens](https://github.com/settings/tokens)
2. Click **Generate new token** → **Generate new token (classic)**
3. Set token name: "Terraform"
4. Set expiration: 90 days (you'll need to rotate regularly)
5. Select scopes:
   - `admin:org` (Full control of organisations and teams)
6. Click **Generate token**
7. **Copy the token immediately** - you won't see it again

#### Update the .envrc File

In your repository root, you should have created an `.envrc` file. Add the following line:

```sh
# GitHub Token for Terraform
export GITHUB_TOKEN="ghp_your_token_here"
```

Replace `ghp_your_token_here` with your actual GitHub token.

Verify it's working:

```sh
cd ~/projects/data/data-stack-infrastructure
echo $GITHUB_TOKEN  # Should show your token
cd ~
echo $GITHUB_TOKEN  # Should be empty
```

Update the `.envrc.example` with the following contents exactly as they are - do not add your own token:

```sh
# GitHub Token for Terraform
export GITHUB_TOKEN="ghp_your_token_here"
```

## File Organization

For this guide, we'll organise resources into separate files by type:

- `imports.tf` - Import blocks (temporary, deleted after import)
- `organisation.tf` - Organisation settings and policies
- `teams.tf` - Team definitions
- etc.


This approach makes it easier to find and manage resources as your infrastructure grows. You can quickly locate all teams in one file, all organisation settings in another, etc.

!!! tip "Alternative Organisation Approaches"
    Some teams prefer different organisation strategies:

    - **Single file** (`github.tf`): Simple for very small configurations
    - **By resource type** (what we're using): `organisations.tf`, `teams.tf`, `repositories.tf`
    - **By domain** (`admin.tf`, `engineering.tf`): Group resources by team or business domain

    Choose what works best for your team. The important thing is consistency.

## What's Next

You are now ready to configure the [backend](1-setup-github-backend.md).