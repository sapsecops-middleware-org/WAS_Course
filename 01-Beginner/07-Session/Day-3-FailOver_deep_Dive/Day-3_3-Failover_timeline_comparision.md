# DAY 54 — FAILOVER TIMELINE: M-to-M vs DB Persistence

Explained From Absolute Zero — Assume You Know Nothing. Sit with me. I'll explain like you've never touched a server in your life. No jargon without explanation. Step by step.

---

## PART 1 — THE BASICS (Build the Foundation First)

### 1.1 What is a Server / JVM?
* A **server** is a computer program that answers requests from users.
* **JVM (Java Virtual Machine)** is the program that runs a Java application.
* Think of `JVM1` and `JVM2` as two workers sitting at two separate desks.
* Both workers can do the same job. That's why we keep two — if one worker faints, the other takes over. This is called **failover**.

### 1.2 What is RAM?
* **RAM** = the short-term memory of a computer.
* Like your own brain holding a phone number someone just told you.
* Super fast to read and write. ⚡
* But — the moment power goes off (or the program dies), RAM is completely wiped empty.
* **Analogy:** Writing on a whiteboard. Erase the board (or unplug it) → everything gone.

### 1.3 What is a Disk / Database?
* **Disk** = permanent storage. Like a notebook or a filing cabinet.
* Slower to read/write than RAM. 🐢
* But — power goes off? The notebook is still there. Nothing is lost.
* A **database** is a very organized filing cabinet that programs can read from and write to.

> [!TIP]
> **Remember this pair forever:**
> * **RAM = whiteboard.** Fast, but erased when power dies.
> * **Disk/DB = notebook.** Slower, but permanent.

### 1.4 What is a Session?
When Ravi logs into NetBanking, the bank's server creates a small "folder" just for him. This folder is called a **session**.

**What's inside Ravi's session folder?**
* Proof that he logged in (his **LTPA token** — think of it as a wristband given at a concert gate. As long as you wear it, you don't show your ticket again.)
* His half-filled form data (beneficiary name, amount).
* His session ID: `xT9k82mB:cloneABC1` (a unique name tag for his folder).

**Where does this folder live?** In the RAM of `JVM1` — on the whiteboard.

### 1.5 The Big Problem 💥
* Ravi's session lives ONLY on `JVM1`'s whiteboard.
* `JVM1` crashes → whiteboard erased → Ravi's session vanishes.
* Ravi gets kicked to the login page mid-transfer. He's furious.
* Banks can't afford this. So we need a backup copy of the session somewhere safer.
* There are two ways to make that backup: **Memory-to-Memory Replication** and **Database Persistence**.

---

## PART 2 — THE TWO BACKUP METHODS

### Method 1: M-to-M Replication (Memory-to-Memory)

#### The Analogy 📄
You write important notes on your whiteboard. Every 2 seconds, you photocopy your whiteboard and put the copy in your colleague's drawer. If your desk collapses, you walk to your colleague, open the drawer, and continue working.

#### How it works in WAS
* `JVM1` and `JVM2` are a pair (a "replication domain").
* Every 2 seconds, `JVM1` sends copies of its sessions to `JVM2`.
* `JVM2` stores them in its own RAM (its own whiteboard corner).
* The tool that does this copying is called **DRS (Data Replication Service)**. Built into WebSphere. You just switch it on.

#### Good and bad
* Backup is in RAM → reading it back is almost instant (microseconds).
* Only ~2 seconds of data can ever be lost.
* RAM on `JVM2` gets used up (cost).
* If both JVMs die (power cut, full restart) → both whiteboards are wiped. All backups gone.

---

### Method 2: DB Persistence

#### The Analogy 🏦
Instead of photocopying to a colleague's drawer, you periodically walk to the bank vault and file a paper copy of your notes. If your desk collapses AND your colleague's desk collapses, the vault copy still exists.

#### How it works in WAS
* `JVM1` writes session data into a database table on disk.
* When does it write? Depends on configuration:
  * End of each request (when Ravi's click finishes processing), or
  * At a scheduled time (e.g., nightly cleanup), or
  * Every time the session changes.
* When Ravi moves to `JVM2` after a crash, `JVM2` reads the session back from the database.

#### Good and bad
* Data is on disk → survives power cuts, reboots, even a full cluster restart.
* Reading from disk is slow (100–500 milliseconds vs microseconds).
* The database is now a dependency — if the DB itself is down, session recovery fails.
* Needs a DB server, network, connection pool = extra cost and setup.

---

## PART 3 — THE CRASH STORY, SECOND BY SECOND

### The Setup (Same story for both methods)
* Ravi is doing a money transfer. He's on `JVM1`.
* Session ID: `xT9k82mB:cloneABC1`
* **11:15:03 AM** → `JVM1` crashes 💥
* **11:15:09 AM** → Ravi clicks again (humans naturally wait a few seconds and retry).
* **The Plugin:** A small traffic policeman sitting in the web server in front of the JVMs. Its job: send each request to a JVM. If a JVM doesn't answer, it sends the request to the other one.

---

### ⚡ STORY A — Crash with M-to-M Replication

#### Before the crash — what was happening quietly in the background
* **11:14:57** → Ravi updates something. `JVM1`'s DRS copies it to `JVM2`'s RAM.
* **11:14:59** → Another update → copied again.
* **11:15:01** → Ravi types the beneficiary name → copied.
* **11:15:03** → Ravi clicks Submit → `JVM1` crashes mid-click 💥
* The next DRS copy was "about to happen" — it never fired.
* **Result:** The last 2 seconds of changes never reached `JVM2`. Lost.

#### After the crash — what Ravi experiences
| Time | What's happening (in plain English) |
|---|---|
| 11:15:03 | `JVM1` dies. Its door (port 9080) stops answering. |
| 11:15:03 | The Plugin sends Ravi's click to `JVM1` anyway (doesn't know yet it's dead). |
| 11:15:03–08 | `JVM1` says nothing. Plugin stands there waiting... like knocking on a door no one answers. |
| 11:15:08 | 5 seconds pass → Plugin gives up ("ConnectTimeout"), marks `JVM1` as DEAD, pastes a "503 error" on Ravi's browser. |
| 11:15:09 | Ravi clicks again. |
| 11:15:09 | Plugin now routes him to `JVM2`. |
| 11:15:09 | `JVM2` sees his session ID, opens its own drawer (RAM) → finds the backup copy. |
| 11:15:09 | Loads it. Takes microseconds (RAM is that fast). |
| 11:15:09 | Ravi is back. Still logged in (LTPA wristband intact). Form data intact — minus the last 2 seconds. |

**The two numbers to remember:**
* Session recovery itself: **~5 milliseconds** (Just opening the drawer).
* Ravi's total wait: **~6 seconds** (Almost all of it was the Plugin's 5-second timeout — not the session lookup).

> [!NOTE]
> **Golden insight:** The user feels the timeout, not the replication. Replication speed almost doesn't matter to the user!

---

### 🐢 STORY B — Crash with DB Persistence

#### Before the crash — what was happening quietly in the background
* **11:10:00** → Ravi's previous click finished → session written to DB (that was the last vault trip).
* **11:10 → 11:15** → Ravi keeps typing and clicking. All those changes live only on `JVM1`'s whiteboard (RAM). Not in the vault yet.
* **11:15:03** → Ravi clicks Submit → `JVM1` crashes mid-request 💥
* The request never finished → its "end-of-request" DB write never happened.
* **Result:** 5 minutes of Ravi's work never made it to the vault. Lost.

#### After the crash — what Ravi experiences
| Time | What's happening (in plain English) |
|---|---|
| 11:15:03 | `JVM1` dies. |
| 11:15:08 | Plugin's 5-second timeout. Same as before. |
| 11:15:09 | Ravi retries → routed to `JVM2`. |
| 11:15:09 | `JVM2` checks its own RAM → NOTHING there! (DB method never copies to another JVM's RAM.) |
| 11:15:09 | `JVM2` walks to the vault: sends a question to the database — `SELECT * FROM SESSIONS WHERE SESSION_ID = 'xT9k82mB'` |
| 11:15:09+ | Database searches its disk and returns the paper copy. Takes 100–500 ms (disk is slow). |
| 11:15:09+ | `JVM2` "unwraps" the data back into a usable Java object (this unpacking is called deserialization). |
| 11:15:10 | Ravi is back — still logged in. BUT his session is the 11:10 version. His last 5 minutes of form-filling? Gone. He retypes it. |

**The two numbers to remember:**
* Session recovery: **~500–600 ms** (100× slower than M-to-M).
* Data lost: **up to 5 minutes** (everything since the last DB write).

---

## PART 4 — SIDE-BY-SIDE (The Interview Table)

| Question | M-to-M Replication | DB Persistence |
|---|---|---|
| **Where is the backup?** | Another JVM's RAM (drawer) | Database on disk (vault) |
| **How often saved?** | Every 2 seconds (automatic whisper) | End of request / scheduled |
| **How fast to recover?** | Microseconds ⚡ | 100–500 ms 🐢 |
| **How much data lost on one JVM crash?** | Last ~2 seconds | Last request's changes (can be minutes) |
| **Survives ONE JVM crash?** | Yes | Yes |
| **Survives FULL cluster restart?** | No — all RAM wiped | Yes — DB on disk survives |
| **Survives server/OS reboot?** | No | Yes |
| **What does it cost?** | Extra RAM on every JVM | DB server + connections (more setup) |
| **Any new dependency?** | None | Yes — if DB is down, recovery fails |
| **Best for?** | Real-time payments (UPI, NEFT, RTGS) | DR, batch jobs, maintenance windows, audits |

**One-line memory trick:**
> M-to-M = fast but fragile. DB = slow but tough.

---

## PART 5 — THE HONEST TRUTH: What is ALWAYS Lost (Even With Perfect Setup)

Beginners think failover saves everything. Experts know better. When a JVM dies in the middle of processing a request, these are lost in BOTH methods:

1. **The request that was in progress ❌**
   * Ravi's Submit click was 80% processed when `JVM1` died.
   * That work-in-progress is gone. The new JVM starts that request from scratch.
2. **The response ❌**
   * `JVM1` was about to send the answer back. It never did.
   * Ravi just saw an error/timeout.
3. **Uncommitted database transactions ❌**
   * Suppose `JVM1` was mid-way writing the transfer into the NEFT table.
   * A proper transaction ends with a `COMMIT` (final seal).
   * Crash before `COMMIT` → the database automatically rolls back → the transfer never happened.
   * *(This is actually GOOD — a half-done money transfer would be a disaster!)*
4. **JVM-local memory ❌**
   * Static variables, application-level caches — these live on `JVM1`'s whiteboard only.
   * Neither method copies them.

**What IS saved (with proper config) ✅**
* The `HttpSession` (login state, cart, form data)
* The LTPA token → user stays logged in

> [!NOTE]
> **The expert one-liner:**
> "Replication saves the SESSION. Nothing saves the REQUEST that was in flight."

---

## PART 6 — WHY THE "FULL CLUSTER RESTART" DIFFERENCE MATTERS (Real Bank Story)

### The Situation
* ICICI plans maintenance at 2 AM Sunday — a WAS patch.
* All 4 JVMs must be restarted.
* 3,000 night-shift users are still logged in, doing batch uploads and transfers.

### Scenario A — Sessions configured with M-to-M
1. **2:00 AM** → Admin restarts `JVM1`.
2. `JVM2` (also being restarted!) had the backups in its RAM → erased too.
3. **Result at 2:05 AM** → all 3,000 users' sessions are gone. Everyone is dumped at the login page. Angry complaints, dashboards screaming, incident ticket raised.
4. Users must log in again. Anything half-filled is lost.

### Scenario B — Same restart, sessions configured with DB Persistence
1. **2:00 AM** → Admin restarts all JVMs.
2. Sessions are safely sitting in the database — untouched by the restart. 💾
3. **2:05 AM** → JVMs come back up.
4. **2:05:10 AM** → A user clicks something.
5. JVM checks its RAM: nothing. Walks to the vault: finds his session.
6. User is back where he was — logged in, mid-task, unaware anything happened.

> [!NOTE]
> This is the single biggest reason DB persistence exists. **M-to-M protects against ONE JVM dying. DB persistence protects against EVERYTHING dying.**

---

## PART 7 — WHERE YOU CLICK IN THE WAS ADMIN CONSOLE (For Reality)

I'm showing you this so the theory connects to the actual tool. Don't memorize paths — just see that these are two simple switches.

### For M-to-M Replication
* Admin Console → **Servers** → **Server Types** → **WebSphere application servers** → **JVM1**
* **Session Management** → **Distributed session management**
* Tick **"Distributed session management"**
* Under Memory-to-Memory replication, set:
  * **Topology:** "Both client and server" (this JVM both sends AND receives backups)
  * **Replication frequency:** e.g., 2 seconds
* Do the same for `JVM2`, in the same replication domain.

### For DB Persistence
* Same screen: **Session Management** → **Distributed session management**
* Tick **"Database persistence"** (it appears under "Use database")
* Give it a **DataSource** — basically the "address and key" to reach the database.
* Choose write frequency:
  * **End of request** → safest, writes after every click
  * **Manual/scheduled** → fastest, but riskiest (more data lost on crash)

```sql
SELECT * FROM SESSIONS WHERE SESSION_ID = 'xT9k82mB';
```

> [!WARNING]
> **Exam-style trap:** M-to-M and DB persistence are configured in the SAME screen but only one method can be active at a time. You pick one.

---

## PART 8 — HOW TO ANSWER IN THE INTERVIEW (Scripted)

If asked: *"What happens when a JVM fails mid-transaction — walk me through failover?"*

Say it in this order (interviewers love structure):

1. **"First, the Plugin notices."** — It waits for its 5-second `ConnectTimeout`, marks the dead JVM, and shows 503 to that one request.
2. **"On retry, it routes to the healthy JVM."** — The user's next click lands on `JVM2`.
3. **"Session recovery depends on the method."** — With M-to-M, the backup is already in `JVM2`'s RAM, recovery in ~5 ms. With DB persistence, `JVM2` queries the database, ~100–500 ms plus deserialization.
4. **"The user stays logged in."** — Because the LTPA token travels with the session.
5. **"But the in-flight request is lost in both methods."** — The dying request never completes. Only the session survives, never the request.
6. **"And if the whole cluster restarts, M-to-M loses everything, but DB persistence survives."**

That last two points are what separate a beginner from a senior. Most candidates stop at "session is recovered." You now know what is NOT recovered — that's the expert answer.

---

## PART 9 — MEMORY HOOKS (Learn These Five Lines)

* **RAM = whiteboard** (fast, erased on death). **Disk = vault** (slow, permanent).
* **M-to-M = photocopy to colleague's drawer.** Every 2 seconds. Microsecond recovery.
* **DB = file a paper copy in the vault.** End of request. Survives full restart.
* **Replication saves the SESSION.** Nothing saves the REQUEST that was in flight.
* **Fast but fragile vs slow but tough.**
---
# How to Verify Active Session Persistence Method in WebSphere Application Server

This guide explains how to identify whether an active WebSphere Application Server environment is configured for **Memory-to-Memory (M-to-M) replication**, **Database (DB) persistence**, or **Local sessions (None)** using the Integrated Solutions Console (Admin Console) and `wsadmin` scripting.

---

## Overview

WebSphere Application Server supports three distributed session persistence mechanisms:

| Method | Description | Configuration Indicator |
| :--- | :--- | :--- |
| **None** | Local session memory only; no replication across cluster members | Neither DRS nor DatabaseStore configured |
| **Memory-to-Memory** | Peer-to-peer or client/server session replication using Data Replication Service (DRS) | `DRSSettings` configured; replication domain linked |
| **Database** | Session offloading and state storage in an external relational database | `DatabaseStore` configured; target JDBC Data Source JNDI mapped |

---

## Method 1: Verification via WebSphere Admin Console

Follow the navigation path below to inspect the session management settings via the WebSphere administrative console:

1. Log in to the **WebSphere Admin Console**.
2. Navigate to:  
   **Servers** $\rightarrow$ **Server Types** $\rightarrow$ **WebSphere Application Servers** $\rightarrow$ **[your server]**
3. Under the **Container Services** section, expand or click **Session Management**.
4. Under **Additional Properties**, click **Distributed environment settings**.

### Evaluation Points

Inspect the active radio button selection:

* `None`: Local sessions only (no replication).
* `Memory-to-memory replication`: M-to-M mode is enabled.
* `Database`: Database persistence is enabled.

> [!NOTE]
> * If **Memory-to-memory replication** is selected, verify the **Replication domain** field to check which domain is linked.
> * If **Database** is selected, inspect the **Data source JNDI name** field to see the active persistence database.

---

## Method 2: Verification via wsadmin (Jython Script)

To audit multiple nodes and cluster members rapidly without using the GUI, run the following Jython script via `wsadmin`.

### Script: `check_session_persistence_method.py`

```python
# check_session_persistence_method.py
# Shows you M-to-M or DB for all servers in cluster

print("=== Session Persistence Method — All Cluster Members ===\n")

servers = AdminConfig.list('Server').splitlines()

for s in servers:
    if s.strip():
        name = AdminConfig.showAttribute(s, 'name')
        # Find SessionManager under this server
        sm = AdminConfig.list('SessionManager', s)
        if sm:
            # Check for DRS (M-to-M)
            drs = AdminConfig.list('DRSSettings', sm)
            # Check for DB persistence
            db  = AdminConfig.list('DatabaseStore', sm)

            if drs:
                domain = AdminConfig.showAttribute(drs, 'dataReplicationMode')
                print("Server: %-30s  Method: M-to-M  Mode: %s" % (name, domain))
            elif db:
                jndi = AdminConfig.showAttribute(db, 'dataSourceJNDIName')
                print("Server: %-30s  Method: DB       JNDI: %s" % (name, jndi))
            else:
                print("Server: %-30s  Method: LOCAL (no replication!)" % name)
```

### Execution Command

Execute the script from the `<WAS_HOME>/bin` directory:

```bash
./wsadmin.sh -lang jython -f check_session_persistence_method.py
```

> [!TIP]
> In secure WebSphere profiles, append `-user <admin_user> -password <admin_password>` to the execution command.

---

## Expected Output

### Case A: Memory-to-Memory (M-to-M) Active

```text
=== Session Persistence Method — All Cluster Members ===

Server: PaymentCluster_server1   Method: M-to-M  Mode: BOTH
Server: PaymentCluster_server2   Method: M-to-M  Mode: BOTH
Server: PaymentCluster_server3   Method: M-to-M  Mode: BOTH
Server: PaymentCluster_server4   Method: M-to-M  Mode: BOTH
```

### Case B: Database Persistence Active

```text
=== Session Persistence Method — All Cluster Members ===

Server: PaymentCluster_server1   Method: DB      JNDI: jdbc/SessionDB
Server: PaymentCluster_server2   Method: DB      JNDI: jdbc/SessionDB
Server: PaymentCluster_server3   Method: DB      JNDI: jdbc/SessionDB
Server: PaymentCluster_server4   Method: DB      JNDI: jdbc/SessionDB
```