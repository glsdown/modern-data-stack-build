# VPC and Networking

On this page, you will:

- [x] Understand when you need VPC infrastructure
- [x] Create a VPC with public and private subnets
- [x] Set up NAT Gateway for private subnet internet access
- [x] Create a default security group

## Key Concepts

Before diving into the setup, here's what these networking components do:

### VPC (Virtual Private Cloud)

A VPC is your own isolated network within AWS. Think of it as your private data centre in the cloud - you control the IP address ranges, who can access what, and how traffic flows between resources.

Without a VPC, AWS services like EC2 instances would be exposed directly to the internet. A VPC lets you place resources in a private network where you control all access.

### Subnets

Subnets divide your VPC into smaller network segments. There are two types:

| Type | Description | Use For |
|------|-------------|---------|
| **Public subnet** | Has a route to the internet via an Internet Gateway. Resources here can receive inbound traffic from the internet. | NAT Gateways, bastion hosts (if needed) |
| **Private subnet** | No direct internet access. Resources here are protected from inbound internet traffic. | EC2 instances, ECS tasks, anything that shouldn't be directly accessible |

The security benefit: resources in private subnets can't be reached from the internet, even if you accidentally misconfigure a security group.

### NAT Gateway

A NAT (Network Address Translation) Gateway allows resources in private subnets to make **outbound** requests to the internet (e.g., downloading packages, calling APIs) whilst remaining unreachable for **inbound** connections.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                INTERNET                                     │
└───────────────────────────────────-─────────────────────────────────────────┘
                                    ▲
                    ┌───────────────┴───────────────┐
                    │                               │
                    ▼                               |
          ┌─────────────────┐             ┌─────────────────┐
          │ Internet Gateway│             │   NAT Gateway   │
          │  (bidirectional)│             │ (outbound only) │
          └────────-────────┘             └────────-────────┘
                   ▲                               ▲
                   ▼                               |
          ┌─────────────────┐             ┌─────────────────┐
          │  Public Subnet  │             │ Private Subnet  │ Unreachable for
          │  (NAT Gateway)  │             │   (ECS, EC2)    │ inbound requests
          └─────────────────┘             └─────────────────┘
```

Without a NAT Gateway, your ECS tasks in private subnets couldn't pull Docker images, call external APIs, or download dependencies.

### Route Tables

Route tables contain rules that determine where network traffic is directed. Each subnet is associated with a route table that defines its connectivity:

- **Public subnet route table**: Routes `0.0.0.0/0` (all internet traffic) to the Internet Gateway
- **Private subnet route table**: Routes `0.0.0.0/0` to the NAT Gateway (for outbound only)

This is what makes a subnet "public" or "private" - not the subnet itself, but the route table it's associated with.

## When Do You Need a VPC?

A VPC is required for any AWS compute resources:

- **EC2 instances** - virtual servers
- **ECS/Fargate tasks** - containerised workloads
- **RDS databases** - managed PostgreSQL, MySQL, etc.
- **Lambda functions** - when they need to access private resources
- **PrivateLink** - private connectivity to SaaS services (e.g., Snowflake)

!!! tip "Do I Need This Now?"
    If you're only using managed services that don't require VPC (like S3, Secrets Manager, or Prefect Cloud with local workers), you can skip this page. Come back when you need to run compute workloads in AWS.

## Architecture Overview

This setup creates a pragmatic, cost-effective VPC:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                   VPC                                       │
│                              10.0.0.0/16                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────┐    ┌─────────────────────────────┐         │
│  │     Availability Zone A     │    │     Availability Zone B     │         │
│  │         (eu-west-2a)        │    │         (eu-west-2b)        │         │
│  │                             │    │                             │         │
│  │  ┌───────────────────────┐  │    │  ┌───────────────────────┐  │         │
│  │  │    Public Subnet      │  │    │  │    Public Subnet      │  │         │
│  │  │     10.0.1.0/24       │  │    │  │     10.0.2.0/24       │  │         │
│  │  │    (NAT Gateway)      │  │    │  │                       │  │         │
│  │  └───────────────────────┘  │    │  └───────────────────────┘  │         │
│  │                             │    │                             │         │
│  │  ┌───────────────────────┐  │    │  ┌───────────────────────┐  │         │
│  │  │   Private Subnet      │  │    │  │   Private Subnet      │  │         │
│  │  │     10.0.11.0/24      │  │    │  │     10.0.12.0/24      │  │         │
│  │  │     (ECS, EC2)        │  │    │  │      (ECS, EC2)       │  │         │
│  │  └───────────────────────┘  │    │  └───────────────────────┘  │         │
│  │                             │    │                             │         │
│  └─────────────────────────────┘    └─────────────────────────────┘         │
│                                                                             │
│  ┌─────────────────────────────┐    ┌─────────────────────────────┐         │
│  │      Internet Gateway       │    │         NAT Gateway         │         │
│  │     (Public → Internet)     │    │    (Private → Internet)     │         │
│  └─────────────────────────────┘    └─────────────────────────────┘         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Design decisions:**

- **10.0.0.0/16 CIDR block**: Uses the RFC 1918 private address range (10.x.x.x). The /16 gives 65,536 IP addresses - plenty of room for subnets. This is a common convention that avoids conflicts with typical home networks (192.168.x.x).
- **2 Availability Zones**: Sufficient for high availability, simpler than 3
- **Single NAT Gateway**: Saves ~$64/month vs one per AZ (trade-off: single point of failure for outbound traffic)
- **Separate public/private subnets**: Security best practice - workloads in private subnets

## Cost Breakdown

| Component | Monthly Cost |
|-----------|-------------|
| VPC | Free |
| Subnets | Free |
| Internet Gateway | Free |
| NAT Gateway | ~$32 + $0.045/GB data processed |
| Flow Logs (production only) | ~$1-5 (CloudWatch ingestion + storage) |
| **Total (minimal)** | **~$35/month** |

!!! warning "NAT Gateway Costs"
    NAT Gateway is the main cost. For development environments with light traffic, expect ~$35/month. High-traffic production environments could be significantly more due to data processing charges.

!!! note "Flow Logs Costs"
    Flow Logs are only enabled for production (`enable_flow_logs = true`). CloudWatch charges ~$0.50/GB for log ingestion and ~$0.03/GB/month for storage. For most VPCs, this is a few dollars per month.

## Create the VPC Module

Create a reusable VPC module.

### Module Structure

```
terraform/aws/
├── modules/
│   └── vpc/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── config/
    └── vpc.tf
```

### Create the Module

Create `modules/vpc/main.tf`:

```hcl
# =============================================================================
# VPC Module
# =============================================================================
# Creates a VPC with public and private subnets across 2 AZs,
# with a single NAT Gateway for cost efficiency.

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# -----------------------------------------------------------------------------
# Data Sources
# -----------------------------------------------------------------------------

# Get available AZs in the current region, then take the first 2.
# This makes the module region-agnostic - it works in any AWS region.
data "aws_availability_zones" "available" {
  state = "available"
}

locals {
  azs = slice(data.aws_availability_zones.available.names, 0, 2)
}

# -----------------------------------------------------------------------------
# VPC
# -----------------------------------------------------------------------------
resource "aws_vpc" "this" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(var.tags, {
    Name = var.name
  })
}

# -----------------------------------------------------------------------------
# Internet Gateway
# -----------------------------------------------------------------------------
resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id

  tags = merge(var.tags, {
    Name = "${var.name}-igw"
  })
}

# -----------------------------------------------------------------------------
# Public Subnets
# -----------------------------------------------------------------------------
resource "aws_subnet" "public" {
  count = length(local.azs)

  vpc_id                  = aws_vpc.this.id
  cidr_block              = cidrsubnet(var.cidr_block, 8, count.index + 1)
  availability_zone       = local.azs[count.index]
  map_public_ip_on_launch = true

  tags = merge(var.tags, {
    Name = "${var.name}-public-${local.azs[count.index]}"
    Tier = "public"
  })
}

# -----------------------------------------------------------------------------
# Private Subnets
# -----------------------------------------------------------------------------
resource "aws_subnet" "private" {
  count = length(local.azs)

  vpc_id            = aws_vpc.this.id
  cidr_block        = cidrsubnet(var.cidr_block, 8, count.index + 11)
  availability_zone = local.azs[count.index]

  tags = merge(var.tags, {
    Name = "${var.name}-private-${local.azs[count.index]}"
    Tier = "private"
  })
}

# -----------------------------------------------------------------------------
# NAT Gateway (single, in first public subnet)
# -----------------------------------------------------------------------------
resource "aws_eip" "nat" {
  count  = var.enable_nat_gateway ? 1 : 0
  domain = "vpc"

  tags = merge(var.tags, {
    Name = "${var.name}-nat-eip"
  })

  depends_on = [aws_internet_gateway.this]
}

resource "aws_nat_gateway" "this" {
  count = var.enable_nat_gateway ? 1 : 0

  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.public[0].id

  tags = merge(var.tags, {
    Name = "${var.name}-nat"
  })

  depends_on = [aws_internet_gateway.this]
}

# -----------------------------------------------------------------------------
# Route Tables
# -----------------------------------------------------------------------------

# Public route table - routes to Internet Gateway
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.this.id
  }

  tags = merge(var.tags, {
    Name = "${var.name}-public-rt"
  })
}

resource "aws_route_table_association" "public" {
  count = length(local.azs)

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Private route table - routes to NAT Gateway
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.this.id

  dynamic "route" {
    for_each = var.enable_nat_gateway ? [1] : []
    content {
      cidr_block     = "0.0.0.0/0"
      nat_gateway_id = aws_nat_gateway.this[0].id
    }
  }

  tags = merge(var.tags, {
    Name = "${var.name}-private-rt"
  })
}

resource "aws_route_table_association" "private" {
  count = length(local.azs)

  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id
}

# -----------------------------------------------------------------------------
# VPC Flow Logs (Optional)
# -----------------------------------------------------------------------------
# Flow logs capture metadata about network traffic in your VPC (source/dest IPs,
# ports, protocols, accept/reject status). Useful for debugging connectivity
# issues and security monitoring. Logs are sent to CloudWatch and retained for
# 14 days. Only enabled for production by default to save costs.

resource "aws_flow_log" "this" {
  count = var.enable_flow_logs ? 1 : 0

  vpc_id          = aws_vpc.this.id
  traffic_type    = "ALL"
  iam_role_arn    = aws_iam_role.flow_logs[0].arn
  log_destination = aws_cloudwatch_log_group.flow_logs[0].arn

  tags = merge(var.tags, {
    Name = "${var.name}-flow-logs"
  })
}

resource "aws_cloudwatch_log_group" "flow_logs" {
  count = var.enable_flow_logs ? 1 : 0

  name              = "/aws/vpc/${var.name}/flow-logs"
  retention_in_days = 14

  tags = var.tags
}

resource "aws_iam_role" "flow_logs" {
  count = var.enable_flow_logs ? 1 : 0

  name = "${var.name}-flow-logs-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "vpc-flow-logs.amazonaws.com"
        }
      }
    ]
  })

  tags = var.tags
}

resource "aws_iam_role_policy" "flow_logs" {
  count = var.enable_flow_logs ? 1 : 0

  name = "${var.name}-flow-logs-policy"
  role = aws_iam_role.flow_logs[0].id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents",
          "logs:DescribeLogGroups",
          "logs:DescribeLogStreams"
        ]
        Effect   = "Allow"
        Resource = "*"
      }
    ]
  })
}
```

### Module Variables

Create `modules/vpc/variables.tf`:

```hcl
variable "name" {
  description = "Name prefix for VPC resources"
  type        = string
}

variable "cidr_block" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "enable_nat_gateway" {
  description = "Create NAT Gateway for private subnet internet access"
  type        = bool
  default     = true
}

variable "enable_flow_logs" {
  description = "Enable VPC flow logs to CloudWatch"
  type        = bool
  default     = false
}

variable "tags" {
  description = "Tags to apply to all resources"
  type        = map(string)
  default     = {}
}
```

### Module Outputs

Create `modules/vpc/outputs.tf`:

```hcl
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.this.id
}

output "vpc_cidr_block" {
  description = "CIDR block of the VPC"
  value       = aws_vpc.this.cidr_block
}

output "public_subnet_ids" {
  description = "IDs of the public subnets"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "IDs of the private subnets"
  value       = aws_subnet.private[*].id
}

output "public_subnet_cidrs" {
  description = "CIDR blocks of the public subnets"
  value       = aws_subnet.public[*].cidr_block
}

output "private_subnet_cidrs" {
  description = "CIDR blocks of the private subnets"
  value       = aws_subnet.private[*].cidr_block
}

output "nat_gateway_public_ip" {
  description = "Public IP of the NAT Gateway"
  value       = var.enable_nat_gateway ? aws_eip.nat[0].public_ip : null
}

output "internet_gateway_id" {
  description = "ID of the Internet Gateway"
  value       = aws_internet_gateway.this.id
}

output "availability_zones" {
  description = "Availability zones used"
  value       = local.azs
}

output "private_route_table_id" {
  description = "ID of the private route table"
  value       = aws_route_table.private.id
}
```

## Use the VPC Module

Create `vpc.tf` in your AWS config directory:

```hcl
# =============================================================================
# VPC Infrastructure
# =============================================================================
# Creates VPC with public and private subnets for data platform workloads.

module "vpc" {
  source = "../modules/vpc"

  name       = "${var.project_name}-${var.environment}"
  cidr_block = "10.0.0.0/16"

  enable_nat_gateway = true
  enable_flow_logs   = var.environment == "production"

  tags = {
    Environment = var.environment
  }
}

# -----------------------------------------------------------------------------
# Outputs
# -----------------------------------------------------------------------------
output "vpc_id" {
  description = "VPC ID"
  value       = module.vpc.vpc_id
}

output "public_subnet_ids" {
  description = "Public subnet IDs"
  value       = module.vpc.public_subnet_ids
}

output "private_subnet_ids" {
  description = "Private subnet IDs"
  value       = module.vpc.private_subnet_ids
}

output "nat_gateway_ip" {
  description = "NAT Gateway public IP"
  value       = module.vpc.nat_gateway_public_ip
}
```

Add required variables to `variables.tf`:

```hcl
variable "project_name" {
  description = "Project name used for resource naming"
  type        = string
}

variable "environment" {
  description = "Environment name (dev, staging, production)"
  type        = string
  default     = "production"
}
```

## Default Security Group

Create a default security group that allows outbound internet access. This is useful for any workload in a private subnet that needs to call external APIs, download packages, or access AWS services.

Add to `vpc.tf`:

```hcl
# -----------------------------------------------------------------------------
# Default Security Group - Outbound Internet Access
# -----------------------------------------------------------------------------
resource "aws_security_group" "default" {
  name        = "${var.project_name}-${var.environment}-default"
  description = "Default security group with outbound internet access"
  vpc_id      = module.vpc.vpc_id

  egress {
    description = "All outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.project_name}-${var.environment}-default"
  }
}

output "security_group_default_id" {
  description = "Default security group ID (outbound internet access)"
  value       = aws_security_group.default.id
}
```

!!! note "Adding More Security Groups"
    Create additional security groups when you add specific services. For example, when adding an RDS database, create a security group that allows PostgreSQL access from your application's security group.

## Deploy the VPC

Commit and push to deploy via CI/CD:

```sh
git add modules/vpc/ vpc.tf security_groups.tf variables.tf
git commit -m "Add VPC module and networking infrastructure"
git push
```

Create a PR to review the plan, then merge to apply.

## Verify the Setup

After deployment, verify the VPC:

```sh
# List VPCs
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=*data-platform*" \
    --query 'Vpcs[*].[VpcId,CidrBlock,Tags[?Key==`Name`].Value|[0]]' \
    --output table

# List subnets
aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-xxxxxxxxx" \
    --query 'Subnets[*].[SubnetId,CidrBlock,AvailabilityZone,Tags[?Key==`Name`].Value|[0]]' \
    --output table

# Check NAT Gateway
aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=vpc-xxxxxxxxx" \
    --query 'NatGateways[*].[NatGatewayId,State,NatGatewayAddresses[0].PublicIp]' \
    --output table
```

## Cost Optimisation Options

### Option 1: Disable NAT Gateway for Dev

For development environments where you don't need private subnet internet access:

```hcl
module "vpc_dev" {
  source = "../modules/vpc"

  name               = "${var.project_name}-dev"
  enable_nat_gateway = false  # Saves ~$32/month
}
```

Resources in private subnets won't have internet access, but you can still access AWS services via VPC endpoints.

### Option 2: Add VPC Endpoints

VPC endpoints allow resources in private subnets to access AWS services without going through the NAT Gateway. This reduces NAT Gateway data processing costs and keeps traffic within the AWS network.

There are two types:

| Type | How It Works | Cost | Use For |
|------|--------------|------|---------|
| **Gateway** | Adds a route to your route table | Free | S3, DynamoDB |
| **Interface** | Creates an ENI in your subnet | ~$7/month + $0.01/GB | Most other AWS services |

**Common endpoints for a data platform:**

```hcl
# -----------------------------------------------------------------------------
# S3 Gateway Endpoint (Free)
# -----------------------------------------------------------------------------
# Routes S3 traffic directly, bypassing NAT Gateway entirely.
# Essential if you're moving significant data to/from S3.

resource "aws_vpc_endpoint" "s3" {
  vpc_id       = module.vpc.vpc_id
  service_name = "com.amazonaws.${var.aws_region}.s3"

  route_table_ids = [module.vpc.private_route_table_id]

  tags = {
    Name = "${var.project_name}-${var.environment}-s3-endpoint"
  }
}

# -----------------------------------------------------------------------------
# Secrets Manager Interface Endpoint (~$7/month)
# -----------------------------------------------------------------------------
# Allows private subnet resources to fetch secrets without NAT Gateway.
# Useful if you have many secrets lookups.

resource "aws_vpc_endpoint" "secretsmanager" {
  vpc_id              = module.vpc.vpc_id
  service_name        = "com.amazonaws.${var.aws_region}.secretsmanager"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = module.vpc.private_subnet_ids
  security_group_ids  = [aws_security_group.default.id]
  private_dns_enabled = true

  tags = {
    Name = "${var.project_name}-${var.environment}-secretsmanager-endpoint"
  }
}

# -----------------------------------------------------------------------------
# ECR Endpoints (~$14/month for both)
# -----------------------------------------------------------------------------
# Required for ECS tasks to pull container images without NAT Gateway.
# You need both ecr.api and ecr.dkr endpoints.

resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id              = module.vpc.vpc_id
  service_name        = "com.amazonaws.${var.aws_region}.ecr.api"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = module.vpc.private_subnet_ids
  security_group_ids  = [aws_security_group.default.id]
  private_dns_enabled = true

  tags = {
    Name = "${var.project_name}-${var.environment}-ecr-api-endpoint"
  }
}

resource "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id              = module.vpc.vpc_id
  service_name        = "com.amazonaws.${var.aws_region}.ecr.dkr"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = module.vpc.private_subnet_ids
  security_group_ids  = [aws_security_group.default.id]
  private_dns_enabled = true

  tags = {
    Name = "${var.project_name}-${var.environment}-ecr-dkr-endpoint"
  }
}
```

!!! tip "When to Add Endpoints"
    - **S3 endpoint**: Add immediately if you're using S3 significantly - it's free
    - **ECR endpoints**: Add if running ECS tasks and want to reduce NAT costs
    - **Secrets Manager endpoint**: Add if you have frequent secrets lookups

    For most small deployments, the NAT Gateway is simpler and the data costs are minimal. Add endpoints when NAT Gateway data processing charges become significant.

## Summary

You've created a foundational VPC setup:

- [x] VPC with public and private subnets across 2 AZs
- [x] Single NAT Gateway for cost efficiency
- [x] Default security group for outbound access
- [x] Terraform module for reusability

This VPC can support any compute workloads you add to your data platform - EC2 instances, ECS tasks, RDS databases, or Lambda functions that need VPC access.

## What's Next

Continue with S3 data lake if you haven't already:

Continue to [S3 Data Lake](1-s3-data-lake.md) →
