# IBM HTTP Server (IHS) Log Analysis — Complete Beginner's Guide

A practical, command-by-command reference for reading, searching, and analyzing IBM HTTP Server logs in production.

---

## 1. What is IBM HTTP Server (IHS)?

IBM HTTP Server acts as the front door for your WebSphere applications:

```
User Request → IHS → WebSphere Application Server → Response
```

IHS records all activity in two primary log files, located by default in `/opt/IBM/HTTPServer/logs/`:

| File | Records | Analogy |
|------|---------|---------|
| `access_log` | Every request: client IP, timestamp, URL, status code, bytes | Visitor register |
| `error_log` | Problems, crashes, SSL/TLS issues | Complaint book |

---

## 2. Anatomy of a Log Line

```text
10.45.99.5 - - [10/Jun/2025:14:32:07 +0530] "POST /netbanking/login HTTP/1.1" 500 2321 45321
```

| Field | Value | Meaning |
|-------|-------|---------|
| `$1` | `10.45.99.5` | Client IP address |
| `2‘,‘2`, `2‘,‘3` | `- -` | Identity fields (usually unused) |
| `$4` | `[10/Jun/2025:14:32:07 +0530]` | Date & time (with timezone) |
| `5‘–‘5`–`5‘–‘7` | `"POST /netbanking/login HTTP/1.1"` | Request method, URL, protocol |
| `$8` | `500` | HTTP status code |
| `$9` | `2321` | Bytes sent |
| `$NF` | `45321` | Response time in microseconds (only if `%D` is logged last) |

### HTTP Status Codes at a Glance

| Code | Meaning |
|------|---------|
| 2xx | Success ✅ |
| 3xx | Redirect |
| 4xx (401/403/404) | Client-side mistake |
| 5xx (500/503) | Server-side mistake 🔥 |

---

## 3. Watching Logs Live

```bash
# Watch error_log live during a restart or incident
tail -f /opt/IBM/HTTPServer/logs/error_log

# Watch access_log live to see requests hitting IHS
tail -f /opt/IBM/HTTPServer/logs/access_log

# Watch both simultaneously (run in background)
tail -f /opt/IBM/HTTPServer/logs/error_log &
tail -f /opt/IBM/HTTPServer/logs/access_log &
```

**Key points:**

- `tail` = show the end of the file; `-f` = follow (live stream).
- `&` = run in background so both commands can run in one terminal.
- `screen` / `tmux` = persistent terminal sessions that survive disconnects.
- `Ctrl+C` stops following.
- `tail -100f` shows the last 100 lines first, then follows.

---

## 4. grep — Searching Logs

```bash
# All 503 errors (last 100 shown)
grep " 503 " /opt/IBM/HTTPServer/logs/access_log | tail -100

# All 500 errors for the login URL
grep "POST /netbanking/login" /opt/IBM/HTTPServer/logs/access_log | grep " 500 "

# All requests from a specific IP
grep "^10.45.99.5 " /opt/IBM/HTTPServer/logs/access_log

# Count 503 errors
grep " 503 " /opt/IBM/HTTPServer/logs/access_log | wc -l

# SSL/TLS problems
grep -i "ssl\|handshake\|certificate" /opt/IBM/HTTPServer/logs/error_log
```

| Flag / Pattern | Meaning |
|----------------|---------|
| `-i` | Case-insensitive match |
| `\|` (inside pattern) | OR between patterns |
| `^pattern` | Match at line start only |
| `wc -l` | Count matching lines |

> [!TIP]
> The spaces in `" 503 "` prevent false matches such as `1503` or `5030`.
> The `^` anchor ensures the IP is the **first** field — important when the real client IP arrives via the `X-Forwarded-For` header.

---

## 5. awk — Timing Analysis

With `%D` in the `LogFormat` (request duration in microseconds), and `%D` as the **last** field:

```bash
# Requests that took more than 5 seconds
awk '$NF > 5000000' /opt/IBM/HTTPServer/logs/access_log
```

- `$NF` = last field on the line (`N` = number of fields).
- Unit math: `1,000,000 µs = 1 second` → `5,000,000 µs = 5 s`. Divide by 1000 for milliseconds.

---

## 6. Verifying Log Configuration

```bash
grep -E "^ErrorLog|^LogLevel|^CustomLog|^TransferLog" \
  /opt/IBM/HTTPServer/conf/httpd.conf
```

| Pattern | Shows |
|---------|-------|
| `^ErrorLog` | Error log path(s) |
| `^LogLevel` | Verbosity: `warn`, `error`, `info`, `debug` |
| `^CustomLog` | Access log paths + format nicknames |
| `^TransferLog` | Legacy access log directive (if present) |

> [!NOTE]
> The `^` anchor prevents false matches on commented-out lines such as `# ErrorLog ...`.

---

## 7. Six Production Recipes

### 7.1 Requests per Minute (Live Monitoring)

```bash
tail -f /opt/IBM/HTTPServer/logs/access_log | \
  awk '{print $4}' | cut -c1-18 | uniq -c
```

- `tail -f` → live stream
- `$4` → timestamp `[10/Jun/2025:14:32:07`
- `cut -c1-18` → first 18 characters = date + hour + minute
- `uniq -c` → count per minute → live traffic meter

### 7.2 Top 10 URLs Getting 500 Errors

```bash
grep " 500 " /opt/IBM/HTTPServer/logs/access_log | \
  awk '{print $7}' | sort | uniq -c | sort -rn | head -10
```

Filter 500s → extract URL (column 7) → group & count → sort descending → show top 10.
Answers: *"Which pages are breaking?"*

### 7.3 Slowest Requests (> 3 Seconds)

> [!NOTE]
> Requires `%D` in the `LogFormat`.

```bash
awk 'NF > 3000000 {print0}' /opt/IBM/HTTPServer/logs/access_log | \
  tail -20
```

- `$0` = the whole line
- `tail -20` = 20 most recent slow requests

### 7.4 Count Requests by Hour (Find Peak Time)

```bash
awk '{print $4}' /opt/IBM/HTTPServer/logs/access_log | \
  cut -c14-15 | sort | uniq -c
```

- In `[DD/Mon/YYYY:HH:MM:SS`, characters 14–15 = hour
- Output = hourly traffic histogram → capacity planning and safe restart windows

### 7.5 Unique IPs That Got 401 (Unauthorized)

```bash
grep " 401 " /opt/IBM/HTTPServer/logs/access_log | \
  awk '{print $1}' | sort -u
```

- `sort -u` = unique IPs only
- Security tip: one IP with many 401s may indicate brute-force password guessing.

### 7.6 Total Data Transferred (Sum of Bytes)

> [!WARNING]
> Only works if the bytes column is field `$10`. Verify your `LogFormat` first.

```bash
awk '{sum += $10} END {print sum/1024/1024 " MB"}' \
  /opt/IBM/HTTPServer/logs/access_log
```

- Adds column 10 (bytes) across all lines; the `END` block prints once at EOF.
- Divides by 1024 twice → megabytes.

---

## 8. Tool Cheat Sheet

| Tool | Job | Key Flags / Usage |
|------|-----|-------------------|
| `tail` | Show end of file / live follow | `-f`, `-100f`, `&` |
| `grep` | Search text | `-i`, `\|` (OR), `^` (anchor) |
| `awk` | Pick columns, do math | `1‘,‘1`, `1‘,‘NF`, `$0`, `sum +=`, `END` |
| `cut` | Slice characters by position | `-c1-18`, `-c14-15` |
| `sort` | Sort lines | `-r` reverse, `-n` numeric, `-u` unique |
| `uniq` | Count/merge duplicates | `-c` (always sort first!) |
| `wc` | Count | `-l` lines |
| `head` | Show top lines | `-10` |

> [!IMPORTANT]
> **Golden rule:** `uniq` only merges *adjacent* duplicates → always pipe through `sort` before `uniq`.

---

## 9. Real Production Flow — "Netbanking Is Slow"

A 5-minute diagnosis checklist:

1. `tail -f error_log` → crashes? SSL errors?
2. `grep " 503 " access_log | wc -l` → backend down?
3. **Recipe 7.3** → which requests are slow?
4. **Recipe 7.2** → which URLs? If `/netbanking/login` → check WebSphere.
5. **Recipe 7.5** → attack from a single IP?
6. **Recipe 7.4** → just peak-hour load?

---

## 10. Quick Reference Card

```bash
# Live watch
tail -f /opt/IBM/HTTPServer/logs/error_log

# Count 5xx errors
grep -c " 503 " /opt/IBM/HTTPServer/logs/access_log

# Slow requests (> 5s, requires %D last)
awk '$NF > 5000000' /opt/IBM/HTTPServer/logs/access_log

# Peak hour analysis
awk '{print $4}' /opt/IBM/HTTPServer/logs/access_log | cut -c14-15 | sort | uniq -c

# Suspicious IPs (many 401s)
grep " 401 " /opt/IBM/HTTPServer/logs/access_log | awk '{print $1}' | sort -u

# Total data transferred
awk '{sum += $10} END {print sum/1024/1024 " MB"}' /opt/IBM/HTTPServer/logs/access_log

# Verify log configuration
grep -E "^ErrorLog|^LogLevel|^CustomLog|^TransferLog" /opt/IBM/HTTPServer/conf/httpd.conf
```
