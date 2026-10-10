# DAY 53 — Full Failover, Explained from Absolute Zero

> [!NOTE]
> Taught by Ox Alpha — plain English, everyday examples, no jargon assumed.

---

## 🧠 Part 1: First, Understand the Problem in Daily Life

Imagine you go to a bank branch to transfer money:
1. You meet a clerk named Mr. Sharma.
2. You hand him your documents. He starts writing your transfer slip.
3. Halfway through writing, Mr. Sharma faints and collapses.

Now ask yourself:
* Does the bank say *"sorry, come tomorrow"*? ❌ **No.**
* Does another clerk take over? ✅ **Yes.**
* But here's the problem: **Does the new clerk know what you were doing?**

The new clerk only knows **IF** the bank has a system where clerks share their notes with each other:
* **If clerks copy their notes to each other every few seconds** $\rightarrow$ the new clerk picks up the notes and continues smoothly. ✅
* **If they don't share notes** $\rightarrow$ the new clerk says *"Who are you? Fill a new form. Start from the beginning."* ❌

That is the entire story of failover. Now let's map this directly to technology.

---

## 🧩 Part 2: The Characters — Meet Everyone One by One

### 1️⃣ The Browser (Ravi's Computer)
Ravi uses a website. His browser (Chrome, Edge, etc.) is like you, the customer standing at the counter.

### 2️⃣ IHS (IBM HTTP Server) — "The Reception Desk"
Before your request reaches the actual application, it first lands on a dedicated web server called IHS. Think of it as the receptionist at the bank entrance. She doesn't do the core banking work herself — she simply routes and passes your papers.

### 3️⃣ The Plugin — "The Smart Receptionist's Guidebook"
Inside IHS, there is a small intelligent component called the WebSphere web server plugin. It functions like a rulebook the receptionist follows that dictates:
* *"Send customer #1 to Counter A."*
* *"Counter A is closed? Send them to Counter B."*

The plugin evaluates incoming requests and decides which application server gets your traffic.

### 4️⃣ JVM1, JVM2, JVM3, JVM4 — "The 4 Counters"
The actual banking application runs across 4 identical servers, called **JVMs** (*Java Virtual Machines* — simply think: 4 identical instances of the banking app).

Why run 4 copies?
* One computer can handle only so many concurrent users.
* If one dies, the others keep working without disruption. This setup is called a **cluster**.

### 5️⃣ Session — "Your File at the Counter"
When you log in, the server creates a temporary file containing your context:
* Your user ID, your login status, what page you're on, and uncommitted form entries.

This file is called a **session**.

> [!WARNING]
> This session file lives in that specific server's memory (RAM) by default — not on disk. If that server process dies or crashes, the file normally vanishes instantly with it.

### 6️⃣ JSESSIONID — "Your Token Number"
How does the server know which memory file belongs to you? When you authenticate, it provides your browser with a small tracking cookie called `JSESSIONID`. Your browser presents it on every subsequent HTTP request, acting like a token number at a bank counter.

It looks like this:
```text
JSESSIONID = xT9k82mB:cloneABC1
```

It contains two distinct parts:
* `xT9k82mB` $\rightarrow$ Your unique personal session identifier.
* `cloneABC1` $\rightarrow$ A hidden label called the **CloneID**, indicating: *"My primary session file lives on JVM1."*

The CloneID allows the web server plugin to maintain **session affinity** (sticky routing).

### 7️⃣ LTPA Token — "Your Entry Pass"
When Ravi logs in with his username and password, WebSphere issues an encrypted token called an **LTPA token** (*Lightweight Third-Party Authentication*). It serves as cryptographic proof: *"This person is already verified — do not prompt for the password again."* It behaves like a security wristband at an event — show the wristband, walk through any checkpoint freely.

### 8️⃣ Replication (DRS) — "Clerks Sharing Notes"
This is the redundancy mechanism that prevents data loss. The 4 JVMs continuously synchronize each other's active session data across memory. This is called **memory-to-memory replication (M-to-M)**, managed internally by the **Data Replication Service (DRS)**.

In this architecture:
* Each active session is replicated to **2 backup servers**.
* The replication broadcast triggers **every 2 seconds**.

Like clerks photocopying their in-flight case files to 2 colleagues every 2 seconds.

### 9️⃣ Two Critical Plugin Timers
* `ConnectTimeout = 5 seconds` $\rightarrow$ If a JVM fails to respond to a connection attempt within 5 seconds, the plugin flags it as unresponsive.
* `RetryInterval = 60 seconds` $\rightarrow$ Every 60 seconds, the plugin sends a health probe to the dead server: *"Are you back online?"* If it answers, it resumes routing traffic to it.

---

## 🎬 Part 3: The Story — Step by Step, Slowly

```
[Ravi's Browser] ──> [IHS Web Server (Plugin)] ──> [JVM1 (Active Session)]
                                                        │
                                    (DRS M-to-M Copy)   ├──> [JVM2 (Backup Copy)]
                                                        └──> [JVM3 (Backup Copy)]
```

### 📌 Step 1: Ravi Logs In (11:10 AM)
1. Ravi's browser sends an authentication request $\rightarrow$ IHS $\rightarrow$ Plugin $\rightarrow$ (initial round-robin) $\rightarrow$ **JVM1**.
2. JVM1 authenticates the user and creates his in-memory session file.
3. Ravi receives a response with the cookie: `JSESSIONID = xT9k82mB:cloneABC1` (the CloneID designates JVM1).
4. Ravi is fully authenticated; the LTPA token is stored within the session context.

### 📌 Step 2: The Background Replication (11:10–11:15 AM)
While Ravi is busy filling out his NEFT transfer form (Beneficiary Account, IFSC Code, ₹2,50,000):

Every 2 seconds:
* **JVM1:** *"Here is my latest delta of Ravi's session data"* $\rightarrow$ broadcasts via DRS to **JVM2** and **JVM3**.
* **JVM2 & JVM3:** Store the received state as an unpromoted **backup copy** in RAM.

At any given second, JVM2 holds a state that is at most 2 seconds old.

### 📌 Step 3: The Click and the Crash (11:15:03 AM)
Ravi clicks **"Confirm Transfer"**. His browser dispatches an HTTP `POST` request.

> [!NOTE]
> A `POST` request carries payload meant to modify persistent state (such as debiting an account). A `GET` request retrieves data idempotently — it is safe to repeat without side effects.

The `POST` reaches JVM1...

💥 **JVM1 experiences a fatal Out of Memory (OOM) error.** The Java process crashes. Listening port 9080 goes silent. Ravi sees a loading spinner on his screen.

---

## ⏱️ Part 4: What Happens Next — Second by Second

### ⏱️ Seconds 0–5: The Plugin Waits, Then Declares Death
The plugin has already forwarded Ravi's `POST` request to JVM1, but no TCP response packets return.
1. The plugin waits up to 5 seconds (`ConnectTimeout`).
2. Upon reaching the 5-second boundary without an ACK/response, the plugin marks JVM1 as offline in its shared runtime table:

```text
Server "jvm1:9080" has been marked down. Fail-over to next server.
```

The plugin maintains an internal operational scorecard across all 4 JVMs. JVM1 is now flagged with a down status.

### ⏱️ Second 5: Ravi Receives an Error — Why Not Auto-Retry?
WebSphere implements strict traffic protection guidelines based on HTTP method semantics:

| Request Type | Semantic Meaning | Plugin Failover Action |
|:---|:---|:---|
| **GET** | Read-only; repeating it carries no persistent side effects. | Plugin silently retries the request on JVM2. ✅ |
| **POST** | State-changing transaction (e.g., balance debit). | Plugin **does not** auto-retry. ❌ |

#### Why the plugin rejects auto-retrying a POST
JVM1 crashed while processing the transfer. If it completed the database commit 1 millisecond before the process crashed, an automated retry forwarded to JVM2 would cause the database execution to run a second time — transferring ₹5,00,000 instead of ₹2,50,000.

To avoid double-submission, the web server plugin halts automatic resubmission of non-idempotent methods. Ravi receives an HTTP 503 error page: *"Request could not be processed. Please try again."*

> [!TIP]
> **Interview Line:** *"The WebSphere plugin protects against transaction double-submission by refusing to auto-retry failed POST requests."*

### ⏱️ Second 5.1: Ravi Clicks "Try Again" — The Critical Moment
Ravi clicks retry. His browser sends a fresh request carrying the existing cookie:
```http
Cookie: JSESSIONID=xT9k82mB:cloneABC1
```

The plugin parses the header:
1. The CloneID states: *"This user is pinned to JVM1."*
2. The internal scorecard states: *"JVM1 is marked DOWN."*
3. Algorithmic override: *"If the preferred primary server is marked down, route to the next available healthy server in the cluster."*
4. The plugin routes the request to **JVM2**. ✅

### ⏱️ Second 5.2: JVM2 Receives Ravi — Two Possible Scenarios
How the cluster responds depends on whether session replication was correctly implemented.

---

## 🌍 Part 5: Two Worlds — The Heart of Everything

```
                       [Ravi Retries Request]
                                 │
                                 ▼
                     [JVM2 Receives Request]
                                 │
         ┌───────────────────────┴───────────────────────┐
         ▼                                               ▼
   [WORLD 1: No Replication]                   [WORLD 2: DRS Replication]
   • Session not found in RAM                  • Found in backup store
   • Empty session created                     • Promoted: BACKUP ──> PRIMARY
   • LTPA token lost                           • LTPA validated: No re-login
   • Redirect to /login                        • Form pre-filled, user submits
```

### ❌ World 1: No Replication Configured (The Unprepared Bank)
* JVM2 looks up the session: *"Token `xT9k82mB`? No record exists in my local RAM."*
* JVM2 creates an empty, unauthenticated session container.

**The User Impact:**
* Ravi is redirected to the authentication login screen.
* His active session is discarded; the LTPA token context is lost.
* The form input values are wiped out.
* The user cannot verify whether the funds were debited, causing customer friction and support desk escalation.
* The operational payment SLA is missed.

### ✅ World 2: Replication Configured via DRS (The Production Setup)
JVM2 has been receiving synchronized backup copies of Ravi's session every 2 seconds. The crash occurred 3 seconds after the last synchronization window.

**Internal JVM2 Sequence:**
1. Lookup: *"Do I have session ID `xT9k82mB`?"*
2. Result: Match found in the local **backup memory table**. ✅
3. State Transition: JVM2 executes **Session Promotion** (upgrades backup copy to **PRIMARY** owner).
4. Deserialization: Reconstructs the live session state from the replicated byte stream.
5. Security: Extracts and validates the embedded LTPA token.
6. Identity Confirmation: Authentication remains intact — bypasses credentials challenge. ✅

**The User Impact:**
* Authentication is maintained; no login screen or password prompt is displayed.
* The transfer form displays with prior form values intact.
* A banner notifies: *"Your previous transaction could not be completed. Please verify and resubmit."*
* Ravi confirms details, submits, and completes the transaction on time.

---

## 📊 Part 6: Full Timeline on One Page

| Time | JVM1 | IHS / Plugin | JVM2 | Ravi |
|:---|:---|:---|:---|:---|
| **11:14:58** | Primary active | Routes to JVM1 | Backup updated via DRS ✅ | Enters form data |
| **11:15:00** | Synced to JVM2 | Operational | Backup updated via DRS ✅ | Enters form data |
| **11:15:03** | 💥 **CRASH (OOM)** | Forwards POST; waits | Holds backup state | Clicks "Submit" |
| **11:15:08** | Process dead | Marks JVM1 DOWN (5s timeout) | Idle | Receives 503 error |
| **11:15:09** | Dead | Routes retry to JVM2 | Locates backup; promotes to PRIMARY | Clicks "Try Again" |
| **11:15:10** | Dead | Operational | Session live on JVM2 | Sees pre-filled form ✅ |
| **11:28:00** | Dead | Operational | Processes POST; commits to DB | Sees confirmation screen |
| **11:45:00** | Admin restarts JVM1 | Health check succeeds (`RetryInterval`) $\rightarrow$ Marks UP ✅ | Continues running | Unaware of backend changes |

When JVM1 is restarted, the plugin automatically detects that the port is listening (via the `RetryInterval` probe of 60 seconds), re-enables it in the active server pool, and routes new traffic to it without requiring manual intervention.

---

## 🧯 Part 7: What Is Still Lost — Even with Perfect Replication?

Failover mechanisms protect application state, but physical process crashes still carry boundary conditions:

| Component | Survived? | Technical Reason |
|:---|:---:|:---|
| **Last ~2 seconds of session data** | ⚠️ At Risk | Replication runs on a 2-second interval timer. Unsynchronized deltas within the crash window are lost. |
| **The active in-flight POST request** | ❌ Lost | JVM1 terminated execution mid-flight. The socket closed before completion. |
| **The server response payload** | ❌ Lost | JVM1 crashed before returning HTTP headers; the plugin returned a local 503. |
| **The database record** | ❌ Not Committed | The crash happened before the JTA transaction committed to the database storage engine. |
| **Authentication / Login state** | ✅ Survived | The session container and serialized LTPA tokens were preserved in JVM2's backup memory. |
| **Form field data** | ⚠️ Partial | Retained if the application architecture commits form deltas into the HTTP session prior to submission; lost if stored only in the browser DOM. |
| **Duplicate debit protection** | ✅ Protected | The plugin blocks automatic retries of `POST` requests, preventing multiple executions. |

> [!TIP]
> **Interview Line:** *"Failover preserves session identity and authentication, but an in-flight transaction is lost if the JVM crashes before commit. The user must resubmit, which is why banking applications instruct the user to verify and try again."*

---

## 🧰 Part 8: The Configuration — What You Actually Set Up

Failover requires configuring the Application Server runtime, the replication domain, and the web server plugin.

```
       [WebSphere Cell Console]
                  │
  ┌───────────────┴───────────────┐
  ▼                               ▼
[Session Management]       [plugin-cfg.xml]
  • M-to-M Enabled           • ConnectTimeout = "5"
  • Replicas = 2             • RetryInterval = "60"
  • Interval = 2 sec
```

### 1️⃣ Enable Session Replication (WAS Admin Console)
Navigate to:
`Servers` $\rightarrow$ `Server Types` $\rightarrow$ `WebSphere application servers` $\rightarrow$ `[Server_Name]` $\rightarrow$ `Web Container Settings` $\rightarrow$ `Session Management` $\rightarrow$ `Distributed environment settings`

* Select **Memory-to-memory replication**.
* **Replication domain:** Attach the shared domain common to the cluster.
* **Number of replicas:** Set to `2` (the primary session is mirrored across two peer JVMs).

> [!WARNING]
> This configuration must be applied across every node member in the cluster (JVM1, JVM2, JVM3, and JVM4). Missing a single node creates an asymmetrical cluster where sessions landing on that server cannot be recovered.

### 2️⃣ Tune Backup Frequency
Navigate to:
`Session Management` $\rightarrow$ `Distributed environment settings` $\rightarrow$ `Custom Tuning Parameters`
* Set **Tuning method** to **Very high (write contents on time intervals)**.
* Set **Write interval** to `2` seconds.

A lower interval decreases the window of potential data loss, but increases DRS network utilization across the backplane.

### 3️⃣ Enable CloneID Visibility
In the Web Container custom properties, verify that session cookie rewriting includes the server CloneID (e.g., `cloneABC1`). Without the CloneID suffixed to the `JSESSIONID` string, the IHS plugin cannot identify which server maintains primary ownership.

### 4️⃣ Configure Plugin Timers (`plugin-cfg.xml`)
The generated configuration file read by the web server must declare the timeout policies:

```xml
<ServerCluster Name="AppCluster">
   <Server Name="jvm1_node">
      <Transport Hostname="app-host1" Port="9080" Protocol="http"/>
   </Server>
   <Server Name="jvm2_node">
      <Transport Hostname="app-host2" Port="9080" Protocol="http"/>
   </Server>

   <!-- Connection & Failure Recovery Timers -->
   <Property Name="ConnectTimeout" Value="5"/>
   <Property Name="RetryInterval" Value="60"/>
</ServerCluster>
```
- `ConnectTimeout = 5` → declare a server dead after 5 seconds of silence
- `RetryInterval = 60` → poke dead servers every 60 seconds to detect recovery

> [!IMPORTANT]
> After editing, remember: **regenerate and propagate** the `plugin-cfg.xml`, then **restart IHS**. A changed file that never reached the web server does nothing.

> [!TIP]
> **Interview line:** "Failover configured in three layers: session replication on the JVMs, the replication domain, and plugin timeouts on IHS. All three must agree."

---

## 🧪 Part 9: How You Prove It Works — The Actual Test

In real projects, you don't trust configuration on faith. **You test it.** Here's the standard lab test:

### Setup:

1. Deploy a tiny test app that shows: "Session ID: ___, Server name: ___" on screen.
2. Log in through IHS. Note which JVM served you (shown on screen).
3. Fill a form field with the word **HELLO** and save (so it goes into the session).

### The kill:

4. While staying on the same browser window, open the WAS admin console and **stop JVM1** (or `kill -9` the process on Linux).
5. In the browser, refresh or submit again.

### Pass criteria:

- ✅ The page still loads (you did **NOT** get a login page)
- ✅ Same JSESSIONID token in your browser cookie
- ✅ The word **HELLO** is still remembered (session survived)
- ✅ Screen now shows a **different server name** (JVM2 or JVM3)

Check the logs to complete the story:

**Plugin log** (`http_plugin.log` on IHS):

```text
Server "jvm1:9080" has been marked down. Fail-over to next server.
```

**JVM2's `SystemOut.log`**: the session was found in the backup store and promoted.

> [!NOTE]
> If any check fails, typical culprits: replication not enabled on one JVM, replicas set to 0, wrong replication domain, or `plugin-cfg.xml` never regenerated.

> [!TIP]
> **Interview line:** "I validated failover by killing a JVM mid-session and confirming session continuity, preserved JSESSIONID, and automatic plugin rerouting in the logs."

---

## 📝 Part 10: Glossary — Every Term in One Line

| Term | One-line meaning |
|---|---|
| **Session** | The server's memory file about one logged-in user |
| **JSESSIONID** | The token number your browser carries to identify your session |
| **CloneID** | The hidden tag inside JSESSIONID saying which JVM owns you |
| **LTPA token** | Your "already verified" wristband — proves login without re-typing password |
| **M-to-M replication (DRS)** | Servers constantly photocopying each other's files between memories |
| **Replicas = 2** | Every session kept in 3 places: 1 primary + 2 backups |
| **POST** | A request that changes something (submit payment) — never auto-retried |
| **GET** | A request that just reads a page — safe to auto-retry |
| **503** | HTTP code meaning "server couldn't process your request" |
| **ConnectTimeout (5s)** | How long the plugin waits before declaring a server dead |
| **RetryInterval (60s)** | How often the plugin checks if a dead server came back |
| **Failover** | A dead server's users being seamlessly moved to a healthy one |
| **Session promotion** | A backup copy being upgraded to the official primary on the new JVM |
| **plugin-cfg.xml** | The receptionist's rulebook — routing table for IHS |

---
