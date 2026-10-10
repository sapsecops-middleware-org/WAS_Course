# Types of Certificate Authorities (CAs) in Banking Infrastructure

## Overview

In enterprise banking architecture, digital certificates secure transport layer security (TLS/SSL) connections across public-facing channels and internal server-to-server communications. Banking environments implement a two-tier certificate authority classification to separate trust boundaries between customer traffic and backend infrastructure.

---

## CA Classifications

### 1. Public CA (External)

Public Certificate Authorities issue certificates that must be universally recognized by external client applications, mobile operating systems, and public web browsers without manual root certificate installation.

* **Examples:** DigiCert, Comodo, GlobalSign, Sectigo
* **Use Cases:** Customer-facing applications (Internet Banking portals, Payment Gateways, Mobile Banking REST/GraphQL APIs)
* **Cost:** Paid — ₹5,000 to ₹1,00,000 per certificate/year (varies by validation level: DV, OV, EV)
* **Trust Store:** Pre-installed in major operating systems, browsers, and standard Java runtime trust stores worldwide
* **Issuance Workflow:**
  1. Generate private key and Certificate Signing Request (CSR)
  2. Submit CSR to Public CA portal
  3. CA executes domain ownership and organizational identity validation
  4. CA signs and issues certificate chain
  5. Download and bind certificate to web servers/load balancers

> [!NOTE]
> Public CAs must strictly adhere to CA/Browser Forum Baseline Requirements and support Certificate Transparency (CT) logging.

---

### 2. Internal/Private CA (Enterprise Bank CA)

Internal Certificate Authorities are privately managed public key infrastructures (PKI) designed for non-public assets, operational consoles, and inter-service mesh communication.

* **Examples:** Microsoft Active Directory Certificate Services (AD CS), OpenSSL PKI, HashiCorp Vault PKI Engine
* **Use Cases:** Internal service communication — WebSphere Application Server (WAS)-to-WAS, WAS-to-LDAP, WAS-to-IBM MQ, WAS-to-Database (DB2), and WebSphere Deployment Manager (DMGR) Admin Consoles
* **Cost:** Operational/infrastructure overhead only (no per-certificate vendor licensing)
* **Trust Store:** Private trust scope; root and intermediate certificates are distributed to bank-managed workstations and servers via Group Policy Objects (GPO), MDM, or automated configuration management
* **Issuance Workflow:**
  1. Generate CSR on the target server/keystore
  2. Submit CSR to internal bank PKI/Security Operations team
  3. Automated or manual role-based verification by internal team
  4. Internal CA signs and returns certificate
  5. Average turnaround: Hours instead of business days

> [!TIP]
> Internal CAs allow banks to issue certificates with custom SAN profiles and extended lifecycles while enforcing strict internal revocation lists (CRLs/OCSP).

---

## CA Comparison Matrix

| Attribute | Public CA (External) | Internal/Private CA |
| :--- | :--- | :--- |
| **Trust Boundary** | Global / Universal | Internal Bank Infrastructure Only |
| **Typical Vendors / Tech** | DigiCert, Sectigo, GlobalSign | Microsoft AD CS, HashiCorp Vault, OpenSSL |
| **Target Audience** | External Customers, Public APIs | Bank Employees, Backend Microservices, DBs |
| **Cost Profile** | Per-cert subscription (₹5,000–₹1,00,000/yr) | Free (Infrastructure/management cost) |
| **Turnaround Time** | Days (Identity/Org validation required) | Hours (Internal workflow) |
| **Root Distribution** | Embedded in OS/Browsers | Enterprise GPO / Fleet Management |

---

## Architecture Reference: BankCell01

The following matrix details the active certificate mapping across network ports and communication paths for the reference cell `BankCell01`:

### Certificate Mapping Matrix

| App / Component | CA Used | Port | Purpose / Access Scope |
| :--- | :--- | :--- | :--- |
| **IHS (Internet Banking)** | DigiCert | `443` | External customer access |
| **IHS (Payment Gateway)** | DigiCert | `443` | External customer & merchant access |
| **WAS Admin Console (DMGR)** | Bank Internal CA | `9043` | Internal administrators only |
| **WAS PaymentCluster** | Bank Internal CA | `9443` | Internal application transport |
| **Node Agent ↔ DMGR** | Self-signed | `9353` | IBM WebSphere internal cell sync |
| **WAS → LDAP (`ldaps`)** | Bank Internal CA | `636` | Internal directory authentication |
| **WAS → MQ (SSL Channel)** | Bank Internal CA | — | Internal message bus connectivity |
| **WAS → DB2 (SSL JDBC)** | Bank Internal CA | — | Encrypted database transactions |