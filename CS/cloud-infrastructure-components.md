---
topic_type: devops
status: learning
difficulty: intermediate
tags:
  - cs
  - cloud-computing
  - load-balancing
  - cdn
  - dns
  - object-storage
  - secrets-management
  - aws
  - system-design
prerequisites:
  - "[[Cloud Networking VPC]]"
  - "[[Cloud Compute Options]]"
date: 2026-04-29
updated: 2026-06-14
---

# Cloud Infrastructure Components

## Overview

Beyond compute and networking, cloud infrastructure includes load balancers, CDNs, DNS, object storage, and secrets management. These components form the supporting infrastructure that makes applications scalable, performant, secure, and reliable.

## Load Balancers

### What They Do

Load balancers distribute incoming traffic across multiple targets (instances, containers, IP addresses) to ensure high availability, fault tolerance, and efficient resource utilization.

```mermaid
graph TD
    Users[Users] --> LB[Load Balancer]
    LB --> T1[Target 1: Healthy]
    LB --> T2[Target 2: Healthy]
    LB --> T3[Target 3: Healthy]
    LB -.->|Health check fails| T4[Target 4: Unhealthy]
    
    LB --> HC[Health Checks]
    HC --> T1
    HC --> T2
    HC --> T3
    HC --> T4
    
    classDef users fill:#ffcccc,stroke:#cc0000
    classDef lb fill:#ffffcc,stroke:#999900
    classDef healthy fill:#ccffcc,stroke:#006600
    classDef unhealthy fill:#ffcccc,stroke:#cc0000
    classDef hc fill:#ccccff,stroke:#000066
    
    class Users users
    class LB lb
    class T1,T2,T3 healthy
    class T4 unhealthy
    class HC hc
```

### Application Load Balancer (ALB) — Layer 7

Operates at the **application layer** (HTTP/HTTPS). Makes routing decisions based on content.

**Features**:
- **Path-based routing**: `/api/*` → API servers, `/static/*` → S3
- **Host-based routing**: `api.example.com` → API, `www.example.com` → web app
- **WebSocket support**: Real-time bidirectional communication
- **HTTP/2 and gRPC**: Modern protocol support
- **SSL/TLS termination**: Decrypt HTTPS at the load balancer

```mermaid
graph TD
    Client[Client: HTTPS] --> ALB[ALB: TLS Termination]
    
    ALB -->|Path: /api/*| API[API Target Group]
    ALB -->|Path: /static/*| S3Target[S3 Target Group]
    ALB -->|Host: admin.*| Admin[Admin Target Group]
    ALB -->|Default| Web[Web Target Group]
    
    API --> A1[API Server 1]
    API --> A2[API Server 2]
    
    classDef client fill:#ffcccc,stroke:#cc0000
    classDef alb fill:#ffffcc,stroke:#999900
    classDef tg fill:#ccffcc,stroke:#006600
    classDef target fill:#ccccff,stroke:#000066
    
    class Client client
    class ALB alb
    class API,S3Target,Admin,Web tg
    class A1,A2 target
```

**When to use ALB**: HTTP/HTTPS traffic, need content-based routing, microservices with different paths.

### Network Load Balancer (NLB) — Layer 4

Operates at the **transport layer** (TCP/UDP). Makes routing decisions based on IP protocol data.

**Features**:
- **Ultra-low latency**: Microsecond-level processing
- **Static IPs**: One elastic IP per AZ (useful for whitelisting)
- **Preserves source IP**: Backend sees the actual client IP
- **Handles millions of requests per second**: Extreme throughput
- **TCP, UDP, TLS**: Protocol support

| Feature | ALB (L7) | NLB (L4) |
|---------|----------|----------|
| **Layer** | Application (HTTP/HTTPS) | Transport (TCP/UDP) |
| **Routing** | Path, host, headers, methods | IP address, port |
| **Latency** | Milliseconds | Microseconds |
| **Throughput** | Millions of RPS | Tens of millions of RPS |
| **Source IP** | Via X-Forwarded-For header | Preserved natively |
| **Static IP** | No (DNS name only) | Yes (elastic IP per AZ) |
| **WebSocket** | Yes | Yes (TCP passthrough) |
| **SSL termination** | Yes | Yes (TLS listener) |
| **Cost** | ~$22/month + LCU charges | ~$22/month + NLCU charges |

**When to use NLB**: TCP/UDP traffic, extreme performance needs, gaming, IoT, financial trading, or when you need static IPs.

### Health Checks

Load balancers continuously check target health:

```mermaid
sequenceDiagram
    participant LB as Load Balancer
    participant T1 as Target 1
    participant T2 as Target 2

    LB->>T1: GET /health (every 30s)
    T1-->>LB: 200 OK
    LB->>T2: GET /health (every 30s)
    T2-->>LB: 503 Service Unavailable
    
    Note over LB,T2: After 3 consecutive failures
    LB->>LB: Mark Target 2 as unhealthy
    LB->>LB: Stop sending traffic to Target 2
    
    T2->>T2: Recovers
    LB->>T2: GET /health
    T2-->>LB: 200 OK
    
    Note over LB,T2: After 2 consecutive successes
    LB->>LB: Mark Target 2 as healthy
    LB->>T2: Resume traffic
```

## CDN (Content Delivery Network)

### What It Is

A CDN caches your content at **edge locations** — data centers distributed globally, closer to your users. When a user requests content, it's served from the nearest edge location instead of your origin server.

```mermaid
graph TD
    User1[User: London] --> Edge1[Edge: London]
    User2[User: Tokyo] --> Edge2[Edge: Tokyo]
    User3[User: New York] --> Edge3[Edge: New York]
    
    Edge1 -->|Cache miss| Origin[Origin Server<br/>us-east-1]
    Edge2 -->|Cache miss| Origin
    Edge3 -->|Cache hit| Origin
    
    Origin -->|Fetch & cache| Edge1
    Origin -->|Fetch & cache| Edge2
    
    classDef user fill:#ffcccc,stroke:#cc0000
    classDef edge fill:#ccffcc,stroke:#006600
    classDef origin fill:#ffffcc,stroke:#999900
    
    class User1,User2,User3 user
    class Edge1,Edge2,Edge3 edge
    class Origin origin
```

### What to Cache

| Content Type | Cache Strategy | TTL |
|-------------|---------------|-----|
| **Static assets** (CSS, JS, images) | Cache aggressively | 1 year + cache busting |
| **API responses** | Cache with validation | Minutes to hours |
| **HTML pages** | Cache or bypass | Depends on dynamic content |
| **Video/audio** | Cache aggressively (large files) | 1 year |
| **Personalized content** | Don't cache | N/A |
| **Authentication pages** | Never cache | N/A |

### Benefits

| Benefit | Description |
|---------|-------------|
| **Lower latency** | Content served from nearby edge location (10-50ms vs 200-500ms) |
| **Reduced origin load** | Cache hits never reach your origin server |
| **DDoS protection** | CDN absorbs traffic spikes and attacks |
| **Bandwidth savings** | Less data transfer from origin = lower costs |
| **Global reach** | Serve users worldwide without deploying in every region |

### Cache Invalidation

```mermaid
sequenceDiagram
    participant User
    participant CDN
    participant Origin

    User->>CDN: GET /image.jpg
    CDN->>Origin: Cache miss, fetch
    Origin-->>CDN: image.jpg + Cache-Control: max-age=3600
    CDN-->>User: image.jpg
    
    Note over Origin: Image updated
    Origin->>CDN: Invalidate /image.jpg
    CDN->>CDN: Remove from edge caches
    
    User->>CDN: GET /image.jpg
    CDN->>Origin: Cache miss (invalidated), fetch
    Origin-->>CDN: New image.jpg
    CDN-->>User: New image.jpg
```

> [!tip] Cache Busting
> Instead of invalidating caches, use cache busting: append a version hash to filenames (`style.a1b2c3.css`). When the file changes, the URL changes, and the CDN fetches the new version automatically. This is faster and cheaper than cache invalidation.

## DNS (Route 53)

### What It Does

DNS translates human-readable domain names (`example.com`) into IP addresses (`93.184.216.34`). Route 53 is AWS's managed DNS service.

### Routing Policies

| Policy | Behavior | Use Case |
|--------|----------|----------|
| **Simple** | One record, one value | Single resource |
| **Weighted** | Distribute traffic by weight (e.g., 80/20) | A/B testing, gradual migration |
| **Latency-based** | Route to lowest-latency endpoint | Global applications |
| **Geolocation** | Route based on user's geographic location | Content localization, compliance |
| **Geoproximity** | Route based on user location + bias | Traffic shifting between regions |
| **Failover** | Active-passive: primary + secondary | Disaster recovery |
| **Multi-value** | Return multiple IPs, client picks one | Simple load balancing |

```mermaid
graph TD
    User[User Query] --> R53[Route 53]
    
    R53 -->|Latency Policy| US[US Endpoint<br/>50ms]
    R53 -->|Latency Policy| EU[EU Endpoint<br/>120ms]
    R53 -->|Latency Policy| AP[AP Endpoint<br/>200ms]
    
    User -->|From Europe| EU
    
    R53 -->|Failover Policy| Primary[Primary: Healthy]
    R53 -->|Failover Policy| Secondary[Secondary: Standby]
    
    R53 -->|Weighted Policy| V1[Version 1: 90%]
    R53 -->|Weighted Policy| V2[Version 2: 10%]
    
    classDef user fill:#ffcccc,stroke:#cc0000
    classDef dns fill:#ffffcc,stroke:#999900
    classDef endpoint fill:#ccffcc,stroke:#006600
    
    class User user
    class R53 dns
    class US,EU,AP,Primary,Secondary,V1,V2 endpoint
```

### Health Checks

Route 53 can monitor endpoint health and automatically route traffic away from unhealthy endpoints:

- **HTTP/HTTPS/TCP health checks** every 10 or 30 seconds
- **Integration with CloudWatch alarms**
- **Automatic DNS failover** when health check fails

## Object Storage (S3)

### What It Is

Amazon S3 (Simple Storage Service) provides unlimited, highly durable object storage. Data is stored as objects (files) within buckets, with each object having a key (name), data, and metadata.

### Durability and Availability

- **Durability**: 99.999999999% (11 nines) — designed to sustain concurrent loss of data in two facilities
- **Availability**: 99.99% (Standard) — four nines uptime SLA

### Storage Classes

| Class | Access Pattern | Retrieval Time | Cost (per GB/month) | Best For |
|-------|---------------|----------------|-------------------|----------|
| **S3 Standard** | Frequent access | Milliseconds | $0.023 | Active data, websites, mobile apps |
| **S3 Intelligent-Tiering** | Unknown/changing | Milliseconds | $0.023 (auto-moves) | Data with unknown access patterns |
| **S3 Standard-IA** | Infrequent access | Milliseconds | $0.0125 + retrieval fee | Backups, disaster recovery |
| **S3 One Zone-IA** | Infrequent, single AZ | Milliseconds | $0.01 + retrieval fee | Reproducible data, secondary backups |
| **S3 Glacier Instant** | Archive, instant access | Milliseconds | $0.004 | Long-term data needing instant retrieval |
| **S3 Glacier Flexible** | Archive | 3-5 hours | $0.0036 | Compliance archives, backups |
| **S3 Glacier Deep Archive** | Rare archive | 12-48 hours | $0.00099 | Regulatory archives (7+ years) |

### Use Cases

```mermaid
graph TD
    S3[(S3 Bucket)]
    
    S3 --> Static[Static Website Hosting]
    S3 --> Uploads[File Uploads]
    S3 --> Backups[Backups & DR]
    S3 --> DataLake[Data Lake]
    S3 --> Media[Media Storage]
    S3 --> Logs[Log Storage]
    
    Static --> HTML[HTML/CSS/JS]
    Uploads --> UserFiles[User-uploaded files]
    Backups --> DBBackup[Database snapshots]
    DataLake --> Analytics[Analytics processing]
    Media --> Video[Video on demand]
    Logs --> CloudTrail[Audit logs]
    
    classDef s3 fill:#ffffcc,stroke:#999900
    classDef usecase fill:#ccffcc,stroke:#006600
    classDef detail fill:#ccccff,stroke:#000066
    
    class S3 s3
    class Static,Uploads,Backups,DataLake,Media,Logs usecase
    class HTML,UserFiles,DBBackup,Analytics,Video,CloudTrail detail
```

### S3 as Static Website Host

```mermaid
graph LR
    User[User] --> CF[CloudFront CDN]
    CF --> S3[(S3 Bucket<br/>Static Website)]
    
    S3 --> index[index.html]
    S3 --> css[style.css]
    S3 --> js[app.js]
    S3 --> img[images/]
    
    classDef user fill:#ffcccc,stroke:#cc0000
    classDef cdn fill:#ffffcc,stroke:#999900
    classDef s3 fill:#ccffcc,stroke:#006600
    classDef files fill:#ccccff,stroke:#000066
    
    class User user
    class CF cdn
    class S3 s3
    class index,css,js,img files
```

> [!tip] S3 + CloudFront Pattern
> Always put CloudFront in front of S3 for static websites. Benefits: HTTPS (S3 alone doesn't support custom SSL), caching at edge, DDoS protection, and lower S3 request costs.

## Secrets Management

### Why Not Environment Variables or .env Files

| Approach | Problem |
|----------|---------|
| **Hardcoded in code** | Leaked in version control, impossible to rotate |
| **.env files** | Can be committed accidentally, no rotation, no audit trail |
| **Environment variables** | Visible in process listings, logs, error messages |
| **Config files on disk** | Must be managed separately, no encryption at rest by default |

### AWS Secrets Manager vs SSM Parameter Store

| Feature | Secrets Manager | SSM Parameter Store |
|---------|----------------|--------------------|
| **Purpose** | Secrets (passwords, API keys, certificates) | Configuration parameters |
| **Automatic rotation** | Built-in (Lambda-based rotation) | Manual or custom Lambda |
| **Encryption** | KMS encryption | KMS encryption (SecureString) |
| **Cross-account access** | Supported | Supported |
| **Cost** | $0.40/secret/month + $0.05/10K API calls | Free (Standard), $0.05/parameter/month (Advanced) |
| **Secret size limit** | 65 KB | 8 KB (Standard), 8 KB (Advanced) |
| **Versioning** | Automatic versioning with staging labels | Versioning available |
| **Audit trail** | CloudTrail integration | CloudTrail integration |

### When to Use Each

```mermaid
graph TD
    Start{What are you storing?}
    
    Start -->|Database passwords<br/>API keys<br/>TLS certificates| SM[Secrets Manager<br/>Auto-rotation, versioning]
    Start -->|Feature flags<br/>Configuration values<br/>Public keys| SSM[SSM Parameter Store<br/>Free, hierarchical]
    Start -->|Large secrets<br/>> 8 KB| SM
    
    classDef decision fill:#ffffcc,stroke:#999900
    classDef sm fill:#ffcccc,stroke:#cc0000
    classDef ssm fill:#ccffcc,stroke:#006600
    
    class Start decision
    class SM sm
    class SSM ssm
```

### Best Practices

```mermaid
graph TD
    A[Secrets Management] --> B[Never commit secrets]
    A --> C[Use IAM for access control]
    A --> D[Rotate regularly]
    A --> E[Audit access]
    A --> F[Use least privilege]
    
    B --> B1[Use .gitignore, pre-commit hooks]
    B --> B2[Scan repos with git-secrets, trufflehog]
    
    C --> C1[IAM policies, not shared credentials]
    C --> C2[Role-based access per service]
    
    D --> D1[Automated rotation (Secrets Manager)]
    D --> D2[90-day rotation policy]
    
    E --> E1[CloudTrail logging]
    E --> E2[Alert on unusual access patterns]
    
    F --> F1[Each service gets only its secrets]
    F --> F2[No admin-level secret access]
    
    classDef main fill:#ffffcc,stroke:#999900
    classDef practice fill:#ccffcc,stroke:#006600
    classDef detail fill:#ccccff,stroke:#000066
    
    class A main
    class B,C,D,E,F practice
    class B1,B2,C1,C2,D1,D2,E1,E2,F1,F2 detail
```

### Example: Retrieving Secrets in Application

```typescript
// Using AWS SDK v3 to retrieve a secret
import { SecretsManagerClient, GetSecretValueCommand } from "@aws-sdk/client-secrets-manager";

const client = new SecretsManagerClient({ region: "us-east-1" });

async function getDatabaseCredentials(): Promise<{
  username: string;
  password: string;
  host: string;
  port: number;
}> {
  const command = new GetSecretValueCommand({
    SecretId: "prod/database/credentials",
  });

  const response = await client.send(command);
  const secret = JSON.parse(response.SecretString!);

  return {
    username: secret.username,
    password: secret.password,
    host: secret.host,
    port: secret.port,
  };
}

// Cache the secret — don't fetch on every request
let cachedCredentials: ReturnType<typeof getDatabaseCredentials> | null = null;
const CACHE_TTL = 3600; // 1 hour
let cacheExpiry = 0;

async function getCachedCredentials() {
  if (!cachedCredentials || Date.now() > cacheExpiry) {
    cachedCredentials = await getDatabaseCredentials();
    cacheExpiry = Date.now() + CACHE_TTL * 1000;
  }
  return cachedCredentials;
}
```

> [!warning] Secret Rotation Gotcha
> When rotating database passwords, there's a window where the old password is invalid but the application hasn't picked up the new one. Use dual-password support (PostgreSQL 14+) or implement graceful reconnection with secret refresh.

## How Components Work Together

### Request Flow Through Infrastructure

```mermaid
sequenceDiagram
    participant User
    participant DNS as Route 53
    participant CDN as CloudFront
    participant ALB as ALB
    participant App as App Server (Private Subnet)
    participant SM as Secrets Manager
    participant DB as RDS (Isolated Subnet)
    participant S3

    User->>DNS: Resolve example.com
    DNS-->>User: CloudFront IP
    
    User->>CDN: GET /page
    alt Cache hit
        CDN-->>User: Cached page
    else Cache miss
        CDN->>ALB: Forward request
        ALB->>App: Route to healthy target
        App->>SM: Get DB credentials
        SM-->>App: Credentials
        App->>DB: Query data
        DB-->>App: Results
        App-->>ALB: HTML response
        ALB-->>CDN: Response
        CDN-->>User: Response (cached)
    end
    
    User->>CDN: GET /static/logo.png
    CDN-->>User: Cached image (from edge)
    
    App->>S3: Upload user file
    S3-->>App: Upload confirmed
```

## Key Details

> [!warning] Common Pitfalls
> - **ALB without health checks** — traffic sent to unhealthy instances causes errors
> - **CDN without cache headers** — origin server gets hit on every request
> - **S3 bucket public by accident** — always block public access unless intentionally hosting a public website
> - **Secrets in Lambda environment variables** — use Secrets Manager or SSM Parameter Store instead
> - **Single AZ deployment** — always deploy across multiple AZs for high availability
> - **Missing DNS TTL consideration** — high TTL means slow failover; use low TTL (60s) for critical services

> [!tip] Infrastructure Best Practices
> - Use Infrastructure as Code (Terraform, CloudFormation) for all resources
> - Tag everything for cost allocation and management
> - Enable access logging on load balancers, CloudFront, and S3
> - Use WAF (Web Application Firewall) in front of ALB/CloudFront
> - Implement least-privilege IAM policies for every component
> - Test failover scenarios regularly — don't assume DR works until you've tested it

## When to Use

- **System design interviews** — selecting the right infrastructure components for architecture
- **Production deployment** — building resilient, scalable infrastructure
- **Performance optimization** — using CDN and caching to reduce latency
- **Security hardening** — proper secrets management and network isolation

## Related Topics

- [[Cloud Networking VPC]] — load balancers and DNS operate within VPC networking
- [[Cloud Compute Options]] — infrastructure components route traffic to compute resources
- [[Microservices Architecture]] — load balancers and API gateways are essential for microservices
- [[Cloud Cost Optimization]] — CDN caching, right-sizing, and storage class selection reduce costs

## External Links

- [AWS Elastic Load Balancing](https://aws.amazon.com/elasticloadbalancing/)
- [Amazon CloudFront](https://aws.amazon.com/cloudfront/)
- [Amazon Route 53](https://aws.amazon.com/route53/)
- [Amazon S3](https://aws.amazon.com/s3/)
- [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/)
- [AWS Systems Manager Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
