# Create a Snowflake Account

On this page, you will:

- [x] Create a new Snowflake account
- [x] Create an admin user with proper security
- [x] Update your Snowflake account identifier
- [x] Set up MFA and best practices
- [x] (Optional) Configure SSO

## Create a New Account

First, visit the [Snowflake sign-up link](https://signup.snowflake.com/) and complete the sign-up form. The details you provide will create a user with `ORGADMIN` and `ACCOUNTADMIN` privileges - the highest levels of access in Snowflake. Make sure you understand the [recommended usage of these roles](https://docs.snowflake.com/en/user-guide/security-access-control-considerations.html#using-the-accountadmin-role).

### Selecting Edition and Provider

For the initial setup, selecting the **Standard** edition provided by **AWS** is a good starting point. A drop-down menu will appear asking you to select your region - choose the region nearest to your data and that fulfills any local data residency requirements (e.g., GDPR).

!!! info "About Snowflake Editions"
    You can learn about different editions [here](https://docs.snowflake.com/en/user-guide/intro-editions). Standard is [cheaper](https://www.snowflake.com/en/data-cloud/pricing-options/) than Enterprise but lacks some features like multi-cluster warehouses, granular governance controls, and extended Time Travel windows. You can easily upgrade to Enterprise later, so Standard is sufficient for initial setup.

!!! info "About Cloud Providers"
    While AWS is used here (largest market share), you could choose [GCP or Azure](https://docs.snowflake.com/en/user-guide/intro-cloud-platforms.html) if that better aligns with your existing infrastructure. Most Snowflake features offer cross-cloud support. It's generally recommended to keep the same provider to reduce data transfer costs. Learn more about provider differences [here](https://www.digitalocean.com/resources/article/comparing-aws-azure-gcp).

## Create Your Admin User

After clicking 'Get Started', you'll receive an email asking you to create a username and password. Think carefully about your username structure. For admin accounts, consider a format like:

- `firstinitiallastname_admin` (e.g., `jbloggs_admin`)
- `firstname_lastname_admin` (e.g., `jane_bloggs_admin`)

!!! warning "Separate Admin Users from Regular Users"
    It's critical to have separate users for admin privileges versus regular work. For example, Jane Bloggs should have:

    - **Regular user**: `jbloggs` (for day-to-day work, connected to SSO)
    - **Admin user**: `jbloggs_admin` (with `ORGADMIN`/`ACCOUNTADMIN` access, NOT connected to SSO)

    This separation provides better security and audit trails.

### Password Requirements

Create a secure password. Snowflake's [default password policy](https://docs.snowflake.com/en/sql-reference/sql/create-password-policy) requires:

- 8-256 characters
- At least 1 uppercase letter
- At least 1 lowercase letter
- At least 1 number

!!! tip "Store in 1Password"
    Create a new entry in 1Password for "Snowflake Admin User" in the appropriate vault - only limited users should have access to this, so don't add to a more general "data engineer" type vault. Use 1Password to generate a strong password and store:

    - Account URL (you'll get this after sign-up)
    - Username
    - Password
    - You'll add MFA codes to this entry next

## Secure Your Admin User

### Enable MFA

As soon as you create the admin user, [enable MFA](https://docs.snowflake.com/en/user-guide/security-mfa.html) immediately. This is especially critical for accounts with admin privileges.

To enable MFA:

1. Click your username in the bottom-left corner
2. Select **My Profile**
3. Click **Enroll** next to Multi-factor authentication
4. Scan the QR code with 1Password (or your authenticator app)
5. Enter the verification code to confirm

!!! tip "Using 1Password for MFA"
    Add a one-time password field to your "Snowflake Admin User" entry in 1Password and scan the QR code. This keeps your password and MFA codes together securely.

!!! warning
    MFA cannot be set at an account level in Snowflake - it must be enabled for individual users. This is a manual step you should complete immediately after account creation.

### Set a Safe Default Role

Once logged in, access the SQL worksheet by clicking 'Worksheets', then the '+ Worksheet' button. Your first action should be to set a safe default role for your admin user.

Run the following query (update `username` to your actual username):

```sql
USE ROLE ORGADMIN;
ALTER USER username SET DEFAULT_ROLE = 'PUBLIC';
```

!!! tip "Running Queries in Snowflake"
    By default, Snowflake runs the **currently selected** query:

    - If text is highlighted, only that portion runs
    - Otherwise, it runs the statement between semicolons containing your cursor
    - To run all text, select everything with ++cmd+a++ (Mac) or ++ctrl+a++ (Windows)
    - Use ++cmd+return++ (Mac) or ++ctrl+return++ (Windows) as a keyboard shortcut to execute

    Learn more about [executing queries here](https://docs.snowflake.com/en/user-guide/ui-worksheet#executing-queries).

This ensures the admin user starts with minimal permissions and must explicitly elevate privileges when needed - a security best practice.

## Explore Your Account

Now that your user is properly secured, explore the account identifiers:

```sql
USE ROLE ORGADMIN;
SHOW ORGANIZATION ACCOUNTS;
```

In the results, look for the `ACCOUNT_URL` field. It follows the format:

```
https://organizationName-accountName.snowflakecomputing.com/
```

!!! tip "Update 1Password"
    Add the account URL to your "Snowflake Admin User" entry in 1Password. This makes it easy to find the correct login URL later.

### Update Organization Name

The organization name should be meaningful to your company. To [change it](https://docs.snowflake.com/en/user-guide/organizations-gs#changing-the-name-of-your-organization), you need to [contact Snowflake support](https://community.snowflake.com/s/article/How-To-Submit-a-Support-Case-in-Snowflake-Lodge). This can take time, so do it early in your setup process.

!!! info "Multiple Accounts"
    You can [create additional accounts](https://docs.snowflake.com/en/user-guide/organizations-manage-accounts.html) within your organization if needed, but this is beyond the scope of this initial setup.

## (Optional) Set Up SSO

After updating your organization name, consider linking your SSO provider to Snowflake. Setting up SSO doesn't force everyone to use it - it simply provides the option.

Configuration guides vary by provider:

- [Google (G Suite) SSO guide](https://community.snowflake.com/s/article/configuring-g-suite-as-an-identity-provider)
- [Okta SSO guide](https://docs.snowflake.com/en/user-guide/admin-security-fed-auth-use.html#okta)
- [Azure AD SSO guide](https://docs.snowflake.com/en/user-guide/admin-security-fed-auth-use.html#microsoft-azure-ad)

### Enable SSO for Users

To allow a user to authenticate via SSO in the future, set their `LOGIN_NAME` to their SSO email address:

```sql
USE ROLE ACCOUNTADMIN;
ALTER USER username
    SET LOGIN_NAME = 'first.last@email.com',
        DISPLAY_NAME = 'Firstname Lastname';
```

!!! warning
    Do **not** set your admin users (those with `ACCOUNTADMIN` or `ORGADMIN` roles) to use SSO. Keep them as separate, password-based accounts for emergency access and security separation.

## Next Steps

!!! success
    You now have a Snowflake data warehouse with a properly secured super-admin user!

Before creating any resources in Snowflake, you'll set up Terraform to manage your infrastructure as code. This ensures all future resources are:

- Version controlled
- Reproducible
- Documented
- Reviewed by your team

Continue to [Set Up Terraform](../terraform-setup/index.md) →
