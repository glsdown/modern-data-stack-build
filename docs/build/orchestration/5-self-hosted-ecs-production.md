# Self-Hosted: ECS Production

!!! warning "Advanced - Requires VPC and Networking"
    This page covers production self-hosted Prefect on ECS, which requires:

    - AWS VPC with public and private subnets across multiple AZs
    - NAT gateways for private subnet internet access
    - Security group configuration
    - Optionally, a domain and ACM certificate for HTTPS

    **This infrastructure is not covered in the current guide.** Most users should use [Prefect Cloud](3-prefect-cloud-setup.md) instead.

    Self-hosting on ECS is only recommended if you have strict data sovereignty requirements AND existing AWS networking expertise.

On this page, you will:

- [x] Deploy Prefect server on ECS Fargate
- [x] Configure RDS PostgreSQL for state storage
- [x] Set up Application Load Balancer for access
- [x] Configure auto-scaling and monitoring

## Overview

This is the production-grade self-hosted option - Prefect server runs on ECS Fargate with RDS PostgreSQL for persistence. This provides high availability, managed scaling, and reduced operational burden compared to Docker Compose.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              AWS INFRASTRUCTURE                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────┐        │
│  │                    Application Load Balancer                    │        │
│  │                         (HTTPS :443)                            │        │
│  └────────────────────────────┬────────────────────────────────────┘        │
│                               │                                             │
│  ┌────────────────────────────▼────────────────────────────────────┐        │
│  │                      ECS Cluster (Fargate)                      │        │
│  │  ┌─────────────────────────────────────────────────────────┐    │        │
│  │  │              Prefect Server Service                     │    │        │
│  │  │  ┌─────────────┐  ┌─────────────┐                       │    │        │
│  │  │  │   Task 1    │  │   Task 2    │  (Multi-AZ)           │    │        │
│  │  │  └─────────────┘  └─────────────┘                       │    │        │
│  │  └─────────────────────────────────────────────────────────┘    │        │
│  │                                                                 │        │
│  │  ┌─────────────────────────────────────────────────────────┐    │        │
│  │  │              Prefect Worker Service                     │    │        │
│  │  │  ┌─────────────┐  ┌─────────────┐  (Auto-scaling)       │    │        │
│  │  │  │   Task 1    │  │   Task N    │                       │    │        │
│  │  │  └─────────────┘  └─────────────┘                       │    │        │
│  │  └─────────────────────────────────────────────────────────┘    │        │
│  └─────────────────────────────────────────────────────────────────┘        │
│                               │                                             │
│  ┌────────────────────────────▼────────────────────────────────────┐        │
│  │                      RDS PostgreSQL                             │        │
│  │                    (Multi-AZ optional)                          │        │
│  └─────────────────────────────────────────────────────────────────┘        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Estimated cost: ~$67/month** (Fargate + RDS + ALB)

## Prerequisites

- [x] **AWS VPC with multi-AZ networking** - Public subnets for ALB, private subnets for ECS/RDS, NAT gateways. This is not covered in this guide.
- [x] Terraform configured with remote state
- [x] Domain name (optional, for HTTPS)
- [x] ACM certificate (optional, for HTTPS)

!!! tip "Don't Have VPC Infrastructure?"
    Setting up VPC with multi-AZ networking is a significant undertaking. We recommend using [Prefect Cloud](3-prefect-cloud-setup.md) instead - it provides the same features without any infrastructure management.

## Terraform Module

Prefect provides an official Terraform module for ECS deployment. We'll use it with customisations.

### Project Structure

```
terraform/
└── prefect-ecs/
    ├── config/
    │   ├── backend.tf
    │   ├── main.tf
    │   ├── providers.tf
    │   ├── variables.tf
    │   ├── terraform.tfvars
    │   ├── rds.tf
    │   ├── ecs.tf
    │   ├── alb.tf
    │   └── outputs.tf
    └── modules/
```

### Backend and Provider

Create `terraform/prefect-ecs/config/backend.tf`:

```hcl
terraform {
  backend "s3" {
    bucket         = "your-terraform-state-bucket"
    key            = "prefect-ecs/terraform.tfstate"
    region         = "eu-west-2"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

Create `terraform/prefect-ecs/config/main.tf`:

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.0"
    }
  }
}
```

Create `terraform/prefect-ecs/config/providers.tf`:

```hcl
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = "data-platform"
      ManagedBy   = "terraform"
      Component   = "prefect-server"
      Environment = var.environment
    }
  }
}
```

### Variables

Create `terraform/prefect-ecs/config/variables.tf`:

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "eu-west-2"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "production"
}

variable "vpc_id" {
  description = "VPC ID"
  type        = string
}

variable "private_subnet_ids" {
  description = "Private subnet IDs for ECS tasks and RDS"
  type        = list(string)
}

variable "public_subnet_ids" {
  description = "Public subnet IDs for ALB"
  type        = list(string)
}

variable "allowed_cidr_blocks" {
  description = "CIDR blocks allowed to access Prefect UI"
  type        = list(string)
  default     = ["10.0.0.0/8"]
}

variable "prefect_image" {
  description = "Prefect Docker image"
  type        = string
  default     = "prefecthq/prefect:3-latest"
}

variable "server_cpu" {
  description = "CPU units for Prefect server (256, 512, 1024, etc.)"
  type        = number
  default     = 512
}

variable "server_memory" {
  description = "Memory (MB) for Prefect server"
  type        = number
  default     = 1024
}

variable "db_instance_class" {
  description = "RDS instance class"
  type        = string
  default     = "db.t3.micro"
}

variable "db_allocated_storage" {
  description = "RDS allocated storage in GB"
  type        = number
  default     = 20
}
```

### RDS PostgreSQL

Create `terraform/prefect-ecs/config/rds.tf`:

```hcl
# -----------------------------------------------------------------------------
# Random Password for RDS
# -----------------------------------------------------------------------------
resource "random_password" "db_password" {
  length  = 32
  special = false
}

# -----------------------------------------------------------------------------
# Security Group for RDS
# -----------------------------------------------------------------------------
resource "aws_security_group" "rds" {
  name        = "prefect-rds-${var.environment}"
  description = "Security group for Prefect RDS"
  vpc_id      = var.vpc_id

  ingress {
    description     = "PostgreSQL from ECS"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.ecs_tasks.id]
  }

  tags = {
    Name = "prefect-rds-${var.environment}"
  }
}

# -----------------------------------------------------------------------------
# RDS Subnet Group
# -----------------------------------------------------------------------------
resource "aws_db_subnet_group" "prefect" {
  name       = "prefect-${var.environment}"
  subnet_ids = var.private_subnet_ids

  tags = {
    Name = "prefect-${var.environment}"
  }
}

# -----------------------------------------------------------------------------
# RDS Instance
# -----------------------------------------------------------------------------
resource "aws_db_instance" "prefect" {
  identifier = "prefect-${var.environment}"

  engine         = "postgres"
  engine_version = "15"
  instance_class = var.db_instance_class

  allocated_storage     = var.db_allocated_storage
  max_allocated_storage = var.db_allocated_storage * 2
  storage_type          = "gp3"
  storage_encrypted     = true

  db_name  = "prefect"
  username = "prefect"
  password = random_password.db_password.result

  db_subnet_group_name   = aws_db_subnet_group.prefect.name
  vpc_security_group_ids = [aws_security_group.rds.id]

  backup_retention_period = 7
  backup_window           = "03:00-04:00"
  maintenance_window      = "sun:04:00-sun:05:00"

  skip_final_snapshot       = false
  final_snapshot_identifier = "prefect-${var.environment}-final"

  tags = {
    Name = "prefect-${var.environment}"
  }
}

# -----------------------------------------------------------------------------
# Store Password in Secrets Manager
# -----------------------------------------------------------------------------
resource "aws_secretsmanager_secret" "db_password" {
  name        = "prefect/rds-password"
  description = "Prefect RDS PostgreSQL password"
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id     = aws_secretsmanager_secret.db_password.id
  secret_string = random_password.db_password.result
}
```

### ECS Cluster and Services

Create `terraform/prefect-ecs/config/ecs.tf`:

```hcl
# -----------------------------------------------------------------------------
# ECS Cluster
# -----------------------------------------------------------------------------
resource "aws_ecs_cluster" "prefect" {
  name = "prefect-${var.environment}"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

# -----------------------------------------------------------------------------
# Security Group for ECS Tasks
# -----------------------------------------------------------------------------
resource "aws_security_group" "ecs_tasks" {
  name        = "prefect-ecs-tasks-${var.environment}"
  description = "Security group for Prefect ECS tasks"
  vpc_id      = var.vpc_id

  ingress {
    description     = "HTTP from ALB"
    from_port       = 4200
    to_port         = 4200
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    description = "All outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "prefect-ecs-tasks-${var.environment}"
  }
}

# -----------------------------------------------------------------------------
# IAM Roles for ECS
# -----------------------------------------------------------------------------
resource "aws_iam_role" "ecs_task_execution" {
  name = "prefect-ecs-execution-${var.environment}"

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
}

resource "aws_iam_role_policy_attachment" "ecs_task_execution" {
  role       = aws_iam_role.ecs_task_execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

resource "aws_iam_role" "ecs_task" {
  name = "prefect-ecs-task-${var.environment}"

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
}

# -----------------------------------------------------------------------------
# CloudWatch Log Group
# -----------------------------------------------------------------------------
resource "aws_cloudwatch_log_group" "prefect" {
  name              = "/ecs/prefect-${var.environment}"
  retention_in_days = 30
}

# -----------------------------------------------------------------------------
# Task Definition - Prefect Server
# -----------------------------------------------------------------------------
resource "aws_ecs_task_definition" "prefect_server" {
  family                   = "prefect-server-${var.environment}"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = var.server_cpu
  memory                   = var.server_memory
  execution_role_arn       = aws_iam_role.ecs_task_execution.arn
  task_role_arn            = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([
    {
      name  = "prefect-server"
      image = var.prefect_image

      command = ["prefect", "server", "start", "--host", "0.0.0.0"]

      environment = [
        {
          name  = "PREFECT_API_DATABASE_CONNECTION_URL"
          value = "postgresql+asyncpg://prefect:${random_password.db_password.result}@${aws_db_instance.prefect.endpoint}/prefect"
        },
        {
          name  = "PREFECT_SERVER_API_HOST"
          value = "0.0.0.0"
        },
        {
          name  = "PREFECT_SERVER_API_PORT"
          value = "4200"
        }
      ]

      portMappings = [
        {
          containerPort = 4200
          protocol      = "tcp"
        }
      ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.prefect.name
          "awslogs-region"        = var.aws_region
          "awslogs-stream-prefix" = "server"
        }
      }

      healthCheck = {
        command     = ["CMD-SHELL", "curl -f http://localhost:4200/api/health || exit 1"]
        interval    = 30
        timeout     = 5
        retries     = 3
        startPeriod = 60
      }
    }
  ])
}

# -----------------------------------------------------------------------------
# ECS Service - Prefect Server
# -----------------------------------------------------------------------------
resource "aws_ecs_service" "prefect_server" {
  name            = "prefect-server"
  cluster         = aws_ecs_cluster.prefect.id
  task_definition = aws_ecs_task_definition.prefect_server.arn
  desired_count   = 1
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = var.private_subnet_ids
    security_groups  = [aws_security_group.ecs_tasks.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.prefect.arn
    container_name   = "prefect-server"
    container_port   = 4200
  }

  depends_on = [aws_lb_listener.prefect]
}
```

### Application Load Balancer

Create `terraform/prefect-ecs/config/alb.tf`:

```hcl
# -----------------------------------------------------------------------------
# Security Group for ALB
# -----------------------------------------------------------------------------
resource "aws_security_group" "alb" {
  name        = "prefect-alb-${var.environment}"
  description = "Security group for Prefect ALB"
  vpc_id      = var.vpc_id

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = var.allowed_cidr_blocks
  }

  ingress {
    description = "HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = var.allowed_cidr_blocks
  }

  egress {
    description = "All outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "prefect-alb-${var.environment}"
  }
}

# -----------------------------------------------------------------------------
# Application Load Balancer
# -----------------------------------------------------------------------------
resource "aws_lb" "prefect" {
  name               = "prefect-${var.environment}"
  internal           = true  # Change to false for public access
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = var.public_subnet_ids

  tags = {
    Name = "prefect-${var.environment}"
  }
}

# -----------------------------------------------------------------------------
# Target Group
# -----------------------------------------------------------------------------
resource "aws_lb_target_group" "prefect" {
  name        = "prefect-${var.environment}"
  port        = 4200
  protocol    = "HTTP"
  vpc_id      = var.vpc_id
  target_type = "ip"

  health_check {
    enabled             = true
    healthy_threshold   = 2
    interval            = 30
    matcher             = "200"
    path                = "/api/health"
    port                = "traffic-port"
    timeout             = 5
    unhealthy_threshold = 3
  }
}

# -----------------------------------------------------------------------------
# Listener (HTTP - redirect to HTTPS in production)
# -----------------------------------------------------------------------------
resource "aws_lb_listener" "prefect" {
  load_balancer_arn = aws_lb.prefect.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.prefect.arn
  }
}

# For HTTPS, add:
# resource "aws_lb_listener" "prefect_https" {
#   load_balancer_arn = aws_lb.prefect.arn
#   port              = 443
#   protocol          = "HTTPS"
#   ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
#   certificate_arn   = var.acm_certificate_arn
#
#   default_action {
#     type             = "forward"
#     target_group_arn = aws_lb_target_group.prefect.arn
#   }
# }
```

### Outputs

Create `terraform/prefect-ecs/config/outputs.tf`:

```hcl
output "ecs_cluster_name" {
  description = "ECS cluster name"
  value       = aws_ecs_cluster.prefect.name
}

output "alb_dns_name" {
  description = "ALB DNS name"
  value       = aws_lb.prefect.dns_name
}

output "prefect_ui_url" {
  description = "Prefect UI URL"
  value       = "http://${aws_lb.prefect.dns_name}"
}

output "prefect_api_url" {
  description = "Prefect API URL for workers and CLI"
  value       = "http://${aws_lb.prefect.dns_name}/api"
}

output "rds_endpoint" {
  description = "RDS endpoint"
  value       = aws_db_instance.prefect.endpoint
}
```

## Deploy the Infrastructure

```sh
cd terraform/prefect-ecs/config

terraform init
terraform plan
terraform apply
```

This will take 10-15 minutes to provision all resources.

## Connect to Prefect

Configure your CLI to use the self-hosted server:

```sh
# Get the ALB DNS name from Terraform outputs
export PREFECT_API_URL="http://ALB_DNS_NAME/api"

# Verify connection
prefect version
prefect work-pool ls
```

## Create Work Pools

```sh
# Create a process work pool for simple tasks
prefect work-pool create default --type process

# Create an ECS work pool for containerised flows
prefect work-pool create production --type ecs
```

## Monitoring

### CloudWatch Metrics

ECS automatically publishes metrics to CloudWatch:

- CPU/Memory utilisation
- Running task count
- Service health

### CloudWatch Logs

View logs in the AWS Console or via CLI:

```sh
aws logs tail /ecs/prefect-production --follow
```

### Alarms (Optional)

Add CloudWatch alarms for critical metrics:

```hcl
resource "aws_cloudwatch_metric_alarm" "prefect_cpu" {
  alarm_name          = "prefect-high-cpu-${var.environment}"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/ECS"
  period              = 300
  statistic           = "Average"
  threshold           = 80

  dimensions = {
    ClusterName = aws_ecs_cluster.prefect.name
    ServiceName = aws_ecs_service.prefect_server.name
  }

  alarm_actions = [var.sns_topic_arn]  # Add SNS topic for notifications
}
```

## Summary

You've deployed a production-grade Prefect server on AWS:

- [x] ECS Fargate for managed container orchestration
- [x] RDS PostgreSQL with automated backups
- [x] Application Load Balancer for access
- [x] CloudWatch for logging and monitoring

## What's Next

With the server running, configure work pools and workers for your specific workloads.

Continue to [Work Pools and Workers](6-work-pools-and-workers.md) →
