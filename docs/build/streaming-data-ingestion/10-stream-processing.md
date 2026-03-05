# Stream Processing (Optional)

!!! info "Optional Advanced Topic"
    This page is **optional**. Most teams can process streaming data using dbt after it lands in Snowflake. Only add stream processing if you need sub-second transformations, real-time aggregations, or complex event processing that can't wait for dbt runs.

On this page, you will:

- [x] Understand when to process in-stream vs in-warehouse
- [x] Deploy ksqlDB for SQL-based stream processing
- [x] Deploy Apache Flink on ECS for complex event processing
- [x] Create real-time aggregations and windowed calculations
- [x] Implement sessionisation and pattern matching
- [x] Evaluate cost implications of adding stream processing
- [x] Choose the right tool for your use case

## Overview

**Stream processing** transforms data in-flight before it lands in Snowflake. This enables real-time use cases that can't wait for batch dbt transformations.

**Trade-off:** Added complexity and cost vs real-time insights.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    STREAM PROCESSING OPTIONS                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Without Stream Processing (Recommended Default)                        │
│  ───────────────────────────────────────────────                        │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐   │
│  │ Producer │─────▶│  Kafka   │─────▶│ Snowflake│─────▶│   dbt    │   │
│  │          │ <1s  │  Topic   │ <5s  │  Table   │ 5min │ Transform│   │
│  └──────────┘      └──────────┘      └──────────┘      └──────────┘   │
│                                                                         │
│  Total latency: ~5 minutes (sufficient for most use cases)             │
│  Cost: $200/month (Confluent Cloud + Snowflake)                        │
│                                                                         │
│  ─────────────────────────────────────────────────────────────────────  │
│                                                                         │
│  With Stream Processing (Real-Time Aggregations)                        │
│  ────────────────────────────────────────────────                       │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐   │
│  │ Producer │─────▶│  Kafka   │─────▶│ ksqlDB / │─────▶│  Kafka   │   │
│  │          │ <1s  │  Topic   │ <1s  │  Flink   │ <1s  │  Topic   │   │
│  └──────────┘      └──────────┘      └──────────┘      └──────────┘   │
│                                            │                   │         │
│                                            ▼                   ▼         │
│                                       ┌──────────┐      ┌──────────┐   │
│                                       │ Snowflake│      │ Snowflake│   │
│                                       │ (raw)    │      │(processed)│  │
│                                       └──────────┘      └──────────┘   │
│                                                                         │
│  Total latency: ~3 seconds (real-time aggregations)                    │
│  Cost: $400-600/month (+$200-400 for processing layer)                 │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## When to Use Stream Processing

### Decision Tree

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    STREAM PROCESSING DECISION TREE                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Do you need results within seconds (not minutes)?                      │
│  ├─ NO  → Use dbt in Snowflake (sufficient for most use cases)         │
│  └─ YES → Continue                                                      │
│                                                                         │
│  Can you wait for dbt to run every 5 minutes?                           │
│  ├─ YES → Use dbt (simpler, cheaper)                                   │
│  └─ NO  → Continue                                                      │
│                                                                         │
│  Is the logic stateless (per-event filtering/enrichment)?               │
│  ├─ YES → Use Kafka Streams in producer app (simplest)                 │
│  └─ NO  → Continue                                                      │
│                                                                         │
│  Is the logic SQL-based (aggregations, joins, windowing)?               │
│  ├─ YES → Use ksqlDB                                                    │
│  └─ NO  → Continue                                                      │
│                                                                         │
│  Is the logic complex (CEP, ML, custom state management)?               │
│  ├─ YES → Use Apache Flink                                             │
│  └─ NO  → Reconsider if you need stream processing                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Use Cases by Latency Requirement

| Use Case | Acceptable Latency | Recommended Approach |
|----------|-------------------|---------------------|
| **Daily dashboards** | Hours | dbt batch (scheduled) |
| **Hourly reports** | 5-15 minutes | dbt micro-batching (every 5 min) |
| **Real-time dashboards** | 30-60 seconds | dbt + Snowflake auto-refresh |
| **Operational alerts** | < 10 seconds | ksqlDB or Flink |
| **Fraud detection** | < 1 second | Flink + in-memory state |
| **High-frequency trading** | < 100ms | Custom in-process (not Kafka) |

### Good Reasons to Add Stream Processing

✅ **Real-time alerting** - Detect anomalies and trigger actions within seconds
✅ **Live aggregations** - Website visitor counts, active sessions, running totals
✅ **Complex event processing** - Pattern detection across multiple event types
✅ **Sessionisation** - Group events by user sessions with timeouts
✅ **Real-time enrichment** - Join streaming events with reference data
✅ **Data quality filtering** - Drop invalid events before storage

### Bad Reasons

❌ **"We want real-time"** - without specific business value measured in $
❌ **Over-engineering** - Adding complexity before validating need
❌ **Resume building** - Using Flink because it's trendy, not because it's needed
❌ **Avoiding dbt** - Stream processing is harder to maintain than dbt

## ksqlDB for SQL-Based Processing

### What is ksqlDB?

**ksqlDB** is a SQL streaming database built on Kafka. It lets you:

- Write SQL queries that run continuously on Kafka topics
- Create materialized views (aggregations updated in real-time)
- Join streams with tables
- Window data by time (tumbling, hopping, session windows)

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ksqlDB ARCHITECTURE                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Kafka Topics                                                           │
│  ────────────                                                           │
│  ┌────────────────────┐          ┌────────────────────┐                │
│  │ order-events       │          │ customer-events    │                │
│  │ (raw orders)       │          │ (raw customers)    │                │
│  └────────────────────┘          └────────────────────┘                │
│           │                               │                             │
│           └───────────────┬───────────────┘                             │
│                           ▼                                             │
│  ksqlDB Server (ECS Fargate)                                            │
│  ────────────────────────────                                           │
│  ┌─────────────────────────────────────────────────────┐               │
│  │ CREATE STREAM orders ...                            │               │
│  │ CREATE TABLE order_totals_by_customer AS            │               │
│  │   SELECT customer_id,                               │               │
│  │          COUNT(*) as order_count,                   │               │
│  │          SUM(order_total) as total_spent            │               │
│  │   FROM orders                                       │               │
│  │   WINDOW TUMBLING (SIZE 1 HOUR)                     │               │
│  │   GROUP BY customer_id;                             │               │
│  └─────────────────────────────────────────────────────┘               │
│                           │                                             │
│                           ▼                                             │
│  Output Topic                                                           │
│  ────────────                                                           │
│  ┌────────────────────┐                                                 │
│  │ order_totals       │────▶ Snowflake Sink ────▶ Snowflake            │
│  │ (aggregated)       │                                                 │
│  └────────────────────┘                                                 │
│                                                                         │
│  Cost: $30-60/month (ECS Fargate, 1-2 vCPU)                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Deployment on ECS Fargate

#### Terraform Configuration

Create `repositories/terraform/aws/ksqldb/main.tf`:

```hcl
# ECS task definition for ksqlDB
resource "aws_ecs_task_definition" "ksqldb" {
  family                   = "ksqldb"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = "1024"  # 1 vCPU
  memory                   = "2048"  # 2 GB
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn            = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([
    {
      name  = "ksqldb-server"
      image = "confluentinc/ksqldb-server:0.29.0"

      portMappings = [
        {
          containerPort = 8088
          protocol      = "tcp"
        }
      ]

      environment = [
        {
          name  = "KSQL_BOOTSTRAP_SERVERS"
          value = var.kafka_bootstrap_servers
        },
        {
          name  = "KSQL_LISTENERS"
          value = "http://0.0.0.0:8088"
        },
        {
          name  = "KSQL_KSQL_SERVICE_ID"
          value = "ksqldb-cluster"
        },
        {
          name  = "KSQL_KSQL_SCHEMA_REGISTRY_URL"
          value = var.schema_registry_url
        },
        {
          name  = "KSQL_KSQL_LOGGING_PROCESSING_STREAM_AUTO_CREATE"
          value = "true"
        },
        {
          name  = "KSQL_KSQL_LOGGING_PROCESSING_TOPIC_AUTO_CREATE"
          value = "true"
        }
      ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.ksqldb.name
          "awslogs-region"        = "eu-west-2"
          "awslogs-stream-prefix" = "ksqldb"
        }
      }
    }
  ])

  tags = {
    Name = "ksqldb-task"
  }
}

resource "aws_cloudwatch_log_group" "ksqldb" {
  name              = "/ecs/ksqldb"
  retention_in_days = 7

  tags = {
    Name = "ksqldb-logs"
  }
}

# ECS service
resource "aws_ecs_service" "ksqldb" {
  name            = "ksqldb"
  cluster         = var.ecs_cluster_id
  task_definition = aws_ecs_task_definition.ksqldb.arn
  desired_count   = 1
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = var.private_subnet_ids
    security_groups  = [aws_security_group.ksqldb.id]
    assign_public_ip = false
  }

  tags = {
    Name = "ksqldb-service"
  }
}

# Security group
resource "aws_security_group" "ksqldb" {
  name_description = "ksqldb"
  description = "Security group for ksqlDB server"
  vpc_id      = var.vpc_id

  ingress {
    from_port   = 8088
    to_port     = 8088
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr_block]
    description = "ksqlDB API from VPC"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "ksqldb-sg"
  }
}

# Variables
variable "kafka_bootstrap_servers" {
  description = "Kafka bootstrap servers"
  type        = string
}

variable "schema_registry_url" {
  description = "Schema Registry URL"
  type        = string
}

variable "ecs_cluster_id" {
  description = "ECS cluster ID"
  type        = string
}

variable "vpc_id" {
  description = "VPC ID"
  type        = string
}

variable "vpc_cidr_block" {
  description = "VPC CIDR block"
  type        = string
}

variable "private_subnet_ids" {
  description = "Private subnet IDs"
  type        = list(string)
}

# Outputs
output "ksqldb_endpoint" {
  value       = "http://${aws_ecs_service.ksqldb.name}:8088"
  description = "ksqlDB server endpoint"
}
```

#### Deploy ksqlDB

```sh
cd repositories/terraform/aws/ksqldb

terraform init
terraform plan -out=tfplan
terraform apply tfplan
```

### Example: Real-Time Order Aggregations

Create a ksqlDB query to aggregate orders by customer in real-time.

#### Connect to ksqlDB

```sh
# Port-forward to ksqlDB server
aws ecs execute-command \
    --cluster streaming-cluster \
    --task <task-id> \
    --container ksqldb-server \
    --interactive \
    --command "/bin/bash"

# Inside container, start ksqlDB CLI
ksql http://localhost:8088
```

#### Create Stream from Topic

```sql
-- Define stream from order-events topic
CREATE STREAM order_events (
    order_id BIGINT,
    customer_id STRING,
    order_status STRING,
    order_total DOUBLE,
    created_at BIGINT
) WITH (
    KAFKA_TOPIC='order-events',
    VALUE_FORMAT='AVRO'
);
```

#### Create Aggregation Table

```sql
-- Real-time aggregation: total orders and spend per customer
CREATE TABLE customer_order_totals AS
    SELECT
        customer_id,
        COUNT(*) AS order_count,
        SUM(order_total) AS total_spent,
        MAX(created_at) AS last_order_time
    FROM order_events
    WINDOW TUMBLING (SIZE 1 HOUR)
    GROUP BY customer_id
    EMIT CHANGES;
```

This query:
- **Continuously processes** new order events
- **Aggregates** by customer in 1-hour windows
- **Writes results** to a new Kafka topic (`customer_order_totals`)
- **Updates** as new events arrive

#### Query the Table

```sql
-- Query current aggregations
SELECT * FROM customer_order_totals WHERE customer_id = 'cust_123';

-- Result (updated in real-time):
-- customer_id | order_count | total_spent | last_order_time
-- cust_123    | 15          | 1,234.56    | 1709643845000
```

#### Sink to Snowflake

Add a Snowflake Sink Connector for the `customer_order_totals` topic - it lands in Snowflake within seconds.

### Example: Sessionisation

Group events by user sessions with 30-minute inactivity timeout.

```sql
-- Define clickstream events
CREATE STREAM clickstream (
    user_id STRING,
    page_url STRING,
    event_time BIGINT
) WITH (
    KAFKA_TOPIC='clickstream-events',
    VALUE_FORMAT='AVRO'
);

-- Sessionise: group events into sessions (30-min timeout)
CREATE TABLE user_sessions AS
    SELECT
        user_id,
        COUNT(*) AS event_count,
        COLLECT_LIST(page_url) AS pages_visited,
        MIN(event_time) AS session_start,
        MAX(event_time) AS session_end
    FROM clickstream
    WINDOW SESSION (30 MINUTES)
    GROUP BY user_id
    EMIT CHANGES;
```

**Use case:** Real-time analytics on active user sessions for dashboards.

### ksqlDB Cost

| Component | Monthly Cost | Notes |
|-----------|-------------|-------|
| **ECS Fargate** (1 vCPU, 2 GB) | $30 | 24/7 |
| **CloudWatch Logs** | $5 | Processing logs |
| **Additional Kafka storage** | $10 | Output topics |
| **Ops time** | 2-4 hours/month | Query tuning, monitoring |
| **Total** | **$45 + $120-240 ops** = **$165-285/month** | |

## Apache Flink for Complex Event Processing

### What is Apache Flink?

**Apache Flink** is a distributed stream processing framework for complex, stateful computations. Use it for:

- **Complex event processing (CEP)** - Pattern detection across multiple events
- **Stateful processing** - Maintaining large state (GBs) across events
- **Custom business logic** - Python/Java code for non-SQL transformations
- **Low-latency ML inference** - Real-time model scoring
- **Exactly-once guarantees** - Critical financial transactions

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    FLINK ON ECS                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Flink Cluster (ECS)                                                    │
│  ───────────────────                                                    │
│  ┌────────────────────────────────────────────────────────────┐        │
│  │ JobManager (1 task, 1 vCPU, 2 GB)                         │        │
│  │ • Coordinates job execution                                │        │
│  │ • Manages checkpoints                                      │        │
│  └────────────────────────────────────────────────────────────┘        │
│                           │                                             │
│                           ▼                                             │
│  ┌────────────────────────────────────────────────────────────┐        │
│  │ TaskManagers (2 tasks, 2 vCPU each, 4 GB each)            │        │
│  │ • Execute streaming jobs                                   │        │
│  │ • Maintain state                                           │        │
│  │ • Parallel processing                                      │        │
│  └────────────────────────────────────────────────────────────┘        │
│                           │                                             │
│                           ▼                                             │
│  S3 (Checkpoints)                                                       │
│  ────────────────                                                       │
│  ┌────────────────────────────────────────────────────────────┐        │
│  │ State snapshots every 5 minutes                            │        │
│  │ • Enables recovery from failures                           │        │
│  │ • Exactly-once processing guarantees                       │        │
│  └────────────────────────────────────────────────────────────┘        │
│                                                                         │
│  Cost: $150-250/month (JobManager + 2 TaskManagers)                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Deployment on ECS Fargate

#### Terraform Configuration

Create `repositories/terraform/aws/flink/main.tf`:

```hcl
# S3 bucket for Flink checkpoints
resource "aws_s3_bucket" "flink_checkpoints" {
  bucket = "your-account-flink-checkpoints"

  tags = {
    Name = "flink-checkpoints"
  }
}

resource "aws_s3_bucket_versioning" "flink_checkpoints" {
  bucket = aws_s3_bucket.flink_checkpoints.id

  versioning_configuration {
    status = "Enabled"
  }
}

# IAM role for Flink tasks
resource "aws_iam_role" "flink_task" {
  name = "flink-task-role"

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
    Name = "flink-task-role"
  }
}

resource "aws_iam_role_policy" "flink_s3_access" {
  name = "s3-checkpoint-access"
  role = aws_iam_role.flink_task.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject",
          "s3:ListBucket"
        ]
        Resource = [
          aws_s3_bucket.flink_checkpoints.arn,
          "${aws_s3_bucket.flink_checkpoints.arn}/*"
        ]
      }
    ]
  })
}

# ECS task definition for Flink JobManager
resource "aws_ecs_task_definition" "flink_jobmanager" {
  family                   = "flink-jobmanager"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = "1024"  # 1 vCPU
  memory                   = "2048"  # 2 GB
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn            = aws_iam_role.flink_task.arn

  container_definitions = jsonencode([
    {
      name  = "jobmanager"
      image = "flink:1.18-scala_2.12-java11"
      command = ["jobmanager"]

      portMappings = [
        {
          containerPort = 8081
          protocol      = "tcp"
        },
        {
          containerPort = 6123
          protocol      = "tcp"
        }
      ]

      environment = [
        {
          name  = "JOB_MANAGER_RPC_ADDRESS"
          value = "jobmanager"
        },
        {
          name  = "FLINK_PROPERTIES"
          value = <<EOF
jobmanager.rpc.address: jobmanager
taskmanager.numberOfTaskSlots: 2
parallelism.default: 2
state.backend: rocksdb
state.checkpoints.dir: s3://${aws_s3_bucket.flink_checkpoints.bucket}/checkpoints
state.savepoints.dir: s3://${aws_s3_bucket.flink_checkpoints.bucket}/savepoints
execution.checkpointing.interval: 300000
execution.checkpointing.mode: EXACTLY_ONCE
EOF
        }
      ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.flink.name
          "awslogs-region"        = "eu-west-2"
          "awslogs-stream-prefix" = "jobmanager"
        }
      }
    }
  ])

  tags = {
    Name = "flink-jobmanager-task"
  }
}

# ECS task definition for Flink TaskManager
resource "aws_ecs_task_definition" "flink_taskmanager" {
  family                   = "flink-taskmanager"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = "2048"  # 2 vCPU
  memory                   = "4096"  # 4 GB
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn            = aws_iam_role.flink_task.arn

  container_definitions = jsonencode([
    {
      name  = "taskmanager"
      image = "flink:1.18-scala_2.12-java11"
      command = ["taskmanager"]

      environment = [
        {
          name  = "JOB_MANAGER_RPC_ADDRESS"
          value = "jobmanager.internal"  # Use service discovery
        },
        {
          name  = "FLINK_PROPERTIES"
          value = <<EOF
jobmanager.rpc.address: jobmanager.internal
taskmanager.numberOfTaskSlots: 2
taskmanager.memory.process.size: 3072m
EOF
        }
      ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.flink.name
          "awslogs-region"        = "eu-west-2"
          "awslogs-stream-prefix" = "taskmanager"
        }
      }
    }
  ])

  tags = {
    Name = "flink-taskmanager-task"
  }
}

resource "aws_cloudwatch_log_group" "flink" {
  name              = "/ecs/flink"
  retention_in_days = 7

  tags = {
    Name = "flink-logs"
  }
}

# ECS services
resource "aws_ecs_service" "flink_jobmanager" {
  name            = "flink-jobmanager"
  cluster         = var.ecs_cluster_id
  task_definition = aws_ecs_task_definition.flink_jobmanager.arn
  desired_count   = 1
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = var.private_subnet_ids
    security_groups  = [aws_security_group.flink.id]
    assign_public_ip = false
  }

  service_registries {
    registry_arn = aws_service_discovery_service.jobmanager.arn
  }

  tags = {
    Name = "flink-jobmanager-service"
  }
}

resource "aws_ecs_service" "flink_taskmanager" {
  name            = "flink-taskmanager"
  cluster         = var.ecs_cluster_id
  task_definition = aws_ecs_task_definition.flink_taskmanager.arn
  desired_count   = 2  # 2 TaskManagers for parallelism
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = var.private_subnet_ids
    security_groups  = [aws_security_group.flink.id]
    assign_public_ip = false
  }

  tags = {
    Name = "flink-taskmanager-service"
  }
}

# Service discovery for JobManager
resource "aws_service_discovery_private_dns_namespace" "flink" {
  name = "flink.internal"
  vpc  = var.vpc_id

  tags = {
    Name = "flink-service-discovery"
  }
}

resource "aws_service_discovery_service" "jobmanager" {
  name = "jobmanager"

  dns_config {
    namespace_id = aws_service_discovery_private_dns_namespace.flink.id

    dns_records {
      ttl  = 10
      type = "A"
    }
  }

  tags = {
    Name = "jobmanager-discovery"
  }
}

# Security group
resource "aws_security_group" "flink" {
  name_description = "flink"
  description = "Security group for Flink cluster"
  vpc_id      = var.vpc_id

  # JobManager RPC
  ingress {
    from_port = 6123
    to_port   = 6123
    protocol  = "tcp"
    self      = true
  }

  # JobManager Web UI
  ingress {
    from_port   = 8081
    to_port     = 8081
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr_block]
  }

  # TaskManager data exchange
  ingress {
    from_port = 6121
    to_port   = 6125
    protocol  = "tcp"
    self      = true
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "flink-sg"
  }
}
```

#### Deploy Flink

```sh
cd repositories/terraform/aws/flink

terraform init
terraform plan -out=tfplan
terraform apply tfplan
```

### Example: Fraud Detection with Pattern Matching

Detect suspicious patterns: 3+ failed login attempts within 5 minutes from the same IP.

#### Flink Job (Python)

Create `fraud_detection_job.py`:

```python
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.datastream.connectors.kafka import FlinkKafkaConsumer, FlinkKafkaProducer
from pyflink.common.serialization import SimpleStringSchema
from pyflink.common.typeinfo import Types
from pyflink.datastream.functions import KeyedProcessFunction, RuntimeContext
from pyflink.datastream.state import ValueStateDescriptor
import json
from datetime import datetime, timedelta

env = StreamExecutionEnvironment.get_execution_environment()
env.set_parallelism(2)

# Kafka consumer (login events)
kafka_consumer = FlinkKafkaConsumer(
    topics='login-events',
    deserialization_schema=SimpleStringSchema(),
    properties={
        'bootstrap.servers': 'broker1:9092,broker2:9092',
        'group.id': 'fraud-detection'
    }
)

# Kafka producer (alerts)
kafka_producer = FlinkKafkaProducer(
    topic='fraud-alerts',
    serialization_schema=SimpleStringSchema(),
    producer_config={
        'bootstrap.servers': 'broker1:9092,broker2:9092'
    }
)

class FraudDetector(KeyedProcessFunction):
    def open(self, runtime_context: RuntimeContext):
        # State: track failed login count and timestamps
        self.failed_logins_state = runtime_context.get_state(
            ValueStateDescriptor("failed-logins", Types.LIST(Types.LONG()))
        )

    def process_element(self, value, ctx):
        event = json.loads(value)

        # Only process failed logins
        if event['status'] != 'FAILED':
            return

        current_time = ctx.timestamp()
        ip_address = event['ip_address']

        # Get failed login timestamps for this IP
        failed_logins = self.failed_logins_state.value() or []

        # Remove timestamps older than 5 minutes
        five_minutes_ago = current_time - (5 * 60 * 1000)
        recent_failures = [ts for ts in failed_logins if ts > five_minutes_ago]

        # Add current failure
        recent_failures.append(current_time)

        # Update state
        self.failed_logins_state.update(recent_failures)

        # Alert if 3+ failures in 5 minutes
        if len(recent_failures) >= 3:
            alert = {
                'alert_type': 'SUSPECTED_FRAUD',
                'ip_address': ip_address,
                'failure_count': len(recent_failures),
                'time_window': '5_minutes',
                'timestamp': current_time
            }
            yield json.dumps(alert)

# Build streaming pipeline
login_events = env.add_source(kafka_consumer)

fraud_alerts = (login_events
    .key_by(lambda event: json.loads(event)['ip_address'])
    .process(FraudDetector()))

fraud_alerts.add_sink(kafka_producer)

env.execute("Fraud Detection Job")
```

#### Package and Deploy

```sh
# Build Flink job JAR
python -m PyFlinkJob fraud_detection_job.py

# Upload to S3
aws s3 cp fraud_detection_job.jar s3://your-flink-jobs-bucket/

# Submit job to Flink cluster
flink run -d \
    -m jobmanager.internal:8081 \
    s3://your-flink-jobs-bucket/fraud_detection_job.jar
```

### Flink Cost

| Component | Monthly Cost | Notes |
|-----------|-------------|-------|
| **JobManager ECS** (1 vCPU, 2 GB) | $30 | 24/7 |
| **TaskManager ECS** (2 × 2 vCPU, 4 GB) | $120 | 24/7 |
| **S3 checkpoints** | $5 | State snapshots |
| **CloudWatch Logs** | $10 | Processing logs |
| **Development time** (one-time) | - | 40-80 hours @ $60 = $2,400-4,800 |
| **Ops time** (monthly) | 4-8 hours/month | Job monitoring, tuning |
| **Total** | **$165 + $240-480 ops** = **$405-645/month** | |

## Stream Processing vs dbt

### Latency Comparison

| Approach | Latency | Cost/Month | Complexity |
|----------|---------|-----------|-----------|
| **dbt hourly** | 30-60 min | $50 | Low |
| **dbt every 5 min** | 5-10 min | $100 | Low |
| **ksqlDB** | < 10 sec | $165-285 | Medium |
| **Flink** | < 1 sec | $405-645 | High |

### When to Use Each

| Requirement | Use dbt | Use ksqlDB | Use Flink |
|-------------|---------|-----------|-----------|
| **Batch reports** | ✅ | ❌ | ❌ |
| **Simple aggregations** | ✅ | ✅ | ❌ |
| **Real-time dashboards (< 1 min)** | ⚠️ | ✅ | ✅ |
| **Sessionisation** | ❌ | ✅ | ✅ |
| **Pattern detection** | ❌ | ⚠️ | ✅ |
| **ML inference** | ❌ | ❌ | ✅ |
| **Custom business logic** | ✅ | ⚠️ | ✅ |

**Recommendation:** Start with dbt. Add ksqlDB if you need < 10s latency for SQL transformations. Add Flink only for complex CEP or < 1s latency.

## Summary

You've learned when and how to add stream processing:

- [x] **Decision framework** - Most teams don't need stream processing (dbt is sufficient)
- [x] **ksqlDB** - SQL-based processing for real-time aggregations and joins ($165-285/month)
- [x] **Apache Flink** - Complex event processing and pattern detection ($405-645/month)
- [x] **Use cases** - Fraud detection, sessionisation, real-time dashboards
- [x] **Cost trade-offs** - 2-4× cost increase for sub-second latency
- [x] **Complexity** - Significant operational burden vs dbt

**Key insight:** Stream processing adds cost and complexity. Only add it when **business value** (measured in $) exceeds the **$200-400/month premium**.

## What's Next

Learn about Change Data Capture (CDC) for real-time database replication.

Continue to [Change Data Capture](11-change-data-capture.md) →
