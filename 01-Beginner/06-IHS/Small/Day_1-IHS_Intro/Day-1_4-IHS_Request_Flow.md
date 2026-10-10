# PART 9 — Complete DigiBank Request Flow (Step by Step)

## 📌 Diagram 1 — Step-by-Step Flow (Main Diagram)
```text
 ┌─────────────────────┐
 │   💻 RAVI'S BROWSER │
 │  Types:             │
 │  digibank.com/      │
 │  internetbanking/   │
 │  login              │
 └─────────┬───────────┘
           │
           │  ① HTTPS Request
           │  Port 443 (Encrypted 🔒)
           ▼
 ┌─────────────────────┐
 │  ⚖️ LOAD BALANCER   │
 │  203.0.113.50       │
 │                     │
 │  • Checks health    │
 │  • Picks best IHS   │
 └─────────┬───────────┘
           │
           │  ② HTTPS
           │  Port 443
           ▼
 ┌─────────────────────────────────────┐
 │   🌐 IHS VM1  (192.168.1.10)        │
 │   ┌───────────────────────────────┐ │
 │   │ ③ TLS HANDSHAKE               │ │
 │   │   • Certificate shown 📜      │ │
 │   │   • Browser verifies ✅ │ │
 │   │   • Encrypted tunnel 🔒       │ │
 │   │   • Request DECRYPTED         │ │
 │   └───────────────────────────────┘ │
 │   ┌───────────────────────────────┐ │
 │   │ ④ DECISION MAKING             │ │
 │   │   • VirtualHost match?  ✅    │ │
 │   │   • Static file?        ❌    │ │
 │   │   • Plugin handles it?  ✅    │ │
 │   └───────────────────────────────┘ │
 │   ┌───────────────────────────────┐ │
 │   │ ⑤ WEBSPHERE PLUGIN            │ │
 │   │   Reads plugin-cfg.xml 📄     │ │
 │   └───────────────────────────────┘ │
 │   ┌───────────────────────────────┐ │
 │   │ ⑥ LOAD BALANCING (Round Robin)│ │
 │   │   Server01 ← You are here 🎯  │ │
 │   └───────────────────────────────┘ │
 └─────────┬───────────────────────────┘
           │
           │  ⑦ HTTP (Plain — Internal Network)
           │  Port 9080
           ▼
 ┌─────────────────────────────────────────────────┐
 │         WAS CLUSTER : DigiBankCluster           │
 │   ┌──────────────────┐    ┌──────────────────┐  │
 │   │ 🟢 SERVER01      │    │ ⚪ SERVER02      │  │
 │   │ 192.168.1.20     │    │ 192.168.1.30     │  │
 │   │  Port 9080       │    │  Port 9080       │  │
 │   │  (Node01)        │    │  (Node02)        │  │
 │   └────────┬─────────┘    └──────────────────┘  │
 │            │                                    │
 │            ▼  ⑧                                 │
 │   ┌──────────────────────────────┐              │
 │   │  internetbanking.war         │              │
 │   │  ┌────────────────────────┐  │              │
 │   │  │ ☕ DigiBank Java Code  │  │              │
 │   │  │ • Check credentials    │  │              │
 │   │  │ • Build login page     │  │              │
 │   │  └──────────┬─────────────┘  │              │
 │   └─────────────┼────────────────┘              │
 └─────────────────┼───────────────────────────────┘
                   │
                   ▼
        ┌────────────────────┐
        │  🗄️ DigiBank DB    │
        │  • Verify user     │
        │  • Fetch data      │
        └────────────────────┘
```

---

## 📌 Diagram 2 — Response Journey (Return Trip)

```text
 ┌──────────┐
 │ 🗄️ DATABASE      │
 └────────┬─────────┘
          │  Data
          ▼
 ┌──────────────────┐
 │ ☕ WAS SERVER01  │  Builds HTML
 └────────┬─────────┘
          │  HTML Response (HTTP 9080)
          ▼
 ┌──────────────────┐
 │ 🔌 PLUGIN        │  Passes back
 └────────┬─────────┘
          │
          ▼
 ┌──────────────────┐
 │ 🌐 IHS           │  RE-ENCRYPTS 🔒
 └────────┬─────────┘  (TLS)
          │  HTTPS 443
          ▼
 ┌──────────────────┐
 │ ⚖️ LOAD BALANCER │
 └────────┬─────────┘
          │  HTTPS 443
          ▼
 ┌──────────────────┐
 │ 💻 RAVI'S BROWSER│  🎉 Login page shown!
 └──────────────────┘



---

## 🏦 The Real-Life Analogy First

Think of a bank branch:

| Banking World | Tech World |
|---|---|
| Customer walks in | Browser sends request |
| Receptionist (security desk) | Load Balancer |
| Clerk at the counter | IHS (Web Server) |
| Clerk's assistant who knows which officer to send you to | WebSphere Plugin |
| The actual bank officer | WAS (WebSphere Application Server) |
| Bank's records room | Database |

---

## Step 1 — Ravi Types the URL

Ravi types:

```text
https://www.digibank.com/internetbanking/login
```

**What happens first? DNS lookup.**

- Computers don't understand names. They understand numbers (IP addresses).
- DNS = the "phone book" of the internet.
- DNS says: `www.digibank.com = 203.0.113.50`

**Point to remember:**
- That IP belongs to the **Load Balancer**, not the actual application server.
- The customer never knows what's behind it. That's by design — security.

Browser then sends an **HTTPS request on port 443**.

- HTTPS = HTTP + encryption (padlock 🔒)
- Port 443 = the standard door for HTTPS
- Plain HTTP uses port 80 (never used in banking)

---

## Step 2 — Load Balancer (The Traffic Police)

**Job:** Decide which web server gets this request.

It checks:

- Which IHS servers are **alive and healthy**?
- Is VM1 busy? Is VM2 free?
- Sends the request to the healthiest one.

Say it picks:

```text
IHS VM1 → 192.168.1.10:443
```

**Real-life example:**
> Like a bank with 5 counters. The receptionist sends you to the counter with the shortest queue.

**Why do we need this?**

- If VM1 crashes, VM2 takes over. Customer never notices.
- No single point of failure.

---

## Step 3 — IHS Receives the Request (TLS Handshake)

IHS = IBM HTTP Server. It's a **web server**. It listens on **port 443**.

Before any data moves, a **TLS Handshake** happens:

1. Browser: "Hello, prove who you are."
2. IHS shows its **SSL certificate** (issued for `www.digibank.com`).
3. Browser checks the certificate:
   - Is it genuine?
   - Is it for the right website?
   - Is it expired?
4. Both agree on an encryption key.
5. Now everything travels **encrypted**.

**Real-life example:**
> Like showing your ID card at the bank door. The guard verifies it before letting you in. After that, you talk privately.

**Key point:** IHS **decrypts** the request. Now it can read:

```http
GET /internetbanking/login HTTP/1.1
Host: www.digibank.com
```

---

## Step 4 — IHS Decides What To Do

IHS asks 3 questions:

**Q1: Which website is this?**
- Reads the `Host` header: `www.digibank.com`
- Matches it to a **VirtualHost** block in `httpd.conf`.
- One IHS can host many websites. VirtualHost = which website each request belongs to.

**Q2: Is this a static file?**
- Static = images, CSS, JS files. IHS serves these **itself**. Fast.
- This is a login page request → Not static.

**Q3: Does the plugin handle this URL?**
- Checks `plugin-cfg.xml`.
- `/internetbanking/*` is mapped → YES.
- Hand it to the WebSphere Plugin.

**Real-life example:**
> The clerk thinks: "Is this a simple enquiry I can answer myself? No. This needs a bank officer. Let me pass it on."

---

## Step 5 — WebSphere Plugin Reads plugin-cfg.xml

The plugin is a small module **inside IHS**. It is the **bridge** between IHS and WAS.

`plugin-cfg.xml` tells it:

```xml
URL /internetbanking/*  →  DigiBankCluster

DigiBankCluster contains:
   Server01 → 192.168.1.20:9080
   Server02 → 192.168.1.30:9080
```

It also knows:

- Load balancing rules
- Timeouts
- Retry rules

**Memory trick:**

| File | Role |
|---|---|
| `httpd.conf` | IHS's own brain (web server config) |
| `plugin-cfg.xml` | Plugin's brain (routes to WAS) |

> ⚠️ `plugin-cfg.xml` is generated **from WAS** (via `genplugin`) — never edit it by hand.

---

## Step 6 — Plugin Picks a WAS Server

Default rule: **Round Robin**.

- Request 1 → Server01
- Request 2 → Server02
- Request 3 → Server01 ... and so on.

So Ravi's request goes to **Server01 (192.168.1.20:9080)**.

**Note the change:**

| Path | Protocol | Port |
|---|---|---|
| Browser → IHS | HTTPS (encrypted) | 443 |
| Plugin → WAS | HTTP (plain, internal network) | 9080 |

This internal path is safe because it never touches the outside world.

---

## Step 7 — WAS Does the Real Work

Server01 receives the request.

- URL `/internetbanking/*` belongs to **internetbanking.war**.
- WAR = the packaged Java application (like a sealed envelope of code).

The Java code now:

1. Checks Ravi's login credentials
2. Connects to the DigiBank database
3. Verifies username/password
4. Builds the login page HTML

**Real-life example:**
> The bank officer checks your ID, pulls your account file from the records room, and prepares your statement.

---

## Step 8 — Response Travels Back (Reverse Order)

```text
WAS Server01
   → Plugin
   → IHS (re-encrypts with TLS)
   → Load Balancer
   → Ravi's Browser
```

Ravi sees the login page. Total time: usually under a second.

**Note:** IHS **re-encrypts** the response before sending it back. The customer's data is always encrypted outside the data center.

---

## 🔁 What If Server01 Fails?

The plugin is smart:

- It detects Server01 is **not responding** (timeout).
- It marks Server01 as **unhealthy**.
- It **retries** the same request on **Server02**.
- Ravi sees no error. He doesn't even know.

This is called **failover** — the biggest reason banks use clusters.

---