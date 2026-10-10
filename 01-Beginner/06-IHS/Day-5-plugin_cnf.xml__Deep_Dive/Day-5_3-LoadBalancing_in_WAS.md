# Load Balancing in WebSphere — A Complete Guide

> [!NOTE]
> This guide explains WebSphere load balancing from first principles using the **WebSphere Plugin** (`plugin-cfg.xml`), covering Round Robin, Weighted Round Robin, Random, and automatic failover.

---

## 1. What Is Load Balancing? (The Foundation)

### The Problem
- One server (JVM) runs the entire application.
- A traffic spike (e.g., 500 concurrent users) overloads it.
- The server slows down or crashes → errors → downtime.

### The Solution
- Run **multiple JVMs** (e.g., 4) behind a traffic distributor.
- Each incoming request is routed to a different JVM.
- No single server gets overloaded.

### Terminology Map

| Real World | WebSphere World |
|---|---|
| Traffic policeman / Queue manager | WebSphere Plugin (load balancer) |
| Teller counters | JVMs (application servers) |
| The list of counters the policeman knows | `plugin-cfg.xml` |

### What Is the Plugin?
- A small module running **inside the web server** (e.g., IBM HTTP Server / Apache).
- Request flow:

```
Browser → Web Server → Plugin → picks one JVM → request handled
```

- The plugin reads `plugin-cfg.xml` to know which JVMs exist and how to distribute traffic.

---

## 2. Method 1 — Round Robin

### The Rule
> Send requests one by one to each JVM in a circle. When the last JVM is reached, start again from the first.

### Example

```
Request 1 → JVM1
Request 2 → JVM2
Request 3 → JVM3
Request 4 → JVM4
Request 5 → JVM1   ← circle repeats
Request 6 → JVM2
```

### When It Works Well ✅
- All JVMs have **equal hardware** (same CPU, RAM).
- All requests take **similar time** (e.g., simple lookups).
- It is the **default and most common** setting — predictable.

### When It Fails ❌
- Uneven hardware: JVM1 is old, JVM2–4 are new and fast.
- Round Robin still sends 25% of traffic to JVM1, which can only handle 15%.
- Result: JVM1 chokes while other servers idle.

> [!TIP]
> Fix uneven hardware with **Weighted Round Robin** (next section).

---

## 3. Method 2 — Weighted Round Robin

### The Idea
> Same circle, but strong servers get **more** requests, weak servers get **fewer**.

### Configuration

In `plugin-cfg.xml`, add a `LoadBalanceWeight` to each server:

```xml
<Server Name="was1_PaymentCluster_server1"
        CloneID="1a2b3c"
        LoadBalanceWeight="10"
        ConnectTimeout="5">
</Server>

<Server Name="was2_PaymentCluster_server1"
        CloneID="4d5e6f"
        LoadBalanceWeight="30"
        ConnectTimeout="5">
</Server>

<Server Name="was3_PaymentCluster_server1"
        CloneID="7g8h9i"
        LoadBalanceWeight="30"
        ConnectTimeout="5">
</Server>
```

### Reading the Config

| Attribute | Meaning |
|---|---|
| `Name` | Which server this entry belongs to |
| `CloneID` | Unique ID of each JVM clone (used for session affinity) |
| `LoadBalanceWeight` | How "strong" the server is — bigger number = more traffic |
| `ConnectTimeout` | Seconds the plugin waits before giving up on a connection |

### Traffic Distribution (weights 10 / 30 / 30)

- Total = 70
- JVM1 → ~14% of traffic
- JVM2 → ~43%
- JVM3 → ~43%

> [!TIP]
> **Migration pattern:** During a hardware refresh, set low weights on old servers and high weights on new ones. Old servers handle minimal load while staying online, then get decommissioned later — zero downtime, zero user impact.

---

## 4. Method 3 — Random

### The Rule
> The plugin picks any JVM at random for each request.

### Example

```
Request 1 → JVM3
Request 2 → JVM1
Request 3 → JVM3   ← same JVM again, by luck
```

### Why Banks Avoid It ❌
- **Unpredictable** — bad luck can send 70% of requests to one JVM.
- **No control** — no fairness guarantee.
- Banks value predictability; random = risk.

> [!NOTE]
> Random is rarely used in production. Prefer Round Robin or Weighted Round Robin.

---

## 5. Failover — What Happens When a JVM Dies

This is where the plugin earns its keep.

### Timeline (JVM2 dies at 2:00 PM)

```
2:00 PM   User request → Plugin tries JVM2 → CONNECTION REFUSED
2:00 PM   Plugin retries the next JVM in the list
          → Request goes to JVM3 instead (user never notices)
2:00 PM   Plugin marks JVM2 "sick" for RetryInterval (default 60 sec)
2:01 PM+  ALL requests skip JVM2 → only JVM1, JVM3, JVM4 get traffic
2:02 PM   Plugin quietly probes JVM2 ("Are you alive?")
          If alive  → traffic flows to it again automatically
          If dead   → keeps skipping it
```

### Key Points
- **Automatic failover** — no human intervention required.
- **The user sees no error** — the plugin retries on another JVM.
- After `RetryInterval` (default: 60 sec), the plugin re-tests the dead JVM.
- When the JVM recovers → it **rejoins the rotation automatically**.

### Real-World Scenario
- A payment JVM crashes at 2 PM on payday.
- The plugin shifts all traffic to the remaining 3 JVMs instantly.
- Users keep making payments; no tickets raised.
- Admin fixes the JVM at leisure; it rejoins the cluster.

---

## 6. Quick Revision Card

| Concept | One-Line Memory Hook |
|---|---|
| Load Balancer | The queue manager at the bank door |
| Plugin | Software policeman inside the web server |
| `plugin-cfg.xml` | The policeman's notebook — list of JVMs + rules |
| Round Robin | Go in a circle, everyone equal |
| Weighted RR | Strong get more, weak get less |
| Random | Dice roll — avoid in production |
| Failover | JVM dies → plugin skips it → retries after 60 sec |
| `RetryInterval` | How long the plugin avoids a "sick" JVM |
| `LoadBalanceWeight` | Traffic share per JVM (bigger = more traffic) |
| `CloneID` | Unique ID of each JVM clone |

---
# View/Change Load Balancing method

```
Admin Console
  → Servers → Clusters → WebSphere Application Server Clusters
    → Click PaymentCluster
      → "Custom properties" or Cluster settings
```