# Self-Hosted Lightdash

On this page, you will:

- [x] Understand the cost savings of self-hosting Lightdash (~$50/month vs $2400/month Cloud)
- [x] Deploy Lightdash to AWS ECS Fargate using Terraform
- [x] Configure RDS PostgreSQL for Lightdash metadata storage
- [x] Set up an Application Load Balancer with HTTPS
- [x] Connect self-hosted Lightdash to GitHub and Snowflake
- [x] Manage updates, backups, and monitoring

## Overview

Self-hosted Lightdash runs on your own infrastructure (AWS, GCP, Azure, or on-premises). It provides the same features as Lightdash Cloud but requires you to manage deployment, updates, and backups.

**Why self-host?**
- **Cost savings**: ~$50/month (infrastructure) vs $2400/month (Cloud)
- **Full control**: Customise deployment, networking, and security
- **Data residency**: Keep all metadata in your own AWS account
- **Infrastructure as code**: Terraform-managed, version-controlled

**Trade-offs:**
- Requires infrastructure knowledge (ECS, RDS, ALB)
- You manage updates and security patches
- No vendor support (community support only)
- 6-10 hours/month maintenance overhead

This page covers deploying Lightdash to **AWS ECS Fargate** with **RDS PostgreSQL**. This is the recommended self-hosted setup for most teams.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  SELF-HOSTED LIGHTDASH ARCHITECTURE                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  AWS Infrastructure                                                     │
│  ──────────────────                                                     │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ Application Load Balancer (ALB)                                │    │
│  │ • HTTPS termination (ACM certificate)                          │    │
│  │ • Health checks                                                │    │
│  │ • Path-based routing                                           │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                             │                                           │
│                             ▼                                           │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ ECS Fargate Cluster                                            │    │
│  │ ┌──────────────────┐                                           │    │
│  │ │ Lightdash Server │   (2 vCPUs, 4GB RAM, auto-scaling 1-3)    │    │
│  │ └──────────────────┘                                           │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                             │                                           │
│                             ▼                                           │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ RDS PostgreSQL (db.t3.micro)                                   │    │
│  │ • Lightdash metadata (dashboards, users, queries)              │    │
│  │ • Automated backups (7-day retention)                          │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ Secrets Manager                                                │    │
│  │ • Lightdash secret key                                         │    │
│  │ • Snowflake credentials (SVC_LIGHTDASH)                        │    │
│  │ • GitHub token                                                 │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  External Dependencies                                                  │
│  ─────────────────────                                                  │
│  • GitHub: dbt-transform repository                                     │
│  • Snowflake: ANALYTICS.REPORTING schema                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Cost Comparison: Self-Hosted vs Cloud

| Component | Self-Hosted (AWS) | Lightdash Cloud |
|-----------|------------------|-----------------|
| **Application hosting** | ECS Fargate: ~$30/month | Included |
| **Database** | RDS db.t3.micro: ~$15/month | Included |
| **Load balancer** | ALB: ~$20/month | Included |
| **Total infrastructure** | ~$65/month | $0 |
| **Lightdash subscription** | $0 (open source) | $2400/month |
| **Snowflake compute** | ~$25/month | ~$25/month |
| **Engineering time** | 6-10 hours/month (setup + maintenance) | 1-2 hours/month |
| **Total monthly cost** | ~$90 + eng time | $2425 |

**Break-even:** Self-hosting saves ~$2335/month if you have engineering capacity for maintenance.

## Prerequisites

Before deploying, ensure you have:

- [x] Completed [Snowflake Infrastructure](5-snowflake-infrastructure.md) — `SVC_LIGHTDASH` service account created
- [x] Terraform configured for AWS (from [AWS Infrastructure](../../getting-started/terraform-setup/aws/index.md))
- [x] VPC with public and private subnets (from [VPC Setup](../aws/2-vpc-networking.md))
- [x] Domain name (optional, for custom HTTPS) or use ALB default DNS

!!! tip "VPC Requirements"
    Lightdash requires:
    - **Public subnets** for the ALB (internet-facing)
    - **Private subnets** for ECS tasks and RDS (no direct internet access)
    - **NAT Gateway** for ECS tasks to pull Docker images and access GitHub/Snowflake

    If you haven't set up a VPC yet, use the default VPC for testing (not recommended for production).

## Step 1: Create RDS PostgreSQL Database

Lightdash stores metadata (dashboards, users, queries, cached results) in PostgreSQL.

### Terraform Module for RDS

Create a Terraform module for the RDS instance:

```hcl
# terraform/aws/lightdash-rds.tf
resource "aws_db_subnet_group" "lightdash" {
  name       = "lightdash-db-subnet-group"
  subnet_ids = module.vpc.private_subnet_ids  # Reference your VPC module

  tags = {
    Name      = "Lightdash DB Subnet Group"
    ManagedBy = "terraform"
  }
}

resource "aws_security_group" "lightdash_rds" {
  name        = "lightdash-rds-sg"
  description = "Security group for Lightdash RDS instance"
  vpc_id      = module.vpc.vpc_id

  ingress {
    description     = "PostgreSQL from ECS tasks"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.lightdash_ecs.id]  # Only ECS can connect
  }

  egress {
    description = "Allow all outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name      = "Lightdash RDS Security Group"
    ManagedBy = "terraform"
  }
}

resource "random_password" "lightdash_db_password" {
  length  = 32
  special = true
}

resource "aws_secretsmanager_secret" "lightdash_db_password" {
  name        = "lightdash/database-password"
  description = "PostgreSQL password for Lightdash database"

  tags = {
    ManagedBy = "terraform"
  }
}

resource "aws_secretsmanager_secret_version" "lightdash_db_password" {
  secret_id     = aws_secretsmanager_secret.lightdash_db_password.id
  secret_string = random_password.lightdash_db_password.result
}

resource "aws_db_instance" "lightdash" {
  identifier     = "lightdash-db"
  engine         = "postgres"
  engine_version = "15.4"
  instance_class = "db.t3.micro"

  allocated_storage     = 20
  max_allocated_storage = 100  # Auto-scaling storage
  storage_type          = "gp3"
  storage_encrypted     = true

  db_name  = "lightdash"
  username = "lightdash"
  password = random_password.lightdash_db_password.result

  db_subnet_group_name   = aws_db_subnet_group.lightdash.name
  vpc_security_group_ids = [aws_security_group.lightdash_rds.id]

  backup_retention_period = 7
  backup_window           = "03:00-04:00"  # UTC
  maintenance_window      = "sun:04:00-sun:05:00"  # UTC

  skip_final_snapshot       = false
  final_snapshot_identifier = "lightdash-db-final-snapshot-${formatdate("YYYY-MM-DD-hhmm", timestamp())}"

  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]

  tags = {
    Name      = "Lightdash PostgreSQL Database"
    ManagedBy = "terraform"
  }
}

output "lightdash_db_endpoint" {
  description = "Lightdash RDS endpoint"
  value       = aws_db_instance.lightdash.endpoint
}
```

Apply Terraform:

```sh
cd terraform/aws
terraform plan
terraform apply
```

The database password is stored in AWS Secrets Manager at `lightdash/database-password`.

## Step 2: Create ECS Cluster and Task Definition

### Security Group for ECS Tasks

```hcl
# terraform/aws/lightdash-ecs-sg.tf
resource "aws_security_group" "lightdash_ecs" {
  name        = "lightdash-ecs-sg"
  description = "Security group for Lightdash ECS tasks"
  vpc_id      = module.vpc.vpc_id

  ingress {
    description     = "HTTP from ALB"
    from_port       = 8080  # Lightdash listens on 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.lightdash_alb.id]
  }

  egress {
    description = "Allow all outbound (GitHub, Snowflake, Docker Hub)"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name      = "Lightdash ECS Security Group"
    ManagedBy = "terraform"
  }
}
```

### ECS Cluster

```hcl
# terraform/aws/lightdash-ecs.tf
resource "aws_ecs_cluster" "lightdash" {
  name = "lightdash-cluster"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }

  tags = {
    Name      = "Lightdash ECS Cluster"
    ManagedBy = "terraform"
  }
}
```

### Task Definition

```hcl
# terraform/aws/lightdash-task-definition.tf
resource "aws_ecs_task_definition" "lightdash" {
  family                   = "lightdash"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "2048"  # 2 vCPUs
  memory                   = "4096"  # 4GB

  execution_role_arn = aws_iam_role.ecs_task_execution_role.arn
  task_role_arn      = aws_iam_role.lightdash_task_role.arn

  container_definitions = jsonencode([{
    name  = "lightdash"
    image = "lightdash/lightdash:latest"

    portMappings = [{
      containerPort = 8080
      protocol      = "tcp"
    }]

    environment = [
      { name = "PGHOST", value = aws_db_instance.lightdash.address },
      { name = "PGPORT", value = "5432" },
      { name = "PGDATABASE", value = "lightdash" },
      { name = "PGUSER", value = "lightdash" },
      { name = "LIGHTDASH_INSTALL_TYPE", value = "docker_image" },
      { name = "LIGHTDASH_LOG_LEVEL", value = "info" },
      { name = "SITE_URL", value = "https://${aws_lb.lightdash.dns_name}" },  # Or custom domain
    ]

    secrets = [
      {
        name      = "PGPASSWORD"
        valueFrom = aws_secretsmanager_secret.lightdash_db_password.arn
      },
      {
        name      = "LIGHTDASH_SECRET"
        valueFrom = aws_secretsmanager_secret.lightdash_secret_key.arn
      }
    ]

    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"         = aws_cloudwatch_log_group.lightdash.name
        "awslogs-region"        = data.aws_region.current.name
        "awslogs-stream-prefix" = "lightdash"
      }
    }

    healthCheck = {
      command     = ["CMD-SHELL", "wget --quiet --tries=1 --spider http://localhost:8080/api/v1/health || exit 1"]
      interval    = 30
      timeout     = 5
      retries     = 3
      startPeriod = 60
    }
  }])

  tags = {
    Name      = "Lightdash Task Definition"
    ManagedBy = "terraform"
  }
}
```

### Generate Lightdash Secret Key

Lightdash requires a secret key for session encryption:

```sh
# Generate a random 32-character secret
export LIGHTDASH_SECRET=$(openssl rand -hex 32)

# Store in AWS Secrets Manager
aws secretsmanager create-secret \
    --name "lightdash/secret-key" \
    --description "Secret key for Lightdash session encryption" \
    --secret-string "$LIGHTDASH_SECRET" \
    --profile infrastructure-admin
```

### IAM Roles

```hcl
# terraform/aws/lightdash-iam.tf

# ECS task execution role (pulls images, writes logs, reads secrets)
resource "aws_iam_role" "ecs_task_execution_role" {
  name = "LightdashECSTaskExecutionRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      }
    }]
  })

  tags = {
    Name      = "Lightdash ECS Task Execution Role"
    ManagedBy = "terraform"
  }
}

resource "aws_iam_role_policy_attachment" "ecs_task_execution_role_policy" {
  role       = aws_iam_role.ecs_task_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

resource "aws_iam_role_policy" "ecs_secrets_access" {
  name = "ECSSecretsAccess"
  role = aws_iam_role.ecs_task_execution_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "secretsmanager:GetSecretValue"
      ]
      Resource = [
        aws_secretsmanager_secret.lightdash_db_password.arn,
        aws_secretsmanager_secret.lightdash_secret_key.arn,
        "arn:aws:secretsmanager:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:secret:lightdash/snowflake-credentials*"
      ]
    }]
  })
}

# Task role (permissions for Lightdash application itself)
resource "aws_iam_role" "lightdash_task_role" {
  name = "LightdashTaskRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      }
    }]
  })

  tags = {
    Name      = "Lightdash Task Role"
    ManagedBy = "terraform"
  }
}

# Add policies as needed (e.g., S3 access for cached query results)
```

### CloudWatch Logs

```hcl
# terraform/aws/lightdash-cloudwatch.tf
resource "aws_cloudwatch_log_group" "lightdash" {
  name              = "/ecs/lightdash"
  retention_in_days = 7

  tags = {
    Name      = "Lightdash Logs"
    ManagedBy = "terraform"
  }
}
```

## Step 3: Create Application Load Balancer

```hcl
# terraform/aws/lightdash-alb.tf
resource "aws_security_group" "lightdash_alb" {
  name        = "lightdash-alb-sg"
  description = "Security group for Lightdash ALB"
  vpc_id      = module.vpc.vpc_id

  ingress {
    description = "HTTPS from internet"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTP from internet (redirect to HTTPS)"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "Allow all outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name      = "Lightdash ALB Security Group"
    ManagedBy = "terraform"
  }
}

resource "aws_lb" "lightdash" {
  name               = "lightdash-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.lightdash_alb.id]
  subnets            = module.vpc.public_subnet_ids

  enable_deletion_protection = false  # Set to true for production

  tags = {
    Name      = "Lightdash ALB"
    ManagedBy = "terraform"
  }
}

resource "aws_lb_target_group" "lightdash" {
  name        = "lightdash-tg"
  port        = 8080
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

  tags = {
    Name      = "Lightdash Target Group"
    ManagedBy = "terraform"
  }
}

resource "aws_lb_listener" "lightdash_http" {
  load_balancer_arn = aws_lb.lightdash.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

resource "aws_lb_listener" "lightdash_https" {
  load_balancer_arn = aws_lb.lightdash.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.lightdash.arn  # Create ACM certificate or use self-signed for testing

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.lightdash.arn
  }
}

output "lightdash_url" {
  description = "Lightdash application URL"
  value       = "https://${aws_lb.lightdash.dns_name}"
}
```

!!! tip "HTTPS Certificate"
    For production, request an ACM certificate for your custom domain (e.g., `lightdash.yourcompany.com`). For testing, you can use the ALB's default DNS name with a self-signed certificate (browsers will show a warning).

## Step 4: Create ECS Service

```hcl
# terraform/aws/lightdash-ecs-service.tf
resource "aws_ecs_service" "lightdash" {
  name            = "lightdash-service"
  cluster         = aws_ecs_cluster.lightdash.id
  task_definition = aws_ecs_task_definition.lightdash.arn
  desired_count   = 1
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = module.vpc.private_subnet_ids
    security_groups  = [aws_security_group.lightdash_ecs.id]
    assign_public_ip = false  # Use NAT Gateway for outbound
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.lightdash.arn
    container_name   = "lightdash"
    container_port   = 8080
  }

  depends_on = [aws_lb_listener.lightdash_https]

  tags = {
    Name      = "Lightdash ECS Service"
    ManagedBy = "terraform"
  }
}
```

## Step 5: Apply Terraform and Verify Deployment

```sh
cd terraform/aws
terraform plan
# Review: Creates RDS, ECS cluster, task definition, service, ALB, security groups

terraform apply
```

After Terraform completes (RDS takes the longest to provision):

```sh
# Get the Lightdash URL
terraform output lightdash_url
# Output: https://lightdash-alb-123456789.eu-west-2.elb.amazonaws.com
```

Navigate to the URL in your browser. You should see the Lightdash login page.

!!! info "First-Time Setup"
    On the first visit, Lightdash will:
    1. Run database migrations (create tables in PostgreSQL)
    2. Prompt you to create an admin account
    3. Guide you through connecting GitHub and Snowflake

## Step 6: Connect GitHub and Snowflake

The configuration steps are identical to [Lightdash Cloud Setup](6-lightdash-setup.md):

1. **Create admin account** (email + password)
2. **Connect GitHub**:
   - Install Lightdash GitHub App
   - Select `dbt-transform` repository
   - Choose `main` branch

3. **Connect Snowflake**:
   - Warehouse type: Snowflake
   - Account: `your-account.snowflakecomputing.com`
   - User: `SVC_LIGHTDASH`
   - Role: `SVC_LIGHTDASH`
   - Database: `ANALYTICS`
   - Warehouse: `REPORTING`
   - Schema: `REPORTING`
   - Authentication: Key pair (paste private key)

4. **Compile dbt project**:
   - Lightdash clones the repo and runs `dbt compile`
   - Discovers models and metrics

Refer to [Lightdash Setup](6-lightdash-setup.md#connect-to-snowflake) for detailed instructions.

## Maintenance and Updates

### Updating Lightdash

Lightdash releases updates regularly. To upgrade:

1. **Check for new releases**: [Lightdash Releases](https://github.com/lightdash/lightdash/releases)
2. **Update the image tag** in your task definition:

```hcl
# terraform/aws/lightdash-task-definition.tf
image = "lightdash/lightdash:v0.123.0"  # Update to latest version
```

3. **Apply Terraform**:

```sh
terraform apply
# ECS will deploy the new task definition with zero downtime (rolling update)
```

4. **Verify** the upgrade by checking the Lightdash UI → Settings → About

!!! warning "Database Migrations"
    Lightdash automatically runs database migrations on startup. Always review the [changelog](https://github.com/lightdash/lightdash/blob/main/CHANGELOG.md) before upgrading to check for breaking changes.

### Backups

RDS automatically backs up the database daily (7-day retention). To restore:

```sh
# List available snapshots
aws rds describe-db-snapshots \
    --db-instance-identifier lightdash-db \
    --profile infrastructure-admin

# Restore from snapshot
aws rds restore-db-instance-from-db-snapshot \
    --db-instance-identifier lightdash-db-restored \
    --db-snapshot-identifier rds:lightdash-db-2026-02-19-03-00 \
    --profile infrastructure-admin
```

### Scaling

To handle increased load, scale the ECS service:

```hcl
# terraform/aws/lightdash-ecs-service.tf
desired_count = 3  # Run 3 tasks
```

Or enable auto-scaling based on CPU/memory:

```hcl
resource "aws_appautoscaling_target" "lightdash_ecs" {
  max_capacity       = 5
  min_capacity       = 1
  resource_id        = "service/${aws_ecs_cluster.lightdash.name}/${aws_ecs_service.lightdash.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "lightdash_cpu" {
  name               = "lightdash-cpu-autoscaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.lightdash_ecs.resource_id
  scalable_dimension = aws_appautoscaling_target.lightdash_ecs.scalable_dimension
  service_namespace  = aws_appautoscaling_target.lightdash_ecs.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value = 70.0
  }
}
```

### Monitoring

Monitor Lightdash with CloudWatch:

```sh
# View logs
aws logs tail /ecs/lightdash --follow --profile infrastructure-admin

# View ECS service metrics (CPU, memory, task count)
# Navigate to AWS Console → ECS → lightdash-cluster → lightdash-service → Metrics
```

Set up alarms for:
- High CPU/memory usage
- Task failures
- RDS storage approaching capacity
- Unhealthy target count (ALB health checks failing)

## Summary

You've deployed self-hosted Lightdash:

- [x] Created RDS PostgreSQL for Lightdash metadata storage
- [x] Deployed Lightdash to ECS Fargate with auto-scaling capability
- [x] Configured Application Load Balancer with HTTPS
- [x] Connected Lightdash to GitHub (dbt-transform repo) and Snowflake (SVC_LIGHTDASH)
- [x] Understood maintenance tasks (updates, backups, scaling, monitoring)

**Monthly cost: ~$65 (infrastructure) + $25 (Snowflake compute) = ~$90/month**

This is **96% cheaper** than Lightdash Cloud ($2400/month) but requires ongoing infrastructure management.

## What's Next

Now that Lightdash is running (Cloud or self-hosted), connect it to your dbt project and configure how it discovers models and metrics.

Continue to [Connect dbt Project](8-connect-dbt-project.md) →
