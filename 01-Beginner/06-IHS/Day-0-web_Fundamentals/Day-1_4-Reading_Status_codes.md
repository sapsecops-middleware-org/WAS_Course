---
# 🛠️ PART 10 — Reading Status Codes with curl and F12

> So far you've learned what the codes **mean**.
> Now let's learn how to **catch them in the act** — like a detective. 🕵️
> Two weapons: **curl** (from the server) and **F12** (from the browser).

---

## 🎯 Why This Part Matters (One Line)

**Users say "the site is slow/broken" — curl and F12 tell you WHERE and WHY, in seconds.**

---

## 🧰 Weapon 1: curl (Your Command-Line Detective)

### 🔍 Check 1 — Get only the status code (clean & fast)

```bash
curl -o /dev/null -s -w "%{http_code}\n" https://www.citibank.co.in/netbanking/
```

**Output:**
```text
200
```

| Flag | Meaning |
|------|---------|
| `-o /dev/null` | Throw away the page body (we only want the code) |
| `-s` | Silent — no progress bar |
| `-w "%{http_code}\n"` | Print **just the HTTP status code** |

> 📌 **Memorize this one command.** You will use it every single day.

---

### 🔍 Check 2 — Full verbose (see everything)

```bash
curl -v https://www.citibank.co.in/netbanking/ 2>&1 | grep "< HTTP"
```

**Output:**
```text
< HTTP/1.1 200 OK
```

> The `<` lines are what the **server sent back**. The `>` lines are what **you sent**.

---

### 🔍 Check 3 — Follow redirects (watch each hop)

```bash
curl -L -v http://www.citibank.co.in/ 2>&1 | grep "< HTTP\|Location"
```

**Output:**
```text
< HTTP/1.1 301 Moved Permanently
< Location: https://www.citibank.co.in/
< HTTP/1.1 200 OK
```

**What happened:**

```text
Hop 1: http  → server says "301 — go to https instead"
Hop 2: https → server says "200 OK" ✅
```

> `-L` = follow redirects. Without it, you'd only see the 301 and think something's wrong!

---

### 🔍 Check 4 — Test WAS directly (BYPASS IHS) ⭐

```bash
curl -o /dev/null -s -w "%{http_code}\n" http://192.168.2.10:9080/netbanking/health
```

**How:**

| Result | Meaning |
|--------|---------|
| `200` | WAS is fine ✅ — problem is IHS or plugin |
| `000` | WAS not reachable (connection refused) ❌ — JVM down? |
| `503` | WAS is overloaded ⚠️ — thread pool full? |

> 🏆 **This is the senior's trick:**
> - Via IHS = 503, **direct to WAS :9080 = 200** → problem is **IHS/plugin**, not WAS!
> - Direct to WAS also fails → problem is **WAS itself**.
>
> One curl command splits the problem in half.

---

## 🖥️ Weapon 2: F12 (Browser DevTools — No Server Access Needed)

**When to use:** You're at your desk (or the user's desk) with just a browser.

### 🪜 Step-by-Step

1. Open **Chrome → press F12** → click the **Network** tab
2. Type your banking URL and press **Enter**
3. Click the **first request** in the list (the main page)
4. Look at these three things 👇

---

### 👀 What to Look At #1 — Status

```text
Status: 200    ← The code
```

---

### 👀 What to Look At #2 — Response Headers

```text
Headers tab → Response Headers:

  X-Powered-By: WebSphere Application Server/9.0
  ↑ Confirms WAS is the one responding

  Set-Cookie: JSESSIONID=ABC123:0001; Secure; HttpOnly
  ↑ Note the :0001  ← This is the CLONE ID!
```

> 🔑 **Clone ID bonus (interview gold!):**
> In a cluster, each JVM has a clone ID (`:0001`, `:0002`...).
> It's encoded in `JSESSIONID` so the plugin can route the user
> back to the **same JVM** (session affinity/sticky sessions).
> If users keep getting logged out → check if clone IDs are rotating!

---

### 👀 What to Look At #3 — Timing

```text
Timing tab:

  Waiting (TTFB): 1,250 ms    ← If this is >3s, WAS is struggling
```

> **TTFB = Time To First Byte.**
> It measures how long the server **thought** before answering —
> i.e., how long WAS spent processing.

**How to read TTFB:**

| TTFB | Verdict |
|------|---------|
| < 500 ms | Healthy ✅ |
| 0.5 – 3 s | Getting slow ⚠️ — check DB queries |
| > 3 s | WAS is struggling ❌ — thread pool, GC, or slow backend |

---

## 🆚 curl vs F12 — When to Use Which?

| | **curl** 🧰 | **F12** 🖥️ |
|---|------------|-----------|
| Where | On the server (SSH) | Any browser |
| Best for | Bypass tests (direct to WAS :9080) | Headers, cookies, clone ID, timing |
| Scripts? | ✅ Yes — automate health checks | ❌ Manual |
| User demo | ❌ | ✅ Perfect for showing users/devs |

> 🏆 **Pro combo:** F12 shows 503 → SSH to server → curl WAS directly →
> 200? → it's IHS/plugin. That's a full diagnosis in under 2 minutes.

---

## 📋 Cheat Sheet Summary 📌

```text
✔ One-liner:  curl -o /dev/null -s -w "%{http_code}\n" <URL>
✔ Verbose:    curl -v ... | grep "< HTTP"
✔ Redirects:  curl -L -v ... (watch each hop)
✔ Bypass IHS: curl http://<was-ip>:9080/...  → splits problem in half
✔ F12 Network: Status → Headers (clone ID!) → Timing (TTFB)
✔ TTFB > 3s   → WAS is struggling
✔ 000 in curl → connection refused — WAS down or wrong port
```

---

## ❓ Quick Self-Test 🎯

1. WAS gives 503 via IHS but 200 when curled directly on :9080 — where's the problem?
2. What does `:0001` in the JSESSIONID mean?
3. TTFB is 4 seconds. What does that tell you?

<details>
<summary>👉 Click for Answers</summary>

1. **IHS or the plugin** — WAS itself is healthy.
2. The **clone ID** — identifies which cluster JVM (JVM #1) owns the session (sticky sessions).
3. **WAS is struggling** — check thread pool, GC, or slow DB queries.

</details>

---

## 🔟 One-Line Summary

> **curl tells you WHAT the server said. F12 tells you WHY it took so long.**
> Master both, and "the site is broken" becomes a 2-minute diagnosis. 🕵️‍♂️
---
# 🏦 PART 11 — Banking Scenario (Real 2 AM Story)

> Everything you learned in Parts 5–10 — now used for real.
> This is the night you put it all together. 🌙

---

## 🚨 The Alert

```text
⏰ 2:15 AM — PagerDuty alert fires:

"NetBanking 503 — Customers Cannot Login"

 Worse: NEFT/RTGS batch is running.
 3,000 scheduled transfers haven't processed.
 Money is stuck. The clock is ticking.
```

> 🧯 **Rule #1 at 2 AM: Don't think. Follow the flowchart.**
> (You already built it in Part 8 — now we run it.)

---

## 🛠️ The Rescue — Step by Step

### 🪜 Step 1 — SSH to IHS server (30 seconds)

```bash
ssh wasadmin@ihs-server-01.citibank.internal
```

---

### 🪜 Step 2 — Is IHS running?

```bash
ps -ef | grep httpd | grep -v grep
```

**Output:**
```text
4 httpd processes → IHS is UP ✅
```

> Note the `grep -v grep` — it removes the grep process itself from results.
> Small habit, clean output.

---

### 🪜 Step 3 — Quick curl test

```bash
curl -o /dev/null -s -w "%{http_code}\n" http://localhost/netbanking/health
```

**Output:**
```text
503
```

> Confirmed: users aren't lying. It's a real 503.

---

### 🪜 Step 4 — Check plugin log (your best friend 👯)

```bash
tail -50 /opt/IBM/HTTPServer/logs/http_plugin.log
```

**Output:**
```text
[error] ws_server: serverSetFailoverStatus: Marking ws-was-01:9080 down
[error] ws_server: serverSetFailoverStatus: Marking ws-was-02:9080 down
[error] ws_common: websphereFindTransport: Failed to find an available server
```

**Reading it like a senior:**

```text
ALL WAS servers marked down → IHS is fine, the problem is BEHIND it.
Go check WAS. →
```

> 🎯 We just eliminated half the architecture in 2 minutes.
> That's the power of the plugin log.

---

### 🪜 Step 5 — SSH to WAS server — is it running?

```bash
ssh wasadmin@was-node-01.citibank.internal

ps -ef | grep java | grep AppSrv01
```

**Output:**
```text
(nothing) ← WAS is NOT running! ❌
```

> From the "3 D's": this 503 was **D-ead**, not **D-uped** or **D-rowned**.

---

### 🪜 Step 6 — WHY did it die? Check WAS logs

```bash
tail -100 /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/server1/SystemOut.log
```

**Output:**
```text
java.lang.OutOfMemoryError: Java heap space
```

> 💀 **Root cause found:** JVM ran out of heap memory and crashed.
> (Remember Part 7? Big batch jobs + customer load = heap pressure.)

---

### 🪜 Step 7 — Start WAS

```bash
cd /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/bin/
./startServer.sh server1
```

---

### 🪜 Step 8 — Watch it come up (2–3 minutes)

```bash
tail -f /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/server1/SystemOut.log
```

**Wait for:**
```text
server1 open for e-business ✅
```

> 🔑 From Part 7: **don't rush users back in before it's fully started!**
> "Open for e-business" is the green light.

---

### 🪜 Step 9 — Test again

```bash
curl -o /dev/null -s -w "%{http_code}\n" http://localhost/netbanking/health
```

**Output:**
```text
200 ← RESTORED! 🎉
```

---

### 🪜 Step 10 — Log the incident (NEVER skip this)

```text
Incident Report:
  Root cause: JVM heap exhausted due to NEFT batch + overnight customer load
  Action:     Increase JVM max heap from 2GB → 3GB tomorrow (change request)
```

> ⚠️ **The restart is the TEMPORARY fix. The heap increase is the REAL fix.**
> Skip Step 10 and you'll be back here next Tuesday at 2 AM. Same alert. Same coffee. ☕😅

---

## ⏱️ The Scoreboard

```text
Total time from alert to resolution: ~15 minutes
```

| Admin | Time | Approach |
|-------|------|----------|
| 🥉 Junior | 1–2 hours | Panics, restarts everything randomly, skips logs |
| 🥇 Senior | **15 minutes** | **Systematic: IHS → plugin log → WAS → logs → fix** |

---

## 🧠 What Made This Fast? (The Real Lesson)

```text
1. Followed a FLOWCHART, not feelings
2. Plugin log eliminated IHS instantly
3. Checked process BEFORE logs (is it alive? then why?)
4. Found root cause (OOM) BEFORE restarting
5. Tested with curl AFTER starting (verify, don't assume)
6. Logged the root cause → permanent fix scheduled
```

> 🎓 **Trainer's rule: "Restart without root cause = scheduling your next outage."**

---

## 📋 Cheat Sheet Summary 📌

```text
✔ Alert → SSH IHS → ps httpd → curl → plugin log
✔ "All servers marked down" → go to WAS
✔ No java process → check SystemOut.log for WHY
✔ OOM in log → start server → tail -f → "open for e-business"
✔ curl = 200 → verify → log root cause → permanent fix
✔ 15 min systematic > 2 hours panicking
```

---

## ❓ Quick Self-Test 🎯

1. The plugin log says all servers are "marked down." What does that tell you immediately?
2. You find no java process. Should you restart right away? Why not?
3. What's the difference between the temporary fix and the real fix here?

<details>
</summary>

1. **IHS is fine** — the problem is behind it (WAS). Half the architecture eliminated instantly.
2. **No — check the logs FIRST.** You need the root cause (here: OOM); otherwise the crash will repeat.
3. Temporary = restarting WAS. Real = increasing heap (2GB → 3GB) so the batch workload fits.

</details>

---

## 🔟 One-Line Summary

> **A 2 AM outage doesn't test what you know — it tests whether you follow the flowchart instead of panicking.**
> IHS → plugin → WAS → logs → fix → verify → log. 15 minutes. Be the senior. 🏆
---
# 🎯 HTTP Status Codes in IHS & WAS — Simple Guide

| Code | Meaning | Layer | Your Action |
|------|---------|-------|-------------|
| **200** ✅ | Success | IHS or WAS | Nothing — life is good |
| **301** | Permanent Redirect | IHS | Verify config is correct |
| **302** | Temp Redirect | Usually WAS | Normal app behavior |
| **403** 🚫 | Forbidden | IHS | Check permissions / IP rules |
| **404** 🔍 | Not Found | IHS or WAS | Check URL, deploy, plugin |
| **500** 💥 | App Crashed | WAS | Check `SystemOut.log` |
| **502** ⚠️ | Bad Response from Backend | Plugin → IHS | Check WAS crash / timeout |
| **503** 🚨 | Backend Unavailable | Plugin → IHS | Check WAS down / thread pool ⚠️ |
---
## Simple Explination

---

## ✅ 200 — Success

| | |
|---|---|
| **Meaning** | Everything worked fine. |
| **Where** | IHS or WAS. |
| **Your action** | Nothing. 😌 |

**Example:** You open your balance page, money shows. Done.

> 🧠 **Remember:** 200 = All good. Move on.

---

## 🔁 301 — Permanent Redirect

| | |
|---|---|
| **Meaning** | The URL has **moved forever** to a new address. |
| **Where** | Mostly IHS. |
| **Your action** | Check IHS config is correct (e.g., `httpd.conf` rewrite rules). |

**Example:** Bank moved from `oldbank.com` to `newbank.com`. Old address always points to the new one. **Even your bookmarks work.**

> 🧠 **Remember:** 301 = Moved house permanently. Update your address book.

---

## 🔁 302 — Temporary Redirect

| | |
|---|---|
| **Meaning** | Sent to another page **just for now**. |
| **Where** | Usually WAS (your app does it). |
| **Your action** | Normal. No issue. ✅ |

**Example:** You try to open your account page without logging in → app redirects you to the login page. After login, back to normal.

> 🧠 **Remember:** 302 = Temporarily away. Will be back.

---

## 🚫 403 — Forbidden

| | |
|---|---|
| **Meaning** | "I know who you are, but you are **not allowed** here." |
| **Where** | IHS. |
| **Your action** | Check permissions, IP blocking rules in IHS config. |

**Example:** You try to enter the bank's staff-only room. You're a valid customer, but this room is not for you. The guard says **NO**.

**Common causes:**
```text
✔ Your IP is blocked
✔ File/folder permissions wrong
✔ IHS security rules deny the request
```

> 🧠 **Remember:** 403 = The door is locked AND the guard knows why.

---

## 🔍 404 — Not Found

| | |
|---|---|
| **Meaning** | The page/URL **does not exist**. |
| **Where** | Could be IHS **or** WAS. |
| **Your action** | • Check the URL is typed correctly<br>• Check app is deployed properly in WAS<br>• Check plugin is routing to the right app |

**Example:** You ask for a "Gold Savings Account" form, but the bank never made that form. It doesn't exist.

> 🧠 **Remember:** 404 = You asked for something that isn't there.

---

## 💥 500 — Internal Server Error

| | |
|---|---|
| **Meaning** | The application **crashed while running**. |
| **Where** | WAS. |
| **Your action** | Check `SystemOut.log` (and `SystemErr.log`) in WAS logs. **Look for stack traces.** |

**Example:** You ask for a loan, and the officer's computer crashes mid-process. The problem is **inside the office (WAS)**, not at the door.

> 🔑 **Key point:** 500 = The app itself has a bug or error. Look at logs.

> 🧠 **Remember:** 500 = The officer fainted at his desk.

---

## 🔌 502 — Bad Gateway

| | |
|---|---|
| **Meaning** | IHS asked WAS for an answer. WAS gave a **bad or broken answer** (or crashed mid-way). |
| **Where** | Plugin → IHS. |
| **Your action** | • Check if WAS crashed<br>• Check timeouts<br>• Check WAS logs |

**Example:** The receptionist (plugin) asks the officer (WAS) a question. The officer gives a **nonsense answer or collapses**. Receptionist says: *"Sorry, I got a bad response."*

> 🧠 **Remember:** 502 = The middleman got a bad answer from the backend.

---

## ⚠️ 503 — Service Unavailable

| | |
|---|---|
| **Meaning** | WAS is **completely down or too busy**. Cannot take requests at all. |
| **Where** | Plugin → IHS. |
| **Your action** | • Is WAS running? Check server status<br>• Check thread pool (all workers busy?)<br>• Restart WAS if needed |

**Example:** Bank is **closed for the day**, or the queue is so long the guard **stops letting people in**. *"Come back later."*

> 🧠 **Remember:** 503 = Backend is dead or full. **Most serious of the 5xx errors.**

---

## 🗂️ The Deck at a Glance (Memory Hooks Only)

| Code | Memory Hook |
|------|-------------|
| 200 ✅ | All good. Move on. |
| 301 🔁 | Moved house **permanently**. |
| 302 🔁 | Away **temporarily**. Will be back. |
| 403 🚫 | Guard knows you — still says NO. |
| 404 🔍 | That thing doesn't exist. |
| 500 💥 | Officer fainted at his desk. |
| 502 🔌 | Middleman got a **bad answer**. |
| 503 ⚠️ | Backend **dead or full**. Worst one. |

---

## ❓ Quick Self-Test 🎯

1. Redirect after login — 301 or 302? Why?
2. 404 appears with no deployment — which file do you diff first?
3. Receptionist asks the officer, officer collapses mid-sentence — which code?

<details>
<summary>👉 Click for Answers</summary>

1. **302** — it's temporary/conditional (you're only redirected because you're not logged in). 301 is for *forever* moves like domain changes.
2. **plugin-cfg.xml** (UriGroup section) — then httpd.conf, per Part 12 Q2.
3. **502** — bad/incomplete answer from backend. (If he refuses to even come to the desk → 503.)

</details>

---

## 🔟 One-Line Summary

> **Eight codes. Eight cards. Each one tells you the layer, the log, and the fix — before you even type a command.**
> Master the deck, and production errors stop being surprises. They become checklists. 🃏🏆
