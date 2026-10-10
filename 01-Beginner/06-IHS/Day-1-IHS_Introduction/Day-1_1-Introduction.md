# IHS, Why It Sits in Front of WAS, and the Plugin

---

## 🧱 PART 1 — What is IHS?

**IHS = IBM HTTP Server.**

It is a web server. Its job is very simple:

1. Receive the request from the browser
2. Look at the URL
3. Decide: *"Serve it myself"* or *"Send it to WAS"*

That's all. Nothing more.

### 🔵 Banking Example: The Reception Desk

Think of Citibank's reception desk.

- Customer walks in and says, *"I want to check my balance."*
- The receptionist does **NOT** process the balance.
- She just checks what you need and points you to the right counter.

**IHS is that receptionist.**

### Who made IHS?

- Made by **IBM**
- Based on **Apache HTTP Server** (a famous free web server)
- IBM took Apache → added security + IBM support + WebSphere integration → called it IHS

> 💡 **Important:** IHS uses the same config files as Apache — like `httpd.conf`. So if you know Apache, you already know 80% of IHS.

---

## 🏗️ PART 2 — in Front of WAS?

### ❌ What if there is NO IHS?

Browser talks directly to WAS:

```
Browser ──────────▶ WAS
```

This is dangerous. Here's why:

| Problem | Why it's bad (Banking example) |
|---|---|
| ❌ Security risk | WAS runs your payment and loan code. Exposing it to the internet is like leaving the bank vault door open on the street. |
| ❌ Bad at static content | WAS is slow at serving images, Like asking the loan officer to also hand out water bottles at reception. |
| ❌ SSL is costly | Decrypting HTTPS eats CPU. WAS should spend CPU on banking logic, not decryption. |
| ❌ No single entry | If you have 4 WAS servers, which one does the customer hit? Confusing. You need one front door. |

### ✅ With IHS — The Clean Way

```
Internet
   │
   ▼
[Firewall]          ← only ports 80/443 open
   │
   ▼
[IHS]               ← sits in DMZ (front door)
   │  (via Plugin)
   ▼
[WAS JVMs]          ← safe inside internal network
   │
   ▼
[Oracle/DB2 DB]
```

### What is the DMZ?

**DMZ** = a buffer zone between the internet and the bank's internal network.

> 🏠 **Real-life example:** Like the security gate of a housing society. Visitors stop at the gate. They never walk straight into your flat. If anything bad comes, it stops at the gate.

### IHS's 5 Jobs

| # | Job | Banking meaning |
|---|---|---|
| 1 | Front door | One entry point for all netbanking traffic |
| 2 | Static content | Serves the bank's logo, CSS, HTML itself — doesn't bother WAS |
| 3 | SSL termination | Decrypts HTTPS here, so WAS doesn't waste CPU |
| 4 | Security filter | Blocks bad IPs / bad URLs before they reach WAS |
| 5 | Forwarder | Sends requests like `/NetBanking/login` to WAS |

> 🏦 **Banking reality:** At Citibank, IHS sits in the DMZ. Firewall allows only ports 80/443. WAS ports (9080, 9443) are never open to the internet.

---

## 🔌 PART 3 — What is the Plugin?

Here's the catch: **IHS and WAS are two separate products. They don't talk automatically.**

So IBM made a middleman: the **WebSphere Plugin**.

### 🔵 Banking Example: The Telephone Operator

Imagine an old bank switchboard:

1. Customer calls the bank (**IHS** receives the call)
2. The operator (**Plugin**) checks: *"NetBanking department? Transferring you to extension 9080."*
3. The operator has a directory (`plugin-cfg.xml`) telling her which extension to dial.

**The Plugin = the operator. The directory = `plugin-cfg.xml`.**

### Key

- It is a **module loaded inside IHS**
- It reads `plugin-cfg.xml`
- That file tells it:
  - WAS server hostnames + ports
  - Which URL goes to which cluster
  - Load balancing rules
  - Health checks

### Simplified `plugin-cfg.xml` example

```xml
<ServerCluster Name="PaymentCluster">
    <Server Name="WAS_JVM1" CloneID="abc123">
        <Transport Hostname="was-server1.bank.internal" Port="9080"/>
    </Server>
    <Server Name="WAS_JVM2" CloneID="def456">
        <Transport Hostname="was-server2.bank.internal" Port="9080"/>
    </Server>
</ServerCluster>
```

**In plain English:** *"PaymentCluster has 2 WAS servers. Send traffic to either one."*

> 📌 We'll study `plugin-cfg.xml` deeply on Days 15–16. For now, just remember: **Plugin + plugin-cfg between IHS and WAS.**

---

## 🗺️ PART 4 — Full Picture (Memorize This!)

```
INTERNET
   │
   ▼
FIREWALL (only 80/443 open)
   │
   ▼
IHS  (in DMZ) — front door, static files, SSL, loads Plugin
   │
   │  via Plugin + plugin-cfg.xml
   ▼
INTERNAL NETWORK
   ├── WAS JVM 1 (NetBanking)
   ├── WAS JVM 2 (NetBanking)
   └── WAS JVM 3 (Payments)
        │
        ▼
   Oracle
```

---

## ✅ Quick Revision (Say These Out Loud)

- **IHS** = IBM's web server, based on Apache. It's the reception desk.
- **Why before WAS?** Security + static content + SSL + single entry.
- **DMZ** = buffer zone, like the society security gate.
- **Plugin** = translator/operator inside IHS that connects IHS to WAS.
- **plugin-cfg.xml** = the phone directory — tells the Plugin where WAS servers are.

---

## 🎯 One-Line Memory Trick

> *"IHS is the front door, the Plugin is the operator, plugin-cfg.xml is the phone directory, and WAS is the back office."*
---
# 📝 The Monday Morning Story — A Request's Journey Through the Bank

> **Senior Trainer Mode — Simple English, Banking Examples**
> I'm **Ox trainer. Today we follow ONE customer request — step by step — from the browser all the way to WAS and back.

---

## Step 1: Browser sends HTTPS request to port 443

Customer opens `https://netbanking.citibank.co.in`

- **HTTPS** = secure HTTP (data is locked/encrypted)
- **443** = the door number for HTTPS
- HTTP uses port **80**. HTTPS uses port **443**. Remember this pair.

> 💡 **Banking rule:** Banking data must ALWAYS be encrypted. That's why you see the 🔒 padlock in the browser.

---

## Step 2: Firewall — only 443 is open

**Firewall** = security guard of the bank.

It checks: *"Which door is this knock coming to?"*

- Port 443? ✅ Allowed. Come in.
- Port 9080 (WAS)? ❌ Blocked. Go away.

> 💡 Think of it like the bank guard: *"Customers enter from. Nobody enters from the back door."*

---

## Step 3: IHS receives it — SSL handshake, decrypts

**IHS** = IBM HTTP Server. It's a web server. Its job, and forward.

**SSL Handshake** = security IHS:

1. Browser: *"Prove you are really Citibank."*
2. IHS shows its SSL ID card)
3. Browser: *"OK, verified."* Now both agree on a secret key.
4. IHS decrypts the HTTPS → converts it to plain HTTP

**Why?** Because inside the network (behind the firewall), it's a trusted zone. Decryption happens here.

> 💡 Like a bank: Security guard checks customer ID at the gate. Inside the branch, staff talk freely — no need to verify every second.

---

## Step 4: IHS checks the URL — is it static or dynamic?

This is a key concept. **Write it down.**

| Type | Meaning | Example | Who handles it |
|---|---|---|---|
| **Static** | Same content for everyone | Logo image, CSS, help PDF | IHS serves it directly ⚡ |
| **Dynamic** | Different per customer | `/netbanking/login.jsp`, balance page | Must go to WAS 🏃 |

`/netbanking/login.jsp` ends in `.jsp` = Java code = **dynamic**

IHS thinks: *"I can't run Java. to WAS."*

> 💡 **Banking example:**
> - **Static** = the bank's rate board on the wall (same for everyone — reception can answer it)
> - **Dynamic** = *"What's MY balance?"* (needs a cashier — WAS — to check the system)

---

## Step 5: IHS hands it to the Plugin

- **Plugin** = a small connector sitting inside IHS
- IHS cannot talk to WAS directly by itself
- Plugin is the middleman — the token machine at reception

Flow inside IHS:

```
Browser → IHS → Plugin → WAS
```

---

## Step 6: Plugin reads plugin-cfg.xml

`plugin-cfg.xml` = the plugin's **address book / phone directory**.

It contains:

- Which clusters exist → e.g., `PaymentCluster`
- How many JVMs → e.g., 3 JVMs
- Their addresses and ports (e.g., JVM1:9081, JVM2:9082, JVM3:9083)

Plugin thinks: *"PaymentCluster has 3 JVMs. Which one gets this request?"*

> ⚠️ **Interview alert:** If `plugin-cfg.xml` is wrong or stale → requests go to dead JVMs → **503 errors**. This file is regenerated when topology changes. Remember that for your interviews.

---

## Step 7: Plugin sends to WAS JVM 2 — Round Robin

**Round-Robin** = turn by turn, one by one.

Like a bank token counter: *"Next customer to Counter 1... next to Counter 2... next to Counter 3... back to Counter 1..."*

| Request # | Goes to |
|---|---|
| 1 | JVM 1 |
| 2 | JVM 2 ✅ (our customer) |
| 3 | JVM 3 |
| 4 | JVM 1 again |
| ... | ... |

So load is shared equally. No single JVM gets tired.

**If JVM 2 is down?** Plugin skips it and sends to the next healthy JVM. Customers never know.

>ing example:** One cashier goes on lunch break. Token system the other counters. Nobody queues at a closed counter. That's **plugin failover**.

---

## Step 8: WAS JVM work

Inside JVM 2:

1. Runs the **login logic** (your deployed application — Java code)
2. Queries **Oracle DB**: *"Is this user ID valid? Is password correct?"*
3. Builds the **HTML response** (login page / next page)
4. Sends it back

---

## Step 9: Response travels

```
WAS JVM 2 → Plugin → I Browser
```

- Same path in reverse
- IHS **encrypts it again** into HTTPS before sending to browser
- Customer sees the login page in ~200ms

That's it. **One full round trip.**

---

## 🚨 Now the Most Important Part — Why IHS at All?

**Question:** Why not let customers hit WAS directly?

**Answer:** Because WAS will die.

### Banking Example: Salary Day Rush

Salary day. **50,000 customers** walk in.

**Without reception (IHS):**
- All 50,000 rush straight to 3 cashiers.
- Cashiers can't even count the crowd.
- No queue system. No order. collapses.

**With reception (IHS):**
- Reception absorbs the crowd (**buffering**)
- Token machine distributes evenly (**round-robin**)
- Guard stops troublemakers at the gate (**security**)
-, one customer at a time

### What IHS gives you (memorize these 4)

| # | Benefit | Simple Meaning |
|---|---|---|
| 1 | Buffer takes the hit of 50,000 requests; WAS gets controlled flow |
| 2 | Load Balancing | Round-robins equally |
| 3 | Security | WAS is never exposed to the internet. Firewall only opens 443 |
| 4 | Serving static content | Images/CSS handled by IHS — WAS does only real work |

---

## 💀 The Real Failure Story — Explained

A junior admin opened WAS port 9080 directly on the firewall *"for testing."*

**What happened:**

1. Port 9080 was now open to the whole internet ❌
2. Bots constantly scan the internet for open ports (they scan millions of ports every hour found WAS port 9080 **. Bot found the admin console behind it
5. Bot started brute force — trying thousands of username/password combos
6. Within ** compromised**

> 💡 **Banking analogy:** It's like leaving the vault door open because *"the reception was busy."* Doesn't matter how good reception is — if the vault is open, you're robbed.

### 🥇 Golden Rules (write on your desk Internet touches **IHS only** (port 44 ✅ WAS stays **behind the firewall** — internal network only
- ❌ **Never** open 9080/admin port) to the internet
- ❌ *" on production = career-ending move

---

## 🧾 One-Line Cheat Sheet

| Component | One line to remember |
|---| | Guard. Only 443 open. |
| IHS | Reception + bodyguard. Receives, decrypts, buffers, protects. |
| Plugin | Token machine. Reads plugin-cfg.xml, distributes requests. |
| plugin-cfg.xml | Plugin's address book (cluster, JVMs, ports). |
| Round-Robin | Turn by turn — JVM1 → JVM2 → JVM3 → repeat. |
| WAS JVM | Cashier. Runs Java, queries DB, builds the page. |
| Oracle DB | Vault. Holds all customer data. |
| Static content | Rate board — IHS answers itself. |
| Dynamic content (.jsp) | "My balance?" — must go to WAS. |

---

## ✅ Quick Self-Test (answer before next lesson)

1. Which port does HTTPS use?
2. Who decides which JVM gets the request — I
3. `login.jsp` is static or dynamic?
4. What file tells the Plugin about the JVMs?
5. Why should port 9080 never face the internet
<summary>👉 Click here for Answers</summary>

1. **443**
2. **PluginDynamic**
4. **plugin-cfg.xml**
5. **Bots will find it and brute-force the admin console**

</details>

---

## 🎯 One-Line Memory Trick

> *"The firewall is the guard, IHS is the reception, the Plugin is the token machine, WAS is the cashier, and the database is the vault."*
