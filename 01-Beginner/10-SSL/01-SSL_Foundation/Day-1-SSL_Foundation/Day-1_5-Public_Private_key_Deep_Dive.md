# Public/Private Key Pair — The Deep Dive

> [!IMPORTANT]
> This is the foundational section of the SSL/TLS architecture. Understanding asymmetric key mechanics resolves the majority of SSL configuration, keystore pairing, and handshake issues encountered in WebSphere and enterprise middleware environments.

---

## 1. The Mailbox Analogy

Think of asymmetric cryptography like a secure physical mailbox:

* **Public Key (The Slot):** Anyone can drop a letter in. It is visible to the public and not a secret.
* **Private Key (The Key):** Only the owner holds the physical key. It is never shared and remains secured inside the server.

### Operational Mechanics

* **Anyone can deposit:** Encrypt with the **Public Key**.
* **Only the owner can retrieve:** Decrypt with the **Private Key**.

When a client initiates communication:
1. The server presents its open padlock (**Public Key**).
2. The client encrypts the payload using this key.
3. Once encrypted, no party—including the client that locked it—can read the payload.
4. Only the server possessing the mathematically linked **Private Key** can decrypt and read the payload.

### Dual Functions of Key Pairs

| Function | Key Used | Verification / Decryption | Enterprise Purpose |
| :--- | :--- | :--- | :--- |
| **Encryption** | Public Key 🔓 | Only Private Key can decrypt | Securing data in transit (passwords, tokens, payloads) |
| **Digital Signature** | Private Key 🔑 | Anyone can verify with Public Key | Proving origin identity and certifying authenticity (CA signing) |

---

## 2. Keystore vs. Truststore Architecture

In enterprise Java environments such as WebSphere Application Server (WAS), keys and certificates are stored in dedicated binary keystore files.

### Keystore (`key.p12` / `key.jks`) — "This is ME"

Holds the local server's cryptographic identity:

```text
KEYSTORE (e.g., key.p12 / key.jks / key.kdb)
─────────────────────────────────────────────────────────────────
Personal Certificate Entry:
  ├── Alias: PaymentCluster_Cert
  ├── Certificate (Public Key included)  ✅
  └── Private Key                        ✅  <-- SENSITIVE ASSET

Signer Certificate Entries (Optional Trust Anchors):
  ├── Alias: DigiCert_Root_CA
  └── Certificate Only (Public Key)      ✅ (No private key)
─────────────────────────────────────────────────────────────────
```

> [!WARNING]
> The keystore is the most sensitive asset on the server. Compromise allows an adversary to impersonate the application cluster.

### Truststore (`trust.p12`) — "This is WHO I TRUST"

Holds public certificates of trusted external parties and Certificate Authorities (CAs):

```text
TRUSTSTORE (e.g., trust.p12)
─────────────────────────────────────────────────────────────────
Signer Certificate Entries:
  ├── DigiCert Root CA                  (Public key only)
  ├── Internal Enterprise Root CA       (Public key only)
  └── Identity Provider / LDAP Cert     (Public key only)
─────────────────────────────────────────────────────────────────
* Contains NO private keys.
```

> [!NOTE]
> When a client or server initiates an outbound SSL/TLS connection (e.g., to LDAP, databases, or third-party APIs), it validates the remote certificate against the truststore. A failure produces:
> `javax.net.ssl.SSLHandshakeException: PKIX path building failed`
> This indicates the remote certificate chain does not anchor to any signer inside the local truststore.

### Storage Formats

| Format | Extension | Common Usage | Notes |
| :--- | :--- | :--- | :--- |
| **PKCS#12** | `.p12` / `.pfx` | Modern WAS, JVM standard | Industry standard, portable across platforms |
| **Java KeyStore** | `.jks` | Legacy Java deployments | Proprietary Java keystore format |
| **IBM Key Database** | `.kdb` | Legacy IBM HTTP Server / WAS | Managed via IBM `gskcmd` / `iKeyman` |
| **Raw Certificate** | `.cer` / `.crt` / `.arm` | Certificate exchange | Contains public keys only; safe to distribute |

---

## 3. The Iron Rules of Private Key Management

> [!CAUTION]
> **Strict Operational Security Policies:**
> * The private key **NEVER** leaves the server file system.
> * The private key is **NEVER** sent to a Certificate Authority (CA).
> * The private key is **NEVER** transmitted over email or ticket attachments.
> * The private key is **NEVER** supplied to external vendors for troubleshooting.
> * The private key is **NEVER** committed to version control repositories (Git) or shared document stores.
> * Only the Certificate Signing Request (CSR) is transmitted to external entities.

Compromising a private key enables unauthorized traffic decryption, man-in-the-middle (MITM) attacks, and full identity impersonation. Remediating a leaked private key requires immediate certificate revocation, full key regeneration, CSR re-issuance, and emergency deployment across all infrastructure nodes.

---

## 4. The Certificate Signing Request (CSR) Lifecycle

A CSR is an encoded text file generated from the target keystore containing:
* Subject Distinguished Name (DN): Common Name (`CN`), Organization (`O`), Organizational Unit (`OU`), Country (`C`)
* Subject Alternative Names (`SAN`)
* Server **Public Key**
* Digital signature generated using the server's **Private Key** (Proof of Possession)

> [!TIP]
> A CSR **never** contains the private key. It is strictly public metadata signed by the applicant.

### End-to-End CSR and Provisioning Workflow

```text
LOCAL SERVER (e.g., BankCell01)                    CERTIFICATE AUTHORITY (CA)
───────────────────────────────                    ──────────────────────────

[Step 1: Generate Key Pair]
  ├── Private Key (Persisted in keystore)
  └── Public Key  (Exported to CSR)
           │
[Step 2: Generate CSR]
  ├── DN: CN=pay.bankingapp.com
  ├── SAN: pay.bankingapp.com, *.bankingapp.com
  └── Public Key + Signature Proof
           │
           │────────── Step 3: Transmit CSR (Safe via email/portal) ─────────►│
                                                                               │
                                                                 [Step 4: CA Verification]
                                                                   ├── Validate domain control
                                                                   ├── Validate organizational identity
                                                                   └── Sign with CA Private Key
                                                                               │
           │◄──────── Step 5: Return Signed Public Certificate ────────────────│
           │
[Step 6: Import Certificate]
  └── Import into keystore
      (Pairs with existing Private Key)
```

### Real-World Analogy

1. Fill out an identity application form with personal details and a photo (**CSR with Public Key**).
2. Submit the form to the passport authority (**CA**).
3. The authority verifies credentials and seals the document with an official stamp (**CA Signature**).
4. The signed passport (**Certificate**) is returned.
5. The applicant retains the passport; private assets such as home keys (**Private Key**) were never handed over.

### Common Implementation Pitfalls

* **Key Pair Mismatch:** Generating a new key pair during a renewal cycle without updating the associated alias, causing mismatched certificate-to-key pairings during keystore reload.
* **Missing Subject Alternative Names (SAN):** Modern TLS clients and browsers reject certificates using only the `CN` attribute. Missing `SAN` entries require complete CSR re-issuance.
* **Accidental Keystore Exposure:** Distributing the identity keystore (`.p12` / `.jks`) instead of the exported CSR file.
* **Incorrect Import Target:** Importing a signed personal certificate into the truststore rather than the keystore holding the matching private key.
* **Hostname Mismatch:** Generating the CSR using internal server hostnames instead of public-facing endpoints, resulting in host validation failures.

---

## 5. Reference Summary

```text
PUBLIC KEY   = Mailbox Slot  --> Unrestricted distribution, included in certificates.
PRIVATE KEY  = Mailbox Key   --> Retained inside keystore, never exported or transmitted.

KEYSTORE     = Server Identity (Personal Certificate + Private Key)
TRUSTSTORE   = Trust Anchors (CA Signer Certificates, Public Keys only)

CSR CONTENT  = CN + SAN + Organization details + Public Key (Plain text, shareable)
CA ACTION    = Signs CSR with CA private key --> Returns signed public certificate.
FINAL STEP   = Import certificate into the originating keystore to bind with the private key.

TROUBLESHOOTING:
• PKIX Path Building Failure  --> Missing CA root/intermediate certificate in Truststore.
• Bad Certificate / Untrusted --> Expired certificate, host/SAN mismatch, or self-signed cert not in client truststore.
```