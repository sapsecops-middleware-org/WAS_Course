# Memory-to-Memory Session Replication in IBM WebSphere Application Server

A complete, beginner-friendly reference covering sessions, cookies, DRS, serialization, failover, tuning, and troubleshooting.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Core Concepts](#2-core-concepts)
3. [The Session Lifecycle](#3-the-session-lifecycle)
4. [The Cookie (JSESSIONID)](#4-the-cookie-jsessionid)
5. [The Failure Scenario](#5-the-failure-scenario)
6. [DRS — Data Replication Service](#6-drs--data-replication-service)
7. [Serialization — The Heart of the Topic](#7-serialization--the-heart-of-the-topic)
8. [End-to-End Happy Path](#8-end-to-end-happy-path)
9. [Primary and Backup JVMs](#9-primary-and-backup-jvms)
10. [Tunables](#10-tunables)
11. [Facts and Gotchas](#11-facts-and-gotchas)
12. [Configuration (High Level)](#12-configuration-high-level)
13. [Troubleshooting Checklist](#13-troubleshooting-checklist)
14. [One-Paragraph Summary](#14-one-paragraph-summary)

---

## 1. Overview

**Memory-to-Memory Session Replication** is a WebSphere feature in which each JVM continuously makes backup copies of HTTP sessions into the **memory (RAM)** of another JVM in the cluster — no database, no disk.

> [!TIP]
> Mental model: **memory of JVM1 → memory of JVM2. RAM to RAM.**

### Why it exists

If the JVM holding a user's session crashes, the user would otherwise be logged out mid-transaction. With replication enabled, the backup JVM already holds a copy, and the user continues seamlessly.

---

## 2. Core Concepts

### What is a JVM?

A **JVM (Java Virtual Machine)** is a single working process on a server that runs the application. One physical server typically runs multiple JVMs (JVM1, JVM2, JVM3) for redundancy — like multiple bank counters: if one closes, customers go to the next.

### What is a Session?

The web is stateless — every HTTP request arrives "anonymous." A **session** is the server-side memory that keeps a logged-in user's identity and state.

| Mall Analogy | WebSphere Equivalent |
|---|---|
| Locker | Session |
| Bag + items inside | User data (userId, login time, etc.) |
| Key with locker number | Cookie (`JSESSIONID`) |
| Locker room | JVM's memory |

### Example session contents

```text
┌─────────────────────────────────┐
│ Session ID: abc123              │
│ userId    = "RAVI001"           │
│ userRole  = "SAVINGS"           │
│ loginTime = 11:05 AM            │
│ lastPage  = "/dashboard"        │
└─────────────────────────────────┘
```

```java
session.setAttribute("userId", "RAVI001");
```

> [!IMPORTANT]
> The session lives in the memory of **one specific JVM**. Other JVMs know nothing about it until replication is configured.

---

## 3. The Session Lifecycle

### Request flow (first visit)

1. User hits `www.digibank.in/netbanking`
2. Request reaches the **Web Server (IHS)** — the receptionist at the door
3. The **Plugin** inside IHS routes the request to a JVM
   - No cookie yet → **Round Robin** selection (1st visitor → JVM1, 2nd → JVM2, ...)
4. The chosen JVM creates a session and returns a cookie
5. Browser stores the cookie and sends it with every subsequent request
6. The owning JVM matches the cookie to its in-memory session

---

## 4. The Cookie (JSESSIONID)

```http
Set-Cookie: JSESSIONID=abc123:JVM1
```

> [!TIP]
> The `:JVM1` suffix identifies **which JVM owns the session**. In debugging, this instantly tells you where the user's session lives.

---

## 5. The Failure Scenario

Without replication:

1. Ravi logs in → session lives in JVM1's memory
2. Ravi performs transactions
3. **JVM1 crashes**
4. Plugin detects JVM1 is down → routes Ravi to JVM2
5. JVM2: *"What cookie? `abc123`? I've never seen this session."*
6. Ravi is forced to log in again → complaint raised

### The fix

If JVM2 already holds a **backup copy** of the session, Ravi lands on JVM2 and continues as if nothing happened.

---

## 6. DRS — Data Replication Service

**DRS (Data Replication Service)** is the built-in WebSphere component that performs the copying. Think of it as a photocopy operator inside every JVM.

### The DIRTY flag

Copying every session on every wake-up would flood the network. Instead:

- Every `session.setAttribute(...)` call places a **DIRTY** sticker on the session ("I was changed — copy me")
- DRS copies only DIRTY sessions
- After a successful copy, the sticker is removed
- Unchanged sessions are skipped

```text
┌─────────────────────────────────┐
│ Session: abc123    🔴 DIRTY     │
│ userId = "RAVI001"              │
└─────────────────────────────────┘
```

> [!TIP]
> Analogy: a dishwasher only washes dirty plates. Clean plates on the shelf are ignored.

### Transport

DRS sends session bytes to the backup JVM over **TCP port 7272**.

---

## 7. Serialization — The Heart of the Topic

### The problem

A Java object lives in memory as a complex structure connected by references. A network cable carries only flat bytes. You cannot "mail a furnished house" — only a **photograph** of it fits in an envelope.

- **Serialize:** Java Object → byte stream (on the sending JVM)
- **Deserialize:** byte stream → Java Object (rebuilt on the backup JVM)

### The golden rule

> [!IMPORTANT]
> **Every single object stored in a session MUST be `Serializable`.**

```java
public class AccountSummary implements Serializable {  // ← the stamp
    private String accountNo;
    private double balance;
}
```

### Safe by default

| Type | Serializable? |
|---|---|
| `String` | ✅ Yes |
| `Integer`, `Double`, other numbers | ✅ Yes |
| `Date` | ✅ Yes |
| `ArrayList`, `HashMap`, collections | ✅ Yes |
| Custom classes without the stamp | ❌ **No** |
| `Connection` (live DB pipe) | ❌ **Never** |

### The silent killer

```java
public class AccountSummary {           // ← NO "implements Serializable"
    private String accountNo;
    private double balance;
    private Connection dbConn;          // ← non-serializable even if stamped
}
```

When DRS hits this object during serialization:

```text
java.io.NotSerializableException: com.digibank.AccountSummary
```

### Why it is called a *silent* failure

- The session is **NOT copied** — no backup exists
- The user sees **no error** — the app works perfectly on the owning JVM
- No popup, no red screen, no dashboard alert
- The **only clue** is a line in `SystemOut.log`
- Days or weeks later, the JVM crashes → users are thrown out

> [!WARNING]
> **Professional habit:** regularly scan `SystemOut.log` on every JVM for `NotSerializableException`. Fix the missing stamp **before** the crash, not after.

> [!TIP]
> Memory hook: *A photocopier that silently refuses to copy one page. Nobody notices — until the original burns.*

---

## 8. End-to-End Happy Path

| Scene | What happens |
|---|---|
| 1. Birth 🍼 | Ravi logs in on JVM1 → session `abc123` created in JVM1's memory |
| 2. Change 📝 | `setAttribute()` called → DIRTY sticker applied |
| 3. DRS wakes ⏰ | Every 2 seconds DRS checks → finds DIRTY `abc123` |
| 4. Photograph 📷 | Session serialized → flat bytes (~2 KB) |
| 5. Delivery 🚚 | Bytes sent over TCP port 7272 to backup JVM |
| 6. Rebuild 🏗️ | Backup JVM deserializes bytes → stores copy in its memory |
| 7. Receipt 📩 | Backup sends ACK → JVM1 removes DIRTY sticker ✅ |
| 8. Crash 💥 | JVM1 dies mid-transaction |
| 9. Rescue 🛟 | Plugin detects no heartbeat on JVM1 → reroutes to JVM2 |
| 10. Welcome 🎉 | JVM2 sees cookie `abc123:JVM1` → finds backup copy → state intact |
| 11. Continue ✅ | Ravi proceeds without re-login. He never knows a server died |

> [!NOTE]
> Design goal: **the user should never feel a failure.**

---

## 9. Primary and Backup JVMs

- Each JVM has **one primary** and **one backup** partner.
- The live session stays on the JVM where it was born; the copy goes to the partner.

### Ring layout (3-JVM cluster)

```text
JVM1 ⇄ JVM2
JVM2 ⇄ JVM3
JVM3 ⇄ JVM1
```

Each JVM pairs with a different partner, so if one machine dies entirely, no session loses both copies.

> [!NOTE]
> "Preference" settings (both preferred / only preferred / not preferred) control whether a session returns to its original JVM after failover. **Leave defaults unless you have a specific reason.**

---

## 10. Tunables

### Replication interval (default: 2 seconds)

| Interval | Pros | Cons |
|---|---|---|
| 1 sec | Almost zero data loss | High network traffic |
| 2 sec (default) | Good balance | Up to 2s of changes could be lost |
| 10 sec | Very light network | Larger risk window |

**Trade-off:** Faster = safer but heavier.

### Replication type

| Type | Use |
|---|---|
| `SESSION` (default) | Standard HTTP session replication |
| `CATALOG` / `ALL` | Advanced caching (Dynacache) — not needed for sessions |

### Backup size / discard

Limits how many backup sessions a JVM keeps. Usually left at defaults.

---

## 11. Facts and Gotchas

1. **Everything is in memory.** If primary AND backup both crash simultaneously, sessions are gone forever. No disk copy exists. For extreme criticality, use **Database Session Persistence** instead.
2. **Network traffic is real.** Every dirty session = bytes on the wire every interval. Keep sessions **small**.
3. **Big objects hurt twice.** A 5 MB object costs 5 MB of network transfer per change **and** 5 MB of extra RAM on the backup JVM. Store IDs and small flags; fetch heavy data fresh from the database.
4. **Session timeout still applies.** Backup sessions expire like primaries (default: 30 minutes of inactivity).

---

## 12. Configuration (High Level)

1. Admin Console → **Servers → Server Types → WebSphere Application Servers**
2. Click a JVM (e.g., JVM1)
3. **Session Management → Distributed Environment Settings**
4. Select **Memory-to-Memory Replication**
5. Set replication type = `SESSION`
6. Set the interval (e.g., 2 seconds)
7. **Save → Synchronize → Restart the JVMs**

> [!NOTE]
> The replication domain (the "group" of JVMs sharing sessions) is usually created automatically when the cluster is created.

---

## 13. Troubleshooting Checklist

Symptom: *"Users are getting logged out randomly."*

| # | Check | How |
|---|---|---|
| 1 | Is replication enabled? | Admin Console → Session Management → Distributed settings |
| 2 | `NotSerializableException` present? | `grep SystemOut.log` on **all** JVMs |
| 3 | Is DRS running? | Look for DRS / `repOut.log` entries at startup |
| 4 | Port 7272 open between JVMs? | `telnet <jvm2-host> 7272` |
| 5 | Session size reasonable? | Huge objects may fail or time out mid-copy |
| 6 | Inspect the cookie | `:JVM1` suffix → identifies the owning JVM |

---

## 14. One-Paragraph Summary

Every user gets a "locker" (session) in one JVM's memory; a key (cookie) identifies which locker is theirs. A photocopy machine inside each JVM (**DRS**) wakes up every 2 seconds, finds recently changed lockers (**DIRTY** flag), takes their "photograph" (**serialization**), and mails it to a partner JVM's memory over **port 7272**. If the original JVM dies, the user lands on the backup JVM, which already holds the photograph — rebuilds the locker — and the user never notices. The one thing that silently breaks everything: an object in the session that can't be photographed (**not Serializable**). It fails quietly in a log file, and the bomb ticks until a crash.

---
# WebSphere DRS — Session Serialization, Versioning, and Replication Boundaries

## Part 3: Serialization — The Developer vs Admin Problem

Session replication only works if every object stored in the `HttpSession` is serializable. This is a **shared responsibility** between developers and administrators.

### Responsibility Matrix

| Responsibility | Owner | Detail |
|---|---|---|
| Make every session object implement `java.io.Serializable` | **Developer** | Coding standard; enforced at design/code review |
| Configure DRS correctly | **Admin** | Memory-to-Memory replication settings |
| Monitor logs for `NotSerializableException` | **Admin** | Continuous monitoring in production |
| Raise defects when serialization failures appear | **Admin** | Alert the developer team immediately |

> [!IMPORTANT]
> If a session attribute is not serializable, **memory-to-memory (M-to-M) replication silently fails**. The app keeps working on the primary JVM — the failure is only visible in logs.

### The Production Trap

The failure follows a predictable, dangerous pattern:

1. Sessions work fine on a single JVM.
2. UAT passes — UAT runs only **1 JVM**, so no replication is ever exercised.
3. Production runs **4 JVMs** with M-to-M replication configured.
4. The app *appears* to work — replication is failing silently in the background.
5. A JVM dies → sessions are lost → users are logged out mid-transaction.
6. **The admin gets blamed. But the bug is in the code.**

### How to Find `NotSerializableException` in Logs

Search all JVM logs for serialization failures:

```bash
grep -r "NotSerializableException" \
    /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/
```

Search for DRS / Data Replication Service errors:

```bash
grep -r "DRS\|DataReplication" \
    /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/ \
    | grep -i "error\|fail\|exception"
```

> [!TIP]
> If you find `NotSerializableException`, raise a defect with the development team **immediately** and document it with a timestamp. This protects you as the admin.

---

## Part 4: Session Versioning — How DRS Handles Multiple Updates

DRS does **not** replicate on every `setAttribute` call. It replicates on a timer.

### Scenario: Ravi clicks 5 times in 2 seconds

```
T=0.0s  Ravi clicks → session.setAttribute("page", "/dashboard")
T=0.5s  Ravi clicks → session.setAttribute("page", "/transfer")
T=1.0s  Ravi clicks → session.setAttribute("amount", 50000)
T=1.5s  Ravi clicks → session.setAttribute("page", "/confirm")
T=2.0s  DRS TIMER FIRES
```

### Behavior: LAST-WRITE-WINS

- DRS does **not** send 4 separate copies.
- At T=2.0s, DRS takes the **current state** of the session and sends **one snapshot** to JVM2.
- The backup JVM receives only the **most recent** state — intermediate states are skipped.

### Implications

| Aspect | Detail |
|---|---|
| Efficiency | One network transfer per timer interval, not one per click |
| Risk | If JVM1 crashes between T=0 and T=2, the last 2 seconds of changes are lost |
| Acceptable for | Browsing / navigation flows |
| NOT acceptable for | Payment confirmation — the app **must trigger immediate replication** at that point |

> [!NOTE]
> For critical operations (payments, transfers), application code should force immediate replication rather than relying on the DRS timer.

---

## Part 5: What DRS Does NOT Copy

DRS replicates **sessions only** — nothing else on the JVM.

### ✅ What DRS Copies

- `HttpSession` attributes — everything set via `session.setAttribute(...)`
- Session metadata — creation time, last access time, timeout value

### ❌ What DRS Does NOT Copy

- Application code (WAR files, EJBs)
- Database connections
- JVM heap data outside of sessions
- **Static variables** in application code
- Servlet context attributes (`application.setAttribute(...)`)
- Thread-local variables
- File handles or open streams

### The Developer Trap: Static Variables

``` Bad developer code
public class PaymentService {
    // Static variable — shared across all requests on ONE JVM
    // NOT copied by DRS
    private static Map<String, PaymentStatus> pendingPayments = new HashMap();
}

// Session only stores the payment ID
session.setAttribute("paymentId", "PAY001");

// But the actual payment data lives in the static variable.
// When JVM1 dies:
//   → static variable is gone
//   → JVM2 has no payment data
//   → Session survived. Data did not.
```

> [!WARNING]
> This is a very common architectural mistake in banking applications. The session replicates, giving a false sense of safety — but the real data was never in the session.

### Admin Action Item

Educate developers during **design review**:

- All business-critical state must live in the `HttpSession` (as serializable objects), not in static variables or JVM-local structures.
- Anything outside the session is **single-JVM data** and will be lost on failover.

---
# WebSphere DRS — Real Banking Scenario: Silent Serialization Failure in Corporate Net Banking

## Scenario Overview

| Item | Detail |
|---|---|
| Bank | SBI-style PSU Bank |
| Application | Corporate Net Banking |
| Cluster | `CorporateCluster` — 4 JVMs |
| Replication | Memory-to-Memory (M-to-M) |
| `replicaCount` | 1 |
| Replication mode | `BOTH` |

### Problem Reported (Monday Morning)

> "Corporate customers are getting logged out randomly.
> It's happening even when no JVM restarts occurred."

---

## Admin Investigation — Step by Step

### Step 1: Check DRS MBeans

All 4 MBeans present. DRS running on all JVMs.

| Check | Result |
|---|---|
| DRS MBeans on all 4 JVMs | ✅ Present |
| DRS runtime state | ✅ Running |

> [!NOTE]
> Infrastructure is healthy — so the problem is **not** in DRS configuration. Look at the application.

### Step 2: Check Replication Failures in Logs

```bash
grep "NotSerializableException" SystemOut.log
```

Found:

```text
[ERROR] NotSerializableException: com.sbi.corporate.ReportGenerator
[ERROR] Cause: java.io.NotSerializableException:
        org.apache.poi.xssf.usermodel.XSSFWorkbook
```

### Step 3: Root Cause Identified

- A developer stored an **Apache POI Excel Workbook** object (`XSSFWorkbook`) directly in the session.
- `XSSFWorkbook` is **NOT Serializable**.
- DRS **fails to replicate** any session that contains this object.
- These users lose their sessions whenever their primary JVM is restarted — for example, during the **rolling patch restart every Sunday night**.
- Result: corporate users randomly logged out, with no visible app error.

### Step 4: The Fix

**Admin action — raise the defect:**

> Defect: *"XSSFWorkbook stored in session breaks M-to-M replication."*

**Developer fix — never store the workbook, store a pointer:**

```java
// BEFORE (broken) — non-serializable object in session
session.setAttribute("report", new XSSFWorkbook());

// AFTER (fixed) — generate file, store only the PATH
XSSFWorkbook workbook = buildReportWorkbook();      // build in memory
Path path = Files.createTempFile("report-", ".xlsx");
try (FileOutputStream out = new FileOutputStream(path.toFile())) {
    workbook.write(out);
}
session.setAttribute("reportPath", path.toString()); // String → Serializable ✅
```

**Result after deployment:**

- `NotSerializableException` gone from logs ✅
- Corporate users no longer losing sessions ✅

---

## Lessons Learned

| Lesson | Owner |
|---|---|
| DRS **silently fails** when session objects are not `Serializable` | Developer awareness |
| Monitor logs and catch `NotSerializableException` early | Admin |
| Store lightweight, serializable references (file paths, IDs) — not heavy objects | Developer |

> [!IMPORTANT]
> Without the `grep "NotSerializableException"` command, this bug runs **silently for months**. Proactive log monitoring is the admin's primary defense against invisible replication failures.
