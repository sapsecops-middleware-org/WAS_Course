# Scenario 5 — SSL Handshake Failure (Certificate Expiry Chain Reaction)

> [!NOTE]
> **Filename suggestion:** `scenario-05-ssl-handshake-failure.md`
> **Severity:** Critical (outbound payments stopped)
> **Difficulty:** Intermediate
> **Requires:** Truststore + keystore knowledge

---

## 1. Background Concepts

### 1.1 The Handshake in 30 Seconds

When WAS makes an outbound HTTPS call (e.g., to a payment gateway):

```text
1. Client (WAS):  "Hello, here are my ciphers, here is my cert if you want it"
2. Server:        "Here is MY certificate" (signed by a CA)
3. Client:        Is this cert signed by a CA in MY truststore?
                  Is it unexpired? Does the hostname match?
4. If YES → symmetric key exchange → encrypted channel opens
   If NO  → handshake FAILS → connection refused
```

### 1.2 The Two Stores (People Constantly Mix These Up)

| Store | Contains | File (WAS default) | Used For |
|---|---|---|---|
| **Keystore** | MY private key + my certificate | `key.p12` / `DummyClientKeyFile` | Proving who WE are (mTLS) |
| **Truststore** | CA / peer certificates I trust | `trust.p12` / `DummyClientTrustFile` | Verifying who THEY are |

> [!IMPORTANT]
> Outbound handshake failure = **truststore problem** 90% of the time.
> Inbound handshake failure (someone calling us) = **keystore problem**.

### 1.3 Where Certificates Live in WAS

Console: `Security → SSL certificate and key management → Key stores and certificates`
Cell default truststore: `CellDefaultTrustStore` → Signer certificates.

---

## 2. The Incident

- 10:05 AM: All outbound RTGS payments to the central gateway fail.
- App log shows:

```text
java.security.cert.CertificateExpiredException:
    NotAfter: Sun Nov 02 23:59:59 IST 2025
javax.net.ssl.SSLHandshakeException
com.ibm.jsse2.util.h: PKIX path validation failed
```

- Gateway team says: "Our certificate was renewed last night."
- Root cause: **the gateway rotated its certificate; the new signing chain is not in WAS's truststore.**

---

## 3. Diagnosis Step by Step

### Step 1 — Confirm It Is Really SSL

```bash
# Raw SSL test from the app server host
openssl s_client -connect gateway.bank.ad:443 -showcerts </dev/null
```

Look at:
- `Verify return code: 21 (unable to verify the first certificate)` → trust chain missing.
- Certificate dates and **issuer CN in the output — note the issuer's name; that's what you must import.

### Step 2 — Check What Truststore the Server Actually Uses

Console: `Security → SSL certificate and key management → SSL configurations → [DefaultSSLSettings or CellScoped settings] → Trust store`.
Confirm the file path — you may be inspecting the wrong file otherwise:

```bash
# Inspect current signers in the truststore
/opt/IBM/WebSphere/AppServer/bin/keytool -list -v \
  -keystore /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/config/cells/DSBNode01Cell/trust.p12 \
  -storetype PKCS12 -storepass <password> | grep -i "owner\|issuer\|valid"
```

### Step 3 — Confirm Expiry vs Missing Chain

| Evidence | Conclusion |
|---|---|
| `CertificateExpiredException` | You HAVE the cert; it expired → gateway rotated, import the NEW one |
| `PKIX path building failed` | You never had the new issuer → import the new **chain/CA cert** |
| `hostname verification failed` | Cert CN/SAN doesn't match the URL host → wrong cert installed server-side |

---

## 4. The Fix — Import the New Signer

### Method A — Console (Safest for Beginners)

1. Get the gateway's new certificate chain from the gateway team (`.cer`/`.pem` file), or extract it:

```bash
openssl s_client -connect gateway.bank.ad:443 -showcerts </dev/null \
  | awk '/BEGIN CERT/,/END CERT/' > gateway_new.pem
```

2. Console: `Security → SSL certificate and key management → Key stores and certificates → CellDefaultTrustStore → Signer certificates → Retrieve from port`:
   - Host: `gateway.bank.ad`, Port: `443`, Alias: `gateway_bank_2026`
   - Click **Retrieve signer information** → verify fingerprint → **Apply → Save**.
3. **Full resynchronize** nodes (remember Scenario 1 — sync trap!).
4. Restart the app server (SSL runtime caches truststore).

### Method B — Command Line

```bash
/opt/IBM/WebSphere/AppServer/bin/keytool -importcert \
  -keystore /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/config/cells/DSBNode01Cell/trust.p12 \
  -storetype PKCS12 -storepass <password> \
  -alias gateway_bank_2026 -file gateway_new.pem -trustcacerts
# Then resync + restart
```

### Verify

```bash
# After restart, confirm the handshake works
openssl s_client -connect gateway.bank.ad:443 </dev/null 2>&1 | grep "Verify return code"
# Expect: Verify return code: 0 (ok)
```

Then trigger one test outbound payment.

---

## 5. The Chain Reaction Part (Why One Cert Took Down a Whole Flow)

- The gateway's new cert was signed by a **new intermediate CA** — so importing just the leaf cert was NOT enough; the intermediate had to be imported too.
- Meanwhile, three **other** integrations (card verification, SMS gateway, settlement host) used the **same expired root** — their failures started appearing 20 minutes later as retries exhausted, making it look like a second, separate incident.
- Lesson: **check all trust relationships sharing the same CA chain**, not just the one that failed first.

## 6. Prevention Checklist

- [ ] **Certificate expiry inventory:** track expiry dates of ALL certs (ours and partners') in a register; alert at 30/15/7 days.
- [ ] Automate probe:

```bash
#!/bin/bash
echo | openssl s_client -connect gateway.bank.ad:443 2>/dev/null \
  | openssl x509 -noout -enddate
# Compare with today; alert if < 30 days
```

- [ ] Change control: gateway teams must notify WAS team **before** cert rotation.
- [ ] Import the **full chain** (root + intermediate), not just the leaf.
- [ ] Resync + restart documented as mandatory steps in the SSL change runbook.

## 7. Memory Hook

> **"Outbound fail = truststore. Inbound fail = keystore. And when a CA rotates, everyone on that chain falls — check your other integrations."**

---
