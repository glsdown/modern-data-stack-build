# Finishing Up

On this page, you will:

- [x] Review what you've built
- [x] Verify your orchestration setup
- [x] Understand next steps

## Summary

You now have a production-ready orchestration layer for your data platform.

### Architecture Summary

=== "Prefect Cloud"

    ```
    ┌─────────────────────────────────────────────────────────────────────────────┐
    │                          PREFECT CLOUD                                      │
    │                     (Managed by Prefect)                                    │
    │  ┌─────────────────────────────────────────────────────────────────┐        │
    │  │  API  │  Database  │  UI  │  Scheduling  │  State Management    │        │
    │  └─────────────────────────────────────────────────────────────────┘        │
    └─────────────────────────────────────────────────────────────────────────────┘
                                        │
                        ┌───────────────┴───────────────┐
                        ▼                               ▼
    ┌───────────────────────────────┐   ┌───────────────────────────────┐
    │       Work Pool: dev          │   │     Work Pool: production     │
    │        (process)              │   │          (ecs)                │
    └───────────────┬───────────────┘   └───────────────┬───────────────┘
                    │                                   │
                    ▼                                   ▼
    ┌───────────────────────────────┐   ┌───────────────────────────────┐
    │         Worker                │   │     ECS Fargate Tasks         │
    │      (EC2 / Local)            │   │      (Auto-scaling)           │
    └───────────────────────────────┘   └───────────────────────────────┘
    ```

=== "Docker Compose"

    ```
    ┌─────────────────────────────────────────────────────────────────────────────┐
    │                         EC2 INSTANCE                                        │
    │  ┌─────────────────────────────────────────────────────────────────┐        │
    │  │                      Docker Compose                             │        │
    │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │        │
    │  │  │   Prefect   │  │  PostgreSQL │  │   Worker    │              │        │
    │  │  │   Server    │  │             │  │             │              │        │
    │  │  └─────────────┘  └─────────────┘  └─────────────┘              │        │
    │  └─────────────────────────────────────────────────────────────────┘        │
    └─────────────────────────────────────────────────────────────────────────────┘
    ```

=== "ECS + RDS"

    ```
    ┌─────────────────────────────────────────────────────────────────────────────┐
    │                          AWS INFRASTRUCTURE                                 │
    │  ┌─────────────────────────────────────────────────────────────────┐        │
    │  │                   Application Load Balancer                     │        │
    │  └───────────────────────────────┬─────────────────────────────────┘        │
    │                                  │                                          │
    │  ┌───────────────────────────────┴─────────────────────────────────┐        │
    │  │                      ECS Cluster                                │        │
    │  │  ┌─────────────────────────┐  ┌─────────────────────────┐       │        │
    │  │  │   Prefect Server        │  │   Worker Service        │       │        │
    │  │  │   (Multi-AZ)            │  │   (Auto-scaling)        │       │        │
    │  │  └─────────────────────────┘  └─────────────────────────┘       │        │
    │  └─────────────────────────────────────────────────────────────────┘        │
    │                                  │                                          │
    │  ┌───────────────────────────────┴─────────────────────────────────┐        │
    │  │                      RDS PostgreSQL                             │        │
    │  └─────────────────────────────────────────────────────────────────┘        │
    └─────────────────────────────────────────────────────────────────────────────┘
    ```

### Components Deployed

| Component | Description |
|-----------|-------------|
| **Control Plane** | Prefect Cloud, Docker Compose, or ECS + RDS |
| **Work Pools** | `development` (process), `production` (ecs) |
| **Workers** | Process workers for dev, ECS tasks for production |
| **Blocks** | AWS Credentials, Snowflake Connector, S3 Bucket |
| **Terraform** | Infrastructure as code for all components |
| **CI/CD** | GitHub Actions for flow deployment |
| **Alerting** | Automations for Slack and PagerDuty notifications |

## Verification Checklist

Run through this checklist to verify your setup:

### Control Plane

- [ ] Prefect UI is accessible
- [ ] Can authenticate with CLI (`prefect version`)
- [ ] Work pools are visible (`prefect work-pool ls`)

### Workers

- [ ] At least one worker is running
- [ ] Worker is connected to correct work pool
- [ ] Worker shows as "online" in UI

### Deployments

- [ ] Test deployment exists (`prefect deployment ls`)
- [ ] Can trigger a flow run manually
- [ ] Flow completes successfully
- [ ] Logs are visible in UI

### Blocks

- [ ] AWS Credentials block is configured
- [ ] Can fetch secrets from AWS Secrets Manager
- [ ] Snowflake block can connect (if configured)

### CI/CD

- [ ] GitHub Actions workflow is configured
- [ ] Pushing to `main` deploys flows automatically
- [ ] Prefect API key is in AWS Secrets Manager

### Alerting

- [ ] Slack webhook automation is active
- [ ] Test failure triggers notification
- [ ] PagerDuty integration configured (if applicable)

## File Structure

Your project should now look like:

```
data-pipelines/
├── flows/
│   ├── __init__.py
│   └── hello_world.py
├── pipelines/
│   └── __init__.py
├── sources/
│   └── __init__.py
├── utils/
│   ├── __init__.py
│   └── vault_provider.py
├── .dlt/
│   ├── config.toml
│   └── secrets.toml
├── prefect.yaml
├── pyproject.toml
├── uv.lock
└── .github/
    └── workflows/
        └── prefect-deploy.yml

terraform/
├── prefect/                    # Prefect Cloud resources
│   └── config/
│       ├── backend.tf
│       ├── main.tf
│       ├── providers.tf
│       ├── variables.tf
│       ├── terraform.tfvars
│       ├── work_pools.tf
│       └── service_accounts.tf
│
├── prefect-server/             # Docker Compose option
│   ├── config/
│   │   └── ...
│   └── files/
│       ├── docker-compose.yml
│       └── user-data.sh
│
└── prefect-ecs/                # ECS + RDS option
    └── config/
        └── ...
```

## Common Issues

### Worker Not Picking Up Jobs

```sh
# Check worker is running
prefect worker ls --pool your-pool

# Verify API URL
echo $PREFECT_API_URL

# Check worker logs
journalctl -u prefect-worker -f
```

### Flow Deployment Fails

```sh
# Verify deployment config
prefect deployment inspect flow-name/deployment-name

# Check for syntax errors in prefect.yaml
prefect deploy --dry-run
```

### Secrets Not Found

```sh
# Verify secret exists
aws secretsmanager get-secret-value --secret-id your-secret

# Check IAM permissions
aws sts get-caller-identity
```

## Estimated Costs

| Option | Monthly Cost |
|--------|-------------|
| Prefect Cloud (Hobby) | $0 |
| Prefect Cloud (Starter) | $100 |
| Docker Compose | ~$17 |
| ECS + RDS | ~$67 |

Plus worker infrastructure costs (EC2 or additional ECS tasks).

## Summary

You've completed the orchestration section:

- [x] Understood Prefect concepts (flows, tasks, deployments, work pools)
- [x] Chose a deployment option (Cloud, Docker Compose, or ECS)
- [x] Set up the control plane and workers
- [x] Deployed and ran your first flow
- [x] Integrated with AWS Secrets Manager
- [x] Configured CI/CD for automated flow deployment
- [x] Set up alerting for pipeline failures

Your data platform now has a central orchestration layer ready to coordinate all your data workflows.

!!! success "Orchestration Complete"
    You have a production-ready orchestration layer with automated deployment and alerting. Next, you'll build the data pipelines that Prefect will orchestrate — starting with dlt for batch ingestion.

## What's Next

Continue to [Batch Data Ingestion](../batch-data-ingestion/index.md) →
