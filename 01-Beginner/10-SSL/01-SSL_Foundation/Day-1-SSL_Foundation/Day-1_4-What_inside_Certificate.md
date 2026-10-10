# Day 2: What's Inside a Certificate + Public Key Pairs

Welcome to Day 2 of WebSphere administration fundamentals. This guide covers the structure of digital certificates, the mechanics of public-private key pairs, and how to troubleshoot SSL/TLS certificate issues in enterprise environments.

---

## 1. Why Should You Care?

Consider a critical production alert at 2:00 AM:

> `Payment portal is DOWN. Customers cannot log in.`

Reviewing the WebSphere application server logs typically reveals errors such as:

```text
SSLHandshakeException: certificate_unknown
PKIX path validation failed
hostname in certificate didn't match
```

These errors stem directly from certificate configuration issues:
- **Hostname mismatch:** The server hostname does not match the CN or SAN entries.
- **Certificate expiration:** The certificate exceeded its `Valid Until` timestamp.
- **Untrusted certificate authority:** The certificate is signed by an entity absent from the client truststore.

Understanding the components of a digital certificate enables fast diagnosis and remediation during outages.

---

## 2. What is a Certificate? (The ID Card Analogy)

An SSL/TLS certificate functions as an identity card for a server.

### ID Card vs. SSL Certificate Mapping

| ID Card (Human) | SSL Certificate (Server) |
| :--- | :--- |
| Full Name | **CN** (Common Name) |
| Aliases / Other Names | **SAN** (Subject Alternative Names) |
| Organization / Address | **O, OU, L, ST, C** (Distinguished Name fields) |
| Date of Issue | **Valid From** |
| Expiry Date | **Valid Until** |
| Issuing Authority | **Issuer** (Certificate Authority, e.g., DigiCert) |
| Unique Identifier | **Serial Number** |
| Biometric Data / Photo | **Public Key** |
| Official Seal / Stamp | **Digital Signature** |

> [!NOTE]
> **Summary Formula:**  
> `Certificate = Identity Information + Public Key + CA Digital Signature`

---

## 3. Certificate Fields Explained

### 1. CN (Common Name)
The primary Fully Qualified Domain Name (FQDN) to which the certificate belongs.

* **Target URL:** `https://pay.bankingapp.com`
* **Configured CN:** `pay.bankingapp.com`

**Handshake Verification:**
* Hostname matches CN $\rightarrow$ SSL handshake continues.
* Hostname differs from CN $\rightarrow$ `hostname in certificate didn't match` error.

> [!WARNING]
> **Real-World Pitfall:**  
> If an administrator renews a certificate using the internal host identity (e.g., `wasserver01.internal.bank.com`) instead of the public customer entry point (`pay.bankingapp.com`), client connections will trigger browser security warnings and cause unexpected application downtime.

---

### 2. SAN (Subject Alternative Names)
An extension field providing a list of all valid hostnames and IP addresses covered by the certificate.

```text
DNS: pay.bankingapp.com
DNS: upi.bankingapp.com
DNS: *.bankingapp.com
IP:  10.20.30.40
```

> [!IMPORTANT]
> **Modern Browser Verification Behavior (Chrome 58+):**  
> Modern TLS stacks evaluate SAN entries exclusively and ignore the Common Name (CN).
> - Correct CN + Empty/Missing SAN $\rightarrow$ **Validation Failure**
> - Incorrect CN + Valid SAN $\rightarrow$ **Validation Success**
>
> Always specify all required hostnames in the SAN extension when generating Certificate Signing Requests (CSRs).

---

### 3. Issuer (Certificate Authority)
The entity that verified the server's identity and applied the cryptographic signature to the certificate.

#### Chain of Trust Hierarchy
```text
[DigiCert Root CA]                  <-- Pre-installed in operating system/browser trust stores
       │
       └── [DigiCert Intermediate CA]
                 │
                 └── [pay.bankingapp.com]  <-- End-entity server certificate
```

#### Certificate Scope in Enterprise Banking

| Type | Signed By | Primary Use Case |
| :--- | :--- | :--- |
| **External** | Public CA (e.g., DigiCert, Sectigo) | Internet-facing endpoints requiring public client trust. |
| **Internal** | Internal Enterprise PKI / CA | WAS-to-DMGR communication, internal microservices, and backend APIs. |

---

### 4. Valid From / Valid Until (Validity Period)
Defines the absolute operational lifespan of the certificate.

```text
Valid From  : 01-Jan-2024 00:00:00 UTC
Valid Until : 01-Jan-2025 23:59:59 UTC
```

At the exact second the `Valid Until` time expires, SSL/TLS handshakes fail across all infrastructure relying on that certificate.

> [!WARNING]
> **Authentication Outage Scenario:**  
> If an LDAP server certificate expires at midnight and WebSphere connects via LDAPS (port 636) for role authentication, WebSphere immediately terminates directory connectivity. Users cannot authenticate across dependent applications until the LDAP certificate is updated and trusted.

---

### 5. Serial Number
A unique alphanumeric string assigned by the issuing CA to distinguish the certificate from all others.

- **Revocation:** Used to identify compromised certificates on Certificate Revocation Lists (CRLs) or via Online Certificate Status Protocol (OCSP).
- **Audit Verification:** Verifies the exact certificate instance actively deployed in runtime profiles.

---

### 6. Public Key
The cryptographic payload wrapped inside the certificate envelope. Anyone who connects to the server receives the certificate and extracts this public key for asymmetric encryption operations.

```text
Certificate Structure:
[ Identity Metadata ]  +  [ Public Key ]  +  [ CA Signature ]
        │                       │                     │
Identity verification       Encryption base      Authenticity proof
```

---

### 7. Signature Algorithm
The cryptographic hash and signature scheme used by the CA to sign the certificate payload.

| Signature Algorithm | Security Status | Compliance Note |
| :--- | :--- | :--- |
| `SHA1withRSA` | Deprecated / Insecure | Non-compliant with PCI-DSS and regulatory mandates; flagged in security audits. |
| `SHA256withRSA` | Current Industry