# IBM HTTP Server — CustomLog & LogFormat Configuration Guide

A practical guide to configuring access logs in **IBM HTTP Server (IHS)** using `LogFormat` and `CustomLog` directives, including real-client IP logging behind load balancers (F5), performance timing, SSL/TLS fields, and per-site log separation.

---

## Overview

IBM HTTP Server records every incoming HTTP request in an **access log**. The two directives that control this behavior are:

| Directive | Purpose |
|---|---|
| `LogFormat` | Defines a log template and assigns it a nickname (alias) |
| `CustomLog` | Writes log entries to a file using a named format |

### Request Behind a Load Balancer

```text
Client → Internet → F5 (Load Balancer) → IHS
```

Without special handling, IHS logs the **load balancer's IP** (`%h`), not the actual client IP. See [The Real-Client IP Problem](#the-real-client-ip-problem-x-forwarded-for).

---

## Core Concepts

### Mental Model

- ** file** — the server's "visitor register." One line per request.
- **LogFormat** — therecipe*: which fields to record, in what order.
- **CustomLog** — the *action*: apply the recipe and write to a specific file.

> [!TIP]
> Define a `LogFormat` once (with a nickname), then reference that nickname in any number of `CustomLog` directives — including inside individual `<VirtualHost>` blocks.

---

## The Real-Client IP Problem (X-Forwarded-For)

When requests pass through F5 (or any reverse proxy / load balancer), the TCP connection to IHS originates from the **F5's IP address**.h` therefore logs the proxy, not the customer.

### The Fix

F5 injects the original client IP into the `X-Forwarded-For` HTTP request header. Capture it with:

```apache
%{X-ForwardedFor}i
```

**Syntax breakdown:**

| Segment | Meaning |
|---|---|
| `%{...}` | "Look up this" |
| `X-Forwarded-For` | The name |
| `i` | Incoming **request header** |

> [!NOTE]
> Suffix conventions:
> - `%{Name}i` → incoming request header
> - `%{Name}o` → outgoing response header
> - `%{Namex` → SSL module variable (e.g., `SSL_PROTOCOL`)

> [!]
> `X-Forwarded-For` is client-supplied and can be spoofed unless the load balancer overwrites it. For audit-grade logging, ensure F5 is configured to **overwrite** (not append to) the header.

---

## LogFormat Directive

Defines a reusable log template with a nickname.

```apache
LogFormat "<format-string>" <nickname```

**Example:**

```apache
LogFormat "%{X-Forwarded-Fori %h %u %t \"%r\" %>s %b %D \"%{User-Agent}i\" \"%{SSL_PROTOCOL}x\" \"%{SSL_CIPHER}x\"" bank_full
```

The nickname (`_full`) is used later in `CustomLog`.

---

## Format Specifiers Reference

| Specifier | Field | Description |
|---|---|---|
| `%h` Remote host | IP of the direct TCP peer. **Behind5, this is the F5 IP, not the client** |
| `%{X-Forwarded-For}i` | client IP | Client IP injected by the load balancer |
| `%` | Remote logname | Almost always `-` (ident lookup; rarely used) |
| `%u` | Remote user | Authenticated username, if any |
| `%t` | Timestamp | Request time in Apache time format |
| `%{format}t | Custom timestamp | e.g., `%{%Y-%m-%d %H:%M:%S}t` |
| `%r` | First request line | e.g., `GET /login HTTP/1.1` |
| `%>s` | Final status code | `200`, `404` Not Found, `500 Server Error |
| `%b` | Bytes sent | Response body; `-` if zero| `%{Referer}i` | Referrer | The page the user came from |
| `%{User-Agent}i` | User agent | Browser / device / client tool |
| `%D` | Request duration | **Microseconds** (see warning below) |
| `%{SSL_PROTOCOL}x` | SSL/TLS version | e.g., `TLSv1.2`, `TLSv1.3` |
| `%{SSL_CIPHER}x` | SSL/TLS cipher | Exact cipher suite used |

> [!IMPORTANT]
> **`%D` is measured microseconds.**
>
> `250` µs = `250` ms. To convert log values to milliseconds **divide by 0**.

> [!TIP]
> Logging `%{SSL_PROTOCOL}x` and `%SSL_CIPHER}x` lets auditors instantly verify whether legacy protocols (TLS 1.0/1.1) were ever used — a common compliance requirement in banking environments.

---

## Common Format Examples

### 1. `combined` (Default)

```apache
LogFormat "%h %l %ut \"%r\" %>s % \"%{Referer}i\" \"%{User-Agent}i\"" combined
```

Standard NCSA extended format. **Limitation:** `%h` shows the load balancer IP, not the real client.

### 2. `real_ip` (Client IP Corrected)

```apache
LogFormat "%{X-ed-For}i %lu %t \"%r\" %> %b" real_ip
``First field is the real client IP — suitable for environments behind F5.

### 3. `with_time` (Performance Monitoring)

```apache
LogFormat "%h %t \"%r\" %>s %b %D" with_time
```

Adds `%D` ( duration) for latency analysis and performance complaints.

### 4. `bank_full` (Comprehensive / Audit-Grade)

```apache
Log "%{X-Forwarded-}i %h %u %t \"%r\" %>s %bD \"%{User-Agent}i\" \"%{SSL_PROTOCOL}x\" \"%{SSL_CIPHER}x\"" bank_full
```

Captures:

- Real client IP (via `X-Forwarded-For`)
- Direct peer IP (F5)
-enticated user
- Timestamp, request line, status, bytes
- Request duration
- User agent
- TLS protocol version and cipher

> [!T]
> Use `bank_full` for regulated environments (banking healthcare) where auditors require full request traceability and TLS compliance evidence.

---

## CustomLog Directive

Writes access records to a file using a named format.

```apache
CustomLog /opt/IBM/HTTPServer/logs/access_log bank_full
```

**Three components:| Component | Meaning |
|---|---|
| `CustomLog` | The directive |
| `/opt/IBM/HTTPServer/logs/access_log` | Output file path |
| `bank_full` | The `Log` nickname to use |

---

## Per-VirtualHost Log Separation

When hosting multiple sites on one IHS instance, separate logs per site using `<VirtualHost>` blocks.

```apache
<VirtualHost *:443>
    ServerName retail.citibank.co.in
    CustomLog /opt/IBM/HTTPServer/logs/ret_access.log bank_full
    ErrorLog  /opt/IBM/HTTPServer/logs/retail_error.log
</VirtualHost>

<VirtualHost *:443>
   Name corporate.citibank.co.in
    CustomLog /opt/IBM/HTTPServer/logs/corporate_access.log bank_full
    ErrorLog  /opt/IBM/HTTPServer/logs/corporate_error.log
</VirtualHost>
```

**Behavior:** IHS inspects the `Host` header of each request and routes the log entry to the matching VirtualHost's log file.

| Benefit Description |
|---|---|
| Clean separation | One file per site — no mixed traffic |
| Easy audit extraction | "Corporate logs" = one file |
| Independent retention | Rotate/retain per site as required |

---

## Access Log vs Error Log

| Aspect | Access Log (`CustomLog`) | Error Log (`ErrorLog`) |
|---|---|---|
| Contents | Every request (success or failure) | Only problems: crashes, misconfigurations, failed connections |
| Analogy | Visitor register at the gate | Complaint book |
| Use when | "Who visited at 2 AM?" | "Why didn't the page load?" |

## Deployment Checklist

1. **Edit** `httpd.conf` (IHS configuration file).
2. **Add** your `LogFormat` definitions.
3. **Add** your `CustomLog` directive(s).
4. **Validate the configuration **before** restarting:

   ```bash
   /opt/IBMHTTPServer/bin/apachectl configtest
   ```

   - `Syntax OK` → safe to proceed.
   - Any error → fix **before** restarting (never risk a production outage).

5. **Restart** IHS to apply changes.
6. **** by hitting the site and watching the log:

   ```bash
   tail -f /opt/IBM/HTTPServer/logs/access_log
   ```

7. **Confirm** logged line matches your expected format.

---

## Log Rotation

High-traffic sites can generate **gigabytes of logs per day**. Unchecked growth leads to disk exhaustion and outages.

**Recommendations:**

- Rotate logs **daily** (or per size threshold).
 Use `rotatelogs` (bundled with IHS/Apache), `logrotate`, or your organization's standard tooling.
- Compress and archive rotated logs according to your retention policy.

> [!NOTE]
> Consult your platform/ops team for the house-standard rotation method — most enterprises have an established process and retention requirements.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| Log shows F5/proxy IP for all entries | Using `%` without `X-Forwarded-For` | Use `%{X-Forwarded-For}i` |
| `X-Forwarded-For` shows multiple IPs or is missing | Proxy not configured to set/overwrite header | Configure F5 to overwrite the header |
| `%D` values look absurdly large | Reading microseconds as milliseconds | Divide by 1000 |
| `configtest` fails after edit | Typo in format string or missing quote | Check quoting `\"%r\"` and escapes |
| Log file created / empty | Wrong path, permissions, or VirtualHost mismatch | Verify path, ownership, and `ServerName` routing |
| Audit can't find TLS info | SSL fields missing from format | Ensure `%{SSL_PROTOCOL}x` / `%{SSL_CIPHER}x` are in the format and is enabled |

---

## Quick Reference Summary

| Item | One-Liner |
|---|---|
| Log file | The server's visitor register — one line request |
| `%h` | Direct peer IP — **may be the load balancer, not the client** |
| `%{X-Forwarded-For}i` | The real client IP from the proxy-injected header |
| `LogFormat` | Define a template, give it a nickname |
| `CustomLog` | Write entries to a file using a named format |
| `%D` | Request duration in **microseconds**÷ 1000 = ms) |
| `%{SSL_PROTOCOL}x` / `%{SSL_CIPHER}x` | TLS version and cipher audit essentials |
| `<VirtualHost>` logs | One site → one log file pair (access + error) |
| Access log vs Error log | All visits vs only problems |
