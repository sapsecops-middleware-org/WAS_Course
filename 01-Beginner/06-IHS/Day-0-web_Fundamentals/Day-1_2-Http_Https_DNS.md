# 🟢 Lesson 1: What is HTTP? (The Language of the Web)

> 📚 **WAS Admin Day 1 — Networking Fundamentals**

---

## 🍽️ Think of a Restaurant

You walk into a hotel.

- You say: **"One masala dosa please"** → This is your **REQUEST**
- Waiter brings dosa → This is the **RESPONSE works exactly the same way:

- Browser asks: **"Give me this page"** → **REQUEST**
- Server replies: **"Here is the page"** → **RESPONSE**

> 💡 That's all HTTP is. A **question and answer conversation** between browser and server.

---

## 📖 What Does HTTP Stand For?

**HTTP = HyperText Transfer Protocol**

| Word | Meaning |
|---|---|
| HyperText | Web pages |
| Transfer | Sending |
| Protocol | Set of rules (language) |

---

## 📨 What a Request Looks Like

When you type a URL and press Enter, your browser sends this:

```http
GET /netbanking HTTP/1.1
Host: www.citibank.co.in
User-Agent: Mozilla/5.0 (Chrome)
Cookie: JSESSIONID=ABC123XYZ
```

### Word by Word:

| Part | Meaning in Plain English |
|---|---|
| `GET` | "I just want to READ a page" (not sending data) |
| `/netbanking` | "This is the exact page I want" |
| `HTTP/1.1` | "We're speaking version 1.1 of this language" |
| `Host` | "I want THIS website" (one server can host many sites) |
| `User-Agent` | "I'm using Chrome browser" |
| `Cookie` | "I visited before — here's my ID card, remember me?" |

---

### 🔑 Other Request Types You Must Know

| Method | Use | Example |
|---|---|---|
| `GET` | Just reading | View balance page |
| `POST` | Sending data | Login form, money transfer |
| `PUT` | Updating something | Update profile |
| `DELETE` | Deleting something | Delete record |

---

## 📬 What a Response Looks Like

```http
HTTP/1.1 200 OK
Content-Type: text/html
Set-Cookie: JSESSIONID=ABC123XYZ

<html>
  <h1>Welcome to NetBanking</h1>
</html>
```

| Part | Meaning |
|---|---|
| `200 OK` | "All good, here's your page" |
| `Content-Type` | "I'm sending you HTML" |
| `Set-Cookie` | "Here's your session ID — keep it" |
| HTML body | The actual page content |

---

## 📌 Status Codes — MEMORIZE THESE

You'll see these daily in logs and troubleshooting:

| Code | Meaning | Example |
|---|---|---|
| **200** | OK — success | Page loaded fine |
| **301/302** | Redirect | HTTP page → sent to HTTPS |
| **403** | Forbidden | "You're not allowed here" |
| **404** | Not Found | Wrong URL |
| **500** | Server Error | App crashed — WAS problem! |
| **503** | Service Unavailable | All WAS servers down — plugin issue! |

> 🏆 **Interview gold:** If customer gets **500/503**, the problem is usually **behind IHS (WAS side)**. If **404**, it's usually a **URL/routing** problem.

---

# 🟢 Lesson 2: What is HTTPS? (The Locked Envelope)

## ⚠️ The Problem

HTTP sends everything in **plain text**.

Imagine writing your bank password on a **postcard**. Everyone handling it can read it.

Now imagine you're on free WiFi in a coffee shop. Someone nearby with a simple tool (**packet sniffer**) can read account number.

> 🚨 **Dangerous. Especially for banking.**

## ✅ The Solution

**HTTPS = HTTP + Encryption**

Now your postcard goes inside a **locked envelope**. Even if someone steals it, they see only gibberish.

```text
HTTP  → plain text  → like shouting in public
HTTPS → encrypted   → like a sealed letter
```

> 🏦 In banking, HTTPS is **NOT optional**:
>
> - **RBI** guidelines require it
> - **PCI-DSS** standards require it
> - You will **never** see a real bank on plain HTTP

---

## 🔐 How the "Locked Envelope" Works (5 Simple Steps)

1. Browser: "I want to connect securely"
2. IHS: "Here's my **SSL certificate**" (ID proof issued by DigiCert/VeriSign)
3. Browser checks: "Yes, this certificate is genuine and trusted" ✅
 **secret encryption key**
5. Everything after this is **encrypted**

>'t worry about details yet. We'll deep-dive SSL on **Day 25–28**. Just remember: **HTTPS = HTTP inside a locked envelope.**

---

# 🟢 Lesson 3: What are Ports? (Counters in a Bank)

## 🏦 The Analogy

Your bank has **ONE address**:

> "Citibank, MG Road, Mumbai"

But inside, there are many **counters**:

- Counter 1: Home Loans
- Counter 2: Credit Cards
- Counter 43: NEFT/RTGS

> 💡 **Ports = counters inside a server.**
> One server has **ONE IP address**. But many services run on it, each at a different **port number**.

---

## 📌 THE PORT TABLE — MEMORIZE THIS

> ⚠️ This is **interview question #1**. Learn it cold.

| Port | What Runs Here ||---|---|
| **80** | HTTP | Customers → IHS (unsecured) |
| **443** | HTTPS | Customers → IHS (secured — all banking) |
| **9080** | WAS HTTP | IHS Plugin → WAS (internal) |
| **9443** | WAS HTTPS | IHS Plugin → WAS (internal, secured) |
| **9060** | WAS Admin Console | Admins only — **NEVER expose!** |
| **9043** | WAS Admin Console (secure) | Admins only — use this one |
| **8879** | SOAP Connector | DMGR communication |
| **2809** | ORB Bootstrap | Node Agent → DMGR |
| **1521** | Oracle DB | WAS → Database |

---

## 🧠 Memory Trick (say this out loud)

```text
"IHS talks to the WORLD on 80/443.
 IHS talks to WAS on 9080/9443.
 ADMINS talk to WAS on 9060/9043.
 Never expose 9060/9043 to the internet!"
```

---

## 🧱 Firewalls = Rule Books

A firewall is just a list of **ALLOW/DENY** rules:

```text
ALLOW: internet → IHS on 443     ✅ (customers browsing)
ALLOW: IHS → WAS on 9080         ✅ (plugin forwarding)
ALLOW: WAS → Oracle on 1521      ✅ (app reads DB)

DENY:  internet → WAS on 9043    ❌ (admin console — protected!)
DENY:  internet → Oracle on 1521 ❌ (DB must NEVER be reachable from internet)
```

> 💡 This is exactly how a bank's network looks. If a request is blocked, the **firewall log** tells you which rule denied it.

---

## 🟢 Lesson 4: What is DNS? (The Phone Book)

### 📞 The Problem

- Computers talk using numbers → `203.197.112.50`
- Humans remember names → `www.citibank.co.in`

> 💡 **DNS = Domain Name System = the phone book of the internet.**
> You look up "Ramesh" in a phone book → get his phone number.
> Browser looks up `www.citibank.co.in` in DNS → gets the IP address.

### 🔄 How It Works

```
Browser: "What is the IP of www.citibank.co.in?"
DNS:     "It's 203.197.112.50"
Browser: Connects to 203.197.112.50 on port 443
```

### 🌪️ Why YOU Care as a WAS Admin — DR Story

Real scenario: **Disaster Recovery switch.**

- Primary data centre: Mumbai
- Backup data centre: Hyderabad
- Mumbai has a fire/flood/power failure

What do you do?

1. **Change DNS** — point `www.citibank.co.in` from Mumbai IP → Hyderabad IP
2. Wait for **TTL** (Time To Live — how long DNS answers are cached;
   banks use ~5 minutes)
3. Traffic automatically flows to Hyderabad

✅ No customer notices anything. They still type the same website name.

Second reason DNS matters: inside the bank, `plugin-cfg.xml` uses
**hostnames** (like `ihs-server-01.citibank.internal`), not IPs.
Because IPs can change — hostnames stay stable.

---

## 🟢 Lesson 5: THE FULL JOURNEY (The Main Event)

> 🎬 Now let's trace ONE real request. Meet **Rames### 👨‍💻 RAMESH'S LAPTOP

- Types: `https://www.citibank.co.in/netbanking/balance`
- Presses Enter
- Browser prepares an HTTPS GET request

### 📗 STEP 1: DNS RESOLUTION

```
Browser → DNS: "IP for www.citibank.co.in?"
DNS → Browser: "203.197.112.50"
```

> 📝 Note: this IP belongs to the **F5 Load Balancer** — not directly to IHS.

### 📗 STEP 2: TCP CONNECTION (Three-Way Handshake)

Before talking, browser and server mustBrowser → F5: "SYN"      (Hello, I want to connect)
```
F5 → Browser: "SYN-ACK"  (Hello, I'm ready)
Browser → F5: "ACK"      (Great, let's talk)
```

> 🏆 Three messages = **Three-way handshake**. Remember this phrase —
> interviewers love it.

### 📗 STEP 3: PERIMETER FIREWALL

The outside firewall checks:

```
ALLOW: internet → F5 on port 443  ✅  (Pass through)
```

### 📗 STEP 4: F5 LOAD BALANCER

The F5 has two IHS servers registered:

```
IHS-1: 192.168.1.10:443  ← ACTIVE
IHS-2: 192.168.1.11:443  ← STANDBY
```

F5 picks one (round-robin or based on load) and forwards the request.

> 💡 Why load balancer? One IHS = single point of failure.
> Two IHS = if one dies, the other takes over.

### 📗 STEP 5: IHS — IBM HTTP SERVER (in the DMZ)

This is where **your world begins**. IHS does several things:

1. **TLS Handshake** — presents the SSL certificate, browser verifies it
2. **Decrypts** the encrypted request
3. Reads the URL: `/netbanking/balance`
4. Decision time: Is this a static file (image, CSS)? NO — it's dynamic
5. **WebSphere Plugin** (loaded inside IHS) takes over:
   - Reads `plugin-cfg.xml` → `/netbanking/*` belongs to `PaymentCluster`
   - Checks Ramesh's `JSESSIONID` cookie → clone ID suffix `:0001` → JVM1
   - Routes request to WAS-Node1 on port **9080**

### 🔑 Key Terms

| Term | Meaning |
|------|---------|
| **DMZ** | The "half-safe" zone between the internet and internal network |
| **plugin-cfg.xml** | The plugin's routing map (which URL goes to which WAS cluster) |
| **Session affinity** | Same user always goes to the same JVM (that's what the cookie clone ID does) |

### 📗 STEP 6: INTERNAL FIREWALL

```
ALLOW: IHS (192.168.1.10) → WAS on port 9080  ✅
```

### 📗 STEP 7: WAS — WEBSPHERE APPLICATION SERVER

WAS-Node1 (JVM1) receives the request on port 9080:

1. Looks up Ramesh's session (`JSESSIONID = ABC123`)
2. Finds it in JVM memory: `{userId: ramesh, role: customer}`
3. Runs the code: `BalanceServlet.doGet()`
4. Needs data → gets a connection from the **JDBC connection pool**
5. Sends SQL to Oracle

### 📗 STEP 8: INTERNAL FIREWALL (Again)

```
ALLOW: WAS (192.168.2.10) → Oracle on port 1521  ✅
```

### 📗 STEP 9: ORACLE DATABASE

```
Query:  SELECT balance FROM accounts WHERE acc_no = 'CITI0012345'
Answer: 125000.00
```

### 📗 STEP 10: RESPONSE GOES BACK

The answer travels back the **same path, reversed**:

```
Oracle → WAS → IHS → F5 → Firewall → Ramesh's Browser
```

> 💰 Ramesh sees: **"Your Current Balance: ₹1,25,000"**

```
Total time: 200–500 milliseconds.
Faster than an eye blink.
And this happens for EVERY click, EVERY customer, EVERY time.
```

---

## 🧠 ONE-PAGE REVISION CARD

> 📋 Copy this. Stick it near your desk.

### THE JOURNEY:

```
Browser → DNS → TCP Handshake → Perimeter Firewall
       → F5 Load Balancer → IHS (SSL decrypt + Plugin)
       → Internal Firewall → WAS (session + servlet)
       → Internal Firewall → Oracle DB
       → reversed
```

### ONE-LINE VERSION:

```
"DNS finds the address, F5 shares the load,
 IHS unlocks and routes, WAS thinks,
 Oracle remembers, and the reply runs back."
```

---

## 🔧 REAL TROUBLESHOOTING — "WHERE DID IT BREAK?"

> 💡 This is why we learn the journey. Each step has a **signature symptom**.

### Symptom → Suspect

| What the Customer Sees | Which Step Broke | First Thing to Check |
|---|---|---|
| "Server not found" / DNS error | Step 1 — **DNS** | `nslookup www.citibank.co.in` |
| "Connection timed out" | Step 2/3 — **TCP or Firewall** | Is port 443 open? Firewall rules changed? |
| Certificate warning in browser | Step 5 — **SSL on IHS** | Expired certificate? Check with browser or `openssl` |
| **503** Service Unavailable | **IHS → WAS routing** | WAS servers down? Plugin can't reach 9080? Check plugin log |
| **500** Internal Server Error | Step 7 — **WAS itself** | App crashed. Check WAS logs: `SystemOut.log`, trace |
| Page loads but data missing / SQL error | StepDB or DataSource** | Connection pool exhausted? DB down? Check DataSource |
| Very slow, not broken | **Any step** | Check thread dumps, connection pool usage, F5 stats |

### 🏆 The Golden Rule

> **Always find WHICH STEP failed before touching anything.**
>
> Random restarts = **junior admin**.
> Step-by-step isolation = **senior admin**.

### 🛠️ Practical Isolation Trick — Test Each Hop Separately

1. `ping` the IP → network OK?
2. `telnet ihs-host 443` → port open?
3. Hit IHS directly (skip F5) → IHS OK?
4. Hit WAS directly on 9080 (from internal machine) → WAS OK?
5. → DB OK?

> ✅ Wherever the test **fails** — that's your broken step.

---

## 📝 LOG FILES YOU'LL LIVE IN

| Log | Location (Typical) | Tells You |
|---|---|---|
| **plugin log** | IHS: `plugin_root/logs/http_plugin.log` | Plugin routing decisions, "unable to connect" errors |
| **access_log / error_log** | IHS: `logs/` | What requests came in, status codes |
| **SystemOut.log** | WAS: `profiles/AppSrv01/logs/server1/` | WAS startup, app errors, exceptions |
| **SystemErr.log** | Same folder | Stack traces, crashes |
| **messages.log** | WAS (newer versions) | Combined events |

### ⏱️ First 5 Minutes of Any Incident

1. Check **IHS plugin log**. Check **SystemOut.log** → is the app throwing errors?
3. Check the **status codes** → `503` = routing, `500` = app code

---

## 🏠 HOMEWORK (Do This Before Next Class)

- [ ] Open a browser → press **F12** → **Network tab**. Visit any website.
      Watch the status code `200`.
- [ ] On your laptop, run `nslookup www.google.com` in the command prompt.
      See DNS working live.
- [ ] Write the **9-step journey** from memory. Without looking.
      Five times if needed.
- [ ] Memorize the **port table**. Have someone quiz you.

---
# 🛠️ PART 6 — Seeing HTTP in Real Life (Simple Version)

Hi! I'm your senior trainer. Forget fear. **This part is EASY.**

Today you learn: How to actually **SEE** the requests you learned about.

**Two tools only:**

1. **curl** — for the terminal
2. **F12** — for the browser

Let's go step by step.

---

## 🥇 Tool 1: curl — Your Best Friend

### What is curl?

A command-line tool that acts like a browser.

- It **sends** a request.
- It **shows** you the response.
- With `-v` (verbose), it shows **EVERYTHING**.

### Try it now

```bash
curl -v https://www.google.com 2>&1 | head -50
```

### How to read the output

Three symbols. Memorize these:

| Symbol | Meaning |
|--------|---------|
| `*` | Connection info (DNS, TCP, SSL) |
| `>` | What **YOU** sent |
| `<` | What the **SERVER** sent back |

### Example walkthrough

```
* Trying 142.250.183.36:443...     ← DNS worked, trying to connect
* Connected to www.google.com      ← TCP connection OK
* SSL connection using TLS 1.3     ← HTTPS handshake done
* Server certificate: *.google.com ← Server showed / HTTP/2                     ← YOUR request going out
> Host: www.google.com

< HTTP/2 200                       ← Server says "OK"
< content-type: text/html
```

That's it. That's the whole story of a request in front of your eyes. 👀

---

## 🏦 Real Bank Troubleshooting Commands

Learn these **5 commands**. You'll use them daily.

```bash
# 1. Is IHS alive on port 443?
curl -v -k https://ihs-server-01:443/netbanking/health

# 2. Skip IHS. Test WAS directly.
curl -v http://was-node-01:9080/netbanking/health

# 3. Test a POST request (like a money transfer)
curl -v -X POST https://ihs-server-01/netbanking/transfer

# 4. Response headers ONLY (no page body)
curl -I https://www.citibank.co.in

# 5. Follow redirects (HTTP → HTTPS)
curl -L -v http://www.citibank.co.in
```

### Why these matter (real scenario)

Customer says: *"Netbanking is down."*

- You run command **#1** → Works ✅
- You run command **#2** → Fails ❌

**Conclusion:** IHS is fine. Problem is WAS (or the path between them).

You just found the broken layer in 2 commands. That's the power. 💪

### Two flags to remember

| Flag | Meaning |
|------|---------|
| `-k` | Ignore SSL certificate errors (OK for internal testing only) |
| `-L` | Follow redirects automatically |

> ⚠️ **Never use `-k` in production checks against real customer traffic. It hides certificate problems.**

---

## 🥈 Tool 2: Browser F12 (Developer Tools)

### How to open

1. Open Chrome or Firefox
2. Press **F12**
3. Click the **Network** tab
4. Visit any website

Now you see **EVERY request** your browser makes. Click one.

### What you'll see

**Headers tab:**

- Request URL, Method, Status Code
- Request headers (what browser sent — cookies, user-agent)
- Response headers (what server sent — Set-Cookie, content-type)

> 😄 **Fun fact:** You may see `X-Powered-By: WebSphere Application Server` — proof the site runs on WAS!

### Timing tab — THE GOLD 💰

```
DNS Lookup:      5 ms
TCP Connect:    15 ms
TLS Handshake:  45 ms
Request sent:    1 ms
Waiting (TTFB): 180 ms   ← MOST IMPORTANT
Download:       12 ms
Total:         258 ms
```

---

## ⭐ TTFB — The Metric You'll Live By

**TTFB = Time To First Byte**

It means: *"How long did the SERVER take to start answering?"*

- DNS, TCP, TLS are network stuff — usually fast.
- **TTFB = the time your application (WAS) took to think.**

### Simple rule

| TTFB | Meaning |
|------|---------|
| Under 1 second | Healthy ✅ |
| 3–5 seconds | WAS is struggling ⚠️ |
| More | Something is wrong ❌ |

### If TTFB is high, check:

- ❓ High CPU on WAS box?
- ❓ Low memory / garbage collection storms?
- ❓ Slow database queries?
- ❓ Thread pool exhausted?

---

## 📝 Quick Recap (Memorize This)

- ✅ `curl -v` = see HTTP in terminal
- ✅ `*` = connection, `>` = you, `<` = server
- ✅ `curl -k` = skip cert check (**test only**)
- ✅ Test IHS, then test WAS directly → find which layer is broken
- ✅ F12 → Network tab = see requests in browser
- ✅ TTFB = how long WAS took to answer
- ✅ High TTFB = WAS problem (CPU, memory, DB, threads)
---
# 🏦 PART 7 — Banking Scenario

> **Situation:** Monday morning 9 AM. Customers are complaining **NetBanking is slow.**
>
> You get a call. What do you do?

**Answer: Not panic. Systematic. Layer by layer.** 👇

---

## Step 1: Check from your laptop

```bash
curl -v https://netbanking.citibank.co.in/ 2>&1 | grep -E "< HTTP|time|connect"
```

**Ask yourself:**

- Is IHS even responding?
- Is it returning `200`? Or `503`?

---

## Step 2: Check if IHS itself is fine (bypass F5)

```bash
# Test IHS-1 directly (using its real IP, bypassing load balancer)
curl -v -k https://192.168.1.10/netbanking/

# Test IHS-2 directly
curl -v -k https://192.168.1.11/netbanking/
```

> 💡 **If IHS is fine but response is slow → problem is in WAS or DB**

---

## Step 3: Check if WAS is reachable (bypass IHS, go direct)

```bash
# From IHS server, test WAS directly
curl -v http://192.168.2.10:9080/netbanking/health
curl -v http://192.168.2.11:9080/netbanking/health
```

> 💡 **If WAS responds slow → check JVM heap, thread pool, DB connection pool**

---

## Step 4: Check IHS logs

```bash
tail -f /opt/IBM/HTTPServer/logs/access_log
tail -f /opt/IBM/HTTPServer/logs/error_log
```

---

## Step 5: Check WAS SystemOut.log

```bash
tail -f /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/server1/SystemOut.log
```

---

## 🧭 The Troubleshooting Flow (Visual)

```
Customer complains
        │
        ▼
 curl from laptop  ──fail──►  Check IHS / network / LB
        │ OK but slow
        ▼
 curl IHS directly (bypass F5) ──slow──►  Check IHS logs / server load
        │ OK but slow
        ▼
 curl WAS directly (bypass IHS) ──slow──►  JVM heap, thread pool, DB pool
        │ OK but slow
        ▼
 Problem is likely the DATABASE 🎯
```

---

## 📝 Quick Recap

- ✅ Never panic — follow the flow, layer by layer
- ✅ Test from **outside → in**: laptop → IHS → WAS → DB
- ✅ `bypass` the layer above to isolate the layer below
- ✅ Logs tell the truth: IHS `access_log`/`error_log`, WAS `SystemOut.log`
- ✅ Slow WAS = check **heap, threads, DB pool**
---

