# Data Cataloging with OpenMetadata

On this page, you will:

- [x] Understand what a data catalog is and why it matters
- [x] Deploy OpenMetadata to AWS ECS using Terraform
- [x] Connect OpenMetadata to Snowflake for metadata extraction
- [x] Connect OpenMetadata to dbt for lineage and documentation
- [x] Connect OpenMetadata to Prefect for pipeline metadata
- [x] Search and discover data assets across your entire stack

## Overview

A **data catalog** is a centralized metadata repository that helps users discover, understand, and trust data. It answers questions like:

- "What tables exist in our warehouse?"
- "Where does this column come from?" (lineage)
- "Who owns this dataset?"
- "What tests validate this data?"
- "Which dashboards use this table?"

**OpenMetadata** is an open source data catalog that integrates with dbt, Snowflake, Prefect, and BI tools to provide a unified view of your data platform.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    OPENMETADATA ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Data Sources              OpenMetadata              Users              │
│  ────────────              ────────────              ─────              │
│                                                                         │
│  ┌──────────────┐          ┌──────────────┐         ┌──────────────┐  │
│  │ Snowflake    │─────────▶│ Metadata DB  │────────▶│ Data         │  │
│  │ • Tables     │          │ (PostgreSQL) │         │ Analysts     │  │
│  │ • Views      │          │              │         │              │  │
│  │ • Usage      │          │ ┌──────────┐ │         │ Search:      │  │
│  └──────────────┘          │ │ Lineage  │ │         │ "customer"   │  │
│                            │ │ Docs     │ │         │              │  │
│  ┌──────────────┐          │ │ Quality  │ │         │ Results:     │  │
│  │ dbt          │─────────▶│ │ Owners   │ │         │ • dim_       │  │
│  │ • Models     │          │ └──────────┘ │         │   customers  │  │
│  │ • Tests      │          │              │         │ • fct_orders │  │
│  │ • Lineage    │          └──────────────┘         └──────────────┘  │
│  └──────────────┘                  │                                   │
│                                    │                                   │
│  ┌──────────────┐          ┌──────────────┐                           │
│  │ Prefect      │─────────▶│ OpenMetadata │                           │
│  │ • Flows      │          │ UI           │                           │
│  │ • Tasks      │          │ • Search     │                           │
│  │ • Runs       │          │ • Lineage    │                           │
│  └──────────────┘          │ • Quality    │                           │
│                            └──────────────┘                           │
│                                                                         │
│  Ingestion runs daily: Extract metadata → Store in OpenMetadata        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## What OpenMetadata Provides

### 1. Data Discovery

**Search across all data assets:**
- Tables, views, dashboards, pipelines
- Full-text search on names, descriptions, columns
- Filter by tags, owners, tiers, domains

**Example:** Search "revenue" → find `fct_revenue`, `rpt_revenue_summary`, "Revenue Dashboard" (Lightdash), dbt model `int_revenue__daily`

### 2. Lineage

**Column-level lineage:**
- Trace data from source → staging → marts → dashboards
- Understand transformations at each step
- Impact analysis: "What breaks if I change this column?"

**Example:** `dim_customers.email` ← `stg_hubspot__contacts.email_address` ← `raw_hubspot.contacts.properties_email`

### 3. Data Quality

**Test results from dbt:**
- All dbt tests (unique, not_null, custom)
- Test pass/fail history
- Elementary anomaly detection results

**Example:** View that `customer_id` has a "unique" test with 100% pass rate over 30 days.

### 4. Ownership and Documentation

**Data governance:**
- Assign owners to tables, dashboards, pipelines
- Tag datasets by tier (Tier 1 = critical, Tier 2 = important, Tier 3 = nice-to-have)
- Add descriptions and documentation

**Example:** `fct_orders` owned by "Data Engineering Team", Tier 1, description links to Confluence page.

### 5. Usage Analytics

**Who queries what:**
- Top queries by user
- Most-used tables
- Unused tables (candidates for deprecation)

**Example:** `dim_customers` queried 1,250 times last week by 15 users; `dim_products_v1` not queried in 90 days (safe to drop).

## OpenMetadata vs Alternatives

| Feature | OpenMetadata | Atlan | Amundsen | DataHub | Marquez |
|---------|-------------|-------|----------|---------|---------|
| **Cost** | Free (OSS) | $$$ Enterprise | Free (OSS) | Free (OSS) | Free (OSS) |
| **dbt integration** | ✅ Excellent | ✅ Excellent | ⚠️ Basic | ✅ Good | ❌ None |
| **Snowflake integration** | ✅ Native | ✅ Native | ✅ Via plugin | ✅ Native | ⚠️ Basic |
| **Column lineage** | ✅ Yes | ✅ Yes | ❌ No | ✅ Yes | ⚠️ Basic |
| **UI polish** | ⚠️ Good | ✅ Excellent | ⚠️ Basic | ⚠️ Good | ⚠️ Basic |
| **Active development** | ✅ Very active | ✅ Commercial | ⚠️ Slow | ✅ Active | ⚠️ Moderate |
| **Self-hosting ease** | ✅ Docker/K8s | ❌ Cloud only | ⚠️ Complex | ⚠️ Complex | ✅ Simple |

**Why OpenMetadata for this project:**
- Free and open source
- Excellent dbt integration (reads manifest.json natively)
- Active development and community
- Relatively easy to self-host
- Good balance of features vs complexity

## Prerequisites

Before deploying OpenMetadata:

- [x] VPC with public and private subnets (from [VPC Setup](../aws/2-vpc-networking.md))
- [x] dbt project with models and tests (from [Data Transformation](../data-transformation/index.md))
- [x] Snowflake with data (from [Data Warehouse](../data-warehouse/index.md))
- [x] Prefect running pipelines (from [Orchestration](../orchestration/index.md))

## Step 1: Deploy OpenMetadata to ECS

OpenMetadata consists of three services:
1. **OpenMetadata Server** (backend API)
2. **OpenMetadata UI** (web interface)
3. **OpenMetadata Ingestion** (metadata extraction workers)

All three run as Docker containers on ECS.

### Create RDS PostgreSQL for Metadata

OpenMetadata stores metadata in PostgreSQL.

```hcl
# terraform/aws/openmetadata-rds.tf
resource "aws_db_subnet_group" "openmetadata" {
  name       = "openmetadata-db-subnet-group"
  subnet_ids = module.vpc.private_subnet_ids

  tags = {
    Name      = "OpenMetadata DB Subnet Group"
    ManagedBy = "terraform"
  }
}

resource "aws_security_group" "openmetadata_rds" {
  name        = "openmetadata-rds-sg"
  description = "Security group for OpenMetadata RDS"
  vpc_id      = module.vpc.vpc_id

  ingress {
    description     = "PostgreSQL from ECS"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.openmetadata_ecs.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name      = "OpenMetadata RDS SG"
    ManagedBy = "terraform"
  }
}

resource "random_password" "openmetadata_db_password" {
  length  = 32
  special = true
}

resource "aws_secretsmanager_secret" "openmetadata_db_password" {
  name = "openmetadata/database-password"
}

resource "aws_secretsmanager_secret_version" "openmetadata_db_password" {
  secret_id     = aws_secretsmanager_secret.openmetadata_db_password.id
  secret_string = random_password.openmetadata_db_password.result
}

resource "aws_db_instance" "openmetadata" {
  identifier     = "openmetadata-db"
  engine         = "postgres"
  engine_version = "15.4"
  instance_class = "db.t3.small"  # Larger than Lightdash (more metadata)

  allocated_storage     = 50
  max_allocated_storage = 200
  storage_type          = "gp3"
  storage_encrypted     = true

  db_name  = "openmetadata"
  username = "openmetadata"
  password = random_password.openmetadata_db_password.result

  db_subnet_group_name   = aws_db_subnet_group.openmetadata.name
  vpc_security_group_ids = [aws_security_group.openmetadata_rds.id]

  backup_retention_period = 7
  backup_window           = "03:00-04:00"
  maintenance_window      = "sun:04:00-sun:05:00"

  skip_final_snapshot       = false
  final_snapshot_identifier = "openmetadata-final-${formatdate("YYYY-MM-DD-hhmm", timestamp())}"

  tags = {
    Name      = "OpenMetadata PostgreSQL"
    ManagedBy = "terraform"
  }
}
```

### Deploy OpenMetadata Server (ECS)

```hcl
# terraform/aws/openmetadata-ecs.tf
resource "aws_ecs_cluster" "openmetadata" {
  name = "openmetadata-cluster"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

resource "aws_ecs_task_definition" "openmetadata_server" {
  family                   = "openmetadata-server"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "2048"
  memory                   = "4096"

  execution_role_arn = aws_iam_role.ecs_task_execution_role.arn
  task_role_arn      = aws_iam_role.openmetadata_task_role.arn

  container_definitions = jsonencode([{
    name  = "openmetadata-server"
    image = "openmetadata/server:1.3.0"

    portMappings = [{
      containerPort = 8585
      protocol      = "tcp"
    }]

    environment = [
      { name = "DB_DRIVER_CLASS", value = "org.postgresql.Driver" },
      { name = "DB_SCHEME", value = "postgresql" },
      { name = "DB_HOST", value = aws_db_instance.openmetadata.address },
      { name = "DB_PORT", value = "5432" },
      { name = "DB_DATABASE", value = "openmetadata" },
      { name = "DB_USER", value = "openmetadata" },
      { name = "AUTHENTICATION_PROVIDER", value = "basic" },
      { name = "AUTHORIZER_CLASS_NAME", value = "org.openmetadata.service.security.DefaultAuthorizer" }
    ]

    secrets = [
      {
        name      = "DB_USER_PASSWORD"
        valueFrom = aws_secretsmanager_secret.openmetadata_db_password.arn
      }
    ]

    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"         = aws_cloudwatch_log_group.openmetadata.name
        "awslogs-region"        = data.aws_region.current.name
        "awslogs-stream-prefix" = "server"
      }
    }

    healthCheck = {
      command     = ["CMD-SHELL", "wget --quiet --tries=1 --spider http://localhost:8585/api/v1/health || exit 1"]
      interval    = 30
      timeout     = 5
      retries     = 3
      startPeriod = 120
    }
  }])
}

resource "aws_ecs_service" "openmetadata_server" {
  name            = "openmetadata-server"
  cluster         = aws_ecs_cluster.openmetadata.id
  task_definition = aws_ecs_task_definition.openmetadata_server.arn
  desired_count   = 1
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = module.vpc.private_subnet_ids
    security_groups  = [aws_security_group.openmetadata_ecs.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.openmetadata.arn
    container_name   = "openmetadata-server"
    container_port   = 8585
  }
}
```

### Create ALB for HTTPS Access

```hcl
# terraform/aws/openmetadata-alb.tf
resource "aws_lb" "openmetadata" {
  name               = "openmetadata-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.openmetadata_alb.id]
  subnets            = module.vpc.public_subnet_ids

  tags = {
    Name      = "OpenMetadata ALB"
    ManagedBy = "terraform"
  }
}

resource "aws_lb_target_group" "openmetadata" {
  name        = "openmetadata-tg"
  port        = 8585
  protocol    = "HTTP"
  vpc_id      = module.vpc.vpc_id
  target_type = "ip"

  health_check {
    path                = "/api/v1/health"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 3
    matcher             = "200"
  }
}

resource "aws_lb_listener" "openmetadata_https" {
  load_balancer_arn = aws_lb.openmetadata.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.openmetadata.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.openmetadata.arn
  }
}

output "openmetadata_url" {
  value = "https://${aws_lb.openmetadata.dns_name}"
}
```

### Apply Terraform

```sh
cd terraform/aws
terraform plan
terraform apply
```

Once provisioning completes, OpenMetadata is accessible at the ALB URL.

Navigate to `https://openmetadata-alb-xxxxx.eu-west-2.elb.amazonaws.com` and log in with default credentials:
- Username: `admin`
- Password: `admin`

**Change the password immediately** in Settings → Users.

## Step 2: Create SVC_OPENMETADATA Service Account

OpenMetadata needs read-only access to Snowflake's `ACCOUNT_USAGE` schema for metadata extraction.

```hcl
# terraform/snowflake/service-accounts/openmetadata.tf
module "service_user_openmetadata" {
  source = "../modules/snowflake_service_user"

  username    = "SVC_OPENMETADATA"
  comment     = "Service account for OpenMetadata data catalog"
  email       = "data-platform@yourcompany.com"
  rsa_public_key = var.svc_openmetadata_public_key

  user_create_dedicated_role = true
  dedicated_role_grants = [
    "ANALYTICS_REPORTER"  # Read REPORTING schema for table metadata
  ]

  default_warehouse = "REPORTING"
  default_namespace = "ANALYTICS.REPORTING"
  default_role      = "SVC_OPENMETADATA"

  tags = {
    Service   = "openmetadata"
    ManagedBy = "terraform"
  }
}

# Grant access to ACCOUNT_USAGE for query history and metadata
resource "snowflake_grant_privileges_to_account_role" "openmetadata_account_usage" {
  account_role_name = module.service_user_openmetadata.dedicated_role_name
  privileges        = ["IMPORTED PRIVILEGES"]

  on_account_object {
    object_type = "DATABASE"
    object_name = "SNOWFLAKE"
  }
}
```

This grants `SVC_OPENMETADATA`:
- Read access to `ANALYTICS.REPORTING` (table metadata, column names, types)
- Read access to `SNOWFLAKE.ACCOUNT_USAGE` (query history, table usage statistics)

## Step 3: Connect OpenMetadata to Snowflake

### Add Snowflake Service

1. Log in to OpenMetadata UI
2. Navigate to **Settings** → **Services** → **Databases**
3. Click **Add Database Service**
4. Select **Snowflake**

Configure:

| Field | Value |
|-------|-------|
| **Name** | `snowflake-analytics` |
| **Account** | `your-account.snowflakecomputing.com` |
| **Username** | `SVC_OPENMETADATA` |
| **Password** | (leave blank, using key-pair) |
| **Private Key** | Paste contents of `svc_openmetadata_rsa_key.pem` |
| **Warehouse** | `REPORTING` |
| **Database** | `ANALYTICS` |
| **Schema** | `REPORTING` (or leave blank to ingest all schemas) |

Click **Test Connection** → Should succeed.

Click **Save**.

### Configure Metadata Ingestion

1. Click on the `snowflake-analytics` service
2. Navigate to **Ingestion** tab
3. Click **Add Ingestion** → **Metadata**

Configure:

| Field | Value |
|-------|-------|
| **Name** | `snowflake-metadata-daily` |
| **Database Filter** | `ANALYTICS` |
| **Schema Filter** | `REPORTING, MARTS` (comma-separated) |
| **Include Views** | ✅ Yes |
| **Include Tables** | ✅ Yes |
| **Mark Deleted Tables** | ✅ Yes (soft delete in OpenMetadata if table is dropped) |

**Schedule:**
- Frequency: Daily
- Time: 02:00 UTC (after dbt runs complete)

Click **Deploy**.

OpenMetadata will now extract metadata from Snowflake daily at 02:00 UTC.

### Verify Snowflake Ingestion

Navigate to **Explore** → **Tables**.

You should see:
- `analytics.reporting.fct_exchange_rates`
- `analytics.reporting.dim_products`
- `analytics.marts.fct_exchange_rates`
- etc.

Click on a table to see:
- **Schema** (column names, types, descriptions)
- **Sample Data** (first 10 rows)
- **Queries** (who queried this table, when)
- **Lineage** (empty for now — added in next step with dbt)

## Step 4: Connect OpenMetadata to dbt

OpenMetadata reads dbt's `manifest.json`, `catalog.json`, and `run_results.json` to extract:
- Model descriptions
- Column descriptions
- dbt tests
- Lineage (which models depend on which)

### Option A: GitHub Integration (Recommended)

OpenMetadata can clone your dbt repository and read artifacts.

1. Navigate to **Settings** → **Services** → **Pipelines**
2. Click **Add Pipeline Service**
3. Select **dbt**

Configure:

| Field | Value |
|-------|-------|
| **Name** | `dbt-transform` |
| **dbt Config Source** | GitHub |
| **Repository URL** | `https://github.com/your-org/dbt-transform` |
| **Branch** | `main` |
| **dbt Project Path** | `.` (root of repo) |
| **GitHub Token** | Personal access token with `repo` scope |

**dbt Catalog Path:** `target/catalog.json`
**dbt Manifest Path:** `target/manifest.json`
**dbt Run Results Path:** `target/run_results.json`

Click **Test Connection** → **Save**.

### Configure dbt Metadata Ingestion

1. Click on `dbt-transform` service
2. Navigate to **Ingestion** tab
3. Click **Add Ingestion** → **Metadata**

Configure:

| Field | Value |
|-------|-------|
| **Name** | `dbt-metadata-daily` |
| **dbt Update Descriptions** | ✅ Yes (overwrite Snowflake descriptions with dbt descriptions) |
| **Include dbt Tags** | ✅ Yes |

**Schedule:**
- Frequency: Daily
- Time: 09:00 UTC (after dbt runs and uploads artifacts to GitHub)

Click **Deploy**.

### Option B: Upload Artifacts Manually

If you don't want GitHub integration:

1. After `dbt run`, upload `target/manifest.json`, `target/catalog.json`, `target/run_results.json` to S3
2. Configure OpenMetadata to read from S3 (HTTP URL)

### Verify dbt Ingestion

Navigate to **Explore** → **Tables** → `fct_exchange_rates`.

You should now see:
- **Description** from dbt YAML
- **Column descriptions** from dbt YAML
- **Tests** (unique, not_null, dbt_expectations tests)
- **Lineage** (upstream: `stg_dlt__exchange_rates`, downstream: dashboards)

Click **Lineage** tab to see the dbt DAG.

## Step 5: Connect OpenMetadata to Prefect (Optional)

Extract pipeline metadata from Prefect (flows, tasks, runs).

### Add Prefect Service

1. Navigate to **Settings** → **Services** → **Pipelines**
2. Click **Add Pipeline Service**
3. Select **Prefect**

Configure:

| Field | Value |
|-------|-------|
| **Name** | `prefect-orchestration` |
| **Host Port** | `https://api.prefect.cloud` (or self-hosted URL) |
| **API Key** | Prefect API key (create in Prefect Cloud → API Keys) |

Click **Test Connection** → **Save**.

### Configure Prefect Metadata Ingestion

1. Click on `prefect-orchestration` service
2. **Ingestion** → **Add Ingestion** → **Metadata**

Schedule: Daily at 10:00 UTC.

Now you can see Prefect flows in OpenMetadata and their relationship to dbt models.

## Step 6: Search and Discovery

### Global Search

Use the search bar at the top of OpenMetadata UI:

**Search:** `customer`

**Results:**
- `dim_customers` (table)
- `stg_hubspot__contacts` (table)
- `customers` column in `fct_orders` (column)
- "Customer Segmentation" (dashboard, if connected)
- `ingest_hubspot_contacts` (Prefect flow)

### Advanced Filters

Filter search results by:
- **Service** (Snowflake, dbt, Prefect)
- **Tier** (Tier 1 = critical, Tier 2 = important, Tier 3 = experimental)
- **Owner** (assign owners to datasets)
- **Tags** (PII, GDPR, revenue, marketing)

### Assign Ownership

1. Navigate to a table (e.g., `fct_exchange_rates`)
2. Click **Edit** → **Ownership**
3. Select **Owner**: "Data Engineering Team" (create teams in Settings → Teams)
4. Select **Tier**: Tier 1 (critical)
5. Add tags: `revenue`, `finance`, `daily`

Now analysts can filter: "Show me all Tier 1 tables owned by Data Engineering."

## Step 7: Automate Ingestion via CLI

Run OpenMetadata ingestion from the command line (for CI/CD integration).

### Install OpenMetadata CLI

```sh
pip install openmetadata-ingestion[snowflake,dbt]
```

### Create Ingestion Config

```yaml
# ingestion/snowflake-config.yaml
source:
  type: snowflake
  serviceName: snowflake-analytics
  serviceConnection:
    config:
      type: Snowflake
      username: SVC_OPENMETADATA
      privateKey: /path/to/svc_openmetadata_rsa_key.pem
      snowflakeAccount: your-account.snowflakecomputing.com
      warehouse: REPORTING
      database: ANALYTICS
  sourceConfig:
    config:
      type: DatabaseMetadata
      schemaFilterPattern:
        includes:
          - REPORTING
          - MARTS

sink:
  type: metadata-rest
  config:
    workflowConfig:
      openMetadataServerConfig:
        hostPort: https://your-openmetadata-url.com
        authProvider: openmetadata
        securityConfig:
          jwtToken: "YOUR_JWT_TOKEN"

workflowConfig:
  loggerLevel: INFO
  openMetadataServerConfig:
    hostPort: https://your-openmetadata-url.com
```

### Run Ingestion

```sh
metadata ingest -c ingestion/snowflake-config.yaml
```

This can be added to Prefect as a task to run after dbt.

## Cost Summary

| Component | Monthly Cost |
|-----------|-------------|
| **RDS PostgreSQL (db.t3.small)** | ~$30 |
| **ECS Fargate (2 vCPU, 4GB RAM)** | ~$30 |
| **ALB** | ~$20 |
| **Total** | ~$80/month |

This provides a complete data catalog for your entire stack.

## Summary

You've deployed OpenMetadata:

- [x] **OpenMetadata deployed** to ECS with PostgreSQL backend
- [x] **Snowflake connected** — metadata extraction from `ANALYTICS` database
- [x] **dbt connected** — lineage, tests, and documentation imported
- [x] **Prefect connected** (optional) — pipeline metadata
- [x] **Search and discovery** — find tables, columns, dashboards across all systems
- [x] **Ownership and tags** — governance with teams, tiers, and tags

OpenMetadata is now the central catalog for your data platform, unifying metadata from Snowflake, dbt, and Prefect.

## What's Next

Use OpenMetadata's lineage features to trace data flow and perform impact analysis.

Continue to [Data Lineage](5-data-lineage.md) →
