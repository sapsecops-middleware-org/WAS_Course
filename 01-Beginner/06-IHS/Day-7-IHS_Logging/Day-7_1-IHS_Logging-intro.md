# IBM HTTP Server (IHS) Logging

> Day 29: a practical guide to the IHS `error_log` and `access_log`, including configuration, log levels, log formats, and how to read entries.

## 1. Overview

**IHS (IBM HTTP Server)** is IBM's web server software, built on Apache HTTP Server. It receives requests from clients (browsers, mobile apps) and either serves content directly or forwards requests to WebSphere Application Server through the plug-in.

- A **request** is a single client action: opening a page or clicking a button each generate one or more requests.
- A **log** is a file in which IHS records what happens, so you can investigate issues after the fact (for example, an incident at 2 AM).

IHS maintains two primary logs:

| Log | Records | Use it to answer |
|---|---|---|
| `access_log` | Every request received | **Who** did **what**, **when**, and what **response** they got |
| `error_log` | Every problem IHS encountered | **Why** something broke |

> [!NOTE]
> A third file, `http_plugin.log`, records activity of the WebSphere plug-in. It is covered in Day 30 and is out of scope here.

---

## 2. Log Files and Locations

Default locations on a typical Linux installation:

```text
/opt/IBM/HTTPServer/logs/access_log
/opt/IBM/HTTPServer/logs/error_log
```

| Path segment | Meaning |
|---|---|
| `/opt/IBM/HTTPServer/` | IHS installation directory |
| `logs/` | Directory holding the log files |
| `access_log` | Request log |
| `error_log` | Error log |

> [!WARNING]
> Many organizations relocate logs. Never assume the default path. Check `httpd.conf` as described below.

---

## 3. Finding Your Log Configuration

All IHS behavior, including logging, is defined in the main configuration file:

```text
/opt/IBM/HTTPServer/conf/httpd.conf
```

List every logging directive:

```bash
grep -i "ErrorLog\|CustomLog\|TransferLog" /opt/IBM/HTTPServer/conf/httpd.conf
```

| Part | Meaning |
|---|---|
| `grep` | Searches for text within a file |
| `-i` | Case-insensitive match |
| `"ErrorLog\|CustomLog\|TransferLog"` | Match any of the three directives (`\|` means OR) |
| `/opt/.../httpd.conf` | File to search |

The three directives you may see:

| Directive | Controls |
|---|---|
| `ErrorLog` | Location of the error log |
| `CustomLog` | Access log with a **selectable format** (recommended) |
| `TransferLog` | Access log with a **default format** (legacy) |

> [!TIP]
> Virtual hosts can define their own log directives. If the main configuration shows nothing for a site, also search included files: `grep -ri "CustomLog" /opt/IBM/HTTPServer/conf/`.

---

## 4. error_log

### 4.1 What Gets Logged

| Event | Meaning |
|---|---|
| Startup / shutdown messages | IHS started or stopped |
| SSL handshake failure | A secure (HTTPS) connection could not be established |
| Port binding failure | IHS could not listen on a port (already in use or not permitted) |
| Configuration errors | Mistakes in `httpd.conf` |
| Backend connection failure | IHS could not reach WebSphere |
| Permission denied | IHS was not allowed to read a file or directory |

### 4.2 Configuration

```apache
ErrorLog /opt/IBM/HTTPServer/logs/error_log
LogLevel warn
```

- `ErrorLog` defines **where** to write.
- `LogLevel` defines **how much detail** to write.

### 4.3 LogLevel

Levels run from most severe to least severe. Setting a level also logs everything **more severe** than it. A higher-detail level produces more lines and fills disk faster.

```text
LOW detail  <-- emerg, alert, crit, error, warn, notice, info, debug --> HIGH detail
```

| Level | Meaning | Typical example |
|---|---|---|
| `emerg` | System unusable | Whole site down |
| `alert` | Immediate action required | Critical failure in progress |
| `crit` | Critical condition | Important component failed |
| `error` | An operation failed | A request failed |
| `warn` | Something looks wrong; server still works | Non-fatal misconfiguration |
| `notice` | Normal but significant event | Server restart |
| `info` | Informational, verbose | Routine details |
| `debug` | Maximum detail | Step-by-step internals |

### 4.4 Recommended Level by Environment

| Environment | Recommended `LogLevel` | Reason |
|---|---|---|
| Production | `warn` | Sufficient detail without filling the disk |
| DEV / UAT | `debug` (temporarily) | Needed to diagnose a bug |
| After the issue is fixed | Revert to `warn` | Avoid disk exhaustion |

> [!CAUTION]
> Do not leave `debug` enabled in production. It generates large volumes of data and can fill the filesystem.

### 4.5 Anatomy of an error_log Entry

```text
[Sat Sep 19 03:14:22.381262 2026] [ssl:error] [pid 12034] AH02042: rejecting client initiated renegotiation
```

| # | Field | Value | Meaning |
|---|---|---|---|
| 1 | Timestamp | `Sat Sep 19 03:14:22.381262 2026` | When the event occurred |
| 2 | Module : level | `ssl:error` | The `ssl` module logged this at `error` level |
| 3 | PID | `pid 12034` | Process ID that wrote the entry |
| 4 | Message | `AH02042: rejecting client initiated renegotiation` | The actual event, prefixed by an error code |

**About `AH` codes:** `AH` identifies an Apache HTTP Server message ID (IHS is based on Apache), followed by a numeric code.

> [!TIP]
> Search the `AH` code (for example `AH02042`) along with "IHS" or "Apache". Most codes are well documented and include known causes and fixes.

---

## 5. access_log

### 5.1 What Gets Logged

**Every request**, whether it succeeds or fails: page loads, logins, `404` errors, `503` errors, and so on.

This is typically the most important log for audits and investigations:

| Question | Source |
|---|---|
| Why did customers see `404` errors yesterday? | `access_log` |
| Did the HTTP-to-HTTPS redirect work? | `access_log` |
| Which IP made hundreds of login attempts at 3 AM? | `access_log` |
| How many users arrived during peak hours? | `access_log` |

### 5.2 Configuration: `TransferLog` vs `CustomLog`

**`TransferLog`** (legacy; fixed format):

```apache
TransferLog /opt/IBM/HTTPServer/logs/access_log
```

**`CustomLog`** (recommended; you choose the format):

```apache
CustomLog /opt/IBM/HTTPServer/logs/access_log combined
```

| Directive | Format control | Recommendation |
|---|---|---|
| `TransferLog` | None (default format) | Legacy |
| `CustomLog` | Full control via a named `LogFormat` | Recommended |

### 5.3 The `combined` Format

`combined` is a named format defined with `LogFormat`:

```apache
LogFormat "%h %l %u %t \"%r\" %>s %b \"%{Referer}i\" \"%{User-Agent}i\"" combined
```

Each `%` token inserts one piece of request information.

| Token | Name | Meaning | Example |
|---|---|---|---|
| `%h` | Remote host | Client IP address | `10.45.2.31` |
| `%l` | Remote logname | Legacy identity lookup; almost always `-` | `-` |
| `%u` | Remote user | Authenticated username; `-` if none | `rajesh` or `-` |
| `%t` | Time | Request time with timezone | `[19/Sep/2026:03:14:22 +0530]` |
| `%r` | Request line | Method, URL, and protocol | `"POST /netbanking/login HTTP/1.1"` |
| `%>s` | Final status | HTTP status of the final response | `200`, `404`, `503` |
| `%b` | Bytes sent | Response body size; `-` if none | `4823` |
| `%{Referer}i` | Referer header | Page the client came from | `"https://www.example.com/"` |
| `%{User-Agent}i` | User-Agent header | Browser, device, or client software | `"Mozilla/5.0 (Windows NT 10.0)"` |

Notes:

- `%{Name}i` logs the value of the request header called `Name`.
- `\"` places literal quotes around a value so field boundaries are clear.
- The `>` in `%>s` selects the **final** status after any internal redirects (as opposed to the original status).

> [!IMPORTANT]
> In deployments behind a load balancer (such as F5), `%h` usually shows the **load balancer's IP**, not the client's. The real client IP is normally carried in the `X-Forwarded-For` header. To log it, add `%{X-Forwarded-For}i` to your `LogFormat`.

### 5.4 Reading a Real access_log Entry

```text
10.45.2.31 - rajesh [19/Sep/2026:03:14:22 +0530] "POST /netbanking/login HTTP/1.1" 200 4823 "https://www.example.com/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"
```

| # | Value | Token | Interpretation |
|---|---|---|---|
| 1 | `10.45.2.31` | `%h` | Client IP (or the load balancer) |
| 2 | `-` | `%l` | No identity information (normal) |
| 3 | `rajesh` | `%u` | Authenticated user |
| 4 | `19/Sep/2026:03:14:22 +0530` | `%t` | 03:14 AM, IST (UTC+5:30) |
| 5 | `POST /netbanking/login HTTP/1.1` | `%r` | Login form submission |
| 6 | `200` | `%>s` | Success response |
| 7 | `4823` | `%b` | About 4.7 KB returned |
| 8 | `https://www.example.com/` | `%{Referer}i` | Arrived from the home page |
| 9 | `Mozilla/5.0 (Windows NT 10.0; Win64; x64)` | `%{User-Agent}i` | Desktop browser on Windows |

**Summary:** at 03:14 AM, user `rajesh`, on a Windows desktop browser, arriving from the home page, submitted the net banking login form and received a successful response.

> [!NOTE]
> A `200` on a login URL means the server responded successfully. It does not by itself prove the credentials were valid; that depends on how the application responds.

---

## 6. HTTP Status Code Reference

| Range | Category | Examples |
|---|---|---|
| `2xx` | Success | `200` OK |
| `3xx` | Redirection | `301`, `302` (for example, HTTP to HTTPS) |
| `4xx` | Client error | `401` unauthenticated, `403` forbidden, `404` not found |
| `5xx` | Server error | `500` internal error, `503` backend unavailable |

> [!TIP]
> `4xx` means the request was at fault. `5xx` means the server side was at fault.

---

## 7. Real-World Use Cases

### Scenario 1: Production Incident (Site Down)

1. Check `error_log` for backend connection failures. This indicates IHS cannot reach WebSphere.
2. Check `access_log` for a run of `503` responses and note when they began (for example, 01:58).
3. You now know **what** failed and **since when**; escalate to the WebSphere team.

### Scenario 2: Brute-Force / Fraud Investigation

Count login attempts per IP within a time window:

```bash
grep "POST /netbanking/login" access_log | grep "19/Sep/2026:03:" | awk '{print $1}' | sort | uniq -c | sort -rn | head
```

An IP with an unusually high count is a candidate to block.

### Scenario 3: Compliance Audit (HTTPS Enforcement)

Use `access_log` to show that plain-HTTP requests to sensitive URLs receive a redirect (`301`/`302`) rather than content, and that successful logins arrive over HTTPS.

> [!NOTE]
> The default `combined` format does not record the request scheme. For stronger audit evidence, add `%p` (server port) or `%{X-Forwarded-Proto}i` to a custom `LogFormat`. Do not rely on the `Referer` header as proof.

### Scenario 4: Capacity Planning

Count requests in a time window (here, 12:40 to 12:49 on a given day):

```bash
grep "19/Sep/2026:12:4" access_log | wc -l
```

---

## 8. Useful Commands

```bash
# Last 10 access_log entries (newest at the bottom)
tail -10 /opt/IBM/HTTPServer/logs/access_log

# Follow the access log live (Ctrl+C to stop)
tail -f /opt/IBM/HTTPServer/logs/access_log

# All 503 responses
grep " 503 " /opt/IBM/HTTPServer/logs/access_log

# Last 10 error_log entries
tail -10 /opt/IBM/HTTPServer/logs/error_log
```

| Command | Purpose |
|---|---|
| `tail -10 <file>` | Show the last 10 lines |
| `tail -f <file>` | Stream new lines as they are written |
| `grep " 503 " <file>` | Show only lines containing ` 503 ` |
| `wc -l` | Count lines |

> [!TIP]
> Logs grow continuously. Use log rotation (for example Apache's `rotatelogs` piped from `CustomLog`, or `logrotate`) so files do not exhaust disk space.

---

## 9. Quick Reference

| Topic | Key point |
|---|---|
| Config file | `/opt/IBM/HTTPServer/conf/httpd.conf` |
| Default log directory | `/opt/IBM/HTTPServer/logs/` |
| Who did what? | `access_log` |
| Why did it break? | `error_log` |
| Production `LogLevel` | `warn` |
| Recommended access log directive | `CustomLog ... combined` |
| Behind a load balancer | Log `X-Forwarded-For` to capture the real client IP |
| `4xx` / `5xx` | Client-side fault / server-side fault |
| Error code lookup | Search the `AH` code |