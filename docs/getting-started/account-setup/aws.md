# Create an AWS Account

On this page, you will:

- [x] Create a new AWS account
- [x] Secure the root user with MFA
- [x] Create your personal IAM user
- [x] Create `AdminRole`, `DataEngineerRole`, and `InfrastructureAdminRole` for permissions
- [x] Set up cost-based alerts
- [x] Configure AWS CLI for local development

### Set Root User Email Alias

Consider setting up an email alias for your root account (e.g., `aws-root@yourcompany.com`) that forwards to multiple administrators. This ensures business continuity if the original account creator leaves.

You can [change the root email](https://docs.aws.amazon.com/accounts/latest/reference/manage-acct-update-root-user.html) at any time through Account Settings.

## Create a New Account

Visit the [AWS sign-up page](https://portal.aws.amazon.com/billing/signup) and complete the registration process. You'll need to provide:

- Email address (this becomes the root account email)
- Account name (use your company/project name)
- Contact information
- Payment method (credit/debit card)
- Phone verification

!!! info "About the Root User"
    The email you provide creates the **root user** - the most powerful account in AWS with unrestricted access to all services and billing. You should rarely use this account for day-to-day operations.

!!! tip "Store Credentials in 1Password"
    As you complete sign-up, immediately create a new entry in your 1Password vault for "AWS Root User". Store the email address and password securely. You'll add MFA details to this entry shortly. Make sure to store this in a Vault that has locked down access - you should only give root user access to a small handful of people.

### Selecting Your Home Region

During the sign-up process and when first logging in, AWS will ask you to select a default region. Choose the region nearest to your data and that fulfils any data residency requirements.

!!! tip "Choosing a Region"
    Common choices for UK-based companies:

    - **eu-west-2 (London)** - lowest latency for UK users, GDPR-compliant
    - **eu-west-1 (Ireland)** - AWS's largest European region, more services available
    - **us-east-1 (N. Virginia)** - AWS's original region, newest services launch here first

    You can use services in any region, but your "home region" is where you'll primarily work. Learn more about [AWS regions here](https://aws.amazon.com/about-aws/global-infrastructure/regions_az/).

### Account Type: Personal vs Business

AWS offers two account types:

- **Personal** - for individual use, simpler verification
- **Business** - for company use, may require business verification documents

For a data stack project, either works. Business accounts provide some additional support options but aren't required to access any AWS services.

## Secure the Root User

After creating your account, your first priority is securing the root user. You should **never** use the root account for daily operations.

### Sign In as Root

1. Go to the [AWS Console](https://console.aws.amazon.com/)
2. Click "Sign in to the Console"
3. Select "Root user"
4. Enter your root account email address
5. Enter the password you created during sign-up

### Enable MFA on Root User

Multi-factor authentication (MFA) is critical for the root account. Enable it immediately:

1. In the AWS Console, click your account name (top-right)
2. Select "Security credentials"
3. Scroll to "Multi-factor authentication (MFA)"
4. Click "Assign MFA device"
5. Choose your MFA type:
    - **1Password** (recommended) - stores both password and TOTP codes together
    - **Authenticator app** - Google Authenticator, Authy, etc.
    - **Security key** - hardware key like YubiKey
    - **Hardware TOTP token** - physical device that generates codes

!!! tip "Using 1Password for MFA"
    1Password can generate TOTP codes directly. When setting up MFA:

    1. Select "Authenticator app" in AWS
    2. In 1Password, edit your "AWS Root User" entry
    3. Add a new field: **One-time password**
    4. Scan the QR code or enter the secret key
    5. 1Password will now generate codes for this account

    This keeps your password and MFA codes together securely.

!!! warning "Store Recovery Codes"
    When setting up MFA, AWS provides recovery codes. Add these to your 1Password entry as a secure note or attachment. If you lose your MFA device and recovery codes, you cannot access your account.

### When to Use the Root User

Only use the root account for tasks that [specifically require it](https://docs.aws.amazon.com/IAM/latest/UserGuide/root-user-tasks.html), such as:

- Changing account settings
- Restoring IAM user permissions
- Activating IAM access to billing
- Closing the account

For all other tasks, use IAM users or roles (which we'll create next).

## Account alias

Your account can be referenced by both an ID and [an alias](https://docs.aws.amazon.com/IAM/latest/UserGuide/console-account-alias.html). The name should be something human-readable and memorable, for example `my-company-name`.

To change the account alias, when logged in as the root user, open the IAM console at https://console.aws.amazon.com/iam/. In the navigation pane, choose Dashboard. In the AWS Account section, next to Account Alias, choose Create. In the dialog box, enter the name you want to use for your alias, then choose Save changes.

## Enable IAM Access to Billing

By default, IAM users cannot access billing information, even with admin permissions. Enable this so your admin users can manage costs:

1. Still signed in as root, navigate to "Account" (top-right menu)
2. Scroll to "IAM User and Role Access to Billing Information"
3. Click "Edit"
4. Check "Activate IAM Access"
5. Click "Update"

This allows IAM users with appropriate permissions to view and manage billing.

!!! info "Why This Matters"
    Without this enabled, only the root user can see costs and billing, which defeats the purpose of limiting root access. This is a one-time configuration that must be done by the root user.

## Create IAM Roles for Access Control

AWS uses role-based access control (RBAC). Rather than granting permissions directly to users, you create roles with specific permissions and assign roles to users.

We'll create three roles:

- **AdminRole** - full administrative access (use sparingly)
- **DataEngineerRole** - permissions needed for data platform work
- **InfrastructureAdminRole** - permissions needed for managing infrastructure (use only for terraform apply)

!!! note "Additional Roles"
    You will find that you may need to introduce new roles as you scale. We'll talk about better ways of managing roles in the terraform setup. This is just to get you started.

### Create the AdminRole

1. Navigate to **IAM** service (search "IAM" in the top search bar)
2. Click "Roles" in the left sidebar
3. Click "Create role"
4. Select **"AWS account"** as the trusted entity type
5. Select **"This account"** and note the account ID
6. Click "Next"
7. In the permissions policies search, type `AdministratorAccess`
8. Check the box next to **AdministratorAccess** policy
9. Click "Next"
10. For role name, enter: `AdminRole`
11. For description, enter: `Full administrative access for account administrators`
12. Review the settings and click "Create role"

### Create the DataEngineerRole

For the data engineer role, we'll use a managed policy that provides permissions for common data services:

1. Return to IAM → Roles
2. Click "Create role"
3. Select **"AWS account"** as the trusted entity type
4. Select **"This account"**
5. Click "Next"
6. Search for and select the following managed policies:
    - `AmazonS3FullAccess` - for data lake storage
    - `AmazonDynamoDBFullAccess` - for Terraform state locking
    - `SecretsManagerReadWrite` - for managing credentials
    - `CloudWatchFullAccess` - for logging and monitoring
    - `IAMReadOnlyAccess` - to view users and roles
7. Click "Next"
8. For role name, enter: `DataEngineerRole`
9. For description, enter: `Permissions for data platform development and operations`
10. Click "Create role"

!!! tip "Refining Permissions"
    These permissions are a starting point. As you build your data platform, you'll likely need to add more specific permissions for services. We'll address these as they become relevant in later sections.

### Create the InfrastructureAdminRole

This role is specifically for Terraform and infrastructure operations. It has write access to Terraform state files, which DataEngineerRole intentionally lacks.

1. Return to IAM → Roles
2. Click "Create role"
3. Select **"AWS account"** as the trusted entity type
4. Select **"This account"**
5. Click "Next"
6. Search for and select the following managed policies:
    - `AmazonS3FullAccess` - for Terraform state and infrastructure
    - `AmazonDynamoDBFullAccess` - for Terraform state locking
    - `SecretsManagerReadWrite` - for managing infrastructure secrets
    - `IAMFullAccess` - for managing IAM resources
    - `CloudWatchFullAccess` - for logging and monitoring
7. Click "Next"
8. For role name, enter: `InfrastructureAdminRole`
9. For description, enter: `Role for Terraform infrastructure management`
10. Click "Create role"

!!! warning "Why a Separate Infrastructure Role?"
    You might wonder why we need InfrastructureAdminRole when DataEngineerRole has similar permissions. The key difference is **state file access**:

    - **InfrastructureAdminRole**: Full read/write to Terraform state files
    - **DataEngineerRole**: Read-only access to state files (we'll configure this in Terraform setup)

    This separation ensures data engineers can't accidentally run `terraform apply` locally - only the InfrastructureAdminRole (and later, the CI/CD pipeline) can modify infrastructure.

### Configure Role Trust Relationships

Roles need to specify who can assume them. We'll configure both roles to allow users in this account:

1. Click on **AdminRole** from the roles list
2. Click the "Trust relationships" tab
3. Click "Edit trust policy"
4. The policy should look like this:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT_ID:root"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

5. Replace `ACCOUNT_ID` with your actual AWS account ID (visible in the top-right corner)
6. Click "Update policy"
7. Repeat these steps for **DataEngineerRole** and **InfrastructureAdminRole**

## Create Your Personal IAM User

Now create an IAM user for yourself. This user will be able to assume both roles we created.

### Create the User

1. In IAM, click "Users" in the left sidebar
2. Click "Create user"
3. Enter username. Use a consistent format, such as:
    - `firstname.lastname` (e.g., `jane.bloggs`)
    - `firstinitiallastname` (e.g., `jbloggs`)
4. Click "Next"
5. **Do not** attach any policies directly - we'll use role assumption instead
6. Click "Next"
7. Review and click "Create user"

!!! info "Why No Direct Permissions?"
    By not attaching permissions directly to the user, we enforce the practice of assuming roles. This provides:

    - Better audit trails (CloudTrail shows which role was used)
    - Easier permission management (modify the role, not individual users)
    - Principle of least privilege (only assume elevated permissions when needed)

### Enable Console Access

1. Click on your newly created user
2. Click the "Security credentials" tab
3. In "Console sign-in", click "Enable console access"
4. Choose "Custom password" and create a secure password
5. **Uncheck** "Users must create a new password at next sign-in" (optional)
6. Click "Apply"

!!! tip "Store in 1Password"
    Create a new 1Password entry for "AWS IAM User - [your name]" in your personal Vault. Store:

    - Account ID (for sign-in)
    - Username
    - Password
    - You'll add MFA codes to this entry next

### Enable MFA for Your User

1. Still in Security credentials, scroll to "Multi-factor authentication (MFA)"
2. Click "Assign MFA device"
3. Choose device name (e.g., `jbloggs-phone`)
4. Select your MFA type (1Password or authenticator app)
5. If using 1Password, add a one-time password field to your IAM user entry and scan the QR code
6. Follow the setup wizard
7. Click "Add MFA"

!!! warning "MFA is Essential"
    Every IAM user should have MFA enabled, especially those with role assumption permissions. This is a critical security control.

### Allow User to Assume Roles

We need to grant your user permission to assume the roles we created:

1. While viewing your user, click the "Permissions" tab
2. Click "Add permissions" → "Create inline policy"
3. Click the JSON tab
4. Paste the following policy (replace `ACCOUNT_ID` with your account ID):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": [
        "arn:aws:iam::ACCOUNT_ID:role/AdminRole",
        "arn:aws:iam::ACCOUNT_ID:role/DataEngineerRole",
        "arn:aws:iam::ACCOUNT_ID:role/InfrastructureAdminRole"
      ]
    }
  ]
}
```

5. Click "Next"
6. For policy name, enter: `AssumeRolesPolicy`
7. Click "Create policy"

!!! info "Role Usage Guidelines"
    - **DataEngineerRole**: Use for day-to-day data platform work (queries, pipelines, monitoring)
    - **InfrastructureAdminRole**: Use only when running Terraform locally during initial setup
    - **AdminRole**: Use sparingly for account-level administration tasks

### Test Console Access with Roles

1. Sign out of the root account
2. Go to the [AWS Console](https://console.aws.amazon.com/)
3. Click "Sign in to the Console"
4. Select "IAM user"
5. Enter your account ID or account alias
6. Enter your username and password
7. Complete MFA challenge

You're now signed in as your IAM user. To assume a role:

1. Click your username (top-right)
2. Click "Switch role"
3. Enter:
    - **Account**: Your 12-digit account ID
    - **Role**: `DataEngineerRole`
    - **Display Name**: `DataEngineer` (this is what appears in the top-right)
    - **Colour**: Choose a colour to distinguish this role
4. Click "Switch Role"

The console now shows you're using the DataEngineerRole with the permissions attached to that role.

!!! tip "Switching Roles"
    Once you've switched to a role, AWS remembers it. You can quickly switch between your recent roles using the menu in the top-right corner.

## Set Up Cost-Based Alerts

AWS can notify you when spending exceeds thresholds. This is critical to avoid unexpected bills.

### Create a Budget

1. Navigate to **AWS Budgets** (search "Budgets" or go through Billing Dashboard)
2. Click "Create budget"
3. Select **"Customise (advanced)"**
4. Choose **"Cost budget"**
5. Click "Next"
6. Configure budget details:
    - **Name**: `Monthly-Cost-Budget`
    - **Period**: Monthly
    - **Budget effective dates**: Recurring budget
    - **Start month**: Current month
    - **Budgeting method**: Fixed
    - **Budget amount**: Enter your monthly limit (e.g., £100)
7. Click "Next"
8. Click "Add an alert threshold"
9. Configure alert:
    - **Threshold**: 80% of budgeted amount (adjust as needed)
    - **Trigger**: Actual costs
    - **Email recipients**: Your email address
10. Optionally add more thresholds (e.g., 50%, 100%, 120%)
11. Click "Next"
12. Review and click "Create budget"

!!! info "Free Tier Tracking"
    You can also create a budget specifically for **Free Tier usage** to track when you're approaching or exceeding free tier limits. This is particularly useful in the first 12 months of your AWS account.

### Set Up Billing Alerts

In addition to budgets, enable billing alerts:

1. Go to **Billing Preferences** (under your account menu → Billing Dashboard)
2. Scroll to "Alert preferences"
3. Check **"Receive AWS Free Tier alerts"**
4. Check **"Receive Billing Alerts"**
5. Enter your email address
6. Click "Save preferences"

You'll receive an email when you approach free tier limits or exceed defined thresholds.

## Configure AWS CLI Access

The AWS CLI allows you to interact with AWS services from your terminal. We'll configure it to use your IAM user credentials and support role assumption.

### Install AWS CLI

On macOS (using Homebrew):

```sh
brew install awscli
```

Verify installation:

```sh
aws --version
# Expected output: aws-cli/2.x.x Python/3.x.x Darwin/23.x.x
```

!!! info "Other Installation Methods"
    - **Linux**: [AWS CLI Linux installation guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
    - **Windows**: [AWS CLI Windows installation guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

### Create Access Keys

You need programmatic access credentials for the CLI:

1. Sign in to AWS Console as your IAM user
2. Go to IAM → Users → [your username]
3. Click "Security credentials" tab
4. Scroll to "Access keys"
5. Click "Create access key"
6. Select **"Command Line Interface (CLI)"**
7. Check the confirmation box
8. Click "Next"
9. Optionally add a description tag (e.g., `Local development - MacBook Pro`)
10. Click "Create access key"
11. **Important**: Copy the Access Key ID and Secret Access Key - you won't see the secret again

!!! tip "Store Access Keys in 1Password"
    Add the access keys to your "AWS IAM User" entry in 1Password:

    - Add a field for "Access Key ID"
    - Add a field for "Secret Access Key"

    This provides a secure backup if you need to reconfigure the CLI later.

!!! warning "Protect Your Access Keys"
    Access keys are like passwords. Never commit them to Git, share them in Slack, or store them unencrypted.

### Configure AWS CLI

Configure the CLI with your access keys:

```sh
aws configure --profile default
```

When prompted, enter:

```
AWS Access Key ID: [your access key]
AWS Secret Access Key: [your secret key]
Default region name: eu-west-2  # or your preferred region
Default output format: json
```

This creates `~/.aws/credentials` and `~/.aws/config` files with your configuration.

### Configure Role Profiles

To easily switch between roles, add profiles to your AWS config. Edit `~/.aws/config`:

```sh
code ~/.aws/config  # or use your preferred editor
```

Add the following configurations (replace `ACCOUNT_ID` with your account ID):

```ini
[default]
region = eu-west-2
output = json

[profile data-engineer]
role_arn = arn:aws:iam::ACCOUNT_ID:role/DataEngineerRole
source_profile = default
region = eu-west-2
output = json

[profile infrastructure-admin]
role_arn = arn:aws:iam::ACCOUNT_ID:role/InfrastructureAdminRole
source_profile = default
region = eu-west-2
output = json

[profile admin]
role_arn = arn:aws:iam::ACCOUNT_ID:role/AdminRole
source_profile = default
region = eu-west-2
output = json
```

This configuration:

- Uses your IAM user credentials as the base (`default` profile)
- Defines `data-engineer` profile that assumes the DataEngineerRole (day-to-day work)
- Defines `infrastructure-admin` profile that assumes the InfrastructureAdminRole (Terraform operations)
- Defines `admin` profile that assumes the AdminRole (account administration)
- All role profiles use the default credentials to assume their respective roles

### Test Your Configuration

Test the default profile (your IAM user with no role):

```sh
aws sts get-caller-identity
```

Expected output:

```json
{
    "UserId": "AIDAEXAMPLEUSERID",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/jane.bloggs"
}
```

Test the data-engineer profile:

```sh
aws sts get-caller-identity --profile data-engineer
```

Expected output:

```json
{
    "UserId": "AROAEXAMPLEROLEID:aws-cli-session",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/DataEngineerRole/aws-cli-session"
}
```

Notice the ARN shows you're using the assumed role, not your base user.

### Set Default Profile

By default, AWS CLI uses the `default` profile. To use a different profile by default, set an environment variable:

```sh
# Use data-engineer role by default
export AWS_PROFILE=data-engineer
```

Add this to your shell configuration file (`~/.zshrc`) to persist:

```sh
echo 'export AWS_PROFILE=data-engineer' >> ~/.zshrc
source ~/.zshrc
```

### Switch Between Profiles

To temporarily use a different profile for a single command:

```sh
# Use admin role for this command only
aws s3 ls --profile admin

# Use data-engineer role for this command only
aws s3 ls --profile data-engineer

# Use base user (no role) for this command only
aws s3 ls --profile default
```

Or change the environment variable:

```sh
# Switch to admin role for subsequent commands
export AWS_PROFILE=admin

# Verify
aws sts get-caller-identity

# Switch back to data-engineer
export AWS_PROFILE=data-engineer
```

!!! tip "Verify Your Role"
    Get in the habit of running `aws sts get-caller-identity` when switching roles to confirm you're using the intended credentials. This prevents accidentally making changes with the wrong permissions.

## (Optional) Install and Configure aws-vault

For enhanced security, consider using [aws-vault](https://github.com/99designs/aws-vault) to manage AWS credentials. It stores credentials encrypted in your operating system's keychain and generates temporary session credentials.

### Install aws-vault

On macOS:

```sh
brew install aws-vault
```

### Configure aws-vault

Import your existing credentials:

```sh
aws-vault add default
```

When prompted, enter your Access Key ID and Secret Access Key. These are stored encrypted in your macOS Keychain.

You can now remove credentials from `~/.aws/credentials`:

```sh
# Back up first
cp ~/.aws/credentials ~/.aws/credentials.backup

# Remove credentials (keep config)
rm ~/.aws/credentials
```

### Use aws-vault

Execute commands with aws-vault:

```sh
# Using base IAM user
aws-vault exec default -- aws s3 ls

# Using data-engineer role
aws-vault exec data-engineer -- aws s3 ls

# Using admin role
aws-vault exec admin -- aws s3 ls
```

For interactive sessions:

```sh
# Start a shell with data-engineer credentials
aws-vault exec data-engineer

# All subsequent aws commands use those credentials
aws sts get-caller-identity
aws s3 ls

# Exit the session
exit
```

!!! info "Why Use aws-vault?"
    Benefits of aws-vault:

    - Credentials never stored in plaintext
    - Automatic session token rotation
    - MFA support for role assumption
    - Prevents accidental credential exposure
    - Better audit trail of credential usage

### Configure aws-vault with Profiles

Your `~/.aws/config` remains the same as before. aws-vault reads these profiles and uses the encrypted credentials from the Keychain.

Test your setup:

```sh
# Verify base user
aws-vault exec default -- aws sts get-caller-identity

# Verify data-engineer role
aws-vault exec data-engineer -- aws sts get-caller-identity

# Verify admin role
aws-vault exec admin -- aws sts get-caller-identity

# Verify infrastructure admin role
aws-vault exec infrastructure-admin -- aws sts get-caller-identity
```

## Best Practices Summary

!!! success "Security Checklist"
    - [x] Root user secured with MFA and strong password
    - [x] Root user only used for tasks that require it
    - [x] IAM users have MFA enabled
    - [x] No permissions attached directly to users (use roles)
    - [x] Separate roles for different permission levels
    - [x] Access keys stored securely (aws-vault or password manager)
    - [x] Never commit credentials to version control
    - [x] Cost alerts configured to prevent surprise bills
    - [x] IAM user uses `DataEngineerRole` by default
    - [x] `AdminRole` only used when necessary
    - [x] `InfrastructureAdminRole` used for Terraform state operations

## Next Steps

!!! success
    You now have a secure AWS account with proper IAM configuration and CLI access!

Your next step is to configure Terraform to manage AWS infrastructure as code. This ensures all future resources are:

- Version controlled
- Reproducible
- Peer reviewed
- Documented

Continue to [Set Up Terraform for AWS](../../build/infrastructure-as-code/terraform-fundamentals.md) →
