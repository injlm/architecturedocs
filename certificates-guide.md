# The Complete Guide to Certificates, TLS & PKI

> A practical, visual guide for solution architects — from zero to confident.

---

## Table of Contents

1. [The Big Picture](#1-the-big-picture)
2. [Core Building Blocks](#2-core-building-blocks)
3. [How TLS Works (Step by Step)](#3-how-tls-works-step-by-step)
4. [Certificate Signing Requests (CSR)](#4-certificate-signing-requests-csr)
5. [Certificate Authorities & Chain of Trust](#5-certificate-authorities--chain-of-trust)
6. [Certificate Types](#6-certificate-types)
7. [Mutual TLS (mTLS)](#7-mutual-tls-mtls)
8. [Certificate Formats & Encodings](#8-certificate-formats--encodings)
9. [Hands-On Examples with OpenSSL](#9-hands-on-examples-with-openssl)
10. [Common Architectures & Patterns](#10-common-architectures--patterns)
11. [Troubleshooting Cheat Sheet](#11-troubleshooting-cheat-sheet)
12. [Quick Reference Card](#12-quick-reference-card)

---

## 1. The Big Picture

Certificates solve one fundamental problem: **how do two parties trust each other over an untrusted network?**

Think of it like passports. Your passport (certificate) was issued by your government (Certificate Authority). When you show it at a border, the officer doesn't need to know you personally — they trust the government that issued it.

```
┌─────────────────────────────────────────────────────────┐
│                   THE TRUST PROBLEM                     │
│                                                         │
│   Browser ──────── Internet ──────── Server             │
│     🧑                🌐                🖥️              │
│                                                         │
│   "Is this REALLY my bank,                              │
│    or someone pretending to be?"                        │
│                                                         │
│   SOLUTION: The server presents a CERTIFICATE           │
│   signed by a trusted CERTIFICATE AUTHORITY (CA)        │
└─────────────────────────────────────────────────────────┘
```

### The Three Goals of TLS/SSL

| Goal               | What it means                                      | Analogy              |
|--------------------|----------------------------------------------------|----------------------|
| **Confidentiality** | Nobody can read the data in transit                | Sealed envelope      |
| **Integrity**       | Nobody can tamper with the data                    | Tamper-proof seal     |
| **Authentication**  | You're talking to who you think you're talking to  | Checking a passport   |

---

## 2. Core Building Blocks

### 2.1 Asymmetric Cryptography (Public/Private Keys)

This is the foundation of everything. You generate a **key pair**:

```
┌──────────────────────────────────────────────────────┐
│              ASYMMETRIC KEY PAIR                      │
│                                                      │
│   ┌─────────────┐         ┌──────────────┐           │
│   │ PRIVATE KEY │         │  PUBLIC KEY   │           │
│   │   🔑 🔒     │         │   🔑 🔓       │           │
│   │             │         │              │           │
│   │ • Keep      │         │ • Share with │           │
│   │   SECRET    │         │   EVERYONE   │           │
│   │ • Used to   │         │ • Used to    │           │
│   │   SIGN &    │         │   VERIFY &   │           │
│   │   DECRYPT   │         │   ENCRYPT    │           │
│   └─────────────┘         └──────────────┘           │
│                                                      │
│   They are mathematically linked:                    │
│   • What one encrypts, only the other decrypts       │
│   • You CANNOT derive the private from the public    │
└──────────────────────────────────────────────────────┘
```

**Two operations:**

| Operation      | Uses           | Purpose                                    |
|---------------|----------------|--------------------------------------------|
| **Encrypt**    | Public key     | Only the private key holder can read it     |
| **Sign**       | Private key    | Anyone with the public key can verify it    |

### 2.2 What IS a Certificate?

A certificate is just a **public key + identity information**, wrapped together and **signed by a trusted party**.

```
┌──────────────────────────────────────────────┐
│          X.509 CERTIFICATE                   │
│                                              │
│  ┌────────────────────────────────────────┐  │
│  │  Subject: CN=www.example.com           │  │
│  │  Issuer:  CN=Let's Encrypt Authority   │  │
│  │  Valid:   2026-01-01 to 2026-12-31     │  │
│  │  Public Key: [RSA 2048-bit key data]   │  │
│  │  Serial Number: 03:A1:B2:...           │  │
│  │  SANs: example.com, www.example.com    │  │
│  │  Key Usage: Digital Signature,         │  │
│  │             Key Encipherment           │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  ── Digital Signature (by the Issuer) ────── │
│  [Signed with Issuer's PRIVATE key]          │
│  (Verifiable with Issuer's PUBLIC key)       │
└──────────────────────────────────────────────┘
```

**Key fields explained:**

| Field            | What it is                                                        |
|-----------------|-------------------------------------------------------------------|
| **Subject**      | Who this cert belongs to (CN = Common Name)                       |
| **Issuer**       | Who signed/issued this cert (the CA)                              |
| **Validity**     | Not Before / Not After dates                                      |
| **Public Key**   | The subject's public key                                          |
| **SANs**         | Subject Alternative Names — additional domains/IPs covered        |
| **Key Usage**    | What this cert is allowed to do                                   |
| **Signature**    | The CA's cryptographic signature proving it issued this cert      |

### 2.3 Private Key

The private key is the **secret half** of the key pair. It never leaves the machine that generated it.

```
CRITICAL RULE:
┌──────────────────────────────────────────────┐
│  The private key NEVER travels.              │
│  The certificate (with public key) travels.  │
│  The CSR travels.                            │
│  The private key stays HOME.                 │
└──────────────────────────────────────────────┘
```

---

## 3. How TLS Works (Step by Step)

### 3.1 The TLS Handshake (Simplified — TLS 1.3)

```
    Browser (Client)                          Server
         │                                      │
    ①    │──── Client Hello ───────────────────▶│
         │    (supported ciphers, random,       │
         │     TLS version)                     │
         │                                      │
    ②    │◀─── Server Hello ────────────────────│
         │    (chosen cipher, random)           │
         │                                      │
    ③    │◀─── Certificate ─────────────────────│
         │    (server's X.509 cert)             │
         │                                      │
    ④    │  Client VERIFIES the certificate:    │
         │  • Is it signed by a trusted CA?     │
         │  • Is it expired?                    │
         │  • Does the domain match?            │
         │  • Is it revoked?                    │
         │                                      │
    ⑤    │◀──▶ Key Exchange ───────────────────▶│
         │    (Diffie-Hellman to agree on a     │
         │     shared symmetric key)            │
         │                                      │
    ⑥    │◀══▶ Encrypted Application Data ═════▶│
         │    (using the shared symmetric key)  │
         │                                      │
```

**Why switch to symmetric encryption?** Asymmetric crypto is slow. It's only used during the handshake to securely agree on a fast symmetric key (like AES). All actual data uses symmetric encryption.

### 3.2 TLS Versions — What You Need to Know

| Version     | Status          | Notes                                    |
|------------|-----------------|------------------------------------------|
| SSL 2.0/3.0| ❌ Deprecated   | Insecure. Never use.                     |
| TLS 1.0    | ❌ Deprecated    | Insecure. Disable everywhere.            |
| TLS 1.1    | ❌ Deprecated    | Insecure. Disable everywhere.            |
| TLS 1.2    | ✅ Acceptable    | Still widely used. Secure if configured well. |
| TLS 1.3    | ✅ Recommended   | Faster handshake, fewer cipher options (by design), more secure. |

---

## 4. Certificate Signing Requests (CSR)

A CSR is a **formal request** you send to a CA saying: "Please sign my public key and identity."

### The CSR Flow

```
    Your Server                    Certificate Authority (CA)
         │                                    │
    ①    │  Generate key pair                 │
         │  (private key + public key)        │
         │                                    │
    ②    │  Create CSR containing:            │
         │  • Your public key                 │
         │  • Your identity (domain, org)     │
         │  • Signed with YOUR private key    │
         │    (proves you own the key pair)   │
         │                                    │
    ③    │────── Send CSR ──────────────────▶ │
         │                                    │
    ④    │                    CA validates:    │
         │                    • Domain ownership│
         │                    • Organization   │
         │                    • CSR signature  │
         │                                    │
    ⑤    │◀───── Signed Certificate ──────────│
         │       (your public key + identity, │
         │        now signed by the CA)       │
         │                                    │
    ⑥    │  Install certificate + private key │
         │  on your server                    │
         │                                    │
```

**What's IN a CSR:**

```
┌──────────────────────────────────────┐
│  Certificate Signing Request (CSR)   │
│                                      │
│  Subject:                            │
│    CN  = www.example.com             │
│    O   = Example Corp                │
│    OU  = Engineering                 │
│    L   = Berlin                      │
│    ST  = Berlin                      │
│    C   = DE                          │
│                                      │
│  Public Key: [your public key]       │
│                                      │
│  Signature: [signed with YOUR        │
│              private key]            │
└──────────────────────────────────────┘
```


---

## 5. Certificate Authorities & Chain of Trust

### 5.1 The Chain of Trust

CAs don't sign your cert directly with their root key. There's a chain:

```
┌─────────────────────────────────────────────────────────────┐
│                    CHAIN OF TRUST                           │
│                                                             │
│   ┌──────────────┐                                          │
│   │   ROOT CA    │  Self-signed. Pre-installed in           │
│   │  Certificate │  browsers/OS trust stores.               │
│   │   (offline)  │  Kept OFFLINE in a vault.                │
│   └──────┬───────┘                                          │
│          │ signs                                            │
│          ▼                                                  │
│   ┌──────────────────┐                                      │
│   │ INTERMEDIATE CA  │  Signed by Root CA.                  │
│   │   Certificate    │  Does the day-to-day signing.        │
│   │    (online)      │  If compromised, Root CA revokes it. │
│   └──────┬───────────┘                                      │
│          │ signs                                            │
│          ▼                                                  │
│   ┌──────────────────┐                                      │
│   │  LEAF / END      │  Your server's certificate.          │
│   │  ENTITY CERT     │  Signed by Intermediate CA.          │
│   │  (your server)   │  This is what clients see.           │
│   └──────────────────┘                                      │
│                                                             │
│   VERIFICATION (bottom-up):                                 │
│   Client checks: Leaf → Intermediate → Root (trusted?)      │
│   If Root is in the trust store → ✅ TRUSTED                │
└─────────────────────────────────────────────────────────────┘
```

**Why intermediates?** If the Root CA private key is compromised, the entire PKI collapses. By keeping it offline and using intermediates, you limit the blast radius.

### 5.2 Trust Stores

Every OS and browser ships with a pre-installed list of trusted Root CA certificates.

| Platform       | Trust Store Location                          |
|---------------|-----------------------------------------------|
| Linux (Debian) | `/etc/ssl/certs/` or `/usr/share/ca-certificates/` |
| Linux (RHEL)   | `/etc/pki/tls/certs/ca-bundle.crt`           |
| macOS          | Keychain Access → System Roots                |
| Windows        | `certmgr.msc` → Trusted Root CAs             |
| Java           | `$JAVA_HOME/lib/security/cacerts`             |
| Node.js        | Uses OS trust store (or `NODE_EXTRA_CA_CERTS`) |

### 5.3 Self-Signed Certificates

A self-signed cert is one where the **issuer = subject** (it signed itself).

```
┌──────────────────────────────────────┐
│  Self-Signed Certificate             │
│                                      │
│  Subject: CN=myapp.local             │
│  Issuer:  CN=myapp.local   ◄── SAME │
│                                      │
│  ⚠️  NOT in any trust store          │
│  ⚠️  Browsers will show warnings     │
│  ✅  Fine for dev/testing            │
│  ✅  Fine for internal mTLS          │
│     (if you control the trust store) │
└──────────────────────────────────────┘
```

---

## 6. Certificate Types

### 6.1 By Validation Level

```
┌────────────────────────────────────────────────────────────────────┐
│                                                                    │
│  DV (Domain Validation)        ← Cheapest / Free (Let's Encrypt)  │
│  ─────────────────────                                             │
│  CA only checks: "Do you control this domain?"                     │
│  Method: DNS record or HTTP challenge                              │
│  Shows: 🔒 in browser                                              │
│  Use for: Blogs, small sites, APIs                                 │
│                                                                    │
│  OV (Organization Validation)  ← Mid-tier                         │
│  ────────────────────────────                                      │
│  CA checks: Domain + organization legally exists                   │
│  Shows: 🔒 in browser (org info in cert details)                   │
│  Use for: Business websites, SaaS                                  │
│                                                                    │
│  EV (Extended Validation)      ← Most expensive                   │
│  ────────────────────────────                                      │
│  CA checks: Domain + org + physical address + legal standing       │
│  Shows: 🔒 in browser (used to show green bar, not anymore)        │
│  Use for: Banks, e-commerce (declining in popularity)              │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 6.2 By Scope

| Type            | Covers                          | Example                        |
|----------------|----------------------------------|--------------------------------|
| **Single domain** | One exact domain               | `www.example.com`              |
| **Wildcard**      | All subdomains at one level    | `*.example.com`                |
| **Multi-domain (SAN)** | Multiple specific domains | `example.com`, `example.org`   |

**Wildcard gotcha:** `*.example.com` covers `www.example.com` and `api.example.com` but NOT `example.com` (the apex) and NOT `sub.api.example.com` (nested).

### 6.3 By Purpose

| Certificate Type     | Key Usage                        | Use Case                        |
|---------------------|----------------------------------|---------------------------------|
| **Server cert**      | Server Authentication            | HTTPS, TLS termination          |
| **Client cert**      | Client Authentication            | mTLS, API auth                  |
| **Code signing**     | Code Signing                     | Sign software binaries          |
| **CA cert**          | Certificate Signing, CRL Signing | Intermediate/Root CAs           |
| **Email (S/MIME)**   | Email Protection                 | Encrypt/sign emails             |

---

## 7. Mutual TLS (mTLS)

### 7.1 Regular TLS vs mTLS

```
REGULAR TLS (one-way):
──────────────────────
Only the SERVER proves its identity.

    Client                          Server
      │                               │
      │◀──── Server Certificate ──────│  Server proves identity
      │                               │
      │  Client is anonymous          │  (or uses passwords/tokens
      │  (at TLS level)               │   at the application level)


MUTUAL TLS (two-way):
─────────────────────
BOTH sides prove their identity with certificates.

    Client                          Server
      │                               │
      │◀──── Server Certificate ──────│  Server proves identity
      │                               │
      │───── Client Certificate ─────▶│  Client proves identity
      │                               │
      │  Both sides verified!         │
```

### 7.2 mTLS Handshake

```
    Client                                    Server
      │                                          │
  ①   │──── Client Hello ──────────────────────▶ │
      │                                          │
  ②   │◀─── Server Hello ───────────────────────│
      │◀─── Server Certificate ──────────────────│
      │◀─── Certificate Request ─────────────────│  ◄── NEW: server
      │     (server asks client for a cert)      │      asks for
      │                                          │      client cert
  ③   │──── Client Certificate ────────────────▶ │
      │──── Certificate Verify ────────────────▶ │  ◄── Client proves
      │     (signed with client's private key)   │      it owns the cert
      │                                          │
  ④   │◀══▶ Encrypted Data ════════════════════▶ │
      │                                          │
```

### 7.3 When to Use mTLS

| Scenario                          | Use mTLS? | Why                                      |
|----------------------------------|-----------|------------------------------------------|
| Service-to-service (microservices)| ✅ Yes    | Strong identity, no shared secrets       |
| Zero-trust networks               | ✅ Yes    | Every connection is authenticated        |
| API gateway → backend             | ✅ Yes    | Backend trusts only the gateway          |
| IoT devices → cloud               | ✅ Yes    | Device identity via certs                |
| Public website                     | ❌ No     | Users don't have client certs            |
| Mobile app → API                   | ⚠️ Maybe  | Complex cert distribution to devices     |

### 7.4 mTLS in Service Meshes

Service meshes (Istio, Linkerd) automate mTLS between microservices:

```
┌──────────────────────────────────────────────────────────┐
│                    KUBERNETES CLUSTER                     │
│                                                          │
│  ┌──────────┐  mTLS   ┌──────────┐  mTLS  ┌──────────┐ │
│  │ Service A├─────────▶│ Service B├───────▶│ Service C│ │
│  │ (+ proxy)│◀─────────┤ (+ proxy)│◀───────┤ (+ proxy)│ │
│  └──────────┘          └──────────┘        └──────────┘ │
│       │                     │                    │       │
│       └─────────────────────┼────────────────────┘       │
│                             │                            │
│                    ┌────────┴────────┐                   │
│                    │  Service Mesh   │                   │
│                    │  Control Plane  │                   │
│                    │                 │                   │
│                    │ • Issues certs  │                   │
│                    │ • Rotates certs │                   │
│                    │ • Manages CA    │                   │
│                    └─────────────────┘                   │
│                                                          │
│  Developers don't touch certs — the mesh handles it all  │
└──────────────────────────────────────────────────────────┘
```


---

## 8. Certificate Formats & Encodings

This is where most confusion happens. Here's the definitive breakdown:

### 8.1 Encoding vs Format

```
ENCODING = How the binary data is represented
FORMAT   = What data structure is inside

Think of it like:
  Encoding = the language (English vs French)
  Format   = the document type (passport vs driver's license)
```

### 8.2 Encodings

| Encoding | Description                                    | Recognizable by                     |
|----------|------------------------------------------------|-------------------------------------|
| **DER**  | Binary. Raw ASN.1 bytes.                       | Not human-readable. Binary file.    |
| **PEM**  | Base64-encoded DER with header/footer lines.   | `-----BEGIN CERTIFICATE-----`       |

```
PEM file example:
┌──────────────────────────────────────────┐
│ -----BEGIN CERTIFICATE-----              │
│ MIIDXTCCAkWgAwIBAgIJAJC1HiIAZAiUMA0G   │
│ CSqGSIb3Qq4wDQYJKoZIhvcNAQEFBQAwVjEL   │
│ ... (base64 encoded DER data) ...        │
│ -----END CERTIFICATE-----               │
└──────────────────────────────────────────┘
```

### 8.3 Common File Extensions

| Extension       | Encoding | Contains                              | Notes                              |
|----------------|----------|---------------------------------------|------------------------------------|
| `.pem`          | PEM      | Cert, key, CSR, or chain (any)        | Most common on Linux               |
| `.crt` / `.cer` | PEM or DER | Certificate                         | `.crt` usually PEM, `.cer` often DER |
| `.key`          | PEM      | Private key                           | Convention, not enforced            |
| `.csr`          | PEM      | Certificate Signing Request           |                                    |
| `.p12` / `.pfx` | Binary   | Cert + private key + chain (bundled)  | Password-protected. Common on Windows. |
| `.jks`          | Binary   | Java KeyStore (certs + keys)          | Java-specific. Being replaced by PKCS12. |
| `.p7b`          | PEM/DER  | Cert chain (no private key)           | Common on Windows                  |

### 8.4 Conversion Cheat Sheet

```bash
# PEM to DER
openssl x509 -in cert.pem -outform DER -out cert.der

# DER to PEM
openssl x509 -in cert.der -inform DER -outform PEM -out cert.pem

# PEM cert + key → PKCS12 (.p12/.pfx)
openssl pkcs12 -export -in cert.pem -inkey key.pem -out bundle.p12

# PKCS12 → PEM (extract cert)
openssl pkcs12 -in bundle.p12 -clcerts -nokeys -out cert.pem

# PKCS12 → PEM (extract key)
openssl pkcs12 -in bundle.p12 -nocerts -nodes -out key.pem
```

---

## 9. Hands-On Examples with OpenSSL

### 9.1 Generate a Private Key

```bash
# RSA 2048-bit key (widely compatible)
openssl genrsa -out server.key 2048

# RSA 4096-bit key (more secure, slower)
openssl genrsa -out server.key 4096

# ECDSA key (modern, faster, smaller — preferred)
openssl ecparam -genkey -name prime256v1 -noout -out server.key
```

### 9.2 Generate a CSR

```bash
# Interactive (prompts for each field)
openssl req -new -key server.key -out server.csr

# Non-interactive (one-liner)
openssl req -new -key server.key -out server.csr \
  -subj "/C=DE/ST=Berlin/L=Berlin/O=Example Corp/CN=www.example.com"
```

### 9.3 Generate a Self-Signed Certificate

```bash
# Generate key + self-signed cert in one command (valid 365 days)
openssl req -x509 -newkey rsa:2048 -nodes \
  -keyout server.key -out server.crt -days 365 \
  -subj "/CN=localhost"

# With SANs (Subject Alternative Names) — required by modern browsers
openssl req -x509 -newkey rsa:2048 -nodes \
  -keyout server.key -out server.crt -days 365 \
  -subj "/CN=myapp.local" \
  -addext "subjectAltName=DNS:myapp.local,DNS:*.myapp.local,IP:127.0.0.1"
```

### 9.4 Create Your Own CA and Sign Certificates

This is the full workflow — your own mini-PKI:

```bash
# ── Step 1: Create the CA ──

# Generate CA private key
openssl genrsa -out ca.key 4096

# Generate CA certificate (self-signed, valid 10 years)
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 \
  -out ca.crt -subj "/CN=My Internal CA/O=My Company"

# ── Step 2: Create a server certificate signed by your CA ──

# Generate server private key
openssl genrsa -out server.key 2048

# Generate CSR for the server
openssl req -new -key server.key -out server.csr \
  -subj "/CN=api.internal.example.com"

# Create an extensions file for SANs
cat > server.ext << EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = api.internal.example.com
DNS.2 = *.internal.example.com
IP.1 = 10.0.0.50
EOF

# Sign the CSR with your CA (valid 1 year)
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out server.crt -days 365 -sha256 \
  -extfile server.ext

# ── Step 3: Verify ──

# Verify the cert was signed by your CA
openssl verify -CAfile ca.crt server.crt
# Expected output: server.crt: OK
```

### 9.5 Create Client Certificate (for mTLS)

```bash
# Generate client key
openssl genrsa -out client.key 2048

# Generate client CSR
openssl req -new -key client.key -out client.csr \
  -subj "/CN=service-a/O=My Company"

# Create client extensions
cat > client.ext << EOF
basicConstraints=CA:FALSE
keyUsage = digitalSignature
extendedKeyUsage = clientAuth
EOF

# Sign with your CA
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out client.crt -days 365 -sha256 \
  -extfile client.ext
```

### 9.6 Inspect Certificates

```bash
# View certificate details
openssl x509 -in server.crt -text -noout

# View CSR details
openssl req -in server.csr -text -noout

# Check private key
openssl rsa -in server.key -check -noout

# Verify cert matches private key (compare modulus)
openssl x509 -in server.crt -modulus -noout | openssl md5
openssl rsa  -in server.key  -modulus -noout | openssl md5
# If both MD5 hashes match → cert and key are a pair ✅

# Check a remote server's certificate
openssl s_client -connect www.example.com:443 -servername www.example.com </dev/null 2>/dev/null \
  | openssl x509 -text -noout

# Check certificate chain of a remote server
openssl s_client -connect www.example.com:443 -showcerts </dev/null
```

### 9.7 Test mTLS Locally

```bash
# Start a TLS server that requires client certs
openssl s_server -accept 8443 \
  -cert server.crt -key server.key \
  -CAfile ca.crt -Verify 1

# In another terminal — connect with client cert
openssl s_client -connect localhost:8443 \
  -cert client.crt -key client.key \
  -CAfile ca.crt
```

---

## 10. Common Architectures & Patterns

### 10.1 TLS Termination at Load Balancer

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  Client ──HTTPS──▶ Load Balancer ──HTTP──▶ Backend Servers   │
│                    (has the cert                              │
│                     + private key)                            │
│                                                              │
│  ✅ Simple. Backends don't manage certs.                     │
│  ⚠️  Traffic between LB and backends is unencrypted.         │
│     OK if they're in the same VPC/network.                   │
│                                                              │
│  AWS: ALB/NLB + ACM certificate                              │
│  GCP: Cloud Load Balancing + managed cert                    │
│  Azure: Application Gateway + Key Vault cert                 │
└──────────────────────────────────────────────────────────────┘
```

### 10.2 TLS Passthrough

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  Client ──HTTPS──▶ Load Balancer ──HTTPS──▶ Backend Server   │
│                    (TCP proxy,               (has the cert    │
│                     doesn't decrypt)          + private key)  │
│                                                              │
│  ✅ End-to-end encryption.                                   │
│  ✅ LB never sees plaintext.                                 │
│  ⚠️  LB can't inspect HTTP headers (no L7 routing).          │
│                                                              │
│  Use: When compliance requires end-to-end encryption.        │
└──────────────────────────────────────────────────────────────┘
```

### 10.3 TLS Re-encryption (TLS Bridging)

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  Client ──HTTPS──▶ Load Balancer ──HTTPS──▶ Backend Server   │
│                    (has cert A,              (has cert B,     │
│                     decrypts,                re-encrypts)     │
│                     inspects,                                │
│                     re-encrypts)                             │
│                                                              │
│  ✅ End-to-end encryption.                                   │
│  ✅ LB can inspect/route based on HTTP content.              │
│  ⚠️  More complex. Two sets of certs to manage.              │
│                                                              │
│  Use: When you need both L7 features AND end-to-end TLS.    │
└──────────────────────────────────────────────────────────────┘
```

### 10.4 Certificate Management in Cloud

```
┌──────────────────────────────────────────────────────────────┐
│              MANAGED CERTIFICATE SERVICES                    │
│                                                              │
│  AWS ACM (Certificate Manager)                               │
│  ├── Free public certs for AWS services (ALB, CloudFront)    │
│  ├── Auto-renewal                                            │
│  ├── Private CA available (AWS Private CA)                   │
│  └── Cannot export private key (by design)                   │
│                                                              │
│  Let's Encrypt                                               │
│  ├── Free DV certs                                           │
│  ├── 90-day validity (forces automation)                     │
│  ├── ACME protocol (certbot, cert-manager)                   │
│  └── Wildcard certs via DNS-01 challenge                     │
│                                                              │
│  HashiCorp Vault                                             │
│  ├── PKI secrets engine (your own CA)                        │
│  ├── Short-lived certs (hours/days)                          │
│  ├── Dynamic cert generation                                 │
│  └── Great for mTLS in microservices                         │
│                                                              │
│  Kubernetes cert-manager                                     │
│  ├── Automates cert lifecycle in K8s                         │
│  ├── Integrates with Let's Encrypt, Vault, AWS, etc.         │
│  └── Stores certs as K8s Secrets                             │
└──────────────────────────────────────────────────────────────┘
```


---

## 11. Troubleshooting Cheat Sheet

### Common Errors and What They Mean

| Error                                          | Cause                                              | Fix                                                  |
|------------------------------------------------|----------------------------------------------------|------------------------------------------------------|
| `certificate has expired`                      | Cert's `Not After` date has passed                 | Renew the certificate                                |
| `unable to verify the first certificate`       | Intermediate cert missing from chain               | Include the full chain (cert + intermediate)         |
| `self-signed certificate`                      | Cert not signed by a trusted CA                    | Add CA to trust store, or use a public CA            |
| `hostname mismatch`                            | Cert CN/SAN doesn't match the requested domain     | Reissue cert with correct SANs                       |
| `certificate signature failure`                | Cert was tampered with or wrong CA in trust store   | Verify the correct CA cert is being used             |
| `key values mismatch`                          | Private key doesn't match the certificate          | Compare modulus hashes (see 9.6)                     |
| `SSL routines:ssl3_get_record:wrong version`   | TLS version mismatch                               | Ensure client and server support same TLS version    |
| `no suitable key share`                        | TLS 1.3 cipher mismatch                            | Update client/server TLS config                      |
| `DEPTH_ZERO_SELF_SIGNED_CERT` (Node.js)        | Self-signed cert not trusted                       | Set `NODE_EXTRA_CA_CERTS` or add to trust store      |
| `PKIX path building failed` (Java)             | CA not in Java truststore                          | Import CA into `cacerts` with `keytool`              |

### Diagnostic Commands

```bash
# Check what cert a server is presenting
echo | openssl s_client -connect host:443 -servername host 2>/dev/null | openssl x509 -noout -subject -issuer -dates

# Check full chain
echo | openssl s_client -connect host:443 -showcerts 2>/dev/null

# Test specific TLS version
openssl s_client -connect host:443 -tls1_2
openssl s_client -connect host:443 -tls1_3

# Check certificate expiry
echo | openssl s_client -connect host:443 2>/dev/null | openssl x509 -noout -enddate

# Verify cert against a CA
openssl verify -CAfile ca.crt -untrusted intermediate.crt server.crt

# Debug: see the full handshake
openssl s_client -connect host:443 -state -debug

# Check if cert and key match
diff <(openssl x509 -in cert.pem -modulus -noout) <(openssl rsa -in key.pem -modulus -noout)
```

---

## 12. Quick Reference Card

### The Mental Model

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   PRIVATE KEY ──generates──▶ CSR ──sent to──▶ CA               │
│       │                                        │                │
│       │                                    signs it             │
│       │                                        │                │
│       │                                        ▼                │
│       │                                   CERTIFICATE           │
│       │                                   (public key           │
│       │                                    + identity           │
│       └──────── paired with ──────────────  + CA sig)           │
│                                                                 │
│   Install BOTH on your server:                                  │
│   • Private key  (server.key)                                   │
│   • Certificate  (server.crt)                                   │
│   • CA chain     (ca-chain.crt) — if not a well-known CA       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Key Relationships

```
Private Key  ←──── mathematically linked ────→  Public Key (inside cert)
Certificate  ←──── signed by ──────────────→  CA's Private Key
CSR          ←──── contains ───────────────→  Your Public Key + Identity
Trust Store  ←──── contains ───────────────→  Root CA Certificates
```

### One-Liner Recipes

```bash
# Quick self-signed cert for development
openssl req -x509 -newkey rsa:2048 -nodes -keyout dev.key -out dev.crt -days 365 -subj "/CN=localhost"

# Check when a remote cert expires
echo | openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -enddate

# Generate a strong private key
openssl ecparam -genkey -name prime256v1 -noout -out private.key

# View cert in human-readable form
openssl x509 -in cert.pem -text -noout

# Verify cert + key match
diff <(openssl x509 -in c.pem -modulus -noout) <(openssl rsa -in k.pem -modulus -noout)

# Bundle cert + key into PKCS12 for import
openssl pkcs12 -export -in cert.pem -inkey key.pem -out bundle.p12

# Add a CA cert to Linux trust store (Debian/Ubuntu)
sudo cp my-ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

### Glossary

| Term          | Definition                                                                 |
|--------------|-----------------------------------------------------------------------------|
| **PKI**       | Public Key Infrastructure — the entire system of CAs, certs, and policies  |
| **CA**        | Certificate Authority — trusted entity that signs certificates             |
| **CSR**       | Certificate Signing Request — your request to a CA to sign your public key |
| **SAN**       | Subject Alternative Name — additional identities in a cert                 |
| **CN**        | Common Name — primary identity in the cert subject                         |
| **CRL**       | Certificate Revocation List — list of revoked certs published by a CA      |
| **OCSP**      | Online Certificate Status Protocol — real-time cert revocation checking    |
| **ACME**      | Automated Certificate Management Environment — protocol used by Let's Encrypt |
| **PEM**       | Privacy Enhanced Mail — base64 encoding format for certs/keys              |
| **DER**       | Distinguished Encoding Rules — binary format for certs/keys                |
| **PKCS#12**   | Standard for bundling cert + key in one encrypted file (.p12/.pfx)         |
| **TLS**       | Transport Layer Security — the protocol (successor to SSL)                 |
| **mTLS**      | Mutual TLS — both client and server authenticate with certs                |
| **Handshake** | The initial negotiation phase of a TLS connection                          |
| **Cipher Suite** | The set of algorithms used for key exchange, encryption, and MAC        |
| **SNI**       | Server Name Indication — TLS extension that sends hostname in Client Hello |
| **HSTS**      | HTTP Strict Transport Security — forces browsers to use HTTPS              |
| **Certificate Pinning** | Hardcoding expected cert/key in client to prevent MITM              |
| **Key Rotation** | Regularly replacing keys/certs to limit exposure from compromise        |

---

> **Remember:** Certificates are just fancy wrappers around public keys, signed by someone you trust. Everything else is details. Once that clicks, the rest follows.
