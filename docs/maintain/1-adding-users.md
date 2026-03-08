# Runbook: Adding Users

## Summary

Add new team members or service accounts to the data stack. This covers Snowflake access (via Terraform), Prefect Cloud access, and dbt Cloud access.

## When to Use

- A new team member joins and needs data platform access
- A new service account is needed for a tool integration
- An existing user needs upgraded access (e.g. developer to admin)

## Prerequisites

- [ ] Access: Write access to the `terraform` repository
- [ ] Tools: Terraform CLI and AWS CLI configured with `infrastructure-admin` profile
- [ ] Context: User's name, email, and required role (admin, developer, or service account)

## Steps

### 1. Determine the User Category

| Category | Example | Default Role | Default Warehouse | Dedicated Role |
|----------|---------|--------------|-------------------|----------------|
| **Admin** | `JBLOGGS_ADMIN` | SYSADMIN | DEVELOPER | No |
| **Developer** | `JBLOGGS` | ANALYTICS_DEVELOPER | DEVELOPER | No |
| **Transformer** | `SVC_DBT` | ANALYTICS_TRANSFORMER | TRANSFORMING | Yes |
| **Reporter** | `SVC_LIGHTDASH` | ANALYTICS_REPORTER | REPORTING | Yes |
| **Loader** | `SVC_DLT` | Dedicated role | LOADING | Yes |

- **Admin users** have `SYSADMIN` as their default role and can manage Snowflake resources
- **Developer users** have `ANALYTICS_DEVELOPER` which grants read access to all source databases and write access to `ANALYTICS_DEV`
- **Service accounts** always use `SVC_` prefix, key-pair authentication, and `user_create_dedicated_role = true`

### 2. Add to Terraform Configuration

=== "Admin or Developer User"

    Edit `snowflake/config/users.auto.tfvars` and add the user to the appropriate list.

    **For a developer:**

    ```hcl
    developer_users = {
      # ... existing users ...
      JBLOGGS = {
        display_name = "Jane Bloggs"
        first_name   = "Jane"
        last_name    = "Bloggs"
        email        = "jane.bloggs@company.com"
      }
    }
    ```

    **For an admin:**

    ```hcl
    admin_users = {
      # ... existing users ...
      JBLOGGS_ADMIN = {
        display_name = "Jane Bloggs (Admin)"
        first_name   = "Jane"
        last_name    = "Bloggs"
        email        = "jane.bloggs@company.com"
      }
    }
    ```

    !!! info "Admin Naming Convention"
        Admin users use the `_ADMIN` suffix (e.g. `JBLOGGS_ADMIN`). A person can have both a developer and admin account - use the admin account only when elevated access is needed.

=== "Service Account"

    Add a new module block to `snowflake/config/users.tf`:

    ```hcl
    module "user_svc_<tool>" {
      source = "./modules/snowflake_user"

      providers = {
        snowflake.security_admin = snowflake.security_admin
        snowflake.user_admin     = snowflake.user_admin
      }

      user_name                  = "SVC_<TOOL>"
      user_display_name          = "<Tool> Service Account"
      user_comment               = "Service account for <tool>."
      user_is_service_account    = true
      user_create_dedicated_role = true
      user_default_warehouse     = module.warehouse_<workload>.warehouse_name

      user_additional_roles = [
        # Add functional roles as needed
      ]
    }
    ```

    Replace:

    - `<TOOL>` with the tool name in UPPER_CASE (e.g. `FIVETRAN`)
    - `<tool>` with the lowercase name (e.g. `fivetran`)
    - `<workload>` with the appropriate warehouse: `loading`, `transforming`, or `reporting`

### 3. Add Network Policy (if Needed)

If the user connects from a known set of IP addresses (e.g. an office network or a cloud service with static IPs), add them to a network policy.

Edit `snowflake/config/network_policies.auto.tfvars` and add the user to the appropriate policy's user list.

### 4. Run Terraform Plan

```sh
cd snowflake/config
terraform plan
```

Verify the plan shows:

- New user resource being created
- Correct default role and warehouse
- Dedicated role created (for service accounts)
- Role grants are correct
- No unexpected changes to existing resources

### 5. Create Pull Request

```sh
git checkout -b add-user-jbloggs
git add snowflake/config/users.auto.tfvars  # or users.tf for service accounts
git commit -m "Add Jane Bloggs as Snowflake developer"
git push -u origin add-user-jbloggs
```

Create a PR via GitHub. CI/CD runs `terraform plan` automatically. After approval, CI/CD applies the changes.

### 6. Provide Credentials

After the PR is merged and CI/CD has applied the changes:

=== "Human Users"

    1. Set the user's initial password via Snowflake UI (as USERADMIN)
    2. Send credentials via 1Password secure sharing
    3. Ask the user to change their password and enable MFA on first login

=== "Service Accounts"

    1. Generate a key pair:

        ```sh
        openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out svc_<tool>_key.pem -nocrypt
        openssl rsa -in svc_<tool>_key.pem -pubout -out svc_<tool>_key.pub
        ```

    2. Set the public key on the Snowflake user:

        ```sql
        USE ROLE USERADMIN;
        ALTER USER SVC_<TOOL> SET RSA_PUBLIC_KEY = '<public key content>';
        ```

    3. Store the private key in AWS Secrets Manager:

        ```sh
        aws secretsmanager put-secret-value \
            --secret-id "<tool>/snowflake-credentials" \
            --secret-string "{
              \"account\": \"YOUR_ACCOUNT\",
              \"user\": \"SVC_<TOOL>\",
              \"private_key\": \"$(cat svc_<tool>_key.pem)\"
            }" \
            --profile infrastructure-admin
        ```

    4. Delete the local key files:

        ```sh
        rm svc_<tool>_key.pem svc_<tool>_key.pub
        ```

!!! tip "Claude Code Automation"
    If you have the `add-snowflake-user` Claude skill configured, Claude can perform steps 2-5 automatically. Run the skill in the terraform repository and provide the user details. See [Claude Code Setup](../getting-started/initial-setup/5-claude-code-setup.md) for configuration.

### 7. Add to Other Tools (if Needed)

=== "Prefect Cloud"

    1. Navigate to **Prefect Cloud** → **Settings** → **Members**
    2. Click **Invite** and enter the user's email
    3. Assign a workspace role:

        | Role | Access |
        |------|--------|
        | **Owner** | Full access, billing, workspace deletion |
        | **Admin** | Manage deployments, work pools, members |
        | **Worker** | Run flows, view deployments |
        | **Viewer** | Read-only access to flows and runs |

=== "dbt Cloud"

    1. Navigate to **dbt Cloud** → **Account Settings** → **Users**
    2. Click **Invite Users** and enter the user's email
    3. Assign a role:

        | Role | Access |
        |------|--------|
        | **Owner** | Full account access |
        | **Admin** | Manage projects, environments, users |
        | **Developer** | Develop in the IDE, trigger runs |
        | **Read-Only** | View runs and documentation |

    The user's dbt Cloud runs will use the `SVC_DBT` service account's Snowflake credentials - individual Snowflake access is not needed for dbt Cloud users.

## Verification

After completing the steps, verify:

- [ ] Snowflake user exists:

    ```sql
    SHOW USERS LIKE 'JBLOGGS';
    ```

- [ ] Correct roles assigned:

    ```sql
    SHOW GRANTS TO USER JBLOGGS;
    ```

- [ ] User can log in and run a test query:

    ```sql
    SELECT CURRENT_USER(), CURRENT_ROLE(), CURRENT_WAREHOUSE();
    ```

- [ ] Prefect: user visible in workspace members (if applicable)
- [ ] dbt Cloud: user can trigger a manual run (if applicable)

## Rollback

If something goes wrong:

1. **Revert the PR** - GitHub → PR → Revert. Merge the revert PR and CI/CD runs `terraform apply` to remove the user
2. **Or manually** (as USERADMIN):

    ```sql
    DROP USER IF EXISTS JBLOGGS;
    ```

3. **Remove from other tools** - delete the user from Prefect Cloud and/or dbt Cloud via their respective UIs

## Escalation

- **First contact**: Data Engineering team in #data-eng Slack
- **Escalation**: Infrastructure team lead

## See Also

- [Users](../build/data-warehouse/5-users.md) - Full guide on the Snowflake user module and configuration
- [Network Policies](../build/data-warehouse/7-network-policies.md) - IP allowlisting configuration
- [Claude Code Setup](../getting-started/initial-setup/5-claude-code-setup.md) - Configure the `add-snowflake-user` skill
