# Understanding the LDAP SSL Outage: A Beginner's Masterclass

Alright, sit down. I've been doing WebSphere admin work for 25 years. I've seen servers catch fire, disks die at 2 AM, and yes — certificates expire on Friday nights. This incident you're reading about? I've lived versions of it at least five times in my career.

Let me teach you everything from zero. No jargon without explanation. By the end, you'll understand exactly what happened and why.

---

## Part 1: The Building Blocks (Start Here)

Before we touch the incident, you need to understand 5 concepts. Think of this as learning the alphabet before reading the book.

### 1.1 What is WebSphere Application Server (WAS)?

WAS is IBM's software that runs your Java applications.

Think of it as a "restaurant kitchen" for applications:
* Your banking app (the dish) is cooked inside WAS (the kitchen).
* Customers (users) order through the front (their browser).
* A bank might have many copies of WAS running at once. In this incident: 15 nodes.
* A "node" = one copy of WAS running on a server.
* Why 15? So if one dies, others keep working. This is called a cluster.

> **Real-life example:** Think of 15 coffee shops of the same brand. If one closes, you go to the next. But if the coffee bean supplier fails? All 15 shops run out of coffee. That's what happened here.

### 1.2 What is LDAP?

LDAP = Lightweight Directory Access Protocol.

In plain English: it's a phone book for your company. It stores:
* Usernames
* Passwords
* Employee/customer details
* Group memberships (who is an admin, who is a teller, etc.)

Banks don't store passwords inside WAS itself. Instead, WAS asks LDAP: "Is this username and password correct?"

> **Real-life example:** You walk into a bank. The guard (WAS) doesn't know every customer by face. He calls the head office records desk (LDAP) and asks: "Is John Smith valid? Yes or no?"

* **Active Directory (AD)** = Microsoft's version of LDAP. Same idea, different brand.
* In this incident, WAS connects to LDAP over a secure (encrypted) channel.

### 1.3 What is SSL/TLS and a "Handshake"?

SSL/TLS = the technology that scrambles (encrypts) data between two computers so nobody can eavesdrop.

Before any data is sent, the two computers do a "handshake" — a quick greeting ritual:
* **WAS:** "Hello LDAP. Let's talk securely."
* **LDAP:** "Sure. Here is my ID card (certificate) to prove I'm real."
* **WAS:** "Let me check this ID card against my list of trusted issuers..."
* **WAS:** "ID verified. Let's begin."  ✅
* *--- OR ---*
* **WAS:** "I don't trust this ID. GO AWAY." ❌ (`SSLHandshakeException`)

The error you'll see in logs: `SSLHandshakeException`  
It literally means: "The trust check failed. I refuse to talk to this server."

> **Real-life example:** A bouncer at a club checks your ID. If the ID is fake or issued by a country he doesn't recognize, you don't get in. No ID check = no entry.

### 1.4 What is a Certificate and a Certificate Authority (CA)?

* A **certificate** = a digital ID card for a server. It proves "I am really ldap.bank.com."
* A **Certificate Authority (CA)** = the office that issues and signs ID cards.

There are two kinds you must know:
* **Public CAs** (like DigiCert, Let's Encrypt) — trusted by every browser on Earth.
* **Internal/Private CAs** — your own bank runs its own CA for internal systems. Cheaper, faster, but you must manage the trust.

#### The Trust Chain (Very Important!)

The LDAP server's certificate is usually signed by a CA (like a passport signed by a government).

The chain looks like this:
```
Root CA (the ultimate authority, self-signed)
   └── Intermediate CA (optional middleman)
         └── LDAP Server Certificate (the "leaf" — the actual ID card)
```

> [!IMPORTANT]
> WAS doesn't just check the leaf. It walks up the chain to the root. If the root is expired, missing, or untrusted → the whole chain fails.

> **Real-life example:** Your driver's license (leaf) was issued by your state office (intermediate), which is authorized by the national government (root). If the national government's authority expired, your license is worthless paper — even though the license itself looks fine and hasn't expired.

### 1.5 What is a Truststore?

A truststore = the wallet where WAS keeps its list of trusted CA "signatures."

In WAS, this is usually a file called `trust.p12` (a password-protected file format, PKCS#12).

There are two opposite files, don't confuse them:

| File | What it holds | Analogy |
| :--- | :--- | :--- |
| **Keystore** | My OWN identity (my private key + my certificate) | My own ID card |
| **Truststore** | Who I am willing to TRUST (CA certificates) | My wallet of verified government seals |

When WAS connects to LDAP, WAS opens its truststore and checks: "Is LDAP's certificate signed by someone in my wallet?"

If the CA in the wallet is expired → the check fails → handshake fails.

> [!TIP]
> **Memory trick:** Keystore = "who I am." Truststore = "who I trust."

---

## Part 2: The WAS World — How It All Fits Together

### 2.1 Cells, Nodes, and the Deployment Manager

WAS has a management hierarchy. You must understand this to understand the fix.

* **Cell** = the whole kingdom. One cell contains everything.
* **Deployment Manager (DMgr)** = the king's palace. The central control console.
* **Nodes** = the individual WAS servers (the 15 copies).
* **Node Agent** = a little messenger on each node that reports to the DMgr.

#### How configuration spreads:
1. Admin makes a change on the DMgr console (e.g., add a certificate).
2. The change is saved in the cell-level master config.
3. Admin triggers "node sync."
4. The change is copied to every node's local copy.

Why this matters here: The certificate had to be fixed in ONE place (the cell truststore) and then synced to all 15 nodes. You can't fix 15 nodes one by one — too slow and error-prone.

### 2.2 Where Trust Lives in WAS

* **CellDefaultTrustStore** — the main truststore for the cell. This is where the LDAP signer certificate lived.
* **Location on disk (typical):** `profiles/Dmgr01/config/cells/<cellname>/trust.p12`
* **Navigation path in the console (memorize this — it's on every certification exam):**
  ```text
  Security > SSL certificate and key management >
  Key stores and certificates > CellDefaultTrustStore >
  Signer certificates
  ```

---

## Part 3: The Incident — What Actually Happened

Now let's walk through the disaster step by step.

### 3.1 The Setup (Before the Explosion)
* The bank used its own internal Root CA to sign the LDAP server's certificate.
* WAS's truststore (on all 15 nodes) contained a copy of that Root CA certificate.
* This had worked fine for years. Nobody checked the expiry date of the Root CA.

### 3.2 Friday Night, 11:59 PM
* The internal Root CA certificate expired.
* **Important:** Nothing crashed. No alarms rang. No logs showed anything.
* The certificate didn't "break." It just crossed a date. Like a credit card sitting in your wallet — valid at 11:58 PM, expired at 12:00 AM.

### 3.3 Saturday, 12:00 AM — The Chain Reaction
1. A user tries to log in to the banking app.
2. WAS needs to verify the password → contacts LDAP over SSL (port 636).
3. TLS handshake starts. LDAP presents its certificate.
4. WAS walks the chain: leaf → intermediate → root.
5. WAS checks the root against its truststore. Found it. But... the root is expired.
6. WAS says: "I don't trust this." → `SSLHandshakeException`
7. No LDAP connection → no password check → no login.
8. This happens on all 15 nodes simultaneously, because they all share the same expired CA copy.

**Result:** 100% of logins fail. Every customer locked out of the bank.

### 3.4 The Cruel Irony: Monitoring Said "All Green"

This is the part that should scare you. The bank DID have certificate monitoring. But:
* They monitored endpoint certificates — the leaf certificate on the LDAP server, the HTTPS certificate on the web front door.
* Those were all valid and not expiring soon. Scanners reported green.
* Nobody monitored the truststore side — the CA certificates sitting inside WAS.
* The failed component (root CA) was invisible to their tools.

> **Real-life example:** You check the expiry date on your driver's license every year (good!). But you never check whether the government itself still exists. When the "government" (root CA) expires, your valid license becomes worthless.

This is called an **asymmetric monitoring policy** — watching only one side of the handshake.

### 3.5 Why Leaf Monitoring Is Not Enough

```text
What they monitored:          What failed:

LDAP leaf cert ✅             Truststore ROOT CA ❌
HTTPS leaf cert ✅            (expired at 11:59 PM)
```

* The leaf can be perfectly valid.
* The truststore copy can be expired.
* Handshake still fails, because the chain anchor (root) must be valid on the client side too.

> [!NOTE]
> **Golden rule I've learned in 25 years:** TLS has two ends. If you only watch one end, you're checking only half the road.

---

## Part 4: The Fix — How They Recovered

### 4.1 The Diagnosis (30 minutes)
1. On-call admin sees: 100% auth failure.
2. Checks logs: `SSLHandshakeException` on all nodes.
3. Checks LDAP server: up and healthy. Leaf certificate: valid.
4. Conclusion: problem is on the WAS (client) side — the truststore.
5. Opens the truststore, finds the expired Root CA. Calls the PKI team. Gets a renewed CA certificate file (`rootCA_renewed.cer`).

### 4.2 The Recovery Steps (Memorize These 4 Steps)

#### Step 1 — Get the new certificate
* From the internal CA/PKI team, as a `.cer` file.

#### Step 2 — Import it into the Cell Truststore
* Console path:
  ```text
  Security > SSL certificate and key management > Key stores and certificates > CellDefaultTrustStore > Signer certificates
  ```
* Use **Add** (upload the `.cer`) or **Retrieve from port** (pull it live from a running server).

#### Step 3 — Synchronize all nodes
* From the DMgr console, run full node sync.
* This copies the updated truststore to all 15 nodes.
* A nodeagent on each node picks up the new config.

#### Step 4 — Verify
* Test the TLS handshake from the command line:
  ```bash
  openssl s_client -connect ldap.internal.bank:636 -CAfile /path/to/trust.p12
  ```
* Look for `Verify return code: 0 (ok)` in the output. That means trust is good.
* Then test a real login. ✅

### 4.3 The Lucky Break: No Restart Needed

In many cases, changing truststores means restarting every JVM — on 15 nodes, that's a 30–60 minute rolling restart with user disruption.

But WAS supports **dynamic reload of signer certificates** — when you change signers in the config, WAS refreshes its SSL context automatically.

This saved them ~30 minutes. Total downtime: 61 minutes.

> [!WARNING]
> **Caution from experience:** Dynamic reload works for signer changes, but don't bet your life on it. If you change the keystore (private key) or SSL config references, restarts are often still required. Always test in a non-production environment first.

---

## Part 5: The Lessons — Prevention (This Is the Real Value)

### 5.1 The Five Rules I Give Every Junior Admin

1. **Inventory everything.** You cannot monitor what you haven't listed. Build a registry of:
   * Every leaf certificate (server side)
   * Every CA/root/intermediate in every truststore (client side)
   * Every keystore on every server
2. **Monitor BOTH sides of every handshake.** Endpoint certs AND truststore signers.
3. **Alert early, not on expiry day.** Set alerts at 90, 60, 30, and 7 days before expiry. Never rely on a same-day alarm.
4. **Automate renewals.** Manual certificate work at 2 AM is how mistakes happen. Use Ansible/CI-CD to push truststore updates to all nodes.
5. **Practice the failure.** Run a game day: deliberately expire a cert in test, and run drills where you deliberately expire a test certificate, and watch how your team responds. When the real one hits at midnight, muscle memory takes over.

### 5.2 The Action Items From This Incident (Explained Simply)

The bank listed 4 action items. Here's what each really means:

| Action | Plain English | Why It Matters |
| :--- | :--- | :--- |
| **Truststore auditing across all clusters** | A script that opens every `trust.p12` on every node and reads every certificate's expiry date | Catches the exact failure that happened here |
| **CA/intermediate certs into central PKI monitoring** | Add "the signers" to the same dashboard that watches leaf certs | Closes the "asymmetric monitoring" gap |
| **Alerts at 90/60/30/7 days** | Warning bells long before the deadline | Gives weeks to fix calmly instead of minutes to panic |
| **Automate sync via Ansible/CI-CD** | Robot pushes new certs to all 15 nodes with one command | No tired human forgetting node #14 at 3 AM |

### 5.3 A Sample Monitoring Script Idea

You don't need fancy tools to start. A simple script can list expiries:

```bash
# List all certificates and expiry dates inside a truststore
keytool -list -v -keystore trust.p12 -storetype PKCS12 -storepass <password> \
  | grep -E "Alias name|Valid from"
```

* Run it against every truststore in your inventory.
* Parse the dates, compare against today.
* If any cert expires within 90 days → raise an alert.
* Schedule it daily in cron. This alone would have prevented this outage.

> [!TIP]
> `keytool` ships with the JDK on every WAS box. You already have the tool. Use it.

---
# TLS on WebSphere — Banking Audit Guide (Beginner Friendly)

> I'm your senior WebSphere admin. Let's learn this from zero.

---

## 1. What is TLS? (Simple Explanation)

* **TLS** = Transport Layer Security.
* It is a lock on the door between a browser/app and your server.
* When data travels (card numbers, passwords, account info), TLS encrypts it so nobody can read or change it in between.
* TLS replaced the older SSL. But people still say "SSL" out of habit.

### Real-life Example
* **Without TLS:** Sending a postcard (anyone on the way can read it).
* **With TLS:** Sending a locked steel box (only the receiver has the key).

---

## 2. TLS Versions — Why Versions Matter

| Version | Status | Meaning |
| :--- | :--- | :--- |
| **TLS 1.0** | ❌ Dead | Old, has known holes. Auditors hate it. |
| **TLS 1.1** | ❌ Dead | Also old. Must be disabled. |
| **TLS 1.2** | ✅ Standard | Minimum acceptable for banks. |
| **TLS 1.3** | ✅ Best | Strongest, faster. Use if possible. |

> [!TIP]
> **Simple memory trick:**  
> *"1.0 and 1.1 are retired uncles. 1.2 and 1.3 are the working sons."*

### Why disable old versions?
* TLS 1.0 has attacks like POODLE and BEAST.
* PCI-DSS (card payment rules) forbids TLS 1.0/1.1.
* RBI audits check the same thing.

---

## 3. What are Ciphers?

A **cipher suite** = the recipe for encryption.

It is a bundle of:
* Key exchange method (how keys are shared)
* Encryption algorithm (how data is locked)
* Hash function (how data is verified)

> [!NOTE]
> **Think of it like:**  
> Cipher = Lock brand + key type + tamper seal, all combined.

### Weak vs Strong Ciphers

| Cipher | Verdict | Why |
| :--- | :--- | :--- |
| **RC4** | ❌ Disable | Broken, easily attacked |
| **3DES** | ❌ Disable | Old, small key size, weak |
| **AES-128** | ✅ OK | Strong enough |
| **AES-256** | ✅ Best | Strongest common option |

> [!TIP]
> **Memory trick:**  
> *"RC4 and 3DES are rusty padlocks. AES is a bank vault."*

---

## 4. How WebSphere Handles TLS (The Big Picture)

WebSphere uses **JSSE (Java Secure Socket Extension)** under the hood.

### Key pieces you manage in the Admin Console:
* **SSL configuration** — which protocol versions and ciphers are allowed.
* **Keystore** — a file holding your private key + certificate (your identity).
* **Truststore** — a file holding certificates you trust (who you accept).

### Analogy:
* **Keystore** = your ID card (proves who you are).
* **Truststore** = your list of trusted contacts (who you believe).
* **SSL config** = your door's lock settings (which locks you accept).

### Where to find it:
`Admin Console` → `Security` → `SSL certificate and key management` → `SSL configurations` → `your config (e.g., NodeDefaultSSLSettings)` → `Quality of protection (QoP) settings`

---

## 5. How to DISABLE TLS 1.0/1.1 in WebSphere

### Method 1: Admin Console (Easiest)
1. Login to Admin Console.
2. Go to: `Security` → `SSL certificate and key management` → `SSL configurations` → `NodeDefaultSSLSettings` → `QoP settings`.
3. Remove **TLS 1.0** and **TLS 1.1** from the protocol list.
4. Keep only: **TLSv1.2** (and **TLSv1.3** if supported).
5. Save → Sync nodes → Restart servers.

### Method 2: Via wsadmin script (For many servers)
* Use a Jython script to loop over all SSL configs and set protocols.

### Method 3: java.security file (Global hammer)
Edit:
```bash
/AppServer/java/8.0/conf/security/java.security
```

Add:
```properties
jdk.tls.disabledAlgorithms=SSLv3, TLSv1, TLSv1.1, RC4, 3DES_EDE_CBC, DES
```
* This disables them JVM-wide — catches everything.

> [!WARNING]
> **Golden rule:** Always test in DEV first. Restart is mandatory for changes to take effect.

---

## 6. How to Disable Weak Ciphers

In the same **QoP settings** page:

1. Find the cipher list.
2. Remove all entries containing:
   * `RC4`
   * `3DES` / `DES`
   * `NULL` (no encryption!)
   * `EXPORT` (weak, export-grade)
3. Keep: `TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384` and similar AES/GCM suites.
4. Save → Sync → Restart.

> [!TIP]
> **Memory trick for keeping:**  
> *"Keep ECDHE + AES + GCM + SHA256+. Delete everything else."*

---

## 7. The Audit Day Checklist (What Your Scenario Showed)

When the auditor says *"Prove it"*:

### Test TLS 1.0 (must FAIL)
```bash
openssl s_client -connect pay.bankingapp.com:9443 -tls1
```
* `"handshake failure"` = good, disabled ✅
* Shows a certificate = still enabled ❌

### Test TLS 1.1 (must FAIL)
```bash
openssl s_client -connect pay.bankingapp.com:9443 -tls1_1
```

### Test TLS 1.2 (must WORK)
```bash
openssl s_client -connect pay.bankingapp.com:9443 -tls1_2
```
* Look for: `Verify return code: 0 = success` ✅

### Test weak ciphers (must FAIL)
```bash
openssl s_client -connect pay.bankingapp.com:9443 -cipher RC4
openssl s_client -connect pay.bankingapp.com:9443 -cipher 3DES
```

### Test strong cipher (must WORK)
```bash
openssl s_client -connect pay.bankingapp.com:9443 -cipher AES256
```

> [!NOTE]
> **Logic:** Show old doors are welded shut, new door opens smoothly.

---

## 8. Reading openssl Output — Cheat Sheet

| Output you see | Meaning |
| :--- | :--- |
| `handshake failure` / `alert protocol version` | Feature disabled ✅ |
| `Verify return code: 0 (ok)` | Connection worked, cert valid ✅ |
| Certificate details printed | TLS version enabled ❌ (for 1.0/1.1 tests) |
| `Verify return code: 19` | Cert issue (self-signed / expired) — separate problem |

---

## 9. Common Mistakes to Avoid

* ❌ **Changing config but not restarting** → nothing actually changes.
* ❌ **Fixing one node, forgetting others** → auditor finds the weak one.
* ❌ **Disabling TLS 1.0 in JVM but old DMGR/agent still allowing it.**
* ❌ **Leaving NULL/EXPORT ciphers in the list.**
* ❌ **Not testing from an external machine** → always verify from outside the server.
* ❌ **Changing protocols without checking client compatibility** (old ATM apps may break!).

> [!WARNING]
> **Real-world war story:** A bank disabled TLS 1.0 and their old ATM middleware stopped talking to the backend. Always inventory your clients first.

---
# Real Banking Scenario — Mutual SSL (mTLS) Setup Guide

This document outlines the standard end-to-end architecture and implementation workflow for configuring Mutual TLS (mTLS / Two-Way SSL) between a WebSphere Application Server (**WAS**) cluster hosting payment services and a backend **Core Banking API**.

---

## 1. Scenario Architecture & Overview

* **Source System:** WebSphere Application Server (PaymentCluster)
* **Destination System:** Core Banking API Service
* **Security Requirement:** Mandatory Two-Way TLS (mTLS) authentication enforced at the transport layer

```
┌─────────────────────────────┐                    ┌─────────────────────────────┐
│   WebSphere App Server      │                    │      Core Banking API       │
│     (PaymentCluster)        │                    │          (Backend)          │
│                             │                    │                             │
│  ┌───────────────────────┐  │   TLS Handshake    │  ┌───────────────────────┐  │
│  │ Keystore (.jks)       │  │───────────────────>│  │ Truststore            │  │
│  │ (PaymentCluster_Cert) │  │  Sends Client Cert │  │ (Validates WAS Cert)  │  │
│  └───────────────────────┘  │                    │  └───────────────────────┘  │
│                             │                    │                             │
│  ┌───────────────────────┐  │   TLS Handshake    │  ┌───────────────────────┐  │
│  │ Truststore (.jks)     │  │<───────────────────│  │ Keystore              │  │
│  │ (Validates Core Cert) │  │  Sends Server Cert │  │ (Core Banking Cert)   │  │
│  └───────────────────────┘  │                    │  └───────────────────────┘  │
└─────────────────────────────┘                    └─────────────────────────────┘
```

---

## 2. Configuration Matrix & Responsibility Division

| Component / Task | Responsible Team | Artifact | Purpose |
| :--- | :--- | :--- | :--- |
| **Keystore Configuration** | WAS Administration Team | `PaymentCluster.jks` | Houses private key and `PaymentCluster_Cert` for outbound client identification. |
| **Inbound Trust Configuration** | WAS Administration Team | `trust.p12` / `trust.jks` | Stores Core Banking API public certificate (or root/intermediate CA) to trust the server. |
| **Outbound SSL Configuration** | WAS Administration Team | Dynamic SSL Config / Repertoire | Sets the specific key alias to present during outbound TLS handshakes. |
| **Inbound Trust Configuration** | Core Banking API Team | Core Truststore | Stores `PaymentCluster_Cert` public cert (or issuing CA) to validate WAS incoming calls. |
| **mTLS Enforcement** | Core Banking API Team | Server SSL / API Gateway | Configures TLS listener to set client authentication to `REQUIRED` / `NEED`. |

---

## 3. Implementation Workflow

### Phase A: WebSphere Application Server (WAS) Setup

> [!NOTE]
> Ensure you have exported the public portion of the `PaymentCluster_Cert` (`.cer`/`.pem`) without the private key to provide to the Core Banking API team before proceeding.

#### Step 1: Verify Existing WAS Keystore
WAS must have an operational keystore containing the client certificate and private key.
* **Keystore Name:** `PaymentCluster.jks`
* **Entry Alias:** `PaymentCluster_Cert`
* **Contents:** Private Key + Full Certificate Chain

```bash
# Verify the presence of the private key and cert chain in the WAS keystore
keytool -list -v -keystore PaymentCluster.jks -alias PaymentCluster_Cert
```

#### Step 2: Import Core Banking API Certificate into WAS Truststore
Import the public certificate (or intermediate/root CA) provided by the Core Banking API team into the truststore mapped to the `PaymentCluster`.

1. Navigate to **IBM WebSphere Integrated Solutions Console**:
   * **Security** > **SSL certificate and key management** > **Key stores and certificates**
2. Select your cell/node truststore (e.g., `CellTrustStore` or custom cluster truststore).
3. Under **Additional Properties**, click **Signer certificates** > **Add**.
4. Specify:
   * **Alias:** `CoreBankingAPI_CA`
   * **File name:** `/opt/IBM/WebSphere/certs/CoreBanking_RootCA.cer`
   * **Data type:** `Base64-encoded ASCII data`
5. Click **Apply** and save to master configuration.

```bash
# Command line alternative using keytool
keytool -importcert -trustcacerts \
  -alias CoreBankingAPI_CA \
  -file /opt/IBM/WebSphere/certs/CoreBanking_RootCA.cer \
  -keystore trust.p12 \
  -storetype PKCS12
```

#### Step 3: Configure Outbound Mutual SSL Profile in WAS
Ensure WAS presents the specific client certificate when reaching the Core Banking endpoint.

1. In the console, go to **SSL certificate and key management** > **SSL configurations**.
2. Select the SSL configuration assigned to the outbound requests (or create a dedicated profile, e.g., `PaymentToCore_SSLConfig`):
   * **Truststore:** Select the truststore updated in Step 2.
   * **Keystore:** `PaymentCluster.jks`
3. Under **Additional Properties**, click **Quality of protection (QoP) settings**:
   * **Client authentication:** Select `Supported` or `Required` (determines outbound posture).
   * **Protocol:** Ensure compatibility (e.g., `TLSv1.2` or `TLSv1.3`).
4. Set default certificate alias for outbound communication:
   * Set **Default client certificate alias** to `PaymentCluster_Cert`.
5. Map this SSL configuration to the target endpoint via **Dynamic outbound endpoint SSL configurations**:
   * **Host:** `api.corebanking.internal`
   * **Port:** `8443`
   * **SSL Configuration:** `PaymentToCore_SSLConfig`

---

### Phase B: Core Banking API Setup

> [!TIP]
> The Core Banking team should import the issuing Certificate Authority (CA) rather than leaf certificates if an internal enterprise Public Key Infrastructure (PKI) is used.

#### Step 4: Import WAS Client Certificate into Core Truststore
The Core Banking team imports the public certificate of `PaymentCluster_Cert` into their truststore.

```bash
keytool -importcert \
  -alias PaymentCluster_Cert \
  -file /etc/ssl/certs/PaymentCluster_Cert.cer \
  -keystore core_truststore.jks \
  -storepass <truststore_password>
```

#### Step 5: Enforce Client Certificate Authentication on Core Banking Server
Configure the TLS termination point (API Gateway / Reverse Proxy / Application Server) to mandate client certificates.

* **Client Authentication Mode:** Set to `REQUIRED` (or `clientAuth="true"` in Tomcat / `ssl_verify_client on;` in NGINX).

```nginx
# Example: NGINX Reverse Proxy terminating mTLS for Core Banking API
server {
    listen 8443 ssl;
    server_name api.corebanking.internal;

    ssl_certificate         /etc/ssl/certs/core_banking_server.crt;
    ssl_certificate_key     /etc/ssl/private/core_banking_server.key;

    # Truststore for validating incoming client certificates
    ssl_client_certificate  /etc/ssl/certs/was_client_trustchain.crt;
    ssl_verify_client       on;
    ssl_verify_depth        2;

    location / {
        proxy_pass http://backend_core_banking;
    }
}
```

---

## 4. End-to-End Handshake Verification

```
WAS (PaymentCluster)                               Core Banking API
        │                                                  │
        │ ──────────── 1. ClientHello ───────────────────> │
        │                                                  │
        │ <─────────── 2. ServerHello ──────────────────── │
        │ <─────────── 3. Server Certificate ───────────── │
        │ <─────────── 4. CertificateRequest (mTLS) ────── │
        │ <─────────── 5. ServerHelloDone ──────────────── │
        │                                                  │
        │ ── [Validates Server Cert against Truststore] ── │
        │                                                  │
        │ ──────────── 6. Client Certificate ────────────> │
        │              (PaymentCluster_Cert)               │
        │ ──────────── 7. CertificateVerify ─────────────> │
        │ ──────────── 8. Finished ──────────────────────> │
        │                                                  │
        │                               [Validates Client  │
        │                               Cert vs Truststore]│
        │                                                  │
        │ <─────────── 9. Finished ─────────────────────── │
        │                                                  │
        ├──────────────────────────────────────────────────┤
        │           SECURE mTLS TUNNEL ESTABLISHED         │
        └──────────────────────────────────────────────────┘
```

### Result Checklist

- [x] **WAS presents identity:** Outbound connection presents `PaymentCluster_Cert`.
- [x] **Core Banking verification:** Core Banking validates cert against its truststore $\rightarrow$ **Trusted**.
- [x] **Core Banking presents identity:** Server sends its server certificate chain.
- [x] **WAS verification:** WAS checks server cert against its configured truststore $\rightarrow$ **Trusted**.
- [x] **State:** Bidirectional cryptographic identity established; payload transfer begins.