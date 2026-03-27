# Development Workflow

On this page, you will learn:

- [x] The Git branching strategy
- [x] How to create and work on features
- [x] The pull request process
- [x] Code review best practices
- [x] How to merge and deploy changes

## Overview

A consistent development workflow ensures that:

- All changes are reviewed before merging
- The main branch is always in a working state
- Changes are traceable and revertible
- Team members can collaborate effectively

## Branching Strategy

We use a simplified Git Flow strategy optimised for infrastructure work.

### Branch Types

#### `main` Branch

- **Purpose**: Production-ready code
- **Protection**: Protected, requires PR and approvals
- **Deployment**: Automatically deployed to production (via CI/CD)
- **Never commit directly**: Always use pull requests
- **Lifespan**: Protected - never deleted

#### Feature Branches

- **Purpose**: Develop new features or resources
- **Naming**: `feature/short-description` or `feature/ticket-id/short-description`
- **Example**: `feature/add-analytics-warehouse` or `feature/jira-123/add-analytics-warehouse`
- **Lifespan**: Delete after merging

#### Fix Branches

- **Purpose**: Bug fixes and corrections
- **Naming**: `fix/short-description` or `fix/ticket-id/short-description`
- **Example**: `fix/resource-monitor-threshold` or `fix/jira-123/resource-monitor-threshold`
- **Lifespan**: Delete after merging

#### Documentation Branches

- **Purpose**: Documentation updates
- **Naming**: `docs/short-description` or `docs/ticket-id/short-description`
- **Example**: `docs/update-terraform-setup` or `docs/jira-123/update-terraform-setup`
- **Lifespan**: Delete after merging

#### Refactor Branches

- **Purpose**: Code restructuring without changing functionality
- **Naming**: `refactor/short-description` or `refactor/ticket-id/short-description`
- **Example**: `refactor/database-module-structure` or `refactor/jira-123/database-module-structure`
- **Lifespan**: Delete after merging

### Why This Strategy?

- **Simple**: Only one long-lived branch (main)
- **Clear**: Branch names indicate purpose
- **Safe**: Protected main branch prevents accidents
- **Traceable**: All changes go through pull requests
- **Flexible**: Ticket IDs are optional - include them if you use a ticket tracker

!!! note "Ticket IDs"
    If you use a ticket tracker, including the ticket ID in the branch name is good practice. It links the branch to the work that motivated it, makes changes easier to trace, and gives you a quick reference when revisiting old branches.


## Feature Development Workflow

=== "Terminal"


    **Step 1: Start from Main**

    ```sh
    git checkout main
    git pull origin main
    ```

    **Step 2: Create a Feature Branch**

    ```sh
    git checkout -b feature/add-analytics-warehouse
    ```

    **Step 3: Make Your Changes**
    You should regularly commit your changes to provide a good history. Every time you reach a checkpoint, commit the changes.

    ```sh
    git status # See which files have changed
    git diff # See what has changed
    git add path/to/file # Stage files for commit
    git commit -m "Clear description of change made" # Commit
    ```

    **Step 4: Keep Your Branch Updated**
    Make sure to regularly pull any changes on the `main` branch into your feature branch to minimise divergence. If there are conflicts, resolve them and commit.

    ```sh
    git pull origin main
    ```

    **Step 5: Run Local Checks**
    *(Run tests, lint, etc. as appropriate)*

    **Step 6: Push Your Branch**

    ```sh
    git push origin feature/add-analytics-warehouse
    ```

    !!! note "Using the git plugin"
        If you followed along with the local environment setup, you'll have installed the git plugin for Oh-My-Zsh. This comes with a number of shortcuts for these commands. You can see them all [here](https://github.com/ohmyzsh/ohmyzsh/blob/master/plugins/git/git.plugin.zsh).

        ```sh
        gcm # git checkout main
        gl origin main # git pull origin main
        gcb! feature/add-analytics-warehouse # git checkout -b feature/add-analytics-warehouse
        gst # git status
        gd # git diff
        ga path/to/file # git add path/to/file
        gcmsg "Clear description of change made" # git commit -m "Clear description of change made"
        gl origin main # git pull origin main
        ggpush # git push origin feature/add-analytics-warehouse
        ```


=== "VS Code Source Control Panel"
    **Step 1: Start from Main**

    - Open the Source Control panel (⌃⇧G or click the branch icon)
    - Switch to `main` branch
    - Click the "…" menu > Pull (or use the pull button)

    **Step 2: Create a Feature Branch**

    - Click the branch name in the status bar
    - Select "Create new branch"
    - Name it (e.g. `feature/add-analytics-warehouse`)

    **Step 3: Make Your Changes**

    - Edit files as needed
    - The Source Control panel will show changed files
    - Stage files (with the + button)
    - Write a clear commit message and commit

    **Step 4: Keep Your Branch Updated**

    - Click the "…" menu > Pull (with branch selected)

    **Step 5: Run Local Checks**

    - Use the built-in Problems panel, linters, and test runners
    - Ask Claude to do a review (if appropriate)

    **Step 6: Push Your Branch**

    - Click the "…" menu > Push, or use the push button

=== "Autofetch in VS Code"
    To keep your repo up to date automatically:

    1. Open Command Palette (Cmd+Shift+P)
    2. Search for `settings` and open `Preferences: Open Settings (UI)`
    3. Search for `git autofetch`
    4. Enable **Git: Autofetch**

    VS Code will now regularly fetch updates from the remote.

### Good Commit Messages

Follow these patterns:

```
# Good: Clear, specific, explains what and why
Add analytics warehouse for dbt transformations

# Bad: Vague, no context
Add warehouse

# Good: Multi-line with details
Fix resource monitor credit quota calculation

The previous calculation didn't account for Standard edition
credit rates. Updated to use correct multiplier.

# Bad: No explanation
Fix bug
```

!!! note "Conventional Commits"
    If you really like structure, you can also checkout [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) to see a way of structuring commit messages further.

## Pull Request Process

=== "GitHub Web (after push)"
    1. After pushing, copy the link shown in the terminal (or go to your repo on GitHub)
    2. Click **Pull requests** → **New pull request**
    3. Set **base**: `main` ← **compare**: `feature/add-analytics-warehouse`
    4. Click **Create pull request**

=== "GitHub CLI (gh)"
    You can use the [GitHub CLI](https://cli.github.com/) to create a PR from your terminal:
    ```sh
    gh pr create --base main --head feature/add-analytics-warehouse --web
    ```

### Write a Good Pull Request Description

All repos should have a Pull Request template included to help developers structure their Pull Requests in `.github/pull_request_template.md` - you'll see this in use later. This is automatically populated when you go to create the PR. This should be properly populated with details of the changes and tests to provide the reviewer with as much information as possible.

### Using Claude Code to Create Pull Requests

If you have Claude Code installed and `gh` CLI authenticated, you can automate PR creation with a Claude skill. The skill reads your PR template, inspects the diff and commit history, and creates a fully populated PR on your behalf.

Create the file `.claude/skills/create-pr.md` in your repository:

```markdown
Create a pull request for the current branch using the repository's PR template.

Steps:
1. Read `.github/pull_request_template.md` to understand the required sections
2. Run `git log main..HEAD --oneline` to see the commits on this branch
3. Run `git diff main...HEAD --stat` to understand which files changed
4. Infer the PR title from the branch name and commit messages
5. Populate the template:
   - **Description**: summarise what changed and why, based on the diff and commits
   - **Checklist**: leave all items unchecked for the author to review
   - **Screenshots**: omit this section if not applicable
   - **Additional context**: include the ticket ID from the branch name if present
6. Run `gh pr create --base main --title "<title>" --body "<populated template>"`
7. Output the PR URL
```

Once the file is in place, invoke it from Claude Code:

```
/create-pr
```

Claude will inspect your branch, populate the template, and open the PR - leaving the checklist for you to tick off before requesting review.

!!! tip
    You can customise the skill for each repository. For example, a dbt repository might prompt Claude to list which models were added or modified, and a Terraform repository might ask it to summarise the resources being created or changed.

### Request Reviewers

Add at least one reviewer from your team:

1. Click **Reviewers** in the right sidebar (GitHub Web)
2. Or, with GitHub CLI:
    ```sh
    gh pr edit --add-reviewer username
    ```
3. Alternatively, send them a message to ask them to review it.

## Code Review Best Practices

### For Authors

**Before requesting review:**

- ✅ Self-review your own changes first
- ✅ Ensure CI checks pass
- ✅ Write clear PR description
- ✅ Keep changes focused and small

**During review:**

- ✅ Respond to all comments
- ✅ Ask for clarification if needed
- ✅ Mark conversations as resolved when addressed
- ✅ Be open to feedback

### For Reviewers

**What to look for:**

- ✅ Does the code do what the PR says?
- ✅ Are there any security issues?
- ✅ Is the code well-structured and maintainable?
- ✅ Is documentation updated?
- ✅ Could this break existing infrastructure?

**How to give feedback:**

- ✅ Be specific and constructive
- ✅ Explain the "why" behind suggestions
- ✅ Distinguish between "must fix" and "nice to have"
- ✅ Appreciate good work

**Example comments:**

```markdown
# Good: Specific with explanation
Consider using a variable for the warehouse size to make this more reusable.
This would allow different environments to use different sizes.

# Bad: Vague
This could be better.

# Good: Asks questions
Why is auto_suspend set to 300 seconds here? Is this based on expected
usage patterns? Could we document the reasoning?

# Bad: Demands without context
Change this to 600 seconds.
```

### Review Process

1. **Author creates PR** and requests review
2. **Reviewers review code** and leave comments
3. **Author addresses feedback** and pushes changes
4. **Reviewers approve** when satisfied
5. **Author merges** when all approvals received

## Merging Changes

Merging is when the code is pushed to the another branch - in this flow, we are talking about merging to the `main` branch. That is, make our code available in production.

### When to Merge

Only merge when:

- ✅ All required approvals received
- ✅ All CI checks pass
- ✅ All conversations resolved
- ✅ Branch is up to date with main


### How to Merge

We use **squash and merge** to keep history clean:

1. Click **Squash and merge** on GitHub
2. Edit the commit message if needed (should summarise all commits)
3. Click **Confirm squash and merge**
4. Delete the branch (GitHub will prompt you)

The merge to `main` will trigger CI/CD workflows that apply your Terraform changes to production.

## Deployment Process

Check the deployment has succeeded as expected. If it hasn't, you have 2 options:

- [Revert the PR](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/incorporating-changes-from-a-pull-request/reverting-a-pull-request) - undo the changes made
- Apply a "hotfix" / "fix forward" - create a new PR to fix the issue

## Best Practices Summary

### Do's ✅

- **Do** commit frequently with clear messages
- **Do** keep PRs small and focused (one feature/fix per PR)
- **Do** run tests locally before pushing
- **Do** respond to review feedback promptly
- **Do** keep your branch up to date with main
- **Do** delete branches after merging
- **Do** document why, not just what

### Don'ts ❌

- **Don't** commit directly to main
- **Don't** push without running local validation
- **Don't** create massive PRs with multiple unrelated changes
- **Don't** merge your own PRs without review
- **Don't** leave review comments unaddressed
- **Don't** commit sensitive data (passwords, keys)
- **Don't** force push to shared branches

## Common Scenarios

### Scenario: Merge Conflict

This is where two people have made changes to the same file, and git is unable to reconcile the differences. Take a look at the docs [here](https://code.visualstudio.com/docs/sourcecontrol/merge-conflicts) to understand how resolve them in VS Code.

### Scenario: Accidentally Committed to `main`

Hopefully, your `main` branch should have protections on it so that you can't push to `main` which means that any changes you have made are only local. To "undo" the commit simply run:

```sh
# BEFORE pushing, undo the commit
git reset --soft HEAD~1

# Create proper branch
git checkout -b your-branch-name
git commit -m "[your changes]"
```

### Scenario: Accidentally pushed secrets to GitHub

Firstly, we've all been there, or know someone that has. Sometimes, secrets get out. You are working on a tight deadline and realise you've accidentally committed a file that contains secrets. It's about being open about the leak *internally* and mitigating it as quickly as possible.

1. Let your security team know - you should be working in private repos as best practice, but let them know anyway.
2. Follow the docs [here](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository) on how to remove the file from your local and GitHub histories.
3. Add the file to the project and/or your global .gitignore file so it never gets committed again!

### Scenario: Want to Split PR into Smaller PRs

Sometimes, you are working on something, and realise it's become a huge change. The best approach is therefore to split the branch in two:

```sh
# Create new branches from your feature branch
git checkout feature/large-feature
git checkout -b feature/part-1
# Remove unwanted changes
git checkout feature/large-feature
git checkout -b feature/part-2
# Remove unwanted changes from this too
```

!!! success "What You've Accomplished"
    - [x] Understand the Git branching strategy
    - [x] Know how to create feature, fix, and documentation branches
    - [x] Understand the pull request and code review process
    - [x] Can handle common Git scenarios

## What's Next

Before moving on to building infrastructure, let's establish a proper secrets management strategy. This ensures credentials are handled securely from day one.

Continue to [Secrets Management](4-secrets-management.md) →
