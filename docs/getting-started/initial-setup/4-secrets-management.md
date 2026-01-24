# Secrets Management

On this page, you will:

- [x] Understand the importance of proper secrets management
- [x] Set up 1Password with shared vaults for team credential sharing
- [x] Understand the role of AWS Secrets Manager (configured later)
- [x] Learn when to use each tool

## Why Secrets Management Matters

Secrets - API keys, database passwords, access tokens - are the keys to your infrastructure. Poor secrets management leads to:

- **Security breaches**: Credentials in Git history, Slack messages, or shared documents
- **Credential sprawl**: Nobody knows which credentials exist or who has access
- **Rotation difficulty**: Updating a password means tracking down everyone who uses it
- **Audit failures**: No record of who accessed what credentials and when

A proper secrets management strategy addresses all of these concerns by centralising secrets in secure, auditable systems with controlled access.

## The Two-Tool Strategy

For a small data team, two complementary tools provide complete coverage:

| Tool | Purpose | Access Method |
|------|---------|---------------|
| **1Password** | Human access to credentials | Browser extension, desktop app, CLI |
| **AWS Secrets Manager** | Application and CI/CD access | IAM roles, SDK, GitHub Actions |

This separation follows the principle of least privilege: humans access secrets through 1Password, while automated systems access secrets through AWS Secrets Manager.

```
┌─────────────────────────────────────────────────────────────────┐
│                     Secrets Management                          │
├─────────────────────────────┬───────────────────────────────────┤
│         1Password           │       AWS Secrets Manager         │
├─────────────────────────────┼───────────────────────────────────┤
│ • Team member passwords     │ • Database connection strings     │
│ • Personal API tokens       │ • Service account credentials     │
│ • SSH keys                  │ • API keys for applications       │
│ • MFA recovery codes        │ • CI/CD pipeline secrets          │
│ • Shared service logins     │ • Terraform provider credentials  │
└─────────────────────────────┴───────────────────────────────────┘
```

## Setting Up 1Password for Teams

1Password provides secure credential storage with sharing capabilities through vaults.

### Create Your 1Password Account

If you don't already have 1Password:

1. Go to [1password.com](https://1password.com/) and sign up for a Teams or Business account
2. Download the desktop app from [1password.com/downloads](https://1password.com/downloads/mac/)
3. Install the browser extension for your preferred browser

!!! info "Personal vs Team Accounts"
    While personal 1Password accounts work for individual use, a Teams or Business account is required for shared vaults. This is essential for team credential management.

### Understanding Vaults

Vaults are containers for secrets with their own access controls. A typical structure for a data team:

| Vault | Purpose | Access |
|-------|---------|--------|
| **Private** | Personal credentials | Individual only |
| **Data Platform** | Shared infrastructure credentials | Data Platform Admins |
| **Data Engineering** | Team-specific credentials | Data Engineers |
| **Service Accounts** | Non-human service credentials | Admins only |

### Create Shared Vaults

1. In 1Password, go to **New Vault**
2. Name it appropriately (e.g., "Data Platform")
3. Add team members who need access
4. Set appropriate permissions (view, edit, manage)

!!! tip "Vault Naming Convention"
    Use clear, descriptive names that indicate both the scope and sensitivity level. Avoid generic names like "Shared" that don't convey what's inside.

### What to Store in 1Password

Store credentials that humans need to access directly:

- **Cloud console passwords**: AWS root user, Snowflake admin
- **Personal access tokens**: GitHub PATs, API keys for development
- **SSH keys**: Include the private key as an attachment
- **MFA recovery codes**: Critical for account recovery
- **Shared service accounts**: Accounts multiple team members use
- **Secure notes**: Documentation about credential usage

### 1Password CLI Integration

The 1Password CLI allows you to inject secrets into terminal commands without exposing them:

```sh
# Install the CLI (if not already done)
brew install 1password-cli

# Sign in
op signin

# List vaults you have access to
op vault list

# Get a specific secret
op item get "Snowflake Admin" --fields password
```

You can also use secret references in scripts:

```sh
# Instead of hardcoding credentials
export SNOWFLAKE_PASSWORD=$(op item get "Snowflake Admin" --fields password)
```

!!! warning "Don't Commit CLI Commands with Secret References"
    While the CLI is useful for local development, never commit scripts that use `op` commands to Git. These would fail in CI/CD and expose your secret structure.

## Understanding AWS Secrets Manager

AWS Secrets Manager is the second half of your secrets strategy. While 1Password handles human access to credentials, Secrets Manager handles programmatic access for applications and CI/CD pipelines.

!!! info "Hands-On Setup"
    You'll create secrets in AWS Secrets Manager after completing the [AWS Account Setup](../account-setup/aws.md). The [Terraform Deployment](../terraform-setup/4-terraform-deployment.md) guide covers the CLI commands and GitHub Actions integration.

### Why AWS Secrets Manager?

- **IAM integration**: Control access using AWS IAM policies
- **Automatic rotation**: Secrets can rotate automatically
- **Audit logging**: CloudTrail records all access
- **Native integration**: Works seamlessly with AWS services and GitHub Actions
- **Encryption**: Secrets encrypted at rest with KMS

### How It Will Be Organised

When you set up Secrets Manager, you'll use a prefix-based naming convention:

```
terraform/              # Secrets used by Terraform
  github-token
  snowflake-credentials

applications/           # Secrets used by applications
  api-keys
  database-connections

cicd/                   # Secrets specific to CI/CD
  deploy-keys
```

This structure makes it easy to apply IAM policies to groups of secrets, granting access only to the secrets each service needs.

### GitHub Actions Integration

Once configured, GitHub Actions can retrieve secrets from Secrets Manager using OIDC authentication. This eliminates the need to store application secrets directly in GitHub - you'll only need bootstrap secrets like `AWS_ACCOUNT_ID` in GitHub to establish the initial AWS connection.

!!! tip "Bootstrap Secrets Stay in GitHub"
    You'll always need `AWS_ACCOUNT_ID` in GitHub Secrets - it's required to authenticate with AWS in the first place. Once authenticated, your workflows retrieve other secrets from Secrets Manager. See [Terraform Deployment](../terraform-setup/4-terraform-deployment.md) for the complete setup.

## Best Practices

### Secret Rotation

Plan for regular rotation of sensitive credentials:

| Secret Type | Rotation Frequency | Method |
|-------------|-------------------|--------|
| Database passwords | 90 days | Secrets Manager automatic rotation |
| API keys | 180 days | Manual with planned cutover |
| Service account passwords | 90 days | Coordinated with Terraform |
| Personal access tokens | 365 days | Individual responsibility |

### Access Control Principles

1. **Least privilege**: Only grant access to secrets that are needed
2. **Separation of duties**: Different secrets for different roles
3. **Audit regularly**: Review who has access to what
4. **Rotate on departure**: When team members leave, rotate shared secrets

### What Never to Do

!!! warning "Secrets Anti-Patterns"
    - Never commit secrets to Git (even in private repos)
    - Never send secrets via Slack, email, or other messaging - send a timed link to the password in 1Password instead
    - Never store secrets in Confluence, Notion, or shared documents
    - Never hardcode secrets in application code
    - Never share personal credentials with colleagues

### Emergency Procedures

Document your response to a potential secret leak:

1. **Identify**: Determine which secret was exposed and where
2. **Rotate immediately**: Generate new credentials
3. **Revoke**: Disable the compromised credential
4. **Audit**: Check for unauthorised access using the old credential
5. **Update**: Deploy new credentials to all systems
6. **Review**: Understand how the leak happened and prevent recurrence

## Setting Up Your Initial Secrets

Before proceeding to infrastructure setup, create these foundational secrets in 1Password:

### In 1Password (Do Now)

Make sure to add your GitHub organisation owner credentials to the `Service Accounts` vault - or another vault you have that contains top secret credentials.

### In AWS Secrets Manager (Later)

You'll create these after completing the AWS account setup:

- [ ] `terraform/github-token` - GitHub PAT for Terraform
- [ ] `terraform/snowflake-credentials` - Snowflake service account

!!! success "What You've Accomplished"
    - [x] Understand the two-tool secrets management strategy
    - [x] 1Password configured with appropriate vaults
    - [x] Understand when AWS Secrets Manager will be used
    - [x] Know when to use each tool
    - [x] Understand rotation and access control best practices

## Next Steps

With your secrets management strategy in place, you're ready to set up your cloud accounts and begin building infrastructure.

Continue to [Account Setup](../account-setup/index.md) →
