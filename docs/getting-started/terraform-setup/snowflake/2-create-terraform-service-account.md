# Create Terraform Service Account

On this page, you will:

- [x] Generate an RSA key pair for authentication
- [x] Create the `SVC_TERRAFORM` service account in Snowflake
- [x] Store credentials securely in 1Password and AWS Secrets Manager
- [x] Update the provider to use key-pair authentication

## Why a Service Account?

Using your personal admin account for Terraform has several problems:

- **No audit trail**: Hard to distinguish your manual changes from Terraform changes
- **Password rotation**: If you change your password, Terraform breaks
- **Shared credentials**: You'd need to share your credentials with CI/CD
- **Overly broad access**: Your account likely has more permissions than Terraform needs

A dedicated service account (`SVC_TERRAFORM`) solves these issues:

- **Clear ownership**: All Terraform changes are attributed to `SVC_TERRAFORM`
- **Key-pair auth**: No password to rotate, more secure for automation
- **CI/CD ready**: Credentials stored in Secrets Manager for GitHub Actions
- **Least privilege**: Can be scoped to only what Terraform needs (though we start with ACCOUNTADMIN)

## Generate RSA Key Pair

First, generate an RSA key pair for the service account. Snowflake requires at least 2048-bit keys.

```sh
# Create a directory for Snowflake keys (if it doesn't exist)
mkdir -p ~/.snowflake

# Generate a 4096-bit private key (no passphrase for automation)
openssl genrsa 4096 | openssl pkcs8 -topk8 -inform PEM -out ~/.snowflake/svc_terraform_key.p8 -nocrypt

# Extract the public key
openssl rsa -in ~/.snowflake/svc_terraform_key.p8 -pubout -out ~/.snowflake/svc_terraform_key.pub
```

!!! warning "Key Security"
    The private key (`svc_terraform_key.p8`) must be kept secure:

    - Never commit it to version control
    - Store it in 1Password immediately
    - The file permissions should be restrictive (`chmod 600`)

!!! info "Why No Passphrase?"
    For automation (CI/CD), the private key cannot have a passphrase because there's no human to enter it. The key is protected by:

    - Restricted file permissions locally
    - Encryption in AWS Secrets Manager for CI/CD
    - Access controls on who can read the secret

## Create the Service Account in Snowflake

Connect to Snowflake with your admin user and create the service account:

```sql
USE ROLE ACCOUNTADMIN;

-- Create the service account user
CREATE USER SVC_TERRAFORM
    TYPE = SERVICE
    DEFAULT_ROLE = ACCOUNTADMIN
    MUST_CHANGE_PASSWORD = FALSE
    COMMENT = 'Service account for Terraform infrastructure management';

-- Grant ACCOUNTADMIN role
GRANT ROLE ACCOUNTADMIN TO USER SVC_TERRAFORM;
```

!!! info "User Type"
    Setting `TYPE = SERVICE` marks this as a service account. Service accounts:

    - Cannot log in via the web UI
    - Are clearly identified in audit logs
    - Are the recommended type for automation

Now set the public key for authentication. First, extract the key content:

```sh
# Extract just the key content (no headers, single line)
grep -v "PUBLIC KEY" ~/.snowflake/svc_terraform_key.pub | tr -d '\n'
```

This outputs a long base64 string like:

```
MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEA1234...5678IDAQAB
```

Copy the entire output and use it in the SQL command:

```sql
-- Set the public key on the user (replace with YOUR actual key output)
ALTER USER SVC_TERRAFORM SET RSA_PUBLIC_KEY = '<paste-your-entire-key-here>';
```

!!! warning "Replace the Entire String"
    Replace `<paste-your-entire-key-here>` with the complete output from the `grep` command. The key is typically 700+ characters long.

!!! tip "Verify the Key"
    After setting the public key, verify it was assigned:

    ```sql
    DESC USER SVC_TERRAFORM;
    ```

    Look for `RSA_PUBLIC_KEY_FP` - this is the fingerprint of the assigned public key.

## Store Credentials Securely

### Store in 1Password (Local Development)

Create a new entry in 1Password for the service account:

1. Create a new "Secure Note" or "API Credential" entry
2. Name it "Snowflake SVC_TERRAFORM"
3. Add the following fields:
   - **Account**: Your account identifier (e.g. `MYORG-MYACCOUNT`)
   - **Username**: `SVC_TERRAFORM`
   - **Private Key**: Paste the entire contents of `~/.snowflake/svc_terraform_key.p8`
   - **Public Key**: Paste the entire contents of `~/.snowflake/svc_terraform_key.pub`

!!! tip "1Password Document"
    Alternatively, upload the key files as document attachments. This preserves the exact formatting.

!!! info "Why Store the Public Key?"
    You'll need the public key if you ever need to rotate keys or set up another service account. Storing it alongside the private key keeps everything together.

### Store in AWS Secrets Manager (CI/CD)

First, create the secret container via Terraform. Add this resource to your AWS Terraform configuration (`terraform/aws/secrets.tf`):

```hcl
# Snowflake credentials for Terraform
resource "aws_secretsmanager_secret" "snowflake_credentials" {
  name        = "terraform/snowflake-credentials"
  description = "Snowflake credentials for Terraform (SVC_TERRAFORM service account)"

  tags = {
    Name = "terraform/snowflake-credentials"
  }
}
```

Apply the AWS configuration to create the secret container:

```sh
cd ../aws
terraform plan
terraform apply
cd ../snowflake
```

!!! warning "Local plan"
    Remember, we are only using `terraform plan` locally during this initial bootstrapping phase - all changes should be done via GitHub actions.

Now set the secret value via CLI:

```sh
# Read the private key into a variable (preserving newlines)
PRIVATE_KEY=$(cat ~/.snowflake/svc_terraform_key.p8)

# Set the secret value
aws secretsmanager put-secret-value \
    --secret-id "terraform/snowflake-credentials" \
    --secret-string "{
        \"organization_name\": \"MYORG\",
        \"account_name\": \"MYACCOUNT\",
        \"user\": \"SVC_TERRAFORM\",
        \"private_key\": $(echo "$PRIVATE_KEY" | jq -Rs .)
    }" \
    --profile infrastructure-admin
```

!!! warning "Replace Values"
    Replace `MYORG` and `MYACCOUNT` with your actual Snowflake organization and account names.

!!! info "JSON Escaping"
    The `jq -Rs .` command properly escapes the private key for JSON. The private key contains newlines which must be escaped as `\n` in JSON format.

Verify the secret was created:

```sh
aws secretsmanager describe-secret \
    --secret-id "terraform/snowflake-credentials" \
    --profile infrastructure-admin
```

## Update the Terraform Provider

Now update `providers.tf` to use key-pair authentication instead of password:

```hcl
# Snowflake Provider - key-pair authentication
provider "snowflake" {
  organization_name = var.snowflake_organization_name
  account_name      = var.snowflake_account_name
  user              = "SVC_TERRAFORM"
  authenticator     = "JWT"
  private_key       = var.SNOWFLAKE_PRIVATE_KEY
}
```

Update `variables.tf` to add the private key variable (and remove the password variable):

```hcl
# Snowflake Variables
variable "snowflake_organization_name" {
  description = "Snowflake organization name"
  type        = string
}

variable "snowflake_account_name" {
  description = "Snowflake account name"
  type        = string
}

variable "SNOWFLAKE_PRIVATE_KEY" {
  description = "Private key for SVC_TERRAFORM service account"
  type        = string
  sensitive   = true
}
```

Remove the `snowflake_password` variable if you added it in the previous page.

## Set Up Local Authentication

For local development, set the private key as an environment variable:

```sh
# Add to your .envrc file
export TF_VAR_SNOWFLAKE_PRIVATE_KEY=$(cat ~/.snowflake/svc_terraform_key.p8)
```

After updating `.envrc`, reload it:

```sh
direnv allow
```

!!! tip "1Password CLI Alternative"
    If you stored the key in 1Password, you can retrieve it directly:

    ```sh
    export TF_VAR_SNOWFLAKE_PRIVATE_KEY=$(op read "op://Vault/Snowflake SVC_TERRAFORM/Private Key")
    ```

    Or use a 1Password reference in `.envrc`:

    ```sh
    export TF_VAR_SNOWFLAKE_PRIVATE_KEY=$(op read "op://Vault/Snowflake SVC_TERRAFORM/Private Key")
    ```

## Test the New Authentication

Verify the service account can authenticate:

```sh
cd terraform/snowflake
terraform plan
```

Expected output:

```
No changes. Your infrastructure matches the configuration.
```

If you see authentication errors, check:

- The public key was set correctly on the user
- The private key environment variable is set
- The account identifier format is correct

!!! tip "Debug Authentication"
    To see more details about authentication, you can enable provider logging:

    ```sh
    export TF_LOG=DEBUG
    terraform plan
    ```

    Look for lines about authentication and JWT token generation.

## Clean Up Local Key Files

Once you've stored the keys securely in 1Password and AWS Secrets Manager, you can optionally remove the local key files:

```sh
# Keep the keys for now during setup
# After everything is working, you can remove them:
# rm ~/.snowflake/svc_terraform_key.p8
# rm ~/.snowflake/svc_terraform_key.pub
```

!!! warning "Keep Backups"
    Don't delete the key files until you've verified:

    1. Local authentication works (terraform plan succeeds)
    2. Both private and public keys are stored in 1Password
    3. Private key is stored in AWS Secrets Manager
    4. CI/CD can retrieve the keys (tested after finishing-up)

## Update GitHub Actions Permissions

The `TerraformGitHubActionsRole` needs permission to read the new Snowflake credentials secret. If you followed the AWS setup correctly, it should already have this permission through the `SecretsManagerAccess` statement that grants read access to `terraform/*` secrets.

Verify by checking your `iam_roles.tf` in the AWS Terraform directory:

```hcl
# This statement should exist in your terraform_github_actions policy
statement {
  sid       = "SecretsManagerAccess"
  actions   = ["secretsmanager:GetSecretValue"]
  resources = ["arn:aws:secretsmanager:${var.aws_region}:${var.aws_account_id}:secret:terraform/*"]
}
```

If this statement doesn't exist, add it to the `terraform_github_actions` policy document in `terraform/aws/iam_roles.tf`.

## Commit Your Work

Commit your progress:

```sh
git add terraform/snowflake/
git commit -m "Configure SVC_TERRAFORM service account with key-pair auth"
```

!!! warning "Double-Check Before Committing"
    Ensure you're NOT committing:

    - Private key files
    - Passwords
    - Any `TF_VAR_*` values

    The `.gitignore` should exclude these, but always double-check.

## Summary

You've created a secure service account for Terraform:

| Item | Value |
|------|-------|
| **Username** | `SVC_TERRAFORM` |
| **Type** | Service account |
| **Default role** | `ACCOUNTADMIN` |
| **Authentication** | RSA key-pair (JWT) |
| **Local credentials** | 1Password + environment variable |
| **CI/CD credentials** | AWS Secrets Manager (`terraform/snowflake-credentials`) |

## What's Next

With authentication configured, you can now import your existing admin user into Terraform management.

Continue to [import the admin user](3-import-admin-user.md) →
