# Create a Prefect Cloud Account

!!! info "Optional - Only for Prefect Cloud users"
    This page is only required if you choose to use **Prefect Cloud** (the managed service). If you plan to self-host Prefect, skip this page and proceed directly to the [Orchestration build section](../../build/orchestration/index.md).

On this page, you will:

- [x] Create a Prefect Cloud account
- [x] Create a workspace
- [x] Generate an API key
- [x] Store credentials securely

## Why Prefect Cloud?

Prefect Cloud is the managed version of Prefect - the orchestration platform that coordinates your data pipelines. With Prefect Cloud, you get:

- **Zero infrastructure management** - No servers, databases, or upgrades to maintain
- **Built-in authentication** - SSO, service accounts, and API keys
- **Collaboration features** - Multiple users, workspaces, and audit logs
- **High availability** - Prefect manages uptime and scaling

For a comparison of Prefect Cloud versus self-hosted options, see the [Choosing a Deployment](../../build/orchestration/2-choosing-deployment.md) guide.

## Create Your Account

1. Visit [app.prefect.cloud](https://app.prefect.cloud) and click **Sign Up**
2. Choose your authentication method:
    - **Google/GitHub SSO** (recommended for ease of use)
    - **Email and password**
3. Complete the sign-up process and verify your email if required

## Create a Workspace

After signing up, you'll be prompted to create a workspace. Workspaces are isolated environments for your flows and deployments.

For your workspace name, use something meaningful like:

- `production` or `prod` - For production workloads
- `development` or `dev` - For testing and development
- `<company-name>` - If you only need one workspace initially

!!! tip "Start with One Workspace"
    For small teams, a single workspace is often sufficient. You can create additional workspaces later as your needs grow. Multiple workspaces require the Pro tier or higher.

## Generate an API Key

You'll need an API key to authenticate from your local machine and CI/CD pipelines.

### Create a Service Account (Recommended)

For production use, create a service account rather than using a personal API key:

1. Click your profile icon in the bottom-left corner
2. Select **Service Accounts**
3. Click **Create Service Account**
4. Name it something meaningful like `terraform` or `ci-cd`
5. Select the appropriate role (usually **Admin** for initial setup)
6. Click **Create**
7. Copy the generated API key - you won't be able to see it again

### Or Use a Personal API Key

For personal development or testing:

1. Click your profile icon in the bottom-left corner
2. Select **API Keys**
3. Click **Create API Key**
4. Name it (e.g. `laptop` or `development`)
5. Set an expiration (90-180 days is recommended)
6. Click **Create**
7. Copy the generated API key

## Store Your Credentials

### 1Password

Create a new entry in 1Password for "Prefect Cloud" with:

| Field | Value |
|-------|-------|
| Website | `https://app.prefect.cloud` |
| Account ID | Your account ID (found in account settings) |
| Workspace ID | Your workspace ID (found in workspace settings) |
| API Key | The key you generated above |

### Get Your Account and Workspace IDs

You'll need these IDs for Terraform configuration:

1. **Account ID**: Click your profile icon → **Account Settings** → copy the Account ID
2. **Workspace ID**: Click the workspace name in the sidebar → **Settings** → copy the Workspace ID

Store these in 1Password alongside your API key.

!!! note "AWS Secrets Manager"
    Storing the API key in AWS Secrets Manager for CI/CD and Terraform is covered in the [Prefect Cloud Setup](../../build/orchestration/3-prefect-cloud-setup.md) guide.

## Configure the Prefect CLI

Install and configure the Prefect CLI on your local machine:

```sh
# Install Prefect
uv add prefect

# Login to Prefect Cloud
prefect cloud login
```

Follow the prompts to authenticate. You can either:

- Open the browser to authenticate interactively
- Paste your API key directly

Verify the connection:

```sh
prefect version
prefect cloud workspace ls
```

## What's Next

!!! success
    You now have a Prefect Cloud account with a workspace and API key ready for use!

Your Prefect Cloud account is ready. The detailed configuration - including Terraform setup, work pools, and workers - is covered in the build section.

Continue to [Orchestration](../../build/orchestration/index.md) →
