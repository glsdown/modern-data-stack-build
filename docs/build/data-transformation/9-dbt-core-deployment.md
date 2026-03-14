# dbt Core: CI/CD and Deferral

On this page, you will:

- [x] Set up GitHub Actions for slim CI on pull requests
- [x] Set up a production deployment workflow on merge to main
- [x] Configure state-based deferral using S3 artifacts
- [x] Deploy the dbt docs site to S3 and CloudFront

## Overview

dbt Core CI/CD involves two GitHub Actions workflows:

1. **`ci.yml`** — runs on every pull request. Uses the production `manifest.json` to run only changed models and their downstream dependencies (slim CI). Fails the PR if any tests or `dbt_project_evaluator` checks fail.

2. **`deploy.yml`** — runs on every merge to `main`. Runs a full `dbt build`, uploads the new `manifest.json` to S3, and deploys the updated docs site.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       dbt CORE CI/CD PIPELINE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Pull Request                         Merge to main                         │
│  ────────────                         ─────────────                         │
│                                                                             │
│  1. Fetch manifest.json from S3       1. dbt build (all models)             │
│  2. dbt source freshness              2. dbt test (all tests)               │
│  3. dbt build --select                3. dbt_project_evaluator              │
│       state:modified+                 4. Upload manifest.json → S3          │
│       --defer                         5. dbt docs generate                  │
│       --state .artifacts/             6. Deploy docs → S3 + CloudFront      │
│  4. dbt_project_evaluator                                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Prerequisites

### S3 Bucket for Artifacts

Create an S3 bucket in your AWS Terraform for storing dbt state artifacts and the docs site:

```hcl
# terraform/aws/s3.tf — add alongside existing buckets
module "bucket_dbt_artifacts" {
  source = "./modules/s3_bucket"

  bucket_name = "your-org-dbt-artifacts"
  description = "dbt state artifacts (manifest.json) and docs site"

  versioning_enabled = true

  lifecycle_rules = [{
    id      = "expire-old-artifacts"
    enabled = true
    prefix  = "artifacts/"
    expiration_days = 90
  }]

  tags = {
    Project   = "data-platform"
    ManagedBy = "terraform"
  }
}
```

The bucket has two "folders" by convention:

- `s3://your-org-dbt-artifacts/artifacts/` — `manifest.json` from the latest production run
- `s3://your-org-dbt-artifacts/docs/` — the generated dbt docs static site

### IAM Permissions for GitHub Actions

The `TerraformGitHubActionsRole` used by your CI/CD already has S3 read/write permissions from the data lake buckets. Add permissions for the new artifacts bucket:

```hcl
# In the IAM policy for TerraformGitHubActionsRole (in aws/2-import-iam-roles.md)
statement {
  sid    = "DbtArtifactsAccess"
  effect = "Allow"
  actions = [
    "s3:GetObject",
    "s3:PutObject",
    "s3:ListBucket",
    "s3:DeleteObject"
  ]
  resources = [
    module.bucket_dbt_artifacts.bucket_arn,
    "${module.bucket_dbt_artifacts.bucket_arn}/*"
  ]
}
```

### Snowflake Credentials Secret

Confirm the `dbt/snowflake-credentials` secret exists in AWS Secrets Manager (created in [Snowflake Infrastructure](4-snowflake-infrastructure.md)).

## Pull Request CI Workflow

Create `.github/workflows/ci.yml` in the `dbt-transform` repository:

```yaml
name: dbt CI

on:
  pull_request:
    branches:
      - main

permissions:
  id-token: write   # required for OIDC authentication to AWS
  contents: read
  pull-requests: write

jobs:
  dbt-ci:
    name: Slim CI
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::YOUR_ACCOUNT_ID:role/TerraformGitHubActionsRole
          aws-region: eu-west-1

      - name: Install uv
        uses: astral-sh/setup-uv@v4
        with:
          enable-cache: true

      - name: Set up Python
        run: uv python install 3.11

      - name: Install dependencies
        run: uv sync

      - name: Add venv to PATH
        run: echo "$GITHUB_WORKSPACE/.venv/bin" >> $GITHUB_PATH

      - name: Install dbt packages
        run: dbt deps

      - name: Get Snowflake credentials from Secrets Manager
        uses: aws-actions/aws-secretsmanager-get-secrets@v2
        with:
          secret-ids: |
            SNOWFLAKE, dbt/snowflake-credentials
          parse-json-secrets: true

      - name: Write Snowflake private key
        run: |
          echo "$SNOWFLAKE_PRIVATE_KEY" > /tmp/snowflake_key.pem
          chmod 600 /tmp/snowflake_key.pem

      - name: Write profiles.yml
        run: |
          mkdir -p ~/.dbt
          cat > ~/.dbt/profiles.yml << EOF
          dbt_transform:
            target: ci
            outputs:
              ci:
                type: snowflake
                account: "${SNOWFLAKE_ACCOUNT}"
                user: "${SNOWFLAKE_USER}"
                private_key_path: "/tmp/snowflake_key.pem"
                role: "${SNOWFLAKE_ROLE}"
                warehouse: "${SNOWFLAKE_WAREHOUSE}"
                database: ANALYTICS_DEV
                schema: "CI_PR_${{ github.event.number }}"
                threads: 4
          EOF

      - name: Fetch production manifest (for deferral)
        run: |
          mkdir -p .artifacts
          aws s3 cp s3://your-org-dbt-artifacts/artifacts/manifest.json .artifacts/manifest.json \
            || echo "No production manifest found — running without deferral"

      - name: Check source freshness
        run: dbt source freshness --target ci
        continue-on-error: true  # warn but don't fail CI on stale sources

      - name: Run slim CI (changed models + downstream)
        run: |
          if [ -f .artifacts/manifest.json ]; then
            dbt build \
              --target ci \
              --select state:modified+ \
              --defer \
              --state .artifacts/ \
              --fail-fast
          else
            # First run — no manifest yet, build everything
            dbt build --target ci --fail-fast
          fi

      - name: Run dbt_project_evaluator
        run: dbt build --target ci --select package:dbt_project_evaluator

      - name: Clean up CI schema
        if: always()
        run: |
          dbt run-operation drop_schema \
            --args "{schema_name: CI_PR_${{ github.event.number }}, database_name: ANALYTICS_DEV}" \
            --target ci
        continue-on-error: true
```

!!! note "CI schema isolation"
    Each PR gets its own schema (`CI_PR_123`) in `ANALYTICS_DEV`. This keeps PR runs isolated from each other and from developer work. The schema is dropped at the end of the workflow whether it passes or fails.

### The `drop_schema` Macro

Create `macros/drop_schema.sql` to support CI cleanup:

```sql
{% macro drop_schema(schema_name, database_name) %}
    {% set drop_query %}
        drop schema if exists {{ database_name }}.{{ schema_name }} cascade
    {% endset %}
    {% do run_query(drop_query) %}
    {{ log("Dropped schema: " ~ database_name ~ "." ~ schema_name, info=True) }}
{% endmacro %}
```

## Production Deployment Workflow

Create `.github/workflows/deploy.yml`:

```yaml
name: dbt Production Deploy

on:
  push:
    branches:
      - main

permissions:
  id-token: write
  contents: read

jobs:
  dbt-deploy:
    name: Production Run
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::YOUR_ACCOUNT_ID:role/TerraformGitHubActionsRole
          aws-region: eu-west-1

      - name: Install uv
        uses: astral-sh/setup-uv@v4
        with:
          enable-cache: true

      - name: Set up Python
        run: uv python install 3.11

      - name: Install dependencies
        run: uv sync

      - name: Add venv to PATH
        run: echo "$GITHUB_WORKSPACE/.venv/bin" >> $GITHUB_PATH

      - name: Install dbt packages
        run: dbt deps

      - name: Get Snowflake credentials from Secrets Manager
        uses: aws-actions/aws-secretsmanager-get-secrets@v2
        with:
          secret-ids: |
            SNOWFLAKE, dbt/snowflake-credentials
          parse-json-secrets: true

      - name: Write Snowflake private key
        run: |
          echo "$SNOWFLAKE_PRIVATE_KEY" > /tmp/snowflake_key.pem
          chmod 600 /tmp/snowflake_key.pem

      - name: Write profiles.yml
        run: |
          mkdir -p ~/.dbt
          cat > ~/.dbt/profiles.yml << EOF
          dbt_transform:
            target: prod
            outputs:
              prod:
                type: snowflake
                account: "${SNOWFLAKE_ACCOUNT}"
                user: "${SNOWFLAKE_USER}"
                private_key_path: "/tmp/snowflake_key.pem"
                role: "${SNOWFLAKE_ROLE}"
                warehouse: "${SNOWFLAKE_WAREHOUSE}"
                database: ANALYTICS
                schema: ANALYTICS
                threads: 8
          EOF

      - name: Run dbt build (production)
        run: dbt build --target prod --fail-fast

      - name: Upload manifest to S3 (for future deferral)
        if: success()
        run: |
          aws s3 cp target/manifest.json \
            s3://your-org-dbt-artifacts/artifacts/manifest.json

      - name: Generate dbt docs
        if: success()
        run: dbt docs generate --target prod

      - name: Deploy docs to S3
        if: success()
        run: |
          aws s3 sync ./target/ \
            s3://your-org-dbt-artifacts/docs/ \
            --delete \
            --exclude "*.json" \
            --include "index.html" \
            --include "catalog.json" \
            --include "manifest.json" \
            --include "run_results.json"

      - name: Invalidate CloudFront cache
        if: success()
        run: |
          aws cloudfront create-invalidation \
            --distribution-id YOUR_DISTRIBUTION_ID \
            --paths "/*"
```

## CloudFront Docs Site

CloudFront is AWS's Content Delivery Network (CDN). It serves the dbt docs site over HTTPS with caching and global distribution, making the docs fast and secure to access. Without CloudFront, you'd need to make the S3 bucket publicly accessible directly, which is less secure and slower for users geographically distant from your bucket's region.

The CloudFront distribution sits in front of the S3 bucket and provides:

- **HTTPS termination** — encrypted connections without managing certificates on S3
- **Global caching** — docs are cached at edge locations worldwide for fast access
- **Origin Access Control** — S3 bucket remains private, only CloudFront can access it
- **Custom domain support** — add your own domain (e.g. `docs.yourcompany.com`) if needed

Add the CloudFront distribution and S3 static site hosting to Terraform:

```hcl
# terraform/aws/cloudfront.tf

resource "aws_cloudfront_origin_access_control" "dbt_docs" {
  name                              = "dbt-docs-oac"
  description                       = "OAC for dbt docs S3 bucket"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

resource "aws_cloudfront_distribution" "dbt_docs" {
  enabled             = true
  default_root_object = "index.html"
  comment             = "dbt documentation site"

  origin {
    domain_name              = module.bucket_dbt_artifacts.bucket_regional_domain_name
    origin_id                = "dbt-docs-s3"
    origin_path              = "/docs"

    origin_access_control_id = aws_cloudfront_origin_access_control.dbt_docs.id
  }

  default_cache_behavior {
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "dbt-docs-s3"
    viewer_protocol_policy = "redirect-to-https"

    forwarded_values {
      query_string = false
      cookies { forward = "none" }
    }

    min_ttl     = 0
    default_ttl = 300
    max_ttl     = 3600
  }

  restrictions {
    geo_restriction { restriction_type = "none" }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }

  tags = {
    Project   = "data-platform"
    ManagedBy = "terraform"
  }
}

output "dbt_docs_url" {
  description = "URL for the dbt documentation site"
  value       = "https://${aws_cloudfront_distribution.dbt_docs.domain_name}"
}
```

After `terraform apply`, note the CloudFront distribution ID and update `YOUR_DISTRIBUTION_ID` in `deploy.yml`.

!!! tip "Access control for the docs site"
    The configuration above makes the docs site publicly accessible over HTTPS. For an internal-only site, add a CloudFront function or Lambda@Edge to check a shared secret header, or restrict access to your office IP range. Alternatively, host behind a Cognito-authenticated CloudFront distribution.

## Local Development with Deferral

Developers can use the production `manifest.json` locally to defer unchanged models:

```sh
# Fetch the production manifest
mkdir -p .artifacts
aws s3 cp s3://your-org-dbt-artifacts/artifacts/manifest.json .artifacts/manifest.json \
    --profile data-engineer

# Run only your changed models, deferring everything else to production
dbt build \
    --select state:modified+ \
    --defer \
    --state .artifacts/
```

This means a developer working on `fct_contacts` will:

- Build `stg_airbyte__contacts`, `int_contacts__enriched`, and `fct_contacts` in `ANALYTICS_DEV`
- Reference production versions of any other upstream models they haven't touched
- Run tests only on the models they built

The result: a fast, focused development cycle without rebuilding the entire project.

### Add just Commands for Deferral

Create a `justfile` in the repository root to streamline deferral workflows:

```just
# Fetch the production manifest from S3 for deferral
fetch-manifest:
    @echo "Fetching production manifest..."
    @mkdir -p .artifacts
    @aws s3 cp s3://your-org-dbt-artifacts/artifacts/manifest.json .artifacts/manifest.json \
        --profile data-engineer 2>/dev/null \
        || echo "Warning: No production manifest found — running without deferral"

# Run dbt with deferral to production (only changed models)
dev *ARGS: fetch-manifest
    @if [ -f .artifacts/manifest.json ]; then \
        dbt build --select state:modified+ --defer --state .artifacts/ {{ ARGS }}; \
    else \
        dbt build {{ ARGS }}; \
    fi

# Run a specific selection with deferral
dev-select SELECTION: fetch-manifest
    @if [ -f .artifacts/manifest.json ]; then \
        dbt build --select {{ SELECTION }} --defer --state .artifacts/; \
    else \
        dbt build --select {{ SELECTION }}; \
    fi

# Clean local artifacts
clean-artifacts:
    rm -rf .artifacts
```

Usage:

```sh
# Run changed models with deferral
just dev

# Run specific model with deferral
just dev-select fct_exchange_rates

# Run changed models with additional flags
just dev --fail-fast

# Clean downloaded artifacts
just clean-artifacts
```

## Summary

You've configured dbt Core CI/CD:

- [x] Slim CI on pull requests — runs only changed models with deferral to production
- [x] Production deployment on merge to main — full build, uploads manifest to S3
- [x] dbt docs generated and deployed to S3 + CloudFront after each production run
- [x] Local deferral workflow with helper script
- [x] CI schema isolation — each PR builds in a separate `ANALYTICS_DEV.CI_PR_*` schema

## What's Next

If you chose dbt Cloud, set up environments and jobs in the managed platform.

Continue to [dbt Cloud Setup](10-dbt-cloud-setup.md) →
