# Build Your AWS Infrastructure

In this section, you'll build the AWS infrastructure required to support your data platform - S3 buckets for your data lake and optionally VPC networking for compute workloads.

## What You'll Build

By the end of this section, you'll have:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              AWS ACCOUNT                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  S3 BUCKETS (Data Lake)                                                     │
│  ┌───────────────────┐ ┌───────────────────┐ ┌───────────────────┐          │
│  │   data-lake-dev   │ │ data-lake-staging │ │  data-lake-prod   │          │
│  │  (Development)    │ │    (Staging)      │ │   (Production)    │          │
│  └───────────────────┘ └───────────────────┘ └───────────────────┘          │
│                                                                             │
│  VPC NETWORKING (Optional - for ECS/EC2 workloads)                          │
│  ┌─────────────────────────────────────────────────────────────────┐        │
│  │  VPC 10.0.0.0/16                                                │        │
│  │  ├── Public Subnets (2 AZs)   - NAT Gateway                     │        │
│  │  └── Private Subnets (2 AZs)  - ECS, EC2                        │        │
│  └─────────────────────────────────────────────────────────────────┘        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Why Separate AWS Infrastructure?

The [Getting Started](../../getting-started/terraform-setup/aws/index.md) section focused on importing existing AWS resources (IAM roles, state infrastructure, budget alerts) and establishing the [Secrets Manager pattern](../../getting-started/terraform-setup/aws/6-secrets-manager-setup.md). This section creates new infrastructure specifically for your data platform:

- **S3 Buckets**: Storage for data lake files, used by Snowflake storage integrations
- **VPC Networking**: Network infrastructure for EC2 instances, ECS tasks, and other compute workloads

!!! tip "Adding Secrets"
    When you add tools to your stack (dbt, Metabase, etc.), add their secrets following the pattern in [Secrets Manager Setup](../../getting-started/terraform-setup/aws/6-secrets-manager-setup.md#adding-more-secrets).

## Architecture Patterns

### Environment Separation

All resources are created per environment (dev, staging, prod):

| Environment | Purpose | Access |
|-------------|---------|--------|
| **Dev** | Development and experimentation | Developers can read and write |
| **Staging** | Pre-production testing | Transformers can write, developers can read |
| **Prod** | Production data | Transformers write, developers read-only |

### S3 Bucket Module

The S3 bucket module creates buckets with:

- **Versioning**: Recover from accidental deletions or overwrites
- **Encryption**: Server-side encryption with AES-256
- **Public access blocked**: All public access explicitly denied
- **Lifecycle policies**: Automatic cleanup of old versions and incomplete uploads
- **IAM policies**: Pre-built read and write policies for Snowflake integration roles

## Prerequisites

Before starting this section, ensure you have completed:

- [x] [AWS Account Setup](../../getting-started/account-setup/aws.md) - Created your AWS account
- [x] [Add AWS to Terraform](../../getting-started/terraform-setup/aws/index.md) - Imported existing resources

You should have a working `terraform/aws/` directory with:

- IAM roles and users imported
- State infrastructure managed
- CI/CD workflows deploying AWS changes

## Section Overview

| Page | What You'll Build |
|------|-------------------|
| [1. S3 Data Lake](1-s3-data-lake.md) | S3 bucket module and data lake buckets for dev, staging, prod |
| [2. VPC Networking](2-vpc-networking.md) | VPC with public/private subnets, NAT Gateway, security groups (optional) |

!!! note "VPC is Optional"
    VPC networking is only required if you plan to run ECS workers or self-hosted Prefect. If you're using Prefect Cloud with local development workers, you can skip VPC setup for now.

## Cost Considerations

**S3 costs:**

- **Storage**: ~$0.023/GB/month for Standard, less for Infrequent Access
- **Requests**: $0.005 per 1,000 PUT/POST requests, $0.0004 per 1,000 GET requests
- **Data transfer**: Free within the same region

For a small data platform, expect $5-50/month for S3 depending on data volume.

**VPC costs (if enabled):**

- **NAT Gateway**: ~$32/month + $0.045/GB data processed
- **VPC, subnets, Internet Gateway**: Free

For development environments, you can disable NAT Gateway to save costs.

## What's Next

Start by creating the S3 bucket module and data lake buckets.

Continue to [S3 Data Lake](1-s3-data-lake.md) →
