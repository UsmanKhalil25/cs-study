---
topic_type: databases
status: learning
difficulty: intermediate
tags:
  - cs
  - databases
  - cloud-computing
  - aws
  - gcp
  - rds
  - high-availability
  - connection-pooling
  - system-design
prerequisites:
  - "[[SQL vs NoSQL Databases]]"
  - "[[vpc]]"
date: 2026-04-29
updated: 2026-06-14
---

# Database Architecture

## Overview

Production database architecture goes far beyond installing a database on a server. It involves decisions about managed vs self-hosted deployment, high availability through multi-AZ setups, read scaling with replicas, connection pooling for serverless workloads, and backup strategies for disaster recovery.

## Managed vs Self-Hosted Databases

### Managed Databases (RDS, Aurora, Cloud SQL)

The cloud provider handles the operational burden:

```mermaid
graph TD
    subgraph Managed Database Service
        Provider[Cloud Provider]
        Provider --> Backup[Automated Backups]
        Provider --> Patch[OS & DB Patching]
        Provider --> HA[High Availability]
        Provider --> Monitor[Monitoring & Alerts]
        Provider --> Scale[Scaling Operations]
    end
    
    You[Your Team] --> App[Application Code]
    App --> DB[(Managed Database)]
    
    classDef provider fill:#ffffcc,stroke:#999900
    classDef you fill:#ccffcc,stroke:#006600
    classDef db fill:#ccccff,stroke:#000066
    
    class Provider,Backup,Patch,HA,Monitor,Scale provider
    class You,App you
    class DB db
```

| Feature | Managed (RDS, Aurora) | Self-Hosted |
|---------|----------------------|-------------|
| **Setup** | Click to create, minutes | Install, configure, tune — hours to days |
| **Backups** | Automated, point-in-time recovery | Manual scripts, cron jobs |
| **Patching** | Automatic or scheduled window | Manual OS and DB updates |
| **High availability** | One-click multi-AZ | Manual replication setup |
| **Scaling** | Vertical: click to resize. Horizontal: read replicas | Manual — provision new server, configure replication |
| **Monitoring** | Built-in metrics, CloudWatch integration | Install and configure monitoring stack |
| **Access** | Limited — no SSH, no superuser (some engines) | Full root access |
| **Customization** | Limited to supported parameters | Full control — custom extensions, kernel tuning |
| **Cost** | Higher per-hour rate | Lower raw compute cost, but add operational cost |
| **Vendor lock-in** | Higher — migration requires effort | Lower — standard database software |

### Cost Comparison Example

**PostgreSQL, db.r6g.large (2 vCPU, 16 GB RAM), us-east-1:**

| Deployment | Monthly Cost | What's Included |
|-----------|-------------|-----------------|
| **RDS (On-Demand)** | ~$175 | DB engine, automated backups, multi-AZ option (+50%) |
| **EC2 + self-managed** | ~$95 | Raw compute only — you manage everything |
| **RDS (1-year Reserved)** | ~$105 | Same as on-demand, with commitment |

**Hidden costs of self-hosted**:
- DBA time: 5-10 hours/week for maintenance at $100-200/hour = $2,000-$8,000/month
- Backup infrastructure and testing
- Monitoring and alerting setup
- Incident response at 3 AM when replication breaks

> [!tip] When to Self-Host
> Self-host when you need: custom database extensions not supported by managed services, specific OS-level tuning, cost savings at very large scale (100+ instances), or compliance requirements that mandate full control.

> [!tip] When to Use Managed
> Use managed for: most production workloads, teams without dedicated DBAs, startups prioritizing speed, and when the operational savings outweigh the premium cost.

### GCP Cloud SQL

Cloud SQL is Google Cloud's managed relational database service for MySQL, PostgreSQL, and SQL Server. It maps closely to RDS conceptually: Google manages backups, patching, maintenance windows, high availability options, monitoring integration, and instance scaling while your application manages schema, queries, indexes, and connection behavior.

```mermaid
flowchart LR
    A[Cloud Run Service or Job] --> B[Cloud SQL Connector / Auth Proxy]
    B --> C[(Cloud SQL Primary)]
    C --> D[(Standby / HA)]
    C --> E[(Read Replica)]
    C --> F[Automated Backups]
    A --> G[Secret Manager]
    G --> B
```

| Concern | Practical Cloud SQL Choice |
|---------|----------------------------|
| Cloud Run connection spikes | Use connector/proxy plus connection pooling in the app or a pooler where appropriate |
| Private database access | Prefer private IP or controlled connector path instead of public allowlists |
| Availability | Use HA/regional configuration for production |
| Read scaling | Add read replicas only when the app tolerates replication lag |
| Schema changes | Run migrations from Cloud Run jobs or CI, not during every service startup |

> [!warning] Serverless Connections
> Cloud Run can scale many container instances quickly. If every instance opens a large database pool, Cloud SQL can run out of connections. Keep pools small, set maximum Cloud Run instances when needed, and monitor connection count.

## Primary/Replica Architecture

### Read Replicas

Read replicas are copies of the primary database that handle read-only queries, offloading read traffic from the primary.

```mermaid
graph TD
    Writers[Write Queries] --> Primary[(Primary DB<br/>Reads + Writes)]
    Readers[Read Queries] --> R1[(Read Replica 1)]
    Readers --> R2[(Read Replica 2)]
    Readers --> R3[(Read Replica 3)]
    
    Primary -.->|Async Replication| R1
    Primary -.->|Async Replication| R2
    Primary -.->|Async Replication| R3
    
    classDef writers fill:#ffcccc,stroke:#cc0000
    classDef readers fill:#ccffcc,stroke:#006600
    classDef primary fill:#ffffcc,stroke:#999900
    classDef replica fill:#ccccff,stroke:#000066
    
    class Writers writers
    class Readers readers
    class Primary primary
    class R1,R2,R3 replica
```

**How replication works**:

```mermaid
sequenceDiagram
    participant App as Application
    participant Primary
    participant Replica

    App->>Primary: WRITE: INSERT INTO orders ...
    Primary->>Primary: Write to WAL (Write-Ahead Log)
    Primary->>Primary: Commit transaction
    Primary-->>App: Success
    
    Primary->>Replica: Stream WAL changes
    Replica->>Replica: Replay WAL changes
    Note over Replica: Replication lag: ms to seconds
    
    App->>Replica: READ: SELECT * FROM orders
    Replica-->>App: Results (may be slightly stale)
```

### Replication Lag

Replication is **asynchronous** — there's always some delay between writes on the primary and their appearance on replicas.

| Cause of Lag | Impact | Mitigation |
|-------------|--------|-----------|
| **Heavy write load** | Replica falls behind | Scale up replica, reduce write volume |
| **Long-running queries on replica** | Replication pauses | Kill long queries, use dedicated analytics replica |
| **Network latency** | Cross-region lag (100ms+) | Accept lag, route reads to local replica |
| **Large transactions** | Single transaction blocks replication | Break into smaller transactions |
| **Schema changes** | DDL operations can cause significant lag | Schedule during low-traffic windows |

> [!warning] Read-Your-Writes Consistency
> If a user writes data and immediately reads it, they might hit a replica that hasn't received the write yet. Solution: route reads to the primary for a short window after a write, or use session-based routing.

### Multi-AZ Deployment (High Availability)

Multi-AZ creates a synchronous standby replica in a different Availability Zone. If the primary fails, automatic failover promotes the standby.

```mermaid
graph TD
    subgraph AZ-A
        Primary[(Primary DB<br/>Reads + Writes)]
    end
    
    subgraph AZ-B
        Standby[(Standby DB<br/>Synchronous Replica)]
    end
    
    App[Application] --> Primary
    Primary -.->|Sync Replication| Standby
    
    Primary -.->|Health Check| Monitor[Multi-AZ Monitor]
    Monitor -.->|Failure detected| Failover[Automatic Failover]
    Failover --> Standby
    Standby -->|Promoted to Primary| App
    
    classDef az fill:#ffffcc,stroke:#999900
    classDef primary fill:#ffcccc,stroke:#cc0000
    classDef standby fill:#ccccff,stroke:#000066
    classDef app fill:#ccffcc,stroke:#006600
    classDef process fill:#ffccff,stroke:#990099
    
    class AZ-A,AZ-B az
    class Primary primary
    class Standby standby
    class App app
    class Monitor,Failover process
```

| Feature | Multi-AZ | Read Replicas |
|---------|----------|--------------|
| **Purpose** | High availability (failover) | Read scaling |
| **Replication** | Synchronous | Asynchronous |
| **Readable** | No (standby is not accessible) | Yes |
| **Automatic failover** | Yes (60-120 seconds) | Manual promotion |
| **Cost** | ~2x primary cost | Additional cost per replica |
| **Cross-region** | No (same region, different AZ) | Yes (cross-region replicas supported) |

**Combining both**: Production systems often use Multi-AZ for HA plus read replicas for read scaling.

## Connection Pooling

### The Problem

Each database connection consumes memory and CPU on the database server. Serverless functions (Lambda) are especially problematic because each invocation may open a new connection.

```mermaid
graph TD
    subgraph Without Connection Pooling
        L1[Lambda 1] --> DB1[(DB: Connection 1)]
        L2[Lambda 2] --> DB2[(DB: Connection 2)]
        L3[Lambda 3] --> DB3[(DB: Connection 3)]
        L4[Lambda 4] --> DB4[(DB: Connection 4)]
        L5[Lambda 5] --> DB5[(DB: Connection 5)]
        LN[Lambda N] --> DBN[(DB: Connection N)]
    end
    
    classDef lambda fill:#ccffcc,stroke:#006600
    classDef db fill:#ffcccc,stroke:#cc0000
    
    class L1,L2,L3,L4,L5,LN lambda
    class DB1,DB2,DB3,DB4,DB5,DBN db
```

**Result**: 1,000 concurrent Lambda invocations = 1,000 database connections. PostgreSQL default max_connections = 100. **Database crashes.**

### The Solution: Connection Pooling

A connection pool maintains a pool of reusable database connections. Multiple clients share a smaller number of connections to the database.

```mermaid
graph TD
    subgraph With Connection Pooling
        L1[Lambda 1] --> Pool[(Connection Pool<br/>RDS Proxy / PgBouncer)]
        L2[Lambda 2] --> Pool
        L3[Lambda 3] --> Pool
        L4[Lambda 4] --> Pool
        LN[Lambda N] --> Pool
        
        Pool --> C1[Connection 1]
        Pool --> C2[Connection 2]
        Pool --> C3[Connection 3]
        Pool --> C4[Connection 4]
        Pool --> C5[Connection 5]
        
        C1 --> DB[(Database<br/>5 connections)]
        C2 --> DB
        C3 --> DB
        C4 --> DB
        C5 --> DB
    end
    
    classDef lambda fill:#ccffcc,stroke:#006600
    classDef pool fill:#ffffcc,stroke:#999900
    classDef conn fill:#ccccff,stroke:#000066
    classDef db fill:#ffcccc,stroke:#cc0000
    
    class L1,L2,L3,L4,LN lambda
    class Pool pool
    class C1,C2,C3,C4,C5 conn
    class DB db
```

### PgBouncer vs RDS Proxy

| Feature | PgBouncer | RDS Proxy |
|---------|-----------|-----------|
| **Type** | Open-source, self-managed | AWS managed service |
| **Database support** | PostgreSQL only | PostgreSQL, MySQL, Aurora |
| **Pooling modes** | Transaction, Session, Statement | Transaction only |
| **Setup** | Deploy on EC2 or container | Enable on RDS instance |
| **Cost** | EC2 cost only | ~$0.0153 per vCPU-hour |
| **IAM auth** | No | Yes |
| **Automatic scaling** | Manual | Automatic |
| **Failover handling** | Manual reconfiguration | Automatic |

### Pooling Modes

```mermaid
graph TD
    A[Pooling Modes] --> B[Session Pooling]
    A --> C[Transaction Pooling]
    A --> D[Statement Pooling]
    
    B --> B1[One connection per client session]
    B --> B2[Connection held for entire session]
    B --> B3[Safe for all use cases]
    B --> B4[More connections needed]
    
    C --> C1[Connection per transaction]
    C --> C2[Released after COMMIT/ROLLBACK]
    C --> C3[Most efficient for serverless]
    C --> C4[Cannot use session-level features]
    
    D --> D1[Connection per SQL statement]
    D --> D2[Maximum efficiency]
    D --> D3[Breaks multi-statement transactions]
    D --> D4[Rarely used in practice]
    
    classDef main fill:#ffffcc,stroke:#999900
    classDef mode fill:#ccffcc,stroke:#006600
    classDef detail fill:#ccccff,stroke:#000066
    
    class A main
    class B,C,D mode
    class B1,B2,B3,B4,C1,C2,C3,C4,D1,D2,D3,D4 detail
```

> [!warning] Transaction Pooling Limitations
> With transaction pooling, you cannot use PostgreSQL session-level features like `SET` commands, prepared statements, or advisory locks across transactions. Each transaction may run on a different connection.

## Backups and Recovery

### Backup Types

| Type | Description | Recovery |
|------|-------------|----------|
| **Automated backups** | Daily full backup + continuous transaction logs | Point-in-time recovery within retention period |
| **Manual snapshots** | User-initiated full backup | Restore to a new instance |
| **Cross-region snapshots** | Copy of snapshot in another region | Disaster recovery |
| **Export to S3** | Export snapshot data to S3 | Long-term archival, analytics |

### Point-in-Time Recovery (PITR)

```mermaid
timeline
    title Point-in-Time Recovery Timeline
    
    section Backup Window
        Full Backup : Daily automated backup
                     at 03:00 UTC
    section Continuous
        WAL Archiving : Transaction logs
                       archived continuously
        Recovery : Can restore to ANY
                  second between
                  oldest backup and now
```

**How PITR works**:

1. Start from the most recent full backup
2. Replay transaction logs (WAL) up to the desired timestamp
3. Result: database state at that exact moment

> [!tip] Backup Best Practices
> - Set retention period to at least 7 days (35 days for production)
> - Test restore procedures regularly — backups you haven't tested are not backups
> - Use cross-region snapshots for disaster recovery
> - Monitor backup completion — failed backups are silent failures
> - Document your recovery runbook — what to do when you need to restore

### Recovery Time Objective (RTO) and Recovery Point Objective (RPO)

| Metric | Definition | Multi-AZ | Read Replica | Backups |
|--------|-----------|----------|-------------|---------|
| **RTO** | How long to recover | 60-120 seconds | Minutes (manual promotion) | Hours (restore from backup) |
| **RPO** | How much data loss | Zero (sync replication) | Seconds (async lag) | Up to last backup |

## Database Architecture Patterns

### Single-Instance (Development/Testing)

```mermaid
graph LR
    App[Application] --> DB[(Single DB Instance)]
    
    classDef app fill:#ccffcc,stroke:#006600
    classDef db fill:#ffcccc,stroke:#cc0000
    
    class App app
    class DB db
```

### Production: Multi-AZ + Read Replicas

```mermaid
graph TD
    subgraph AZ-A
        Writers[Write Traffic] --> Primary[(Primary<br/>AZ-A)]
    end
    
    subgraph AZ-B
        Primary -.->|Sync| Standby[(Standby<br/>AZ-B)]
        Readers1[Read Traffic] --> RR1[(Read Replica 1<br/>AZ-B)]
    end
    
    subgraph AZ-C
        Readers2[Read Traffic] --> RR2[(Read Replica 2<br/>AZ-C)]
    end
    
    Pool[(Connection Pool)] --> Primary
    Pool --> RR1
    Pool --> RR2
    
    classDef az fill:#ffffcc,stroke:#999900
    classDef primary fill:#ffcccc,stroke:#cc0000
    classDef standby fill:#ccccff,stroke:#000066
    classDef replica fill:#ccffcc,stroke:#006600
    classDef pool fill:#ffccff,stroke:#990099
    
    class AZ-A,AZ-B,AZ-C az
    class Primary primary
    class Standby standby
    class RR1,RR2 replica
    class Pool pool
```

### Global: Multi-Region with Cross-Region Replicas

```mermaid
graph TD
    subgraph Region: us-east-1
        PrimaryUS[(Primary<br/>us-east-1)]
        RR_US[(Read Replica<br/>us-east-1)]
    end
    
    subgraph Region: eu-west-1
        RR_EU[(Read Replica<br/>eu-west-1)]
    end
    
    subgraph Region: ap-southeast-1
        RR_AP[(Read Replica<br/>ap-southeast-1)]
    end
    
    PrimaryUS -.->|Cross-region async| RR_EU
    PrimaryUS -.->|Cross-region async| RR_AP
    PrimaryUS -.->|Sync| RR_US
    
    classDef region fill:#ffffcc,stroke:#999900
    classDef primary fill:#ffcccc,stroke:#cc0000
    classDef replica fill:#ccffcc,stroke:#006600
    
    class Region,Region,Region region
    class PrimaryUS primary
    class RR_US,RR_EU,RR_AP replica
```

> [!warning] Cross-Region Replication Lag
> Cross-region replicas have higher latency (50-200ms) due to physical distance. Replication lag can be several seconds. Only use for read scaling in distant regions, not for failover.

## Key Details

> [!warning] Common Pitfalls
> - **No connection pooling with serverless** — Lambda functions will exhaust database connections
> - **Read replicas for HA** — read replicas are NOT a high availability solution; use Multi-AZ
> - **Ignoring replication lag** — applications assuming read-after-write consistency will break
> - **Untested backups** — the only backup that matters is the one you've successfully restored
> - **Single-AZ in production** — AZ outages happen; always use Multi-AZ for production databases
> - **Oversized instances** — start smaller and monitor; you can always scale up

> [!tip] Database Architecture Best Practices
> - Use managed databases unless you have a specific reason not to
> - Enable Multi-AZ for all production databases
> - Add read replicas when read traffic exceeds 60-70% of total traffic
> - Always use connection pooling, especially with serverless
> - Set backup retention to at least 7 days, test restores quarterly
> - Monitor replication lag and set alerts
> - Use parameter groups to tune database settings for your workload
> - Encrypt databases at rest and in transit

## Connection Pool Exhaustion — Troubleshooting

Connection pool exhaustion is one of the most common production database incidents. Symptoms: `too many connections` errors, slow queries that time out waiting for a connection, cascading service failures.

### Diagnosis

```sql
-- PostgreSQL: see current connections by state and client
SELECT client_addr, state, COUNT(*) as count, wait_event_type, wait_event
FROM pg_stat_activity
GROUP BY client_addr, state, wait_event_type, wait_event
ORDER BY count DESC;

-- See max allowed connections
SHOW max_connections;

-- Current active vs idle
SELECT state, count(*) FROM pg_stat_activity GROUP BY state;
```

### Common Causes and Fixes

| Symptom | Likely Cause | Fix |
|---|---|---|
| Many `idle` connections | App not returning connections to pool | Verify `pool.release()` or use `async using` |
| Many `idle in transaction` | Long transactions left open | Set `statement_timeout` / `idle_in_transaction_session_timeout` |
| Spike on deploy/startup | All instances connect simultaneously | Add connection delay jitter at app startup |
| Steady climb over time | Connection leak (acquired, never released) | Add connection pool logging; monitor `pool.waitingCount` |
| Lambda exhausting connections | No pooler between Lambda and DB | Add RDS Proxy or PgBouncer in front of RDS |

### PostgreSQL Limits to Set

```sql
-- Set in RDS parameter group or postgresql.conf
-- Limit connections per role (prevent one service from taking all slots)
ALTER ROLE app_user CONNECTION LIMIT 50;

-- Kill idle-in-transaction sessions after 30s
ALTER DATABASE mydb SET idle_in_transaction_session_timeout = '30s';

-- Kill queries running longer than 60s
ALTER DATABASE mydb SET statement_timeout = '60s';
```

### Topology: Read Replica + PgBouncer

```mermaid
flowchart LR
    App --> PB[PgBouncer\ntransaction mode]
    PB -->|writes| Primary[(Primary RDS)]
    PB -->|reads| R1[(Read Replica 1)]
    PB -->|reads| R2[(Read Replica 2)]
    Primary -.->|async replication| R1
    Primary -.->|async replication| R2
```

Route read-heavy queries to replicas using PgBouncer or application-level routing. Keep write connection count low to avoid primary bottleneck.

## When to Use

- **System design interviews** — designing database architecture for scalable applications
- **Production planning** — setting up databases for reliability and performance
- **Serverless applications** — connection pooling is essential for Lambda + RDS
- **Disaster recovery planning** — understanding RTO/RPO and backup strategies

## Related Topics

- [[SQL vs NoSQL Databases]] — database engine selection
- [[compute]] — database compute sizing and deployment
- [[aws]] — RDS, EC2, ECS, and common AWS deployment primitives
- [[vpc]] — databases run in isolated subnets
- [[infrastructure]] — secrets management for database credentials
- [[cost]] — database cost optimization strategies

## External Links

- [AWS RDS Documentation](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/)
- [Google Cloud SQL Overview](https://cloud.google.com/sql/docs/introduction)
- [Amazon Aurora Documentation](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/)
- [RDS Proxy Documentation](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.html)
- [PgBouncer Documentation](https://www.pgbouncer.org/)
- [PostgreSQL Replication](https://www.postgresql.org/docs/current/runtime-config-replication.html)
- [Database Reliability Engineering (book)](https://www.oreilly.com/library/view/database-reliability-engineering/9781491925942/)
- [Designing Data-Intensive Applications (book)](https://dataintensive.net/)
