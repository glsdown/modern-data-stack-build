# GitHub Organisation Setup

On this page, you will:

- [x] Create a GitHub organisation for your data stack
- [x] Set up the main infrastructure repository
- [x] Configure team access and permissions
- [x] Set up branch protection rules

## Why a GitHub Organisation?

A GitHub organisation (rather than a personal account) provides:

- **Team collaboration**: Multiple people can contribute with appropriate permissions
- **Centralised billing**: Easier to manage if you need paid features
- **Professional appearance**: Better for business use
- **Scalability**: Easy to add new repositories and team members

!!! info "Infrastructure as Code Coming Soon"
    In this guide, we'll set up the organisation and first repository manually. This is necessary because we need somewhere to store our Terraform code! Once we have this initial repository, we'll use Terraform to manage teams, additional repositories, branch protection, and other GitHub settings. You'll learn this in the [Infrastructure as Code](../build/infrastructure-as-code/index.md) section.

## Create Your Organisation

1. Go to [github.com](https://github.com) and sign in
2. Click your profile picture → **Your organizations**
3. Click **New organization**
4. Choose **Team** plan (you can upgrade later if needed) - this is the [cheapest plan](https://github.com/pricing) with the recommended tools including required reviewers and codeowners.
5. Enter an organisation name (e.g., `your-company-data`, `acme-analytics`)
6. Enter a contact email
7. Select that this belongs to **My business or institution**
8. Complete the setup

!!! tip "Organisation Naming"
    Choose a name that clearly indicates this is for your company's data infrastructure. Avoid names that are too specific to one project, as you'll likely add more repositories over time.

## Create the Infrastructure Repository

A repository (or repo) in git is the name given for what many people would call a "folder of files". This repository will house all your Terraform configurations and infrastructure code.

1. From your organisation page, click **New repository**
2. Set the repository name: `data-stack-infrastructure`
3. Add description: "Infrastructure as code for the modern data stack"
4. Choose **Private** (recommended for production infrastructure)
5. Select **Add a README file**
6. Add `.gitignore` template: Select **Terraform**
7. Choose a licence: **MIT License** (or your preference)
8. Click **Create repository**

## Set Up Team Access

Configure who can access your repository and what they can do.

!!! tip "Automating This Later"
    You're creating teams manually now, but once Terraform is set up, you'll manage teams, membership, and permissions as code. This makes it easy to audit access and maintain consistency across your organisation.

### Create Teams

1. In your organisation, go to **Teams**
2. Click **New team**
3. Create these teams:

#### Data Platform Admins
- **Team name**: `data-platform-admins`
- **Description**: "Full access to data infrastructure"
- **Visibility**: Visible (team members can see who else is on the team)
- **Add members**: Add yourself and other infrastructure owners

#### Data Engineers
- **Team name**: `data-engineers`
- **Description**: "Data Engineers"
- **Visibility**: Visible (team members can see who else is on the team)
- **Add members**: Add your data engineering team

### Assign Repository Permissions

1. Go to the `data-stack-infrastructure` repository
2. Click **Settings** → **Collaborators and teams**
3. Click **Add teams**
4. Add teams with permissions:
   - **data-platform-admins**: Admin
   - **data-engineers**: Write
   - **data-analysts**: Read (optional)

### Create a CODEOWNERS file

A [CODEOWNERS file](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners) defines which teams have responsibility for which parts of the project. You can (and we will) configure repositories to require approvals from code owners before code becomes live to protect the code base.

To create a CODEOWNERS file, click on the "Add file" > "Create new file" button in the repo. At the top, you will see a breadcrumb to the path location and a box that says "Name your file". In this box type `.github/CODEOWNERS`. You should see the path resolves to include `.github` in it.

In the large text editor below enter the following:

```
# Global owners
*       @your-org/data-platform-admins   @your-org/data-engineers
```

Press the "commit changes" button, then type "Add global owners to CODEOWNERS file" as your commit message, ensure "Commit directly to the main branch" is selected, and press "commit changes".

!!! note "Commit"
    A "commit" is the git terminology for creating a save checkpoint. It allows you to describe what changes have been made since the last checkpoint (the commit message), and regular commits will allow you to jump back and forth through your code history much more easily.

## Configure Branch Protection

Protect your main branch from accidental changes and enforce code review.

!!! tip "Automating This Later"
    Branch protection rules can also be managed with Terraform, ensuring consistent policies across all repositories. We'll set this up manually now and migrate to Terraform in the infrastructure-as-code section.

### Set Branch Protection Rules

!!! warning "You can't approve your own code"
    If you are working through this project as a lone wolf, you can't approve your own code. You may want to get these settings ready to go but not enable the branch settings until you are happy that the repository has been set up correctly, and you have another developer with the correct permissions to approve your code. If you don't have a team working with you, you can enable the settings below, but ensure you have disabled "require approvals" and "require review from Code Owners".

1. Go to repository **Settings** → **Branches**
2. Click **Add branch protection rule**
3. Set **Branch name pattern**: `main`
4. Enable these settings:

#### Required Settings

✅ **Require a pull request before merging**
- Require approvals: **1** (increase for larger teams)
- Require review from Code Owners
- Dismiss stale pull request approvals when new commits are pushed

✅ **Require status checks to pass before merging**
- Require branches to be up to date before merging

✅ **Require conversation resolution before merging**

✅ **Do not allow bypassing the above settings**
- Even admins must follow the rules (recommended)

5. Click **Create** to save the rule

### Configure Merge Settings

Now configure which merge methods are allowed and how PRs behave. A merge is how changes are combined together:

1. Go to repository **Settings** → **General**
2. Scroll down to **Pull Requests** section
3. Configure merge options:
   - ❌ **Uncheck** "Allow merge commits"
   - ❌ **Uncheck** "Allow rebase merging"
   - ✅ **Check** "Allow squash merging" (keep this enabled)
4. ✅ Enable **"Always suggest updating pull request branches"**
   - Whenever there are new changes available in the base branch, present an "update branch" option in the pull request
5. ✅ Enable **"Automatically delete head branches"**
   - Deleted branches will still be able to be restored
6. Click **Save changes**

!!! info "Why Squash Merging Only?"
    - **Clean history**: Each feature becomes one commit on main
    - **Easy rollback**: Can revert (undo) entire features with one command
    - **Better readability**: Main branch shows what changed, not how, and link to the PR that changed them for more details.
    - **Consistent**: All PRs merged the same way
    
    Individual commits are still visible in the PR for review, but main stays clean.

### Why These Rules Matter

- **Pull request requirement**: All changes reviewed by at least one other person
- **Status checks**: Automated tests must pass (we'll add these with GitHub Actions)
- **Conversation resolution**: Discussions must be resolved before merging
- **Squash only**: Maintains a clean, readable main branch history
- **Update branch suggestions**: Helps keep PRs current with latest changes
- **Auto-delete branches**: Keeps repository tidy without losing history

!!! success
    You now have a GitHub organisation with a properly configured infrastructure repository!

## What's Next

You've completed the manual GitHub setup that provides the foundation for everything else. Soon you'll learn how to manage teams, repositories, and settings with Terraform, making your entire GitHub organisation infrastructure-as-code.

### Coming Up

- **[Local Development Environment](2-local-environment.md)** - Install Git, Terraform, and other tools
- **[Development Workflow](3-development-workflow.md)** - Learn the branching and PR process
- **[Infrastructure as Code](../build/infrastructure-as-code/index.md)** - Learn Terraform using GitHub as your first provider
- **Automate GitHub** - Move teams, branch protection, and repos to Terraform

Continue to [Local Development Environment](2-local-environment.md) →
