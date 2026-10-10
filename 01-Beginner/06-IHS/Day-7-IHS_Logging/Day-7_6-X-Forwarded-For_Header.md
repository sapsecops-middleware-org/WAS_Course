# Capturing Real Client IP in IBM HTTP Server (IHS) Using X-Forwarded-For

## Overview

In enterprise architectures where IBM HTTP Server (IHS) sits behind load balancers and firewalls (e.g., banking environments with F5, WAF, etc.), the access log records the **load balancer's IP** instead of the actual customer's IP. This document explains the problem, the solution using the `X-Forwarded-For` (XFF) header, and the step-by-step implementation in IHS.

## Architecture Context

A typical traffic flow in a bank:

```text
Customer Browser (115.98.22.5)
        |
     Internet
        |
   Firewall / WAF
        |
   F5 Load Balancer (10.0.0.1)
        |
        IHS
        |
      Plugin
        |
       WAS
```

- **Real client IP:** `115.98.22.5`
- **Last device to touch the request before IHS:** F5 Load Balancer (`10.0.0.1`)
- **IP seen by IHS:** `10.0.0.1` (the F5's internal IP)

## The Problem

Because the F5 is the last hop, IHS logs every request as originating from `10.0.0.1`:

```text
10.0.0.1 - - [19/Sep/2026:03:14:22] "POST /netbanking/login" 200 4823
10.0.0.1 - - [19/Sep/2026:03:14:23] "POST /netbanking/login" 200
10.0.0.1 - - [19/Sep/2026:03:14:24] "POST /netbanking/login" 200 4823
```

### Why This Is Critical in a Bank

| Impact | Description |
|---|---|
| Fraud investigation | Cannot determine which account was accessed from where |
| Brute-force detection | 10,000 failed logins — one attacker or 100 users? Logs show only `10.0.0.1` |
| Regulatory (RBI) audits | Cannot answer "who accessed customer X's account at 3 AM?" |
| IP blocking | Blocking the attacker's IP is impossible — you would block the F5 and take down the whole site |

## The Solution: X-Forwarded-For

`X-Forwarded-For` (XFF) is an HTTP request header that carries the real client IP through all intermediate devices.

### How It Works

The F5 is configured to insert the header before forwarding:

```text
X-Forwarded-For: 115.98.22.5
```

IHS reads this header and logs the real client IP.

### Proxy Chains

If the request traverses multiple devices, each hop appends its IP:

```text
X-Forwarded-For: 115.98.22.5, 172.16.1.5, 10.0.0.1
```

- **First IP** = the real client
- **Remaining IPs** = intermediate hops (WAF, F5, etc.)

## Implementation in IHS

Edit `httpd.conf`:

```apache
# Capture real client IP using X-Forwarded-For
LogFormat "%{X-Forwarded-For}i %h %l %u %t \"%r\" %>s %b \"%{Referer}i\"" real_ip_format

CustomLog /opt/IBM/HTTPServer/logs/access_log real_ip_format
```

### LogFormat Token Reference

| Token | Meaning |
|---|---|
| `%{X-Forwarded-For}i` | Value of the XFF header (real client IP) |
| `%h` | Remote host — the direct connection (the F5, `10.0.0.1`) |
| `%l` | Remote logname (usually `-`) |
| `%enticated user (usually `-`) |
| `%t` | Timestamp |
| `\"%r\"` | First line of request (method, URL, protocol) |
| `%>s` | Final HTTP status code (200, 404, etc.) |
| `%b` | Bytes sent (response size) |
| `%{Referer}i` | Referring page |

> [!NOTE]
> The `i` in `%{...}i` means "read this **incoming** request header."

### Result

**Before:**

```text
10.0.0.1 - - [19/Sep/2026:03:14:22] "POST /netbanking/login" 200 4823
```

**After:**

```text
115.98.22.5 10.0.0.1 - - [19/Sep/2026:03:14:22] "POST /netbanking/login" 200 4823
```

- Field 1 = real customer IP
- Field 2 = F5 IP (direct connection source)

## Implementation Checklist

1. Confirm with the network team that the F5 inserts `X-Forwarded-For` (F5 setting: **HTTP profile → Insert XFF: Enabled**).
2. Back up `httpd.conf`.
3. Add the `LogFormat` directive.
4. Update `CustomLog` to use `real_ip_format`.
5. Test with:
   ```bash
   curl -H "X-Forwarded-For: 1.2.3.4" http://yourserver/
   ```
6. Restart or gracefully reload IHS:
   ```bash
   /opt/IBM/HTTPServer/bin/apachectl graceful
   ```
7. Verify that logs now show real client IPs.

## Operational Warnings

> [!WARNING]
> **XFF can be spoofed by clients.** Anyone can send `X-Forwarded-For: <anything>` from their browser. Only trust XFF originating from your own F5 on the internal network. Always log `%h` alongside XFF to verify the direct connection really came from the F5 (`10.0.0.1`).

> [!WARNING]
> **Handle the comma-separated list.** With multiple hops, the first IP is the client. Your SIEM / log parser must extract the **first** value.

> [!TIP]
> **Test a direct request too.** If a request bypasses the F5 and hits IHS directly, no XFF header exists — the field will show `-`. This is expected; `%h` still identifies the source.

> [!NOTE]
> **Modern alternative:** Newer setups can use `mod_remoteip`, which rewrites the client IP itself so that `%h` shows the real IP. In banking environments with older IHS versions, the `LogFormat` approach above is the safe, standard method.

## Summary

> IBM HTTP Server sees the load balancer's IP, not the customer's. `X-Forwarded-For` is the note the F5 attaches — "here's the real client IP" — and `%{X-Forwarded-For}i` in your `LogFormat` is how you write that note into your access log.
