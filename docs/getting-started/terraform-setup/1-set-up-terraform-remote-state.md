# Set Up Terraform Remote State

On this page, you will:

- [x] Understand why remote state is essential for teams
- [x] Create an S3 bucket for storing Terraform state
- [x] Create a DynamoDB table for state locking
- [x] Configure encryption and versioning for security
- [x] Set up IAM policies for secure access

## Why Remote State?

By default, Terraform stores its state file locally in a file called `terraform.tfstate`. This works fine when you're the only person running Terraform, but it creates problems for teams:

**Problem 1: No collaboration**
If the state file is on your laptop, nobody else can run Terraform. They don't know what infrastructure exists or what Terraform has created.

**Problem 2: No locking**
If two people run Terraform simultaneously, they can corrupt the state file or create conflicting changes.

**Problem 3: No backup**
If you lose your laptop or accidentally delete the state file, Terraform loses track of all your infrastructure. You'd have to manually import everything or start over.

**Problem 4: Secrets in the state**
State files contain sensitive data like passwords and API keys. Having them on laptops increases the risk of exposure.

**The solution: Remote state**

Store the state file in S3, where:
- Everyone on the team can access it
- State locking prevents simultaneous runs
- Versioning provides backup and rollback
- Encryption protects sensitive data
- Access controls limit who can read/write state

## Understanding State Locking

State locking prevents two people from running Terraform at the same time. Without it, this could happen:

1. Alice runs `terraform apply` to create a database
2. Bob runs `terraform apply` at the same time to create a warehouse
3. Both read the state file (showing neither resource exists)
4. Both try to write their changes
5. The state file becomes corrupted or one person's changes are lost

With state locking using DynamoDB:

1. Alice runs `terraform apply`
2. Terraform acquires a lock in DynamoDB
3. Bob tries to run `terraform apply`
4. Terraform sees the lock and waits (or fails with an error)
5. When Alice's run completes, the lock is released
6. Bob's run can then proceed

## Architecture

You'll create:

```
┌─────────────────────────────────────────┐
│ S3 Bucket: terraform-state-<account-id> │
│                                         │
│ - Versioning enabled (can rollback)     │
│ - Encryption enabled (protects secrets) │
│ - Bucket policy (restricts access)      │
│ - Lifecycle rules (old versions → IA)   │
└─────────────────────────────────────────┘
                    │
                    │ Terraform reads/writes state
                    │
┌─────────────────────────────────────--────┐
│ DynamoDB Table: terraform-state-lock      │
│                                           │
│ - Key: LockID (string)                    │
│ - Pay-per-request billing (cost-effective)│
│ - Used only for locking, not data         │
└────────────────────────────────────────--─┘
```

## Create the S3 Bucket

You'll create the S3 bucket manually this first time. Later, you'll import it into Terraform so everything is managed as code.

### Sign in to AWS

Use your IAM user with the DataEngineerRole (or AdminRole if needed):

```sh
aws sts get-caller-identity --profile data-engineer
```

Expected output showing you're using the DataEngineerRole (note - your account ID will be different):

```json
{
    "UserId": "AROAEXAMPLE:aws-cli-session",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/DataEngineerRole/aws-cli-session"
}
```

### Determine Your Bucket Name

S3 bucket names must be globally unique across all AWS accounts. A good pattern is:

```
terraform-state-<your-aws-account-id>
```

Get your AWS account ID:

```sh
aws sts get-caller-identity --query Account --output text --profile data-engineer
```

Example output:
```
123456789012
```

So your bucket name would be: `terraform-state-123456789012`

!!! tip "Why Include Account ID?"
    Including your account ID guarantees the bucket name is unique and makes it clear which AWS account the bucket belongs to. This is helpful when working across multiple AWS accounts.

### Create the Bucket

Create the S3 bucket in your preferred region:

```sh
aws s3api create-bucket \
  --bucket terraform-state-123456789012 \
  --region eu-west-2 \
  --create-bucket-configuration LocationConstraint=eu-west-2 \
  --profile data-engineer
```

!!! note "Region Constraint"
    The `--create-bucket-configuration LocationConstraint=` parameter is required for all regions except `us-east-1`. If you're using `us-east-1`, omit this parameter.

Expected output:

```json
{
    "Location": "http://terraform-state-123456789012.s3.amazonaws.com/"
}
```

### Enable Versioning

Versioning allows you to recover from accidental deletions or corruption:

```sh
aws s3api put-bucket-versioning \
  --bucket terraform-state-123456789012 \
  --versioning-configuration Status=Enabled \
  --profile data-engineer
```

Verify versioning is enabled:

```sh
aws s3api get-bucket-versioning \
  --bucket terraform-state-123456789012 \
  --profile data-engineer
```

Expected output:

```json
{
    "Status": "Enabled"
}
```

### Enable Encryption

Enable default encryption to protect sensitive data in the state file:

```sh
aws s3api put-bucket-encryption \
  --bucket terraform-state-123456789012 \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      },
      "BucketKeyEnabled": true
    }]
  }' \
  --profile data-engineer
```

Verify encryption is enabled:

```sh
aws s3api get-bucket-encryption \
  --bucket terraform-state-123456789012 \
  --profile data-engineer
```

Expected output:

```json
{
    "ServerSideEncryptionConfiguration": {
        "Rules": [
            {
                "ApplyServerSideEncryptionByDefault": {
                    "SSEAlgorithm": "AES256"
                },
                "BucketKeyEnabled": true
            }
        ]
    }
}
```

### Enable Bucket Versioning Lifecycle

Configure a lifecycle rule to move old state versions to cheaper storage:

Create a file called `lifecycle-policy.json`:

```json
{
  "Rules": [
    {
      "Id": "MoveOldVersionsToIA",
      "Status": "Enabled",
      "NoncurrentVersionTransitions": [
        {
          "NoncurrentDays": 30,
          "StorageClass": "STANDARD_IA"
        }
      ],
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 90
      }
    }
  ]
}
```

Apply the lifecycle policy:

```sh
aws s3api put-bucket-lifecycle-configuration \
  --bucket terraform-state-123456789012 \
  --lifecycle-configuration file://lifecycle-policy.json \
  --profile data-engineer
```

This policy:
- Moves versions older than 30 days to Infrequent Access storage (cheaper)
- Deletes versions older than 90 days
- Keeps current version in Standard storage

!!! info "Cost Optimisation"
    Old state file versions are rarely accessed. Moving them to Infrequent Access storage reduces costs whilst maintaining the ability to roll back if needed.

### Block Public Access

Ensure the bucket can never be made public:

```sh
aws s3api put-public-access-block \
  --bucket terraform-state-123456789012 \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true" \
  --profile data-engineer
```

Verify public access is blocked:

```sh
aws s3api get-public-access-block \
  --bucket terraform-state-123456789012 \
  --profile data-engineer
```

Expected output:

```json
{
    "PublicAccessBlockConfiguration": {
        "BlockPublicAcls": true,
        "IgnorePublicAcls": true,
        "BlockPublicPolicy": true,
        "RestrictPublicBuckets": true
    }
}
```

## Create the DynamoDB Table

DynamoDB provides the locking mechanism for Terraform state.

### Create the Table

```sh
aws dynamodb create-table \
  --table-name terraform-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region eu-west-2 \
  --profile data-engineer
```

This creates a table with:
- **Primary key**: `LockID` (string)
- **Billing mode**: Pay-per-request (only pay for what you use)
- No provisioned capacity (scales automatically)

Expected output (truncated):

```json
{
    "TableDescription": {
        "TableName": "terraform-state-lock",
        "TableStatus": "CREATING",
        "KeySchema": [
            {
                "AttributeName": "LockID",
                "KeyType": "HASH"
            }
        ],
        "BillingModeSummary": {
            "BillingMode": "PAY_PER_REQUEST"
        }
    }
}
```

### Wait for Table Creation

The table takes a few moments to create:

```sh
aws dynamodb wait table-exists \
  --table-name terraform-state-lock \
  --region eu-west-2 \
  --profile data-engineer
```

This command waits until the table is ready (no output means success).

Verify the table exists:

```sh
aws dynamodb describe-table \
  --table-name terraform-state-lock \
  --query 'Table.TableStatus' \
  --region eu-west-2 \
  --profile data-engineer
```

Expected output:

```
"ACTIVE"
```

### Enable Point-in-Time Recovery

Enable continuous backups for the DynamoDB table:

```sh
aws dynamodb update-continuous-backups \
  --table-name terraform-state-lock \
  --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true \
  --region eu-west-2 \
  --profile data-engineer
```

Expected output:

```json
{
    "ContinuousBackupsDescription": {
        "ContinuousBackupsStatus": "ENABLED",
        "PointInTimeRecoveryDescription": {
            "PointInTimeRecoveryStatus": "ENABLED"
        }
    }
}
```

!!! tip "Why Enable Backups?"
    Whilst the lock table only stores temporary lock information, enabling point-in-time recovery is a best practice for all DynamoDB tables. The cost is minimal and it provides insurance against accidental deletion.

## Tag Your Resources

Add tags to make it clear these resources are for Terraform state:

### Tag the S3 Bucket

```sh
aws s3api put-bucket-tagging \
  --bucket terraform-state-123456789012 \
  --tagging 'TagSet=[
    {Key=Purpose,Value=TerraformState},
    {Key=ManagedBy,Value=Manual},
    {Key=Environment,Value=Shared}
  ]' \
  --profile data-engineer
```

### Tag the DynamoDB Table

```sh
aws dynamodb tag-resource \
  --resource-arn arn:aws:dynamodb:eu-west-2:123456789012:table/terraform-state-lock \
  --tags Key=Purpose,Value=TerraformStateLock Key=ManagedBy,Value=Manual Key=Environment,Value=Shared \
  --region eu-west-2 \
  --profile data-engineer
```

!!! note "Finding Your Table ARN"
    Replace `123456789012` with your AWS account ID, and `eu-west-2` with your region. Or get it from the table description:

    ```sh
    aws dynamodb describe-table \
      --table-name terraform-state-lock \
      --query 'Table.TableArn' \
      --region eu-west-2 \
      --profile data-engineer
    ```

## Verify Your Setup

### Check S3 Bucket Configuration

```sh
# List all configuration for the bucket
aws s3api head-bucket --bucket terraform-state-123456789012 --profile data-engineer
aws s3api get-bucket-versioning --bucket terraform-state-123456789012 --profile data-engineer
aws s3api get-bucket-encryption --bucket terraform-state-123456789012 --profile data-engineer
aws s3api get-public-access-block --bucket terraform-state-123456789012 --profile data-engineer
```

### Check DynamoDB Table

```sh
aws dynamodb describe-table \
  --table-name terraform-state-lock \
  --region eu-west-2 \
  --profile data-engineer
```

## Cost Considerations

Your remote state infrastructure will cost very little:

**S3 Bucket**
- **Storage**: ~$0.023 per GB per month (Standard storage)
- State files are typically small (< 1 MB for most setups)
- Old versions move to Infrequent Access after 30 days (~$0.0125 per GB)
- **Estimated cost**: < $1 per month

**DynamoDB Table**
- **Billing**: Pay-per-request
- Only charged when Terraform acquires/releases locks
- **Cost per request**: $0.25 per million requests
- Typical usage: 2 requests per Terraform run (acquire + release)
- 100 Terraform runs per month = 200 requests = $0.00005
- **Estimated cost**: < $0.01 per month

**Total estimated cost: ~$1 per month**

!!! tip "Cost Monitoring"
    We will add these resources to your AWS Budget to monitor actual costs. The budget alerts you created in the AWS account setup will notify you if costs exceed expectations.

## Security Best Practices

!!! success "What You've Configured"
    - [x] Encryption at rest (AES256)
    - [x] Versioning enabled (can recover from mistakes)
    - [x] Public access blocked (cannot be made public)
    - [x] Lifecycle rules (old versions archived, then deleted)
    - [x] Point-in-time recovery (DynamoDB backups)
    - [x] Resources tagged (clear purpose and ownership)

Additional security considerations:

**Bucket Policy (Optional)**
You can add a bucket policy to restrict access further. For now, IAM role permissions are sufficient, but you may want to add explicit bucket policies later.

**State File Encryption in Transit**
All communication between Terraform and S3 uses HTTPS, encrypting data in transit automatically.

**Access Logging (Optional)**
You can enable S3 access logging to track who accesses the state bucket. This is useful for compliance but adds complexity. Consider enabling it once your infrastructure matures.

## Troubleshooting

### Error: Bucket name already exists

S3 bucket names are globally unique. If someone else has already taken your chosen name, pick a different one. The pattern `terraform-state-<account-id>-<region>` can help ensure uniqueness.

### Error: Access Denied

Ensure you're using the correct AWS profile and that your DataEngineerRole has the necessary permissions:

```sh
aws sts get-caller-identity --profile data-engineer
```

### Cannot Find lifecycle-policy.json

Make sure you created the `lifecycle-policy.json` file in your current directory. Check with:

```sh
ls -la lifecycle-policy.json
cat lifecycle-policy.json
```

## What's Next

You now have the foundation for remote state management:

- ✅ S3 bucket ready to store state files
- ✅ DynamoDB table ready for state locking
- ✅ Encryption and versioning configured
- ✅ Cost optimisation in place

Next, you'll install Terraform on your local machine and configure it to use this remote state.

Continue to [Set Up Terraform Locally](2-set-up-terraform-locally.md) →
