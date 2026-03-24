# AWS Core Services Changelog

> Baseline: January 2021. Tracking significant changes to Tier 1, 2, and 3 services.
> Last updated: March 2026

---

## How to Read This

- Organized by service tier, then by service
- Each entry includes the year in parentheses
- 🆕 = New capability | 🔄 = Behavior change | ⚠️ = Deprecation or breaking change | 💰 = Pricing change

---

## Tier 1 — Foundation

### IAM

- 🆕 Access Analyzer generates policies from CloudTrail activity (2021)
- 🆕 IAM Roles Anywhere — temporary credentials for on-premises workloads via X.509 certificates (2022)
- 🆕 IAM Identity Center (rebranded from AWS SSO) (2022)
- 🆕 Access Analyzer detects unused permissions (2023)
- 🔄 Root user sessions centrally managed from org management account (2023)
- 🆕 Resource control policies (RCPs) — SCPs but for resources (2024)
- 🆕 Declarative policies for org-wide baseline enforcement (2024)
- 🔄 Access Analyzer recommendations integrated into console policy editor (2025)

### VPC

- 🆕 VPC IPAM — centralized IP address management across accounts (2021)
- 🆕 Network Access Analyzer — verify network segmentation (2021)
- 🆕 VPC Lattice (preview) — application-layer networking across VPCs (2022)
- 🆕 Verified Access — zero-trust app access without VPN (2022)
- 🔄 Security group referencing across peered VPCs (2022)
- 💰 Public IPv4 address charges announced — $0.005/hr per address (2022, effective Feb 2024)
- 🆕 VPC Lattice GA with built-in auth and observability (2023)
- 🔄 VPC Block Public Access — account-level internet traffic control (2024)
- 🆕 VPC Lattice adds TCP/TLS support (2024)
- 🆕 Security group VPC associations — share SGs across VPCs (2024)
- 🔄 VPC IPAM adds automatic CIDR conflict detection (2025)

### S3

- 🔄 S3 Object Ownership — bucket owner enforces ownership, disabling ACLs (2021)
- 🆕 S3 Intelligent-Tiering adds Archive and Deep Archive tiers (2021)
- 🆕 S3 Multi-Region Access Points (2021)
- 🔄 Block Public Access becomes default on new buckets (2022)
- 🔄 ACLs disabled by default on new buckets (2022)
- 🆕 S3 Glacier Instant Retrieval storage class (2022)
- 💰 No charge for DELETE and lifecycle transition requests (2022)
- 🆕 S3 Express One Zone — single-digit ms latency storage class (2023)
- 🆕 Mountpoint for S3 — mount buckets as local file systems (2023)
- 🔄 Default encryption switches to SSE-S3 for all new objects (2023)
- 🆕 S3 Access Grants — map permissions to corporate directory identities (2023)
- 🆕 S3 Tables — managed Apache Iceberg tables in S3 (2024)
- 🆕 S3 Metadata — automatic metadata extraction (2024)
- 🆕 Conditional writes (if-none-match) — prevents overwrites without locking (2024)
- 🔄 S3 Express One Zone expands to additional regions (2025)
- 🆕 S3 Tables adds automated compaction and snapshot management (2025)

### CloudWatch

- 🆕 Metrics Insights — SQL-like query language for metrics (2021)
- 🆕 CloudWatch Evidently — feature flags and A/B testing (2021)
- 🆕 CloudWatch RUM (Real User Monitoring) (2021)
- 🆕 Cross-account observability (2022)
- 🆕 Application Signals — auto-discover and monitor app health with SLOs (2023)
- 🆕 Logs anomaly detection (2023)
- 💰 Infrequent Access log class at 50% lower ingestion cost (2023)
- 🆕 Database Insights — unified monitoring for RDS/Aurora (2024)
- 🆕 Natural language query for Logs Insights (2024)
- 🔄 Alarms support account-level suppression windows (2024)
- 🆕 CloudWatch investigator — AI-assisted root cause analysis (2025)
- 🔄 Application Signals expands to EKS and ECS workloads (2025)

### EC2

- 🆕 Graviton3 processors announced (2021)
- 🆕 Auto Scaling warm pools — pre-initialized instances (2021)
- 🔄 EBS gp3 becomes recommended default over gp2 (2021)
- 🆕 Graviton3 instances GA — C7g, M7g, R7g (2022)
- 🆕 Instance Connect Endpoint — SSH/RDP without public IPs or bastions (2022)
- ⚠️ EC2-Classic fully retired (2022)
- 🆕 M7i, C7i (Intel Sapphire Rapids), M7a, C7a (AMD Genoa) (2023)
- 💰 Public IPv4 charge active: $0.005/hr per address (2024)
- 🆕 Graviton4 instances GA — M8g, C8g, R8g, X8g (2024)
- 🆕 P5 instances (NVIDIA H100) for ML training (2024)
- 🆕 Capacity Blocks for ML — reserve GPU instances for defined windows (2024)
- 🆕 P5e instances (NVIDIA H200) (2025)
- 🆕 Trn2 instances (AWS Trainium2) for cost-efficient ML training (2025)

### Route 53 / ELB

- 🆕 ALB supports TLS 1.3 (2021)
- 🆕 Route 53 Resolver DNS Firewall (2021)
- 🆕 ALB weighted target groups — native canary deployments (2022)
- 🆕 NLB supports security groups (2022)
- 🆕 ALB mutual TLS (mTLS) support (2023)
- 🆕 Route 53 Profiles — share DNS configs across VPCs and accounts (2023)
- 🔄 NLB cross-zone load balancing configurable per target group (2024)
- 🔄 ALB improved connection draining controls (2025)

---

## Tier 2 — Architecture Dependent

### RDS / Aurora

- 🆕 RDS Custom — managed RDS with OS-level access (2021)
- 🔄 RDS Proxy GA for PostgreSQL and MySQL (2021)
- 🆕 Aurora Serverless v2 GA — sub-second scaling in 0.5 ACU increments (2022)
- 🆕 RDS Blue/Green Deployments — managed switchover for upgrades (2022)
- ⚠️ Aurora Serverless v1 enters deprecation path (2022)
- 🆕 Aurora Limitless Database (preview) — horizontal write scaling (2023)
- 🆕 Aurora I/O-Optimized — no per-I/O charges (2023)
- 🔄 Blue/Green Deployments supports Aurora (2023)
- 💰 RDS Extended Support charges for past-EOL engine versions (2024)
- 🆕 Aurora Limitless Database GA (2024)
- 🆕 RDS auto-manages credentials via Secrets Manager at creation (2024)
- 🔄 Aurora Serverless v2 minimum ACU lowered to 0 — scale to zero (2025)

### Lambda

- 🆕 Container image support up to 10 GB (2021)
- 💰 Billing granularity changes to 1ms (from 100ms) (2021)
- 🆕 Graviton2 (arm64) support (2021)
- 🆕 SnapStart (Java) — near-zero cold starts (2022)
- 🆕 Function URLs — built-in HTTPS endpoint without API Gateway (2022)
- 🔄 Max memory increases to 10 GB (2022)
- 🔄 Response streaming for Node.js (2023)
- 🆕 SnapStart expands to Python and .NET (2023)
- ⚠️ Python 3.7, Node.js 14 runtimes deprecated (2023)
- 🆕 12x faster function scaling (2024)
- 🆕 SnapStart for ARM64/Graviton (2024)
- 🆕 Python 3.13, Node.js 22 support (2024)
- 🆕 Ruby 3.4 support (2025)

### ECS / EKS

- 🆕 ECS Anywhere — run ECS tasks on-premises (2021)
- 🆕 EKS Anywhere — run EKS on your own infrastructure (2021)
- 🔄 Fargate supports ephemeral storage up to 200 GB (2021)
- 🆕 ECS Service Connect — built-in service mesh (2022)
- 🆕 EKS Pod Identity (preview) (2022)
- 🆕 EKS Pod Identity GA (2023)
- 🆕 EKS Auto Mode — AWS manages nodes, scaling, and upgrades entirely (2024)
- 🆕 ECS support for EBS volume attachments to tasks (2024)
- 🔄 EKS extended K8s version support (14 → 26 months) (2024)
- 💰 EKS extended version support costs extra after standard window (2024)
- 🔄 EKS Auto Mode expands to additional regions (2025)

### Secrets Manager

- 🆕 Cross-region secret replication (2021)
- 🆕 Batch retrieval of secrets (BatchGetSecretValue) (2022)
- 💰 Pricing simplified — per-secret/month + per-10K API calls (2022)
- 🆕 Integration with AWS Config for compliance tracking (2023)
- 🔄 Improved cross-account sharing via resource policies (2024)

### ACM

- 🆕 Private CA short-lived certificates (2021)
- 🔄 Supports ECDSA P-384 and RSA 3072/4096 key types (2022)
- 🔄 Renewal window extends to 90 days before expiry (2023)
- 🔄 Certificates default to 90-day validity (2024)
- 🆕 Automated renewal for imported certificates with DNS validation (2024)

### KMS

- 🆕 Multi-Region KMS keys (2021)
- 🆕 HMAC key support (2021)
- 🆕 External key store (XKS) — use keys from your own HSM (2022)
- 🆕 Automatic key rotation now configurable: 90 days to 7 years (2023)
- 🔄 Key deletion waiting period can be set to 7 days (from 30) (2023)
- 🆕 ML-KEM post-quantum key encapsulation support (2024)
- 🔄 Post-quantum cryptography expands to additional operations (2025)

---

## Tier 3 — Use-Case Specific

### SQS

- 🆕 High-throughput FIFO queues — up to 3,000 msg/sec with batching (2021)
- 🔄 Dead-letter queue redrive from console (2021)
- 🔄 DLQ redrive available via API (2023)
- 🔄 Default encryption with SQS-managed keys for new queues (2023)
- 🔄 FIFO throughput increases to 70,000 msg/sec with batching (2024)
- 🔄 SQS-managed key encryption becomes default for all new queues (2025)

### SNS

- 🆕 FIFO topics — ordered, deduplicated pub/sub (2021)
- 🆕 Payload-based message filtering (filter on body, not just attributes) (2022)
- 🔄 Message archiving and replay via Kinesis Data Firehose (2023)
- 🆕 In-place encryption with customer-managed KMS keys (2024)
- 🔄 Delivery logging improvements for debugging (2025)

### DynamoDB

- 🆕 PartiQL — SQL-compatible query language (2021)
- 🆕 Kinesis Data Streams integration for change data capture (2021)
- 💰 On-demand pricing reduced up to 50% in some regions (2021)
- 🆕 Table import from S3 — bulk load without custom code (2022)
- 🔄 Standard-IA table class — lower storage cost for cold tables (2022)
- 🆕 Zero-ETL integration with Redshift (2023)
- 🆕 Resource-based policies — attach policies directly to tables (2023)
- 🔄 On-demand capacity scales faster, less throttling on spikes (2023)
- 🆕 Warm throughput — pre-warm tables for predictable spikes (2024)
- 🆕 Multi-Region strong consistency (preview) (2024)
- 🆕 Warm throughput GA (2025)

### CloudFront

- 🆕 CloudFront Functions — lightweight edge compute (2021)
- 🆕 Response headers policies — security headers without code (2021)
- 🆕 Origin Access Control (OAC) replaces OAI for S3 origins (2022)
- 🆕 Continuous deployment — canary releases for distributions (2022)
- ⚠️ OAI still works but OAC recommended for new setups (2022)
- 🆕 KeyValueStore — low-latency key-value data at the edge (2023)
- 🆕 Post-quantum TLS support (2023)
- 🆕 VPC origins — serve from private ALBs/NLBs without public IPs (2024)
- 🆕 Anycast static IPs for allowlisting (2025)

### API Gateway

- 🆕 HTTP APIs support mutual TLS (2021)
- 💰 HTTP APIs remain ~70% cheaper than REST APIs (2021)
- 🔄 Private APIs with VPC Lattice integration (2022)
- 🔄 Improved OpenAPI import/export (2023)
- 🆕 Enhanced per-route CloudWatch metrics (2023)
- 🔄 Larger payload sizes for WebSocket APIs (2024)
- 🔄 Performance improvements for HTTP APIs (2025)

### EventBridge

- 🆕 Schema Registry — auto-discover event schemas (2021)
- 🔄 Archive and replay events for debugging (2021)
- 🆕 EventBridge Pipes GA — point-to-point integrations with filtering (2022)
- 🆕 EventBridge Scheduler — replaces CloudWatch Events for cron/one-time schedules (2022)
- 🔄 Global endpoints — automatic failover across regions (2022)
- 🆕 Pipes adds more enrichment targets (2023)
- 🔄 Scheduler adds flexible time windows for execution (2024)

---

## Update Log

| Date | What changed |
|------|-------------|
| 2026-03-24 | Initial version — baseline 2021 through March 2025 |

<!-- 
  Weekly update template:
  1. Add new entries at the bottom of the relevant service section
  2. Add a row to the Update Log table
  3. Update "Last updated" date at the top
-->
