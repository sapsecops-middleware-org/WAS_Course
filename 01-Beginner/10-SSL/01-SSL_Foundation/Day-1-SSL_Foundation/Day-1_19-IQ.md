# SSL/TLS & Key Compromise Interview Guide

## Q1: What is the difference between SSL and TLS? Why do we still call it SSL in banking?

### Answer (as given in interview):
SSL (Secure Sockets Layer) is the original protocol — versions 2.0 and 3.0 — both are broken and banned. TLS (Transport Layer Security) is the successor — 1.0, 1.1 are now also deprecated; banks enforce TLS 1.2 minimum, with TLS 1.3 being adopted.

We still say "SSL" in banks because the industry terminology stuck — admin consoles, config files, and managers all say "SSL config," "SSL port," "SSL cert." But the actual running protocol is TLS. A 10-year admin knows this distinction and enforces TLS 1.2 in all WebSphere SSL configurations.

---

### Simple English Explanation (for a beginner):
Think of it like this: SSL and TLS are both "security guards" that protect data traveling between your browser and a server.

* **SSL** = the OLD security guard (versions 2.0 and 3.0). Hackers found ways to break him. He is fired and banned.
* **TLS** = the NEW security guard (SSL's replacement). He does the same job but better.

#### Versions status:

| Version | Status |
| :--- | :--- |
| **SSL 2.0, SSL 3.0** | Broken — never use |
| **TLS 1.0, TLS 1.1** | Old — deprecated (banned by banks) |
| **TLS 1.2** | Minimum standard in banks |
| **TLS 1.3** | Newest, fastest — being adopted |

Why do we still say "SSL"? Because the name got stuck in everyone's mouth. Screens, documents, and managers all say "SSL certificate" even though we actually use TLS. It's like calling a fridge a "Frigidaire" — the brand name became the common word.

> [!TIP]
> **Interview one-liner:** "We say SSL out of habit, but we actually run TLS 1.2 or higher."

---

## Q2: Why does SSL use BOTH symmetric and asymmetric encryption? Why not just one?

### Answer (as given in interview):
Asymmetric encryption (RSA) solves the key-exchange problem — you can safely send a secret over a public network without prior sharing. But RSA is computationally expensive — it cannot encrypt gigabytes of banking data at speed.

Symmetric encryption (AES) is extremely fast, but you cannot safely share the key initially.

SSL combines them: RSA for the handshake (to safely exchange a session key), then AES for bulk data transfer. This gives you both security AND performance — critical in a bank processing thousands of transactions per second.

---

### Simple English Explanation:
First, understand the two types:

* **Asymmetric encryption (RSA)** — Two keys: one PUBLIC (anyone can have it), one PRIVATE (only owner has it). Like a letterbox: anyone can drop mail in (public key), only you can open it (private key).
  * ✅ **Good for:** safely exchanging a secret over the internet
  * ❌ **Bad for:** speed — very SLOW for large data
* **Symmetric encryption (AES)** — ONE key used for both locking and unlocking. Like a house key: same key locks and unlocks the door.
  * ✅ **Good for:** speed — very FAST, handles huge data
  * ❌ **Bad for:** how do you safely GIVE the key to the other person first?

#### The problem:
* RSA alone = too slow for banking data
* AES alone = key can't be shared safely

#### The solution — use both:
1. Use RSA briefly during the handshake → safely send the AES session key
2. Then use AES for the rest → fast transfer of all actual data

> [!NOTE]
> **Analogy:** RSA is like a locked safe you use ONCE to hand over a house key. After that, you use the fast house key (AES) for everything.

---

## Q3: In a bank, the private key of the payment server was accidentally shared with a vendor. What is the risk and what do you do?

### Answer (as given in interview):
This is a critical security incident. Anyone with the private key can:

* Decrypt ALL past and future SSL traffic they captured (if they were sniffing the network)
* Impersonate the server (man-in-the-middle)

#### Immediate actions:
* Raise a P1 security incident — notify InfoSec, CISO team
* Immediately revoke the certificate with the CA (Certificate Revocation)
* Generate a new key pair and CSR on the server
* Get a new cert signed by the CA and deploy it
* Rotate keystores across all nodes in the cell
* Audit logs — check if the vendor accessed anything

In WAS, this means generating a new personal cert, doing a full CSR, getting the bank's CA to sign it, and replacing the cert across DMGR + all Node keystores.

---

### Simple English Explanation:

#### Why is this so dangerous?
The private key is like the ONLY key to your bank vault. If someone else has it:
* They can read your secret conversations (decrypt captured traffic)
* They can pretend to BE your server (fake payment server = steal money)

#### What to do — step by step:

* **Step 1: Raise P1 incident**  
  Treat it like a fire alarm. Notify the security team immediately.

* **Step 2: Revoke old certificate**  
  Tell the Certificate Authority (CA): "This certificate is stolen — nobody should trust it anymore." Like canceling a stolen credit card.

* **Step 3: Generate a NEW key pair + CSR**  
  In WAS admin console:
  ```text
  Security → SSL certificate and key management → 
  Key configurations (keystores) → NodeDefaultKeyStore → 
  Personal certificates → Generate Key Pair / Create CSR
  ```

* **Step 4: Get CA to sign the new cert**  
  Send the CSR file to the bank's CA team. They send back a signed certificate.

* **Step 5: Import and replace the certificate**  
  In WAS:
  ```text
  Security → SSL certificate and key management → 
  Key configurations → NodeDefaultKeyStore → 
  Personal certificates → Receive certificate from CA
  ```
  Then synchronize nodes and restart the servers:
  ```bash
  $AdminConfig save
  syncNode.sh (from each node)
  # Restart DMGR and node agents
  stopManager.sh / startManager.sh
  stopNode.sh / startNode.sh
  ```

* **Step 6: Audit**  
  Check logs — did the vendor actually use the stolen key? Which data was accessed?

> [!TIP]
> **Interview one-liner:** "Revoke, regenerate, redeploy, and audit — treat it as a P1 because a leaked private key means the server's identity is compromised."
---
# WebSphere Application Server (WAS) SSL Certificate Guide

---

## Q1: What is the difference between CN and SAN? Which one does a browser check today?

### Simple Explanation

#### CN (Common Name):
* Think of CN as the "main name" written on the certificate
* It can hold only ONE name (example: `payments.bank.com`)
* This is the old method — used many years ago

#### SAN (Subject Alternative Name):
* Think of SAN as a list of names on the certificate
* It can hold MANY names (hostnames + IP addresses)
* Example: `payments.bank.com`, `login.bank.com`, `10.10.1.5` — all in one certificate
* This is the new method — everyone uses this now

#### Which one do browsers check today?
* **Only SAN.** Browsers completely ignore CN.
* Since Chrome 58 (2017), if a certificate has only CN and no SAN → browser shows a security error ❌

#### Real-life example:
Your bank has 3 payment websites. You must add all 3 names in SAN. Or use a wildcard like `*.bankingapp.com` — the `*` covers all sub-domains.

---

### WAS Console Steps (when generating CSR with SAN):
1. Login to WebSphere Admin Console
2. Go to: **Security** → **SSL certificate and key management** → **Key stores and certificates** → `<keystore>` → **Personal Certificate Requests**
3. Click **New**
4. Fill: Common Name, Organization, etc.
5. In the **Subject Alternative Name** / **SAN extensions** section — click and add each hostname (DNS name or IP)
6. Save and generate the CSR — then send to CA

### wsadmin:
```python
wsadmin> AdminTask.createCSRRequest('[-keyStoreName MyKeyStore -commonName payments.bank.com -subjectAltNames [[DNS payments.bank.com][DNS login.bank.com]]]')
wsadmin> AdminConfig.save()
```

> [!TIP]
> **One-line interview answer:** "CN is the old single-name field; SAN is the modern list of names. Browsers today check only SAN and ignore CN, so any certificate must have SAN entries."

---

## Q2: A developer asks you to email him the private key so he can test SSL on his laptop. What do you do?

### Simple Explanation
**Answer: NO.** Never share the private key. This is a hard rule — not your choice.

#### Why is the private key so dangerous?
The private key is like the master key of the bank's locker room. Anyone who has it can:
* 🔓 Decrypt all captured SSL traffic (read secret data)
* 🎭 Pretend to be your server (fool users into a fake bank site)
* Once a private key leaks, you must replace ALL certificates

#### What do you give the developer instead?
* **Self-signed certificate** — generate a fresh (new) certificate with its own key for his test environment only
* **Certificate without the private key** — he can trust your server but cannot impersonate it
* **Client certificate** — if he is testing mutual SSL (2-way), give him his own separate client certificate

> [!NOTE]
> In a bank: sharing private keys = audit violation = you can be fired.

---

### WAS Console Steps (create self-signed cert for dev):
1. Go to: **Security** → **SSL certificate and key management** → **Key stores and certificates** → `<keystore>` → **Personal Certificates**
2. Click **Create Self-Signed Certificate**
3. Fill Common Name (dev hostname), validity days
4. Save → export the certificate → send certificate only to developer (never the key)

### wsadmin:
```python
wsadmin> AdminTask.createSelfSignedCertificate('[-keyStoreName DevKeyStore -certificateAlias devcert -certificateCommonName dev.bank.com -certificateValidity 90]')
wsadmin> AdminConfig.save()
```

> [!TIP]
> **One-line interview answer:** "I will never email the private key — it can decrypt traffic and impersonate the server. Instead, I generate a separate self-signed certificate for dev, or share certificate-only files, or issue a dedicated client certificate for mutual SSL."

---

## Q3: Your payment server SSL certificate expires in 7 days. Walk me through the renewal process at a high level.

### Simple Explanation (step by step, like a checklist):

* **Step 1 — Note down the current certificate's details 📝**
  * CN, SAN names, CA name, key size, algorithm
  * *Why?* The new certificate must match the old one (nothing should break)
* **Step 2 — Generate a new CSR from WebSphere**
  * CSR = "Certificate Signing Request" — a request file you send to the CA
* **Step 3 — Send CSR to the CA (DigiCert or your bank's internal CA team) with a ticket/reference**
  * They sign it and send back the new certificate (usually 1–2 days for external CA)
* **Step 4 — Import the new certificate into the keystore (replace the old one)**
* **Step 5 — Import the CA chain (root + intermediate certificates) into the truststore, if not already there**
  * *Why?* The chain proves your certificate came from a trusted CA
* **Step 6 — Test on a non-production server first** — open in browser, check for errors
* **Step 7 — Raise a Change Request (CR)** — get manager/CAB approvals (bank rule)
* **Step 8 — Apply to production during the approved change window** — rolling restart so users are not disturbed
* **Step 9 — Validate in production**
  * Open the site in a browser → click the padlock 🔒 → check expiry date
  * Check WAS logs (`SystemOut.log`) for SSL errors

---

### WAS Console Steps:
* **Generate CSR:** **Security** → **SSL certificate and key management** → **Key stores and certificates** → `<keystore>` → **Personal Certificate Requests** → **New**
* **Import signed cert:** Same keystore → **Personal Certificates** → **Receive certificate from a certificate authority** → paste/point to the signed cert file
* **Import chain:** **Truststore** → **Signer Certificates** → **Add** (root + intermediate)
* **Sync nodes:** **System Administration** → **Nodes** → **Full Resynchronize**
* **Restart affected servers**

### wsadmin:
```python
wsadmin> AdminTask.createCSRRequest('[-keyStoreName PaymentKeyStore -certificateAlias paymentcert -commonName payments.bank.com -keySize 2048 -signatureAlgorithm SHA256withRSA]')
wsadmin> AdminConfig.save()

# After receiving signed cert from CA:
wsadmin> AdminTask.receiveCertificate('[-keyStoreName PaymentKeyStore -certificateAlias paymentcert -certificateFilePath /tmp/signedcert.cer -certificateFormat PEM]')
wsadmin> AdminConfig.save()
```

> [!TIP]
> **One-line interview answer:** "Note current cert details, generate CSR from WAS with same CN/SAN, send to CA, import the signed certificate into the keystore and CA chain into the truststore, test on non-prod, raise a change request, apply in prod during change window, sync nodes, restart, and verify expiry in browser and logs."
---
# IBM WebSphere Application Server (WAS): SSL/TLS Certificates, Keystores, and CA Lifecycle Guide

---

## Q1: What is the difference between a self-signed certificate and a CA-signed certificate? When do you use each in a bank?

### 🧒 Simple Explanation (Like you know nothing)

Think of certificates like ID cards.

* **Self-signed certificate = ID card made by yourself at home.**
  * You wrote your own name on it and signed it yourself.
  * Nobody else verified it.
  * If a stranger sees it, they will say: *"Why should I trust this? You made it yourself!"*
  * So browsers/other servers do not trust it by default.

* **CA-signed certificate = ID card issued by the government.**
  * A trusted authority (Certificate Authority like DigiCert) checked who you are, then signed your ID.
  * Everyone already trusts the government (CA), so they trust your ID too.
  * Browsers already have trusted CAs built-in, so they accept it without warnings.

---

### 🏦 Where each is used in a bank:

| Certificate Type | Where | Why |
| :--- | :--- | :--- |
| **Self-signed** | WAS-to-WAS communication, Node ↔ DMGR, dev/SIT/UAT environments, admin console (only admins use it) | Internal machines trust each other manually. No customer sees it. WAS ships with default self-signed certs. |
| **Public CA-signed** *(DigiCert, etc.)* | Internet banking, payment portals, mobile banking APIs | Customers' browsers must trust it automatically. Mandatory — no compromise. |
| **Internal CA-signed** | WAS → LDAP, WAS → MQ, WAS → DB (internal server-to-server) | Bank's own IT team pushes the internal CA cert to all employee laptops, so employees' browsers trust it. |

> [!TIP]
> **Interview one-liner:** *"Self-signed for internal trust, CA-signed for anything a customer touches."*

---

## Q2: What is a truststore and why is it different from a keystore? What happens if a certificate is missing from the truststore?

### 🧒 Simple Explanation

Imagine two bags:

* **Keystore = Your own ID card wallet.**
  * Contains **YOUR** certificate + **YOUR private key**.
  * You show this to others to prove *"Hey, I am really me."*
  * 🔑 **Private key** = like your signature. Never share it.

* **Truststore = Your contact list of people you trust.**
  * Contains certificates of others you are willing to talk to (CA certs, other servers' certs).
  * When someone connects to you, you check: *"Is this person in my trusted list?"*
  * **NOT in the list → you refuse to talk.** Simple.

---

### 🚨 What happens if a certificate is missing from the truststore?

The SSL handshake fails immediately and you get errors like:

```text
SSLHandshakeException: certificate_unknown
PKIX path validation failed: unable to find valid certification path
```

The connection is refused — nothing works after that.

#### 😱 Real example:
WAS connects to LDAP (Active Directory / identity system) over port 636 (LDAPS).

* If the LDAP server's CA certificate is not in WAS truststore →
* WAS rejects the LDAP server's certificate →
* Authentication fails →
* **Nobody can log in to ANY application.**
* This is one of the most common SSL outages in banking. 🔥

---

### 🛠️ How to check/fix in WAS (Console + wsadmin):

#### Admin Console steps:
1. Login to DMGR console: `https://dmgr-host:9043/ibm/console`
2. Go to **Security** → **SSL certificate and key management** → **Key stores and certificates**
3. Click your truststore (e.g., `CellDefaultTrustStore` or `NodeDefaultTrustStore`)
4. Click **Signer certificates** — here you see who you trust
5. If missing → click **Retrieve from port**, enter the remote server host + SSL port (e.g., LDAP server, port 636), click **Get Signer Certificate**, then **Apply/Save**

#### wsadmin (Jython):

```python
# Connect wsadmin
# wsadmin.bat -lang jython -username wasadmin -password <password>

# List signer certificates in a truststore
print AdminTask.listSignerCertificates('CellDefaultTrustStore')

# Retrieve (pull) a signer cert directly from a remote server port
AdminTask.retrieveSignerFromPort('[-host ldapserver.bank.com -port 636 -keyStoreName CellDefaultTrustStore -certificateAlias ldapSigner -sslConfigName CellManager/CellDefaultSSLSettings]')

# Save
AdminConfig.save()
```




## Q3: The bank's Internal CA certificate in your WAS truststore is expiring in 30 days. It signs 12 internal server certificates. What is your action plan?

### 🧒 Simple Explanation

Think of the Internal CA like a master stamp that signed 12 ID cards.

* If the stamp itself expires, all 12 cards stamped by it become useless.
* That means **ALL 12 connections break at the same time → total internal outage**.
* So this is a high-impact task, not a routine one.

---

### ✅ Step-by-step Action Plan:

#### Step 1 — Inventory 📋
Find **ALL** places where this CA cert exists.

* Check DMGR, Node01, Node02 truststores, and all cluster keystores.
* **Console:** Security → SSL certificate and key management → Key stores and certificates → [each truststore] → Signer certificates

#### Step 2 — Talk to the Internal CA team 🤝
Ask: *"Are you renewing with the SAME key pair, or issuing a NEW CA?"* This changes everything:

* **Case A — Same key pair, just extended validity ✅ (Easy)**
  * All 12 server certs stay valid (the stamp didn't change, just the expiry date).
  * You only need to: import the renewed CA cert into all truststores.
  * WAS has dynamic signer reload — no restart needed in most cases.

* **Case B — Completely NEW CA (new key pair) 🔥 (Big job)**
  * All 12 server certificates were signed by the OLD CA → they must be reissued by the new CA too.
  * Much bigger operation → raise it as a project-level change, not a small ticket.

#### Step 3 — Test in non-prod first 🧪
* Update dev/SIT/UAT first.
* Test LDAP, MQ, DB connections one by one. Confirm all work.

#### Step 4 — Raise a Change Request (CR) 📄
* Include exact steps, timing, backup steps (export truststores before changing), and rollback plan.

#### Step 5 — Timing ⏰
* Execute at least 2 weeks before expiry. Never wait for the last day.
* CA cert expiry = total outage of all 12 connections.

#### Step 6 — After update, add monitoring 📊
* Add this CA cert's expiry to the monitoring dashboard with a 60-day alert so you never get surprised again.

---

### 🛠️ Useful wsadmin commands for this task:

```python
# Check expiry date of a signer certificate
print AdminTask.getCertificateInfo('[-keyStoreName CellDefaultTrustStore -certificateAlias internalCA]')

# List all signer certs to find where the CA cert exists
print AdminTask.listSignerCertificates('NodeDefaultTrustStore')

# Add the new renewed CA cert from a file
AdminTask.addSignerCertificate('[-keyStoreName CellDefaultTrustStore -keyStoreScope (cell):BankCell01:(cell) -certificateAlias internalCA2025 -certificateFilePath /tmp/internalCA_new.cer -base64Encoded true]')

# Save configuration
AdminConfig.save()
```

---

### Backup before change (export truststore):

* **Console:** Key stores and certificates → [truststore] → Personal/Signer certificates → Export
* Or copy the truststore file physically (e.g., `trust.p12`) to a backup location first.

---
# WebSphere Application Server SSL/TLS Administration & Security Guide

## Q1: SSL Handshake, Negotiation Failures & Resolution

### Overview: What is an SSL Handshake?
When a client browser connects to a secure endpoint (`https://`), the client and server negotiate security parameters before transmitting application data. This setup sequence establishes trust and encryption keys:

1. **Client Hello**: The browser sends supported TLS protocol versions, available cipher suites, and key-exchange parameters.
2. **Server Hello**: The server evaluates the client payload and selects the highest mutually supported TLS protocol version and cipher suite.
3. **Certificate**: The server presents its digital certificate (X.509) to prove identity.
4. **Certificate Verification**: The client validates the certificate chain against its local truststore (validity dates, CA signatures, revocation lists).
5. **Key Exchange**: Both parties establish a shared master secret over the wire using asymmetric cryptography.
6. **Finished**: Both endpoints verify message authentication codes over the handshake exchange; subsequent data is symmetric-key encrypted.

### Point of Failure: "No Cipher Suites in Common"
This handshake failure occurs at **Step 2 (Server Hello)**.

* **Root Cause**: The client provides its supported cipher list in the `Client Hello`. The server compares this list against its own configured cipher suites. If the intersection of these lists is null, the server cannot negotiate a security algorithm and issues a fatal `handshake_failure` alert, terminating the socket.
* **Analogy**: Two parties attempting communication where party A speaks only English and Hindi, while party B speaks only Mandarin. Without a common language, communication terminates immediately.

---

### Remediation via WebSphere Admin Console

1. Determine client-supported ciphers using diagnostic tools (`openssl s_client`) or vendor specifications.
2. Log in to the WebSphere Integrated Solutions Console.
3. Navigate to:  
   `Security` > `SSL certificate and key management` > `SSL configurations` > `<Target_SSL_Config>` > `Quality of protection (QoP) settings`.
4. Under **Cipher suites**, identify a mutually supported cipher suite from the available column and move it to the **Selected ciphers** column.
5. Click **OK**, then click **Save** directly to the master configuration repository.
6. Synchronize configurations across all cell nodes:  
   `System administration` > `Nodes` > select nodes > `Full Resynchronize`.
7. Restart the corresponding application servers and node agents.

---

### Remediation via wsadmin (Jython)

Execute the following commands via the `wsadmin` interface to programmatically update ciphers:

```shell
wsadmin.sh -lang jython -conntype SOAP -port 8879
```

```python
# Update cipher list on target SSL configuration alias
AdminTask.modifySSLConfig('[-alias CellDefaultSSLSettings -scopeName (cell):BankCell01 -enabledCiphers "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"]')

# Persist changes to the master repository
AdminConfig.save()
```

> [!NOTE]
> Perform a full node synchronization and bounce the affected application server processes to ensure the new cipher configuration is actively bound to the transport ports.

> **Key Interview Takeaway:**  
> *"The handshake fails at Server Hello because the client and server share zero common cipher suites — the fix is to enable a mutually supported cipher on both sides."*

---

## Q2: Verifying TLS 1.0 Disablement for PCI-DSS Compliance

### Context: PCI-DSS Compliance
PCI-DSS mandate prohibits legacy cryptographic protocols (TLS 1.0 and TLS 1.1) due to structural vulnerabilities (e.g., BEAST, POODLE). Prior to an external audit, enterprise teams must generate defensible technical evidence demonstrating that legacy protocols are completely disabled across inbound and outbound channels.

### Verification Methods

| Method | Target | Advantage |
| :--- | :--- | :--- |
| `openssl s_client` | Live listening ports (`WC_defaulthost_secure`) | Provides live, black-box audit evidence |
| `wsadmin` scripting | Inbound SSL Config objects (Cell/Node/Cluster) | Audits all configuration scopes in batch |
| `ssl.client.props` | Outbound JVM configurations | Confirms compliance for backend/downstream calls |

---

### Method 1: Live Verification via OpenSSL (Black-Box Audit Evidence)

Run an OpenSSL handshake targeting the application port using only the TLS 1.0 protocol:

```shell
openssl s_client -connect pay.bankingapp.com:9443 -tls1
```

#### Interpreting Results:
* **Compliant (TLS 1.0 Disabled)**:  
  Handshake terminates with an error:
  ```text
  handshake failure / alert protocol version / write:errno=104
  ```
* **Non-Compliant (TLS 1.0 Enabled)**:  
  Handshake succeeds, displaying certificate chain details and an active cipher session.

> [!TIP]
> Capture and redirect the terminal output into an evidence artifact for auditors:
> ```shell
> openssl s_client -connect pay.bankingapp.com:9443 -tls1 > tls1_audit_evidence.txt 2>&1
> ```

---

### Method 2: Configuration Verification via wsadmin (White-Box Audit)

Query the protocol attribute across all defined SSL configuration objects:

```python
AdminTask.getSSLConfig('[-alias CellDefaultSSLSettings -scopeName (cell):BankCell01]')
```

Inspect the output parameter:
```text
protocol=TLSv1.2
```

> [!IMPORTANT]
> The audit requires verifying every SSL configuration alias across the cell hierarchy:
> * `CellDefaultSSLSettings` (Cell scope)
> * `NodeDefaultSSLSettings` (Per-node scope)
> * Custom SSL configurations (e.g., dedicated aliases for payment and customer portal clusters)

---

### Console Steps: Audit & Remediation

1. Navigate to:  
   `Security` > `SSL certificate and key management` > `SSL configurations` > `<SSL_Config_Name>` > `Quality of protection (QoP) settings`.
2. Inspect the **Protocol** property: set to `TLSv1.2` or `TLSv1.3`.
3. Click **Save** to apply to the master repository.
4. Execute `Full Resynchronize` on all nodes and restart server instances.

---

### Auditing Outbound Connections (`ssl.client.props`)

WebSphere operates as a client when communicating with downstream web services, payment gateways, and backend endpoints. Auditors require proof that outbound SSL configurations do not fall back to TLS 1.0.

Verify the configuration file on each node profile:  
`AppServer/profiles/<profile_name>/properties/ssl.client.props`

Ensure the protocol directive is explicitly configured:

```ini
com.ibm.ssl.protocol=TLSv1.2
```

> **Key Interview Takeaway:**  
> *"I verify with openssl for live proof, check all SSL configs via wsadmin, verify ssl.client.props for outbound, and save all outputs as audit evidence."*

---

## Q3: Cryptographic Architecture: ECDHE vs. Plain RSA Key Exchange

### Fundamentals of ECDHE
* **ECDH (Elliptic Curve Diffie-Hellman)**: A key-agreement protocol allowing two endpoints to negotiate a shared secret over an insecure channel using elliptic curve mathematics.
* **E (Ephemeral)**: Denotes that the key pairs used during the exchange are temporary, unique per session, and discarded immediately after handshake completion.

---

### Architectural Vulnerability: Plain RSA Key Exchange

```text
[Client]                                                        [Server]
   |                                                               |
   | ------ Encrypted Session Key (via Server RSA Public Key) ---> |
   |                                                               |
[Traffic Intercepted & Stored by Attacker]
...
(Years Later: Server RSA Private Key Stolen)
...
[Attacker Decrypts Stored Master Key] ---> [Historical Sessions Decrypted]
```

1. In a static RSA handshake, the client encrypts the premaster secret using the server's static RSA public key.
2. The server decrypts it using its long-term RSA private key.
3. **The Vulnerability**: If a threat actor records encrypted wire traffic over a long period and later compromises the server's static private key (via exfiltration, side-channel attack, or court order), they can decrypt **all historical sessions** captured with that key.
4. **Result**: Plain RSA lacks **Forward Secrecy**.

---

### Architectural Defense: ECDHE & Perfect Forward Secrecy (PFS)

```text
[Client]                                                        [Server]
   |                                                               |
   | <--- Server Generates Disposable Ephemeral Key Pair (E) ----> |
   | <--- Server Signs Ephemeral Key with Static RSA Key (Sig) --- |
   |                                                               |
[Both Derive Shared Secret Independently using ECDH Math]
[Ephemeral Keys Wiped from Memory Immediately]
...
(Years Later: Server Static Private Key Stolen)
...
[Attacker Gains Nothing: Ephemeral Keys No Longer Exist on Disk or RAM]
```

1. For each handshake, the server and client dynamically generate one-time, ephemeral public/private key pairs.
2. The static private key is used solely for **authentication/signing**, not encryption of the session secret.
3. After the shared master secret is derived via ECDH arithmetic, the ephemeral key pairs are discarded from memory.
4. **Security Assurance**: Even if the long-term private key is compromised later, prior sessions remain mathematically undecryptable.
5. **Result**: ECDHE enforces **Perfect Forward Secrecy (PFS)**.

---

### Comparison: Key Exchange Implementations

| Security Feature | Static RSA Key Exchange | ECDHE Key Exchange |
| :--- | :--- | :--- |
| **Session Key Generation** | Encrypted with server public key | Derived dynamically via Diffie-Hellman |
| **Ephemeral Keys Used** | No (Static private key used) | Yes (Generated and destroyed per session) |
| **Perfect Forward Secrecy** | ❌ No | ✅ Yes |
| **Impact of Private Key Theft** | Decrypts all historical captured traffic | Zero access to past traffic |
| **Compliance Alignment** | Disallowed by modern standards | Required by PCI-DSS / FIPS-140-3 |

---

### Anatomy of an Enterprise Banking Cipher Suite

Enterprise banking architectures implement modern cipher suites combining forward-secret key exchange, authenticated symmetric encryption, and robust hashing:

```text
TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
│   │     │    │    │       │   │
│   │     │    │    │       │   └── Integrity / MAC: HMAC using SHA-384
│   │     │    │    │       └────── Mode: Galois/Counter Mode (Authenticated Encryption)
│   │     │    │    └────────────── Bulk Encryption: Advanced Encryption Standard (256-bit)
│   │     │    └─────────────────── Bulk Cipher Prefix: With
│   │     └──────────────────────── Authentication: RSA Digital Signature Verification
│   └────────────────────────────── Key Exchange: Ephemeral Elliptic Curve Diffie-Hellman (PFS)
└────────────────────────────────── Protocol: Transport Layer Security
```

> **Key Interview Takeaway:**  
> *"ECDHE gives Perfect Forward Secrecy — a new temporary key per session means a stolen private key today can never decrypt yesterday's recorded traffic. Banks and PCI-DSS demand this to protect historical transaction data."*
---
# SSL/TLS & Mutual Authentication (mTLS) in Enterprise Banking

## Q1: What is Mutual SSL? How is it different from One-Way SSL? Give a real banking example where mTLS is mandatory.

### Understanding SSL in Simple Words
SSL/TLS functions like a lock-and-key system between two computers. Before exchanging data, they verify each other using certificates (electronic ID cards).

---

### One-Way SSL (Standard HTTPS)
* Only **ONE side shows an ID card** — the **server** presents its certificate.
* The client (browser) verifies: *"Is this really the bank's website? Yes? OK, I will talk."*
* The client authenticates using **username/password or OTP** — not via a client certificate.

**Simple Example:**
You open internet banking in your browser. The bank's website proves it is genuine (indicated by the padlock 🔒). You authenticate yourself by entering your user ID and password.

```text
Browser  ──checks──>  Bank Server certificate  ✅ (server only)
Browser  ──sends──>   username + password      (client auth by password)
```

---

### Mutual SSL (mTLS / Two-Way SSL)
* **BOTH sides show ID cards.**
* Server presents its certificate → Client verifies.
* Client ALSO presents its certificate → Server verifies.
* **No username/password required** — the certificate itself IS the credential.

```text
WAS Server  ──shows cert──>  Core Banking  ✅
WAS Server  <──shows cert──  Core Banking  ✅
```

> **Analogy:** At a bank vault door, BOTH the customer and the bank guard must present valid credentials before the door unlocks.

---

### Where is mTLS MANDATORY in Banking?

| Connection | Why mTLS is Forced |
| :--- | :--- |
| **WAS → Visa/Mastercard Gateway** | Card networks will not accept any connection without a client certificate. No certificate = connection dropped/blocked. |
| **WAS → RBI RTGS/NEFT** | Regulatory compliance mandate — mutual certificate exchange is mandatory. |
| **WAS → Core Banking System** | Internal high-security tier. Only authorized application nodes holding valid certificates are permitted access. |

---

### Key Concepts to Remember
* **Keystore:** Stores MY OWN certificate and private key (my personal identity card).
* **Truststore:** Stores public certificates of entities I TRUST (third-party identity cards or trusted CAs).
* In mTLS, **both sides must have both files configured correctly**.
* If either party's certificate is missing from the opposing truststore, the **SSL handshake fails**.

---

## Q2: You need to check the SSL certificate on the production payment server without logging into the server. How do you do it?

### Core Principle
The `openssl` utility on any remote terminal can establish a handshake to a server's SSL-enabled port and inspect the certificate presentation, matching standard client negotiation. **Direct server access/login is not required**; only network line-of-sight to the destination port is necessary.

---

### Diagnostic Steps via CLI

#### Step 1 — View Complete Certificate Chain Details
```bash
openssl s_client -connect pay.bankingapp.com:9443 -showcerts
```
*Connects to port 9443 and outputs the entire public certificate chain presented by the remote server.*

#### Step 2 — Quick Expiry Verification
```bash
echo | openssl s_client -connect pay.bankingapp.com:9443 2>/dev/null | openssl x509 -noout -dates
```
*Extracts and prints the `notBefore` and `notAfter` values to determine certificate validity and expiration dates.*

#### Step 3 — Verify Certificate Subject Identity (CN and SAN)
```bash
echo | openssl s_client -connect pay.bankingapp.com:9443 2>/dev/null | openssl x509 -noout -text | grep -E "Subject:|DNS:"
```
*Confirms entity ownership and all valid Fully Qualified Domain Names (FQDNs) bound to the certificate.*

---

### Additional Production Applications in Banking Infrastructure
* **LDAP over SSL:** Port `636`
* **IBM MQ SSL Channel:** Target MQ listener port
* **Database TLS Listeners:** Secure DB instance ports (e.g., Oracle TCPS / DB2 SSL)

> [!TIP]
> **Production Best Practice:**
> Implement an automated monitoring script via cron across management servers. If any production certificate is within 30 days of expiration, trigger an automated incident ticket to prevent unexpected outages.
>
> ```cron
> 0 8 * * * /scripts/check_cert_expiry.sh pay.bankingapp.com 9443
> ```

---

## Q3: A new WAS node was added to PaymentCluster. mTLS to Core Banking API works on all old nodes but fails on the new node. What is the likely cause and fix?

### Problem Analysis
* When a new WebSphere Application Server (WAS) profile is provisioned, it generates a **unique node-level identity certificate**.
* **Legacy Nodes:** Public certificates were pre-imported into the **Core Banking API's truststore** during initial deployment.
* **New Node:** Possesses a new, untrusted certificate unknown to the Core Banking API.
* **Result:** Core Banking terminates the session due to an untrusted identity, causing the mTLS handshake to fail.

```text
Old Node1 cert  ✅ in Core Banking truststore
Old Node2 cert  ✅ in Core Banking truststore
New Node3 cert  ❌ NOT in Core Banking truststore  →  Handshake fails
```

---

### Troubleshooting & Remediation Workflow

#### Step 1 — Inspect System Logs on the Failed Node
Monitor the runtime logs of the new node:
```bash
tail -f /opt/IBM/WebSphere/AppServer/profiles/Node03/logs/server1/SystemOut.log
```
Common signature exceptions:
* `certificate_unknown`
* `PKIX path building failed: unable to find valid certification path to requested target`

#### Step 2 — Export the Certificate from the New Node
Export the public certificate in RFC/PEM format:
```bash
cd /opt/IBM/WebSphere/AppServer/profiles/Node03/etc

keytool -export -rfc \
  -keystore DummyClientKeyStore.jks \
  -alias default \
  -file newnode3.cer
```
*(Path and store types often align with `key.p12` or `DummyClientKeyStore.jks` using alias `default` on WAS profiles; verify local deployment specs).*

Transmit `newnode3.cer` to the Core Banking platform team through secure provisioning channels.

#### Step 3 — Import Certificate into Core Banking Truststore
Execute the import on the Core Banking server truststore:
```bash
keytool -import -alias wasnode3 \
  -keystore cbTruststore.jks \
  -file newnode3.cer
```
*Reload/restart the Core Banking service or truststore if dynamic reloading is not enabled.*

#### Step 4 — Verify Handshake from the New Node
Perform an active handshake test directly from Node03:
```bash
openssl s_client -connect corebanking.bankingapp.com:9443 \
  -cert /opt/IBM/WebSphere/AppServer/profiles/Node03/etc/client.crt \
  -key /opt/IBM/WebSphere/AppServer/profiles/Node03/etc/client.key
```
*Successful remediation returns: `Verify return code: 0 (ok)`.*

---

> [!NOTE]
> ### Architecture Recommendation (Cluster-Level Identity)
> Rather than provisioning independent per-node identity certificates, implement a **single cluster-level certificate** shared across all participating nodes within the cluster tier. 
> 
> Under this model, the upstream Core Banking truststore only requires a single certificate entry. Horizontally scaling the cluster with additional nodes requires **zero configuration updates or truststore redeployments** on remote target systems.