# WebSphere Session Replication — Backups, Failure Scenarios, and Memory Cost

## 2. The Cast of Characters

| Term | Simple Meaning |
|---|---|
| JVM / server | One running WebSphere instance (`server1`...`server4`) |
| Cluster | A group of JVMs doing the same work (`PaymentCluster`) |
| Primary session | The original copy of your session (on the JVM serving you) |
| Backup session | A spare copy stored on another JVM |
| DRS | Data Replication Service — WebSphere's built-in "photocopy machine" that copies sessions between JVMs |
| `replicaCount` | How many spare copies? `0` = none, `1` = one backup, `2` = two backups |
| Plugin | The web server component that decides which JVM gets each request |
| Mode: `BOTH` | Replication happens at end of each request AND at session updates — safest timing |

---

## 3. How the Backup JVM Is Chosen

> [!NOTE]
> You **cannot** hand-pick "server1 backs up to server2." WebSphere does it automatically.

### Assignment Rules

1. JVMs join the replication domain in **startup order**.
2. DRS lines them up in a **ring** (round-robin).
3. Each JVM's backup = the **next JVM in the ring**.

### Our Ring

```text
JVM1 → JVM2 → JVM3 → JVM4 → back to JVM1
```

### The Backup Map (memorize this!)

| Primary (serves you) | Backup copy stored on |
|---|---|
| JVM1 | JVM2 |
| JVM2 | JVM3 |
| JVM3 | JVM4 |
| JVM4 | JVM1 |

> [!TIP]
> It's a circle. Everyone backs up to their neighbor, and the last one wraps around to the first. Picture it as **four people in a circle, each holding a photocopy of their neighbor's notes.**

---

## 4. Scenario A: One JVM Dies (the Happy Path)

`server1` crashes. **15,000 users** were on it.

1. Plugin notices `server1` is dead.
2. Routes those users to other JVMs — mostly `server2`.
3. `server2` already has a backup copy of all `server1` sessions.
4. `server2` "wakes up" those backups and serves the users.

✅ **Result: zero session loss. Nobody re-logs in.**

> [!IMPORTANT]
> **Why it works:** backup JVM ≠ primary JVM. A single JVM crash can never destroy both copies.

---

## 5. Scenario B: Two JVMs Die — The Co-Location Trap

The subtle part: in real banks, two JVMs often live on the **same physical server** (say Server A hosts JVM1 + JVM2).

**Server A dies → JVM1 AND JVM2 die together.**

Check the map:

| Whose sessions? | Backup was on... | Still alive? |
|---|---|---|
| JVM1's users | JVM2 ❌ (also dead) | **LOST** |
| JVM2's users | JVM3 ✅ | Fine ✅ |

### Result

- 15,000 JVM1 users → **re-login** ❌
- 15,000 JVM2 users → fine ✅

> [!WARNING]
> With `replicaCount = 1`, if a JVM dies **together with its backup JVM**, those sessions are gone. This is the **co-location risk**.

---

## 6. The Fix: `replicaCount = 2`

Store **two backups** of every session — on the next JVM and the one after:

| Primary | Backup 1 | Backup 2 |
|---|---|---|
| JVM1 | JVM2 | JVM3 |
| JVM2 | JVM3 | JVM4 |
| JVM3 | JVM4 | JVM1 |
| JVM4 | JVM1 | JVM2 |

### Replay Scenario B (JVM1 + JVM2 die)

- JVM1's backup on JVM2 → gone ❌
- JVM1's backup on JVM3 → **still alive** ✅
- JVM3 serves JVM1's users.

✅ **Zero session loss even if a whole physical server dies.**

### When to Use `replicaCount = 2`

Peak-risk periods:

- **Diwali** (heavy transaction load in India)
- **Month-end salary days** (1st–3rd, payroll loads)
- Any event where downtime is unacceptable

---

## 7. The Memory Bill (Quote This in Capacity Planning)

**Baseline:** 50,000 users × 5 KB session = **250 MB** total session data.

Every backup is a full extra copy:

| `replicaCount` | Copies per session | Total memory |
|---|---|---|
| 0 | 1 (primary only) | 250 MB |
| 1 | 2 | 500 MB |
| 2 | 3 | 750 MB |

Spread evenly over 4 JVMs:

- `replicaCount = 1` → **125 MB per JVM** for sessions
- `replicaCount = 2` → **187 MB per JVM** for sessions

> [!IMPORTANT]
> Sessions are only **PART** of your heap. Each JVM also holds application memory, caches, and DRS overhead.
>
> - Typical bank JVM heap: `-Xmx2048m` to `-Xmx4096m`
> - Never sized for sessions alone.

---

## 8. One-Line Exam Cheat Sheet

- **Primary** = where the user is being served.
- **Backup** = next JVM(s) in the startup-order ring — **automatic, not manual**.
- `replicaCount = 1` → survives one JVM crash, **not** a "primary + its backup" crash.
- `replicaCount = 2` → survives a full server crash (co-location safe). Costs **3x memory**.
- **Memory formula:** `(replicaCount + 1) × total session size`.
- Mode `BOTH` = replicate at end-of-request AND at updates — safest.
- Always check your `-Xmx` covers **app + sessions + overhead**.

---

## 9. Whiteboard Story for Interviews

Say it like this:

> "We had 4 JVMs in a ring. DRS assigns backups automatically — each JVM backs up to its neighbor. With `replicaCount = 1`, one JVM crash loses nothing, but if two co-located JVMs die — a primary and its backup — those users re-login. So during Diwali we raised `replicaCount` to 2, accepting 3x session memory, so any single server failure still kept every session alive. Total session data was 250 MB, so `replicaCount = 2` meant about 187 MB per JVM, and we sized heap at 4 GB to cover app + sessions + overhead."

> [!TIP]
> That one paragraph shows you understand **mechanism, failure modes, and cost** — exactly what interviewers want.
