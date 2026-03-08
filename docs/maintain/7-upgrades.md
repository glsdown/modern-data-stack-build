# Runbook: Upgrades

## Summary

Upgrade the tools and services in the data stack - Snowflake provider versions, dbt packages, Prefect server/worker, Airbyte, Lightdash, and Terraform providers. This runbook covers safe upgrade procedures with testing and rollback strategies.

## When to Use

- A new version of a tool is available with needed features or bug fixes
- Security patches require immediate upgrades
- Deprecation warnings indicate an upcoming breaking change
- Scheduled quarterly dependency review

## Prerequisites

- [ ] Access: Write access to the relevant repository
- [ ] Tools: Development environment configured for the tool being upgraded
- [ ] Context: Current version, target version, and changelog review

!!! warning "Read the Changelog"
    Always read the changelog and migration guide before upgrading. Pay special attention to breaking changes, deprecated features, and new requirements.

## Steps

### 1. Pre-Upgrade Checklist

Before any upgrade:

- [ ] Read the release notes and changelog for all versions between current and target
- [ ] Check for breaking changes that affect your configuration
- [ ] Create a branch for the upgrade: `git checkout -b upgrade/<tool>-<version>`
- [ ] Ensure CI/CD is passing on `main` before starting

### 2. Upgrade Terraform Providers

#### Snowflake Provider

The Snowflake Terraform provider (`Snowflake-Labs/snowflake`) is updated frequently. Check the [changelog](https://github.com/Snowflake-Labs/terraform-provider-snowflake/blob/main/CHANGELOG.md).

1. **Update the version constraint** in `snowflake/config/main.tf`:

    ```hcl
    terraform {
      required_providers {
        snowflake = {
          source  = "Snowflake-Labs/snowflake"
          version = "~> 0.100"  # update to new version
        }
      }
    }
    ```

2. **Run init to download the new provider:**

    ```sh
    cd snowflake/config
    terraform init -upgrade
    ```

3. **Run plan and review carefully:**

    ```sh
    terraform plan
    ```

    !!! danger "Watch for Replacements"
        If the plan shows resources being destroyed and recreated (`-/+`), the new provider version may have changed resource schemas. This can cause data loss. Review each replacement carefully and consider using `lifecycle { prevent_destroy = true }` as a safety net.

4. **Create a PR** with the updated `.terraform.lock.hcl` and version constraints

#### AWS Provider

Follow the same pattern for the AWS provider in `aws/config/main.tf`. The AWS provider is generally more stable between minor versions.

#### GitHub Provider

Follow the same pattern for the GitHub provider in `github/main.tf`.

### 3. Upgrade dbt

#### dbt Core

1. **Update the version** in the dbt-transform repository:

    ```sh
    uv add dbt-snowflake@<new_version>
    ```

2. **Run deps to update packages:**

    ```sh
    dbt deps
    ```

3. **Run the full test suite locally:**

    ```sh
    dbt build
    ```

4. **Check for deprecation warnings** in the output - these indicate breaking changes in the next major version

5. **Update CI/CD** if the GitHub Actions workflow pins a specific dbt version

!!! info "dbt Minor vs Major Upgrades"
    Minor version upgrades (e.g. 1.7 → 1.8) may include new features but are generally backwards-compatible. Major version upgrades (e.g. 1.x → 2.x) will have breaking changes. Plan major upgrades as a dedicated effort.

#### dbt Packages

Update packages in `packages.yml`:

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: [">=1.3.0", "<2.0.0"]
  - package: calogica/dbt_expectations
    version: [">=0.10.0", "<1.0.0"]
  - package: elementary-data/elementary
    version: [">=0.16.0", "<1.0.0"]
```

Then:

```sh
dbt deps
dbt build --select package:dbt_utils package:dbt_expectations package:elementary
```

#### dbt Cloud

If using dbt Cloud, update the environment settings:

1. Navigate to **dbt Cloud** → **Environments** → select environment
2. Update the **dbt version** setting
3. Run a test job to verify

### 4. Upgrade Prefect

#### Prefect Cloud

Prefect Cloud is managed by Prefect - no action needed for server upgrades. Update the client library:

```sh
cd ~/projects/data/data-pipelines
uv add prefect@<new_version>
```

Test flows locally:

```sh
python -m flows.currencies
```

#### Self-Hosted Prefect (Docker Compose)

1. **Update the image tag** in `docker-compose.yml`:

    ```yaml
    services:
      prefect-server:
        image: prefecthq/prefect:<new_version>-python3.11
    ```

2. **Pull and restart:**

    ```sh
    docker compose pull
    docker compose up -d
    ```

3. **Check the server health:**

    ```sh
    curl http://localhost:4200/api/health
    ```

#### Self-Hosted Prefect (ECS)

1. **Update the task definition** in Terraform with the new image tag
2. **Create a PR** and let CI/CD deploy the update
3. **Monitor the ECS service** for healthy task replacement

### 5. Upgrade Airbyte

#### Airbyte Cloud

Managed by Airbyte - no action needed. Connector versions update automatically.

#### Airbyte Self-Hosted

1. **Back up the database** before upgrading
2. **Update the image tags** in your deployment configuration
3. **Follow Airbyte's migration guide** for your version - some upgrades require database migrations
4. **Verify all connections sync successfully** after the upgrade

### 6. Upgrade Lightdash

#### Self-Hosted Lightdash

1. **Check the [Lightdash changelog](https://github.com/lightdash/lightdash/releases)** for breaking changes
2. **Update the image tag** in your ECS task definition or Docker Compose
3. **Deploy** and verify charts and dashboards still render correctly

### 7. Upgrade Confluent/Kafka

#### Confluent Cloud

Managed by Confluent - cluster upgrades are transparent. Update client libraries:

```sh
cd ~/projects/data/data-pipelines
uv add confluent-kafka@<new_version>
```

#### MSK Self-Hosted

Follow [AWS MSK upgrade documentation](https://docs.aws.amazon.com/msk/latest/developerguide/version-upgrades.html). MSK supports in-place version upgrades for minor versions.

## Verification

After any upgrade:

- [ ] `terraform plan` shows no unexpected changes
- [ ] `dbt build` completes successfully with all tests passing
- [ ] Prefect flows run successfully (trigger a manual test run)
- [ ] All Airbyte connections sync successfully
- [ ] Dashboards load correctly in Lightdash/Snowsight
- [ ] No new errors in logs
- [ ] CI/CD pipelines pass

## Rollback

=== "Terraform Provider"

    Revert the version constraint in `main.tf` and run `terraform init -upgrade`. The lock file (`.terraform.lock.hcl`) will revert to the previous version.

=== "dbt"

    ```sh
    uv add dbt-snowflake@<previous_version>
    dbt deps
    dbt build
    ```

=== "Prefect"

    ```sh
    uv add prefect@<previous_version>
    ```

    For self-hosted, revert the image tag and redeploy.

=== "Docker-Based Services"

    Revert the image tag in the deployment configuration and redeploy. Previous container images remain cached.

## Escalation

- **First contact**: Data Engineering team in #data-eng Slack
- **Terraform provider issues**: [GitHub Issues](https://github.com/Snowflake-Labs/terraform-provider-snowflake/issues) for the Snowflake provider
- **dbt issues**: [dbt Community](https://community.getdbt.com/) or [GitHub Issues](https://github.com/dbt-labs/dbt-core/issues)
- **Prefect issues**: [Prefect Community](https://community.prefect.io/) or [GitHub Issues](https://github.com/PrefectHQ/prefect/issues)

## See Also

- [CI/CD Deployment](../build/orchestration/9-ci-cd-deployment.md) - Automated deployment workflows
- [dbt Core Deployment](../build/data-transformation/9-dbt-core-deployment.md) - dbt CI/CD configuration
- [Terraform Deployment](../getting-started/terraform-setup/4-terraform-deployment.md) - Terraform CI/CD
