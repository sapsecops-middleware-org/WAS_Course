# WebSphere Application Server (WAS) — Transaction Log Deep Dive

A production-focused reference for understanding the WAS transaction log (tranlog): its structure, contents, recovery behavior, and protection rules.

> [!IMPORTANT]
> This is the part most admins skip — and exactly why they get paged at 2 AM when recovery fails. The tranlog is the single most critical file in WAS transaction recovery.

---

## 1. Where the Log Lives

Default location:

```text
/opt/IBM/WebSphere/AppServer/profiles/AppSrv01/tranlog/
```

Organized per server:

```text
tranlog/
  └── server1/
        ├── transaction.log          ← Active journal (being written RIGHT NOW)
        ├── transaction.log.1        ← Previous segment
        └── transaction.log.2        ← Older segment
```

### Key Facts

- **Binary files.** You cannot `cat`, `vi`, or open them in a text editor. You will see garbage characters — this is normal. Don't panic.
- **Rotated:** active → `.1` → `.2`. Older segments are eventually reused.
- **One log per server** (`server1`, `server2`, ...). They are never shared.
- **Small by default** (a few MB). It only holds in-flight and unresolved transactions, not the day's full history.

> [!TIP]
> Analogy: It's not a full accounting book. It's a sticky note pad — only what's currently unfinished. Once a transaction completes, its entry is cleaned up.

### Changing the Location (Admin Console)

```text
Servers → Server Types → WebSphere application servers → server1
  → Container Services → Transaction Service
     → "Transaction log directory"
```

In production, point this at safer storage (SAN / redundant disk). See Section 5.

---

## 2. What Actually Gets Written (The 5 Entries)

For every XA transaction, WAS writes diary entries in this order:

### Entry 1 — Transaction Started

```text
TXN-ID: XA-20240909-001
Started: 09:00:01.001
Resources enrolled: Oracle, DB2
```

> "I've started a transaction. Two databases are involved."

### Entry 2 — Prepare Sent (Phase 1 begins)

```text
TXN-ID: XA-20240909-001
Oracle: PREPARE sent
DB2:    PREPARE sent
```

> "I asked both databases to get ready."

### Entry 3 — Prepare Responses Received

```text
TXN-ID: XA-20240909-001
Oracle: PREPARED ✅
DB2:    PREPARED ✅
```

> "Both said yes. Data is locked and waiting."

### Entry 4 — THE COMMIT DECISION ⭐ (Point of No Return)

```text
TXN-ID: XA-20240909-001
DECISION: COMMIT
Written at: 09:00:01.040
```

> [!IMPORTANT]
> This is the most important line in the entire log. The instant WAS writes this entry, the transaction is permanently decided in WAS's view. **Before this entry: rollback is still possible. After this entry: it will be committed, come hell or high water.**

> Analogy: like signing a cheque. Once your signature is on it, you can't "unsign" it — the rest is just delivering it.

### Entry 5 — Resources Committed

```text
TXN-ID: XA-20240909-001
Oracle: COMMITTED ✅
DB2:    COMMITTED ✅
Transaction: COMPLETE
```

> "Delivery done. Both databases confirmed. Diary entry can be erased."

---

## 3. The Crash Gap (Why Entry 4 Is Everything)

```text
Entry 3: Both PREPARED        ✅ written
Entry 4: DECISION = COMMIT    ✅ written
        💥💥 CRASH HERE 💥💥
Entry 5: Both COMMITTED       ❌ NEVER written
```

### What Happens on Restart

1. WAS opens its diary.
2. Finds: "TXN-001, I decided COMMIT, but never got confirmation it finished."
3. Treats this as **unfinished business**.
4. Calls Oracle: "Did my COMMIT arrive?" → Oracle: "No, I'm still PREPARED, waiting." → WAS: "Commit now." → Oracle commits ✅
5. Calls DB2: same conversation → DB2 commits ✅
6. Transaction complete. Both databases agree. Money moved correctly. ✅

> [!NOTE]
> This is exactly how the ~95% "clean recovery" scenario works. The diary turns a potential disaster into a 10-second non-event.

---

## 4. When the Log Is Lost (The Horror Stories)

If the log is gone or corrupted (disk died, someone deleted it, profile wiped), WAS restarts, opens the diary... blank. No Entry 4.

WAS makes a **safe default decision: ROLLBACK**.

> [!NOTE]
> Why rollback? If WAS has no record, it doesn't know the transaction was meant to commit. Committing risks "creating something from nothing." For a bank, assuming nothing happened is the safer bet.

The problem: the databases may end up in **different states**.

### Case A — Bad but Survivable

| Resource | Outcome |
|---|---|
| Oracle | Rolls back the debit ✅ |
| DB2 | Rolls back the credit ✅ |

**Result:** Transaction cancelled. Customer sees "Payment Failed." Bank is consistent. No money created or destroyed. Acceptable.

### Case B — The Nightmare (Split-Brain)

| Resource | Outcome |
|---|---|
| Oracle | back the debit ✅ (money stays in the customer's account) |
| DB2 | Cannot be reached. Rolls back later — or the merchant already got the credit |

**Result:**

- Customer's account: money NOT taken out 💰
- Merchant's account: money WAS credited 💰
- **₹50,000 exists in TWO places — created from NOTHING.**

This is a **heuristic mixed outcome** — one side decided one way, the other decided differently. In a bank, this is fraud-shaped. Auditors will find it. Customers will find it first.

> [!TIP]
> Analogy: You told two tellers to cancel a transaction. One heard you and cancelled it. The other was in the washroom, came back, and completed it as originally instructed. Now the books don't balance.

---

## 5. The Rules That Prevent This (Memorize These)

Break these and you WILL have a Case B story.

### Rule 1 — Log on local or SAN disk. NEVER on NFS.

- The log write must be fast and guaranteed (a **forced write** — written to physical disk immediately, not cached).
- NFS/network mounts can "lie" — they may report a write as complete when it's still in a cache somewhere.
- Crash + NFS + cache = lost Entry 4 **without knowing it**. Worst possible combination.

### Rule 2 — Log on redundant storage (RAID, mirrored disk)

- If the physical disk dies, the log must survive.
- Use RAID-1 / RAID-10 or SAN-backed storage. Not a single laptop-grade disk.

### Rule 3 — Monitor the log disk

- If the disk fills up, WAS cannot write its decision. Transactions hang or fail.
- Set disk space alerts on the tranlog filesystem. Simple, cheap, lifesaving.

### Rule 4 — Back up / protect the tranlog directory

- Know what's in it. Know where it is. Know it's included in your DR plan.

### Rule 5 — Never delete a profile with active recovery pending

- Decommissioning a server? Check for in-doubt transactions **FIRST**.
- Deleting the log = creating orphaned XA transactions.

### Rule 6 — One server = one log. Don't get clever.

- Two servers pointing at the same log directory = corruption. WAS locks logs per server.

---

## 6. Quick Sanity Checks You Can Do Today

### Check 1 — Find your log

```bash
ls -l /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/tranlog/server1/
```

Files exist? Good. Note the disk they're on.

### Check 2 — Confirm the disk type

Ask your storage team: *"Is this local/SAN with write caching protected (battery-backed)?"* Not NFS? Good.

### Check 3 — Disk free space

Check free space on that filesystem and add it to monitoring.

### Check 4 — Don't read the log contents

> [!]
> The tranlog is binary by design. Recovery is automatic — you verify recovery through `SystemOut.log` (WTRN messages), **never** by reading the tranlog files.

---

## 7. Memory Summary

- **Location:** `<profile>/tranlog/<server>/transaction.log` — binary, rotated, one per server.
- **5 entries per transaction:** Started → Prepare sent → Prepared → **DECISION** → Committed.
- **Entry 4 is the point of no return.** Everything before it = can rollback. After it = must commit.
- **Crash between Entry 4 and Entry 5** = WAS finishes the job on restart. Automatic.
- **Log lost** = WAS defaults to ROLLBACK.
- **back on one DB but not the other** = heuristic mixed outcome = money created from nothing = nightmare.
- **Protect the log:** local/SAN disk, not NFS, redundant storage, monitored space.
- **You never read the tranlog.** You verify recovery via `SystemOut.log` WTRN messages.
