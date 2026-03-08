# Work Pools and Workers Reference

On this page, you will:

- [x] Understand work pool types and when to use each
- [x] Learn how workers execute flows
- [x] Find resources for advanced configurations

!!! note "Work Pools Already Created"
    If you followed [Prefect Cloud Setup](3-prefect-cloud-setup.md), your work pools (`development` and `production`) are already created via Terraform. This page is a reference for understanding how they work and configuring advanced scenarios.

## Overview

Work pools and workers are how Prefect executes your flows. The work pool defines **where** and **how** flows run; workers pick up jobs from pools and execute them.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          PREFECT (Cloud or Server)                          │
│  ┌─────────────────────────────────────────────────────────────────┐        │
│  │                         Work Pools                              │        │
│  │  ┌───────────────┐  ┌───────────────┐                           │        │
│  │  │  development  │  │  production   │                           │        │
│  │  │   (process)   │  │    (ecs)      │                           │        │
│  │  └───────┬───────┘  └───────┬───────┘                           │        │
│  └──────────│──────────────────│───────────────────────────────────┘        │
└─────────────│──────────────────│────────────────────────────────────────────┘
              │                  │
     ┌────────▼────────┐  ┌──────▼──────┐
     │     Worker      │  │   Worker    │
     │     (Local)     │  │   (ECS)     │
     └─────────────────┘  └─────────────┘
```

## Work Pool Types

| Type | Infrastructure | Best For |
|------|---------------|----------|
| `process` | Local process | Development, simple workloads |
| `docker` | Docker containers | Isolated environments |
| `ecs` | AWS ECS Fargate | AWS production, serverless |
| `kubernetes` | Kubernetes pods | K8s environments |

### Process Work Pools

The simplest option - flows run as local processes on the worker machine. This is what the `development` pool uses.

**Pros:** No container overhead, fast startup, simple to debug

**Cons:** Dependencies must be installed on the worker, no isolation between flows

### ECS Work Pools

Flows run as AWS ECS Fargate tasks - serverless containers. This is what the `production` pool uses.

**Pros:** Serverless (scales automatically), AWS-native, no servers to manage

**Cons:** AWS-specific, cold start latency, requires VPC configuration

## Running Workers

### Development Worker (Local)

For local development, start a worker in your terminal:

```sh
# Start a worker for the development pool
prefect worker start --pool development
```

Keep this running while developing flows. The worker will execute any flows deployed to the `development` pool.

### Production Workers

For production, you have two main options:

**Option 1: ECS Push Work Pool (Recommended)**

With a "push" work pool, Prefect Cloud submits tasks directly to ECS - no long-running worker needed. This is the simplest production option.

**Option 2: ECS Worker Service**

Run a worker as an ECS service that polls for work and submits ECS tasks. Use [Prefect's official Terraform module](https://registry.terraform.io/modules/PrefectHQ/ecs-worker/prefect/latest) for this.

!!! info "Production Workers"
    Setting up ECS workers is covered in the [ECS Production Setup](5-self-hosted-ecs-production.md) guide, but for most teams starting out, the development worker is sufficient until you need to scale.

## Deployment Configuration

When deploying flows, you specify which work pool to use:

```yaml
# prefect.yaml
deployments:
  - name: my-flow
    entrypoint: flows/my_flow.py:my_flow
    work_pool:
      name: development  # or production
```

You can also override work pool defaults per-deployment:

```yaml
deployments:
  - name: heavy-etl
    entrypoint: flows/heavy_etl.py:heavy_etl_flow
    work_pool:
      name: production
      job_variables:
        cpu: 1024      # Override default CPU
        memory: 2048   # Override default memory
```

## Monitoring Workers

Check worker status via CLI:

```sh
# List workers for a pool
prefect worker ls --pool development

# Inspect work pool details
prefect work-pool inspect development
```

The Prefect UI also shows:

- Connected workers and their status
- Work pool queue depth
- Running and pending flow runs

## Further Reading

For advanced configurations:

- [Prefect Work Pools Documentation](https://docs.prefect.io/latest/concepts/work-pools/)
- [Prefect Workers Documentation](https://docs.prefect.io/latest/concepts/workers/)
- [ECS Worker Terraform Module](https://registry.terraform.io/modules/PrefectHQ/ecs-worker/prefect/latest)

## Summary

You've learned:

- [x] Work pool types (`process`, `ecs`) and when to use each
- [x] How workers poll work pools and execute flow runs
- [x] Where to find resources for advanced configurations

## What's Next

Continue to deploy your first flow.

Continue to [Your First Flow](7-first-flow.md) →
