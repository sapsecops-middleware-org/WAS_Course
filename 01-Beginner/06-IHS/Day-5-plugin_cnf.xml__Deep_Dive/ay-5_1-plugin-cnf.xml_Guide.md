# WebSphere plugin-cfg.xml — Complete Reference Guide

## Complete plugin-cfg.xml skeleton (put it all together)
```
<?xml version="1.0" encoding="UTF-8"?>
<Config RefreshInterval="60" ...>

  <!-- WHERE TO WRITE PLUGIN LOGS -->
  <Log LogLevel="Error"
       Name="/opt/IBM/HTTPServer/Plugins/logs/webserver1/http_plugin.log"/>

  <!-- WHICH HOSTNAMES WE HANDLE -->
  <VirtualHostGroup Name="NetBanking_Hosts">
      <VirtualHost Name="www.citibank.co.in:443"/>
      <VirtualHost Name="www.citibank.co.in:80"/>
  </VirtualHostGroup>

  <!-- WHICH URLs GO TO WAS -->
  <UriGroup Name="NetBanking_URIs">
      <Uri AffinityCookie="JSESSIONID"
           AffinityURLIdentifier="jsessionid"
           Name="/NetBanking/*"/>
  </UriGroup>

  <!-- THE WAS CLUSTER -->
  <ServerCluster Name="PaymentCluster"
                 LoadBalance="Round Robin"
                 RetryInterval="60">

      <Server Name="was1_PaymentCluster_server1"
              CloneID="1a2b3c"
              ConnectTimeout="5"
              ServerIOTimeout="60">
          <Transport Hostname="was1.citi.internal" Port="9080" Protocol="http"/>
      </Server>

      <Server Name="was2_PaymentCluster_server1"
              CloneID="4d5e6f"
              ConnectTimeout="5"
              ServerIOTimeout="60">
          <Transport Hostname="was2.citi.internal" Port="9080" Protocol="http"/>
      </Server>

      <PrimaryServers>
          <Server Name="was1_PaymentCluster_server1"/>
          <Server Name="was2_PaymentCluster_server1"/>
      </PrimaryServers>

  </ServerCluster>

  <!-- THE CONNECTOR — TIES EVERYTHING TOGETHER -->
  <Route ServerCluster="PaymentCluster"
         UriGroup="NetBanking_URIs"
         VirtualHostGroup="NetBanking_Hosts"/>

</Config>
```

## Overview

`plugin-cfg.xml` is the routing configuration file that enables IBM HTTP Server (IHS) to forward dynamic requests to WebSphere Application Server (WAS). Without it, IHS can only serve static content (HTML, images, CSS).

### Request Flow

```
Browser → Firewall → IHS → [plugin-cfg.xml decides] → WAS JVM → App → Database
```

### File Locations

| Location | Path | Purpose |
|---|---|---|
| IHS (active copy) | `/opt/IBM/HTTPServer/Plugins/config/webserver1/plugin-cfg.xml` | The copy the plugin actually reads |
| WAS (generated) | `/opt/IBM/WebSphere/AppServer/profiles/Dmgr01/config/cells/<CellName>/nodes/<WebServerNode>/servers/webserver1/plugin-cfg.xml` | Where WAS generates the file |

> [!NOTE]
> After deploying a new application, you must **Generate Plugin** in the WAS admin console **and** propagate the file to the IHS server. Generating without propagating changes nothing.

---

## Document Structure

```xml
<Config>                  <!-- Root. Global settings -->
  <Log>                   <!-- Plugin log location and level -->
  <VirtualHostGroup>      <!-- Hostnames/ports the plugin claims -->
  <ServerCluster>         <!-- WAS cluster and its JVMs -->
     <Server>             <!-- One JVM -->
       <Transport>        <!-- Host + port + protocol -->
  <UriGroup>              <!-- URL patterns routed to WAS -->
  <Route>                 <!-- URL + Hostname → Cluster mapping -->
</Config>
```

Routing logic: **WHO** (VirtualHost) + **WHAT URL** (UriGroup) → **WHICH cluster** (ServerCluster), tied by **Route**.

---

## `<Config>` — Root Element

Holds global plugin behavior settings.

```xml
<Config ASDisableNagle="false"
        AcceptAllContent="false"
        AppServerPortPreference="HostHeader"
        ChunkedResponse="false"
        IgnoreDNSFailures="false"
        RefreshInterval="60"
        ResponseChunkSize="64"
        VHostMatchingCompat="false">
```

### Key Attributes

| Attribute | Meaning | Notes |
|---|---|---|
| `RefreshInterval="60"` | Plugin re-reads the file every 60 seconds | Regenerate + propagate → **no IHS restart needed** |
| `AcceptAllContent="false"` | Only accept POST bodies WAS understands | Keep `false` — blocks malformed requests |
| `ResponseChunkSize="64"` | Buffers 64 KB of response before sending | Tune for large downloads |
| `IgnoreDNSFailures="false"` | Unresolvable WAS hostname → fail | Set `true` only if hosts may be temporarily down |
| `AppServerPortPreference="HostHeader"` | Port used for redirects | Default is fine |

> [!TIP]
> **Zero-downtime deployments:** With `RefreshInterval="60"`, deploy a new app by regenerating the file, propagating it to IHS, and waiting up to 60 seconds — no IHS restart required.

---

## `<Log>` — Plugin Logging

```xml
<Log LogLevel="Error"
     Name="/opt/IBM/HTTPServer/Plugins/logs/webserver1/http_plugin.log"/>
```

- **Name** — log file path; first stop when troubleshooting routing failures.
- **LogLevel** — verbosity level.

### LogLevel Ladder (noisy → quiet)

```
Trace → Debug → Detail → Warn → Error → Off
```

| Level | When to Use |
|---|---|
| `Error` | Production default |
| `Warn` | Post-incident investigation |
| `Detail` | Troubleshooting — logs every routing decision |
| `Trace` | Deep debugging — **never in production** |

> [!WARNING]
> Leaving `LogLevel="Trace"` in production fills the disk, which prevents IHS from writing logs and can stop the plugin from routing entirely — a full portal outage. **Always revert to `Error` after debugging.**

---

## `<VirtualHostGroup>` — Hostname Filter

Defines which `hostname:port` combinations the plugin routes to WAS. Everything else is handled by IHS itself.

```xml
<VirtualHostGroup Name="NetBanking_Hosts">
    <VirtualHost Name="www.citibank.co.in:443"/>
    <VirtualHost Name="www.citibank.co.in:80"/>
    <VirtualHost Name="corporate.citibank.co.in:443"/>
    <VirtualHost Name="*:80"/>
</VirtualHostGroup>
```

### New Hostname Checklist

- ✅ DNS entry
- ✅ IHS virtual host
- ✅ WAS virtual host
- ✅ `VirtualHostGroup` in `plugin-cfg.xml` ← the one most often forgotten

> [!TIP]
> A missing `VirtualHost` entry results in 404s because the plugin does not recognize the hostname.

---

## `<ServerCluster>` — Cluster Definition

Defines the WAS cluster, its JVMs, and load balancing behavior.

```xml
<ServerCluster CloneID="14fd3b2a"
               LoadBalance="Round Robin"
               Name="PaymentCluster"
               PostSizeLimit="-1"
               RemoveSpecialHeaders="true"
               RetryInterval="60">
```

### Attributes

| Attribute | Meaning | Why It Matters |
|---|---|---|
| `Name` | Cluster name | Must exactly match the WAS ND cluster name |
| `LoadBalance="Round Robin"` | Requests distributed 1→2→3→4→1... | Predictable, fair load spread |
| `RetryInterval="60"` | Skip a down JVM for 60 seconds | Prevents hammering dead servers |
| `CloneID` | Used for session affinity | Keeps logged-in users on the same JVM |
| `RemoveSpecialHeaders="true"` | Strips internal headers before forwarding | Prevents header spoofing |
| `PostSizeLimit="-1"` | Max POST body size (`-1` = unlimited) | Required for file uploads |

> [!NOTE]
> Under heavy load, Round Robin ensures even distribution (e.g., 4 JVMs × 12,500 sessions each). Random balancing can overload a single JVM, causing `OutOfMemoryError` and cascade failure.

---

## `<Server>` and `<Transport>` — Individual JVMs

Each JVM is a `<Server>` with one or more `<Transport>` entries describing how to reach it.

```xml
<Server CloneID="1a2b3c" ConnectTimeout="5"
        Name="was1_PaymentCluster_server1"
        ServerIOTimeout="60">
    <Transport Hostname="was1.citi.internal" Port="9080" Protocol="http"/>
    <Transport Hostname="was1.citi.internal" Port="9443" Protocol="https"/>
</Server>
```

| Attribute | Meaning |
|---|---|
| `CloneID` | JVM ID used for session affinity |
| `ConnectTimeout="5"` | Max 5 seconds to open a connection |
| `ServerIOTimeout="60"` | Max 60 seconds waiting for the JVM's response |
| `Port 9080` | Plain HTTP to WAS |
| `Port 9443` | HTTPS to WAS — preferred for security-sensitive apps |

> [!TIP]
> Analogy: `ConnectTimeout` = how long you wait for someone to pick up the phone; `ServerIOTimeout` = how long you wait for their answer.

---

## `<PrimaryServers>` and `<BackupServers>` — DR Pattern

```xml
<PrimaryServers>
    <Server Name="was1_PaymentCluster_server1"/>
    <Server Name="was2_PaymentCluster_server1"/>
</PrimaryServers>

<BackupServers>
    <Server Name="was3_PaymentCluster_server1"/>
</BackupServers>
```

- **PrimaryServers** — handle normal daily traffic.
- **BackupServers** — idle until **all** primaries fail; then silently take over.

> [!NOTE]
> This is the standard DR pattern inside the plugin: no DNS change, no restart — backups activate automatically on failure.

---

## Quick Revision Table

| Element | One-Line Meaning |
|---|---|
| `<Config>` | Global rules; `RefreshInterval="60"` = no restart needed |
| `<Log>` | Plugin's log; production = `Error` |
| `<VirtualHostGroup>` | Hostnames/ports the plugin claims as "mine" |
| `<ServerCluster>` | WAS cluster + JVMs + load balancing |
| `<Server>` | One JVM, with timeouts and CloneID |
| `<Transport>` | Host + port + protocol to reach the JVM |
| `<PrimaryServers>` | Normal traffic takers |
| `<BackupServers>` | Last-resort / DR JVMs |

---

## Interview One-Liners

- `plugin-cfg.xml` is the routing map that lets IHS forward requests to WAS.
- `RefreshInterval` lets the plugin pick up regenerated config without restarting IHS.
- Never run `LogLevel=Trace` in production — it fills disk and can take the portal down.
- PrimaryServers handle traffic; BackupServers only activate when all primaries fail.
---

# IBM WebSphere plugin-cfg.xml Continue — `<Server>`, `<Transport>`, `<UriGroup>` & `<Route>` Deep-Dive

> [!NOTE]
> **The Call Centre Analogy (keep this in your head):**
>
> ```
> Customer calls → Receptionist (IHS) answers
>                 ↓
> She reads her manual (plugin-cfg.xml):
>
> VirtualHostGroup → "Which phone numbers do we answer?"
> ServerCluster    → "Which departments exist?"
> UriGroup         → "Which topics go to which department?"   ← TODAY
> Route            → "Phone number + topic → department"     ← TODAY
> Server           → "Actual people in that department"      ← TODAY
> Transport        → "Their extension number"                ← TODAY
> ```

---

## 1. `<Server>` — One Individual WAS JVM

### What It Is

A `<Server>` is a group. But a group of what? Of **individual JVMs**. Each JVM gets its own `<Server>` block inside the cluster.

### Example

```xml
<ServerCluster Name="PaymentCluster" LoadBalance="Round Robin" RetryInterval="60">

    <Server CloneID="1a2b3c"
            ConnectTimeout="5"
            MaxConnections="-1"
            Name="was1_PaymentCluster_server1"
            ServerIOTimeout="60"
            WaitForContinue="false">
    </Server>

    <Server CloneID="4d5e6f"
            ConnectTimeout="5"
            MaxConnections="-1"
            Name="was2_PaymentCluster_server1"
            ServerIOTimeout="60">
    </Server>

</ServerCluster>
```

This cluster has **two JVMs** — one on `was1`, one on `was2`.

### Attribute Reference

| Attribute | Meaning in One Line |
|---|---|
| `Name` | Nickname of this JVM. **Must match exactly** what WAS calls it. |
| `CloneID` | Unique ID for this JVM. Used for session affinity (sticking a user to the same JVM). |
| `ConnectTimeout="5"` | Wait max **5 seconds** to connect. If I can't connect → mark this JVM as down and try the next one. |
| `ServerIOTimeout="60"` | After connecting, wait max **60 seconds** for a response. Protects against hung, slow requests. |
| `Connections="-1"` | `-1` = no limit. Or set a number (e.g. `200`) to cap connections to this JVM. |
| `WaitForContinue="false"` | Don't wait for a `100 Continue` HTTP header before sending POST data. Leave it `false` — it's faster. |

> [!TIP]
> A timeout is what enables failover. **No timeout = no failover.** Always set `ConnectTimeout="5"` in production.

### 🏦 Real Bank Scenario — Protecting a Weak JVM

Your payment cluster has 4 JVMs. JVM3 runs on an older box with less RAM.

- **Fix:** Set `MaxConnections="150"` on JVM3, leave `-1 on the others.
- Now even if Round Robin sends heavy traffic to JVM3, the plugin caps it at 150 connections. JVM3 can't run out of memory and die.

- **Without it:** JVM3 gets 500 simultaneous connections → `OutOfMemory` → dies → live payment sessions lost.

### ⚠️ Real Production Failure — The Missing Timeout

- `ConnectTimeout` left at default `0` (wait forever).
- One JVM hung — not down, just frozen.
- Plugin tried to connect to it **forever**. Never gave up. Never marked it down. No failover.
-Result:** 300 users staring at spinning wheels for 8 minutes.

---

## 2. `<Transport>` — Which Port to Talk To the JVM On

### What It Is

The plugin now knows the JVM exists. But how do I physically reach it? Answer: **hostname + port + protocol**.

### Example

```xml
<Server Name="was1_PaymentCluster_server1" CloneID="1a2b3c" ...>

    <Transport Hostname="was1.citi.internal"
               Port="9080"
               Protocol="http"/>

    <Transport Hostname="was1.citi.internal"
               Port="9443"
               Protocol="https"/>

</Server>
```

### Attribute Reference

| Attribute | Meaning |
|---|---|
| `Hostname` | The server name or IP where WAS is running (internal name — users never see it) |
| `Port` | WAS's port. Defaults: `9080` = HTTP, `9443` = HTTPS |
| `Protocol` | `http` or `https` |

### Why TWO Transport Entries?

Because there are two ways to connect IHS to WAS:

**Option A — SSL Offload (most common in banks):**

```
Customer → [HTTPS 443] → IHS decrypts → [HTTP 9080] → WAS
```

**Option B — End-to-End SSL (high security):**

```
Customer → [HTTPS 443] → IHS → [HTTPS 9443] → WAS
```

The two `<Transport>` blocks let the plugin pick the right path depending on the incoming request.

### 🏦 Bank Context

- **SSL offload:** HTTPS ends at IHS. Inside the DMZ, traffic runs on plain HTTP 9080. This is safe because the internal network is a private, firewalled LAN.
- **Why banks love it:** WAS JVMs don't burn CPU decrypting SSL. That CPU goes to the application instead.
- **Exception:** Very sensitive apps (core banking, trading systems) demand end-to-end HTTPS all the way to WAS. More secure, but more CPU.

> [!TIP]
> **Interview one-liner:** *"SSL offload means encryption terminates at the web server; the plugin talks to WAS over HTTP 9080."*

---

## 3. `<UriGroup>` — Which URLs Should Go to WAS

### The Key Idea

IHS serves two kinds of content:

- **Static files** — images, CSS, HTML sitting on disk. IHS serves these itself. Fast, no WAS involved.
- **Dynamic requests** — anything needing Java logic. These must go to WAS.

`<UriGroup>` is the list that says: *"These URL patterns → forward to WAS. Everything else → serve from disk."*

### Example

```xml
<UriGroup Name="NetBanking_URIs">
    <Uri AffinityCookie="JSESSIONID"
         AffinityURLIdentifier="jsessionid"
         Name="/NetBanking/*"/>
    <Uri Name="/PaymentPortal/*"/>
    <Uri Name="/AccountServices/*"/>
</UriGroup>
```

### How to Read It

- `/NetBanking/*` → any URL starting with `/NetBanking/` goes to WAS. The `*` means "everything after it."
- `/images/logo.png` is **not** in the list → IHS serves it straight from disk.

### The Two Affinity Attributes (⭐ Interview Favourites)

| Attribute | Plain English |
|---|---|
| `AffinityCookie="JSESSIONID"` | "Look for a cookie called `JSESSIONID` in the request. Use it to stick the user to their JVM." |
| `AffinityURLIdentifier="jsessionid"` | "If there's no cookie, look in the URL for `;jsessionid=...` instead." |

> [!NOTE]
> This is where **session affinity** begins. This tag is how the plugin starts tracking which user belongs to which JVM.

### 🏦 Real Scenario — Why This Filtering Matters

At Citi:

- App server handles `/NetBanking/*` (dynamic).
- Homepage `/index.html`, `/css/`, `/images/` — all static, served by IHS from disk.

If this filtering weren't precise:

- Every CSS file, every image would hit WAS.
- WAS threads wasted on serving pictures.
- At 50K users → **5 million unnecessary WAS hits per hour**.

Precise UriGroups keep WAS doing what it's good at: **running Java**.

---

## 4. `<Route>` — The Final Connector

### What It Is

The simplest block in the whole file — but the one that makes everything work. It connects exactly **three** things:

```xml
<Route ServerCluster="PaymentCluster"
       UriGroup="NetBanking_URIs"
       VirtualHostGroup="NetBanking_Hosts"/>
```

### Read It as a Sentence

> "When a request comes in for a host in `NetBanking_Hosts`, and the URL matches something in `NetBanking_URIs` → send it to `PaymentCluster`."

### Three Ingredients

| Ingredient | Question It Answers |
|---|---|
| `VirtualHostGroup` | **Who** is asking? (which website name) |
| `UriGroup` | **What** are they asking for? (which URL) |
| `ServerCluster` | **Where** should it go? (which WAS cluster) |

All three must match → the request is forwarded. That's the whole magic.

---

## 5. Full Flow — Put It All Together

Trace one real request end to end:

```
User types: https://www.citibank.co.in/NetBanking/login
```

1. **IHS receives** the request on port 443.
2. **Plugin loads:** "Is `www.citibank.co.in` in a VirtualHostGroup?" → Yes: `NetBanking_Hosts` ✔
3. **"Does the URL match a UriGroup?"** → `/NetBanking/*` matches `NetBanking_URIs` ✔
4. **"Which Route connects these two?"** → Route says: go to `PaymentCluster` ✔
5. **"Which JVM in PaymentCluster?"** → Round Robin picks one. Check session affinity via `CloneID`/`JSESSIONID`.
6. **"How do I reach it?"** → Transport says: `was1.citi.internal:9080`, `http`
7. **Connect** (`ConnectTimeout=5s`), send request, wait for response (`ServerIOTimeout=60s`).
8. **WAS replies** → IHS returns the page to the user. Done.

> [!NOTE]
> If any step fails to match → IHS handles it itself (static file, 404, etc.).

---

## 6. Memory Summary

| Tag | One-Line Memory Hook |
|---|---|
| `<Server>` | "One JVM. Its timeouts and limits." |
| `<Transport>` | "Hostname + port + protocol = how to reach it." |
| `<UriGroup>` | "URL patterns that go to WAS. Everything else = static." |
| `<Route>` | "VirtualHost + URI → Cluster. The glue." |

---
# 🗺️ Full plugin-cfg.xml Flow — Simple Diagram
```
REQUEST: https://www.citibank.co.in/NetBanking/login
                  ↓
        ┌─────────────────────┐
        │   VirtualHostGroup  │  ← Is "www.citibank.co.in:443" listed? YES ✅
        └─────────────────────┘
                  ↓
        ┌─────────────────────┐
        │      UriGroup       │  ← Does "/NetBanking/*" match? YES ✅
        └─────────────────────┘
                  ↓
        ┌─────────────────────┐
        │       Route         │  ← VirtualHostGroup + UriGroup → PaymentCluster
        └─────────────────────┘
                  ↓
        ┌─────────────────────────────────────┐
        │         ServerCluster               │
        │   (Round Robin between 4 JVMs)      │
        │                                     │
        │  JVM1 ← Transport: was1:9080       │
        │  JVM2 ← Transport: was2:9080       │
        │  JVM3 ← Transport: was3:9080       │
        │  JVM4 ← Transport: was4:9080       │
        └─────────────────────────────────────┘
                  ↓
           Request reaches WAS ✅
```