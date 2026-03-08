# AWS SSO Setup (Optional)

This page covers setting up AWS IAM Identity Centre (formerly AWS SSO) as an
alternative to the IAM user approach described in [AWS Account Setup](aws.md).
IAM Identity Centre is recommended for teams of five or more, or organisations
that already use an identity provider such as Okta, Azure AD, or Google
Workspace.

On this page, you will:

- [x] Understand when IAM Identity Centre is preferable to IAM users
- [x] Enable IAM Identity Centre in your AWS account
- [x] Create permission sets that mirror the existing role model
- [x] Configure AWS CLI for SSO-based access
- [x] Understand the Terraform integration

## Overview

The main documentation uses **IAM users with role assumption** — you create IAM
users, assign them policies that let them assume roles (`DataEngineerRole`,
`InfrastructureAdminRole`, `AdminRole`), and configure CLI profiles for each
role. This approach is straightforward for small teams.

**IAM Identity Centre** centralises identity management. Instead of creating
individual IAM users with long-lived access keys, team members authenticate
through a central identity provider and receive temporary credentials
automatically.

```
IAM Users Approach                    IAM Identity Centre Approach
──────────────────                    ────────────────────────────

IAM User (jane.bloggs)                Identity Provider (Okta/Azure AD)
  │                                     │
  ├── AssumeDataEngineerPolicy          ├── Permission Set: DataEngineer
  ├── AssumeInfraAdminPolicy            ├── Permission Set: InfraAdmin
  └── Long-lived access keys            └── Temporary credentials (12h)
         │                                       │
     ~/.aws/credentials                      aws sso login
     (rotate manually)                    (browser-based, automatic)
```

## When to Use IAM Identity Centre

| Scenario | Recommendation |
|----------|---------------|
| Solo developer or 2-person team | IAM users (simpler setup) |
| Team of 3-5 without existing IdP | Either approach works |
| Team of 5+ | IAM Identity Centre |
| Organisation with Okta/Azure AD/Google Workspace | IAM Identity Centre |
| Compliance requirements (SOC 2, ISO 27001) | IAM Identity Centre (audit trail, no long-lived keys) |
| Multiple AWS accounts | IAM Identity Centre (manages cross-account access) |

!!! info "Not Mutually Exclusive"
    You can use IAM Identity Centre for human users while keeping IAM service
    accounts (like `TerraformGitHubActionsRole`) for CI/CD. The two approaches
    coexist without conflict.

## Enable IAM Identity Centre

### Step 1: Enable the Service

1. Sign in to the AWS Console as the root user or an admin
2. Navigate to **IAM Identity Centre** (search for "IAM Identity Centre" in the
   console search bar)
3. Click **Enable**
4. Choose your identity source:

=== "IAM Identity Centre Directory (Built-in)"

    Use the built-in directory if you do not have an existing identity provider.
    You create and manage users directly in IAM Identity Centre.

    1. Select **Identity Centre directory** (the default)
    2. Click **Enable**

    This is the simplest option for small teams without an existing IdP.

=== "External Identity Provider (Okta, Azure AD, Google Workspace)"

    Connect your existing identity provider for centralised user management.

    1. Select **External identity provider**
    2. Download the IAM Identity Centre SAML metadata file
    3. In your identity provider (e.g. Okta):
        - Create a new SAML application for AWS
        - Upload the IAM Identity Centre metadata
        - Configure attribute mappings (email, name, groups)
        - Download the IdP SAML metadata
    4. Back in IAM Identity Centre, upload the IdP metadata
    5. Enable automatic provisioning (SCIM) for user/group sync

    !!! tip "SCIM Provisioning"
        SCIM automatically creates and deactivates users in AWS when you add or
        remove them from your identity provider. Without SCIM, you must manually
        manage users in both systems.

### Step 2: Create Users (Built-in Directory Only)

If using the built-in directory, create users manually:

1. Navigate to **Users** in IAM Identity Centre
2. Click **Add user**
3. Enter the user's email, first name, and last name
4. The user receives an email to set their password and configure MFA

### Step 3: Create Groups

Create groups that map to your permission model:

| Group | Purpose | Maps to |
|-------|---------|---------|
| `DataEngineers` | Day-to-day data platform work | `DataEngineerRole` |
| `InfrastructureAdmins` | Terraform operations | `InfrastructureAdminRole` |
| `Administrators` | Account administration | `AdminRole` |

1. Navigate to **Groups** in IAM Identity Centre
2. Create each group
3. Add users to the appropriate groups

## Create Permission Sets

Permission sets define what users can do when they access an AWS account. Create
one permission set per role, matching the existing role-based access model.

### DataEngineer Permission Set

1. Navigate to **Permission sets** in IAM Identity Centre
2. Click **Create permission set**
3. Select **Custom permission set**
4. Name: `DataEngineer`
5. Session duration: 12 hours
6. Attach the same IAM policies that `DataEngineerRole` uses:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::your-data-lake-*",
        "arn:aws:s3:::your-data-lake-*/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:*:*:secret:*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::your-terraform-state-bucket",
        "arn:aws:s3:::your-terraform-state-bucket/*"
      ]
    }
  ]
}
```

!!! tip "Reuse Existing Policies"
    If you have already created managed policies for your IAM roles, you can
    attach those same policies to the permission set rather than defining
    inline JSON. This keeps both approaches in sync.

### InfrastructureAdmin Permission Set

Follow the same process with the policies from `InfrastructureAdminRole` — full
read/write access to Terraform state (S3 + DynamoDB), IAM read access, and
Secrets Manager access.

### Administrator Permission Set

Attach the AWS-managed `AdministratorAccess` policy. Restrict this permission
set to the `Administrators` group.

### Assign Permission Sets to Groups

1. Navigate to **AWS accounts** in IAM Identity Centre
2. Select your AWS account
3. Click **Assign users or groups**
4. Select a group (e.g. `DataEngineers`)
5. Select the corresponding permission set (e.g. `DataEngineer`)
6. Repeat for each group/permission set combination

## Configure AWS CLI for SSO

### Step 1: Configure SSO Profile

Run the SSO configuration wizard:

```sh
aws configure sso
```

You will be prompted for:

```
SSO session name: my-sso
SSO start URL: https://d-xxxxxxxxxx.awsapps.com/start
SSO region: eu-west-2
SSO registration scopes: sso:account:access
```

The CLI opens your browser for authentication. After signing in, select your
account and permission set. The CLI writes the configuration to `~/.aws/config`:

```ini
[profile data-engineer]
sso_session = my-sso
sso_account_id = 123456789012
sso_role_name = DataEngineer
region = eu-west-2

[profile infrastructure-admin]
sso_session = my-sso
sso_account_id = 123456789012
sso_role_name = InfrastructureAdmin
region = eu-west-2

[profile admin]
sso_session = my-sso
sso_account_id = 123456789012
sso_role_name = Administrator
region = eu-west-2

[sso-session my-sso]
sso_start_url = https://d-xxxxxxxxxx.awsapps.com/start
sso_region = eu-west-2
sso_registration_scopes = sso:account:access
```

### Step 2: Log In

```sh
aws sso login --profile data-engineer
```

This opens your browser. After authenticating, the CLI receives temporary
credentials that last for the session duration (default 12 hours).

### Step 3: Verify

```sh
aws sts get-caller-identity --profile data-engineer
```

Expected output:

```json
{
    "UserId": "AROAEXAMPLEROLEID:jane.bloggs@company.com",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/AWSReservedSSO_DataEngineer_abc123/jane.bloggs@company.com"
}
```

### Step 4: Set Default Profile

```sh
export AWS_PROFILE=data-engineer
```

Add to your shell configuration:

```sh
echo 'export AWS_PROFILE=data-engineer' >> ~/.zshrc
```

!!! info "No Long-Lived Credentials"
    Unlike IAM users, SSO profiles do not store access keys in
    `~/.aws/credentials`. Credentials are temporary and refreshed
    automatically when you run `aws sso login`. If your session expires,
    the CLI prompts you to re-authenticate.

## Terraform Integration

### AWS Provider Configuration

Update the AWS provider to use SSO profiles:

```hcl
provider "aws" {
  region  = "eu-west-2"
  profile = "infrastructure-admin"
}
```

Terraform reads the SSO profile from `~/.aws/config` and uses the temporary
credentials. No changes to your Terraform code are needed beyond the profile
name — the same `--profile` flag works for both IAM user and SSO profiles.

### CI/CD (No Change)

CI/CD pipelines continue to use OIDC authentication with
`TerraformGitHubActionsRole`. SSO is for human users only — service accounts
and automation use IAM roles with OIDC or key-pair authentication.

```yaml
# .github/workflows/terraform.yml — unchanged
- name: Configure AWS Credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/TerraformGitHubActionsRole
    aws-region: eu-west-2
```

### Managing IAM Identity Centre with Terraform

You can optionally manage IAM Identity Centre resources with Terraform using the
`aws_ssoadmin_*` resources:

```hcl
# Permission set
resource "aws_ssoadmin_permission_set" "data_engineer" {
  name             = "DataEngineer"
  instance_arn     = tolist(data.aws_ssoadmin_instances.this.arns)[0]
  session_duration = "PT12H"
  description      = "Day-to-day data engineering access."
}

# Attach policy to permission set
resource "aws_ssoadmin_managed_policy_attachment" "data_engineer" {
  instance_arn       = tolist(data.aws_ssoadmin_instances.this.arns)[0]
  managed_policy_arn = aws_iam_policy.data_engineer.arn
  permission_set_arn = aws_ssoadmin_permission_set.data_engineer.arn
}

# Assign permission set to group for an account
resource "aws_ssoadmin_account_assignment" "data_engineers" {
  instance_arn       = tolist(data.aws_ssoadmin_instances.this.arns)[0]
  permission_set_arn = aws_ssoadmin_permission_set.data_engineer.arn

  principal_id   = aws_identitystore_group.data_engineers.group_id
  principal_type = "GROUP"

  target_id   = data.aws_organizations_organization.this.roots[0].id
  target_type = "AWS_ACCOUNT"
}
```

!!! warning "AWS Organisations Required"
    IAM Identity Centre requires AWS Organisations to be enabled. If you are
    using a standalone AWS account, enabling IAM Identity Centre automatically
    creates an organisation with your account as the management account.

## Migration from IAM Users

If you started with IAM users and want to migrate to IAM Identity Centre:

1. **Enable IAM Identity Centre** and create permission sets (above)
2. **Create SSO users/groups** matching your existing IAM users
3. **Update CLI profiles** — replace credential-based profiles with SSO profiles
4. **Test all workflows** — verify Terraform plan, AWS CLI commands, and
   application access
5. **Deactivate IAM user access keys** — do not delete users immediately;
   deactivate keys first and monitor for breakage
6. **Remove IAM users** after a validation period (1-2 weeks)

!!! danger "Do Not Delete IAM Users Before Testing"
    Deactivate access keys first and leave IAM users in place for a rollback
    window. Some applications or scripts may still reference IAM credentials.
    Only delete IAM users after confirming all access has migrated.

## Summary

| Aspect | IAM Users | IAM Identity Centre |
|--------|-----------|-------------------|
| **Credentials** | Long-lived access keys | Temporary (12h default) |
| **MFA** | Per-user configuration | Centralised policy |
| **User management** | AWS Console or Terraform | IdP or built-in directory |
| **Audit trail** | CloudTrail | CloudTrail + IdP logs |
| **Multi-account** | Separate IAM users per account | Single identity across accounts |
| **Setup complexity** | Lower | Higher (but scales better) |
| **Best for** | Small teams (1-4 people) | Growing teams, compliance requirements |

!!! success "Key Takeaways"
    - [x] IAM Identity Centre replaces long-lived access keys with temporary credentials
    - [x] Permission sets map directly to your existing role model
    - [x] CLI profiles work the same way — just `aws sso login` instead of storing keys
    - [x] CI/CD is unaffected — continue using OIDC with `TerraformGitHubActionsRole`
    - [x] You can migrate incrementally from IAM users to SSO

## See Also

- [AWS Account Setup](aws.md) — the IAM user approach used in the main documentation
- [SSO Setup in Snowflake](../../build/data-warehouse/9-sso-setup.md) — SAML integration for Snowflake
- [AWS IAM Identity Centre documentation](https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html)
