# IBM HTTP Server — Real Production Scenarios & Log Analysis

Real-world incident walkthroughs using IHS logs: a critical startup failure at 3:30 AM and a security investigation request. Includes diagnosis, fix commands, and prevention steps.

---

## Table of Contents

1. [Scenario 1 — IHS Won't Start After Midnight Patch (3:30 AM)](#scenario-1)
2. [Scenario 2 — Brute-Force Attack Investigation (10 AM)](#scenario-2)
3. [Key Takeaways](#key-takeaways)

---

## Scenario 1 — IHS Won't Start After Midnight Patch (3:30 AM) {#scenario-1}

> **Incident:** 3:30 AM, UPI batch window. Phone rings: *"IHS is throwing errors. Transactions not going through."*

### Step 1 — Check the Error Log

```bash
tail -100 /opt/IBM/HTTPServer/logs/error_log
```

You see:

```text
[crit] (13)Permission denied: /opt/IBM/HTTPServer/logs/access_log could not be opened
```

### Diagnosis

| Clue | Meaning |
|---|---|
| `[crit]` | Critical severity — IHS cannot continue cleanly |
| `(13)Permission denied` | OS-level file permission failure (errno 13) |
| `could not be opened` | IHS cannot write its own access log |

**Root cause:** A midnight OS patch broke log file permissions. IHS can't open its log file and refuses to start cleanly.

### Fix

```bash
# 1. Check ownership of log files
ls -la /opt/IBM/HTTPServer/logs/

# 2. Fix ownership (IHS runs as 'wasadmin' user)
chown wasadmin:wasadmin /opt/IBM/HTTPServer/logs/access_log
chown wasadmin:wasadmin /opt/IBM/HTTPServer/logs/error_log

# 3. Graceful restart
/opt/IBM/HTTPServer/bin/apachectl graceful
```

> [!TIP]
> Use `apachectl graceful` instead of a hard restart — it finishes in-flight requests before reloading, so UPI transactions in progress are not dropped.

### Prevention

> [!IMPORTANT]
> Add a **log file permission check** to your pre-patch checklist:
>
> ```bash
> # Verify the IHS user owns the logs before/after any patching
> ls -la /opt/IBM/HTTPServer/logs/ | grep wasadmin
> ```

---

## Scenario 2 — Brute-Force Attack Investigation (10 AM) {#scenario-2}

> **Request from Security Team:** *"Detect brute-force login attempts on the NetBanking portal. Give me the last 24 hours."*

### The One-Liner

```bash
# Count login POST requests per IP, sorted descending
grep "POST /netbanking/login" /opt/IBM/HTTPServer/logs/access_log \
  | awk '{print $1}' \
  | sort | uniq -c | sort -rn | head -20
```

### How the Pipeline Works

| Step | Command | What It Does |
|---|---|---|
| 1 | `grep "POST /netbanking/login"` | Keep only login attempts |
| 2 | `awk '{print $1}'` | Extract the client IP (first field) |
| 3 | `sort` | Group identical IPs together |
| 4 | `uniq -c` | Count occurrences per IP |
| 5 | `sort -rn` | Sort by count, highest first |
| 6 | `head -20` | Show top 20 offenders |

### Sample Output

```text
3847  185.234.100.5    ← BRUTE FORCE — block this IP immediately
 142  10.0.1.45
  89  10.0.1.92
  12  10.0.2.11
```

### Interpretation

| IP | Requests | Verdict |
|---|---|---|
| `185.234.100.5` | 3,847 | **Brute force** — external IP hammering the login page. Block immediately. |
| `10.0.1.45` / `10.0.1.92` | 142 / 89 | Internal IPs — verify with application team (could be a monitoring service or misconfigured batch job). |
| `10.0.2.11` | 12 | Normal volume — no action. |

**Action:** Hand the analysis to the firewall team → IP blocked in 10 minutes.

---

## Key Takeaways

- **`error_log` is your first stop** in any incident — `[crit]` lines tell you exactly why IHS failed.
- **Permission errors after patching are common** — the IHS runtime user (`wasadmin`) must own the log files.
- **`access_log` is your CCTV footage** — a simple `grep | awk | sort | uniq -c | sort -rn` pipeline turns millions of lines into a ranked attacker list in seconds.
- **Always include prevention** — every incident should end with a checklist update so it never repeats.

> [!TIP]
> Memorize the counting pipeline — it works on any log, any field:
>
> ```bash
> grep "<pattern>" <logfile> | awk '{print $1}' | sort | uniq -c | sort -rn | head -20
> ```
