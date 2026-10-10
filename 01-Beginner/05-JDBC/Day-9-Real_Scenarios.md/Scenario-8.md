# Scenario 8 — The Midnight Crash (OutOfMemoryError / Heap Dump Day)

> [!NOTE]
> **Filename suggestion:** `scenario-08-oom-heap-dump.md`
> **Severity:** Critical (server down overnight)
> **Difficulty:** Intermediate–Advanced

---

## 1. Background Concepts

### 1.1 The Two OOMs (Never Confuse Them)

| Error | Meaning | Usual Cause |
|---|---|---|
| `java.lang.OutOfMemoryError: Java heap space` | Objects filled the **heap** | Memory leak, oversized caches, huge result sets |
| `java.lang.OutOfMemoryError: native memory / Metaspace` | JVM ran out of **native** memory | Thread explosion, direct buffers, classloader leak (redeploys) |

> [!CAUTION]
> Fixing the wrong OOM type is the classic mistake: adding `-Xmx` when the problem is native memory can make it WORSE (bigger heap = less native headroom in a fixed-size container/VM).

### 1.2 WAS Memory behaviour

- Default: WAS writes a **heap dump (phd) + javacore** automatically on OOM into the profile root.
- Files look like: `heapdump.<timestamp>.phd`, `javacore.<timestamp>.txt`, `Snap.*.trc`.

---

## 2. The Incident

- 01:40: server1 dies. Restarted by the night operator at 01:55 — works fine.
- 03:20: dies again. Restart again.
- By morning: three crashes, `OutOfMemoryError: Java heap space` in logs.
- Heap `-Xmx2048`. Survives all day; only crashes at night → **schedule-dependent leak**.

---

## 3. Diagnosis

### Step 1 — Secure the Evidence FIRST

```bash
ls -lht /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/ | head
# heapdump.2026.01.15.phd  javacore.2026.01.15.txt  Snap.20260115.trc
# COPY these off the server before any restart wipes your chance
```

### Step 2 — Read the Javacore (Free Information)

```bash
grep -A3 "OutOfMemory\|Memory" javacore.2026.01.15.txt | head -30
# Also count threads:
grep -c "3XMTHREADINFO" javacore.2026.01.15.txt
```

- 2000+ threads? → thread leak (native OOM pattern).
- Normal thread count? → genuine heap leak → analyze the phd.

### Step 3 — Analyze the Heap Dump

Tool: **IBM HeapAnalyzer** or **Eclipse MAT** (with the IBM DTFJ plugin for .phd files).

What to look for:

| Finding | Meaning |
|---|---|
| One class holds 60–80% of the heap | Leak suspect #1 |
| `byte[]` / `char[]` dominant with huge arrays | Big result sets / file loads held in memory |
| Growing count of `HashMap$Entry` in a cache class | Unbounded cache |
| Session objects count enormous | Session bloat / session never timing out |

### Step 4 — Correlate With the Nightly Schedule

- Crash times: 01:40, 03:20 → check cron / scheduler: **01:30 EOD interest batch, 03:00 archive job**.
- Pattern fits: the batch loads the day's transactions into memory and **keeps a reference** (static map) — heap never released, second run at 03:00 finishes the job.

---

## 4. The Fix

### Immediate

1. Restart window planned; servers stable during day → operate until fix.
2. Temporary mitigation: raise heap to `-Xmx3072` **only if native headroom allows** (`vmstat`/container limits checked first) — buys days, not a fix.

### Root Cause (with the developer)

1. Find the static/cached collection holding batch data → bound it (e.g., `LinkedHashMap` with access-order + `removeEldestEntry`, or evict after processing).
2. Stream results instead of loading all rows (`fetchSize`, cursor-based processing).
3. Fix session hygiene: `session.invalidate()` on logout; verifydefault 30 min is fine; session data size is the issue).

### WAS Tuning (Supporting, Not a Cure)

Console: `Application servers → server1 → Process definition → Java Virtual Machine`:

| Setting | Value | Note |
|---|---|---|
| Initial heap | 102-ish to max reduces resize pauses |
| Maximum heap | 2048–3072 | Only with native headroom |
| `-Xgcpolicy:gencon` (IBM9) | add as generic arg | Good default for throughput + pause balance |
| Verbose GC | enable temporarily | `-verbose:gc` → GC logs to confirm leak curve |

### Leak Curve Verification with verbose:gc

```text
Healthy:-tooth returns to the same baseline each cycle
Leaking:  baseline climbs steadily → heap exhausted after N hours
```

Run overnight with the batch enabled; the climbing baseline confirms the leak is fixed only when the baseline is flat.

---

## 5. Prevention Checklist

- [ ] All static collections must have a **bound or eviction policy** — code review rule.
- [ ] Nightly crash + "operator restart" must raise an **incident**, not be normal ops (restart hides the leak, evidence rots).
- [ ] Heap dumps auto-collected (default) AND copied to an evidence share before restart.
- [ ] verbose:gc or PMI HeapSize monitoring with alert on sustained growth.
- [ ] Redeploy-heavy environments: watch for classloader leaks (native/Metaspace OOM after many redeploys — use `AdminControl` to check classloader counts).

## 6. Memory Hook

> **"Nightly crashes that restart fine = a leak with a schedule. Heap dump is the confession — copy it before you restart the suspect."**

---

# Quick Reference Card — All 8 Scenarios

| # | Symptom | First Move | Golden Rule |
|---|---|---|---|
| 1 | New app not appearing | Check sync status | ** ≠ Sync ≠ Running** |
| 2 | Money missing, no errors | Check DS types | XA All or Nothing |
| 3 | 403 for everyone | `curl` direct to :9080 | Four liars — interrogate one at a time |
| 4 | ORA-00060 / SQL0911 | DBA trace file