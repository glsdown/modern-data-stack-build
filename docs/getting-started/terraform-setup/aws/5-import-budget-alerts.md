# Import Budget Alerts

On this page, you will:

- [x] Import your existing AWS budget into Terraform
- [x] Configure budget notifications
- [x] Set up a pattern for multiple budgets

## Navigate to Your Terraform Directory

```sh
cd ~/projects/data/data-stack-infrastructure/terraform/aws
```

## Understanding AWS Budgets

You created a cost budget in [AWS Account Setup](../../account-setup/aws.md) to monitor your spending. AWS Budgets allows you to:

- Set monthly spending limits
- Receive alerts at configurable thresholds
- Track actual vs forecasted spending

We'll import this budget into Terraform so it's managed alongside your other infrastructure.

## Gather Budget Information

First, retrieve information about your existing budgets:

```sh
# List all budgets
aws budgets describe-budgets \
  --account-id $(aws sts get-caller-identity --query Account --output text --profile infrastructure-admin) \
  --profile infrastructure-admin
```

Note the budget name and configuration details.

!!! tip "Finding Your Account ID"
    The command above retrieves your account ID automatically. You can also get it directly:

    ```sh
    aws sts get-caller-identity --query Account --output text --profile infrastructure-admin
    ```

## Add Variables for Budgets

Update `variables.tf`:

```hcl
# Budget Configuration
variable "budget_limit_amount" {
  description = "Monthly budget limit in USD"
  type        = string
  default     = "100"
}

variable "budget_alert_email" {
  description = "Email address for budget alerts"
  type        = string
}

variable "budget_alert_threshold" {
  description = "Percentage threshold for budget alerts"
  type        = number
  default     = 80
}
```

Update `terraform.tfvars`:

```hcl
# Budget Configuration
budget_limit_amount    = "100"                    # Replace with your budget limit
budget_alert_email     = "alerts@yourcompany.com" # Replace with your email
budget_alert_threshold = 80                       # Alert at 80% of budget
```

## Create Budget Configuration

Create `budgets.tf`:

```hcl
# =============================================================================
# AWS Budgets
# =============================================================================

# -----------------------------------------------------------------------------
# Monthly Cost Budget
# -----------------------------------------------------------------------------
resource "aws_budgets_budget" "monthly_cost" {
  name         = "Monthly-Cost-Budget"
  budget_type  = "COST"
  limit_amount = var.budget_limit_amount
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = var.budget_alert_threshold
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = [var.budget_alert_email]
  }

  tags = {
    Name        = "Monthly-Cost-Budget"
    Environment = "Production"
  }
}
```

!!! info "Budget Notifications"
    The notification block configures alerts when actual spending exceeds the threshold percentage. You can add multiple notification blocks for different thresholds (e.g. 50%, 80%, 100%).

## Multiple Alert Thresholds

For more comprehensive monitoring, add multiple notification thresholds:

```hcl
resource "aws_budgets_budget" "monthly_cost" {
  name         = "Monthly-Cost-Budget"
  budget_type  = "COST"
  limit_amount = var.budget_limit_amount
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  # Alert at 50% of budget
  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 50
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = [var.budget_alert_email]
  }

  # Alert at 80% of budget
  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = [var.budget_alert_email]
  }

  # Alert at 100% of budget
  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 100
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = [var.budget_alert_email]
  }

  # Forecasted alert - warns if you're on track to exceed
  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 100
    threshold_type             = "PERCENTAGE"
    notification_type          = "FORECASTED"
    subscriber_email_addresses = [var.budget_alert_email]
  }

  tags = {
    Name        = "Monthly-Cost-Budget"
    Environment = "Production"
  }
}
```

## Add Import Block

Add this import block to `imports.tf`:

```hcl
# AWS Budget
import {
  to = aws_budgets_budget.monthly_cost
  id = "123456789012:Monthly-Cost-Budget"  # Replace with your account ID and budget name
}
```

!!! warning "Import ID Format"
    The import ID for AWS budgets is `account_id:budget_name`. Ensure you use the exact budget name as it appears in AWS.

## Plan and Apply

```sh
terraform plan
```

Review the output. If importing an existing budget, you should see:

```
Plan: 1 to import, 0 to add, 0 to change, 0 to destroy.
```

If Terraform shows changes, your configuration may not match the existing budget exactly. Common differences:

- **Notification thresholds**: Ensure thresholds match the existing budget
- **Email addresses**: Must match exactly
- **Budget amount**: Should match the configured limit

Apply the import:

```sh
terraform apply
```

## Service-Specific Budgets (Optional)

For more granular cost control, you can create budgets for specific services:

```hcl
variable "service_budgets" {
  description = "Map of service-specific budgets"
  type = map(object({
    limit_amount = string
    service_name = string
  }))
  default = {}
}

resource "aws_budgets_budget" "service" {
  for_each = var.service_budgets

  name         = "Service-${each.key}-Budget"
  budget_type  = "COST"
  limit_amount = each.value.limit_amount
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  cost_filter {
    name   = "Service"
    values = [each.value.service_name]
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = [var.budget_alert_email]
  }

  tags = {
    Name    = "Service-${each.key}-Budget"
    Service = each.key
  }
}
```

Example `terraform.tfvars`:

```hcl
service_budgets = {
  "S3" = {
    limit_amount = "20"
    service_name = "Amazon Simple Storage Service"
  }
  "EC2" = {
    limit_amount = "50"
    service_name = "Amazon Elastic Compute Cloud - Compute"
  }
}
```

## Add Outputs

Update `outputs.tf`:

```hcl
# Budget outputs
output "monthly_budget_id" {
  description = "ID of the monthly cost budget"
  value       = aws_budgets_budget.monthly_cost.id
}

output "monthly_budget_name" {
  description = "Name of the monthly cost budget"
  value       = aws_budgets_budget.monthly_cost.name
}
```

## Verify the Import

```sh
terraform state list | grep aws_budgets
```

Expected output:

```
aws_budgets_budget.monthly_cost
```

## Commit Your Work

```sh
git add terraform/aws/
git commit -m "Import AWS budget into Terraform"
```

## Troubleshooting

### Error: Budget Not Found

If the import fails with "budget not found":

1. Verify the budget name is correct (case-sensitive)
2. Ensure you're using the correct account ID
3. Check you have permissions to view budgets

```sh
# List all budgets to verify the name
aws budgets describe-budgets \
  --account-id $(aws sts get-caller-identity --query Account --output text --profile infrastructure-admin) \
  --profile infrastructure-admin
```

### Notification Email Differences

If Terraform wants to update notification emails, ensure the email in your configuration matches exactly what's configured in AWS. AWS may have normalised the email (e.g. lowercase).

### Budget Already Exists

If you're creating a new budget and get "budget already exists", you need to import it instead:

```sh
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text --profile infrastructure-admin)
terraform import aws_budgets_budget.monthly_cost "${ACCOUNT_ID}:Monthly-Cost-Budget"
```

## What's Next

You've successfully imported budget alerts into Terraform:

- [x] Monthly cost budget managed in code
- [x] Alert notifications configured
- [x] Pattern established for service-specific budgets

Continue to [set up Secrets Manager](6-secrets-manager-setup.md) →
