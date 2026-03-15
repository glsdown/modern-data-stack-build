# Finishing Up

On this page, you will:

- [x] Remove import blocks
- [x] Verify changes
- [x] Create GitHub README
- [x] Format and push your code

## Remove Import Blocks

Now that all resources are imported, delete the `imports.tf` file.

Import blocks are only needed once. After successful import, the entire file can be deleted.

### Delete imports.tf

```sh
rm imports.tf
```

Your configuration now consists of:
- `backend.tf` - S3 backend configuration
- `main.tf` - Terraform and provider version requirements
- `providers.tf` - GitHub provider configuration
- `variables.tf` - Input variable definitions
- `terraform.tfvars` - General configuration (org name, teams)
- `members.auto.tfvars` - Membership configuration (auto-loaded)
- `outputs.tf` - Output definitions
- `organisation.tf` - Organisation settings
- `teams.tf` - Team definitions and team memberships with validation
- `members.tf` - Organisation member management

### Verify No Changes

```sh
terraform plan
```

Expected output:

```
No changes. Your infrastructure matches the configuration.
```

If Terraform shows planned changes, something is misconfigured. Review the diff and adjust your resource definitions to match reality.

## Format and Validate

Format your code:

```sh
terraform fmt -recursive
```

Validate syntax:

```sh
terraform validate
```

Expected output:

```
Success! The configuration is valid.
```

## Create GitHub README

Document your GitHub Terraform configuration:

Create or update `github/README.md` in the `terraform/` directory:

```markdown
# GitHub Infrastructure

This directory manages GitHub organisation settings and teams via Terraform.

## Resources Managed

### Organisation
- Organisation settings and policies
- Default permissions for new repositories
- Security settings (Dependabot, secret scanning)
- Member permissions

### Teams
- **Team definitions**: Managed via `teams` variable in `terraform.tfvars`
- **data-platform-admins**: Full access to data infrastructure
- **data-engineers**: Data Engineers
- **data-analysts**: Data Analysts
- Teams created using `for_each` for easy scalability

### Members
- **Organisation members**: Managed via `organization_members` in `members.auto.tfvars`
- **Team memberships**: Managed via `team_memberships` in `members.auto.tfvars`
- **Validation**: Ensures all team members are organisation members first
- **CODEOWNERS**: Membership changes require admin approval

## What We Don't Manage Here

- **Repositories**: Create repositories through GitHub UI or CLI as needed. Managing repos in Terraform creates tight coupling and makes it difficult to delete repositories.
- **Branch protection rules**: These are repository-specific and better managed per-repository through GitHub's UI.

## File Structure

- `backend.tf` - S3 backend configuration
- `main.tf` - Terraform and provider version requirements
- `providers.tf` - GitHub provider configuration
- `variables.tf` - Input variable definitions
- `terraform.tfvars` - General configuration (org name, teams)
- `members.auto.tfvars` - Membership configuration (auto-loaded, requires admin approval)
- `outputs.tf` - Output definitions
- `organisation.tf` - Organisation settings and policies
- `teams.tf` - Team definitions and team memberships with validation
- `members.tf` - Organisation member management

## Configuration Approach

This setup uses a clear separation:

- **`terraform.tfvars`**: General configuration (organisation name, billing email, team definitions)
- **`members.auto.tfvars`**: Membership configuration (organisation members, team memberships)
  - Auto-loaded by Terraform (no `-var-file` flag needed)
  - Protected by CODEOWNERS requiring admin approval
  - Contains only public GitHub usernames (no secrets)
- **`organisation.tf`**: Organisation settings and security policies
- **`teams.tf`**: Team resources with `for_each` and validation logic
- **`members.tf`**: Organisation member resources

This means you:
- Update `terraform.tfvars` for teams and general settings
- Update `members.auto.tfvars` for membership changes (requires admin approval)
- Everything else is reusable configuration

## Making Changes

### Adding a New Team

1. Add the team to `terraform.tfvars` in the `teams` variable
2. Run `terraform plan` to preview
3. Create a PR for review
4. After approval, merge and CI/CD will apply changes

### Adding/Removing Organisation Members

1. Update `members.auto.tfvars` with the new member in `organization_members`
2. Optionally add them to teams in `team_memberships`
3. Run `terraform plan` to preview (validation will check team members are org members)
4. Create a PR for review
5. Get approval from data-platform-admins team (required by CODEOWNERS)
6. After approval, merge and CI/CD will apply changes

### Changing Organisation Settings

1. Update `organisation.tf` with desired changes
2. Run `terraform plan` to preview
3. Create a PR for review
4. After approval, merge and CI/CD will apply changes
```

## Commit Your Work

Once again, you should commit the work that you have done. Then, you can push to GitHub:

```sh
git push
```

!!! tip "Pre-commit Hooks Run on Push"
    When you push, pre-commit hooks will automatically format, validate, and document your Terraform code. If any checks fail, fix the issues and push again.

Once you have done this, create the PR, get the relevant approvals and merge to `main`.

## Understanding What You've Done

You've now brought your GitHub infrastructure under Terraform management:

### Imported Existing Resources
- ✅ Organisation settings
- ✅ Two existing teams (data-platform-admins, data-engineers)

### Created New Resources via Terraform
- ✅ data-analysts team
- ✅ Organisation members
- ✅ Team memberships

### Advanced Patterns Implemented
- ✅ **For_each loops**: Scalable team and member management
- ✅ **Validation**: Preconditions ensure team members are org members
- ✅ **Flatten pattern**: Many-to-many team membership relationships
- ✅ **Auto-loaded tfvars**: `members.auto.tfvars` loads automatically
- ✅ **CODEOWNERS**: Membership changes require admin approval

### Key Benefits
- **Version controlled**: All GitHub organisational structure is now in code
- **Validated**: Errors caught during plan, not apply
- **Auditable**: Changes tracked in Git history with approval requirements
- **Repeatable**: Can recreate team structure from code
- **Collaborative**: Team can review changes via PRs
- **Documented**: Configuration serves as documentation
- **Scalable**: Adding teams/members doesn't require code changes

## What's Next

You've successfully imported GitHub resources and created new ones with Terraform:

- ✅ Organisation settings managed in code
- ✅ Teams managed in code
- ✅ New data-analysts team created
- ✅ All changes version-controlled
- ✅ Organisational structure is now repeatable and auditable

Next, you'll set up CI/CD to automatically plan and apply Terraform changes through GitHub Actions. This ensures that all infrastructure changes go through code review and are applied consistently.

Continue to [Terraform Deployment with CI/CD](../4-terraform-deployment.md) →
