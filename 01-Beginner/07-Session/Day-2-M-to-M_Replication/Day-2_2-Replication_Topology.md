# Replication Topology

### 2.1 Single-Row (Peer) Replication — "Everyone copies to everyone"
```
┌──────────────────────────────────────────────────────┐
│              PAYMENTCLUSTER (4 JVMs)                 │
│                                                      │
│   JVM1 ◄──────────────────────────► JVM2            │
│     │                                  │             │
│     │                                  │             │
│   JVM4 ◄──────────────────────────► JVM3            │
│                                                      │
│   All 4 JVMs replicate to ALL others                 │
│   1 session = 1 primary + backup copies everywhere   │
└──────────────────────────────────────────────────────┘
```

Every JVM replicates its sessions to **all** other JVMs in the cluster.

| Attribute | Detail |
|---|---|
| Topology | Full mesh — N JVMs, each replicates to N-1 peers |
| Write amplification | 1 write → N-1 copies |
| Redundancy | Highest — survives multiple simultaneous JVM failures |
| Best for | Small clusters (2–4 JVMs), small session data (< 5 KB) |
| Drawback | Traffic and memory grow linearly with cluster size (12 JVMs → 11 copies per write) |

**Use when:** maximum session availability outweighs network/memory cost.

> [!NOTE]
> One-line rule: **Single-Row = "Tell everyone." Safe, but expensive as you grow.**

### 2.2 Multi-Row (Partitioned) Replication — "Everyone copies to one partner"
```
┌──────────────────────────────────────────────────────┐
│              PAYMENTCLUSTER (4 JVMs)                 │
│                                                      │
│   ROW 1:  JVM1 ◄────────────────► JVM2              │
│           (JVM1 primary, JVM2 = backup)              │
│                                                      │
│   ROW 2:  JVM3 ◄────────────────► JVM4              │
│           (JVM3 primary, JVM4 = backup)              │
│                                                      │
│   JVM1 does NOT replicate to JVM3 or JVM4           │
└──────────────────────────────────────────────────────┘
```

J into pairs. Each JVM backs up sessions **only** to its designated partner.

| Attribute | Detail |
|---|---|
| Topology | Partitioned pairs (e.g., JVM1↔JVM2, JVM3↔JVM4) |
| Write amplification | 1 write → 1 copy |
| Redundancy | Single partner only — losing both pair members loses sessions |
| Best for | Large clusters (8+ JVMs), large session data (20 KB+) |
| Drawback | Paired failure = session loss (e.g., both JVMs in one rack lose power) |

> [!WARNING]
> Place each replication pair on **different racks / power feeds** to avoid correlated failure.

> [!NOTE]
> One-line rule: **Multi-Row = "Tell only your buddy." Cheap and scalable, but one point of failure per pair.**

### 2.3 Topology Decision Guide

| Question | Recommendation |
|---|---|
| Cluster ≤ 4 JVMs? | Single-Row |
| Cluster > 4 JVMs? | Multi-Row |
| Session < 5 KB? | Single-Row is fine |
| Session > 20 KB? | Multi-Row (saves memory) |
| Need ultra-high redundancy? | Single-Row |
| Need controlled/predictable traffic? | Multi-Row |

Real bank answer: "We use multi-row in production clusters > 6 JVMs and single-row in UAT/smaller clusters"


---
# How DRS Actually Works — The Deep Dive

Step-by-step mechanics of the IBM WebSphere Application Server **Data Replication Service (DRS)** and the complete lifecycle of memory-to-memory session replication — from a user click to seamless failover.

---

## 1. The Cast of Characters

Before following the replication story, understand each component's role:

| Name | What it is | Analogy |
|---|---|---|
| User's Session | A small memory box holding user data (login token, txn amount) | A folder on a clerk's desk |
| Primary JVM (JVM1) | The JVM where the session was born and is being used | The clerk holding the original folder |
| Backup JVM (JVM2) | The JVM holding a photocopy of the session | The clerk next door holding the photocopy |
| SessionManager | Part of WAS that manages all sessions in that JVM | The clerk's DIRTY flag | A marker: "This session changed, it needs re-copying" | A sticky note: "Updated — please photocopy again" |
| DRS | Data Replication Service — the courier that copies sessions between JVMs | The office boy running photocopies between desks |
| Web Server Plugin | Sits on the web server, routes each request to a JVM | The receptionist directing customers to clerks |
| JSESSIONID | The session's unique ID | File number on the folder |

> [!NOTE]
> **Two jobs, two tools:** DRS copies the data. The Web Server Plugin redirects the user. They are separate components with separate responsibilities — never confuse them.

---

## 2. The Replication Lifecycle (Step by Step)

Scenario: a user clicks **"Confirm NEFT"** for ₹2,00,000.

### Step 1 — Application Code Writes to the Session

```java
session.setAttribute("txnAmount", 200000);
```

- Developer's servlet code writes a value into the session object.
- A completely normal action — thousands happen per second.
- The session object sitting in JVM1's memory now contains `txnAmount = 200000`.

> [!TIP]
> Think of it as: the clerk writes a new line into the user's folder.

### Step 2 — WAS Marks the Session as DIRTY

- WAS detects that the session was just modified.
- It flags the session as **DIRTY**.

Why this is clever design:

- WAS does **not** copy sessions blindly all the time.
- Only sessions that changed get copied.
- If 10,000 users are logged in but only 400 clicked something this second — only those sessions get marked DIRTY and replicated [!TIP]
> **DIRTY** = "edited, needs a fresh photocopy." **Clean** = "unchanged, the old photocopy is still valid."

### Step 3 — DRS Picks Up the DIRTY Session

- DRS runs in the background inside every JVM, always watching.
- At the configured frequency/mode, DRS asks the SessionManager: *"Any dirty sessions? Hand them over."*
- The SessionManager hands over the dirty session.

Key tuning concepts embedded in this step:

- **When** replication happens (timing settings)
- **How** it happens (`MANUAL` vs `SET_AND_GET` vs `ALL_REQUEST`, etc. — replication mode)
- **Where** it goes (replica count, replication domain)

### Step 4 — DRS Serializes the Session ⚠️ (The #1 Production Killer)

**What serialization is:**

- A live Java object in memory is a complex structure — pointers, references, connected objects.
- Serialization = converting that live object into a **flat stream of bytes**.

> [!TIP]
> Analogy: a folded pop-up greeting card can't go through a fax machine. You flatten it, scan it into one flat image (bytes), send the image. The other side re-folds/rebuilds it. That's serialization → transfer → deserialization.

Why this step breaks deployments:

- To flatten the session, **every object inside it must be Serializable** (`java.io.Serializable`).
- `String` token, `Integer` amount → fine.
- A database `Connection` or file handle → **.
- Result: DRS **fails silently The page still works, no visible error.
- The backup JVM never receives the session.
- Weeks later: JVM1 crashes → sessions vanish → "sessions randomly disappearing" ticket. Very hard to debug.

> [!IMPORTANT]
> **Golden rule:** *Whatever goes into session must be Serializable.*
> **Bank rule of thumb:** keep only simple things in session — Strings, numbers, small DTOs. Never connections, never live objects.

### Step 5 — DRS Sends the Bytes Over the Network

- DRS opens a JVM-to-JVM network connection (internal,
- The byte stream ships to the designated backup JVM(s).

How many backups depend on topology:

| Topology | Copies per write |
|---|---|
| Single-Row (4 JVMs) | 3 copies (all peers) |
| Multi-Row | 1 copy (partner only) |

Traffic math — why topology matters:

```
2,000 active sessions × 20 KB × user clicks 5×/minute

Single-Row (4 JVMs):  2000 × 20 KB × 3 copies × 5/min ≈ ~600 KB/sec constant traffic
                       + 20 KB backup memory on every JVM
Multi-Row:            one-third of that
```

> [!NOTE]
> Multiply by a real bank's traffic volume and you see why partitioned (multi-row) replication exists.

### Step 6 — Backup JVM Receives, Deserializes, Stores

- JVM2 receives the byte stream.
- **Deserialization** rebuilds the bytes into a live Java object in JVM2's own memory.
- JVM2's SessionManager stores it, tagged with the same `JSESSIONID`.
- JVM2 does **not** serve it — it only holds it.

> [!TIP]
> The backup is a **spare tyre, not a second primary** — it holds the copy until failover.

- The DIRTY flag on JVM1 is cleared. Copy done — until the next click.

### Step 7 — JVM1 Crashes; Plugin Detects and Reroutes

- JVM1 dies: process killed, OOM, hung, hardware failure — the cause doesn't matter.
- The user's next request arrives at the web server plugin.
- The plugin continuously checks JVM health (heartbeat / timeouts).
- Plugin sees JVM1 is dead → consults its routing table → knows JVM2 is JVM1's backup partner → routes the request to JVM plugin forwards the user's `JSESSIONID` (from the cookie) along.

> [!TIP]
> Analogy: the receptionist sees clerk #1's desk is empty, remembers clerk #2 holds his photocopies, and sends the customer there.

> [!IMPORTANT]
> The plugin does **failover routing**. DRS does **data copying**. Two different components,

### Step 8 — JVM2 Promotes the Backup Session

- JVM2 receives the request with `JSESSIONID`.
- Its SessionManager: *"I don't have this as a primary… wait — I have a replicated copy!"*
- It **promotes** the backup copy: it becomes the active/primary session on JVM2.
- JVM2 now behaves as if the session was always born there. Per configuration, it may replicate a new backup elsewhere.

### Step 9 — The User Continues, Unaware

- The page loads. Login valid. `txnAmount = 200000` is present.
- The NEFT transaction completes.
- The user never knew a JVM died. **Zero disruption.**

> [!NOTE]
> The best failover is the one the customer never notices. In banking, that's not "nice to have" — it's a regulatory and revenue requirement.

---

## 3. The Whole Flow in One Diagram

```text
User clicks "Confirm NEFT"
        │
        ▼
[1] session.setAttribute(...)  ── data written in JVM1 memory
        │
        ▼
[2] Session marked DIRTY       ── "needs fresh copy"
        │
        ▼
[3] DRS picks it up            ── courier takes the folder
        │
        ▼
[4] SERIALIZATION              ── flatten object → bytes   ⚠ (failure point!)
        │
        ▼
[5] Bytes sent over network    ── to backup JVM(s)
        │
        ▼
[6] JVM2 deserializes + stores ── bytes → live object, held as backup
        │
   ...later...
        │
   JVM1 CRASHES ☠
        │
        ▼
[7] Plugin detects, routes to JVM2
        │
        ▼
[8] JVM2 finds JSESSIONID → promotes backup to primary
        │
        ▼
[9] User continues the NEFT — never knew anything happened ✅
```

---

## 4. Memory Tricks & Key Takeaways

- **The replication chain:** Write → Dirty → DRS → Serialize → Send → Store → Failover → Serve.
- **Two jobs, two tools:**
  - **DRS** = copies the data
  - **Plugin** = redirects the user
  - Never confuse them in interviews.
- **"Serialization is the silent killer"** — a non-serializable session object works fine until a crash, then sessions vanish.
- **Backup is a spare tyre, not a second car** — it holds the copy; it doesn't serve it until failover.

| Component | Responsibility |
|---|---|
| Application code | Writes session data (`setAttribute`) |
| SessionManager | Tracks sessions, marks DIRTY, hands to DRS |
| DRS | Serialize → transmit → replicate |
| Web Server Plugin | Health checks + failover routing |
| Backup JVM | Stores copy, promotes on failover |

---

## 4. The Serialization Requirement

Everything placed into an HTTP session **must implement `java.io.Serializable`**.

### Failure Mode

- A developer stores a non-serializable object (e.g., a `java.sql.Connection`, file handle, or stream).
- DRS **fails silently** — no loud error is thrown, and the page continues to work on the primary.
- The problem surfaces only when a JVM crashes and sessions fail to recover → a production incident.

> [!IMPORTANT]
> **Golden rule:** *If it goes in session, it must be Serializable.*

### Correct Pattern

```java
public class TransactionData implements java.io.Serializable {
    private static final long serialVersionUID = 1L;
    private String accountNumber;
    private double txnAmount;
    // getters/setters
}
```

### Anti-Pattern

``` NOT DO THIS
session.setAttribute("conn", dataSource.getConnection()); // not Serializable — silent DRS failure
```

> [!TIP]
> Proactively scan session attributes during development/testing rather than discovering gaps during a failover event.

---

## 5. Quick Revision Card

- **M-to-M** = copying sessions across JVMs so a crash doesn't lose users.
- **Single-Row:** everyone → everyone. Small clusters, small sessions, max redundancy.
- **Multi-Row:** paired JVMs. Big clusters, fat sessions, controlled traffic. Risk: pair dies together.
- **DRS flow:** set attribute → dirty flag → serialize → send → deserialize → store → plugin fails over.
- **Serialization is mandatory.** Non-serializable session objects = silent breakage = production incident.