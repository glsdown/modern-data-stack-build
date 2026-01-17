# Set Up Terraform Locally

On this page, you will:

- [x] Install Terraform on your local machine
- [x] Verify the installation
- [x] Understand the Terraform workflow
- [x] Learn basic Terraform commands
- [x] Create a test configuration to verify everything works

## Verify Terraform Installation

You should have already installed Terraform as part of the [Local Development Environment](../initial-setup/2-local-environment.md#install-terraform) setup. Let's verify it's working correctly.

### Check Terraform Version

```sh
terraform version
```

Expected output (version number may vary):

```
Terraform v1.14.3
on darwin_arm64
```

!!! tip "Terraform Version Management"
    We installed Terraform using `tfenv`, which allows you to switch between different Terraform versions easily. This is helpful when working on multiple projects that require different Terraform versions.

    To install a different version: `tfenv install 1.x.x && tfenv use 1.x.x`

### If Terraform Is Not Installed

If you skipped the local environment setup or need to install Terraform now, follow these instructions:

**macOS (using tfenv - recommended):**

```sh
brew install tfenv
tfenv install 1.14.3
tfenv use 1.14.3
```

**Other platforms:**

<details>
<summary>Linux (Ubuntu/Debian)</summary>

```sh
# Install tfenv
git clone https://github.com/tfutils/tfenv.git ~/.tfenv
echo 'export PATH="$HOME/.tfenv/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Install Terraform
tfenv install 1.14.3
tfenv use 1.14.3
```

</details>

<details>
<summary>Windows (Chocolatey)</summary>

```powershell
choco install terraform
```

Or download the binary from [terraform.io/downloads](https://www.terraform.io/downloads) and add it to your PATH.

</details>

## Understanding the Terraform Workflow

Before writing any code, understand how Terraform works:

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  1. Write Configuration (.tf files)                     │
│     ↓                                                   │
│  2. terraform init                                      │
│     - Downloads providers                               │
│     - Configures backend (remote state)                 │
│     ↓                                                   │
│  3. terraform plan                                      │
│     - Compares desired state with current state         │
│     - Shows what will change                            │
│     ↓                                                   │
│  4. Review the plan                                     │
│     - Check the changes make sense                      │
│     - Ensure no unexpected deletions                    │
│     ↓                                                   │
│  5. terraform apply                                     │
│     - Executes the plan                                 │
│     - Creates/updates/deletes resources                 │
│     - Updates state file                                │
│     ↓                                                   │
│  6. Verify in provider (AWS Console, GitHub, etc.)      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Key Commands

**`terraform init`**
Initialises a Terraform working directory. Downloads providers and configures the backend. Run this first in any new Terraform project, and re-run it when you add new providers.

**`terraform plan`**
Shows what changes Terraform will make without actually making them. Always run this before apply to review changes.

**`terraform apply`**
Executes the changes shown in the plan. Creates, updates, or deletes resources to match your configuration. **You should never run this locally - this should always be handled by a service principle.**

**`terraform destroy`**
Removes all resources managed by Terraform. Use with extreme caution in production. **You should never run this locally - this should always be handled by a service principle.**

**`terraform fmt`**
Formats your Terraform files to standard style. Run before committing code.

**`terraform validate`**
Checks your configuration for syntax errors. Useful for catching mistakes early.

## Create a Test Configuration

Let's create a simple test to verify Terraform works correctly.

### Create a Test Directory

Ensure you are in a directory where you store your projects. Run the following:

```sh
cd projects/data  # or wherever you store your projects
take terraform-test
code .
```

### Write a Simple Configuration

Create a file called `main.tf`:

```hcl
terraform {
  required_version = ">= 1.0"

  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.0"
    }
  }
}

resource "local_file" "test" {
  filename = "${path.module}/test-output.txt"
  content  = "Hello from Terraform! Created at ${timestamp()}"
}

output "file_path" {
  value       = local_file.test.filename
  description = "Path to the created file"
}
```

This configuration:
- Specifies Terraform version requirements
- Uses the `local` provider (for creating local files)
- Creates a file called `test-output.txt`
- Outputs the file path

### Initialise Terraform

```sh
terraform init
```

Expected output:

```
Initializing the backend...

Initializing provider plugins...
- Finding hashicorp/local versions matching "~> 2.0"...
- Installing hashicorp/local v2.4.0...
- Installed hashicorp/local v2.4.0 (signed by HashiCorp)

Terraform has been successfully initialized!
```

This downloads the `local` provider which Terraform needs to create files.

### Plan the Changes

```sh
terraform plan
```

Expected output:

```
Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # local_file.test will be created
  + resource "local_file" "test" {
      + content              = "Hello from Terraform! Created at 2026-01-17T22:00:00Z"
      + content_base64sha256 = (known after apply)
      + content_base64sha512 = (known after apply)
      + content_md5          = (known after apply)
      + content_sha1         = (known after apply)
      + content_sha256       = (known after apply)
      + content_sha512       = (known after apply)
      + directory_permission = "0777"
      + file_permission      = "0777"
      + filename             = "/Users/yourname/terraform-test/test-output.txt"
      + id                   = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + file_path = "/Users/yourname/terraform-test/test-output.txt"
```

Notice:
- `+` means "create"
- Some values show `(known after apply)` - these are computed after the resource is created
- The plan shows 1 resource to add
- Outputs show what will be displayed after apply

### Apply the Changes

```sh
terraform apply
```

Terraform shows the plan again and asks for confirmation:

```
Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value:
```

Type `yes` and press Enter.

Expected output:

```
local_file.test: Creating...
local_file.test: Creation complete after 0s [id=abc123...]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

file_path = "projects/data/terraform-test/test-output.txt"
```

### Verify the Result

Check the file was created:

```sh
cat test-output.txt
```

Expected output:

```
Hello from Terraform! Created at 2026-01-17T22:00:00Z
```

### Check the State File

Terraform created a state file tracking what it created:

```sh
ls -la
```

You'll see:

```
-rw-r--r--  1 yourname  staff   123 Jan 17 22:00 main.tf
-rw-r--r--  1 yourname  staff   456 Jan 17 22:00 terraform.tfstate
-rw-r--r--  1 yourname  staff    89 Jan 17 22:00 test-output.txt
```

!!! warning "State File Contains Sensitive Data"
    The `terraform.tfstate` file contains the complete state of your infrastructure, including sensitive data like passwords and API keys. Never commit this file to Git. For real projects, you'll use remote state (which we set up in the previous page).

### Modify the Configuration

Let's make a change. Edit `main.tf` and change the content:

```hcl
terraform {
  required_version = ">= 1.0"

  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.0"
    }
  }
}

resource "local_file" "test" {
  filename = "${path.module}/test-output.txt"
  content  = "Hello from Terraform! Updated at ${timestamp()}"
}

output "file_path" {
  value       = local_file.test.filename
  description = "Path to the created file"
}
```

Run plan to see what will change:

```sh
terraform plan
```

Expected output:

```
Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  ~ update in-place

Terraform will perform the following actions:

  # local_file.test will be updated in-place
  ~ resource "local_file" "test" {
      ~ content              = "Hello from Terraform! Created at ..." -> "Hello from Terraform! Updated at ..."
        id                   = "abc123..."
        # (8 unchanged attributes hidden)
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

Notice:
- `~` means "update in-place"
- Terraform shows what's changing (content) and what's staying the same
- Plan shows 1 to change

Apply the change:

```sh
terraform apply -auto-approve
```

!!! tip "Auto-Approve Flag"
    The `-auto-approve` flag skips the confirmation prompt. Use it when you're confident in the changes, but be careful - it's safer to review the plan first.

### Clean Up

Remove the test resources:

```sh
terraform destroy
```

Terraform shows what it will delete:

```
Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  - destroy

Terraform will perform the following actions:

  # local_file.test will be destroyed
  - resource "local_file" "test" {
      - content              = "Hello from Terraform! Updated at ..." -> null
      - filename             = "/Users/yourname/terraform-test/test-output.txt" -> null
      - id                   = "abc123..." -> null
      # (8 unchanged attributes hidden)
    }

Plan: 0 to add, 0 to change, 1 to destroy.

Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value:
```

Type `yes` to confirm.

The file and state are removed:

```sh
ls -la
```

Only `main.tf` remains (and `.terraform/` directory with provider plugins).

## Terraform Configuration Language Basics

Now that you've seen Terraform in action, let's understand the syntax.

### Blocks

Terraform uses blocks to define configuration:

```hcl
block_type "label" "name" {
  argument = "value"

  nested_block {
    nested_argument = "value"
  }
}
```

Common block types:
- `terraform` - Terraform settings
- `provider` - Provider configuration
- `resource` - Infrastructure resources
- `data` - Data sources (read existing resources)
- `variable` - Input variables
- `output` - Output values
- `locals` - Local values (computed within config)
- `module` - Reusable modules

### Resources

Resources are the most important block type:

```hcl
resource "resource_type" "local_name" {
  argument1 = "value1"
  argument2 = "value2"
}
```

- **resource_type**: Provider-specific type (e.g., `aws_s3_bucket`, `github_repository`)
- **local_name**: Name you use to reference this resource in your config
- **arguments**: Resource-specific settings

### Variables

Variables make your configuration reusable:

```hcl
variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string
  default     = "dev"
}

resource "aws_s3_bucket" "data" {
  bucket = "my-data-${var.environment}"
}
```

### Outputs

Outputs display values after apply:

```hcl
output "bucket_name" {
  value       = aws_s3_bucket.data.bucket
  description = "Name of the S3 bucket"
}
```

### Expressions and Functions

Terraform supports expressions and built-in functions:

```hcl
# String interpolation
name = "bucket-${var.environment}"

# Functions
timestamp = timestamp()
upper_env = upper(var.environment)

# Conditionals
count = var.create_bucket ? 1 : 0

# Lists and maps
tags = {
  Environment = var.environment
  ManagedBy   = "terraform"
}
```

## Best Practices for Local Development

### Directory Structure

Organise your Terraform code logically:

```
terraform-project/
├── main.tf           # Main resources
├── variables.tf      # Variable definitions
├── outputs.tf        # Output definitions
├── providers.tf      # Provider configurations
├── backend.tf        # Backend configuration (remote state)
├── terraform.tfvars  # Variable values (don't commit if contains secrets)
└── .gitignore        # Ignore state files and secrets
```

### .gitignore for Terraform

Always use a `.gitignore` to prevent committing sensitive files:

```gitignore
# Local .terraform directories
**/.terraform/*

# .tfstate files
*.tfstate
*.tfstate.*

# Crash log files
crash.log
crash.*.log

# Exclude all .tfvars files which may contain sensitive data
*.tfvars
*.tfvars.json

# Ignore override files
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# Ignore CLI configuration files
.terraformrc
terraform.rc
```

### Format Your Code

You can format your code manually:

```sh
terraform fmt -recursive
```

However, we'll set up pre-commit hooks to do this automatically when you commit. This ensures consistent style across your team without having to remember to run the command.

### Validate Before Committing

Check for errors:

```sh
terraform validate
```

This catches syntax errors and invalid references.

### Use Pre-commit Hooks

You should have already installed `pre-commit` as part of the [Local Development Environment](../initial-setup/2-local-environment.md#install-pre-commit) setup. Pre-commit automatically runs checks before pushing code to the remote repository.

We'll set up pre-commit hooks for Terraform in the next page when we create the repository structure. These hooks will automatically run on push:

- Format Terraform files with `terraform fmt`
- Validate Terraform syntax
- Generate documentation with `terraform-docs`
- Check for security issues
- Ensure consistent code style

!!! tip "Pre-commit on Push vs Commit"
    We configure pre-commit to run on push rather than on every commit. This gives you flexibility to make incremental commits during development without waiting for all checks to pass. When you're ready to push your work, pre-commit ensures everything meets quality standards before it reaches the remote repository.

## Common Terraform Patterns

### Managing Multiple Environments

Use workspaces or separate directories:

**Option 1: Workspaces**
```sh
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

terraform workspace select dev
terraform apply
```

**Option 2: Separate Directories** (Recommended)
```
terraform/
├── dev/
│   ├── main.tf
│   └── terraform.tfvars
├── staging/
│   ├── main.tf
│   └── terraform.tfvars
└── prod/
    ├── main.tf
    └── terraform.tfvars
```

We'll use separate directories as it's clearer and safer.

### Using Modules

Modules allow code reuse:

```hcl
module "s3_bucket" {
  source = "./modules/s3-bucket"

  bucket_name = "my-data-bucket"
  environment = "prod"
}
```

### Remote State for Teams

For team projects, always use remote state:

```hcl
terraform {
  backend "s3" {
    bucket         = "terraform-state-123456789012"
    key            = "project/terraform.tfstate"
    region         = "eu-west-2"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

We have already built the resources we need for this, but we'll configure it for your production repo in the next page.

## Troubleshooting

### Error: Command not found

If `terraform` isn't found, ensure it's in your PATH:

```sh
which terraform
```

For Homebrew on macOS, it should be at `/opt/homebrew/bin/terraform`.

### Error: Lock timeout

If someone else is running Terraform, you might see:

```
Error: Error acquiring the state lock
```

Wait for their run to complete, or if it's stuck:

```sh
terraform force-unlock <lock-id>
```

!!! warning "Force Unlock Carefully"
    Only force unlock if you're certain no one else is running Terraform. Otherwise, you risk corrupting the state.

### Error: Provider version conflict

If you see version conflicts, update your required_providers block to use compatible versions:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"  # Accept 5.x versions
    }
  }
}
```

## What's Next

You now have Terraform installed and understand the basic workflow:

- [x] Terraform installed and verified
- [x] Understand the workflow (init → plan → apply)
- [x] Created a test configuration
- [x] Know basic Terraform syntax
- [x] Understand best practices

Next, you'll create the actual Terraform repository structure and configure it to use the remote state you set up in S3.

Continue to [Create the Terraform Repository](3-create-the-terraform-repo.md) →
