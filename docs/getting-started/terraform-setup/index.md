# Infrastructure as Code with Terraform

On this page, you will:

- [x] Understand what Infrastructure as Code (IaC) is and why it matters
- [x] Learn what Terraform does and how it works
- [x] See the incremental approach we'll take to build your infrastructure
- [x] Understand what you'll manage with Terraform

## What is Infrastructure as Code?

Infrastructure as Code (IaC) means managing your infrastructure - servers, databases, networks, users, permissions - using code files rather than clicking through web consoles or running manual commands.

Instead of:
1. Logging into AWS console
2. Clicking through menus to create an S3 bucket
3. Manually configuring permissions
4. Documenting what you did in a wiki
5. Hoping you remember the settings when you need to recreate it

You write:

```hcl
resource "aws_s3_bucket" "data_lake" {
  bucket = "my-company-data-lake"

  versioning {
    enabled = true
  }

  tags = {
    Environment = "production"
    ManagedBy   = "terraform"
  }
}
```

This code creates the bucket with the exact same configuration every time you run it.

### Why Infrastructure as Code Matters

**Version Control**
Every change to your infrastructure is tracked in Git. You can see who changed what, when, and why. You can roll back mistakes. You can review changes before they happen.

**Reproducibility**
Need to create a new environment? Run the same code. Need to recover from a disaster? Run the same code. No guesswork, no "I think we set it up like this".

**Documentation**
The code *is* the documentation. It's always up-to-date because it's what actually creates the infrastructure. No more stale wiki pages.

**Collaboration**
Multiple people can work on infrastructure using the same pull request and code review process you use for application code. Changes are proposed, reviewed, and merged.

**Consistency**
Every environment (dev, staging, production) can use the same code with different variables. This eliminates "it works in dev but not in prod" problems caused by configuration drift.

**Automation**
Once your infrastructure is code, you can automate its deployment. CI/CD pipelines can test changes, show you what will change, and apply updates automatically.

## What is Terraform?

Terraform is an Infrastructure as Code tool created by HashiCorp. It's one of the most popular IaC tools and works with hundreds of different services (called "providers") including:

- **Cloud providers**: AWS, GCP, Azure
- **SaaS platforms**: Snowflake, Confluent Cloud, dbt Cloud
- **Developer tools**: GitHub, GitLab, Terraform Cloud
- **Databases**: PostgreSQL, MySQL, MongoDB
- **And many more**: [see the provider registry](https://registry.terraform.io/browse/providers)

### How Terraform Works

Terraform uses a declarative approach. You describe the *desired state* of your infrastructure, and Terraform figures out how to make it happen.

**Declare what you want:**
```hcl
resource "github_repository" "data_stack" {
  name        = "data-stack-infrastructure"
  description = "Infrastructure as code for the modern data stack"
  visibility  = "private"
}
```

**Terraform plans the changes:**
```
Terraform will perform the following actions:

  # github_repository.data_stack will be created
  + resource "github_repository" "data_stack" {
      + name        = "data-stack-infrastructure"
      + description = "Infrastructure as code for the modern data stack"
      + visibility  = "private"
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

**You review and approve:**
```
Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes
```

**Terraform applies the changes:**
```
github_repository.data_stack: Creating...
github_repository.data_stack: Creation complete after 2s

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

### Key Concepts

**Resources**
The fundamental building blocks. Each resource represents something you want to create - a database, a user, a role, a network policy.

**Providers**
Plugins that let Terraform interact with different services. You'll use the GitHub provider, AWS provider, and Snowflake provider.

**State**
Terraform tracks what it has created in a state file. This lets it know what exists, what needs to be created, and what needs to be updated or deleted.

**Modules**
Reusable pieces of Terraform code. Instead of copying and pasting configuration, you can create a module and use it multiple times with different inputs.

## What You'll Build

In this section, you'll progressively build out your infrastructure as code, starting simple and adding complexity as you learn.

### 1. Set Up Terraform Remote State

First, you'll configure AWS to store Terraform's state file remotely in S3. This is critical for team collaboration - everyone works with the same state file, preventing conflicts.

!!! info "State File"
  A state file is the record of what Terraform believes the infrastructure currently looks like. Having it stored remotely in S3 means that everyone has the same reference point to work from.

**You'll create:**
- S3 bucket for state storage
- DynamoDB table for state locking
- IAM policies for secure access

[Set Up Terraform Remote State](1-set-up-terraform-remote-state.md) →

### 2. Set Up Terraform Locally

Install Terraform on your local machine and configure it to use the remote state you just created.

**You'll learn:**
- How to install Terraform
- How to configure the backend
- Basic Terraform commands
- The Terraform workflow

[Set Up Terraform Locally](2-set-up-terraform-locally.md) →

### 3. Create the Terraform Repository

Set up your `data-stack-infrastructure` repository with proper structure and configure the GitHub, AWS, and Snowflake providers.

**You'll create:**
- Directory structure for Terraform code
- Provider configurations
- Variables and outputs
- `.gitignore` for sensitive files

[Create the Terraform Repository](3-create-the-terraform-repo.md) →

### 4. Add GitHub Resources to Terraform

Take the GitHub organisation and repository you created manually and bring them under Terraform management. This is called "importing" existing infrastructure.

**You'll manage:**
- GitHub organisation settings
- Teams (data-platform-admins, data-engineers, data-analysts)
- Repository configuration
- Team permissions
- Branch protection rules

[Add GitHub to Terraform](4-add-github-to-terraform.md) →

### 5. Terraform Deployment with CI/CD

Set up GitHub Actions to automatically plan and apply Terraform changes when you merge pull requests. This brings code review and automation to your infrastructure.

**You'll create:**
- GitHub Actions workflow
- Terraform plan on pull requests
- Terraform apply on merge to main
- State management in CI/CD

[Terraform Deployment](5-terraform-deployment.md) →

### 6. Add AWS Resources to Terraform

Bring your AWS infrastructure under Terraform management, including the resources created for remote state and in the account setup guide.

**You'll manage:**
- S3 buckets (including state bucket)
- DynamoDB tables (state locking)
- IAM roles (AdminRole, DataEngineerRole)
- IAM users
- Budget alerts
- Service accounts for Terraform

[Add AWS Resources](6-aws-resources.md) →

### 7. Add Snowflake Resources to Terraform

Finally, bring the Snowflake account you created in the account setup guide under Terraform management.

**You'll manage:**
- Admin user and default role configuration
- Service account for Terraform

!!! note "Starting Small with Snowflake"
    This page focuses only on the resources created during account setup. You'll add warehouses, databases, roles, and other Snowflake infrastructure incrementally in the Build section as you need them. This keeps complexity manageable whilst you're still learning.

[Add Snowflake Resources](7-snowflake-resources.md) →

## The Incremental Approach

Notice we're not trying to do everything at once. Each page builds on the previous one:

1. **Remote state** - Foundation for team collaboration
2. **Local setup** - Get Terraform working on your machine
3. **Repository structure** - Organise your code properly
4. **GitHub first** - Practice with a simple provider
5. **CI/CD** - Automate before adding complexity
6. **AWS** - Add cloud infrastructure
7. **Snowflake** - Add data warehouse infrastructure

This approach means:
- You learn one concept at a time
- Each step is testable before moving on
- You build confidence gradually
- You can pause at any point and have working infrastructure

## What About the Resources You've Already Created?

You might be wondering: "I already created my GitHub organisation, AWS roles, and Snowflake account manually. Do I need to delete everything and start over?"

**No!** Terraform can import existing resources. You'll learn this in the subsequent pages.

The process is:
1. Write the Terraform code describing the resource
2. Import the existing resource into Terraform state
3. From then on, Terraform manages it

Your existing infrastructure stays intact - you're just bringing it under Terraform's management.

!!! warning "Manual Changes vs Terraform Management"
    Once a resource is managed by Terraform, you must make changes through Terraform code, not through the UI or CLI. If you create something manually in the AWS Console or Snowflake UI whilst Terraform is managing your infrastructure, Terraform won't know about it.

    Even worse: if you make manual changes to a Terraform-managed resource, the next time Terraform runs, it may undo your changes to match what's in the code. This is by design - Terraform enforces the declared state.

    **Best practice**: Decide which resources Terraform manages and which you'll manage manually. Once a resource is in Terraform, always use Terraform to change it. If you need to make an emergency manual change, document it and update the Terraform code afterwards.

## Why This Order?

We're starting with GitHub for good reasons:

**GitHub is simple**
The GitHub provider is straightforward. You're not dealing with complex networking, encryption, or regional concerns. This lets you focus on learning Terraform itself.

**GitHub is visible**
You can immediately see the results in the GitHub UI. Created a team? You'll see it in your organisation. This instant feedback helps learning.

**GitHub is safe**
Making mistakes with GitHub teams is low-risk. You can easily fix them. Compare this to accidentally deleting production data or exposing sensitive infrastructure.

**GitHub is your foundation**
Your Terraform code lives in GitHub. The CI/CD that deploys it runs in GitHub. So it makes sense to manage GitHub itself with Terraform first.

Once you're comfortable with Terraform using GitHub, adding AWS and Snowflake will feel natural - it's the same patterns and concepts, just different resources.

## Prerequisites

Before starting this section, you should have completed:

- [x] [GitHub Organisation Setup](../1-github-setup.md) - You need a GitHub organisation and repository
- [x] [AWS Account Setup](../account-setup/aws.md) - You need an AWS account with CLI access configured
- [x] [Snowflake Account Setup](../account-setup/snowflake.md) - You need a Snowflake account (for later pages)

You should also be comfortable with:
- Using Git and GitHub (cloning, committing, pushing, pull requests)
- Running commands in a terminal
- Basic text file editing

## What You'll Learn

By the end of this section, you'll:

- Understand Infrastructure as Code principles
- Be proficient with basic Terraform workflows
- Have a complete Terraform setup managing GitHub, AWS, and Snowflake
- Have automated deployment via GitHub Actions
- Know how to import existing infrastructure into Terraform
- Be ready to expand your infrastructure as the project grows

## Security Note

Throughout this section, we'll emphasise security best practices:

- **State files contain secrets** - Store them securely, never commit to Git
- **Use service accounts** - Not personal credentials for Terraform
- **Principle of least privilege** - Grant only required permissions
- **Code review everything** - Terraform changes go through pull requests
- **Separate environments** - Dev, staging, production use different state files

!!! warning "State Files Contain Sensitive Data"
    Terraform state files contain sensitive information including passwords, API keys, and other secrets. Never commit state files to Git. We'll configure remote state in S3 with encryption and access controls.

## Ready to Begin?

Let's start by setting up remote state storage in AWS, which is the foundation for team collaboration with Terraform.

Continue to [Set Up Terraform Remote State](1-set-up-terraform-remote-state.md) →
