# AWS IAM: The Complete Mental Model

## Table of Contents

1. [The Core Idea](#1-the-core-idea)
2. [IAM Building Blocks](#2-iam-building-blocks)
3. [How AWS Decides "Allow or Deny"](#3-how-aws-decides-allow-or-deny)
4. [Trust Relationships & AssumeRole (STS)](#4-trust-relationships--assumerole-sts)
5. [IRSA: IAM Roles for Service Accounts](#5-irsa-iam-roles-for-service-accounts)
6. [Pod Identity: The Modern IRSA Replacement](#6-pod-identity-the-modern-irsa-replacement)
7. [Practical Example: EKS + RDS + Secrets Manager](#7-practical-example-eks--rds--secrets-manager)
8. [Debugging Checklist](#8-debugging-checklist)

---

## 1. The Core Idea

Every single AWS API call answers one question:

> **"Is PRINCIPAL X allowed to perform ACTION Y on RESOURCE Z, given CONDITIONS C?"**

That's it. Every piece of IAM exists to feed data into that question. If you understand this, you understand IAM.

There are only two sides to every permission:

| Side | Question it answers | Where it lives |
|------|-------------------|----------------|
| **Identity policy** | "What can this principal do?" | Attached to users, groups, or roles |
| **Resource policy** | "Who can access this resource?" | Attached to the resource (S3 bucket, SQS queue, Secrets Manager secret, etc.) |

A request is allowed if **either side** grants access (for cross-account, **both sides** must agree).

---

## 2. IAM Building Blocks

### Principals — "Who is making the request?"

A principal is any entity that can make an AWS API call:

- **IAM User** — long-lived credentials (access key + secret key). Avoid for applications.
- **IAM Role** — no permanent credentials. Instead, something *assumes* the role and gets temporary credentials. This is what you should use for everything.
- **AWS Service** — e.g., EC2, Lambda, EKS. Services assume roles on behalf of your code.
- **Federated identity** — an external identity (Google, GitHub, your Kubernetes cluster) that AWS trusts to map to a role.

**Key insight:** Roles are not "assigned" to things. Roles are *assumed* by things. The role has a **trust policy** that says who is allowed to assume it.

### Policies — "What is allowed or denied?"

A policy is a JSON document. Every policy has one or more **statements**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowReadSecrets",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:eu-west-1:111122223333:secret:myapp/db-*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "eu-west-1"
        }
      }
    }
  ]
}
```

Breaking it down:

| Field | Meaning |
|-------|---------|
| `Effect` | `Allow` or `Deny`. Deny always wins. |
| `Action` | The API calls (e.g., `s3:GetObject`, `secretsmanager:GetSecretValue`). |
| `Resource` | The ARN(s) this applies to. Use `*` only when you truly mean "all". |
| `Condition` | Optional extra checks (IP range, tags, time, source VPC, etc.). |

### Policy types ranked by priority

1. **Service Control Policies (SCPs)** — org-level guardrails. If the SCP doesn't allow it, nothing else matters.
2. **Permission boundaries** — a ceiling on what an identity policy can grant.
3. **Resource policies** — attached to the resource itself.
4. **Identity policies** — attached to the user/group/role.
5. **Session policies** — passed at assume-role time to further restrict.

Think of it as layers of filters. The request must pass through ALL of them.

---

## 3. How AWS Decides "Allow or Deny"

```
Request comes in
       │
       ▼
┌─────────────────┐
│ Any EXPLICIT     │──── Yes ──▶ DENIED (always wins)
│ Deny?            │
└────────┬────────┘
         │ No
         ▼
┌─────────────────┐
│ Any EXPLICIT     │──── No ───▶ DENIED (implicit deny)
│ Allow?           │
└────────┬────────┘
         │ Yes
         ▼
      ALLOWED
```

**The golden rule:** Deny is the default. An explicit Deny beats everything. You need at least one explicit Allow, and zero explicit Denies.

For **cross-account** access, both the identity policy (caller's account) AND the resource policy (resource's account) must allow the action. Neither side alone is sufficient.

---

## 4. Trust Relationships & AssumeRole (STS)

This is where most confusion starts. Let's demystify it.

### What is STS?

AWS Security Token Service (STS) is the service that hands out **temporary credentials**. The most important API call is:

```
sts:AssumeRole
```

When something calls `AssumeRole`, STS checks the role's **trust policy** and, if allowed, returns temporary credentials (access key + secret key + session token) that expire.

### The Trust Policy (AssumeRolePolicyDocument)

Every IAM role has a trust policy. It answers: **"Who is allowed to assume this role?"**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

This says: "The EC2 service can assume this role." This is how EC2 instance profiles work — EC2 assumes the role and injects temporary credentials into the instance metadata.

### The Two-Sided Handshake

For a role assumption to work, **two things** must be true:

1. The **trust policy on the role** must allow the caller as a principal.
2. The **caller** must have permission to call `sts:AssumeRole` on that role's ARN (unless the caller is an AWS service, which handles this implicitly).

Think of it like a door with two locks — both must be unlocked.

### AssumeRole Chain Example

```
Developer (IAM User in Account A)
    │
    │  sts:AssumeRole
    ▼
Role: "DeployRole" (Account B)
    │
    │  Now has temporary creds for Account B
    │  sts:AssumeRole
    ▼
Role: "DatabaseAdminRole" (Account B)
    │
    │  Now has temporary creds scoped to DB admin
    ▼
  Makes RDS API calls
```

Each hop requires the target role's trust policy to allow the previous role as a principal.

### AssumeRoleWithWebIdentity

This is the variant used by IRSA and federated identities. Instead of an IAM principal calling AssumeRole, an **external identity provider** (like your EKS OIDC provider, Google, GitHub Actions) presents a **JWT token**, and STS validates it against the trusted provider.

```
Pod in EKS
    │
    │  Has a projected service account JWT token
    │
    │  sts:AssumeRoleWithWebIdentity
    │  (presents the JWT + role ARN)
    ▼
STS validates:
  1. Is the OIDC provider trusted by this role?
  2. Does the JWT's sub/aud match the trust policy conditions?
  3. Is the token valid and not expired?
    │
    │  Yes to all
    ▼
Returns temporary AWS credentials
```

---

## 5. IRSA: IAM Roles for Service Accounts

IRSA connects Kubernetes identity to AWS identity. Here's the full picture.

### The Problem IRSA Solves

Without IRSA, every pod on an EKS node shares the **node's IAM role** (the EC2 instance profile). That means every pod has the same AWS permissions. A logging pod has the same access as your payment processing pod. That's terrible.

IRSA gives **per-pod** AWS permissions by mapping a Kubernetes ServiceAccount to an IAM Role.

### How IRSA Works — Step by Step

```
┌─────────────────────────────────────────────────────────┐
│                    EKS Cluster                          │
│                                                         │
│  ┌──────────────┐    ┌──────────────────────────────┐   │
│  │  Pod          │    │ Projected Service Account    │   │
│  │  (your app)   │◄───│ Token (JWT)                  │   │
│  │               │    │                              │   │
│  │ AWS_ROLE_ARN  │    │ Contains:                    │   │
│  │ AWS_WEB_      │    │  - iss: OIDC provider URL    │   │
│  │ IDENTITY_     │    │  - sub: system:serviceaccount │   │
│  │ TOKEN_FILE    │    │        :namespace:sa-name     │   │
│  └──────┬───────┘    │  - aud: sts.amazonaws.com     │   │
│         │            └──────────────────────────────┘   │
└─────────┼───────────────────────────────────────────────┘
          │
          │ sts:AssumeRoleWithWebIdentity
          ▼
┌─────────────────────────────────────────────────────────┐
│  AWS STS                                                │
│                                                         │
│  1. Fetches OIDC provider's public keys (JWKS)          │
│  2. Validates the JWT signature                         │
│  3. Checks trust policy conditions (sub, aud)           │
│  4. Returns temporary credentials                       │
└─────────────────────────────────────────────────────────┘
```

### The Four Pieces You Need

**1. OIDC Provider** — EKS creates one automatically. You register it in IAM.

```bash
# Get your cluster's OIDC issuer URL
aws eks describe-cluster --name my-cluster \
  --query "cluster.identity.oidc.issuer" --output text

# Register it as an IAM OIDC provider
eksctl utils associate-iam-oidc-provider --cluster my-cluster --approve
```

**2. IAM Role with a trust policy pointing to the OIDC provider**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::111122223333:oidc-provider/oidc.eks.eu-west-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.eu-west-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:sub": "system:serviceaccount:myapp-ns:myapp-sa",
          "oidc.eks.eu-west-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

The `Condition` block is critical — it restricts which namespace and service account can assume this role. Without it, ANY service account in the cluster could assume the role.

**3. Kubernetes ServiceAccount annotated with the role ARN**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: myapp-ns
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::111122223333:role/myapp-role
```

**4. Pod using that ServiceAccount**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp-ns
spec:
  template:
    spec:
      serviceAccountName: myapp-sa
      containers:
        - name: myapp
          image: myapp:latest
```

The EKS pod identity webhook automatically injects `AWS_ROLE_ARN` and `AWS_WEB_IDENTITY_TOKEN_FILE` environment variables into the pod. The AWS SDK picks these up automatically.

---

## 6. Pod Identity: The Modern IRSA Replacement

AWS introduced **EKS Pod Identity** as a simpler alternative to IRSA. The key differences:

| Aspect | IRSA | Pod Identity |
|--------|------|-------------|
| Setup complexity | OIDC provider + conditions in trust policy | Install addon + create association |
| Trust policy | Must include OIDC URL with exact conditions | Simple `pods.eks.amazonaws.com` service principal |
| Cross-account | Complex OIDC provider setup per account | Role chaining built-in |
| ServiceAccount annotation | Required | Not required |

### Pod Identity Trust Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "pods.eks.amazonaws.com"
      },
      "Action": [
        "sts:AssumeRole",
        "sts:TagSession"
      ]
    }
  ]
}
```

Much simpler. The scoping (which namespace/SA can use this role) is handled by the **Pod Identity Association** resource, not by the trust policy conditions.

```bash
aws eks create-pod-identity-association \
  --cluster-name my-cluster \
  --namespace myapp-ns \
  --service-account myapp-sa \
  --role-arn arn:aws:iam::111122223333:role/myapp-role
```

If you're starting fresh, prefer Pod Identity. If you have existing IRSA setups, they continue to work fine.

### Migrating from IRSA to Pod Identity

IRSA and Pod Identity can coexist on the same cluster. You don't need a big-bang migration. The recommended approach:

1. **Install the Pod Identity Agent addon** (if not already present):
   ```bash
   aws eks create-addon \
     --cluster-name my-cluster \
     --addon-name eks-pod-identity-agent
   ```

2. **Create a new trust policy** for the existing role (or create a new role). Replace the OIDC-based trust policy with the simpler `pods.eks.amazonaws.com` one shown above.

3. **Create the Pod Identity Association:**
   ```bash
   aws eks create-pod-identity-association \
     --cluster-name my-cluster \
     --namespace myapp-ns \
     --service-account myapp-sa \
     --role-arn arn:aws:iam::111122223333:role/myapp-role
   ```

4. **Remove the `eks.amazonaws.com/role-arn` annotation** from the ServiceAccount. If both IRSA and Pod Identity are configured for the same SA, Pod Identity takes precedence — but leaving both is confusing. Clean it up.

5. **Restart the pods** so they pick up the new credential source.

6. **After verifying everything works**, remove the old OIDC conditions from the trust policy (or delete the old role if you created a new one).

**What can go wrong during migration:**
- If you update the trust policy but forget to create the Pod Identity Association, the pods lose access entirely — the old IRSA path is gone and the new path isn't set up yet. Always create the association first.
- Pod Identity requires the agent addon running as a DaemonSet. If a node doesn't have the agent pod running, pods on that node silently fall back to the node role (not IRSA). Check with `kubectl get ds -n kube-system eks-pod-identity-agent`.

---

## 7. Practical Example: EKS + RDS + Secrets Manager

Let's build a real scenario end-to-end. You have:

- An EKS cluster
- An RDS PostgreSQL database
- Database credentials stored in Secrets Manager
- An application running in EKS that needs to read the secret and connect to RDS

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│  VPC                                                                │
│                                                                     │
│  ┌─────────────────────────────────┐  ┌──────────────────────────┐  │
│  │  EKS Cluster                    │  │  RDS (private subnet)    │  │
│  │                                 │  │                          │  │
│  │  ┌───────────────────────────┐  │  │  PostgreSQL              │  │
│  │  │ Pod: myapp                │  │  │  Port 5432               │  │
│  │  │ SA: myapp-sa              │  │  │                          │  │
│  │  │                           │  │  │  Security Group:         │  │
│  │  │ 1. Assumes IAM Role ──────┼──┼──┼──▶ (allows EKS SG)      │  │
│  │  │ 2. Reads secret ──────────┼──┼──┼──▶ Secrets Manager       │  │
│  │  │ 3. Connects to DB ────────┼──┼──┼──▶ Port 5432             │  │
│  │  └───────────────────────────┘  │  └──────────────────────────┘  │
│  │                                 │                                │
│  │  Security Group: eks-nodes-sg   │                                │
│  └─────────────────────────────────┘                                │
└─────────────────────────────────────────────────────────────────────┘
```

### Step-by-Step Implementation

Every step below maps to a specific IAM concept from the earlier sections.

---

### Step 1: Create the EKS Cluster Role

The EKS **control plane** itself needs an IAM role to manage AWS resources (ENIs, security groups, etc.) on your behalf.

Trust policy — lets the EKS service assume this role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

```bash
# Create the role
aws iam create-role \
  --role-name eks-cluster-role \
  --assume-role-policy-document file://eks-cluster-trust-policy.json

# Attach the AWS-managed policy for EKS clusters
aws iam attach-role-policy \
  --role-name eks-cluster-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
```

**Why this role exists:** You're telling AWS "I trust the EKS service to act on my behalf to manage cluster infrastructure." Without this, EKS can't create the network interfaces, load balancers, or security group rules it needs.

---

### Step 2: Create the Node Group Role

Worker nodes (EC2 instances) need their own role. This is the **baseline** permission every pod gets if you don't use IRSA/Pod Identity.

Trust policy — lets EC2 assume this role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

```bash
aws iam create-role \
  --role-name eks-node-role \
  --assume-role-policy-document file://ec2-trust-policy.json

# These are the minimum policies for nodes to function
aws iam attach-role-policy --role-name eks-node-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy

aws iam attach-role-policy --role-name eks-node-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy

aws iam attach-role-policy --role-name eks-node-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
```

**Critical point:** Do NOT put your application permissions (Secrets Manager, RDS, S3, etc.) on the node role. That gives every pod on the node those permissions. Use IRSA or Pod Identity instead.

---

### Step 3: Create the EKS Cluster

```bash
aws eks create-cluster \
  --name myapp-cluster \
  --role-arn arn:aws:iam::111122223333:role/eks-cluster-role \
  --resources-vpc-config subnetIds=subnet-aaa,subnet-bbb,securityGroupIds=sg-xxx

# Wait for cluster to be active
aws eks wait cluster-active --name myapp-cluster

# Create the managed node group
aws eks create-nodegroup \
  --cluster-name myapp-cluster \
  --nodegroup-name myapp-nodes \
  --node-role arn:aws:iam::111122223333:role/eks-node-role \
  --subnets subnet-aaa subnet-bbb \
  --instance-types t3.medium \
  --scaling-config minSize=2,maxSize=4,desiredSize=2
```

---

### Step 4: Store Database Credentials in Secrets Manager

```bash
aws secretsmanager create-secret \
  --name myapp/db-credentials \
  --secret-string '{"username":"myapp_user","password":"<password>","host":"mydb.xxxx.eu-west-1.rds.amazonaws.com","port":"5432","dbname":"myappdb"}'
```

---

### Step 5: Create the Application IAM Role (IRSA)

This is the role your application pod will assume. It needs two things:
- A **trust policy** that allows the EKS OIDC provider (so pods can assume it)
- A **permissions policy** that allows reading the specific secret

**5a. Register the OIDC provider:**

```bash
eksctl utils associate-iam-oidc-provider --cluster myapp-cluster --approve
```

**5b. Get the OIDC issuer ID:**

```bash
OIDC_ID=$(aws eks describe-cluster --name myapp-cluster \
  --query "cluster.identity.oidc.issuer" --output text | cut -d'/' -f5)
```

**5c. Create the trust policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::111122223333:oidc-provider/oidc.eks.eu-west-1.amazonaws.com/id/OIDC_ID_HERE"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.eu-west-1.amazonaws.com/id/OIDC_ID_HERE:sub": "system:serviceaccount:myapp:myapp-sa",
          "oidc.eks.eu-west-1.amazonaws.com/id/OIDC_ID_HERE:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

Let's understand every line of this trust policy:

- `Federated` — the principal is not an AWS user or service, it's an external identity provider (your EKS cluster's OIDC endpoint).
- `AssumeRoleWithWebIdentity` — the STS action for OIDC-based federation (not regular `AssumeRole`).
- `sub` condition — restricts to ONLY the service account named `myapp-sa` in the `myapp` namespace. This is your security boundary. Without this, any SA in the cluster could assume this role.
- `aud` condition — the audience must be `sts.amazonaws.com`, preventing token reuse for other purposes.

**5d. Create the role and attach permissions:**

```bash
aws iam create-role \
  --role-name myapp-secrets-role \
  --assume-role-policy-document file://myapp-trust-policy.json
```

**5e. Create the permissions policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowReadDbSecret",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:eu-west-1:111122223333:secret:myapp/db-credentials-*"
    }
  ]
}
```

Note the `*` at the end of the resource ARN — Secrets Manager appends a random 6-character suffix to the secret ARN. You need the wildcard to match it.

```bash
aws iam put-role-policy \
  --role-name myapp-secrets-role \
  --policy-name secrets-access \
  --policy-document file://myapp-secrets-policy.json
```

**Why we use an inline policy here:** For application-specific permissions that are tightly coupled to a single role, inline policies are fine and keep everything together. Use managed policies when you need to share the same policy across multiple roles.

---

### Step 6: Create the Kubernetes ServiceAccount and Deployment

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
---
# serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: myapp
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::111122223333:role/myapp-secrets-role
---
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      serviceAccountName: myapp-sa
      containers:
        - name: myapp
          image: 111122223333.dkr.ecr.eu-west-1.amazonaws.com/myapp:latest
          env:
            - name: AWS_REGION
              value: eu-west-1
            - name: SECRET_NAME
              value: myapp/db-credentials
```

When this pod starts, the EKS pod identity webhook automatically injects:

```
AWS_ROLE_ARN=arn:aws:iam::111122223333:role/myapp-secrets-role
AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token
```

Your application code doesn't need to know about any of this. The AWS SDK automatically uses these environment variables.

---

### Step 7: Application Code (Python Example)

```python
import boto3
import json
import psycopg2

# The SDK automatically uses IRSA credentials — no keys, no config
secrets_client = boto3.client("secretsmanager", region_name="eu-west-1")

response = secrets_client.get_secret_value(SecretId="myapp/db-credentials")
creds = json.loads(response["SecretString"])

conn = psycopg2.connect(
    host=creds["host"],
    port=creds["port"],
    dbname=creds["dbname"],
    user=creds["username"],
    password=creds["password"],
    sslmode="require",
)
```

No access keys anywhere. The SDK's credential chain automatically discovers the IRSA token and calls `AssumeRoleWithWebIdentity` behind the scenes.

### ⚠️ Token Expiration: The Silent Trap

IRSA tokens are short-lived (default 24 hours, configurable down to 1 hour) and the kubelet rotates them automatically on disk. The AWS SDK handles this correctly — it re-reads the token file and refreshes credentials before they expire.

**But** if your application does any of the following, you'll get mysterious "Access Denied" errors after the first token expires:

- **Caching the credentials manually** instead of letting the SDK handle it. Don't do `creds = boto3.Session().get_credentials()` once at startup and reuse forever.
- **Creating a single boto3 client at import time** in a long-running process. The client will refresh automatically, but if you extracted the raw credentials from it, those are stale.
- **Using a non-AWS HTTP client** to call AWS APIs directly with credentials you fetched once.

The fix is simple: let the SDK manage credentials. Create clients normally and don't cache raw access keys:

```python
# ✅ Correct — SDK refreshes credentials automatically
def get_secret():
    client = boto3.client("secretsmanager", region_name="eu-west-1")
    return client.get_secret_value(SecretId="myapp/db-credentials")

# ✅ Also correct — client-level caching is fine, SDK handles refresh internally
secrets_client = boto3.client("secretsmanager", region_name="eu-west-1")
def get_secret():
    return secrets_client.get_secret_value(SecretId="myapp/db-credentials")

# ❌ Wrong — raw credentials go stale
session = boto3.Session()
frozen_creds = session.get_credentials().get_frozen_credentials()
# These will expire and never refresh
```

If you need to customize the token expiration (e.g., for compliance), set it on the ServiceAccount:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: myapp
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::111122223333:role/myapp-secrets-role
    eks.amazonaws.com/token-expiration: "3600"  # 1 hour, in seconds
```

---

### Step 8: Network Security (Security Groups)

IAM handles *authorization* (can you call this API?), but you also need *network connectivity* (can your packets reach the database?).

```bash
# Allow EKS nodes to reach RDS on port 5432
aws ec2 authorize-security-group-ingress \
  --group-id sg-rds-security-group \
  --protocol tcp \
  --port 5432 \
  --source-group sg-eks-nodes-security-group
```

---

### The Complete Permission Chain

When your pod calls `secretsmanager:GetSecretValue`, here's what happens:

```
1. Pod starts → webhook injects token + role ARN env vars
2. AWS SDK detects env vars → reads JWT from token file
3. SDK calls sts:AssumeRoleWithWebIdentity
   ├── Presents: JWT token + role ARN
   ├── STS fetches OIDC provider's public keys
   ├── STS validates JWT signature
   ├── STS checks trust policy:
   │   ├── Is the OIDC provider in the Principal.Federated? ✓
   │   ├── Does the token's "sub" match the Condition? ✓
   │   └── Does the token's "aud" match the Condition? ✓
   └── Returns temporary credentials (valid ~1 hour, auto-refreshed)
4. SDK uses temp credentials to call secretsmanager:GetSecretValue
   ├── IAM evaluates the role's identity policy
   │   ├── Action secretsmanager:GetSecretValue? ✓ (in policy)
   │   ├── Resource matches arn:...myapp/db-credentials-*? ✓
   │   └── Any explicit Deny? ✗
   └── Request ALLOWED
5. Secrets Manager returns the secret value
6. App parses credentials and connects to RDS (network level, not IAM)
```

---

### Optional: RDS IAM Authentication (No Passwords at All)

Instead of storing passwords in Secrets Manager, you can use **IAM database authentication**. The app uses its IAM role to generate a short-lived auth token:

```python
rds_client = boto3.client("rds", region_name="eu-west-1")

token = rds_client.generate_db_auth_token(
    DBHostname="mydb.xxxx.eu-west-1.rds.amazonaws.com",
    Port=5432,
    DBUsername="myapp_user",
    Region="eu-west-1",
)

conn = psycopg2.connect(
    host="mydb.xxxx.eu-west-1.rds.amazonaws.com",
    port=5432,
    dbname="myappdb",
    user="myapp_user",
    password=token,  # This is a temporary IAM token, not a real password
    sslmode="require",
)
```

This requires an additional IAM permission:

```json
{
  "Effect": "Allow",
  "Action": "rds-db:connect",
  "Resource": "arn:aws:rds-db:eu-west-1:111122223333:dbuser:db-XXXX/myapp_user"
}
```

And enabling IAM auth on the RDS instance + creating the DB user with IAM auth:

```sql
CREATE USER myapp_user WITH LOGIN;
GRANT rds_iam TO myapp_user;
```

---

## 8. Debugging Checklist

When something doesn't work, check in this order:

### "Access Denied" on AssumeRoleWithWebIdentity

1. **Is the OIDC provider registered in IAM?**
   ```bash
   aws iam list-open-id-connect-providers
   ```

2. **Does the trust policy's `Federated` ARN match exactly?**
   - Compare the OIDC provider ARN in IAM with the one in the trust policy.
   - **Common gotcha: wrong cluster.** If you have multiple clusters (dev, staging, prod), each has a different OIDC issuer URL. A trust policy pointing to the dev cluster's OIDC provider will silently reject tokens from the staging cluster. The error is the same generic "Access Denied" — no hint about which side failed.
   ```bash
   # Verify which OIDC issuer your cluster actually uses
   aws eks describe-cluster --name my-cluster \
     --query "cluster.identity.oidc.issuer" --output text
   # Then confirm it matches what's in the trust policy
   ```

3. **Do the `sub` and `aud` conditions match?**
   - Decode the JWT token from inside the pod:
   ```bash
   kubectl exec -it <pod> -n myapp -- cat /var/run/secrets/eks.amazonaws.com/serviceaccount/token | cut -d. -f2 | base64 -d 2>/dev/null | jq .
   ```
   - Check that `sub` matches `system:serviceaccount:<namespace>:<sa-name>` exactly.
   - **This is case-sensitive and has no fuzzy matching.** A typo like `myapp-NS` vs `myapp-ns` or `my-app-sa` vs `myapp-sa` gives you the same "Access Denied" as a completely wrong trust policy. There's no partial match, no "did you mean..." — just denied.

4. **Is the ServiceAccount annotation correct?**
   ```bash
   kubectl get sa myapp-sa -n myapp -o yaml
   ```

### "It works in one cluster but not another"

This is almost always an OIDC provider mismatch. Each EKS cluster has a unique OIDC issuer URL. Check:

1. **The trust policy is pointing to the right cluster's OIDC provider:**
   ```bash
   # Get the OIDC issuer for EACH cluster
   for cluster in dev staging prod; do
     echo "$cluster: $(aws eks describe-cluster --name $cluster \
       --query 'cluster.identity.oidc.issuer' --output text)"
   done
   ```
   If the trust policy has the dev cluster's OIDC ID but you're running in staging, it will fail.

2. **The OIDC provider is registered in IAM for that cluster:**
   ```bash
   aws iam list-open-id-connect-providers --output text
   ```
   You need one registered provider per cluster. If you created the staging cluster but forgot `eksctl utils associate-iam-oidc-provider --cluster staging --approve`, no pod in that cluster can assume any IRSA role.

3. **The namespace and SA name are the same across clusters.** If dev uses `namespace: app` but staging uses `namespace: myapp`, the `sub` condition won't match.

**Pro tip:** For multi-cluster setups, consider using `StringLike` instead of `StringEquals` in the trust policy condition with a pattern, or use separate roles per environment (cleaner and more auditable).

### "Access Denied" on the actual API call (GetSecretValue, etc.)

1. **Does the role have the right permissions policy?**
   ```bash
   aws iam list-role-policies --role-name myapp-secrets-role
   aws iam get-role-policy --role-name myapp-secrets-role --policy-name secrets-access
   ```

2. **Does the Resource ARN match?** Remember the Secrets Manager random suffix.

3. **Is there an SCP or permission boundary blocking it?**
   ```bash
   aws iam get-role --role-name myapp-secrets-role
   # Check PermissionsBoundary field
   ```

4. **Is there a resource policy on the secret denying access?**
   ```bash
   aws secretsmanager get-resource-policy --secret-id myapp/db-credentials
   ```

### "Connection refused" to RDS

This is NOT an IAM issue. Check:

1. Security groups — does the RDS SG allow inbound from the EKS node SG?
2. Subnets — are EKS nodes and RDS in the same VPC or peered VPCs?
3. RDS is in a private subnet with no public access? Make sure EKS nodes are in the same VPC.

### "It worked for a while, then started failing"

This is usually a token or credential expiration issue:

1. **Is your app caching credentials manually?** The AWS SDK refreshes IRSA credentials automatically, but if your code extracted raw access keys at startup, they go stale after the session expires (default 1 hour for STS credentials). Let the SDK manage the credential lifecycle.

2. **Was the ServiceAccount token rotated?** The kubelet rotates the projected token before it expires. Check the token's expiry from inside the pod:
   ```bash
   kubectl exec -it <pod> -n myapp -- cat /var/run/secrets/eks.amazonaws.com/serviceaccount/token | cut -d. -f2 | base64 -d 2>/dev/null | jq '.exp | todate'
   ```

3. **Did someone modify the trust policy or permissions policy?** Check CloudTrail for recent IAM changes:
   ```bash
   aws cloudtrail lookup-events \
     --lookup-attributes AttributeKey=EventName,AttributeValue=UpdateAssumeRolePolicy \
     --max-results 10
   ```

### General IAM Debugging

```bash
# Simulate a policy evaluation (very useful)
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::111122223333:role/myapp-secrets-role \
  --action-names secretsmanager:GetSecretValue \
  --resource-arns "arn:aws:secretsmanager:eu-west-1:111122223333:secret:myapp/db-credentials-AbCdEf"

# Check CloudTrail for the actual error
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=GetSecretValue \
  --max-results 5
```

---

## Quick Reference: IAM Mental Model

```
┌──────────────────────────────────────────────────────────────┐
│                    "Can X do Y on Z?"                        │
│                                                              │
│  X = Principal (who?)                                        │
│      ├── IAM User (avoid for apps)                           │
│      ├── IAM Role (prefer this)                              │
│      │   └── Assumed by: EC2, Lambda, EKS pod, another role  │
│      └── Federated Identity (OIDC, SAML)                     │
│                                                              │
│  Y = Action (what API call?)                                 │
│      └── e.g., secretsmanager:GetSecretValue                 │
│                                                              │
│  Z = Resource (which specific thing?)                        │
│      └── e.g., arn:aws:secretsmanager:...:secret:myapp/db-*  │
│                                                              │
│  Evaluated against:                                          │
│      1. SCPs (org guardrails)                                │
│      2. Permission boundaries (ceiling)                      │
│      3. Identity policies (on the principal)                 │
│      4. Resource policies (on the resource)                  │
│      5. Session policies (at assume-role time)               │
│                                                              │
│  Rule: Default DENY → Explicit DENY wins → Need Allow        │
└──────────────────────────────────────────────────────────────┘
```

**Remember:** IAM is just answering "who can do what on which resource." Everything else — STS, IRSA, Pod Identity, trust policies — is just machinery to establish *who the caller is* before that question gets answered.
