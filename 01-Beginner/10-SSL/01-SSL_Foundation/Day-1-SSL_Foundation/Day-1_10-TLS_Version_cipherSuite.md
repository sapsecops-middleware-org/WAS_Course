# TLS Versions & Cipher Suites — Explained From Absolute Zero

Hi, I'm O Alpha, your senior WebSphere trainer. Forget everything you know. Today I'll explain only two things: TLS Versions and Cipher Suites — so simply that you could explain them to your family at dinner.

Take a cup of chai ☕ and read slowly.

---

## 🟦 PART 1: FIRST — WHAT IS TLS? (The Foundation)

Before versions and ciphers, understand what TLS itself is.

### The Problem It Solves
You send your bank password over the internet. The internet is like a public road — anyone standing on the roadside can see what you carry.

Without protection:
- 👀 **Eavesdropping:** Anyone can read your data
- ✍️ **Tampering:** Anyone can change your data
- 🎭 **Fake Identity:** Anyone can pretend to be your bank

### TLS = The Armored Van 🚚
TLS (Transport Layer Security) wraps your data in armor so that on the public road:
- Nobody can read it
- Nobody can change it
- You know you're really talking to the real bank

> [!NOTE]
> **Small History Note:**
> - `SSL` = Secure Sockets Layer → the old name (versions 1, 2, 3)
> - `TLS` = Transport Layer Security → the new name (versions 1.0 onwards)
> 
> Everyone still says "SSL certificate" out of habit, but technically the world uses TLS. Think of it like: everyone says "Xerox" but the machine is actually a photocopier. Same thing.

---

## 🟦 PART 2: TLS VERSIONS — Full Detail

### 2.1 What Does "Version" Mean?
A TLS version = which generation of the security language is being used.

#### Daily-Life Analogy: Mobile Network Generations 📱

| Generation | What It Gave You |
| :--- | :--- |
| **2G** | Calls only, no security |
| **3G** | Better, but slow |
| **4G** | Fast, reliable |
| **5G** | Fastest, newest features |

**Key point:** A 5G phone cannot talk to a 2G tower if the tower doesn't support it. Both sides must support the same generation.

TLS works exactly the same way:
- Browser says: *"I know 1.2 and 1.3."*
- Server says: *"I know 1.0 and 1.2."*
- They pick the common one: **1.2**.

If there's no common version → connection fails immediately.

### 2.2 The Full Version Timeline

| Year | Version | Status | Practical Impact |
| :--- | :--- | :--- | :--- |
| **1995** | SSL 2.0 | ☠️ DEAD | Broken badly, never use |
| **1996** | SSL 3.0 | ☠️ DEAD | Killed by POODLE attack (2014) |
| **1999** | TLS 1.0 | 🚫 BANNED | Too weak for banking today |
| **2006** | TLS 1.1 | 🚫 BANNED | Slightly better, still weak |
| **2008** | TLS 1.2 | ✅ STANDARD | Today's minimum everywhere |
| **2018** | TLS 1.3 | ✅ BEST | Fastest + safest, banks adopting |

> [!TIP]
> **Simple memory rule:** The older the version, the weaker the security. Hackers broke the old ones. So the industry bans old versions one by one — like old currencies being withdrawn.

### 2.3 WHY Was Each Old Version Banned? (Understand the Reasons)

#### ☠️ SSL 2.0 and SSL 3.0 — The POODLE Attack
POODLE (2014) — yes, a funny name (it stands for something technical, but remember the dog 🐩).

How it worked in simple words:
1. Browser and server start a modern, strong TLS 1.2 connection.
2. A hacker sitting in the middle (like on the same coffee-shop WiFi) tricks them:
   > *"Psst… the other side only understands the OLD language (SSL 3.0)"*
3. The connection downgrades to SSL 3.0.
4. SSL 3.0 is broken → hacker can now read and steal the session cookies.

**Why session cookies matter:** When you log into internet banking, the bank gives your browser a "cookie" — like a wristband at a concert 🎫. Show the wristband, you're in — no password needed. If a hacker steals your wristband, he IS you now. He can transfer money without knowing your password.

This is called a **downgrade attack** — forcing two parties to speak a weaker language.

#### 🚫 TLS 1.0 and 1.1 — Too Weak by Modern Math
These weren't killed by one dramatic attack. They simply became too weak over time:
- Computers got faster → old encryption became easier to crack
- Known weaknesses (like BEAST attack on TLS 1.0)
- They allow old, broken ciphers (you'll see why that matters in Part 3)

#### 🏦 Who Enforces the Ban? — PCI-DSS
PCI-DSS = Payment Card Industry Data Security Standard.

Think of it as the rulebook that all banks must follow to be allowed to handle card payments. (Visa/Mastercard created it — so banks MUST obey.)

**Its rule:**
> "All systems touching card payments must disable SSL 3.0, TLS 1.0, and TLS 1.1. TLS 1.2 minimum."

**What happens if a bank ignores it:**
- Fails the PCI audit
- Can legally not process card payments 💳❌
- That means the bank's card business stops. Terrifying. So banks take this seriously.

### 2.4 TLS 1.3 — Why Is It "5G"? (Two Big Improvements)

#### Improvement 1: FASTER (Fewer Round Trips)
Round trip = one message sent + one reply received. Each round trip takes time.

**TLS 1.2 handshake (2 round trips):**
```text
Browser ──── "Hello" ────────────────► Server
Browser ◄─── "Hello, let's agree..." ─ Server
Browser ──── "Here's my key stuff" ──► Server
Browser ◄─── "Finished" ───────────── Server
═══════ NOW data starts flowing ═══════
```

**TLS 1.3 handshake (1 round trip):**
```text
Browser ──── "Hello + here's my key stuff already!" ► Server
Browser ◄─── "Hello + Finished" ──────────────────── Server
═══════ NOW data starts flowing ═══════
```

**The clever trick:** In TLS 1.3, the browser doesn't wait. It puts its key-exchange material inside the very first "Hello" message. Server replies with everything needed in one go.
- **Result:** ~50% faster handshake.
- **🏦 Banking impact:** In UPI payments, millions of connections happen every minute. Every millisecond saved × millions of payments = huge. That's why HDFC, ICICI, SBI are all moving to TLS 1.3.

#### Improvement 2: SIMPLER = SAFER
TLS 1.3 deleted all weak ciphers from its menu.

Think of it like a restaurant that removed all the unhealthy dishes — now whatever you order is good. With TLS 1.3, you cannot accidentally choose a broken cipher. That's a security feature in itself.

### 2.5 Your Job as a WAS Admin (Practical)
In WebSphere SSL configuration → protocol settings:
- ✅ Enable **TLS 1.2** (minimum)
- ✅ Enable **TLS 1.3** (if all connected systems support it)
- ❌ Disable **SSL 2.0**, **SSL 3.0**, **TLS 1.0**, **TLS 1.1**

> [!WARNING]
> Before turning off TLS 1.0 on a server, check: *"Do any OLD clients still connect to me using TLS 1.0?"* (old legacy apps often do). If yes, fix those clients first — otherwise you'll break them and get a 3 AM call.

---

## 🟦 PART 3: CIPHER SUITES — Full Detail

### 3.1 What Is a Cipher?
A cipher = an algorithm = a mathematical recipe for scrambling data.

```text
Plain text:   Transfer 50000
After cipher: xK9#mP2$vLq8@zW...
```

Only someone with the right key can unscramble it back.

### 3.2 So What Is a Cipher SUITE?
Encrypting is not ONE job. A secure connection needs 4 different jobs done:
1. 🔑 **Key Exchange:** Share a secret key safely
2. 🪪 **Authentication:** Prove who you are
3. 🔒 **Bulk Encryption:** Scramble the actual data
4. 🏷️ **Integrity/Hash:** Detect if data was tampered

One single algorithm can't do all 4 jobs well. So we combine 4 algorithms — one per job.

> **A cipher suite = the combination of 4 algorithms, given one name.**

### 3.3 The Thali Analogy 🍽️ (This Will Stick Forever)
When you order a "Special Thali", you don't order rice, dal, curry, roti, sweet separately. One name → whole combo delivered.

```text
"Special Thali" = rice + dal + paneer curry + 3 roti + gulab jamun

"TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"
 = ECDHE + RSA + AES_256_GCM + SHA384
```

One name. Four items inside. That's all a cipher suite is.

### 3.4 How to READ a Cipher Suite Name (Go Slowly, Piece by Piece)

```text
TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
```

Let's split it:
- `TLS` → "This is a TLS cipher suite" (just a label)
- `ECDHE` → Job 1: KEY EXCHANGE 🔑
- `RSA` → Job 2: AUTHENTICATION 🪪
- `AES_256_GCM` → Job 3: ENCRYPTION 🔒
- `SHA384` → Job 4: INTEGRITY/HASH 🏷️

#### Each Piece Explained Like You're 10:

##### 🔑 ECDHE — Key Exchange
**Question it answers:** *"How do both sides get the same secret key, without ever sending the key over the network?"*

**Analogy — The color mixing trick 🎨:**
1. You pick yellow, server picks red (both secret).
2. Both publicly mix in a common base color (say, white).
3. You send yellow+white; server sends red+white — publicly visible.
4. You add your secret yellow to their mix; server adds its secret red.
5. Both end up with the SAME orange — but the thief on the road only saw yellow-white and red-white. He cannot figure out the orange without knowing a secret color.

That's the mathematical magic ECDHE does. The **E** at the end = **Ephemeral** = temporary key, thrown away after use. That gives forward secrecy (explained in 3.6).

##### 🪪 RSA — Authentication
**Question it answers:** *"How do I know this server is really the bank and not a fake?"*

Uses the certificate's public/private key pair. The server proves it owns the certificate by doing something only the private key holder can do.

##### 🔒 AES_256_GCM — Bulk Encryption
**Question it answers:** *"How is the actual data scrambled?"*
- **AES:** The world's most trusted encryption algorithm (banks, governments, everyone uses it)
- **256:** Key size in bits. Bigger number = stronger lock. (128 is also fine and fast; 256 is the strongest common option)
- **GCM:** A modern "mode" of using AES. Bonus: GCM encrypts AND checks integrity at the same time — two jobs in one, very fast.

##### 🏷️ SHA384 — Hash (Integrity)
**Question it answers:** *"Did anyone tamper with the data on the way?"*

**Analogy — Tamper-evident seal on medicine 💊:**
When you buy medicine, the seal proves nobody opened the bottle. SHA is a mathematical seal:
1. Data is passed through SHA → produces a unique "fingerprint" number.
2. Receiver does the same calculation → must get the SAME fingerprint.
3. Different fingerprint = someone changed the data in transit 🚨.

`384` = the fingerprint length in bits (`SHA256` = 256 bits, `SHA384` = 384 bits — bigger = stronger).

### 3.5 Memory Trick — Burn This into Your Brain:
```text
🔑 KEY + 🪪 ID + 🔒 LOCK + 🏷️ SEAL
```
> Every cipher suite = these four answers bundled under one name.
> When you see any cipher suite name, just ask: "Which part is the key exchange? Which is the ID check? Which is the lock? Which is the seal?" — and you can read ANY cipher name.

### 3.6 Why Is Plain "RSA" Key Exchange Bad? (Forward Secrecy — Simply)
You'll see two kinds of key exchange: ECDHE and plain RSA.

#### Plain RSA Key Exchange Problem:
1. Browser encrypts the secret key with the server's public key → sends it.
2. A hacker records this encrypted traffic today (cannot read it yet).
3. 5 years later, the server's private key leaks (server stolen, misconfigured, etc.).
4. Hacker decrypts the old recorded traffic → ALL past data exposed 😱.

#### ECDHE (with the E = Ephemeral):
- Keys are created fresh for each session and destroyed after.
- Even if the private key leaks tomorrow, old recorded traffic stays safe forever.
- This property is called **Forward Secrecy**.

> [!NOTE]
> **Forward secrecy in one line:** ECDHE = burn-after-use keys. Even if the master key leaks tomorrow, yesterday's conversations stay encrypted forever. 🔥
> 
> This is exactly why PCI-DSS and bank security teams love ECDHE suites and dislike plain RSA key exchange.

### 3.7 Strong vs Weak Cipher Suites — The Audit List

#### ✅ STRONG (These Should Be in Your Config):
```text
TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384   ← Best choice (gold standard)
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256   ← Good (fast, still strong)
TLS_DHE_RSA_WITH_AES_256_GCM_SHA384     ← Good (DHE = same as ECDHE but slower, no "E" for elliptic)
TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384 ← Good (if using ECDSA certificates)
```
*Pattern to notice: Strong suites = ECDHE/DHE key exchange + AES with GCM + SHA256/384.*

#### ❌ WEAK (BANNED in Banks — Auditors Will Flag Every One of These):

| Cipher Pattern | Why It's Banned (Simple Reason) |
| :--- | :--- |
| `...WITH_RC4_128_SHA` | **RC4** — the math is broken; hackers can predict parts of the encrypted stream |
| `...WITH_DES_CBC_SHA` | **DES** — key is only 56 bits; modern computers crack it in hours |
| `...WITH_3DES_EDE_CBC_SHA` | **3DES** — old DES patched 3 times; SWEET32 attack (2016) broke it |
| `...WITH_NULL_SHA` | **NULL** = NO encryption at all! Data travels as plain text 🤯 |
| `...EXPORT_WITH_RC4_40_MD5` | **Export grade** — 40-bit key, deliberately weak from 1990s US export laws; breakable in minutes |
| `...WITH_AES_CBC_SHA1` | **SHA1** — broken seal; CBC mode also has known weaknesses |

> [!WARNING]
> **Two dangerous names to recognize instantly:**
> - **"NULL"** in a cipher name → means no encryption. Seeing this enabled = instant audit failure.
> - **"EXPORT"** in a cipher name → means intentionally crippled 1990s cryptography. Instant ban.

**The FREAK connection (real-world example):**
Because export ciphers existed, hackers built the FREAK attack (2015): trick modern servers into downgrading to 40-bit export ciphers, then crack them in minutes with cloud computing. This is why even unused weak ciphers must be removed, not just ignored — their mere existence in your config enables downgrade attacks (just like POODLE abused old TLS versions).

### 3.8 The Golden Rule for Cipher Lists:
> **Short list = strong server. Long list = weak server.**
> 
> A hardened bank server typically enables just 3–5 strong suites (all ECDHE + AES-GCM + SHA256/384) and nothing else.

---

## 🟦 PART 4: THE #1 PRODUCTION FAILURE — "No Cipher Suites in Common"

This is the error you WILL face in your career. Learn it deeply.

### 4.1 Remember the Rule from Part 2?
Both sides must agree on:
1. A **TLS version** AND
2. One common **cipher suite**.

If no match on either → connection refuses to start.

### 4.2 The Story — How It Happens in a Real Bank
**The setup (before):**
```text
WAS (BankCellL01) offers:      MQ server accepts:
  AES_256_GCM                    AES_256_GCM  ✅
  AES_128_GCM                    AES_128_GCM  ✅
  3DES                           3DES         ✅
```
*Everything works. Payments flow. 🟢*

Then the security team does "hardening" on MQ:
> *"3DES is weak! Remove all weak ciphers from MQ!"*

MQ now accepts ONLY:
```text
  ECDHE_RSA_AES_256_GCM_SHA384   ← only this
```

**After hardening:**
```text
WAS offers:   AES_256, AES_128, 3DES
MQ accepts:   ECDHE_RSA_AES_256 only

NO OVERLAP → ❌ HANDSHAKE FAILS
```

### 4.3 The Conversation Between the Two Servers:
```text
WAS: "Hello! I speak TLS 1.2, AES_256, AES_128, 3DES"
MQ:  "Hello! I speak TLS 1.2... but I only accept ECDHE_AES_256"
MQ:  "We have nothing in common. Goodbye." 👋
```

### 4.4 What You See:
```text
SSLHandshakeException: no cipher suites in common
    at com.ibm.ws.ssl...
```

**And the business impact:**
- ❌ MQ channel is DOWN
- ❌ Payment messages not flowing
- ❌ Transactions failing
- ❌ Phone rings 📞 (possibly at 2 AM)

### 4.5 🧠 The Diagnosis Trick (Memorize This Table)

| Error Message | Root Cause | What to Fix |
| :--- | :--- | :--- |
| `no protocols in common` | TLS version mismatch | Fix protocol list (enable common version) |
| `no cipher suites in common` | Cipher list mismatch | Add common cipher suite |
| `certificate expired / PKIX / untrusted` | Certificate problem | Fix cert or import chain |
| `hostname verification failed` | Cert name ≠ server name | Fix certificate's CN/SAN |

*Read the error message carefully — it literally tells you WHICH agreement failed.*

### 4.6 The Fix (2 Options)

#### Option 1 — Fix WAS Side ✅ (Preferred)
Add the strong cipher to WAS so both sides share it:

```text
WAS Console →
  Security → SSL certificate and key management →
  SSL configurations → [your SSL config] →
  Quality of protection (QoP) settings → Cipher suites →
  Add: SSL_ECDHE_RSA_AES_256_GCM_SHA384
  → Save → Sync nodes → Restart servers
```

*(Note: IBM WAS names ciphers slightly differently — `SSL_ECDHE_RSA_AES_256_GCM_SHA384` instead of `TLS_...`. Don't panic when you see this; it's the same suite.)*

**Result:**
```text
WAS offers:   AES_256, AES_128, 3DES, ECDHE_AES_256  ← added
MQ accepts:   ECDHE_AES_256
              ✅ MATCH → Channel UP → Payments flow 🟢
```

#### Option 2 — Fix MQ Side (Rarely Allowed)
Add a common cipher back on MQ.

> [!CAUTION]
> Usually you can't — the security team owns the MQ config now, and they'll refuse to re-add weak ciphers.

### 4.7 🚨 The WRONG Fix — Never Do This:
> *"Let's just re-enable 3DES on MQ so the old WAS can connect."*

This "fixes" the connection but:
- ❌ You fail the next PCI/security audit
- ❌ You leave a downgrade-attack door open
- ❌ Security team finds out → very bad meeting for you

> [!WARNING]
> **Golden Rule:** Never fix a cipher mismatch by ADDING WEAK ciphers to the server. Always ADD the STRONG cipher to the client (WAS).

### 4.8 🧠 The Senior Admin's Prevention Habit:
Before ANY hardening change on any server, ask two questions:
1. *"Which TLS versions will remain enabled after the change?"*
2. *"Which cipher suites will remain, and does my WAS config have at least ONE of them?"*

Better still — test before production:
```bash
openssl s_client -connect mqserver:1414 -tls1_2 -cipher ECDHE-RSA-AES256-GCM-SHA384
```
If this returns a successful handshake → WAS with the same suite will connect fine.

---

## 🟦 PART 5: FINAL RECAP CARD 📋 (Screenshot This)

### TLS Versions:
- `SSL 2.0/3.0`, `TLS 1.0/1.1` → ☠️ **BAN THEM** (POODLE, PCI-DSS)
- `TLS 1.2` → ✅ **Minimum standard**
- `TLS 1.3` → ✅ **Best:** 1 round trip, weak ciphers deleted
- **Rule:** BOTH sides must support a common version

### Cipher Suites:
One name = 4 tools bundled:
```text
🔑 KEY  = key exchange  (ECDHE = best, forward secrecy)
🪪 ID   = authentication (RSA or ECDSA)
🔒 LOCK = encryption    (AES_128/256_GCM = best)
🏷️ SEAL = integrity     (SHA256/384 = best)
```

- **Banned words in a cipher name:** `RC4`, `DES`, `3DES`, `NULL`, `EXPORT`, `MD5`, `SHA1`
- **Golden rule:** SHORT cipher list = STRONG server

### The Famous Failure:
```text
"No cipher suites in common"
= WAS offers X, server only accepts Y, no overlap
= Fix: add the strong suite on the WAS side
= NEVER fix by re-enabling weak ciphers
```

### One-Line Summary of the Whole Lesson:
- **TLS version** = which generation of the security language.
- **Cipher suite** = the 4 tools (`KEY` + `ID` + `LOCK` + `SEAL`) both sides agree to use.
- *Both sides must agree on both — or the handshake fails.*
---
## 4. Method 1 — Admin Console (The GUI Way)

Best suited for initial deployments, learning environments, and manual single-point modifications.

### Step 1 — Log in to DMGR Console
* **URL**: `https://dmgr01.internal.bank.com:9043/ibm/console`
* **Architecture**: The Deployment Manager (DMGR) represents the centralized administrative engine controlling all federated nodes and clusters.
* **Port 9043**: Default administrative console secure transport port (HTTPS).

### Step 2 — Navigate to SSL Configurations
1. Go to **Security** $\rightarrow$ **SSL certificate and key management** $\rightarrow$ **SSL configurations**.
2. Select **CellDefaultSSLSettings**.

### Step 3 — Set the TLS Protocol
1. Under the **Additional Properties** menu, click **Quality of Protection (QoP) settings**.
2. Locate the **Protocol** dropdown menu.
3. Select **TLSv1.2**.

> [!WARNING]
> Explicitly avoid and remove: `SSL`, `SSL_TLS`, `SSLv3`, `TLSv1`, and `TLSv1.1`. These legacy protocols allow protocol downgrade attacks and expose the system to vulnerabilities such as POODLE and BEAST.

### Step 4 — Set the Cipher Suites
1. Navigate to **Cipher suite properties** (within the QoP settings panel).
2. Note the two lists displayed: **Available** and **Selected**.
3. The **Selected** list defines the ciphers actively negotiated by WebSphere endpoints.

#### Approved Ciphers (Move to Selected)
* `TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384`
* `TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256`
* `TLS_RSA_WITH_AES_256_GCM_SHA256`

#### Deprecated Ciphers (Remove from Selected)
* `TLS_RSA_WITH_3DES_EDE_CBC_SHA` (3DES is cryptographically broken)
* `TLS_RSA_WITH_RC4_128_SHA` (RC4 is completely broken)

> [!NOTE]
> `ECDHE` ciphers ensure Perfect Forward Secrecy. Even if the server's private key is compromised in the future, past recorded session traffic cannot be decrypted.

### Step 5 — Save & Sync Nodes
1. Click **Apply**, then select **Save** to commit changes to the master repository.
2. Synchronize configurations across all managed nodes:
   * Navigate to **System Administration** $\rightarrow$ **Nodes**.
   * Select all target nodes.
   * Click **Full Resynchronize**.

### Step 6 — Restart
* TLS configurations are loaded into memory exclusively during JVM bootstrap.
* You must perform a full process restart of the affected JVMs (application servers, node agents, and the DMGR if scoped) for changes to take effect.

---

## 5. Method 2 — wsadmin (The Scripting Way)

Designed for automation, repeatability, CI/CD pipelines, and configuration promotion across multi-tiered environments (Dev $\rightarrow$ Test $\rightarrow$ Prod).

### Connection Setup
Execute the wsadmin command-line tool via Jython against the Deployment Manager SOAP endpoint:

```bash
ssh wasadmin@dmgr01.internal.bank.com
cd /opt/IBM/WebSphere/AppServer/bin
./wsadmin.sh -lang jython -conntype SOAP \
  -host dmgr01.internal.bank.com -port 8879 \
  -user wasadmin -password wasadmin@123
```

> [!TIP]
> Never hardcode plaintext credentials within administrative scripts. Leverage secure credential stores, profile properties, or standard runtime parameter injection.

### Script Execution Sequence

#### 1. Query Existing Configurations
```python
AdminTask.listSSLConfigs('[-scopeName (cell):BankCell01]')
```

#### 2. Inspect Active State
```python
AdminTask.getSSLConfig('[-alias CellDefaultSSLSettings -scopeName (cell):BankCell01]')
```

#### 3. Update TLS Protocol Version
```python
AdminTask.modifySSLConfig('[-alias CellDefaultSSLSettings -scopeName (cell):BankCell01 -sslProtocol TLSv1.2]')
```

#### 4. Configure Approved Cipher Suites
```python
AdminTask.modifySSLConfig('[-alias CellDefaultSSLSettings -scopeName (cell):BankCell01 -enabledCiphers TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 TLS_RSA_WITH_AES_256_GCM_SHA256]')
```

#### 5. Persist Configuration Changes
```python
AdminConfig.save()
```

#### 6. Trigger Node Synchronization
```python
nodes = AdminControl.queryNames('type=NodeSync,*').splitlines()
for node in nodes:
    AdminControl.invoke(node, 'sync')
```

#### 7. Verify Configuration Update
```python
AdminTask.getSSLConfig('[-alias CellDefaultSSLSettings -scopeName (cell):BankCell01]')
```

---

## 6. Method 3 — ssl.client.props (The File Way)

Applied specifically to **outbound SSL connections**—instances where WebSphere acts as a client consuming external endpoints (e.g., third-party web services, external payment processors).

### Key Architectural Distinction
* **Inbound SSL (Client/Browser $\rightarrow$ WAS)**: Governed by WebSphere Admin Console / XML SSL Configurations.
* **Outbound SSL (WAS $\rightarrow$ External Services)**: Governed by the client profile configuration file `ssl.client.props`.
* Enterprise compliance audits examine both traffic trajectories.

### Configuration File Path
```bash
vi /opt/IBM/WebSphere/AppServer/profiles/Dmgr01/properties/ssl.client.props
```

### Parameter Updates
Modify the protocol and cipher directives as follows:

```properties
com.ibm.ssl.protocol=TLSv1.2
com.ibm.ssl.enabledCipherSuites=\
  TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,\
  TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,\
  TLS_RSA_WITH_AES_256_GCM_SHA256
```

> [!WARNING]
> Each profile maintains an isolated `ssl.client.props` file. Ensure updates are applied to the correct profile directory (AppServer, Node, or Dmgr). Restart the associated runtime process to apply modifications.

---

## 7. How to Verify Your Work

Always validate cryptographic changes from an external network host prior to closing configuration change windows.

### OpenSSL Verification

#### 1. Validate TLS 1.2 Handshake (Expected: Success)
```bash
openssl s_client -connect dmgr01.internal.bank.com:9443 -tls1_2
```

#### 2. Validate TLS 1.1 Downgrade Rejection (Expected: Handshake Failure)
```bash
openssl s_client -connect dmgr01.internal.bank.com:9443 -tls1_1
```

### Automated Cipher Auditing

#### Nmap Cipher Enumeration
```bash
nmap --script ssl-enum-ciphers -p 9443 dmgr01.internal.bank.com
```

#### Complete Cryptographic Assessment
Execute `testssl.sh` against the secure interface to generate an audit-ready compliance report:
```bash
./testssl.sh [https://dmgr01.internal.bank.com:9443](https://dmgr01.internal.bank.com:9443)
```

---

## 8. Common Mistakes

| Mistake | Consequence | Remediation |
| :--- | :--- | :--- |
| **Updated configuration without restarting JVMs** | Runtime ignores settings. TLS contexts load exclusively during JVM initialization. | Schedule and perform an operational process restart. |
| **Omitted node synchronization** | Deployment Manager config is updated, but Node Agents continue executing outdated configurations. | Trigger a full manual synchronization across all nodes before restarting. |
| **Removed all legacy ciphers without client validation** | Service outage. Legacy consumer applications (Java 6, legacy ESBs) fail to complete handshakes. | Inventory client TLS capability matrices prior to disabling legacy ciphers. |
| **Updated inbound SSL but ignored `ssl.client.props`** | Outbound integration handshakes fail, or security auditors flag non-compliant egress calls. | Harden both inbound Admin configurations and outbound profile properties files. |
| **No configuration backup taken before modification** | Inability to revert settings during emergency production rollback scenarios. | Execute a full WebSphere configuration backup (`backupConfig.sh`) prior to execution. |
| **Hardcoded administrative credentials in scripts** | Direct violation of enterprise security policies and compliance frameworks. | Externalize credentials via secured parameter files or interactive input prompts. |