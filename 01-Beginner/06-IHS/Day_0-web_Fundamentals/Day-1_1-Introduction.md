# 👨‍🏫 Web Server vs App Server vs Database

---

## 🍽️ First — A Restaurant Story (Remember This Forever)

Imagine a restaurant:

| Restaurant | Bank/IT World |
|---|---|
| Reception desk (takes your order, gives menu) | **Web Server (IHS)** |
| Kitchen (actually cooks the food) | **App Server (WAS)** |
| Store room (raw materials, ingredients) | **Database (Oracle)** |

- The receptionist **doesn't cook**.
- The kitchen **doesn't greet customers**.
- The store room **doesn't talk to anyone** — kitchen takes from it.

**That's the whole architecture. Everything else is detail.**

---

## 🟢 PART 1 — Web Server (IBM HTTP Server / IHS)

### What does it do?
- Sits at the **front door**.
- Listens on **port 443 (HTTPS)**.
- Talks to your **browser**.
- Serves **static files**: HTML pages, images, CSS, JavaScript.
- **Forwards** business requests to the App Server.

### What does it NOT do?
- ❌ No business logic (won't calculate your balance)
- ❌ No database connection
- ❌ No payment processing

### What is IHS really?
> **IHS = Apache + IBM patches + IBM support**

Same files (`httpd.conf`), same commands (`apachectl`). IBM tested it with WebSphere and gives support. That's it.

### Bank example:
You open `https://www.citibank.co.in/netbanking`.

- The **login page** (the look, the buttons, the images) → served by **IHS**.
- The actual **"check my balance"** work → NOT IHS. It forwards it.

### One-line memory hook:
> 🚪 **IHS = the doorman. He opens the door and passes your slip to the kitchen. He never cooks.**

---

## 🟢 PART 2 — Application Server (WebSphere / WAS)

### What does it do?
- **Runs your Java business logic** (the real brain).
- **Talks to the database** (read/write data).
- **Manages sessions** (remembers you're logged in between clicks).
- **Handles transactions** (money transfer either fully succeeds or fully fails — never half).

### Bank example:
You click **"Transfer ₹50,000"**.

- WAS runs the rules: Is balance enough? Is account active? Is there a limit?
- WAS tells the DB: *debit ₹50,000 from account A, credit to account B.*
- If anything fails mid-way → **everything rolls back**. Your money never vanishes.

### Ports to remember:
- **9080** (HTTP) / **9443** (HTTPS) — WAS listens here. Behind firewalls. Not on the internet.

### One-line memory hook:
> 🧠 **WAS = the brain. It thinks, calculates, and talks to the store room (DB).**

---

## 🟢 PART 3 — Database Server (Oracle / DB2)

### What does it do?
- **Stores everything permanently**: balances, account numbers, transaction history, passwords.
- **Runs SQL queries** sent by WAS.
- Guarantees **ACID** — your money never disappears, even if servers crash.

### Common DBs in banks:
- **Oracle** — most common (SBI, ICICI, HDFC) — port **1521**
- **DB2** — IBM's DB, common with WebSphere
- **SQL Server** — some private banks

### Simple example of what happens inside:

```sql
WAS asks:   "SELECT balance FROM accounts WHERE acc_no = 12345"
DB replies: ₹1,25,000
```

### One-line memory hook:
> 🗄️ **DB = the store room + ledger book. Nobody touches it directly. Only the kitchen (WAS) can.**

---

## 🏦 PART 4 — Full Flow: What Happens When You Check Balance

Follow the journey of ONE click:

```text
1. YOUR LAPTOP
   Type: https://www.citibank.co.in/netbanking
        ▼
2. PERIMETER FIREWALL
   Only port 443 allowed in. Everything else blocked.
        ▼
3. LOAD BALANCER (F5)
   You have 2 IHS servers (for backup).
   F5 picks one → IHS-1.
        ▼
4. IHS (in DMZ — the "outer zone")
   → Decrypts HTTPS (SSL termination)
   → Reads URL: /netbanking = business request!
   → Hands it to the WAS Plugin
        ▼
5. WAS PLUGIN (plugin-cfg.xml)
   A small helper inside IHS.
   Knows: "/netbanking → send to WAS cluster"
        ▼
6. INTERNAL FIREWALL
   Only ports 9080/9443 allowed.
        ▼
7. WAS CLUSTER
   → Checks JSESSIONID (are you logged in?)
   → Runs business logic
        ▼
8. INTERNAL FIREWALL #2
   Only DB port 1521 allowed.
        ▼
9. ORACLE DB
   → "Balance = ₹1,25,000"
        ▼
10. Response travels BACK up the same chain
        ▼
11. Browser shows: "Your Balance: ₹1,25,000" ✅
```

### Three small terms you must know:

| Term | Meaning |
|---|---|
| **DMZ** | The "semi-exposed" zone facing the internet. IHS lives here. |
| **SSL termination** | IHS removes the HTTPS lock so inner servers don't have to. |
| **JSESSIONID** | A cookie that says "this is user X, already logged in." |
| **Cluster** | Multiple WAS copies of the same app — if one dies, another works. |

---

## 🔑 WHY 3 Separate Servers? (Interview Gold ⭐)

Why not put everything on ONE machine?

| Reason | Without separation... |
|---|---|
| 🔒 **Security** | Hacker takes the web server → touches DB directly. Game over. |
| 📈 **Scalability** | Need more "balance checkers"? You'd have to add more web servers too. Wasteful. |
| 🔧 **Maintenance** | Restart app server to deploy new code → whole site goes down. |
| ⚡ **Performance** | App servers are bad at serving images. Web servers are bad at running Java. Each does its own job best. |
| 📋 **Compliance** | RBI / PCI-DSS **legally require** separated zones (DMZ / App / DB). |

### The security story in one line:
> Even if a hacker breaks into IHS (DMZ), there's **another firewall** before WAS, and **another** before the DB. Layered defense = onion 🧅.

---

## 📝 Final Summary Card (Memorize This)

| Layer | Product | Port | Job | Lives in |
|---|---|---|---|---|
| Web Server | **IHS** | 443 | Greet browser, serve static files, forward | DMZ |
| App Server | **WAS** | 9080/9443 | Business logic, sessions, transactions | App zone |
| Database | **Oracle** | 1521 | Store data forever, ACID | Secure zone |

### The 3 hooks:
1. 🚪 **IHS = doorman** (never cooks)
2. 🧠 **WAS = brain** (thinks and calculates)
3. 🗄️ **DB = ledger** (only WAS touches it)

---

## 🔜 Up Next (Choose One)

1. **How IHS + WAS Plugin actually connect** (plugin-cfg.xml explained simply)
2. **What a WAS "Cell, Node, Server" is** (the WAS topology — very important for interviews)
3. **What happens during a WAS deployment** (deploying an EAR step by step)
---
# 🏦 PART 5 — Banking Scenario: Why Separation Saves the Bank

> **Trainer:** Ox Alpha (Senior WAS Trainer)
> **Goal:** See how the 3-layer architecture works in a REAL bank, on a REAL busy Monday morning.

---

## ☀️ The Scene: Monday, 10:30 AM

Peak banking hours. Right now, at this exact moment:

- **50,000 customers** are using internet banking **simultaneously**:
  - Checking balances 💰
  - Paying credit card bills 💳
  - Doing NEFT transfers 🔄

### The bank's setup:

| Layer | Setup | Load |
|---|---|---|
| Web layer | **2 IHS servers** (IHS-1, IHS-2) behind an **F5 load balancer** | Shared |
| App layer | **4 WAS JVMs** in a cluster | ~12,500 users each |
| DB layer | **Oracle RAC** (2 DB nodes) | Redundant |

### Quick terms:

| Term | Meaning |
|---|---|
| **JVM** | Java Virtual Machine — one running copy of the app server |
| **F5 Load Balancer** | Traffic police — decides which server gets each request |
| **Oracle RAC** | 2 database nodes running as one — if one dies, the other takes over |

---

## 😬 The Problem: A Junior Admin Restarts IHS-1

A junior admin needs to apply an `httpd.conf` change. So he restarts **IHS-1**.

**Question: Does the bank crash? Do50,000 customers panic?**

**Answer: NO. Nothing happens. Here's why 👇**

---

## ✅ What Actually Happens (Step by Step)

```text
1. Junior admin restarts IHS-1
        ▼
2. IHS-1 goes DOWN
        ▼
3. F5 detects: "IHS-1 is not responding" (health check fails)
        ▼
4. F5 automatically sends 100% traffic → IHS-2
        ▼
5. IHS-2 forwards requests → WAS cluster (4 JVMs)
        ▼
6. 👥 0 customers notice ANYTHING. Banking continues.
        ▼
7. IHS-1 comes back UP with the new config
        ▼
8. F5 slowly sends traffic back to IHS-1
   (gradually, so IHS-1 is not hit with a sudden flood)
        ▼
9. Everything back to normal. Nobody knew. ✅
```

> 🎯 **Key point:** The restart was done on the WEB layer only.
> The **WAS cluster** and **Oracle DB** were never touched. They kept running.

---

## ❌ Now Imagine the OLD Way — Everything on ONE Machine

```
1 MACHINE = IHS + WAS + Oracle all together
        ▼
Admin restarts it for a config change
        ▼
💀 ENTIRE BANK GOES OFFLINE
        ▼
50,000 customers see errors
        ▼
News headline: "Citibank NetBanking Down!"
        ▼
Admin gets fired 😅
```

- Every customer session dies.
- Every running transaction dies.
- Every minute of downtime = lost money + lost trust + RBI questions.

---

## 🧠 The Big Lesson (Interview Gold ⭐)

> **Layered architecture = independent restarts.**
>
> You can restart/patch/upgrade ONE layer while the other layers keep running.
>
> - Restart IHS → WAS + DB keep working ✅
> - Restart one WAS JVM → other 3 JVMs handle the load ✅
> - One Oracle RAC node fails → second node takes over ✅
>
> This is called **High Availability (HA)**.

### One-line memory hook:
> 🧅 **Layers are like floors of a building under renovation — you renovate one floor while customers keep shopping on the others.**

---

## 📋 Bonus: Golden Rules for WAS Admins

1. ⚠️ **Always do restarts during low-traffic hours** (e.g., 2 AM), even with HA in place.
2. 🔄 **One node at a time** — never restart both IHS servers (or all 4 JVMs) together.
3. 🧪 **Test config changes on a test/UAT environment first** — never on production directly.
4. 👀 **Watch the F5 health checks** after restart — confirm the node rejoined the pool.
5. 📈 **Watch IHS-2's load** during the restart — it's handling double traffic.

---

## 📝 Summary Card

| Event | With 3-layer setup | With 1 big machine |
|---|---|---|
| Restart IHS for config change | ✅ Customers unaffected | 💀 Bank goes offline |
| One WAS JVM crashes | ✅ Other 3 JVMs take over | 💀 Bank goes offline |
| One DB node fails | ✅ Oracle RAC node 2 handles it | 💀 Bank goes offline |

### The 3 hooks:
1. 🚦 **F5 = traffic police** (detects a dead server, reroutes instantly)
2. 🛡️ **Cluster = backup team** (one player rests, game continues)
3. 🧅 **Layers = independence** (fix one floor, other floors keep working)

---
# 🎤 WAS Interview Q&A — Part 1: Architecture & Security

> **Trainer:** Ox Alpha (Senior WAS Trainer)
> **Format:** Real interview questions + model answers. Learn the *structure* of answering, not just the words.

