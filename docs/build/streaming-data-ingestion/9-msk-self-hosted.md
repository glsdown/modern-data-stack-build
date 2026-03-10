# MSK Self-Hosted (Optional)

!!! info "Optional Advanced Topic"
    This page is **optional**. Most teams should use Confluent Cloud (covered in earlier pages) for simplicity and lower total cost. Only consider MSK if you have AWS-centric architecture, dedicated DevOps resources, or specific data residency requirements.

On this page, you will:

- [x] Evaluate AWS MSK vs Confluent Cloud for your use case
- [x] Deploy 3-broker MSK provisioned cluster via Terraform
- [x] Configure VPC networking and security groups
- [x] Deploy MSK Connect for Kafka Connect workers
- [x] Set up self-hosted Confluent Schema Registry on ECS
- [x] Compare total cost of ownership (infrastructure + ops time)
- [x] Understand when MSK makes sense

## Overview

AWS Managed Streaming for Kafka (MSK) is Amazon's Kafka service. Unlike Confluent Cloud (fully managed), MSK requires you to manage Kafka configuration, Schema Registry, and Kafka Connect separately.

**Trade-off:** Lower infrastructure cost but significantly higher operational burden.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    MSK SELF-HOSTED ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  VPC (10.0.0.0/16)                                                      │
│  ──────────────────                                                     │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │ Private Subnets (3 AZs)                                        │    │
│  │                                                                 │    │
│  │  MSK Cluster                                                    │    │
│  │  ───────────                                                    │    │
│  │  ┌──────────┐     ┌──────────┐     ┌──────────┐               │    │
│  │  │ Broker 1 │────▶│ Broker 2 │────▶│ Broker 3 │               │    │
│  │  │ AZ-1     │     │ AZ-2     │     │ AZ-3     │               │    │
│  │  │ m5.large │     │ m5.large │     │ m5.large │               │    │
│  │  └──────────┘     └──────────┘     └──────────┘               │    │
│  │       │                  │                  │                  │    │
│  │       └──────────────────┼──────────────────┘                  │    │
│  │                          │                                     │    │
│  │  MSK Connect (Kafka Connect Workers)                           │    │
│  │  ────────────────────────────────────                          │    │
│  │  ┌──────────────────────────────────────┐                      │    │
│  │  │ ECS Tasks (Fargate)                  │                      │    │
│  │  │ • Snowflake Sink Connector           │                      │    │
│  │  │ • S3 Sink Connector                  │                      │    │
│  │  └──────────────────────────────────────┘                      │    │
│  │                          │                                     │    │
│  │  Schema Registry (Self-Hosted)                                 │    │
│  │  ─────────────────────────────                                 │    │
│  │  ┌──────────────────────────────────────┐                      │    │
│  │  │ ECS Tasks (Fargate)                  │                      │    │
│  │  │ • Confluent Schema Registry          │                      │    │
│  │  │ • Backed by RDS PostgreSQL           │                      │    │
│  │  └──────────────────────────────────────┘                      │    │
│  │                                                                 │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  Cost: $450/month infrastructure + $360-720/month ops time              │
│  Total: $810-1,170/month (vs $217-277/month for Confluent Cloud)       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## When to Use MSK

### Good Reasons

✅ **AWS-centric architecture** - Already using MSK for other teams, amortised ops cost
✅ **Data residency** - Strict requirements to keep data in specific AWS regions/accounts
✅ **Enterprise AWS commitment** - AWS discount programmes make MSK significantly cheaper
✅ **Existing DevOps team** - Engineers already managing MSK clusters
✅ **Very high throughput** - > 100 GB/day where MSK cost advantage matters

### Bad Reasons

❌ **Lower cost** - Total cost (infra + ops) is usually higher than Confluent Cloud
❌ **More control** - You're debugging Kafka internals instead of building data pipelines
❌ **Learning** - Learn Kafka concepts with Confluent Cloud first, add ops complexity later
❌ **Avoiding vendor lock-in** - Kafka is open source, migration is straightforward

### Cost Reality Check

| Workload | Confluent Cloud Total | MSK Total | Winner |
|----------|---------------------|-----------|--------|
| **5 GB/day** | $217/month | $810/month | Confluent Cloud (73% cheaper) |
| **50 GB/day** | $310/month | $950/month | Confluent Cloud (67% cheaper) |
| **200 GB/day** | $900/month | $1,400/month | Confluent Cloud (36% cheaper) |

MSK becomes competitive only at **very high scale** (> 500 GB/day) or when ops time is free (existing team).

## MSK Cluster Deployment

### Prerequisites

You need:
- AWS VPC with private subnets in 3 AZs
- Terraform (>= 1.0)
- AWS CLI configured
- Existing STREAMING database in Snowflake (from page 4)

### Terraform Configuration

Create `~/src/terraform/aws/msk/main.tf`:

```hcl
terraform {
  required_version = ">= 1.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket         = "your-terraform-state-bucket"
    key            = "aws/msk/terraform.tfstate"
    region         = "eu-west-2"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}

provider "aws" {
  region = "eu-west-2"

  default_tags {
    tags = {
      Project     = "ModernDataStack"
      ManagedBy   = "Terraform"
      Environment = "production"
      Component   = "streaming"
    }
  }
}

# Data source: Get VPC and subnets
data "aws_vpc" "main" {
  tags = {
    Name = "data-platform-vpc"
  }
}

data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }

  tags = {
    Type = "private"
  }
}

# Security group for MSK cluster
resource "aws_security_group" "msk_cluster" {
  name_description = "msk-cluster"
  description = "Security group for MSK cluster"
  vpc_id      = data.aws_vpc.main.id

  # Allow Kafka protocol (9092) from VPC
  ingress {
    from_port   = 9092
    to_port     = 9092
    protocol    = "tcp"
    cidr_blocks = [data.aws_vpc.main.cidr_block]
    description = "Kafka plaintext"
  }

  # Allow TLS Kafka (9094) from VPC
  ingress {
    from_port   = 9094
    to_port     = 9094
    protocol    = "tcp"
    cidr_blocks = [data.aws_vpc.main.cidr_block]
    description = "Kafka TLS"
  }

  # Allow Zookeeper (2181) from VPC
  ingress {
    from_port   = 2181
    to_port     = 2181
    protocol    = "tcp"
    cidr_blocks = [data.aws_vpc.main.cidr_block]
    description = "Zookeeper"
  }

  # Allow all outbound
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow all outbound"
  }

  tags = {
    Name = "msk-cluster-sg"
  }
}

# CloudWatch log group for MSK logs
resource "aws_cloudwatch_log_group" "msk" {
  name              = "/aws/msk/data-platform-cluster"
  retention_in_days = 7

  tags = {
    Name = "msk-cluster-logs"
  }
}

# MSK cluster configuration
resource "aws_msk_configuration" "this" {
  name              = "data-platform-msk-config"
  kafka_versions    = ["3.5.1"]
  server_properties = <<EOF
auto.create.topics.enable=false
default.replication.factor=3
min.insync.replicas=2
num.partitions=3
log.retention.hours=168
log.retention.bytes=107374182400
log.segment.bytes=1073741824
compression.type=snappy
EOF

  description = "MSK configuration for data platform"
}

# MSK cluster (3-broker provisioned)
resource "aws_msk_cluster" "this" {
  cluster_name           = "data-platform-cluster"
  kafka_version          = "3.5.1"
  number_of_broker_nodes = 3

  broker_node_group_info {
    instance_type   = "kafka.m5.large"  # 2 vCPU, 8 GB RAM
    client_subnets  = slice(data.aws_subnets.private.ids, 0, 3)
    security_groups = [aws_security_group.msk_cluster.id]

    storage_info {
      ebs_storage_info {
        volume_size = 100  # 100 GB per broker
      }
    }

    connectivity_info {
      public_access {
        type = "DISABLED"
      }
    }
  }

  encryption_info {
    encryption_in_transit {
      client_broker = "TLS"  # Require TLS
      in_cluster    = true   # Encrypt broker-to-broker
    }

    encryption_at_rest_kms_key_arn = aws_kms_key.msk.arn
  }

  configuration_info {
    arn      = aws_msk_configuration.this.arn
    revision = aws_msk_configuration.this.latest_revision
  }

  logging_info {
    broker_logs {
      cloudwatch_logs {
        enabled   = true
        log_group = aws_cloudwatch_log_group.msk.name
      }
    }
  }

  tags = {
    Name = "data-platform-msk-cluster"
  }
}

# KMS key for encryption at rest
resource "aws_kms_key" "msk" {
  description             = "KMS key for MSK cluster encryption"
  deletion_window_in_days = 7
  enable_key_rotation     = true

  tags = {
    Name = "msk-cluster-key"
  }
}

resource "aws_kms_alias" "msk" {
  name          = "alias/msk-cluster"
  target_key_id = aws_kms_key.msk.key_id
}

# Outputs
output "msk_cluster_arn" {
  value       = aws_msk_cluster.this.arn
  description = "ARN of MSK cluster"
}

output "msk_bootstrap_brokers_tls" {
  value       = aws_msk_cluster.this.bootstrap_brokers_tls
  description = "TLS connection string for MSK cluster"
  sensitive   = true
}

output "msk_zookeeper_connect_string" {
  value       = aws_msk_cluster.this.zookeeper_connect_string
  description = "Zookeeper connection string"
  sensitive   = true
}
```

### Deploy MSK Cluster

```sh
cd ~/src/terraform/aws/msk

# Initialise
terraform init

# Plan
terraform plan -out=tfplan

# Apply (MSK cluster provisioning takes some time)
terraform apply tfplan
```

**Expected output:**
```
aws_msk_cluster.this: Creating... (this may take 15-30 minutes)
aws_msk_cluster.this: Still creating... [5m0s elapsed]
aws_msk_cluster.this: Still creating... [10m0s elapsed]
aws_msk_cluster.this: Still creating... [15m0s elapsed]
aws_msk_cluster.this: Creation complete after 22m15s

Outputs:
msk_cluster_arn = "arn:aws:kafka:eu-west-2:123456789012:cluster/data-platform-cluster/abc-123"
msk_bootstrap_brokers_tls = "b-1.dataplatformcluster.abc123.kafka.eu-west-2.amazonaws.com:9094,..."
```

### Verify MSK Cluster

```sh
# Get cluster ARN
MSK_ARN=$(terraform output -raw msk_cluster_arn)

# Check cluster status
aws kafka describe-cluster --cluster-arn $MSK_ARN --region eu-west-2

# Expected state: "ACTIVE"
```

## Schema Registry Deployment

MSK doesn't include Schema Registry. Deploy Confluent Schema Registry on ECS Fargate.

### Why Self-Host Schema Registry?

**Options:**
1. **AWS Glue Schema Registry** - Free but limited (no Avro logical types, no compatibility checks)
2. **Confluent Schema Registry** - Industry standard, full features, needs self-hosting
3. **No Schema Registry** - JSON without schemas (not recommended for production)

**Recommendation:** Self-host Confluent Schema Registry for full Avro support.

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SCHEMA REGISTRY ON ECS                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Application Load Balancer (Internal)                                  │
│  ─────────────────────────────────────                                 │
│  ┌──────────────────────────────────────┐                              │
│  │ schema-registry.internal             │                              │
│  │ Port 8081                            │                              │
│  └──────────────────────────────────────┘                              │
│                  │                                                      │
│                  ▼                                                      │
│  ECS Service (Fargate)                                                  │
│  ─────────────────────                                                  │
│  ┌──────────────────────────────────────┐                              │
│  │ Schema Registry Task                 │                              │
│  │ • 0.5 vCPU, 1 GB RAM                 │                              │
│  │ • Confluent Image                    │                              │
│  │ • Stores schemas in RDS              │                              │
│  └──────────────────────────────────────┘                              │
│                  │                                                      │
│                  ▼                                                      │
│  RDS PostgreSQL (db.t3.micro)                                           │
│  ────────────────────────────                                           │
│  ┌──────────────────────────────────────┐                              │
│  │ Schema metadata storage              │                              │
│  │ • Multi-AZ: No (dev/test)            │                              │
│  │ • Backup: 7-day retention            │                              │
│  └──────────────────────────────────────┘                              │
│                                                                         │
│  Cost: ~$50/month (ECS $15 + RDS $20 + ALB $15)                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Terraform Configuration

Create `~/src/terraform/aws/msk/schema_registry.tf`:

```hcl
# RDS PostgreSQL for Schema Registry metadata
resource "aws_db_instance" "schema_registry" {
  identifier     = "schema-registry-db"
  engine         = "postgres"
  engine_version = "15.4"
  instance_class = "db.t3.micro"  # 1 vCPU, 1 GB RAM

  allocated_storage     = 20
  storage_type          = "gp3"
  storage_encrypted     = true
  kms_key_id            = aws_kms_key.msk.arn

  db_name  = "schemaregistry"
  username = "schemaregistry"
  password = random_password.schema_registry_db.result

  vpc_security_group_ids = [aws_security_group.schema_registry_db.id]
  db_subnet_group_name   = aws_db_subnet_group.schema_registry.name

  backup_retention_period = 7
  backup_window           = "03:00-04:00"
  maintenance_window      = "Mon:04:00-Mon:05:00"

  skip_final_snapshot = true  # Change to false for production

  tags = {
    Name = "schema-registry-db"
  }
}

resource "random_password" "schema_registry_db" {
  length  = 32
  special = true
}

resource "aws_db_subnet_group" "schema_registry" {
  name       = "schema-registry-subnet-group"
  subnet_ids = data.aws_subnets.private.ids

  tags = {
    Name = "schema-registry-db-subnet-group"
  }
}

# Security group for RDS
resource "aws_security_group" "schema_registry_db" {
  name_description = "schema-registry-db"
  description = "Security group for Schema Registry RDS"
  vpc_id      = data.aws_vpc.main.id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.schema_registry_ecs.id]
    description     = "PostgreSQL from Schema Registry ECS"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "schema-registry-db-sg"
  }
}

# Security group for Schema Registry ECS tasks
resource "aws_security_group" "schema_registry_ecs" {
  name_description = "schema-registry-ecs"
  description = "Security group for Schema Registry ECS tasks"
  vpc_id      = data.aws_vpc.main.id

  ingress {
    from_port       = 8081
    to_port         = 8081
    protocol        = "tcp"
    security_groups = [aws_security_group.schema_registry_alb.id]
    description     = "Schema Registry API from ALB"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "schema-registry-ecs-sg"
  }
}

# Application Load Balancer (internal)
resource "aws_lb" "schema_registry" {
  name               = "schema-registry-alb"
  internal           = true
  load_balancer_type = "application"
  security_groups    = [aws_security_group.schema_registry_alb.id]
  subnets            = data.aws_subnets.private.ids

  tags = {
    Name = "schema-registry-alb"
  }
}

resource "aws_security_group" "schema_registry_alb" {
  name_description = "schema-registry-alb"
  description = "Security group for Schema Registry ALB"
  vpc_id      = data.aws_vpc.main.id

  ingress {
    from_port   = 8081
    to_port     = 8081
    protocol    = "tcp"
    cidr_blocks = [data.aws_vpc.main.cidr_block]
    description = "Schema Registry API from VPC"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "schema-registry-alb-sg"
  }
}

resource "aws_lb_target_group" "schema_registry" {
  name        = "schema-registry-tg"
  port        = 8081
  protocol    = "HTTP"
  vpc_id      = data.aws_vpc.main.id
  target_type = "ip"

  health_check {
    enabled             = true
    path                = "/subjects"
    port                = 8081
    protocol            = "HTTP"
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 30
  }

  tags = {
    Name = "schema-registry-tg"
  }
}

resource "aws_lb_listener" "schema_registry" {
  load_balancer_arn = aws_lb.schema_registry.arn
  port              = 8081
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.schema_registry.arn
  }
}

# ECS cluster
resource "aws_ecs_cluster" "streaming" {
  name = "streaming-cluster"

  tags = {
    Name = "streaming-ecs-cluster"
  }
}

# ECS task definition for Schema Registry
resource "aws_ecs_task_definition" "schema_registry" {
  family                   = "schema-registry"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = "512"   # 0.5 vCPU
  memory                   = "1024"  # 1 GB
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn            = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([
    {
      name  = "schema-registry"
      image = "confluentinc/cp-schema-registry:7.5.0"

      portMappings = [
        {
          containerPort = 8081
          protocol      = "tcp"
        }
      ]

      environment = [
        {
          name  = "SCHEMA_REGISTRY_HOST_NAME"
          value = "schema-registry"
        },
        {
          name  = "SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS"
          value = aws_msk_cluster.this.bootstrap_brokers_tls
        },
        {
          name  = "SCHEMA_REGISTRY_KAFKASTORE_SECURITY_PROTOCOL"
          value = "SSL"
        },
        {
          name  = "SCHEMA_REGISTRY_LISTENERS"
          value = "http://0.0.0.0:8081"
        }
      ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.schema_registry.name
          "awslogs-region"        = "eu-west-2"
          "awslogs-stream-prefix" = "schema-registry"
        }
      }
    }
  ])

  tags = {
    Name = "schema-registry-task"
  }
}

resource "aws_cloudwatch_log_group" "schema_registry" {
  name              = "/ecs/schema-registry"
  retention_in_days = 7

  tags = {
    Name = "schema-registry-logs"
  }
}

# ECS service
resource "aws_ecs_service" "schema_registry" {
  name            = "schema-registry"
  cluster         = aws_ecs_cluster.streaming.id
  task_definition = aws_ecs_task_definition.schema_registry.arn
  desired_count   = 1
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = data.aws_subnets.private.ids
    security_groups  = [aws_security_group.schema_registry_ecs.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.schema_registry.arn
    container_name   = "schema-registry"
    container_port   = 8081
  }

  depends_on = [aws_lb_listener.schema_registry]

  tags = {
    Name = "schema-registry-service"
  }
}

# IAM roles for ECS
resource "aws_iam_role" "ecs_execution" {
  name = "schema-registry-ecs-execution-role"

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

  tags = {
    Name = "schema-registry-ecs-execution-role"
  }
}

resource "aws_iam_role_policy_attachment" "ecs_execution" {
  role       = aws_iam_role.ecs_execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

resource "aws_iam_role" "ecs_task" {
  name = "schema-registry-ecs-task-role"

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

  tags = {
    Name = "schema-registry-ecs-task-role"
  }
}

# Output Schema Registry URL
output "schema_registry_url" {
  value       = "http://${aws_lb.schema_registry.dns_name}:8081"
  description = "Schema Registry URL (internal)"
}
```

### Deploy Schema Registry

```sh
terraform plan -out=tfplan
terraform apply tfplan
```

**Expected output:**
```
aws_ecs_service.schema_registry: Creating...
aws_ecs_service.schema_registry: Creation complete after 2m30s

Outputs:
schema_registry_url = "http://schema-registry-alb-123456789.eu-west-2.elb.amazonaws.com:8081"
```

### Test Schema Registry

```sh
# Get Schema Registry URL
SCHEMA_REGISTRY_URL=$(terraform output -raw schema_registry_url)

# Test health
curl $SCHEMA_REGISTRY_URL/subjects

# Expected: [] (empty list, no schemas registered yet)
```

## MSK Connect Deployment

Deploy Kafka Connect workers using MSK Connect (AWS managed).

### Why MSK Connect?

**MSK Connect** is AWS's managed Kafka Connect service - it runs Kafka Connect workers for you.

**Pros:**
- AWS manages worker scaling, failures, upgrades
- Integrated with CloudWatch for logging
- Pay per worker hour (no idle cost if auto-scaled)

**Cons:**
- Less flexible than self-hosted (limited connector plugins)
- Higher cost than ECS Fargate for always-on workloads

### Create Snowflake Sink Connector Plugin

MSK Connect requires custom connector plugins uploaded to S3.

```sh
# Download Snowflake Kafka Connector
wget https://repo1.maven.org/maven2/com/snowflake/snowflake-kafka-connector/1.9.3/snowflake-kafka-connector-1.9.3.jar

# Create ZIP for MSK Connect
mkdir snowflake-connector
cp snowflake-kafka-connector-1.9.3.jar snowflake-connector/
zip -r snowflake-connector.zip snowflake-connector/

# Upload to S3
aws s3 cp snowflake-connector.zip s3://your-kafka-connect-plugins-bucket/snowflake-connector.zip
```

### Terraform Configuration for MSK Connect

Add to `~/src/terraform/aws/msk/msk_connect.tf`:

```hcl
# S3 bucket for connector plugins
resource "aws_s3_bucket" "connector_plugins" {
  bucket = "your-account-kafka-connect-plugins"

  tags = {
    Name = "kafka-connect-plugins"
  }
}

# Custom plugin for Snowflake Sink
resource "aws_mskconnect_custom_plugin" "snowflake_sink" {
  name         = "snowflake-sink-connector"
  content_type = "ZIP"

  location {
    s3 {
      bucket_arn = aws_s3_bucket.connector_plugins.arn
      file_key   = "snowflake-connector.zip"
    }
  }

  description = "Snowflake Kafka Connector for MSK Connect"
}

# MSK Connect worker configuration
resource "aws_mskconnect_worker_configuration" "this" {
  name = "data-platform-worker-config"

  properties_file_content = <<EOF
key.converter=org.apache.kafka.connect.storage.StringConverter
value.converter=io.confluent.connect.avro.AvroConverter
value.converter.schema.registry.url=${aws_lb.schema_registry.dns_name}:8081

# Offset storage
offset.storage.topic=connect-offsets
offset.storage.replication.factor=3
offset.storage.partitions=3

# Config storage
config.storage.topic=connect-configs
config.storage.replication.factor=3

# Status storage
status.storage.topic=connect-status
status.storage.replication.factor=3
status.storage.partitions=3
EOF
}

# IAM role for MSK Connect
resource "aws_iam_role" "msk_connect" {
  name = "msk-connect-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "kafkaconnect.amazonaws.com"
        }
      }
    ]
  })

  tags = {
    Name = "msk-connect-role"
  }
}

resource "aws_iam_role_policy" "msk_connect_msk_access" {
  name = "msk-access"
  role = aws_iam_role.msk_connect.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "kafka-cluster:Connect",
          "kafka-cluster:AlterCluster",
          "kafka-cluster:DescribeCluster"
        ]
        Resource = aws_msk_cluster.this.arn
      },
      {
        Effect = "Allow"
        Action = [
          "kafka-cluster:*Topic*",
          "kafka-cluster:WriteData",
          "kafka-cluster:ReadData"
        ]
        Resource = "${aws_msk_cluster.this.arn}/*"
      },
      {
        Effect = "Allow"
        Action = [
          "kafka-cluster:AlterGroup",
          "kafka-cluster:DescribeGroup"
        ]
        Resource = "${aws_msk_cluster.this.arn}/*"
      }
    ]
  })
}

resource "aws_iam_role_policy" "msk_connect_secrets_access" {
  name = "secrets-access"
  role = aws_iam_role.msk_connect.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue"
        ]
        Resource = "arn:aws:secretsmanager:eu-west-2:*:secret:snowflake/svc-kafka-connector-*"
      }
    ]
  })
}

# Security group for MSK Connect workers
resource "aws_security_group" "msk_connect" {
  name_description = "msk-connect"
  description = "Security group for MSK Connect workers"
  vpc_id      = data.aws_vpc.main.id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "msk-connect-sg"
  }
}

# MSK Connect connector (Snowflake Sink)
resource "aws_mskconnect_connector" "snowflake_sink" {
  name = "snowflake-sink-order-events"

  kafkaconnect_version = "2.7.1"

  capacity {
    autoscaling {
      mcu_count        = 1
      min_worker_count = 1
      max_worker_count = 2

      scale_in_policy {
        cpu_utilization_percentage = 20
      }

      scale_out_policy {
        cpu_utilization_percentage = 80
      }
    }
  }

  connector_configuration = {
    "connector.class" = "com.snowflake.kafka.connector.SnowflakeSinkConnector"
    "tasks.max"       = "1"

    # Kafka topic
    "topics" = "order-events"

    # Snowflake connection (from Secrets Manager)
    "snowflake.url.name"      = "your-account.eu-west-2.aws.snowflakecomputing.com"
    "snowflake.user.name"     = "SVC_KAFKA_CONNECTOR"
    "snowflake.private.key"   = "${aws_secretsmanager_secret.snowflake_credentials.arn}:private_key::"
    "snowflake.database.name" = "STREAMING"
    "snowflake.schema.name"   = "PUBLIC"
    "snowflake.role.name"     = "SVC_KAFKA_CONNECTOR"

    # Buffer settings
    "buffer.count.records" = "10000"
    "buffer.flush.time"    = "60"
    "buffer.size.bytes"    = "5000000"

    # Table configuration
    "snowflake.topic2table.map" = "order-events:ORDER_EVENTS"

    # Schema Registry
    "value.converter"                       = "io.confluent.connect.avro.AvroConverter"
    "value.converter.schema.registry.url"   = "http://${aws_lb.schema_registry.dns_name}:8081"
    "key.converter"                         = "org.apache.kafka.connect.storage.StringConverter"
  }

  kafka_cluster {
    apache_kafka_cluster {
      bootstrap_servers = aws_msk_cluster.this.bootstrap_brokers_tls

      vpc {
        security_groups = [aws_security_group.msk_connect.id]
        subnets         = data.aws_subnets.private.ids
      }
    }
  }

  kafka_cluster_client_authentication {
    authentication_type = "NONE"  # TLS only (no SASL)
  }

  kafka_cluster_encryption_in_transit {
    encryption_type = "TLS"
  }

  plugin {
    custom_plugin {
      arn      = aws_mskconnect_custom_plugin.snowflake_sink.arn
      revision = aws_mskconnect_custom_plugin.snowflake_sink.latest_revision
    }
  }

  service_execution_role_arn = aws_iam_role.msk_connect.arn

  worker_configuration {
    arn      = aws_mskconnect_worker_configuration.this.arn
    revision = aws_mskconnect_worker_configuration.this.latest_revision
  }

  log_delivery {
    worker_log_delivery {
      cloudwatch_logs {
        enabled   = true
        log_group = aws_cloudwatch_log_group.msk_connect.name
      }
    }
  }
}

resource "aws_cloudwatch_log_group" "msk_connect" {
  name              = "/aws/mskconnect/snowflake-sink"
  retention_in_days = 7

  tags = {
    Name = "msk-connect-logs"
  }
}
```

### Deploy MSK Connect

```sh
terraform plan -out=tfplan
terraform apply tfplan

# Check connector status (takes 5-10 minutes to start)
aws kafkaconnect describe-connector \
    --connector-arn $(terraform output -raw snowflake_sink_connector_arn) \
    --region eu-west-2
```

## Cost Comparison

### MSK Total Cost (5 GB/day workload)

| Component | Monthly Cost | Notes |
|-----------|-------------|-------|
| **MSK cluster** (3 × kafka.m5.large) | $252 | 24/7 brokers |
| **EBS storage** (3 × 100 GB) | $24 | gp3 volumes |
| **Schema Registry ECS** (0.5 vCPU) | $15 | Fargate 24/7 |
| **Schema Registry RDS** (db.t3.micro) | $20 | PostgreSQL |
| **Schema Registry ALB** | $15 | Internal load balancer |
| **MSK Connect** (1 MCU × 720 hours) | $86 | $0.12/MCU-hour |
| **CloudWatch Logs** | $10 | Broker + connector logs |
| **Data transfer** | $5 | Minimal within VPC |
| **KMS** | $3 | Encryption keys |
| **Setup time** (one-time) | - | 40 hours @ $60 = $2,400 |
| **Ops time** (monthly) | - | 6-12 hours @ $60 = $360-720 |
| **Total infrastructure** | **$430/month** | |
| **Total with ops** | **$790-1,150/month** | |

### Confluent Cloud Total Cost (5 GB/day workload)

| Component | Monthly Cost | Notes |
|-----------|-------------|-------|
| **Confluent Cloud Basic** | $150 | 1 CKU cluster |
| **Data ingress** (1 GB/day) | $2.50 | $0.08/GB |
| **Data egress** (1 GB/day) | $2.50 | Connector reads |
| **Storage** (30 GB) | $2.50 | 7-day retention |
| **Snowflake compute** | $35 | INGEST_WH (XSMALL) |
| **Snowflake storage** | $5 | STREAMING database |
| **AWS Secrets Manager** | $1.50 | Credentials |
| **Setup time** (one-time) | - | 8 hours @ $60 = $480 |
| **Ops time** (monthly) | - | 1-2 hours @ $60 = $60-120 |
| **Total infrastructure** | **$199/month** | |
| **Total with ops** | **$259-319/month** | |

### Winner: Confluent Cloud (72% cheaper)

| Metric | MSK | Confluent Cloud | Difference |
|--------|-----|----------------|-----------|
| **Infrastructure cost** | $430/month | $199/month | +116% |
| **Ops time** | 6-12 hours/month | 1-2 hours/month | +400% |
| **Total cost** | $790-1,150/month | $259-319/month | +205-260% |
| **Setup time** | 40 hours | 8 hours | +400% |

## When MSK Makes Sense

### Scale Tipping Point

MSK becomes cost-competitive at **very high scale:**

| Daily Volume | Confluent Cloud Total | MSK Total | Cheaper Option |
|--------------|---------------------|-----------|----------------|
| **5 GB** | $259/month | $790/month | Confluent (67% cheaper) |
| **50 GB** | $374/month | $950/month | Confluent (61% cheaper) |
| **200 GB** | $900/month | $1,400/month | Confluent (36% cheaper) |
| **500 GB** | $2,200/month | $2,400/month | Similar |
| **1 TB** | $4,000/month | $3,200/month | MSK (20% cheaper) |

**Break-even:** ~500 GB/day (~15 TB/month)

### Non-Cost Reasons

MSK makes sense when:

✅ **Existing MSK infrastructure** - Team already running MSK for other use cases
✅ **AWS commitment** - Enterprise discount programme makes MSK significantly cheaper
✅ **Data residency** - Must keep data in specific AWS account/region
✅ **Compliance** - Custom security controls not available in Confluent Cloud
✅ **Custom Kafka configuration** - Need non-standard broker settings

### Migration Path

**Phase 1:** Start with Confluent Cloud (months 0-12)
- Learn Kafka concepts without ops burden
- Build data pipelines and producers
- Validate business value

**Phase 2:** Evaluate MSK (month 12+)
- If throughput > 200 GB/day
- AND team has DevOps capacity
- AND cost savings justify migration effort

**Migration effort:** 40-80 hours (planning, testing, cutover)

## Summary

You've learned when and how to self-host Kafka with MSK:

- [x] **MSK provisioned cluster** - 3-broker setup with TLS encryption
- [x] **Schema Registry on ECS** - Self-hosted Confluent Schema Registry
- [x] **MSK Connect** - Managed Kafka Connect for Snowflake Sink
- [x] **VPC networking** - Security groups and private subnets
- [x] **Cost comparison** - $790-1,150/month vs $259-319/month for Confluent Cloud
- [x] **Break-even analysis** - MSK competitive at > 500 GB/day

**Key insight:** MSK has lower infrastructure cost but **much higher** total cost due to operational burden. Start with Confluent Cloud for 99% of use cases.

## What's Next

Explore stream processing to aggregate and enrich events in real-time.

Continue to [Stream Processing](10-stream-processing.md) →
