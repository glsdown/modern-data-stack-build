# Self-Hosted Setup

On this page, you will:

- [x] Deploy Airbyte on AWS ECS Fargate with Terraform
- [x] Configure RDS PostgreSQL for Airbyte metadata
- [x] Set up an ALB for secure access
- [x] Run Airbyte locally with Docker Compose for development

!!! note "Optional Page"
    This page covers self-hosted Airbyte deployment. If you're using Airbyte Cloud, skip to [Snowflake Infrastructure](5-snowflake-infrastructure.md).

## Overview

Self-hosted Airbyte runs on your own infrastructure, giving you full control over data residency, costs, and customisation.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SELF-HOSTED ARCHITECTURE (AWS)                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌───────────────────────────────────────────────────────────┐              │
│  │                          VPC                              │              │
│  │                                                           │              │
│  │  ┌─────────────┐     ┌─────────────────────────────────┐  │              │
│  │  │    ALB      │     │          ECS Cluster            │  │              │
│  │  │  (HTTPS)    │────▶│                                 │  │              │
│  │  └─────────────┘     │  ┌───────────┐  ┌───────────┐   │  │              │
│  │                      │  │  Server   │  │  Worker   │   │  │              │
│  │                      │  │  (Fargate)│  │  (Fargate)│   │  │              │
│  │                      │  └─────┬─────┘  └───────────┘   │  │              │
│  │                      └────────┼────────────────────────┘  │              │
│  │                               │                           │              │
│  │                      ┌────────▼────────┐                  │              │
│  │                      │  RDS PostgreSQL │                  │              │
│  │                      │  (metadata)     │                  │              │
│  │                      └─────────────────┘                  │              │
│  └───────────────────────────────────────────────────────────┘              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Local Development with Docker Compose

Before deploying to AWS, test Airbyte locally with Docker Compose.

### Prerequisites

- Docker Desktop installed and running
- At least 4 GB RAM allocated to Docker

### Run Airbyte Locally

```sh
# Clone Airbyte repository
git clone https://github.com/airbytehq/airbyte.git
cd airbyte

# Run Airbyte
./run-ab-platform.sh
```

Airbyte will be available at `http://localhost:8000`.

Default credentials:

- **Username**: `airbyte`
- **Password**: `password`

!!! warning "Resource Usage"
    Airbyte runs several containers and uses significant resources. Close it when not in use:
    ```sh
    docker compose down
    ```

### Test the UI

1. Open `http://localhost:8000`
2. Log in with default credentials
3. Explore the Sources and Destinations pages
4. Verify you can see the HubSpot source connector

!!! danger "Testing only"
  This local instance is for testing only. Do not use it for production workloads.

## ECS Deployment with Terraform

### Project Structure

Add Airbyte infrastructure to your Terraform repository:

```
terraform/
├── aws/
│   ├── airbyte/
│   │   ├── main.tf           # ECS cluster, services, task definitions
│   │   ├── variables.tf      # Input variables
│   │   ├── outputs.tf        # Output values
│   │   ├── rds.tf            # RDS PostgreSQL for metadata
│   │   ├── alb.tf            # Application Load Balancer
│   │   ├── iam.tf            # IAM roles for ECS tasks
│   │   ├── security_groups.tf # Security group rules
│   │   └── secrets.tf        # Secrets Manager references
│   └── ...
└── ...
```

### ECS Cluster

```hcl
# terraform/aws/airbyte/main.tf

resource "aws_ecs_cluster" "airbyte" {
  name = "airbyte"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }

  tags = {
    Project     = "modern-data-stack"
    Component   = "airbyte"
    Environment = "production"
  }
}
```

### RDS PostgreSQL

Airbyte requires a PostgreSQL database for storing configuration, connection state, and job history.

```hcl
# terraform/aws/airbyte/rds.tf

resource "aws_db_instance" "airbyte_metadata" {
  identifier     = "airbyte-metadata"
  engine         = "postgres"
  engine_version = "16.4"
  instance_class = "db.t4g.micro"

  allocated_storage     = 20
  max_allocated_storage = 50
  storage_type          = "gp3"

  db_name  = "airbyte"
  username = "airbyte"
  password = var.rds_password

  vpc_security_group_ids = [aws_security_group.airbyte_rds.id]
  db_subnet_group_name   = aws_db_subnet_group.airbyte.name

  backup_retention_period = 7
  skip_final_snapshot     = false
  final_snapshot_identifier = "airbyte-metadata-final"

  tags = {
    Project   = "modern-data-stack"
    Component = "airbyte"
  }
}
```

### ECS Task Definition

The Airbyte server runs as a Fargate task:

```hcl
# terraform/aws/airbyte/main.tf

resource "aws_ecs_task_definition" "airbyte_server" {
  family                   = "airbyte-server"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = 1024  # 1 vCPU
  memory                   = 2048  # 2 GB
  execution_role_arn       = aws_iam_role.airbyte_execution.arn
  task_role_arn            = aws_iam_role.airbyte_task.arn

  container_definitions = jsonencode([
    {
      name      = "airbyte-server"
      image     = "airbyte/server:latest"
      essential = true

      portMappings = [
        {
          containerPort = 8001
          protocol      = "tcp"
        }
      ]

      environment = [
        {
          name  = "DATABASE_URL"
          value = "jdbc:postgresql://${aws_db_instance.airbyte_metadata.endpoint}/airbyte"
        },
        {
          name  = "DATABASE_USER"
          value = "airbyte"
        },
      ]

      secrets = [
        {
          name      = "DATABASE_PASSWORD"
          valueFrom = aws_secretsmanager_secret.airbyte_rds_password.arn
        }
      ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.airbyte.name
          "awslogs-region"        = var.aws_region
          "awslogs-stream-prefix" = "server"
        }
      }
    }
  ])
}
```

### ECS Service

```hcl
resource "aws_ecs_service" "airbyte_server" {
  name            = "airbyte-server"
  cluster         = aws_ecs_cluster.airbyte.id
  task_definition = aws_ecs_task_definition.airbyte_server.arn
  desired_count   = 1
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = var.private_subnet_ids
    security_groups  = [aws_security_group.airbyte_server.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.airbyte.arn
    container_name   = "airbyte-server"
    container_port   = 8001
  }
}
```

### Application Load Balancer

```hcl
# terraform/aws/airbyte/alb.tf

resource "aws_lb" "airbyte" {
  name               = "airbyte-alb"
  internal           = true  # Internal only — not public-facing
  load_balancer_type = "application"
  security_groups    = [aws_security_group.airbyte_alb.id]
  subnets            = var.private_subnet_ids

  tags = {
    Project   = "modern-data-stack"
    Component = "airbyte"
  }
}

resource "aws_lb_listener" "airbyte_https" {
  load_balancer_arn = aws_lb.airbyte.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = var.certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.airbyte.arn
  }
}

resource "aws_lb_target_group" "airbyte" {
  name        = "airbyte-server"
  port        = 8001
  protocol    = "HTTP"
  vpc_id      = var.vpc_id
  target_type = "ip"

  health_check {
    path                = "/api/v1/health"
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 30
  }
}
```

### IAM Roles

```hcl
# terraform/aws/airbyte/iam.tf

# Execution role — used by ECS to pull images and write logs
resource "aws_iam_role" "airbyte_execution" {
  name = "airbyte-ecs-execution"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ecs-tasks.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "airbyte_execution" {
  role       = aws_iam_role.airbyte_execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

# Task role — used by the running container
resource "aws_iam_role" "airbyte_task" {
  name = "airbyte-ecs-task"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ecs-tasks.amazonaws.com"
        }
      }
    ]
  })
}

# Allow task to read secrets
resource "aws_iam_role_policy" "airbyte_secrets" {
  name = "airbyte-secrets-access"
  role = aws_iam_role.airbyte_task.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue",
          "secretsmanager:DescribeSecret"
        ]
        Resource = [
          "arn:aws:secretsmanager:${var.aws_region}:*:secret:airbyte/*"
        ]
      }
    ]
  })
}
```

### Security Groups

```hcl
# terraform/aws/airbyte/security_groups.tf

resource "aws_security_group" "airbyte_alb" {
  name_prefix = "airbyte-alb-"
  vpc_id      = var.vpc_id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = var.allowed_cidr_blocks  # Restrict to your network
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "airbyte_server" {
  name_prefix = "airbyte-server-"
  vpc_id      = var.vpc_id

  ingress {
    from_port       = 8001
    to_port         = 8001
    protocol        = "tcp"
    security_groups = [aws_security_group.airbyte_alb.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "airbyte_rds" {
  name_prefix = "airbyte-rds-"
  vpc_id      = var.vpc_id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.airbyte_server.id]
  }
}
```

### Variables

```hcl
# terraform/aws/airbyte/variables.tf

variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "eu-west-2"
}

variable "vpc_id" {
  description = "VPC ID for Airbyte deployment"
  type        = string
}

variable "private_subnet_ids" {
  description = "Private subnet IDs for ECS and RDS"
  type        = list(string)
}

variable "allowed_cidr_blocks" {
  description = "CIDR blocks allowed to access Airbyte UI"
  type        = list(string)
}

variable "certificate_arn" {
  description = "ACM certificate ARN for HTTPS"
  type        = string
}

variable "rds_password" {
  description = "Password for RDS PostgreSQL"
  type        = string
  sensitive   = true
}
```

### Deploy

```sh
cd terraform/aws/airbyte

terraform init
terraform plan
terraform apply
```

### Store Self-Hosted Credentials

For self-hosted Airbyte, store the API URL in Secrets Manager:

```sh
aws secretsmanager create-secret \
    --name "airbyte/api-credentials" \
    --description "Airbyte self-hosted API credentials" \
    --secret-string '{
        "api_url": "https://airbyte.internal.example.com/api/v1",
        "username": "airbyte",
        "password": "YOUR_PASSWORD_HERE"
    }' \
    --region eu-west-2
```

## Upgrading Airbyte

Self-hosted Airbyte requires manual upgrades:

1. **Check release notes** at [github.com/airbytehq/airbyte/releases](https://github.com/airbytehq/airbyte/releases)
2. **Update the Docker image tag** in your ECS task definition
3. **Run database migrations** if required (check release notes)
4. **Deploy the update** via Terraform

```hcl
# Update the image tag
image = "airbyte/server:0.64.0"  # Update to desired version
```

!!! warning "Breaking Changes"
    Major version upgrades may require migration steps. Always review release notes before upgrading. Test upgrades in a staging environment first.

## Monitoring

### CloudWatch Logs

Airbyte server and worker logs are sent to CloudWatch:

```sh
aws logs tail /ecs/airbyte --follow
```

### CloudWatch Alarms

Set up alarms for key metrics:

```hcl
resource "aws_cloudwatch_metric_alarm" "airbyte_cpu" {
  alarm_name          = "airbyte-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/ECS"
  period              = 300
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "Airbyte ECS CPU utilization is high"

  dimensions = {
    ClusterName = aws_ecs_cluster.airbyte.name
    ServiceName = aws_ecs_service.airbyte_server.name
  }
}
```

## Summary

You've deployed self-hosted Airbyte:

- [x] Ran Airbyte locally with Docker Compose for testing
- [x] Deployed ECS Fargate infrastructure with Terraform
- [x] Configured RDS PostgreSQL for metadata storage
- [x] Set up ALB with HTTPS for secure access
- [x] Stored API credentials in AWS Secrets Manager

## What's Next

With Airbyte deployed, set up the Snowflake infrastructure for Airbyte-loaded data.

Continue to [Snowflake Infrastructure](5-snowflake-infrastructure.md) →
