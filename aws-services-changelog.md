# AWS Core Services Changelog

> Baseline: January 2021. Tracking significant changes to Tier 1, 2, and 3 services.
> Last updated: March 2026

---

## How to Read This

- Organized by year, then by service tier
- Each entry is a notable change — not every minor update, but things that affect how you architect or operate
- 🆕 = New capability | 🔄 = Behavior change | ⚠️ = Deprecation or breaking change | 💰 = Pricing change

---

## Tier 1 — Foundation (IAM, VPC, S3, CloudWatch, EC2, Route 53/ELB)

### 2021 (Baseline Year)

**IAM**
- 🆕 IAM Access Analyzer adds policy generation from CloudTrail activity
- 🆕 Service control policies (SCPs) support new condition keys for tagging

**VPC**
- 🆕 VPC IPAM (IP Address Manager) — centralized IP address planning across accounts
- 🆕 Network Access Analyzer — verify network segmentation intent
- 🔄 Security group rules now support descriptive rule IDs

**S3**
- 🔄 S3 Object Ownership — bucket owner can now enforce ownership of all objects (disabling ACLs)
- 🆕 S3 Intelligent-Tiering adds Archive Access and Deep Archive tiers
- 🆕 S3 Multi-Region Access Points — single global endpoint across regions
- 💰 No more charge for S3 GET requests on Intelligent-Tiering when objects move tiers

**CloudWatch**
- 🆕 CloudWatch Metrics Insights — SQL-like query language for metrics
- 🆕 CloudWatch Evidently — feature flags and A/B testing
- 🆕 CloudWatch RUM (Real User Monitoring)

**EC2**
- 🆕 Graviton3 processors announced
- 🆕 M6i, C6i, R6i instances (Intel Ice Lake)
- 🆕 EC2 Auto Scaling warm pools — pre-initialized instances for faster scaling
- 🔄 EBS gp3 volumes now default (replacing gp2 as recommended)

**Route 53 / ELB**
- 🆕 ALB supports TLS 1.3
- 🆕 NLB supports IPv6 targets
- 🆕 Route 53 Resolver DNS Firewall

---

### 2022

**IAM**
- 🆕 IAM Roles Anywhere — use X.509 certificates to get temporary AWS credentials from on-premises workloads (no more long-lived keys for hybrid setups)
- 🆕 IAM Identity Center (rebranded from AWS SSO) — centralized workforce access
- 🔄 IAM Access Analyzer adds custom policy checks against security standards

**VPC**
- 🆕 VPC Lattice (preview) — application-layer networking across VPCs and accounts
- 🆕 Verified Access — zero-trust access to applications without VPN
- 🔄 Security group referencing across peered VPCs now supported
- 💰 IPv4 public address charges announced (effective Feb 2024)

**S3**
- 🔄 S3 Block Public Access becomes default on all new buckets
- 🔄 ACLs disabled by default on new buckets (Object Ownership = BucketOwnerEnforced)
- 🆕 S3 Object Lambda — transform data on retrieval
- 🆕 S3 Glacier Instant Retrieval storage class
- 💰 No charge for S3 DELETE and lifecycle transition requests

**CloudWatch**
- 🆕 CloudWatch cross-account observability — centralized monitoring across accounts
- 🆕 CloudWatch Log Insights adds pattern analysis
- 🔄 Lambda Insights now built into CloudWatch (no separate install)

**EC2**
- 🆕 Graviton3-based instances GA (C7g, M7g, R7g)
- 🆕 Hpc6a instances for high-performance computing
- 🆕 EC2 Instance Connect Endpoint — SSH/RDP without public IPs or bastion hosts
- ⚠️ EC2-Classic fully retired

**Route 53 / ELB**
- 🆕 ALB weighted target groups — native canary deployments
- 🆕 Route 53 HTTPS health checks with SNI support
- 🆕 NLB supports security groups (finally)

---

### 2023

**IAM**
- 🆕 IAM Access Analyzer adds unused access findings — identifies permissions granted but never used
- 🔄 Root user sessions can now be centrally managed from the org management account
- 🆕 IAM policy recommendations based on Access Analyzer findings

**VPC**
- 🆕 VPC Lattice GA — service-to-service networking with built-in auth, observability
- 🔄 VPC peering supports larger CIDR blocks
- 🆕 Subnet-level DHCP options

**S3**
- 🆕 S3 Express One Zone — single-digit millisecond latency storage class (for compute-adjacent workloads)
- 🆕 Mountpoint for S3 — mount S3 buckets as local file systems on Linux
- 🔄 S3 default encryption switches to SSE-S3 (all new objects encrypted automatically)
- 🆕 S3 Access Grants — map S3 permissions to corporate directory identities

**CloudWatch**
- 🆕 CloudWatch Application Signals — auto-discover and monitor application health (SLOs)
- 🆕 CloudWatch Logs anomaly detection
- 🔄 Log class options: Standard and Infrequent Access (cheaper for logs you rarely query)
- 💰 Infrequent Access log class at 50% lower ingestion cost

**EC2**
- 🆕 Graviton4-based instances announced (R8g)
- 🆕 M7i, C7i instances (Intel Sapphire Rapids)
- 🆕 M7a, C7a instances (AMD Genoa)
- 💰 Public IPv4 address charge: $0.005/hr per address (effective Feb 2024)
- 🆕 EC2 Instance Connect Endpoint GA

**Route 53 / ELB**
- 🆕 ALB mutual TLS (mTLS) support
- 🆕 Route 53 Profiles — share DNS configurations across VPCs and accounts
- 🔄 ALB supports HTTP/2 gRPC health checks

---

### 2024

**IAM**
- 🆕 Resource control policies (RCPs) — like SCPs but for resources instead of identities
- 🔄 IAM Access Analyzer adds public and cross-account access previews before deployment
- 🆕 Declarative policies for enforcing baseline configs across org (e.g., block public AMIs)

**VPC**
- 🔄 VPC Block Public Access — account-level control to block internet traffic to/from VPCs
- 🆕 VPC Lattice adds TCP/TLS support (not just HTTP anymore)
- 💰 Public IPv4 charges now active ($0.005/hr per address — affects ELBs, NAT GWs, EC2)
- 🆕 Security group VPC associations — share security groups across VPCs

**S3**
- 🆕 S3 Tables — managed Apache Iceberg tables directly in S3 (analytics-native)
- 🆕 S3 Metadata — automatic metadata extraction on objects
- 🔄 S3 Access Grants integrates with IAM Identity Center
- 🆕 Conditional writes on S3 (if-none-match) — prevents overwrites without external locking

**CloudWatch**
- 🆕 CloudWatch Database Insights — unified monitoring for RDS/Aurora
- 🆕 CloudWatch natural language query for Logs Insights
- 🔄 CloudWatch alarms support account-level suppression windows (maintenance mode)

**EC2**
- 🆕 Graviton4 instances GA (M8g, C8g, R8g, X8g)
- 🆕 P5 instances (NVIDIA H100) for ML training
- 🆕 Capacity Blocks for ML — reserve GPU instances for defined time windows
- 🔄 Default EBS volume type is gp3 in console and most IaC tools

**Route 53 / ELB**
- 🆕 ALB advanced request routing with header-based conditions
- 🆕 Route 53 Resolver on Outposts
- 🔄 NLB cross-zone load balancing now configurable per target group

---

### 2025 (through March)

**IAM**
- 🔄 IAM Access Analyzer recommendations now integrated into policy editor in console
- 🆕 Expanded support for resource control policies across more service types

**VPC**
- 🆕 VPC Lattice multi-protocol support improvements
- 🔄 VPC IPAM adds automatic conflict detection for overlapping CIDRs

**S3**
- 🔄 S3 Express One Zone expands to additional regions
- 🆕 S3 Tables adds automated compaction and snapshot management

**CloudWatch**
- 🆕 CloudWatch investigator — AI-assisted root cause analysis
- 🔄 Application Signals expands to EKS and ECS workloads

**EC2**
- 🆕 Graviton4 instances expand to more sizes and regions
- 🆕 P5e instances (NVIDIA H200) for next-gen ML workloads
- 🆕 Trn2 instances (AWS Trainium2) for cost-efficient ML training

**Route 53 / ELB**
- 🔄 ALB adds improved connection draining controls
- 🆕 Route 53 Profiles expands to cross-account sharing

---

## Tier 2 — Architecture Dependent (RDS, Lambda, ECS/EKS, Secrets Manager, ACM, KMS)

### 2021

**RDS**
- 🆕 RDS Custom — managed RDS with OS-level access (for legacy apps needing custom configs)
- 🆕 Aurora Serverless v2 (preview) — scales to fractional ACUs
- 🔄 RDS Proxy GA for PostgreSQL and MySQL

**Lambda**
- 🆕 Lambda container image support (up to 10 GB images)
- 🆕 Lambda extensions GA — integrate monitoring tools into the runtime lifecycle
- 🔄 Lambda billing granularity changes to 1ms (from 100ms)
- 🆕 Graviton2 support for Lambda (arm64)

**ECS / EKS**
- 🆕 ECS Anywhere — run ECS tasks on on-premises servers
- 🆕 EKS Anywhere — run EKS on your own infrastructure
- 🆕 EKS managed addons expand (CoreDNS, kube-proxy, VPC CNI)
- 🔄 Fargate supports ephemeral storage up to 200 GB

**Secrets Manager**
- 🆕 Automatic rotation for additional database types
- 🔄 Cross-region secret replication

**ACM**
- 🔄 ACM certificates auto-renew up to 60 days before expiry
- 🆕 ACM Private CA short-lived certificates

**KMS**
- 🆕 Multi-Region KMS keys — same key material replicated across regions
- 🆕 HMAC key support

---

### 2022

**RDS**
- 🆕 Aurora Serverless v2 GA — scales in 0.5 ACU increments, sub-second scaling
- 🆕 RDS Blue/Green Deployments — managed switchover for major version upgrades
- 🔄 RDS Optimized Reads (instance store caching)
- ⚠️ Aurora Serverless v1 enters deprecation path

**Lambda**
- 🆕 Lambda SnapStart (Java) — near-zero cold starts via snapshot restore
- 🆕 Lambda function URLs — built-in HTTPS endpoint without API Gateway
- 🔄 Lambda max memory increases to 10 GB
- 🆕 Lambda supports Node.js 18, Python 3.10

**ECS / EKS**
- 🆕 EKS Pod Identity (preview) — simpler alternative to IRSA
- 🆕 ECS Service Connect — built-in service mesh
- 🔄 Fargate supports Windows containers
- 🆕 EKS add-ons support ADOT (AWS Distro for OpenTelemetry)

**Secrets Manager**
- 💰 Pricing simplified — per-secret per-month + per-10K API calls
- 🆕 Batch retrieval of secrets (BatchGetSecretValue)

**ACM**
- 🔄 ACM supports ECDSA P-384 and RSA 3072/4096 key types
- 🆕 ACM integration with CloudFormation improves (faster validation)

**KMS**
- 🆕 External key store (XKS) — use keys stored outside AWS (in your HSM)
- 🔄 KMS key policies support ABAC (attribute-based access control)

---

### 2023

**RDS**
- 🆕 Aurora Limitless Database (preview) — horizontal write scaling beyond single-writer
- 🆕 RDS Extended Support — pay to keep older engine versions past EOL
- 🔄 RDS Blue/Green Deployments supports Aurora
- 🆕 Aurora I/O-Optimized — predictable pricing for I/O-heavy workloads (no per-I/O charges)

**Lambda**
- 🆕 Lambda supports Python 3.12, Node.js 20, Java 21
- 🔄 Lambda response streaming for Node.js — send partial responses as they're ready
- 🆕 Lambda SnapStart expands to Python and .NET
- ⚠️ Python 3.7, Node.js 14 runtimes deprecated

**ECS / EKS**
- 🆕 EKS Pod Identity GA — simplified IAM for pods (see your IAM guide)
- 🆕 ECS auto-scaling improvements with target tracking on custom metrics
- 🆕 EKS supports Kubernetes 1.28, 1.29
- 🔄 Fargate ephemeral storage increases to 200 GB default

**Secrets Manager**
- 🆕 Secrets Manager integrates with AWS Config for compliance tracking
- 🔄 Rotation window scheduling improvements

**ACM**
- 🔄 ACM certificate renewal window extends to 90 days before expiry
- 🆕 ACM supports importing certificates with longer chains

**KMS**
- 🆕 KMS automatic key rotation now configurable (90 days to 7 years, was fixed at 1 year)
- 🔄 KMS key deletion waiting period can be set to 7 days (from 30)

---

### 2024

**RDS**
- 🆕 Aurora Limitless Database GA
- 🆕 Aurora PostgreSQL Optimized Reads with tiered caching
- 🔄 RDS Extended Support pricing kicks in for older engine versions
- 💰 RDS Extended Support: additional charges for running past-EOL versions
- 🆕 RDS integration with Secrets Manager for automatic credential management at creation

**Lambda**
- 🆕 Lambda supports Python 3.13, Node.js 22
- 🆕 Lambda SnapStart for ARM64 (Graviton)
- 🔄 Lambda max timeout remains 15 minutes but recursive loop detection improves
- 🆕 Lambda scales faster — up to 12x faster function scaling

**ECS / EKS**
- 🆕 EKS Auto Mode — AWS manages node groups, scaling, and upgrades entirely
- 🆕 ECS support for EBS volume attachments to tasks
- 🔄 EKS extended support for Kubernetes versions (14 months → 26 months)
- 💰 EKS extended K8s version support costs extra after standard support ends

**Secrets Manager**
- 🆕 Secrets Manager supports zero-ETL integration with Redshift
- 🔄 Improved cross-account secret sharing via resource policies

**ACM**
- 🔄 ACM certificates now default to 90-day validity (following industry trend)
- 🆕 ACM supports automated renewal for imported certificates (if DNS validation is set up)

**KMS**
- 🆕 KMS supports ML-KEM post-quantum key encapsulation (preparing for quantum computing)
- 🔄 KMS key policies now support service-linked roles more granularly

---

### 2025 (through March)

**RDS**
- 🔄 Aurora Serverless v2 minimum ACU lowered to 0 (scale to zero, cold start on first connection)
- 🆕 RDS Blue/Green Deployments supports additional engine versions

**Lambda**
- 🔄 Lambda SnapStart improvements reduce restore latency further
- 🆕 Lambda adds support for Ruby 3.4

**ECS / EKS**
- 🔄 EKS Auto Mode expands to additional regions
- 🆕 ECS adds native support for service discovery via Cloud Map improvements

**Secrets Manager**
- 🔄 Rotation Lambda templates updated for latest runtime versions

**ACM**
- 🔄 ACM renewal automation improvements for multi-region certificates

**KMS**
- 🔄 Post-quantum cryptography support expands to additional key operations

---

## Tier 3 — Use-Case Specific (SQS, SNS, DynamoDB, CloudFront, API Gateway, EventBridge)

### 2021

**SQS**
- 🆕 SQS supports high-throughput FIFO queues (up to 3,000 msg/sec with batching)
- 🔄 Dead-letter queue redrive — move messages back to source queue from console

**SNS**
- 🆕 SNS FIFO topics — ordered, deduplicated pub/sub
- 🆕 SNS message filtering supports nested attributes

**DynamoDB**
- 🆕 PartiQL support — SQL-compatible query language for DynamoDB
- 🆕 Kinesis Data Streams integration for change data capture (alternative to DynamoDB Streams)
- 💰 On-demand pricing reduced by up to 50% in some regions

**CloudFront**
- 🆕 CloudFront Functions — lightweight edge compute (cheaper/faster than Lambda@Edge for simple transforms)
- 🆕 Response headers policies — add security headers without code
- 🔄 Origin Shield — centralized caching layer to reduce origin load

**API Gateway**
- 🆕 HTTP APIs support mutual TLS
- 🔄 WebSocket API improvements for connection management
- 💰 HTTP APIs remain ~70% cheaper than REST APIs

**EventBridge**
- 🆕 EventBridge Schema Registry — auto-discover event schemas
- 🆕 EventBridge Pipes — point-to-point integrations with filtering and enrichment
- 🔄 Archive and replay events for debugging

---

### 2022

**SQS**
- 🔄 SQS message throughput increases for standard queues
- 🆕 SQS supports attribute-based access control (ABAC)

**SNS**
- 🆕 SNS payload-based message filtering (filter on message body, not just attributes)
- 🔄 SNS delivery retries become more configurable

**DynamoDB**
- 🆕 DynamoDB table import from S3 — bulk load without custom code
- 🆕 DynamoDB Contributor Insights included in standard metrics
- 🔄 DynamoDB Standard-IA table class — lower storage cost for infrequently accessed tables

**CloudFront**
- 🆕 CloudFront Origin Access Control (OAC) — replaces Origin Access Identity (OAI) for S3
- ⚠️ OAI still works but OAC is recommended for new setups
- 🆕 CloudFront continuous deployment — canary releases for distributions

**API Gateway**
- 🔄 API Gateway supports private APIs with VPC Lattice integration
- 🆕 Improved request validation and transformation templates

**EventBridge**
- 🆕 EventBridge Pipes GA — connect sources to targets with optional filtering/enrichment
- 🆕 EventBridge Scheduler — cron and one-time scheduled events (replaces CloudWatch Events rules)
- 🔄 EventBridge global endpoints — automatic failover across regions

---

### 2023

**SQS**
- 🔄 SQS dead-letter queue redrive now available via API (not just console)
- 🆕 SQS supports server-side encryption with SQS-managed keys by default

**SNS**
- 🔄 SNS message archiving and replay (via Kinesis Data Firehose)
- 🆕 SNS supports delivery to Amazon Q endpoints

**DynamoDB**
- 🆕 DynamoDB zero-ETL integration with Redshift — real-time analytics without pipelines
- 🆕 DynamoDB resource-based policies — attach policies directly to tables
- 🔄 DynamoDB on-demand capacity scales faster (no more throttling on sudden spikes)

**CloudFront**
- 🆕 CloudFront KeyValueStore — low-latency key-value data at the edge
- 🔄 CloudFront Functions adds support for async event handling
- 🆕 CloudFront supports post-quantum TLS (FIPS 140-3)

**API Gateway**
- 🔄 API Gateway improves OpenAPI import/export
- 🆕 Enhanced observability with CloudWatch detailed metrics per route

**EventBridge**
- 🆕 EventBridge Pipes adds more enrichment targets
- 🔄 EventBridge event bus throughput increases
- 🆕 EventBridge integration with Partner SaaS events expands

---

### 2024

**SQS**
- 🔄 SQS FIFO throughput increases (up to 70,000 msg/sec with batching)
- 🆕 SQS supports dead-letter queue redrive policies with more granular controls

**SNS**
- 🆕 SNS supports in-place message encryption with customer-managed KMS keys
- 🔄 SNS FIFO topics throughput improvements

**DynamoDB**
- 🆕 DynamoDB warm throughput — pre-warm tables for predictable traffic spikes
- 🆕 DynamoDB multi-Region strong consistency (preview)
- 🔄 Global tables v2 improvements for conflict resolution

**CloudFront**
- 🆕 CloudFront VPC origins — serve content from private ALBs/NLBs without public IPs
- 🔄 CloudFront grpc support via HTTP/2 origins
- 🆕 CloudFront embedded points of presence expand

**API Gateway**
- 🔄 API Gateway supports larger payload sizes for WebSocket APIs
- 🆕 Improved integration with VPC Lattice for private API patterns

**EventBridge**
- 🆕 EventBridge adds support for partner event sources from more SaaS providers
- 🔄 EventBridge Scheduler adds flexible time windows for execution

---

### 2025 (through March)

**SQS**
- 🔄 SQS encryption with SQS-managed keys becomes default for all new queues

**SNS**
- 🔄 SNS delivery logging improvements for debugging failed deliveries

**DynamoDB**
- 🔄 DynamoDB zero-ETL expands to additional analytics services
- 🆕 DynamoDB warm throughput GA

**CloudFront**
- 🔄 CloudFront VPC origins expands to additional origin types
- 🆕 CloudFront anycast static IPs — fixed IPs for allowlisting

**API Gateway**
- 🔄 API Gateway performance improvements for HTTP APIs

**EventBridge**
- 🔄 EventBridge Pipes adds additional source and target integrations

---

## Update Log

| Date | What changed |
|------|-------------|
| 2026-03-24 | Initial version — baseline 2021 through March 2025 |

<!-- 
  Weekly update template:
  1. Add new entries under the current year section for each tier
  2. Add a row to the Update Log table
  3. Update "Last updated" date at the top
-->
