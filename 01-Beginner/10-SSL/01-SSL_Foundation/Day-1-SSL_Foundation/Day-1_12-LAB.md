# WebSphere SSL Certificate & Keystore Management Lab

A hands-on lab covering keystore and certificate management for IBM WebSphere Application Server (WAS) using **iKeyman**, **keytool**, and **openssl**.

> [!NOTE]
> All hostnames, paths, and passwords in this lab are **examples for practice only**. Never reuse lab passwords in production.

## Table of Contents

- [Lab Environment](#lab-environment)
- [Tools Overview](#tools-overview)
- [Section A: iKeyman (GUI)](#section-a-ikeyman-gui)
- [Section B: keytool (CLI)](#section-b-keytool-cli)
- [Section C: openssl](#section-c-openssl)
- [Section D: Protect and Clean Up](#section-d-protect-and-clean-up)
- [Section E: Cheat Sheet](#section-e-cheat-sheet)
- [The Complete Workflow](#the-complete-workflow)

## Lab Environment

| Item | Value |
|------|-------|
| Server | `dmgr01.internal.bank.com` |
| User | `wasadmin` |
| WAS Home | `/opt/IBM/WebSphere/AppServer` |
| Profile | `/opt/IBM/WebSphere/AppServer/profiles/Dmgr01` |
| Keystore location | `/opt/IBM/WebSphere/AppServer/profiles/Dmgr01/config/cells/BankCell01/` |

**Terminology**

- `dmgr01`: the Deployment Manager, the server that controls all other WebSphere servers in the cell.
- `Profile`: the folder holding this server's own configuration.
- `cells/BankCell01`: the cell, the top-level container for all configuration in this WebSphere setup.

## Tools Overview

| Tool | Type | Analogy |
|------|------|---------|
| iKeyman | GUI | Managing files with Windows Explorer |
| keytool | Command line | Typing `dir` / `copy` commands |
| openssl | Command line | A universal magnifying glass for certificates |

---

## Section A: iKeyman (GUI)

### A1: Start iKeyman

```bash
ssh -X wasadmin@dmgr01.internal.bank.com
cd /opt/IBM/WebSphere/AppServer/bin
./ikeyman.sh
```

| Piece | Meaning |
|-------|---------|
| `ssh` | Log in to a remote server |
| `-X` | Forward GUI windows to your local screen (X11 forwarding) |
| `./ikeyman.sh` | Run the iKeyman program from the current folder |

> [!WARNING]
> Without `-X`, iKeyman fails with a display error. Your PC also needs an X server (MobaXterm, Xming, or similar) to display the window.

> [!TIP]
> Most production servers have no GUI access. Use iKeyman for learning; on the job you will mostly use `keytool`.

### A2: Create a New Keystore

**Goal:** create an empty container file with no certificates inside.

Menu: **Key Database File → New**

| Field | Value | Why |
|-------|-------|-----|
| Key database type | `JKS` | Java Key Store, the format WebSphere prefers. (PKCS12 = universal; CMS = IBM HTTP Server) |
| File Name | `PaymentCluster.jks` | Named after what it protects |
| Location | `.../cells/BankCell01/` | Kept with the rest of the WebSphere config |

Password screen:

```text
Password : keystoreP@ss123
[x] Stash password
```

> [!IMPORTANT]
> **Always stash the password.** A stash file (`.sth`) lets WebSphere read the password automatically at startup. Without it, someone must type the password on every restart, including at 3 AM during an outage.

### A3: Create a Self-Signed Certificate

**Goal:** create a certificate signed by ourselves, with no CA involved.

Menu: **Personal Certificates → New Self-Signed**

A keystore has two sections:

| Section | Contains |
|---------|----------|
| Personal Certificates | Your own certificates, with private keys |
| Signer Certificates | Certificates of others that you trust |

| Field | Value | Notes |
|-------|-------|-------|
| Key Label (Alias) | `PaymentCluster_Cert` | A nickname used in later commands |
| Version | `X509 V3` | Leave as default |
| Key Size | `2048` | Minimum acceptable today; 1024 is banned by auditors |
| Signature Algorithm | `SHA256WithRSA` | SHA-1 is broken |
| Common Name (CN) | `pay.bankingapp.com` | **Must exactly match the URL users type** |
| Organization (O) | `BankCell01 Ltd` | Company name |
| Org Unit (OU) | `Payment Systems` | Department |
| City (L) | `Mumbai` | |
| State (ST) | `Maharashtra` | |
| Country (C) | `IN` | 2-letter code |
| Validity | `365` days | After this the certificate expires and clients get errors |

> [!TIP]
> If the CN does not match the hostname in the browser, clients get a name-mismatch error.

### A4: Inspect the Certificate

Select `PaymentCluster_Cert` and click **View/Edit**.

```text
Label    : PaymentCluster_Cert
Version  : X.509 V3
Subject  : CN=pay.bankingapp.com, ...
Issuer   : CN=pay.bankingapp.com, ...
Valid    : 01 Jan 2024 - 01 Jan 2025
Serial   : 4A:F2:3B:C1:...
Algorithm: SHA256withRSA
Key Size : 2048 bits
```

**Self-signed test**

| Condition | Meaning |
|-----------|---------|
| Subject == Issuer | Self-signed |
| Subject != Issuer | CA-signed |

**Certificate review checklist**

- [ ] CN matches the server's URL
- [ ] Validity date is in the future
- [ ] Key size is 2048 or larger
- [ ] Algorithm is SHA-256 (not SHA-1)
- [ ] Issuer is what you expect (self-signed for internal, CA for public)

---

## Section B: keytool (CLI)

### Setup (every session)

```bash
export JAVA_HOME=/opt/IBM/WebSphere/AppServer/java/8.0/jre
export PATH=$JAVA_HOME/bin:$PATH
keytool -help
```

- `export` lasts only for the current session. Add the lines to `~/.bashrc` to make them permanent.
- Use the keytool that ships with WAS, not the Linux system Java; their capabilities can differ.
- If you see `command not found`, your `PATH` is wrong.

### B1: Create Keystore and Certificate in One Command

```bash
keytool -genkeypair \
  -alias PaymentCluster_Cert \
  -keyalg RSA \
  -keysize 2048 \
  -sigalg SHA256withRSA \
  -validity 365 \
  -dname "CN=pay.bankingapp.com, OU=Payment Systems, O=BankCell01 Ltd, L=Mumbai, ST=Maharashtra, C=IN" \
  -keystore /opt/IBM/WebSphere/AppServer/profiles/Dmgr01/config/cells/BankCell01/PaymentCluster.jks \
  -storepass keystoreP@ss123 \
  -keypass keystoreP@ss123
```

| Option | Meaning |
|--------|---------|
| `-genkeypair` | Generate a private key and matching public key |
| `-alias` | Nickname (same as Key Label in iKeyman) |
| `-keyalg RSA` | Key algorithm |
| `-keysize 2048` | Key strength |
| `-sigalg SHA256withRSA` | Signature algorithm |
| `-validity 365` | Validity in days |
| `-dname "..."` | Distinguished Name: CN, OU, O, L, ST, C as one string |
| `-keystore` | Keystore file; **created automatically if it does not exist** |
| `-storepass` | Password of the keystore |
| `-keypass` | Password of the key inside the keystore |

> [!NOTE]
> In WebSphere, keep `-storepass` and `-keypass` **identical**. Different values cause confusing errors later.

> [!TIP]
> No output means success. Errors print loudly.

### B2: Inspect with keytool

**Quick listing**

```bash
keytool -list \
  -keystore /opt/.../PaymentCluster.jks \
  -storepass keystoreP@ss123
```

```text
Keystore type: JKS
Your keystore contains 1 entry

PaymentCluster_Cert, Jan 01 2024, PrivateKeyEntry,
Certificate fingerprint (SHA-256): AB:CD:EF:...
```

| Item | Meaning |
|------|---------|
| `PrivateKeyEntry` | Certificate **and** private key present. Required for a server to do SSL |
| `TrustedCertEntry` | Public certificate only. Cannot serve SSL |
| Fingerprint | Unique thumbprint. Identical fingerprints mean the same certificate |

**Detailed view**

```bash
keytool -list -v -alias PaymentCluster_Cert \
  -keystore /opt/.../PaymentCluster.jks \
  -storepass keystoreP@ss123
```

```text
Alias name: PaymentCluster_Cert
Entry type: PrivateKeyEntry
Certificate chain length: 1
Owner: CN=pay.bankingapp.com, ...
Issuer: CN=pay.bankingapp.com, ...
Valid from: Mon Jan 01 2024 until: Wed Jan 01 2025
CertAlg: SHA256withRSA, KeySize: 2048 bits
```

| Test | Result |
|------|--------|
| Owner == Issuer | Self-signed |
| Owner != Issuer | CA-signed |
| Chain length 1 | Self-signed (certificate alone) |
| Chain length 2+ | CA-signed (certificate plus CA certificates) |

### B3: Generate a CSR

A **Certificate Signing Request (CSR)** is a file containing:

- Your public key
- Your identity details (the `dname`)
- **Not** your private key, which never leaves your machine

```bash
keytool -certreq \
  -alias PaymentCluster_Cert \
  -file PaymentCluster.csr \
  -keystore /opt/.../PaymentCluster.jks \
  -storepass keystoreP@ss123
```

> [!NOTE]
> There is no `-dname` here because the identity details were already stored with the key pair in B1.

**Verify the CSR**

```bash
openssl req -in PaymentCluster.csr -noout -text
```

```text
Subject: CN=pay.bankingapp.com, OU=Payment Systems, ...
Public Key Algorithm: rsaEncryption, 2048 bit
Signature Algorithm: sha256WithRSAEncryption
```

> [!CAUTION]
> A CSR must never contain a private key. If you see one in a file you are about to send to a CA, stop immediately.

**Real-world flow**

1. Create key pair (B1)
2. Create CSR (B3)
3. Send CSR to the CA
4. CA signs it and returns a signed certificate
5. Import the signed certificate into the keystore (B5)

### B4: Simulate a CA (Lab Only)

```bash
# Step 1: Create a CA keystore with a CA certificate
# BC:ca:true = BasicConstraints, "this key may sign other certificates"
keytool -genkeypair \
  -alias LabCA \
  -keyalg RSA -keysize 2048 \
  -dname "CN=BankInternal CA, O=BankCell01 Ltd, C=IN" \
  -ext BC:ca:true \
  -validity 3650 \
  -keystore CA.jks -storepass caP@ss123

# Step 2: Export the CA's public certificate
keytool -exportcert \
  -alias LabCA \
  -file CA.crt \
  -keystore CA.jks -storepass caP@ss123

# Step 3: Sign the CSR using the CA key
keytool -gencert \
  -alias LabCA \
  -infile PaymentCluster.csr \
  -outfile PaymentCluster_signed.cer \
  -keystore CA.jks -storepass caP@ss123
```

| Flag | Meaning |
|------|---------|
| `-exportcert` | Export a public certificate from a keystore to a file |
| `-gencert` | Sign a CSR (what a CA does) |
| `-ext BC:ca:true` | Marks the key as a CA. Without it, keytool refuses to sign other certificates |

> [!NOTE]
> Do not put comments after a trailing `\` in a shell command; the line continuation breaks. Keep comments on their own lines as above.

**Verify**

```bash
keytool -printcert -file PaymentCluster_signed.cer
```

```text
Owner: CN=pay.bankingapp.com, ...
Issuer: CN=BankInternal CA, ...
```

Owner differs from Issuer, so the certificate is **CA-signed**.

### B5: Import the Signed Certificate

> [!WARNING]
> Importing the signed certificate directly fails with:
>
> `keytool error: java.lang.Exception: Failed to establish chain from reply`
>
> The keystore does not yet know the CA that signed it. Import the CA certificate **first**.

```bash
# Step 1: Import the CA certificate as a trusted signer
keytool -importcert \
  -alias LabCA \
  -file CA.crt \
  -keystore PaymentCluster.jks \
  -storepass keystoreP@ss123
# Prompt: "Trust this certificate?" -> yes

# Step 2: Import the signed certificate (replaces the self-signed one)
keytool -importcert \
  -alias PaymentCluster_Cert \
  -file PaymentCluster_signed.cer \
  -keystore PaymentCluster.jks \
  -storepass keystoreP@ss123
# Output: Certificate reply was installed in keystore
```

> [!IMPORTANT]
> The alias in Step 2 **must match** the existing alias (`PaymentCluster_Cert`). That is how keytool knows to update the existing entry instead of creating a new one.

**Verify**

```bash
keytool -list -v -alias PaymentCluster_Cert \
  -keystore PaymentCluster.jks -storepass keystoreP@ss123
```

```text
Certificate chain length: 2
Entry type: PrivateKeyEntry
Chain: [0] CN=pay.bankingapp.com
Chain: [1] CN=BankInternal CA
```

### B6: Signer Certificates (Trusting Others)

Import the public certificates of parties your server must trust (for example, a payment gateway).

```bash
keytool -importcert \
  -alias PaymentGatewayCA \
  -file gateway_ca.crt \
  -keystore PaymentCluster.jks \
  -storepass keystoreP@ss123
```

| Section | Contains | Private key? | Purpose |
|---------|----------|--------------|---------|
| Personal Cert | Your certificates | Yes | Prove your identity (SSL server) |
| Signer Cert | Others' certificates | No | Decide whom you trust |

During an SSL handshake, both sides present certificates and check them against their trusted lists. If there is no matching trust, the connection fails.

### B7: Housekeeping

**Check expiry of all certificates (monthly monitoring)**

```bash
keytool -list -v -keystore PaymentCluster.jks -storepass keystoreP@ss123 | grep -i "until"
```

**Delete a certificate**

```bash
keytool -delete -alias OldCert -keystore PaymentCluster.jks -storepass keystoreP@ss123
```

> [!CAUTION]
> Deleting a Personal certificate also deletes its private key. There is no undo. Back up first (see B8).

**Change keystore password**

```bash
keytool -storepasswd -keystore PaymentCluster.jks \
  -storepass keystoreP@ss123 -new NewP@ss456
```

> [!NOTE]
> This changes only the keystore password. If the key password differs, change it separately with `-keypasswd`. In WebSphere, keep both identical.

### B8: Backup and Restore

**Back up a keystore, including private keys, to PKCS12**

```bash
keytool -importkeystore \
  -srckeystore PaymentCluster.jks -srcstorepass keystoreP@ss123 \
  -destkeystore Backup.p12 -deststoretype PKCS12 -deststorepass backupP@ss123
```

**Export only the public certificate (safe to share)**

```bash
keytool -exportcert -alias PaymentCluster_Cert -file mycert.crt \
  -keystore PaymentCluster.jks -storepass keystoreP@ss123
```

**Restore:** run `-importkeystore` with source and destination swapped. If the destination keystore does not exist, keytool creates it with the password you pass via `-deststorepass`.

---

## Section C: openssl

`keytool` works with Java keystores. `openssl` reads raw certificate files (`.crt`, `.cer`, `.pem`) and can test live servers.

### C1: Read a Certificate File

```bash
openssl x509 -in PaymentCluster_signed.cer -text -noout
```

- `x509`: the certificate format
- `-text`: human-readable output
- `-noout`: suppress the raw encoded output

### C2: Inspect a Live Server's Certificate

```bash
openssl s_client -connect pay.bankingapp.com:9443 -showcerts
```

- `s_client` acts as an SSL client.
- `9443` is a common WebSphere HTTPS port (the default for the admin console secure port is 9043; check your configuration).

```text
Certificate chain
Verify return code: 21 (unable to verify the first certificate)
```

| Return code | Meaning |
|-------------|---------|
| `0 (ok)` | Fully trusted |
| `18` / `19` / `20` / `21` | Self-signed or unknown CA (expected in this lab) |

> [!TIP]
> This is your first tool when someone reports "SSL connection to that server is failing": it shows exactly which certificate the server presents.

### C3: Check That a Certificate Matches a Private Key

`keytool` cannot export a private key directly. Convert to PKCS12 first, then compare modulus hashes.

```bash
# Extract the private key from the PKCS12 backup
openssl pkcs12 -in Backup.p12 -nocerts -nodes -out mykey.pem

# Compare modulus hashes
openssl x509 -noout -modulus -in mycert.crt | openssl md5
openssl rsa  -noout -modulus -in mykey.pem  | openssl md5
```

If the hashes match, the certificate and key belong together. A mismatch causes "certificate and private key do not match" errors.

> [!CAUTION]
> `mykey.pem` is an unencrypted private key. Restrict permissions with `chmod 600` and delete it when finished.

---

## Section D: Protect and Clean Up

**Lock down permissions**

```bash
chmod 600 PaymentCluster.jks
chown wasadmin:wasadmin PaymentCluster.jks
ls -l PaymentCluster.jks
# Expected: -rw------- 1 wasadmin wasadmin ...
```

> [!IMPORTANT]
> Bank audits fail if a keystore is world-readable.

**Back up to a safe location**

```bash
cp PaymentCluster.jks /secure/backup/location/
```

**Lab cleanup (lab only)**

```bash
rm CA.jks CA.crt PaymentCluster.csr PaymentCluster_signed.cer
# Keep PaymentCluster.jks (your main keystore)
chmod 600 PaymentCluster.jks
```

> [!WARNING]
> In production, never `rm` keystore files. Archive them with a date instead:
>
> ```bash
> mv PaymentCluster.jks /secure/backup/PaymentCluster_20240101.jks
> ```

---

## Section E: Cheat Sheet

| Task | Command |
|------|---------|
| Create key + certificate | `keytool -genkeypair -alias X -keyalg RSA -keysize 2048 -dname "CN=..." -keystore F.jks` |
| List keystore | `keytool -list -keystore F.jks` |
| Detailed view | `keytool -list -v -alias X -keystore F.jks` |
| Read a certificate file | `keytool -printcert -file X.cer` or `openssl x509 -in X.crt -text -noout` |
| Create CSR | `keytool -certreq -alias X -file req.csr -keystore F.jks` |
| Export public certificate | `keytool -exportcert -alias X -file X.crt -keystore F.jks` |
| Import certificate | `keytool -importcert -alias X -file X.crt -keystore F.jks` |
| Delete certificate | `keytool -delete -alias X -keystore F.jks` |
| Check expiry | `keytool -list -v -keystore F.jks \| grep -i "until"` |
| Backup (JKS to PKCS12) | `keytool -importkeystore -srckeystore F.jks -destkeystore B.p12 -deststoretype PKCS12` |
| Check live server | `openssl s_client -connect host:9443` |

## The Complete Workflow

1. `genkeypair`: create a private key, public key, and self-signed certificate
2. `certreq`: create a CSR (contains no private key)
3. CA signs it: `-gencert` in the lab, a real CA in production
4. Import the **CA certificate first**: teach the keystore to trust the CA
5. Import the signed certificate: chain length becomes 2 (CA-signed)
6. `chmod 600`: lock the file down
7. Monitor expiry monthly with `grep "until"`

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `Failed to establish chain from reply` | CA certificate not in keystore | Import the CA certificate first (B5, Step 1) |
| `keytool: command not found` | `PATH` not set | Re-run the setup exports |
| iKeyman fails to open | Missing `-X` or no local X server | Reconnect with `ssh -X`; start MobaXterm/Xming |
| Browser name-mismatch error | CN does not match URL | Recreate the certificate with the correct CN |
| Server prompts for password on restart | Password not stashed | Create the stash file (`.sth`) |
| Entry shows `TrustedCertEntry` | No private key present | Import the signed reply into a `PrivateKeyEntry` alias |