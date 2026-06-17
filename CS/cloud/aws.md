---
topic_type: devops
status: learning
difficulty: beginner
tags:
  - cs
  - aws
  - cloud-computing
  - ec2
  - ecs
  - rds
  - system-design
prerequisites:
  - "[[vpc]]"
  - "[[compute]]"
date: 2026-06-14
updated: 2026-06-14
---

# AWS Basics

## Overview

AWS is a cloud platform built from composable services: networking, compute, storage, databases, identity, observability, and deployment primitives. The practical skill is knowing which managed service owns which responsibility, and how requests move through VPC, load balancing, compute, data, and IAM boundaries.

## Core Mental Model

```mermaid
flowchart TD
    A[User] --> B[Route 53 / DNS]
    B --> C[CloudFront or Load Balancer]
    C --> D[VPC Public Subnets]
    D --> E[ALB]
    E --> F{Compute}
    F --> G[EC2 Instances]
    F --> H[ECS Service]
    F --> I[Lambda]
    H --> J[ECR Container Image]
    G --> K[(EBS Volume)]
    H --> L[(RDS / Aurora)]
    I --> L
    H --> M[S3]
    H --> N[Secrets Manager]
    H --> O[CloudWatch Logs/Metrics]
```

Start with the network boundary: most production resources live inside a **VPC**. Public subnets expose load balancers or NAT gateways; private subnets run application tasks, instances, and databases. IAM controls who or what can call AWS APIs; security groups control network reachability.

## Core Services

| Service | What It Is | Use It For |
|---------|------------|------------|
| **EC2** | Virtual machines in AWS | Full OS control, legacy apps, custom runtimes, predictable long-running workloads |
| **ECS** | Managed container orchestration | Running Docker services and batch tasks without managing Kubernetes |
| **Fargate** | Serverless compute engine for ECS | Running containers without managing EC2 capacity |
| **RDS** | Managed relational databases | PostgreSQL, MySQL, MariaDB, SQL Server, Oracle, Db2 with backups/patching/HA options |
| **S3** | Object storage | Static assets, uploads, backups, data lake files |
| **ALB / NLB** | Managed load balancers | Routing HTTP traffic or TCP/UDP traffic to services |
| **Route 53** | DNS service | Domains, DNS records, health-check-based routing |
| **CloudWatch** | Logs, metrics, alarms | Observability and operational alerts |
| **Secrets Manager** | Managed secrets storage | Database passwords, API keys, rotation workflows |
| **IAM** | Identity and access management | Least-privilege permissions for users, roles, and services |

## ECS Basics

ECS runs containers using **task definitions**, **tasks**, **services**, and **clusters**.

```mermaid
flowchart LR
    A[Dockerfile] --> B[Container Image]
    B --> C[ECR Repository]
    C --> D[Task Definition]
    D --> E{Run Mode}
    E -->|Long-running| F[ECS Service]
    E -->|One-off work| G[ECS Task]
    F --> H[ALB Target Group]
    F --> I[Auto Scaling]
    G --> J[Batch / Scheduled Work]
```

| ECS Term | Meaning |
|----------|---------|
| **Cluster** | Logical place where ECS runs workloads |
| **Task definition** | Blueprint: image, CPU, memory, ports, env vars, IAM role, logging |
| **Task** | One running copy of a task definition |
| **Service** | Keeps N copies of a task running and replaces failed tasks |
| **Launch type** | Fargate for serverless capacity, EC2 for self-managed instances |
| **ECR** | AWS container registry for Docker images |

Use **ECS service** for APIs and workers that should stay running. Use **one-off tasks** for migrations, scripts, and scheduled jobs. Use **Fargate** when you want simpler operations; use **ECS on EC2** when you need more control over instance types, GPUs, custom agents, or lower cost at high steady utilization.

## EC2 Basics

EC2 is the raw VM primitive. You choose an AMI, instance type, subnet, security group, IAM role, storage, and bootstrap script.

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant EC2
    participant VPC
    participant EBS
    participant CW as CloudWatch

    Dev->>EC2: Launch instance from AMI
    EC2->>VPC: Attach subnet + security group
    EC2->>EBS: Attach root volume
    EC2->>EC2: Run user data / boot app
    EC2->>CW: Emit logs and metrics
```

EC2 gives maximum control but also maximum responsibility: patching, hardening, deployment automation, log shipping, process supervision, backups, and autoscaling. Use Auto Scaling Groups for fleets rather than hand-managed instances.

## RDS Basics

RDS is the default AWS choice for relational databases unless you need a specialized database or full self-managed control.

| RDS Feature | Why It Matters |
|-------------|----------------|
| Automated backups | Enables point-in-time restore within retention window |
| Multi-AZ | Provides high availability through synchronous standby/failover |
| Read replicas | Scales read traffic, but replicas can lag |
| Parameter groups | Database engine tuning without editing raw config files |
| Security groups | Restrict database network access to app subnets/security groups |
| RDS Proxy | Helps pool connections for serverless or bursty apps |

> [!warning] RDS Is Managed, Not Magic
> You still own schema design, indexes, query performance, migrations, connection pooling, access control, and backup restore testing.

## Common Deployment Shape

```mermaid
flowchart TD
    A[Git Push] --> B[CI Build]
    B --> C[Docker Image]
    C --> D[ECR]
    D --> E[ECS Service Deploy]
    E --> F[ALB Routes Traffic]
    F --> G[Tasks in Private Subnets]
    G --> H[(RDS in Isolated Subnets)]
    G --> I[S3]
    G --> J[Secrets Manager]
    G --> K[CloudWatch]
```

This shape is the AWS equivalent of many Docker-to-managed-container deployments: build an image, push it to a registry, deploy it to a managed container service, route traffic through a load balancer, and connect to managed data/secrets services.

## Code Examples

### Fetch a Secret (TypeScript)

```typescript
import { SecretsManagerClient, GetSecretValueCommand } from "@aws-sdk/client-secrets-manager";

const client = new SecretsManagerClient({ region: "us-east-1" });

async function getSecret(secretName: string): Promise<string> {
  const response = await client.send(
    new GetSecretValueCommand({ SecretId: secretName })
  );
  return response.SecretString ?? "";
}
```

### Upload to S3 (TypeScript)

```typescript
import { S3Client, PutObjectCommand, GetObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

const s3 = new S3Client({ region: "us-east-1" });

async function uploadFile(bucket: string, key: string, body: Buffer): Promise<void> {
  await s3.send(new PutObjectCommand({ Bucket: bucket, Key: key, Body: body }));
}

async function getPresignedUrl(bucket: string, key: string): Promise<string> {
  return getSignedUrl(s3, new GetObjectCommand({ Bucket: bucket, Key: key }), {
    expiresIn: 3600,
  });
}
```

### Run an ECS Task (TypeScript)

```typescript
import { ECSClient, RunTaskCommand } from "@aws-sdk/client-ecs";

const ecs = new ECSClient({ region: "us-east-1" });

async function runMigration(cluster: string, taskDef: string, subnet: string): Promise<void> {
  await ecs.send(new RunTaskCommand({
    cluster,
    taskDefinition: taskDef,
    launchType: "FARGATE",
    networkConfiguration: {
      awsvpcConfiguration: {
        subnets: [subnet],
        assignPublicIp: "DISABLED",
      },
    },
    overrides: {
      containerOverrides: [{ name: "app", command: ["node", "migrate.js"] }],
    },
  }));
}
```

### EC2 Instance Types — Quick Reference

| Family | Use Case | Examples |
|--------|----------|---------|
| `t4g` / `t3` | Dev/test, burstable workloads | `t3.micro`, `t4g.small` |
| `m7g` / `m6i` | General-purpose production | `m6i.xlarge`, `m7g.2xlarge` |
| `c7g` / `c6i` | CPU-intensive (encoding, API) | `c6i.2xlarge` |
| `r7g` / `r6i` | Memory-intensive (Redis, caches) | `r6i.large` |
| `p4d` / `g5` | ML training, GPU workloads | `g5.xlarge` |

**Graviton (`g`/`a` suffix)** — ARM-based, ~20% cheaper for same performance. Use for new greenfield services.

### S3 Storage Classes

| Class | Retrieval | Cost | Use For |
|-------|-----------|------|---------|
| Standard | Instant | High | Active assets, CDN origin |
| Intelligent-Tiering | Instant | Auto-optimized | Unknown access patterns |
| Standard-IA | Instant | Lower | Backups, infrequent reads |
| Glacier Instant | Instant | Low | Archives with rare access |
| Glacier Flexible | Minutes–hours | Very low | Long-term cold archives |
| Deep Archive | Hours | Lowest | Compliance retention |

Set lifecycle rules to automatically transition objects: `Standard → Standard-IA (30d) → Glacier (90d)`.

## When to Use

- **EC2** when you need OS-level control, long-running stable capacity, custom networking/agents, or legacy deployment patterns.
- **ECS Fargate** when you want to run Docker services with low operational overhead.
- **ECS on EC2** when you have high steady container utilization or need instance-level control.
- **RDS** when you need a managed relational database with backups, failover options, and familiar SQL engines.
- **S3** for object files, static assets, backups, and large blobs that do not belong in a database.
- **Secrets Manager** for credentials that should be versioned, access-controlled, and rotated.

## Related Topics

- [[compute]] — VM, container, and serverless compute tradeoffs
- [[vpc]] — VPC, subnets, gateways, route tables, security groups
- [[infrastructure]] — load balancers, CDN, DNS, object storage, secrets
- [[architecture]] — managed databases, replicas, backups, and pooling
- [[cost]] — pricing models, right-sizing, and cost traps

## External Links

- [Amazon ECS Developer Guide](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html)
- [Amazon EC2 User Guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html)
- [Amazon RDS User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Welcome.html)
- [AWS Fargate for ECS](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Fargate.html)
- [Amazon S3 User Guide](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)
- [AWS Secrets Manager User Guide](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)
