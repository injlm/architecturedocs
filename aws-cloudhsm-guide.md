# AWS CloudHSM — Architecture & Operations Guide

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Responsibility Model](#responsibility-model)
- [Deployment Lifecycle](#deployment-lifecycle)
  - [Step 1 — Create the Cluster](#step-1--create-the-cluster)
  - [Step 2 — Add an HSM Instance](#step-2--add-an-hsm-instance)
  - [Step 3 — Sign the CSR and Initialize](#step-3--sign-the-csr-and-initialize)
  - [Step 4 — Connect and Configure Users](#step-4--connect-and-configure-users)
  - [Step 5 — Create Keys and Perform Crypto Operations](#step-5--create-keys-and-perform-crypto-operations)
  - [Step 6 — Add HSMs for High Availability](#step-6--add-hsms-for-high-availability)
- [Management Layers](#management-layers)
- [User Types](#user-types)
- [Integration Options](#integration-options)
- [Network Requirements](#network-requirements)
- [Terraform Reference](#terraform-reference)
- [FIPS Compliance](#fips-compliance)

---

## Overview

AWS CloudHSM provides dedicated Hardware Security Modules (HSMs) running inside your VPC. Unlike AWS KMS, where AWS manages the cryptographic material, CloudHSM gives you full ownership and control of your encryption keys. AWS manages the hardware lifecycle but has **zero access** to your cryptographic material.

CloudHSM is FIPS 140-2 Level 3 certified, meaning the physical devices include tamper-detection and response mechanisms.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                            Your VPC                                 │
│                                                                     │
│  ┌──────────────────────┐       ┌──────────────────────┐            │
│  │   Availability Zone A │       │   Availability Zone B │           │
│  │                        │       │                        │          │
│  │  ┌──────────────────┐ │       │  ┌──────────────────┐ │          │
│  │  │   Subnet A        │ │       │  │   Subnet B        │ │         │
│  │  │                    │ │       │  │                    │ │         │
│  │  │  ┌──────────┐     │ │       │  │  ┌──────────┐     │ │         │
│  │  │  │  HSM 1   │     │ │       │  │  │  HSM 2   │     │ │         │
│  │  │  │ (ENI)    │     │ │       │  │  │ (ENI)    │     │ │         │
│  │  │  └────┬─────┘     │ │       │  │  └────┬─────┘     │ │         │
│  │  │       │            │ │       │  │       │            │ │         │
│  │  └───────┼────────────┘ │       │  └───────┼────────────┘ │        │
│  │          │              │       │          │              │         │
│  └──────────┼──────────────┘       └──────────┼──────────────┘        │
│             │    ┌─────────────────┐          │                       │
│             │    │  CloudHSM       │          │                       │
│             └───►│  Cluster        │◄─────────┘                       │
│                  │  (auto-sync)    │                                   │
│                  └────────┬────────┘                                   │
│                           │                                           │
│                  ┌────────▼────────┐                                   │
│                  │  EC2 Instance   │                                   │
│                  │  ┌────────────┐ │                                   │
│                  │  │cloudhsm-cli│ │                                   │
│                  │  │  / PKCS#11 │ │                                   │
│                  │  └────────────┘ │                                   │
│                  └─────────────────┘                                   │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

Each HSM is attached to your VPC via an Elastic Network Interface (ENI). All communication happens over private IP within your VPC — no traffic traverses the public internet.

---

## Responsibility Model

```
┌─────────────────────────────────────────────────────┐
│                    YOU OWN                           │
│                                                     │
│   • HSM users & passwords                           │
│   • Cryptographic keys                              │
│   • Crypto operations (sign, encrypt, etc.)         │
│   • Application integration                         │
│   • Trust anchor (CA certificate)                   │
│                                                     │
├─────────────────────────────────────────────────────┤
│                    AWS OWNS                          │
│                                                     │
│   • Physical hardware lifecycle                     │
│   • Firmware updates                                │
│   • Hardware monitoring & replacement               │
│   • Network infrastructure                          │
│   • Cluster orchestration & sync                    │
│                                                     │
└─────────────────────────────────────────────────────┘
```

> **Key point:** AWS cannot access, recover, or view your cryptographic material under any circumstances. If you lose your credentials, AWS cannot help you recover them.

---

## Deployment Lifecycle

The following diagram shows the end-to-end deployment flow:

```
  ┌──────────────┐     ┌──────────────┐     ┌──────────────────┐
  │  1. Create   │────►│  2. Add HSM  │────►│  3. Sign CSR &   │
  │   Cluster    │     │   Instance   │     │   Initialize     │
  │              │     │              │     │                  │
  │ State:       │     │ Generates    │     │ Upload signed    │
  │ UNINITIALIZED│     │ cluster CSR  │     │ cert + CA cert   │
  └──────────────┘     └──────────────┘     └───────┬──────────┘
                                                    │
                                                    ▼
  ┌──────────────┐     ┌──────────────┐     ┌──────────────────┐
  │  6. Add more │◄────│  5. Create   │◄────│  4. Connect &    │
  │  HSMs (HA)   │     │   Keys       │     │  Setup Users     │
  │              │     │              │     │                  │
  │ Auto-synced  │     │ AES, RSA, EC │     │ Change default   │
  │ across AZs   │     │ via CU user  │     │ admin password   │
  └──────────────┘     └──────────────┘     └──────────────────┘
```

### Step 1 — Create the Cluster

A cluster is a logical grouping of HSMs that stay synchronized. You specify a VPC and subnets across multiple Availability Zones.

At this stage, the cluster is in `UNINITIALIZED` state — it is an empty container with no HSM hardware provisioned yet.

**What you need:**
- A VPC with subnets in at least 2 AZs
- Subnets must have available IP addresses for HSM ENIs

### Step 2 — Add an HSM Instance

Launch at least one HSM instance into a subnet. AWS provisions the physical device and attaches an ENI to your VPC.

When the **first** HSM comes online, the cluster generates a **Certificate Signing Request (CSR)**. This CSR represents the cluster's identity and is the starting point for trust establishment.

### Step 3 — Sign the CSR and Initialize

This is the trust establishment step. The process:

```
┌──────────────┐          ┌──────────────┐          ┌──────────────┐
│  Your CA     │          │  Cluster CSR │          │  AWS CloudHSM│
│  (self-signed│          │  (generated  │          │  API         │
│   or corp PKI)│         │   by cluster)│          │              │
└──────┬───────┘          └──────┬───────┘          └──────┬───────┘
       │                         │                         │
       │  1. Download CSR        │                         │
       │◄────────────────────────┤                         │
       │                         │                         │
       │  2. Sign CSR with       │                         │
       │     your CA key         │                         │
       ├─────────┐               │                         │
       │         │               │                         │
       │◄────────┘               │                         │
       │                         │                         │
       │  3. Upload signed cert + CA cert                  │
       ├──────────────────────────────────────────────────►│
       │                         │                         │
       │                         │    4. Cluster state:    │
       │                         │       INITIALIZED       │
       │                         │◄────────────────────────┤
       │                         │                         │
```

**Why this matters:** This creates a chain of trust. Your applications will use the CA certificate to verify they are communicating with the legitimate HSM, preventing man-in-the-middle attacks.

**Options for the CA:**
- Self-signed CA (simplest, fine for most use cases)
- Corporate PKI / internal CA
- Any X.509 CA you control

### Step 4 — Connect and Configure Users

This is where `cloudhsm-cli` becomes essential.

**What is `cloudhsm-cli`?**

It is the command-line administration tool for managing the **internal state** of the HSM — users, keys, and crypto operations. It must be installed on a machine (typically an EC2 instance) that has network access to the HSM ENIs within your VPC.

> **Important:** The AWS Console, AWS CLI, and Terraform **cannot** manage anything inside the HSM. They only manage the infrastructure envelope. `cloudhsm-cli` is the only way to interact with the HSM's internal state.

**First connection:**

1. Install the CloudHSM client package on an EC2 instance in the same VPC
2. Configure it with the cluster ID and your CA certificate (trust anchor)
3. Connect using the default pre-configured admin: `admin` / `password`
4. **Immediately change the default password**
5. Create the users your applications will need

### Step 5 — Create Keys and Perform Crypto Operations

With a Crypto User (CU) account, you can:

- Generate symmetric keys (AES-128, AES-256)
- Generate asymmetric key pairs (RSA 2048/4096, EC P-256/P-384)
- Import existing keys (key wrapping)
- Sign and verify data
- Encrypt and decrypt data
- Wrap and unwrap keys

### Step 6 — Add HSMs for High Availability

For production workloads, deploy at least **2 HSMs in different Availability Zones**.

When a new HSM is added to an initialized cluster, the cluster **automatically clones** all users and keys to the new HSM. From that point on, all HSMs in the cluster stay in sync.

```
  HSM 1 (AZ-a)  ◄──── auto-sync ────►  HSM 2 (AZ-b)
  ┌────────────┐                        ┌────────────┐
  │ Users ✓    │                        │ Users ✓    │
  │ Keys  ✓    │                        │ Keys  ✓    │
  │ Config ✓   │                        │ Config ✓   │
  └────────────┘                        └────────────┘
```

If one HSM fails, the other continues serving requests with no data loss.

---

## Management Layers

| Layer | Scope | Tools | Examples |
|-------|-------|-------|----------|
| **Infrastructure** | Cluster, HSMs, networking | AWS CLI, Console, Terraform | Create cluster, add HSMs, initialize |
| **Crypto** | Users, keys, operations | `cloudhsm-cli`, PKCS#11, JCE, OpenSSL | Create users, generate keys, sign data |

AWS never crosses into the crypto layer. This separation is fundamental to the security model.

---

## User Types

| User Type | Role | Created By | Purpose |
|-----------|------|------------|---------|
| **Admin** | Administrator | Pre-configured (change password!) | Manage other users |
| **Crypto User (CU)** | Key owner & operator | Admin | Create/manage keys, perform crypto operations. This is what your applications use. |
| **Appliance User (AU)** | Internal sync | System (automatic) | Handles cluster synchronization. Do not modify. |

---

## Integration Options

Applications connect to CloudHSM through standard cryptographic interfaces:

| Interface | Language / Platform | Use Case |
|-----------|-------------------|----------|
| **PKCS#11** | C/C++, general purpose | Industry-standard HSM API |
| **JCE Provider** | Java | Java KeyStore, TLS, signing |
| **OpenSSL Engine** | Any (via OpenSSL) | TLS offloading, certificate operations |
| **KMS Custom Key Store** | Any (via AWS KMS API) | Use CloudHSM-backed keys through the KMS API |

---

## Network Requirements

| Port Range | Protocol | Direction | Purpose |
|------------|----------|-----------|---------|
| 2223–2225 | TCP | App → HSM ENI | CloudHSM client communication |

Ensure your security groups allow this traffic between your application instances and the HSM ENIs. All traffic stays within the VPC.

---

## Terraform Reference

A complete Terraform configuration for deploying CloudHSM (including CA generation, CSR signing, and cluster initialization) is available in the infrastructure codebase. The key resources involved are:

| Resource | Purpose |
|----------|---------|
| `tls_private_key` | Generate CA private key |
| `tls_self_signed_cert` | Create self-signed CA certificate |
| `aws_cloudhsm_v2_cluster` | Create the HSM cluster |
| `aws_cloudhsm_v2_hsm` | Add HSM instance(s) to the cluster |
| `tls_locally_signed_cert` | Sign the cluster CSR with the CA |
| `null_resource` (local-exec) | Run `aws cloudhsmv2 initialize-cluster` |
| `aws_secretsmanager_secret` | Store the CA private key securely |

> **Note:** Terraform handles steps 1–3 (infrastructure + initialization). Steps 4–6 (user setup, key creation, HA) require `cloudhsm-cli` on an EC2 instance with VPC access to the HSMs.

---

## FIPS Compliance

AWS CloudHSM is **FIPS 140-2 Level 3** certified, which means:

- Physical tamper-detection and response mechanisms on the hardware
- Identity-based authentication for operators
- Role-based access control enforced by the HSM firmware
- Only FIPS-approved cryptographic algorithms are available
- No backdoor access — not even for AWS

This makes CloudHSM suitable for workloads requiring:
- U.S. federal government compliance (FedRAMP, FISMA)
- Financial services regulatory requirements
- Healthcare data protection (HIPAA)
- Any use case where you must demonstrate sole custody of encryption keys
