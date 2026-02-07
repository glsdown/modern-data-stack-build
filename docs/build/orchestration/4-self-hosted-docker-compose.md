# Self-Hosted: Docker Compose

!!! warning "Advanced - Requires VPC Configuration"
    This page covers self-hosted Prefect, which requires AWS VPC and networking infrastructure not yet covered in this guide. **Most users should use [Prefect Cloud](3-prefect-cloud-setup.md)** - it's simpler, has no infrastructure to manage, and the free tier is sufficient for getting started.

    Self-hosting is only recommended if you have:

    - Strict data sovereignty requirements
    - Existing VPC infrastructure
    - Experience managing EC2 instances

On this page, you will:

- [x] Deploy Prefect server on EC2 with Docker Compose
- [x] Configure PostgreSQL for state storage
- [x] Set up Terraform for the EC2 infrastructure
- [x] Configure backups and maintenance

## Overview

This is the simplest self-hosted option - a single EC2 instance running Prefect server, PostgreSQL, and optionally a worker, all managed by Docker Compose.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         EC2 INSTANCE (t3.small)                             │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────┐        │
│  │                      Docker Compose                             │        │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │        │
│  │  │   Prefect   │  │  PostgreSQL │  │   Worker    │              │        │
│  │  │   Server    │  │             │  │  (optional) │              │        │
│  │  │  :4200      │  │  :5432      │  │             │              │        │
│  │  └─────────────┘  └─────────────┘  └─────────────┘              │        │
│  └─────────────────────────────────────────────────────────────────┘        │
│                                                                             │
│  ┌─────────────────┐                                                        │
│  │   EBS Volume    │  PostgreSQL data persistence                           │
│  │    (20GB)       │                                                        │
│  └─────────────────┘                                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Estimated cost: ~$17/month** (t3.small + 20GB EBS)

## Prerequisites

- [x] **AWS VPC with subnets** - You need an existing VPC with at least one subnet (private recommended). VPC setup is not covered in this guide.
- [x] Terraform configured with remote state
- [x] SSH key pair for EC2 access
- [x] VPN or bastion host access to private subnets

!!! tip "Don't Have a VPC?"
    If you don't have VPC infrastructure set up, we recommend using [Prefect Cloud](3-prefect-cloud-setup.md) instead. You can always migrate to self-hosted later when you have networking in place.

## Terraform Infrastructure

### Project Structure

```
terraform/
└── prefect-server/
    ├── config/
    │   ├── backend.tf
    │   ├── main.tf
    │   ├── providers.tf
    │   ├── variables.tf
    │   ├── terraform.tfvars
    │   ├── ec2.tf
    │   ├── security_groups.tf
    │   └── outputs.tf
    └── files/
        ├── docker-compose.yml
        └── user-data.sh
```

### Backend and Provider Configuration

Create `terraform/prefect-server/config/backend.tf`:

```hcl
terraform {
  backend "s3" {
    bucket         = "your-terraform-state-bucket"
    key            = "prefect-server/terraform.tfstate"
    region         = "eu-west-2"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

Create `terraform/prefect-server/config/main.tf`:

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

Create `terraform/prefect-server/config/providers.tf`:

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

Create `terraform/prefect-server/config/variables.tf`:

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
  description = "VPC ID for the Prefect server"
  type        = string
}

variable "subnet_id" {
  description = "Subnet ID for the Prefect server (private subnet recommended)"
  type        = string
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.small"
}

variable "key_name" {
  description = "SSH key pair name for EC2 access"
  type        = string
}

variable "allowed_cidr_blocks" {
  description = "CIDR blocks allowed to access Prefect UI"
  type        = list(string)
  default     = []
}

variable "postgres_password" {
  description = "PostgreSQL password for Prefect"
  type        = string
  sensitive   = true
}
```

Create `terraform/prefect-server/config/terraform.tfvars`:

```hcl
aws_region          = "eu-west-2"
environment         = "production"
vpc_id              = "vpc-xxxxxxxxx"
subnet_id           = "subnet-xxxxxxxxx"
instance_type       = "t3.small"
key_name            = "your-key-pair"
allowed_cidr_blocks = ["10.0.0.0/8"]  # Your VPN/office CIDR
# postgres_password set via environment variable or secrets
```

### Security Groups

Create `terraform/prefect-server/config/security_groups.tf`:

```hcl
# -----------------------------------------------------------------------------
# Security Group for Prefect Server
# -----------------------------------------------------------------------------
resource "aws_security_group" "prefect_server" {
  name        = "prefect-server-${var.environment}"
  description = "Security group for Prefect server"
  vpc_id      = var.vpc_id

  # Prefect UI (internal access only)
  ingress {
    description = "Prefect UI"
    from_port   = 4200
    to_port     = 4200
    protocol    = "tcp"
    cidr_blocks = var.allowed_cidr_blocks
  }

  # SSH access (for maintenance)
  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = var.allowed_cidr_blocks
  }

  # Outbound internet access
  egress {
    description = "All outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "prefect-server-${var.environment}"
  }
}
```

### EC2 Instance

Create `terraform/prefect-server/config/ec2.tf`:

```hcl
# -----------------------------------------------------------------------------
# Latest Amazon Linux 2023 AMI
# -----------------------------------------------------------------------------
data "aws_ami" "amazon_linux_2023" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# -----------------------------------------------------------------------------
# IAM Role for EC2
# -----------------------------------------------------------------------------
resource "aws_iam_role" "prefect_server" {
  name = "prefect-server-${var.environment}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "ssm" {
  role       = aws_iam_role.prefect_server.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

resource "aws_iam_instance_profile" "prefect_server" {
  name = "prefect-server-${var.environment}"
  role = aws_iam_role.prefect_server.name
}

# -----------------------------------------------------------------------------
# EC2 Instance
# -----------------------------------------------------------------------------
resource "aws_instance" "prefect_server" {
  ami                    = data.aws_ami.amazon_linux_2023.id
  instance_type          = var.instance_type
  subnet_id              = var.subnet_id
  vpc_security_group_ids = [aws_security_group.prefect_server.id]
  key_name               = var.key_name
  iam_instance_profile   = aws_iam_instance_profile.prefect_server.name

  root_block_device {
    volume_size = 20
    volume_type = "gp3"
    encrypted   = true
  }

  user_data = templatefile("${path.module}/../files/user-data.sh", {
    postgres_password = var.postgres_password
  })

  tags = {
    Name = "prefect-server-${var.environment}"
  }

  lifecycle {
    ignore_changes = [ami]  # Don't replace on AMI updates
  }
}
```

### User Data Script

Create `terraform/prefect-server/files/user-data.sh`:

```bash
#!/bin/bash
set -e

# Install Docker
dnf update -y
dnf install -y docker
systemctl enable docker
systemctl start docker

# Install Docker Compose
curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# Create Prefect directory
mkdir -p /opt/prefect
cd /opt/prefect

# Create Docker Compose file
cat > docker-compose.yml << 'EOF'
version: "3.9"

services:
  postgres:
    image: postgres:15-alpine
    restart: always
    environment:
      POSTGRES_USER: prefect
      POSTGRES_PASSWORD: ${postgres_password}
      POSTGRES_DB: prefect
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U prefect"]
      interval: 10s
      timeout: 5s
      retries: 5

  prefect-server:
    image: prefecthq/prefect:3-latest
    restart: always
    command: prefect server start --host 0.0.0.0
    ports:
      - "4200:4200"
    environment:
      PREFECT_API_DATABASE_CONNECTION_URL: postgresql+asyncpg://prefect:${postgres_password}@postgres:5432/prefect
      PREFECT_SERVER_API_HOST: 0.0.0.0
      PREFECT_SERVER_API_PORT: 4200
    depends_on:
      postgres:
        condition: service_healthy

  worker:
    image: prefecthq/prefect:3-latest
    restart: always
    command: prefect worker start --pool default
    environment:
      PREFECT_API_URL: http://prefect-server:4200/api
    depends_on:
      - prefect-server

volumes:
  postgres_data:
EOF

# Start services
docker-compose up -d

# Create systemd service for auto-start
cat > /etc/systemd/system/prefect.service << 'SYSTEMD'
[Unit]
Description=Prefect Server
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/prefect
ExecStart=/usr/local/bin/docker-compose up -d
ExecStop=/usr/local/bin/docker-compose down

[Install]
WantedBy=multi-user.target
SYSTEMD

systemctl enable prefect.service
```

### Outputs

Create `terraform/prefect-server/config/outputs.tf`:

```hcl
output "instance_id" {
  description = "EC2 instance ID"
  value       = aws_instance.prefect_server.id
}

output "private_ip" {
  description = "Private IP address of Prefect server"
  value       = aws_instance.prefect_server.private_ip
}

output "prefect_ui_url" {
  description = "Prefect UI URL (internal)"
  value       = "http://${aws_instance.prefect_server.private_ip}:4200"
}

output "prefect_api_url" {
  description = "Prefect API URL for workers and CLI"
  value       = "http://${aws_instance.prefect_server.private_ip}:4200/api"
}
```

## Deploy the Infrastructure

```sh
cd terraform/prefect-server/config

# Set the PostgreSQL password
export TF_VAR_postgres_password="your-secure-password"

# Initialise and apply
terraform init
terraform plan
terraform apply
```

## Connect to Prefect

### Configure CLI

Point your local Prefect CLI to the self-hosted server:

```sh
# Set the API URL (use VPN or bastion if in private subnet)
export PREFECT_API_URL="http://PRIVATE_IP:4200/api"

# Verify connection
prefect version
prefect work-pool ls
```

### Access the UI

The Prefect UI is available at `http://PRIVATE_IP:4200`. Access it via:

- VPN connection to your VPC
- SSH tunnel: `ssh -L 4200:localhost:4200 ec2-user@INSTANCE_IP`
- Bastion host

## Create a Default Work Pool

Unlike Prefect Cloud, self-hosted servers need work pools created manually:

```sh
# Create the default work pool (matches the worker in docker-compose)
prefect work-pool create default --type process
```

## Maintenance

### Viewing Logs

```sh
# SSH to the instance
ssh ec2-user@INSTANCE_IP

# View logs
cd /opt/prefect
docker-compose logs -f prefect-server
docker-compose logs -f postgres
docker-compose logs -f worker
```

### Upgrading Prefect

```sh
# SSH to the instance
ssh ec2-user@INSTANCE_IP

cd /opt/prefect

# Pull latest images
docker-compose pull

# Restart with new images
docker-compose up -d
```

### Database Backups

Add a backup script to cron:

```sh
# Create backup script
cat > /opt/prefect/backup.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/opt/prefect/backups"
mkdir -p $BACKUP_DIR
docker-compose exec -T postgres pg_dump -U prefect prefect > "$BACKUP_DIR/prefect-$(date +%Y%m%d-%H%M%S).sql"
# Keep last 7 days
find $BACKUP_DIR -name "*.sql" -mtime +7 -delete
EOF

chmod +x /opt/prefect/backup.sh

# Add to cron (daily at 2 AM)
echo "0 2 * * * /opt/prefect/backup.sh" | crontab -
```

!!! warning "Off-site Backups"
    For production use, copy backups to S3:

    ```sh
    aws s3 cp "$BACKUP_DIR/prefect-$(date +%Y%m%d).sql" s3://your-backup-bucket/prefect/
    ```

## Limitations

This setup is suitable for small teams but has limitations:

| Aspect | Limitation |
|--------|------------|
| **High availability** | Single point of failure |
| **Scaling** | Vertical only (larger instance) |
| **Backups** | Manual setup required |
| **Networking** | Requires VPN/bastion for access |
| **Upgrades** | Requires SSH and downtime |

For production workloads requiring HA, consider [ECS + RDS](5-self-hosted-ecs-production.md).

## Summary

You've deployed Prefect server on EC2 with Docker Compose:

- [x] Created EC2 infrastructure with Terraform
- [x] Deployed Prefect server, PostgreSQL, and worker
- [x] Configured CLI access
- [x] Set up basic maintenance procedures

## What's Next

With the server running, configure work pools and workers for your specific needs.

Continue to [Work Pools and Workers](6-work-pools-and-workers.md) →
