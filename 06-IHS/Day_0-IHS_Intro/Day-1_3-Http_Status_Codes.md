# HTTP Status Codes
200, 301, 302, 403, 404, 500, 502, 503
Which Server Produces Each — and What To Do at 2 AM

## 🧠 Why Status Codes Matter More Than You Think
Every HTTP response starts with a 3-digit number. That number is the server telling you — in a single glance — what happened.

As a WebSphere Admin, when your phone rings at 2 AM and someone says "the bank website is down" — the FIRST thing you do is check the status code. That code tells you which layer broke — IHS, Plugin, WAS, or DB. It cuts your troubleshooting time from 2 hours to 10 minutes.

Think of status codes like a traffic signal at each layer of your stack:
```
Browser ──► IHS ──► Plugin ──► WAS ──► DB

If signal is RED at IHS   → 4xx or 5xx from IHS
If signal is RED at WAS   → 502 or 503 from Plugin
If signal is RED at DB    → 500 from WAS (DB error inside Java)
```

## 📊 The 5 Families

| Family | Range | Meaning | Simple English |
|--------|-------|---------
| **1xx** | 100–199 | Informational | "Still working on it, hold on" |
| **2xx** | 200–299 | ✅ Success | "Everything worked perfectly" |
| **3xx** | 300–399 | ↩️ Redirect | "Go look somewhere else" |
| **4xx** | 400–499 | ❌ Client Error | "**YOU** made a mistake in the request" |
| **5xx** | 500–599 | 💥 Server Error | "**WE** (the server) have a problem" |
---
# ✅ PART 2 — 200 OK (The Happy Code)

> Teaching 200 OK the way we teach juniors in the bank — slowly, simply, no jargon without explanation.

---

## 🎯 What is 200 OK?

**Simple meaning:**

- You asked for something.
- Server gave it.
- Everything worked.

### 🏠 Real life example

> You ask the waiter: *"One coffee please."*
> Waiter brings coffee.
> **That's 200 OK.** Order complete. Nobody is angry.

---

## 🌐 Browser Example

```text
Browser: "Give me /netbanking/balance"
Server:  "200 OK — here is your balance: ₹1,25,000"
```

- **Request** = browser asking.
- **Response** = 200 + the data.
- Done Happy customer. ✅

---

## 📄 What 200 Looks Like in curl

```bash
curl -I https://www.citibank.co.in/netbanking/
```

**Output:**

```text
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8
Content-Length: 45231
```

**Read it line by line:**

| Line | Meaning |
|------|---------|
| `HTTP/1.1 200 OK` | Status = success |
| `Content-Type: text/html` | "I'm sending you an HTML page" |
| `Content-Length: 45231` | Page size = 45 KB (approx) |

> 💡 **Memory trick:** 200 OK = 2 happy hands (request accepted, response delivered).

---

## 🏦 Who Sends 200 in a Bank Setup?

Remember our setup:

```text
Browser → IHS (web server) WAS (Java app) → Database
```

### Case 1 — Static files

- HTML, CSS, JS, images, logos.
- IHS keeps these ready on disk.
- IHS sends **200 directly**. WAS never touched.
- Fast. Like taking a pre-packed snack off the shelf.

### Case 2 — Dynamic pages

- Balance check, statement, transaction- IHS forwards to WAS → WAS runs Java code → maybe asks → WAS sends 200 back through IHS.
- Slower, but that's where the **real work** happens.

```text
Static:   Browser → IHS → 200                          (fast lane)
Dynamic:  Browser → IHS → WAS → DB → WAS → IHS → 200   (work lane)
```

---

## 📊 Why 200 Rate Matters (Monitoring)

In banks, we watch dashboards all day.📉📈

### Healthy state

- **200 rate ≈ 95–99%**
- Means customers can use netbanking.

### 🚨 Alarm story

```text
200 rate:  98% → 85% → 60%  → 🚨 ALERT FIRES
```

**What could be wrong?**

1. **WAS JVM crashed or hung** → app not responding.
2. **DB connection pool exhausted** → app can't get DB connections → errors instead of 200.
3. **Bad deployment** → new code broke the app.

> **Note:** When 200 drops, those requests don't disappear — they become **5xx errors** (we'll cover that later).
> Your job as WAS admin: **check WAS logs, JVM health, DB pool first.**

---

## 🧠 Quick Recap (Memorize This)

| # | Point |
|---|-------|
| 1 | 200 = success. Request done, data delivered. |
| 2 | IHS sends 200 for **static** files, directly. |
| 3 | WAS sends 200 (via IHS) for **dynamic** Java app work. |
|4 | Check with: `curl -I <url>` — first line tells you everything. |
| 5 | 200 rate = health metric. Drop = fire alarm. 🚨 |
| 6 | 200 drop usually means: JVM, DB pool, or bad deployment. |

---

## ✏️ Mini Exercise (Do This Now)

Open terminal, run:

```bash
curl -I https://www.google.com
```

- Look at the first line. Should be `200 OK`. ✅
- Try your bank's site too.

> You just did your **first health check**.
> That's literally what we do on calls at 2 AM. 😄
---
# ✅ PART 3 — Redirect Codes (301 & 302)

> Forget jargon. Let's build this from zero.

---

## 1️⃣ What is a Redirect?

**Simple idea:**

- You ask for one page.
- Server says: *"It's not here. Go to that other place."*
- Your browser **automatically** goes there. You don't click anything.

### 🏠 Real-life example

> You walk into a bank branch.
> Guard says: *"Loan counter has moved to the 2nd floor."*
> You don't argue. You just go upstairs.
> **That guard = redirect.**

### Why redirects exist in banking

1. Force users from unsafe **HTTP** to safe **HTTPS**.
2. Send expired sessions back to the **login page**.
3. Move pages to new URLs **without breaking old links**.

---

## 2️⃣ 301 — Permanent Redirect

**One line to remember:**

> 301 = *"I have moved PERMANENTLY. Forget the old address."*

### Example

```text
Browser: "Give me http://www.citibank.co.in"     (no S = not secure)
IHS:     "301 → go to https://www.citibank.co.in"
Browser: Automatically opens the HTTPS site.
```

### 🔑 The key fact (interviewers love this)

- The **browser remembers** the 301.
- Next time, it doesn't even try HTTP. It goes **straight to HTTPS**.
- This memory is called **caching**.
- **Google also remembers it** — so search rankings move to the new URL too.

### ⚙️ Where do we configure it?

In **IHS** (the web server), inside `httpd.conf`:

```apache
<VirtualHost *:80>
    ServerName www.citibank.co.in
    Redirect 301 / https://www.citibank.co.in/
</VirtualHost>
```

**Plain English of this config:**

> "Anyone coming on port 80 (HTTP), permanently send them to HTTPS."

### 🏦 Golden rule

> In every bank, **HTTP must redirect to HTTPS. No exceptions.** It's a security mandate.

---

## 3️⃣ 302 — Temporary Redirect

**One line to remember:**

> 302 = *"I have moved TEMPORARILY. Don't remember this. Ask me again next time."*

### Example

```text
Browser: "Give me /netbanking/dashboard"
WAS:     "302 → go to /netbanking/login"    (you're not logged in!)
Browser: Automatically opens the login page.
```

### Why "temporary"?

- Right now you're **not logged in** → go to login.
- But after you log in → you **CAN** see the dashboard.
- So the dashboard "moved" only **for now**. It's not gone forever.

### Where you see 302 in real banking

| Situation | What happens |
|-----------|--------------|
| Session expired | WAS sends 302 → login page |
| Login successful | WAS sends 302 → dashboard |
| Site migration | Old URL `/ibank` → `/netbanking` (302, for now) |

### 👷 Who does this?

- Usually **WAS** (the application decides based on login state).
- **Not IHS.** IHS doesn't know if you're logged in.

---

## 4️⃣ 301 vs 302 — The Interview Table

> 📌 **This table is your interview answer. Memorize it.**

| Point | 301 | 302 |
|-------|-----|-----|
| **Meaning** | Permanent move | Temporary move |
| **Browser caches it?** | YES — forever | NO — asks every time |
| **Typical use** | HTTP → HTTPS, domain change | Login redirect, maintenance page |
| **Who sends it** | Usually IHS (config file) | Usually WAS (application logic) |

---

## 5️⃣ How to Remember Forever — The Analogy

### 🏠 301 = House shift

- You moved to a new house **permanently**.
- Tell everyone the new address. Update your address book. **Never** visit the old one.

### 🏖️ 302 = Vacation

- You're away for a week.
- **Don't** update your address book. When you're back, the old address works again.

---

## 6️⃣ Quick Self-Test (Answer in your head)

| Question | Answer |
|----------|--------|
| User types HTTP URL. What code forces them to HTTPS? | **301** |
| Session expired, user hits dashboard. What code sends them to login? | **302** |
| Which one does the browser remember forever? | **301** |
| Who usually sends 301 — IHS or WAS? | **IHS** |
| Who usually sends 302 — IHS or WAS? | **WAS** |

---

## 7️⃣ One-Line Summary

> **301 says "forever." 302 says "for now."**
> **301 lives in IHS config. 302 lives in WAS logic.**
---
# ✅ PART 4 — 403 Forbidden (Simple Guide)

> Explaining to a fresh graduate on day one. ☕ Slow, step by step.

---

## 1️⃣ What is 403?

**403 = Access Denied**

- The server **found** your page. ✅
- The server **understood** your request. ✅
- But it says: *"NO. You cannot have it."* ❌

### Remember the difference

| Code | Meaning | Example |
|------|---------|---------|
| **404** | Page does NOT exist | Wrong URL |
| **403** | Page EXISTS but you can't have it | No permission |

### 🏦 Real-life example

> You go to a bank.
> The bank building **exists**. ✅
> The vault **exists**. ✅
> The security guard stops you: *"Staff only."* ❌
>
> **That is 403.**

---

## 2️⃣ The 3+1 Reasons for 403 (Simple)

1. Server cannot **READ** the file (file permissions)
2. Server is told to **BLOCK** you (IP restriction)
3. Server is told to **HIDE** things (folder listing off)
4. Server is told to **DEFEND** itself (WAF block)

Let's see each one. 👇

---

## 3️⃣ Cause 1 — File Permission Wrong

### The story

- IHS (the web server) runs as a user like `wasadmin` or `nobody`.
- If a file is **locked to its owner only**, IHS cannot open it.
- IHS can't read it → gives you **403**.

### 🔍 Check it

```bash
ls -la /opt/IBM/HTTPServer/htdocs/netbanking/index.html
```

### How to read permissions (learn this once, use forever)

`rwxr-xr-x` means:

| Position | Who | What |
|----------|-----|------|
| `rwx` | **Owner** | read, write, execute |
| `r-x` | **Group** | read, execute |
| `r-x` | **Others** | read, execute |

> IHS is "others" — it needs that **last `r-x`**.

| Permission | Result |
|------------|--------|
| `-rwx------` (700) | ❌ 403 — IHS can't read |
| `-rwxr-xr-x` (755) | ✅ 200 — works |

### 🔧 Fix

```bash
chmod 755 /opt/IBM/HTTPServer/htdocs/netbanking/index.html
```

> 💡 **Memory trick:**
> `755` = everyone can read = happy web server
> `700` = only owner = 403

---

## 4️⃣ Cause 2 — IP Restriction in httpd.conf

### The story

- Admin writes a rule: *"Only office IPs can see this page."*
- You come from outside that range → blocked → **403**.

### Example config

```apache
<Directory /opt/IBM/HTTPServer/htdocs/admin>
    Require ip 192.168.1.0/24
</Directory>
```

**Plain English:**

- `192.168.1.0/24` = all IPs from `192.168.1.1` to `192.168.1.254`
- Your IP is `10.0.0.5`? → **403. You're not on the list.**

### 🎉 Real life

> It's like a **guest list at a wedding**. Not on it?
> The guard (IHS) stops you at the door.

---

## 5️⃣ Cause 3 — Directory Listing Disabled (This is GOOD ✅)

### The story

- You type: `www.bank.com/images/`
- You hope to see a **list of all files** in that folder.
- But the admin turned that **OFF**:

```apache
Options -Indexes
```

- Result: **403**.

### Why this is correct

- If folder listing was ON, **hackers could see ALL your files**.
- Like leaving the vault door open with a file list stuck on it. 😱
- Banks **ALWAYS** turn this off.

> 💡 **Memory trick:** `-Indexes` = "Don't show folder contents" = security win ✅

---

## 6️⃣ Cause 4 — WAF / mod_security Blocking

### The story

- Banks run a **WAF** (Web Application Firewall).
- Think of it as a **metal detector at the airport**. 🛂
- Your request looks suspicious (SQL injection attempt, weird characters, too many login tries) → WAF blocks it → **403**.
- The request **never reaches WAS**. Blocked at the door.

### Why banks love this

- Stops hackers **before** they touch the application.
- One of the reasons the bank hasn't been in the news. 😄

---

## 7️⃣ Real Scenario — The Auditor Story

**Situation:**

> Auditor calls: *"I get 403 when I open the WAS Admin Console."*

**Your thinking (like a 25-year veteran):** 🧠

1. 403 = **exists but blocked**.
2. Admin console = **sensitive** → probably IP-restricted.
3. Auditor's laptop = `192.168.5.x` → not in the allowed list.
4. Allowed list only has admin subnets.

**Fix (temporary):**

```apache
Require ip 192.168.5.0/24
```

### ⚠️ Important — Professional habits

- ✅ Add the rule **ONLY for the audit period**.
- ✅ **Remove it** after the audit.
- ✅ **Document the change** (who, why, when, until when).

**Why?** PCI-DSS (bank security rules) demands documentation of every access change.

> 🏦 **Golden rule:**
> Temporary access = documented + removed on time.
> Undocumented open access = auditor's nightmare = **your problem**.

---

## 8️⃣ Quick Troubleshooting Steps for 403

Follow this order:

1. ☑️ **Check the file exists** → confirms it's not 404.
2. ☑️ **Check file permissions** → `ls -la`, need `755`.
3. ☑️ **Check `httpd.conf`** → any `Require ip` or `Deny` rules?
4. ☑️ **Check folder listing** → `Options -Indexes`?
5. ☑️ **Check WAF logs** → `mod_security` blocked it?
6. ☑️ **Check `error_log`** → it tells you WHY:

```bash
tail -f /opt/IBM/HTTPServer/logs/error_log
```

> 💡 **Memory trick — "P.I.D.W.L":**
>
> | Letter | Stands for |
> |--------|-----------|
> | **P** | Permissions |
> | **I** | IP rules |
> | **D** | Directory listing |
> | **W** | WAF |
> | **L** | Logs |

---

## 9️⃣ One-Line Summary

> **403 = "The page is there, but the bouncer won't let you in."**
>
> Fix = permissions (`755`), IP rules, or WAF —
> and **always check the logs first**. 🕵️
---
# ✅ PART 5 — 404 Not Found (Learned Like a Fresher)

> Sit down, take a chai. ☕ Let's learn 404 the way I learned it 25 years ago — by fixing it at 2 AM during a payment outage. 😅

---

## 1️⃣ What is 404? (One line)

**404 = "Server is alive, but the page you asked for does NOT exist."**

> 🏦 The bank branch is **OPEN** (server running).
> Staff is there (WAS is fine).
> But you asked for a *"Purple Account"* — it doesn't exist.
>
> **That's 404.**

---

## 2️⃣ Golden Rule to Remember

| Code | Meaning |
|------|---------|
| **404** | Page not found — wrong address or missing app |
| **500** | App broke while running |
| **503** | Nobody is there to serve you |

> 👉 **404 = address problem. NOT a crash.**

---

## 3️⃣ The Journey of a 404 (Simple Picture)

```text
Browser → IHS → WAS
```

**404 can happen at 2 places:**

1. **IHS says 404** → IHS looked for a file. File not there. *(IHS never even asked WAS)*
2. **WAS says 404** → IHS forwarded the request to WAS. WAS says *"I have no app on this URL."*

> 🔑 **First job: find WHO said 404. That saves 80% of your time.**

---

## 4️⃣ The 4 Real Causes (Banking Examples)

### Cause 1 — Customer typed wrong URL (typo)

```text
Customer types: citibank.co.in/netbankig/     (missing 'n')
IHS looks for /netbankig/ → not found → 404
```

**Real life:** You went to the bank and asked for *"Savngs Account"* — you spelt it wrong.

> ✅ **Fix:** Nothing on server. Tell customer / check bookmarks.

---

 not deployed in WAS

```text
IHS is configured to send /netbanking/upi/ to WAS.
But the UPI app never got deployed (deployment failed silently).
WAS has no app → returns 404.
```

> ✅ **Fix:** WAS Admin Console → **Applications → WebSphere enterprise applications** → check the app is there and **Started**.

---

### Cause 3 — Wrong DocumentRoot in IHS (static files)

```apache
httpd.conf says:   DocumentRoot /opt/IBM/HTTPServer/htdocs
File actually at:  /opt/IBM/HTTPServer/html/netbanking/index.html
```

- IHS searches in `htdocs`. File is in `html`. Not found → **404**.

> ✅ **Fix:** Point `DocumentRoot` to the right folder, or move the file.

---

### Cause 4 — Case mismatch (Linux is strict! ⚠️)

```text
IHS routes  /netbanking/*  to WAS.
WAS app is deployed as  /NetBanking/*  (capital N).
Linux treats these as two DIFFERENT things. → 404.
```

> ✅ **Fix:** Make URL spelling match exactly — same case, same spelling.

---

## 5️⃣ How to Diagnose (Do this in this order)

### 🪜 Step 1: Find who gave the 404

```bash
# Look in IHS error log
grep "404" /opt/IBM/HTTPServer/logs/error_log | tail -20
```

- Log says **"File does not exist"** → IHS problem *(Cause 1 or 3)*.
- **No entry**, but user still sees 404 → likely WAS problem *(Cause 2 or 4)*.

### 🪜 Step 2: Check the file exists on IHS

```bash
ls -la /opt/IBM/HTTPServer/htdocs/netbanking/
```

- File missing? → Wrong folder or wrong `DocumentRoot`.
- 💡 **Tip:** Check spelling **AND** case. `NetBanking ≠ netbanking`.

### 🪜 Step 3: Check the app in WAS

- WAS Admin Console → **Applications → Enterprise Applications**.
- Is the app listed? Is it **Started**?

### 🪜 Step 4: Test the URL directly (bypass IHS)

```text
http://WAS-host:9080/netbanking/
```

| Result | Meaning |
|--------|---------|
| ✅ Works | IHS config issue |
| ❌ Fails | WAS app issue |

> 👉 **This one test splits the problem in half. Remember it.**

---

## 6️⃣ Real Banking Story (So You Never Forget) 🌙

> New **payments module launch day**. Customers get 404 on `/netbanking/upi/`.
>
> 1. **IHS logs:** clean. IHS says *"I forwarded it to WAS, not my fault."*
> 2. **WAS Admin Console:** UPI app status = **Stopped** *(deployment half-finished, nobody noticed)*.
> 3. **Restart the app** → 404 gone. ✅
>
> **Lesson:** 404 doesn't always mean "page missing." Sometimes it means **"app not running."**

---

## 7️⃣ Cheat Sheet (Stick on Your Desk) 📌

| Symptom | Likely Cause | Check |
|---------|--------------|-------|
| Typo in URL | Customer error | Requested URL in logs |
| File missing on disk | `DocumentRoot` wrong | `ls` the folder |
| App not deployed / stopped | WAS issue | Admin Console |
| Works in one case, not another | Linux case-sensitivity | Compare exact spelling |
| IHS test fails, WAS test works | IHS/plugin config | `plugin-cfg.xml` UriGroup |

---

## 8️⃣ One-Line Summary

> **404 = "The road exists, but the house doesn't."**
> Check the **address at IHS first**, then the **app in WAS**.

---
# ✅ PART 6 — 500 Internal Server Error (Explained Like You're New)

> Grab a coffee. ☕ Let's go slow.

---

## 1️⃣ What is a 500 Error? (One line)

**The request reached the server, the server TRIED to process it, and it CRASHED halfway.**

That's it. That's the whole idea.

---

## 2️⃣ The Bank Teller Story (Remember this) 🏦

> You go to the bank counter.
>
> - **404** = The counter doesn't exist. You walked to the wrong floor.
> - **503** = The counter exists, but the teller says *"Come back later, too busy."*
> - **500** = The teller exists, takes your form, starts working... **and collapses.** 💥
>
> The teller didn't refuse. The teller **tried**. Something broke **inside** the teller.

**In our world:**

| Story | Real World |
|-------|-----------|
| Teller | Your Java application running inside WAS |
| Collapse | Unhandled exception (error the code didn't know how to handle) |

---

## 3️⃣ Where does the 500 come from?

**The flow:**

```text
Browser → Web Server → WebSphere (WAS) → Java Application → CRASH 💥
                                              ↑
                                       500 born here
```

> 🔑 **Key point:** 500 is an **application-level error** — NOT a web server error, NOT a network Web server is fine ✅
- ✅
- WAS is running ✅
- The Java code inside blew up ❌

---

## 4️⃣ The 3 Most Common Causes (Memorize these) 📌

### Cause 1 — Java Exception (most common)

The code is like:

```java
balance = customer.getAccountBalance();
```

But `customer` is **empty (null)**. There is no customer object!

Java screams: **NullPointerException** →Real-life example:** The teller picks up your form, looks for your account file — the file isn't there. Teller panics. Collapse.

---

### Cause 2 — Database Connection Failed

How WAS talks to the database:

1. WAS keeps a **pool** of DB connections (say 50)
2. Each request **borrows** one connection
3. Uses it → returns it → next request uses it

**What goes wrong:**

```text
50 connections are busy.
New request asks for a connection
Pool says: "None free"
Request WAITS... waits... waits...
Timeout! → Exception → 500
```

> **Real-life example:** Bank has All 50 are occupied. Customer number 51 waits in line forever, gives up, and files a complaint.

---

### Cause 3 — JVM Out of Memory (OOM) 🧠

The JVM has limited memory (called **heap**).

```text
Heap gets full
JVM cannot create new objects
OutOfMemoryError → 500 (and often the whole server starts dying)
```

> **Real-life example:** The teller's desk is completely covered with papers. No space to even pick up a new form. Everything stops.

---

## 5️⃣ What Do YOU Do as WAS Admin (step by step) 🛠️

### 🪜 Step 1: Confirm what the user saw

Ask: **exact time, which URL, which application.**

### 🪜 Step 2: Go to the logs — ALWAYS first

```bash
tail -200 /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/server1/SystemOut.log
```

Shows the last 200 lines. Errors will be at the bottom (latest events).

### 🪜 Step 3: Search for the actual exception

```bash
grep -A 10 "Exception\|Error\|SEVERE" SystemOut.log | tail -50
```

**What this does:**

| Flag | Meaning |
|------|---------|
| `grep "Exception"` | Finds lines containing errors |
| `-A 10` | Shows 10 lines **AFTER** the error (the full stack trace) |
| `tail -50` | Only show the last 50 matches (most recent) |

**You're looking for something like:**

```text
java.lang.NullPointerException
    at com.bank.transfer.TransferService.process(...)
```

> The `at com.bank...` line tells you **EXACTLY which code broke**.

### 🪜 Step 4: Check if it's a memory problem

```bash
grep "OutOfMemory\|heap" SystemOut.log | tail -10
```

If you see `OutOfMemoryError` → it's a **memory issue**, not a code 6️⃣ Whose Problem Is It? (Very important for interviews) 🎯

| Finding | Fix by |
|---------|--------|
| NullPointerException, code bugs | |
| DB connection pool exhausted | **DBA team** (or increase pool size) |
| OutOfMemoryError | **Admin +heap tuning, memory leak in code) |

**Your role as admin:**

> You are **NOT expected**. Your job:
>
> 1. 🔍 **Identify** — which application, which JVM, which exception
> 2. 📁 **Collect** — grab the logs and stack trace
> 3. 📤 **Escalate** — pass it to the right team with evidence

> 🏆 **Interview
> *"500 is an application error. The admin's job is to identify the failing application and exception from the logs, and hand it over to Dev or DBA with the stack trace — not to fix the code ourselves."*

---

## 7️⃣ Quick Memory Card 📌

```text
500 = Application CRASHED while processing

Top 3 causes:
1. Java exception (NullPointerException)
2. DB connection pool exhausted
3. JVM OutOfMemoryError

First command:
tail -200 SystemOut.log

Then:
grep the exception → find the stack trace

Fix belongs to: Dev team / DBA
Admin's job: Identify → Collect logs → Escalate
```

---

## 8️⃣ One-Line Answers (Interview rapid fire) ⚡

**Q: What is a 500 error?**
> A: The application threw an unhandled exception while processing the request.

**Q: Is 500 a web server problem?**
> A: No. Web server and network are fine. The problem is inside the Java application.

**Q: Where do you look first?**
> A: `SystemOut.log` of the affected JVM — using `tail` and `grep` for `Exception`/`Error`.

**Q: Who fixes it?**
> A: Dev team for code bugs, DBA for database issues. Admin identifies and escalates.

---

## 9️⃣ One-Line Summary

> **500 = "The teller tried — and collapsed."**
> Admin's job: **Identify → Collect logs → Escalate.** Never fix the code yourself. 🕵️
---
# 💥 PART 7 — 502 Bad Gateway (Beginner Friendly)

> Hi! I'm Ox Alpha, your WAS trainer. Let's learn this like I'm teaching you over coffee. ☕

---

## 🎯 What is a 502 in One Line?

**IHS (the receptionist) asked WAS (the teller) — and WAS gave garbage or silence back.**

That's it. Remember this.

---

## 🏦 The Bank Story (Never Forget This)

```text
1. You call the bank → Receptionist (IHS) picks up
2. Receptionist transfers you to Teller (WAS)
3. Teller either:
   ❌ Says "BRRRR" (garbage response)
   ❌ Doesn't pick up (no response)
4. Receptionist tells you: "Sorry... 502 Bad Gateway"
```

> 🔑 **Key point:** IHS is working **fine**. The problem is **behind it — with WAS**.

---

##: WAS JVM Crashed Mid-Response 💀

**What happens:**

1. Customer submits a big transaction (like a loan application)
2. JVM runs out of memory (heap) → **JVM dies** 💀
3. IHS already sent the request → gets nothing back → **502**

> **Real-life example:**
> Like the teller **fainting while counting your money**. The money disappears from the counter.

**Common triggers:**

| Trigger | Example |
|---------|---------|
| Huge reports | Month-end statements |
| Big file uploads | Customer document uploads |
| Memory leak in code | Objects never released |

---

## 🚨 Cause 2: WAS is Still Starting Up ⏳

**What happens:**

1. Someone restarted WAS
2. Big banking apps take **60–90 seconds** to load
3. User hits the site during startup → WAS not ready → **502**

> **Real-life example:**
> Teller hasn't come back from lunch yet, but receptionist already sent you to the **empty desk**.

> ✅ **Fix:**
> - Wait for WAS to fully start
> - Check logs for **"Application ... started"**

---

## 🚨 Cause 3: Plugin Timeout ⏱️

**What happens:**

```xml
<!-- plugin-cfg.xml -->
<ServerCluster ... ServerIOTimeout="60" />
```

1. Plugin waits **max 60 seconds** for WAS
2. WAS takes **90 seconds** (slow DB, heavy query)
3. Plugin gives up → **502**

> **Real-life example:**
> You wait 60 seconds on the phone, then hang up. The tell 61 — **too late!**

---

## 🔍 Quick Diagnosis (Your 3-Step Routine) 🛠️

### 🪜 Step 1 — Is WAS even alive?

```bash
ps -ef | grep java | grep AppSrv01
```

> No process? JVM **crashed** or **stopped**.

### 🪜 Step 2 — Did WAS crash or run out of memory?

```bash
grep -i "started\|stopped\|crash\|OutOfMemory" \
  /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/server1/SystemOut.log | tail -20
```

> Look for: `OutOfMemory`, crash, or recent **stop events**.

### 🪜 Step 3 — What does the Plugin say?

```bash
tail -50 /opt/IBM/WebSphere/AppServer/logs/http_plugin.log
```

> Look for: **timeout messages**, `"connection refused"`, `"server not responding"`.

---

## 🧠DNT"

> **502 = WAS is D-ead, N-ot ready, or T-oo slow**

| | Meaning | Check |
|--------|---------|-------|
| **D**ead | JVM crashed /ps -ef`, `SystemOut.log` |
| **N**ot ready | WAS still starting | Wait 60–90 sec, check `"started"` |
| **T**oo slow | TimeoutIOTimeout` in `plugin-cfg.xml` |

---

## 📋 Cheat Sheet Summary 📌

```text
502 = IHS is fine, WAS is the problem

Dead JVM     → check if java process exists
Restarting   → wait for full startup, don't rush
Timeout      → increase ServerIOTimeout or fix slow code/DB
```

---

## ❓ Quick Self-Test 🎯

1. Who is the receptionist and who is the teller?
2. What are the 3 causes of 502?
3. Which file holds `ServerIOTimeout`?

<details>
<summary>👉 Click for Answers</summary>

1. **IHS** and **WAS**
2. **Dead / Not ready / Too slow** (DNT)
3. **plugin-cfg.xml**

</details>

---

## 🔟 One-Line Summary

> **502 = "The receptionist reached the teller — but got garbage or silence."**
> IHS is fine. Check **WAS first**: alive? ready? fast enough? ☕
---
# 💥 PART 8 — 503 Service Unavailable (Senior Trainer Mode)

> Alright, sit down. Get a coffee. ☕ **This is THE one.**
> If you remember only one part of this whole course, remember this one.
> I've got 25 years of 2 AM phone calls. Almost all of them were **503**.

---

## 🎯 What is 503? (One Line)

**503 = "The server is alive, but it cannot serve you RIGHT NOW."**

That's it. Memorize that one line.

---

## 🏦 The Bank Analogy (Don't Skip This)

Picture a bank branch:

| Code | Story |
|------|-------|
| **502** | A teller picked up the phone and **spoke gibberish**. Someone answered, but uselessly. |
| **503** | You walk in and there are **NO tellers at the counter**. Or all tellers have **100 people in their queue** and physically cannot take one more. |

> The receptionist (**IHS**) is still at the desk. She's fine.
> But nobody behind her can work.
>
> So she says: *"Sorry, no one can serve you right now"* → **503**.

---

## 🔑 The 3 Possible Situations Behind EVERY 503

```text
1. WAS is DOWN           → the JVM is not running at all
2. Plugin THINKS WAS is DOWN → plugin-cfg.xml is stale/wrong
3. WAS is OVERWHELMED    → running, but too busy to accept you
```

> Everything below is just **these 3 situations in different clothes**.

---

## 🔥 The 5 Causes (MEMORIZE — this is your exam AND your job)

### Cause 1: WAS JVM is down ⭐ (most common — ~70% of 503s)

- JVM crashed, OOM killed it, or someone stopped it and **forgot to restart**
- Plugin tries every server in its list → all dead → **503**

> **Real life:** Patching team stopped `server1` at 6 PM for "maintenance" and went home. Forgot to restart. Users get 503 at 9 AM.

---

ale plugin-cfg.xml 📄

- New cluster member was added to WAS
- Plugin was **NOT regenerated**
- Plugin still has the **OLD server list** → tries dead/missing servers → **503**

> **Real life:** After every cluster change, **ALWAYS regenerate and copy** `plugin-cfg.xml` to IHS. This bites every junior at least once.

---

### Cause 3: WAS Thread Pool exhausted 🧵

```text
WebContainer thread pool max = 50
All 50 threads stuck on slow DB queries
New request arrives → no free thread → 503
```

> **Analogy:** All tellers are serving 1 customer each — but those customers have **500-page forms**. New customers get turned away.

> ✅ **Fix:** Temporary = restart WAS. Real = increase threads or fix the slow query.

---

### Cause 4: IHS itself overloaded 🌊

```apache
httpd.conf:  MaxClients = 256
```

- 257th connection arrives → queued → timeout → **503**

> **Real life:** Flash sale / salary day traffic spike. Config was fine for normal days — dies on peak days.

---

### Cause 5: Maintenance mode 🚧

- You deliberately put WAS down for deployment
- Plugin marks all503 for everyone**

> **Real life:** This 503 is **EXPECTED**. Check the deployment calendar before panicking!

---

## 🆚 502 vs 503 (This WILL be asked in interviews)

| | **502** | **503** |
|---|---------|---------|
| **Meaning** | Backend responded **BADLY** | Backend **not available AT ALL** |
| **Analogy** | Teller spoke gibberish | No teller / all busy |
| **Feeling** | Wrong answer | **No answer** |

---

## 🌙 The 2 AM Flowchart (Your Lifesaver)

> When the phone rings at 2 AM, **don't think. Follow the steps in order.**

### 🪜 Step 1 — Is IHS running?

```bash
ps -ef | grep httpd
```

> No → `apachectl start`. Done. ✅ Yes → Step 2.

### 🪜 Step 2 — Is WAS running?

```bash
ps -ef | grep java | grep AppSrv01
```

> No → `startServer.sh server1`. Done. ✅ Yes → Step 3.

### 🪜 Step 3 — Read the plugin log (your best friend 👯)

```bash
tail -100 /opt/IBM/HTTPServer/logs/http_plugin.log
```

**Look for:**
- `Could not connect to server`
- `marked down`

### 🪜 Step 4 — Is plugin-cfg.xml stale?

```bash
ls -la /opt/IBM/HTTPServer/conf/plugin-cfg.xml
```

> Older than the last WAS config change? → **Regenerate and copy it.**

### 🪜 Step 5 — Check thread pool

> Admin Console → **Servers → Thread Pools → WebContainer**
> Active = Max? → Pool exhausted. Restart WAS (quick fix), increase threads or fix slow query (real fix).

### 🪜 Step 6 — Check WAS logs

```bash
grep "Exception\|SEVERE\|OutOfMemory" SystemOut.log
```

---

## 🧠 Memory Trick: The "3 D's"

> **503 = 3 D's:**

| D | Meaning |
|---|---------|
| **Dead** | JVM is down |
| **Duped** | Plugin has wrong/stale info |
| **Drowned** | Thread pool / MaxClients exhausted |

> **Every 503 you'll ever see is one of these three.**

---

## ✅ Quick Recap (Say This Out Loud 🗣️)

```text
✔ 503 = server alive but cannot serve right now
✔ Top causes: JVM down, stale plugin-cfg.xml, thread pool full,
  IHS MaxClients hit, maintenance mode
✔ 502 = bad answer;  503 = no answer
✔ 2 AM order: IHS → WAS → plugin log → plugin-cfg.xml →
  thread pool → SystemOut.log
✔ Temporary fixes = restarts.  Real fixes = root cause.
```

---

## 🎓 Trainer's Final Tip

> 📌 Keep the plugin log command on a **sticky note on your monitor**.
> 9 out of 10 times, the answer is written right there.
>
> **Juniors panic and restart everything. Seniors read the log first.**
> **Be a senior.** 🏆

---

## 🔟 One-Line Summary

> **503 = "The bank is open — but there's no teller to serve you."**
> IHS is fine. Find out **why WAS can't serve**: Dead, Duped, or Drowned. 🌙
