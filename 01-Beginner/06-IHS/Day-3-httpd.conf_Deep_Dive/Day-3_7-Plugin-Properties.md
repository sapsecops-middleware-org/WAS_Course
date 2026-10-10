# WebSphere Plugin Properties — IgnoreAffinityRequests, ExtendedHandshake, ChunkedResponse

> [!NOTE]
> This guide explains three key IBM HTTP Server (IHS) plugin properties that control routing and response behavior between IHS and WebSphere Application Server (WAS). Misconfiguring them in production can cause session loss and outages.

## Architecture Overview

```
Users → IHS (Web Server) → Plugin → WebSphere JVMs (was1, was2, was3...)
```

- The **plugin** decides which JVM receives each request.
- These properties control **session affinity**, **health checking**, and **response delivery**.
- Incorrect settings → logged-out users, latency, or corrupted responses.

---

## 1. IgnoreAffinityRequests — "The Sticky Switch"

### Background: What Is Affinity?

- When a user logs in, their session lives on one specific JVM (e.g., `was1`).
- The browser receives a cookie:

```
JSESSIONID=abc123xyz...CloneID:was1
```

- The `CloneID` is a name tag: *"I belong to was1."*
- The plugin reads it and routes the user back to `was1`. This is **session affinity** (stickiness).

### Property Behavior

| Value | Meaning |
|-------|---------|
| `false` (default) | Respect the `CloneID`. Send each user to their own JVM. |
| `true` | Ignore the `CloneID`. Treat every request as new. Pure round-robin. |

### Example Configuration

```ini
IgnoreAffinityRequests="false"
```

> [!WARNING]
> Setting this to `true` in a stateful application causes every request to bounce between JVMs. Sessions are not found, users are logged out on every click — a Sev-1 incident.

### When `true` Might Be Acceptable

- **Load testing** — to spread traffic evenly without stickiness.
- **Stateless REST APIs** — no sessions exist, so affinity is irrelevant.
- **Debugging affinity issues** — temporarily disable to isolate the problem.

> [!TIP]
> Memory trick: **"Ignore = Insanity."** Ignoring affinity means ignoring who the user is — sessions die.

---

## 2. ExtendedHandshake — "The Extra Knock on the Door"

### What It Does

Before forwarding the actual request, the plugin can perform a pre-check: *"WAS, are you alive?"*

| Value | Meaning |
|-------|---------|
| `false` (default) | Send the request directly. `ConnectTimeout` catches dead JVMs. |
| `true` | Perform a handshake first; only forward the request if WAS answers. |

### Example Configuration

```ini
ExtendedHandshake="false"
```

### Why Keep It `false`

- **Latency overhead** — one extra handshake per request × thousands of users adds up.
- **Redundant** — `ConnectTimeout` already detects dead JVMs.
- **Double-checking** the same thing twice wastes time.

> [!TIP]
> Memory trick: **"Two knocks don't make the door open faster."** `ConnectTimeout` already does the job.

---

## 3. ChunkedResponse — "Send It Whole or Send It in Pieces"

### What It Does

HTTP allows a server to stream a large response in chunks — sending data before the full response is ready.

| Value | Meaning |
|-------|---------|
| `false` (default) | Build the complete response, then send it all at once. |
| `true` | Stream the response in chunks as they become available. |

### Example Configuration

```ini
ChunkedResponse="false"
```

### Why Keep It `false`

- **Compatibility** — intermediate proxies or middleware may not handle chunked transfer encoding, leading to corrupted or broken responses (e.g., PDF statements).
- **Document integrity** — banking PDFs and reports must arrive whole.
- **Safer default** — enable only if streaming is genuinely required (large downloads, real-time feeds) **and** the entire delivery chain has been tested.

> [!TIP]
> Memory trick: **"Banks send statements, not water. Deliver it whole."**

---

## Quick Reference Cheat Sheet

| Property | Default | Recommended (Banking) | One-Liner |
|----------|---------|----------------------|-----------|
| `IgnoreAffinityRequests` | `false` | `false` — always | Respect `CloneID` or everyone gets logged out |
| `ExtendedHandshake` | `false` | `false` | Don't knock twice; `ConnectTimeout` already checks |
| `ChunkedResponse` | `false` | `false` | Send responses whole; proxies may break chunks |

---

## Key Takeaway

All three properties default to `false` — and in banking environments, you keep them `false`.

> [!NOTE]
> Defaults exist because someone already broke production learning them.
---
# IBM WebSphere Plugin Properties — Complete Reference Guide

A production-focused reference for tuning the IBM HTTP Server (IHS) WebSphere Plugin (`plugin-cfg.xml`), with recommended settings for high-availability, mission-critical environments (e.g., banking).

---

## 🏦 Architecture Overview

```
User → Browser → IHS (Web Server) → [PLUGIN] → JVM1, JVM2, JVM3 (WAS)
```

- The plugin is the **traffic policeman** between the web server and the WebSphere Application Server (WAS) JVMs.
- These properties are its **rulebook**. Misconfiguration results in errors, timeouts, or session loss.

---

## 1. `IgnoreAffinityRequests` — Sticky Sessions Switch

Controls whether the plugin honors the `CloneID` embedded in the `JSESSIONID` cookie.

| Value   | Behavior                                                        |
|---------|-----------------------------------------------------------------|
| `false` | Respect `CloneID` → route user back to the same JVM (sticky). ✅ |
| `true`  | Ignore `CloneID` → pure Round Robin → users logged out on every click. ❌ |

> [!TIP]
> **Production rule:** Keep `false`. "Ignore = Insanity."

```xml
<Config IgnoreAffinityRequests="false">
```

---

## 2. `LoadBalance` and `LoadBalanceWeight` — Traffic Distribution

### Round Robin (default)

Requests are distributed in fair rotation:

```
Request 1 → JVM1, Request 2 → JVM2, Request 3 → JVM3, Request 4 → JVM1 ...
```

### Weight — Relative Capacity

```xml
<Server Name="was1" LoadBalanceWeight="20">
<Server Name="was2" LoadBalanceWeight="10">
```

- Weight = **relative capacity** of each JVM.
- `was1 : was2 = 20 : 10` → 2:1 traffic ratio.

> [!NOTE]
> **When to use unequal weights:**
> - Larger JVM (more RAM/CPU) on one server → higher weight.
> - Draining a server for maintenance → lower its weight gradually.

> [!WARNING]
> When a weight reaches **0**, the server stops receiving **new** requests — but existing **sticky sessions** still go to it. This is the basis of graceful draining.

---

## 3. `RefreshInterval` — Config Re-Read Frequency

```xml
<Config RefreshInterval="60">
```

- The plugin re-reads `plugin-cfg.xml` every **N seconds** (default: **60**).
- Changes (new JVMs, new weights) are picked up within one refresh cycle.

> [!TIP]
> Default **60s** is fine. During planned maintenance, lower it for faster pickup — but **never below ~10–15s** (extra file reads = overhead).

---

## 4. `ConnectTimeout` — TCP Connection Establishment

```xml
<Server Name="was1" ConnectTimeout="5">
```

| Value | Meaning                                                                  |
|-------|--------------------------------------------------------------------------|
| `0`   | Wait forever — a hung JVM blocks the plugin and pile-ups cascade. ❌      |
| `5`   | If no TCP connection in 5 seconds → mark failed, try the next JVM. ✅     |

> [!WARNING]
> A value of `0` with a hung JVM (memory leak, stuck threads) causes thread pile-up on the web server → total outage.

> [!TIP]
> **Production rule:** 3–10 seconds (typically `5`). **Never `0` in production.**

---

## 5. `ServerIOTimeout` — Waiting for the Response

```xml
<Server Name="was1" ServerIOTimeout="900">
```

- Units: **seconds** in most versions (older fixpacks documented differently — always verify).
- Time the plugin waits for WAS to **finish responding** after connecting.
- `0` = wait forever. ❌

### The Two Phases — Don't Confuse Them

| Phase      | Property         | Analogy                     |
|------------|------------------|-----------------------------|
| Connect    | `ConnectTimeout` | Opening the door            |
| IO         | `ServerIOTimeout`| Waiting for the answer      |

### Common Trap

- Long-running report takes 20 minutes; `ServerIOTimeout=300` → plugin gives up at 5 min, but WAS keeps processing.
- Too **low** → legitimate long requests get killed.
- `0` → hung WAS is never punished.

> [!TIP]
> **Production rule:** Set higher than your longest legitimate request + buffer. Common: **300–900s**. **Never `0`.**

---

## 6. `RetryInterval` — The Penalty Box

```xml
<Server Name="was1" RetryInterval="60">
```

How it works:

1. Request → `was1` → connection refused.
2. Plugin marks `was1` unavailable.
3. All requests fail over to `was2`, `was3`.
4. Every 60 seconds, plugin probes `was1`.
5. When healthy again → back in rotation.

> [!TIP]
> Default **60s** is fine. Shorter = faster recovery detection, but more probe traffic.

---

## 7. Weight Decay — The Graceful Maintenance Trick

Weights **decay** over time: each `RefreshInterval`, each server's effective weight decays relative to others. When a weight hits **0**, the server stops receiving new requests.

### Zero-Downtime Patching Workflow

1. `was1` and `was2` both at weight `2`.
2. Drop `was1` to `0` before patching.
3. New requests gradually shift to `was2` — no sudden flood.
4. Sticky sessions continue to `was1` until users log off naturally.
5. `was1` goes quiet → patch safely → restore weight.

> [!TIP]
> **"Weight 0 = closed for new customers. Old customers can finish their meal."**

---

## 8. Advanced Properties

### a) `MaxConnections` — Concurrent Connections per WAS

```xml
<Transport Hostname="..." Port="9443" MaxConnections="-1">
```

- Max simultaneous connections the plugin holds open to one WAS server.
- `-1` = unlimited (thread pool decides).

> [!WARNING]
> With `-1`, if the web server has more threads than WAS allows, you can overshoot WAS capacity. Tune under load.

### b) `PostSizeLimit` — Maximum POST Body Size

- Maximum size of an incoming POST body (form submissions, file uploads) the plugin forwards.
- Protects the web server from oversized malicious uploads.

> [!TIP]
> Set above your largest legitimate upload, below "attack size."

### c) `CloneID` — The Heart of Stickiness

- Each WAS JVM has a unique `CloneID` embedded in the `JSESSIONID`.
- Plugin matches `CloneID` → routes back to the same JVM.
- **Missing/blank CloneID** → no affinity → silent Round Robin fallback → mysterious logouts.

### d) `WaitForContinue` — HTTP 100-Continue

- Plugin asks WAS: "Big request coming — ready?" before sending.
- Adds an extra round trip (latency).
- Keep `false` unless a specific application requires it.

### e) Failover Retry Behavior (`FailoverToNextServer` / `RetryOnError`)

- If a JVM fails mid-request, the plugin retries on the next JVM.
- Safe for GETs; **risky for POSTs** — a payment POST could be submitted twice.

> [!WARNING]
> Make payment/transaction endpoints **idempotent** at the application level.

### f) Sticky Affinity After Failover

- User pinned to `was1` → `was1` dies → plugin fails over to `was2`.
- Session survives **only if** HTTP Session replication is enabled (memory-to-memory or database persistence).
- Without replication, the user must log in again.

> [!NOTE]
> Plugin failover only works if the session *traveled*. Always pair with session replication.

---

## 📋 Master Cheat Sheet

| Property                 | Default        | What It Does                     | Production Setting                    |
|--------------------------|----------------|----------------------------------|---------------------------------------|
| `IgnoreAffinityRequests` | `false`        | Respect CloneID or not           | `false`                               |
| `LoadBalance`            | Round Robin    | Traffic distribution method      | Round Robin                           |
| `LoadBalanceWeight`      | varies         | Relative capacity per JVM        | Equal for equal JVMs; `0` to drain    |
| `RefreshInterval`        | 60s            | Config re-read frequency         | 60 (lower during maintenance)         |
| `ConnectTimeout`         | 5s             | TCP connection establishment     | 3–10s, **never 0**                    |
| `ServerIOTimeout`        | 0 (forever)    | Time waiting for full response   | 300–900s, **never 0**                 |
| `RetryInterval`          | 60s            | Penalty box for dead JVM         | 60s                                   |
| `MaxConnections`         | `-1`           | Concurrent connections per JVM   | Tune under load                       |
| `PostSizeLimit`          | varies         | Max upload size                  | Limit it                              |

---

## 🧠 End-to-End Example

Follow one user, "venky," through the plugin:

1. venky clicks **Pay Bill** → plugin reads her `JSESSIONID` CloneID → routes to `was1` (`IgnoreAffinityRequests=false`).
2. `was1`'s `LoadBalanceWeight` is healthy → she's allowed in.
3. Plugin opens a connection — `was1` answers in 2s (`ConnectTimeout=5`, passed).
4. Plugin waits for the payment response — done in 3s (`ServerIOTimeout=900`, plenty).
5. Meanwhile, an admin set `was2`'s weight to `0` for patching → plugin re-reads config every `RefreshInterval=60s` → new users drift to `was1`/`was3`.
6. Tonight `was1` crashes → plugin fails venky's request over to `was3` → `was1` sits in the penalty box for `RetryInterval=60s` → probe finds it healthy → back in rotation.
