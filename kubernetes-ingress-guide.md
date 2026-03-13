# Kubernetes Ingress: From Basics to Enterprise Architecture

A comprehensive guide to understanding ingress in Kubernetes, starting from foundational networking concepts and building up to a production-ready enterprise setup on AWS EKS.

---

## Table of Contents

1. [Foundations](#1-foundations)
2. [Kubernetes Service Types](#2-kubernetes-service-types)
3. [What is Ingress in Kubernetes?](#3-what-is-ingress-in-kubernetes)
4. [Ingress Controllers Deep Dive](#4-ingress-controllers-deep-dive)
5. [AWS Load Balancers Explained](#5-aws-load-balancers-explained)
6. [Common EKS Ingress Patterns](#6-common-eks-ingress-patterns)
7. [Enterprise EKS Architecture](#7-enterprise-eks-architecture-final-example)
8. [Summary](#8-summary)

---

## 1. Foundations

Before diving into Kubernetes ingress, we need to understand two core networking concepts: reverse proxies and API gateways. These are the building blocks that ingress is built on.

### 1.1 What is a Reverse Proxy?

A reverse proxy is a server that sits in front of your backend servers and forwards client requests to them. The client talks to the reverse proxy, not directly to the backend.

```
Client ──→ Reverse Proxy ──→ Backend Server(s)
```

For example, a client hits `https://example.com/api` — the reverse proxy receives that request and forwards it to an internal service running on `10.0.0.5:8080`. The client never knows about that internal address.

What it does:

- **Routes requests** to the appropriate backend server based on rules (host, path, headers)
- **Load balances** traffic across multiple backend instances
- **Terminates TLS/SSL** so your backends don't have to handle encryption
- **Caches responses** and compresses content
- **Hides internal topology** — the outside world only sees the proxy

Common reverse proxies: nginx, HAProxy, Traefik, Envoy.

### 1.2 Why is it Called "Reverse"?

The name makes sense when you contrast it with a forward proxy:

- A **forward proxy** sits in front of **clients** — it makes requests on behalf of users going outward. Think of a corporate proxy that employees' traffic goes through to reach the internet.
- A **reverse proxy** sits in front of **servers** — it receives requests coming inward from clients and forwards them to backend servers.

```
Forward Proxy:   Client ──→ [Proxy] ──→ Internet
                 (proxy is on the client side)

Reverse Proxy:   Internet ──→ [Proxy] ──→ Backend Servers
                 (proxy is on the server side)
```

"Reverse" simply means the proxy is on the server side instead of the client side. The direction is flipped.

### 1.3 What is an API Gateway?

An API gateway does everything a reverse proxy does, plus higher-level application concerns:

| Reverse Proxy | API Gateway |
|---|---|
| Routing requests to backends | Everything a reverse proxy does, plus... |
| Load balancing | Authentication / authorization |
| TLS termination | Rate limiting |
| Caching | Request/response transformation |
| | API versioning |
| | Analytics and monitoring |
| | API key management |

An API gateway is a superset. Every API gateway acts as a reverse proxy, but not every reverse proxy is an API gateway.

### 1.4 Different Layers of Concern

Both sit in the same spot in the network path, but they operate at different levels of concern:

```
Client Request
    │
    ▼
┌──────────────────────┐
│   API Gateway        │  ← Auth, rate limiting, transformation, analytics
│   (application logic)│
├──────────────────────┤
│   Reverse Proxy      │  ← Routing, load balancing, TLS, caching
│   (transport logic)  │
├──────────────────────┤
│   Backend Services   │
└──────────────────────┘
```

This isn't strictly about OSI layers — both operate at Layer 7 (HTTP). The distinction is about **scope of responsibility**:

- A reverse proxy asks: **"Where does this request go?"**
- An API gateway asks: **"Should this request be allowed, and how should it be shaped before it goes there?"**

In practice, the lines blur. nginx can do rate limiting and basic auth. Kong (an API gateway) literally runs on top of nginx. But understanding this distinction helps when designing architectures.

Both can run in the same solution. A common setup:

```
Client → API Gateway (auth, rate limiting) → Reverse Proxy (routing) → Services
```

Now that we understand what reverse proxies and API gateways are, let's see how Kubernetes exposes services — which is the foundation for understanding ingress.

---

## 2. Kubernetes Service Types

In Kubernetes, pods are ephemeral — they get created, destroyed, and rescheduled constantly. You can't rely on pod IPs. Services provide a stable way to reach a set of pods.

There are three main Service types, and they build on each other:

### 2.1 ClusterIP (Default)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ClusterIP
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
```

- Creates a virtual IP that's **only reachable from inside the cluster**
- Other pods can reach it via `my-service.default.svc.cluster.local` or just `my-service`
- No external access at all

```
[Inside Cluster]
Pod A ──→ ClusterIP (10.96.0.1:80) ──→ Pod B:8080
```

### 2.2 NodePort

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
```

- Everything ClusterIP does, **plus** opens a specific port on **every node** in the cluster
- Port range: **30000–32767**
- Any traffic hitting `<any-node-ip>:30080` gets forwarded to the target pods
- The node doesn't need to be running the pod — Kubernetes handles forwarding

```
[Outside]
Client ──→ Node1:30080 ──→ Service ──→ Pod (could be on Node2)
Client ──→ Node2:30080 ──→ Service ──→ Pod (could be on Node2)
Client ──→ Node3:30080 ──→ Service ──→ Pod (could be on Node2)
```

Every node listens on port 30080, regardless of where the pod actually runs.

NodePort is rarely used directly in production because exposing raw node IPs isn't ideal. But it's an important building block.

### 2.3 LoadBalancer

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
```

- Everything NodePort does, **plus** provisions an external load balancer from your cloud provider
- On AWS, this creates an NLB (or CLB depending on annotations)
- The load balancer sends traffic to the NodePort across all nodes

```
Client ──→ Cloud Load Balancer ──→ NodePort (on all nodes) ──→ Service ──→ Pod
```

The key insight: **LoadBalancer builds on NodePort, which builds on ClusterIP**. They're layers:

```
ClusterIP    → reachable only inside the cluster
NodePort     → ClusterIP + port on every node (30000-32767)
LoadBalancer → NodePort + external cloud load balancer
```

This is how traffic gets into a cluster. But if you have 20 services, do you want 20 load balancers? That's expensive. This is exactly the problem that Ingress solves.

---

## 3. What is Ingress in Kubernetes?

Ingress is Kubernetes' answer to: "How do I route external HTTP/HTTPS traffic to different services without creating a load balancer for each one?"

It has two parts:

### 3.1 The Ingress Resource

A YAML object where you declare routing rules:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

This says:
- Traffic to `app.example.com/api` → goes to `api-service`
- Traffic to `app.example.com/` → goes to `frontend-service`

One entry point, multiple services. No extra load balancers.

### 3.2 The Ingress Controller

The Ingress resource is just a declaration — a config file. By itself, it does nothing. You need an **Ingress Controller**: the actual reverse proxy that reads Ingress resources and enforces the routing rules.

**Kubernetes does not ship with an ingress controller by default.** You have to install one.

Popular ingress controllers:

| Controller | How it works |
|---|---|
| **nginx ingress** | Runs nginx pods inside the cluster |
| **Traefik** | Runs Traefik pods inside the cluster |
| **HAProxy** | Runs HAProxy pods inside the cluster |
| **AWS Load Balancer Controller** | Provisions AWS ALBs/NLBs (no in-cluster proxy) |
| **Istio Gateway** | Part of the Istio service mesh |

The controller watches the Kubernetes API for Ingress resources and automatically reconfigures itself when they change.

### 3.3 The Traffic Flow

```
Internet ──→ Ingress Controller (reverse proxy pod) ──→ ClusterIP Services ──→ Pods
```

Ingress is essentially Kubernetes' way of letting you define reverse proxy rules as native cluster objects, instead of manually configuring nginx or Traefik yourself.

### 3.4 Important Limitation

Ingress **only handles HTTP/HTTPS** by default. For raw TCP or UDP traffic (databases, message queues, custom protocols), you'd typically use a `Service` of type `LoadBalancer` or `NodePort` directly, or use controller-specific extensions like nginx's TCP/UDP ConfigMaps.

Now let's look at how different ingress controllers work under the hood.

---

## 4. Ingress Controllers Deep Dive

### 4.1 nginx Ingress Controller

The most common ingress controller. It runs nginx as pods inside your cluster:

```
Internet ──→ LoadBalancer Service ──→ nginx pods ──→ ClusterIP Services ──→ App Pods
```

- Deploys as a Deployment or DaemonSet in the cluster
- A `Service` of type `LoadBalancer` exposes the nginx pods externally
- nginx reads Ingress resources and generates its `nginx.conf` automatically
- Supports advanced features: rewrites, custom headers, rate limiting, canary deployments

### 4.2 AWS Load Balancer Controller

Works completely differently. It doesn't run a proxy inside the cluster. Instead, it watches Ingress resources and **provisions actual AWS load balancers** to handle the routing:

```
Internet ──→ AWS ALB (created by the controller) ──→ Pods (via target groups)
```

- No proxy pods inside the cluster
- Each Ingress resource can create an ALB (or they can be grouped — more on this later)
- The ALB routes directly to pods using IP target mode or to nodes via NodePort
- Native integration with AWS services: WAF, ACM, Shield, access logs

### 4.3 Running Multiple Controllers

You can run multiple ingress controllers in the same cluster. Kubernetes uses the `ingressClassName` field to determine which controller handles which Ingress resource:

```yaml
# Handled by nginx
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: internal-app
spec:
  ingressClassName: nginx
  rules:
  - host: internal.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: internal-svc
            port:
              number: 80

---
# Handled by AWS Load Balancer Controller
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: public-app
spec:
  ingressClassName: alb
  rules:
  - host: public.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: public-svc
            port:
              number: 80
```

Both controllers are installed in the cluster. Each watches only for Ingress resources tagged with its class. They operate side by side, handling different traffic paths.

> **Note:** Older setups use the annotation `kubernetes.io/ingress.class` instead of `ingressClassName`. Both work, but `ingressClassName` is the current standard.

This ability to run multiple controllers is key to understanding real-world EKS architectures, where you'll often see both nginx and the AWS Load Balancer Controller installed together.

---

## 5. AWS Load Balancers Explained

On AWS (and specifically EKS), two types of load balancers appear constantly in ingress architectures. Understanding the difference is critical.

### 5.1 NLB — Network Load Balancer (Layer 4)

```
Client ──→ NLB ──→ Target (node or pod)
```

- Operates at **Layer 4 (TCP/UDP)**
- **Does not understand HTTP** — it just forwards raw TCP packets
- Extremely fast and low latency (millions of requests per second)
- Preserves the client's source IP
- Used as a **"dumb pipe"** to get traffic into the cluster

When you install the nginx ingress controller and it creates a `Service` of type `LoadBalancer`, AWS provisions an NLB. The NLB's only job is to forward TCP traffic to the nginx pods. It doesn't know or care about HTTP paths, hosts, or headers.

### 5.2 ALB — Application Load Balancer (Layer 7)

```
Client ──→ ALB ──→ Target (node or pod)
```

- Operates at **Layer 7 (HTTP/HTTPS)**
- **Understands HTTP** — can route based on host, path, headers, query strings
- Integrates natively with AWS services:
  - **WAF** — web application firewall (SQL injection, XSS protection)
  - **ACM** — managed TLS certificates
  - **Shield** — DDoS protection
  - **Access logs** — to S3
- More expensive than NLB, slightly higher latency

### 5.3 When to Use Each

| | NLB | ALB |
|---|---|---|
| **Layer** | 4 (TCP/UDP) | 7 (HTTP/HTTPS) |
| **Understands HTTP** | No | Yes |
| **Use case** | TCP traffic, fronting nginx | HTTP routing, WAF, ACM |
| **AWS integrations** | Minimal | WAF, ACM, Shield, logs |
| **Cost** | Lower | Higher |
| **Latency** | Lower | Slightly higher |

### 5.4 Why They Appear Together

In EKS, you'll often see both:

```
Internet
    │
    ├──→ ALB ──→ Public HTTP services (with WAF, ACM)
    │
    └──→ NLB ──→ nginx ingress pods ──→ Internal services / TCP traffic
```

The ALB handles HTTP traffic that needs AWS-native security features. The NLB is just the entry point for nginx, which handles everything else. They serve different purposes in the same cluster.

---

## 6. Common EKS Ingress Patterns

Here are the three most common patterns you'll encounter in EKS clusters, from simplest to most flexible.

### 6.1 Pattern 1: AWS Load Balancer Controller Only

```
Internet ──→ ALB (per Ingress) ──→ Pods
```

The AWS Load Balancer Controller watches Ingress resources and creates an ALB for each one (by default).

**Pros:**
- Simple to set up
- Native AWS integrations (WAF, ACM, Shield)
- No in-cluster proxy to manage

**Cons:**
- One ALB per Ingress resource by default (~$16-25/month each, even with zero traffic)
- ALB limits per account (default 50 per region)
- Less flexible routing than nginx
- Only HTTP/HTTPS — no TCP support

### 6.2 Pattern 2: nginx Ingress Controller Only

```
Internet ──→ NLB ──→ nginx pods ──→ Services ──→ Pods
```

nginx handles all routing. The NLB is just the entry point.

**Pros:**
- One load balancer for everything
- Very flexible routing (rewrites, custom headers, canary, etc.)
- Can handle TCP/UDP via ConfigMaps
- Portable — not AWS-specific

**Cons:**
- No native WAF integration (need to manage WAF separately or not at all)
- TLS management is on you (cert-manager, etc.)
- nginx pods are a potential bottleneck — need to scale them properly

### 6.3 Pattern 3: Both Controllers Side by Side

```
Internet
    │
    ├──→ ALB (AWS LB Controller) ──→ Public service pods
    │     (ingressClassName: alb)
    │
    └──→ NLB ──→ nginx ingress pods ──→ Internal / TCP service pods
                  (ingressClassName: nginx)
```

Both controllers installed, each handling different Ingress resources based on `ingressClassName`.

**Pros:**
- Best of both worlds
- ALB for public-facing services that need WAF, ACM
- nginx for internal services, complex routing, TCP traffic
- Each path is optimized for its use case

**Cons:**
- More components to manage
- Team needs to understand both

This is the most common pattern in production EKS clusters.

### 6.4 ALB Ingress Grouping

By default, the AWS Load Balancer Controller creates one ALB per Ingress resource. This gets expensive fast. The solution is the `group.name` annotation, which merges multiple Ingress resources into a single ALB:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: page1
  annotations:
    alb.ingress.kubernetes.io/group.name: shared-public
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:eu-west-1:123456789:certificate/abc-123
    alb.ingress.kubernetes.io/wafv2-acl-arn: arn:aws:wafv2:eu-west-1:123456789:regional/webacl/my-waf/abc-123
spec:
  ingressClassName: alb
  rules:
  - host: page1.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: page1-svc
            port:
              number: 80

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: page2
  annotations:
    alb.ingress.kubernetes.io/group.name: shared-public
spec:
  ingressClassName: alb
  rules:
  - host: page2.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: page2-svc
            port:
              number: 80
```

Both Ingress resources share the same ALB because they have the same `group.name`. The wildcard ACM certificate (`*.example.com`) covers all subdomains. The WAF applies to the entire ALB, protecting all sites.

You only need to set the certificate and WAF annotations on one Ingress in the group (or on all — same value, they merge).

**One ALB, one cert, one WAF, many sites.** Much cheaper and cleaner than one ALB per site when they share the same security requirements.

> **When separate ALBs make sense:** If different sites need different WAF rules, different certificates, or security isolation (one site getting DDoS'd shouldn't affect another's ALB), then separate ALBs are the right call. WAF is attached at the ALB level — you can't have different WAF WebACLs on the same ALB for different hosts.

Now let's put everything together into a full enterprise architecture.

---

## 7. Enterprise EKS Architecture (Final Example)

### 7.1 The Requirements

An enterprise application on AWS EKS that needs:

- External **HTTP/HTTPS** traffic (web apps, APIs)
- External **TCP** traffic (databases, custom protocols)
- Controlled access using **WAF** and **AWS Network Firewall**
- **Defense in depth** — multiple security layers
- Worker nodes must not be directly exposed

### 7.2 Architecture Overview

```
                          Internet
                             │
                             ▼
                 ┌───────────────────────┐
                 │   AWS Network Firewall │  ← Network-level inspection
                 │   (Firewall Subnet)    │     IP filtering, IDS/IPS,
                 │                        │     stateful packet inspection
                 └───────────┬───────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
              ▼                             ▼
     ┌─────────────────┐          ┌─────────────────┐
     │   ALB            │          │   NLB            │
     │   (Public Subnet)│          │   (Public Subnet)│
     │                  │          │                  │
     │   + WAF          │          │   Layer 4 only   │
     │   + ACM cert     │          │   TCP forwarding  │
     │   + Shield       │          │                  │
     └────────┬────────┘          └────────┬────────┘
              │                             │
              │  HTTP/S traffic             │  TCP traffic
              │                             │
              ▼                             ▼
     ┌─────────────────┐          ┌─────────────────┐
     │   App Pods       │          │  nginx Ingress   │
     │   (Private       │          │  Pods            │
     │    Subnet)       │          │  (Private Subnet)│
     └─────────────────┘          └────────┬────────┘
                                           │
                                           ▼
                                  ┌─────────────────┐
                                  │   TCP Service    │
                                  │   Pods           │
                                  │  (Private Subnet)│
                                  └─────────────────┘
```

### 7.3 VPC Subnet Layout

```
VPC (10.0.0.0/16)
│
├── Firewall Subnet (10.0.0.0/24)
│   └── AWS Network Firewall endpoints
│
├── Public Subnet (10.0.1.0/24, 10.0.2.0/24)
│   ├── ALB (internet-facing)
│   └── NLB (internet-facing)
│
├── Private Subnet (10.0.10.0/24, 10.0.11.0/24)
│   ├── EKS worker nodes
│   ├── nginx ingress pods
│   ├── Application pods
│   └── TCP service pods
```

All traffic from the internet passes through the firewall subnet before reaching the load balancers. Worker nodes are in private subnets with no direct internet access.

### 7.4 Layer-by-Layer Breakdown

#### Layer 1: AWS Network Firewall (VPC Edge)

Sits in a dedicated firewall subnet. All inbound traffic passes through it before reaching anything else.

- **IP reputation filtering** — block known malicious IPs
- **Stateful packet inspection** — detect and block suspicious traffic patterns
- **Intrusion detection/prevention (IDS/IPS)** — signature-based threat detection
- **Domain filtering** — allow/deny traffic based on domain names
- Protects **all traffic** — both HTTP and TCP

This is critical because WAF only protects what's behind the ALB. TCP traffic going to the NLB has no WAF — the Network Firewall is its protection.

#### Layer 2: ALB with WAF (HTTP/S Path)

The AWS Load Balancer Controller creates ALBs from Ingress resources with `ingressClassName: alb`.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app
  annotations:
    alb.ingress.kubernetes.io/group.name: public
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:eu-west-1:123456789:certificate/abc-123
    alb.ingress.kubernetes.io/wafv2-acl-arn: arn:aws:wafv2:eu-west-1:123456789:regional/webacl/my-waf/abc-123
    alb.ingress.kubernetes.io/shield-advanced-protection: "true"
spec:
  ingressClassName: alb
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
```

What the ALB provides:
- **WAF** — SQL injection, XSS, rate limiting, geo-blocking
- **ACM** — managed TLS termination (wildcard `*.example.com`)
- **Shield** — DDoS protection
- **Access logs** — shipped to S3 for auditing
- **Target type: ip** — routes directly to pod IPs, bypassing NodePort

#### Layer 3: NLB + nginx (TCP Path)

For TCP traffic, the NLB forwards to nginx ingress pods, which route to the appropriate backend.

nginx TCP routing is configured via a ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tcp-services
  namespace: ingress-nginx
data:
  "5432": "default/postgres-svc:5432"
  "6379": "default/redis-svc:6379"
  "1883": "iot/mqtt-broker-svc:1883"
```

This tells nginx:
- TCP traffic on port 5432 → forward to `postgres-svc` in the `default` namespace
- TCP traffic on port 6379 → forward to `redis-svc` in the `default` namespace
- TCP traffic on port 1883 → forward to `mqtt-broker-svc` in the `iot` namespace

The NLB is configured to listen on these ports and forward to the nginx pods.

#### Layer 4: Security Groups

Layered at every level:

- **ALB security group** — allows 443 from internet (or specific CIDRs)
- **NLB** — NLBs don't have security groups, but you control access via node security groups
- **Worker node security group** — allows traffic only from ALB and NLB, not directly from internet
- **Pod-level** — Kubernetes Network Policies restrict pod-to-pod communication

#### Layer 5: Kubernetes Network Policies

The final layer of defense, inside the cluster:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - port: 80
```

This ensures that even if an attacker gets inside the cluster, lateral movement is restricted. The API pods only accept traffic from frontend pods.

### 7.5 Defense in Depth Summary

Traffic passes through multiple security layers before reaching your application:

```
Internet
  → Network Firewall    (network-level: IP filtering, IDS/IPS)
    → WAF on ALB        (application-level: SQLi, XSS, rate limiting)
      → Security Groups (infrastructure-level: port/IP restrictions)
        → Network Policies (cluster-level: pod-to-pod restrictions)
          → Your Application
```

Each layer catches what the previous one doesn't. No single layer is sufficient on its own.

### 7.6 What to Avoid

| Avoid | Why |
|---|---|
| Putting everything through nginx only | You lose AWS-native security integrations (WAF, ACM, Shield) |
| Putting everything through ALB only | ALB can't handle TCP/UDP traffic |
| Skipping Network Firewall | WAF only protects HTTP behind the ALB — TCP traffic is unprotected |
| Worker nodes in public subnets | Direct exposure to the internet, larger attack surface |
| One ALB per Ingress (without grouping) | Expensive, hits account limits, harder to manage |
| Skipping Network Policies | No defense against lateral movement inside the cluster |

---

## 8. Summary

### Component Reference Table

| Component | Layer | Location | Responsibility |
|---|---|---|---|
| **AWS Network Firewall** | Layer 3-4 | VPC edge (firewall subnet) | IP filtering, IDS/IPS, stateful inspection |
| **ALB** | Layer 7 | Public subnet | HTTP/S routing, TLS termination |
| **WAF** | Layer 7 | Attached to ALB | SQL injection, XSS, rate limiting |
| **ACM** | Layer 7 | Attached to ALB | Managed TLS certificates |
| **Shield** | Layer 3-4 | Attached to ALB | DDoS protection |
| **NLB** | Layer 4 | Public subnet | TCP/UDP forwarding (dumb pipe) |
| **nginx Ingress Controller** | Layer 4-7 | Private subnet (in-cluster) | HTTP routing, TCP proxying |
| **AWS Load Balancer Controller** | — | In-cluster (control plane) | Provisions and configures ALBs/NLBs |
| **Security Groups** | Layer 3-4 | AWS infrastructure | Port and IP-based access control |
| **Network Policies** | Layer 3-4 | In-cluster | Pod-to-pod traffic restrictions |

### Key Takeaways

1. **Ingress = reverse proxy rules as Kubernetes objects.** The Ingress resource declares rules; the Ingress Controller enforces them.
2. **You can run multiple ingress controllers** in the same cluster, each handling different Ingress resources via `ingressClassName`.
3. **ALB for HTTP/S, NLB for TCP.** They solve different problems and often coexist.
4. **Group ALB ingresses** with `group.name` to avoid one ALB per service.
5. **Defense in depth** — Network Firewall, WAF, security groups, and network policies each catch different threats.
6. **Keep worker nodes private.** Load balancers in public subnets, everything else in private subnets.
