# Terraform Infrastructure Repository

This repository manages all infrastructure as code for the data platform using Terraform. It provisions and configures resources across GitHub, AWS, and Snowflake.

There are skills available to help with common maintenance tasks - `add-snowflake-user` and `add-data-source`.

---

## Repository Structure

```
terraform/
├── github/                     # GitHub organisation, teams, users
│   ├── backend.tf
│   ├── main.tf
│   ├── providers.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── terraform.tfvars
├── aws/                        # AWS resources (S3, IAM, VPC, Secrets Manager)
│   ├── config/
│   │   ├── backend.tf
│   │   ├── main.tf
│   │   ├── providers.tf
│   │   ├── variables.tf
│   │   ├── s3.tf
│   │   ├── vpc.tf
│   │   ├── iam.tf
│   │   └── secrets.tf
│   └── modules/
│       ├── s3_bucket/
│       └── vpc/
├── snowflake/                  # Snowflake data warehouse
│   ├── config/
│   │   ├── backend.tf
│   │   ├── main.tf
│   │   ├── providers.tf
│   │   ├── variables.tf
│   │   ├── warehouses.tf
│   │   ├── databases.tf
│   │   ├── functional_roles.tf
│   │   ├── users.tf
│   │   ├── schemas.tf
│   │   ├── network_policies.tf
│   │   ├── storage_integrations.tf
│   │   ├── resource_monitor.tf
│   │   ├── sso_integrations.tf
│   │   ├── *.auto.tfvars       # Variable values
│   │   └── terraform.tfvars
│   └── modules/
│       ├── snowflake_database/           # DB + DB_READER + DB_WRITER roles + grants
│       ├── snowflake_user/               # User + optional dedicated role + warehouse + grants
│       ├── snowflake_warehouse/          # Warehouse + usage grants
│       ├── snowflake_role/               # Account-level role
│       ├── snowflake_database_role/      # Database-level role + grants
│       ├── snowflake_schema/             # Schema + grants
│       ├── snowflake_storage_integration/# S3 storage integration
│       ├── snowflake_saml2_integration/  # SSO integration
│       └── snowflake_snowpipe/           # Stage + file format + pipe + SQS notification
└── terraform.tf                # Root-level Terraform version constraint
```

---

## Key Patterns

### Multiple Snowflake Providers

Each Snowflake module accepts provider aliases for different admin roles:

- `snowflake.sys_admin` - Create warehouses, databases
- `snowflake.security_admin` - Manage grants and roles
- `snowflake.user_admin` - Create users
- `snowflake.account_admin` - Account-level settings (rarely used)

All modules declare their required providers in their `main.tf` via `configuration_aliases`.

### Module Design

Each module creates a **complete resource** with all associated permissions:

- `snowflake_database` - Creates database + `DB_READER` / `DB_WRITER` database roles + grants
- `snowflake_user` - Creates user + optional dedicated role (`user_create_dedicated_role`) + optional dedicated warehouse + role grants
- `snowflake_warehouse` - Creates warehouse + usage grants to specified roles
- `snowflake_schema` - Creates schema + read/write grants
- `snowflake_snowpipe` - Creates stage + file format + pipe + SQS event notification

### Service Account Pattern

Service accounts use the `SVC_` prefix: `SVC_TERRAFORM`, `SVC_DLT`, `SVC_DBT`, `SVC_AIRBYTE`, `SVC_KAFKA_CONNECTOR`, `SVC_LIGHTDASH`.

When `user_create_dedicated_role = true`, the user module creates a role matching the user name (e.g. `SVC_DLT` gets a role named `SVC_DLT`). This is always used for service accounts.

### Database Naming

Databases are named after the loader tool: `DLT`, `SNOWPIPE`, `AIRBYTE`, `STREAMING`. The `ANALYTICS` and `ANALYTICS_DEV` databases are for the transformation layer.

### Reader Access Chain

```
{DB}_DB_READER → ANALYTICS_SOURCES_READER → ANALYTICS_DEVELOPER
                                           → ANALYTICS_TRANSFORMER
```

Every new loader database must grant its `DB_READER` role to `ANALYTICS_SOURCES_READER` to maintain this chain.

### State Management

- S3 backend with DynamoDB locking
- Provider-specific state keys (`github/`, `aws/`, `snowflake/`)
- Same S3 bucket, different state paths
- CI/CD uses `TerraformGitHubActionsRole` with OIDC authentication

### IAM Policy Patterns

- Use `aws_iam_policy_document` for all IAM policies (not JSON templates)
- Validates at plan time and references variables/resources directly
- Single-role assume policies for flexible user assignment

---

## Naming Conventions

| Object | Convention | Examples |
|--------|-----------|----------|
| Snowflake objects | UPPER_CASE | `ANALYTICS`, `SVC_DBT`, `LOADING` |
| Terraform resources | snake_case | `module.database_dlt`, `module.user_svc_dbt` |
| Module directories | snake_case | `snowflake_database/`, `snowflake_user/` |
| Service accounts | `SVC_` prefix | `SVC_DLT`, `SVC_AIRBYTE`, `SVC_TERRAFORM` |
| Database roles | `{DB}_DB_READER` / `{DB}_DB_WRITER` | `DLT_DB_READER`, `AIRBYTE_DB_WRITER` |
| Functional roles | `ANALYTICS_` prefix | `ANALYTICS_DEVELOPER`, `ANALYTICS_TRANSFORMER` |

---

## Common Operations

### Adding a New Snowflake User

Use the `add-snowflake-user` skill. The process is:

1. Determine user category (admin, developer, or service account)
2. Add to the appropriate variable list in `*.auto.tfvars` (admin/developer) or add a module call in `users.tf` (service account)
3. Run `terraform plan` in `snowflake/config/` to verify
4. Create PR — CI/CD runs plan automatically and applies after approval

### Adding a New Data Source

Use the `add-data-source` skill. The process is:

1. Add database module call in `databases.tf` (named after the loader tool)
2. Create service account in `users.tf` with `user_create_dedicated_role = true`
3. Add schema(s) to the database
4. Grant `DB_READER` to `ANALYTICS_SOURCES_READER`
5. Grant `DB_WRITER` to the service account's dedicated role
6. Add AWS Secrets Manager container in `aws/config/` for credentials
7. Run `terraform plan` in both `snowflake/config/` and `aws/config/`
8. Create PR — CI/CD applies after approval

---

## Safety Rules

- **Never** run `terraform apply` locally against production — always use CI/CD via pull requests
- **Always** run `terraform plan` before creating a PR to verify changes
- **Never** hard-code ARNs, account IDs, or secret values — use variables and resource references
- **Never** commit `.envrc` or secrets to git
- **Never** set passwords in Terraform — use key-pair authentication for service accounts
- Use `lifecycle { ignore_changes }` for sensitive fields (passwords, RSA keys) — the modules already handle this
- Use `aws_iam_policy_document` for all IAM policies (not inline JSON)
- Test module changes in the development workspace first

---

## Authentication

| Context | Method |
|---------|--------|
| Snowflake service accounts | Key-pair authentication (not passwords) |
| AWS CI/CD | OIDC via `TerraformGitHubActionsRole` |
| AWS local | CLI profiles (`data-engineer`, `infrastructure-admin`) |
| GitHub provider | Personal access token in `.envrc` |

Private keys are stored in 1Password (local) and AWS Secrets Manager (CI/CD).

---

## Style

- Use British English: organise, customise, analyse, colour, centre
- Use spaced hyphens ( - ) for parenthetical statements, not em dashes
