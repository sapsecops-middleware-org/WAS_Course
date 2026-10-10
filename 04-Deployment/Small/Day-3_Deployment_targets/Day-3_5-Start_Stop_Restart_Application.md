# Lesson 3 — Part 6: Starting, Stopping, Restarting at Each Level

> **This is where most juniors get confused.**
> You can start/stop at different levels — and they mean **different things**.
> Picking the wrong level = unnecessary outage, or worse, thinking you fixed something you didn't.

---

## 🎯 The Three Levels

```text
Level 1: CLUSTER        → everything stops (users hit an outage)
Level 2: ONE SERVER     → other server covers (users see nothing)
Level 3: APPLICATION    → JVM stays up, only the app reloads (fastest)
```

> 🧠 **Memory hook:** Always operate at the **lowest level that solves the problem**.
> Stopping the whole cluster to restart one app is like demolishing the building to fix one apartment.

---

# 🔴 Level 1 — Start/Stop the Entire CLUSTER

**Admin Console:**

```text
Servers → Clusters → WebSphere Application Server Clusters
  → Select: DigiStackCluster
  → [Start]  OR  [Stop]
```

**Effect:**

```text
ALL cluster members start/stop together.
AppServer01 starts AND AppServer02 starts.
All applications running on the cluster start with them.
```

**When to use:**

- → Full cluster restart (after config changes)
- → Emergency shutdown of entire banking app

**wsadmin:**

```python
# Start entire cluster
cluster = AdminControl.completeObjectName(
    'cell=DigiStackCell01,type=Cluster,name=DigiStackCluster,*'
)
AdminControl.invoke(cluster, 'start')

# Stop entire cluster
AdminControl.invoke(cluster, 'stop')

# Check cluster state
state = AdminControl.getAttribute(cluster, 'state')
print("Cluster state: " + state)
```

**Possible states:**

```text
rsRunning        → all members up
rsStopped        → all members down
rsPartialStart   → some up, some not — INVESTIGATE THIS
```

> ⚠️ **`rsPartialStart` is a red flag.** One member failed to start — go check that server's `SystemOut.log` immediately.
>
> 🧠 **Memory hook:** Level 1 = **everyone down**. Users get errors. Only for planned maintenance or emergencies.

---

# 🟡 Level 2 — Start/Stop ONE Cluster Member

**Admin Console:**

```text
Servers → Server Types → WebSphere Application Servers
  → Select: Node01:AppServer01
  → [Start] OR [Stop]
```

**Effect:**

```text
ONLY AppServer01 stops.
AppServer02 keeps running.
IHS plugin detects AppServer01 is down → sends all traffic to AppServer02.
Customers see NO interruption.
```

**When to use:**

- → Rolling restart (restart one at a time)
- → Debugging one specific server
- → Maintenance on one node only

**wsadmin:**

```python
# Stop only AppServer01 (keep AppServer02 running)
server1 = AdminControl.completeObjectName(
    'cell=DigiStackCell01,node=Node01,type=Server,name=AppServer01,*'
)
AdminControl.invoke(server1, 'stop')
print("AppServer01 stopped. AppServer02 still serving customers.")

# Start AppServer01 again
AdminControl.invoke(server1, 'start')
print("AppServer01 back online.")
```

> 🧠 **Memory hook:** Notice the MBean query includes **`node=Node01`** — you must say WHICH node the server lives on. This is the key difference from Level 1.

### 🔁 The Rolling Restart Pattern (memorize this!)

```text
1. Stop AppServer01        → IHS routes 100% to AppServer02
2. Do maintenance / restart
3. Start AppServer01       → verify it's healthy
4. Stop AppServer02        → IHS routes 100% to AppServer01
5. Do maintenance / restart
6. Start AppServer02       → cluster fully restored
```

**Customers never notice.** This is the standard procedure for applying fixes in production without an outage.

> ⚠️ **Prerequisite:** The IHS plugin must be configured (Lesson 2) and `plugin-cfg.xml` must be current — otherwise IHS keeps sending traffic to a dead server.

---

# 🟢 Level 3 — Start/Stop the APPLICATION (Without Restarting Servers)

**Admin Console:**

```text
Applications → WebSphere Enterprise Applications
  → Check: digistack-bank-v8
  → [Stop]  then  [Start]
```

**Effect:**

```text
The JVM keeps running.
ONLY the application is stopped and reloaded.
Faster than full server restart.
Sessions are lost (customers get logged out).
```

**When to use:**

- → Quick redeploy during maintenance window
- → App is hung but server is fine
- → Config change that only needs app restart

**wsadmin:**

```python
# Stop application (server stays running)
appMgr = AdminControl.queryNames(
    'cell=DigiStackCell01,node=Node01,type=ApplicationManager,*'
)
AdminControl.invoke(appMgr, 'stopApplication', 'digistack-bank-v8')
print("Application stopped. Server still running.")

# Start application
AdminControl.invoke(appMgr, 'startApplication', 'digistack-bank-v8')
print("Application restarted.")
```

> 🧠 **Memory hook:** Level 3 = **app only**. The JVM, connection pools, and web container all stay warm. Restart takes seconds, not minutes.

### ⚠️ Level 3 limitation:

```text
App restart fixes:    stale app state, hung app code, new EAR redeploy
App restart does NOT: apply JVM args, ports, thread pool changes,
                      OR anything requiring a JVM restart
```

---

# 🌳 Decision Tree — Which Level to Use?

Answer these questions **in order**:

```text
Is the WHOLE cluster behaving badly?
  YES → Stop/Start the CLUSTER

Is only ONE server behaving badly?
  YES → Stop/Start that ONE SERVER ONLY
        (other server keeps serving customers)

Is the server fine but the APP is stuck?
  YES → Stop/Start the APPLICATION
        (fastest, server doesn't restart)

Did you change WAS configuration (ports, JVM args, etc.)?
  YES → Need full SERVER restart (not just app restart)

Did you just redeploy a new EAR version?
  YES → Stop/Start APPLICATION
        (unless JVM settings changed too)
```

### The tree as a table:

| Situation | Level | User impact |
|-----------|-------|-------------|
| Whole app down for planned maintenance | 1 — Cluster | Full outage (planned) |
| AppServer01 has memory leak | 2 — Single server | None — AppServer02 takes over |
| App hung but server healthy | 3 — Application | Sessions lost, quick recovery |
| Changed JVM heap size | Full server restart | None if rolling restart |
| Deployed new EAR version | 3 — Application | Sessions lost (maintenance window) |
| Changed server port | 2 — Server (rolling) | None if rolling restart |

---

## ⚡ Speed Comparison

| Action | Typical time | Sessions |
|--------|-------------|----------|
| Cluster stop → start | Minutes (both JVMs full cycle) | All lost |
| Single server stop → start (rolling) | Minutes per server, zero-downtime overall | Lost on that server only |
| App stop → start | Seconds | Lost for app users |

---

# 📊 Console vs wsadmin — Same Actions

| Action | Console | wsadmin |
|--------|---------|---------|
| Start/stop cluster | Clusters → [Start]/[Stop] | `AdminControl.invoke(cluster, 'start'/'stop')` |
| Check cluster state | Cluster status column | `AdminControl.getAttribute(cluster, 'state')` |
| Start/stop one server | Servers → select → [Start]/[Stop] | `AdminControl.invoke(server, 'start'/'stop')` |
| Start/stop app | Enterprise Applications → [Start]/[Stop] | `AdminControl.invoke(appMgr, 'startApplication'/'stopApplication', name)` |

---

## ✅ Golden Rules

1. **Lowest level wins.** App restart > server restart > cluster restart. Never go bigger than needed.
2. **Rolling restart for zero downtime** — one member at a time, verify health before touching the next.
3. **App restart ≠ config restart.** JVM args, ports, and thread pools need a **server** restart.
4. **`rsPartialStart` means one member failed** — check logs before declaring success.
5. In the MBean query for a server, you **must include `node=`** — servers exist per node.
6. Level 3 restart **loses sessions** — plan it for a maintenance window or ensure session replication is configured.
7. IHS must know the member is down — **keep `plugin-cfg.xml` regenerated** so traffic reroutes automatically.

---

## 🧩 Quick Self-Test (Try Answering!)

1. You need to deploy a fix to the banking app tonight. Which level do you restart, and why?
2. AppServer01 is hung. Your teammate stops the whole cluster. What went wrong with their approach?
3. What does `rsPartialStart` mean and what do you do about it?
4. You changed the JVM heap size on AppServer01. Will stopping/starting the *application* apply it?
5. In the wsadmin script for Level 2, why does the MBean string contain `node=Node01`?
6. During a rolling restart, why does IHS matter?

<details>
<summary>👉 Click to see Answers</summary>

1. **Level 3 — application restart** (on each member, or via cluster app controls). The server/JVM doesn't need to restart for a new EAR. Fastest, minimal impact — only sessions are lost.
2. They used **Level 1 where Level 2 was needed**. Stopping the cluster took AppServer02 down too — customers saw an outage. Only AppServer01 should have been stopped; IHS would have rerouted traffic.
3. Some cluster members started but at least one failed. Check the failing member's `SystemOut.log` / `startServer.log` on its node — don't declare the cluster up until state is `rsRunning`.
4. **No.** JVM settings (heap size) are read by the JVM at startup. A Level 3 app restart doesn't restart the JVM — you need a **server restart**.
5. Server names are only unique **per node**. `AppServer01` on Node01 is a different MBean from an `AppServer01` on Node02. Without `node=`, the query is ambiguous.
6. IHS reads `plugin-cfg.xml` to know which members are alive. When you stop one member, the plugin must detect it and route all traffic to the healthy member — that's what makes it zero-downtime.

</details>

---

## 🧯 Common Errors & Fixes

| Error / Symptom | Cause | Fix |
|-----------------|-------|-----|
| `AdminControl` exception: MBean not found | Node agent or server not running, or wrong `node=` in query | Check node agent status first; verify query with `AdminControl.queryNames('type=Server,*')` |
| App restart doesn't fix behavior | Problem is at JVM level (heap, leak) | Escalate to Level 2 — server restart |
| Stopped one member, customers still get errors | IHS still routing to dead server | Regenerate/propagate `plugin-cfg.xml`; verify plugin has the correct member list |
| Cluster state stuck at `rsPartialStart` | One member failed to start | Check that member's `SystemOut.log` — often port conflict or app startup failure |
| Stop cluster hangs | An app or server won't stop gracefully | Use `AdminControl.invoke(cluster, 'stop', 'true')` for immediate (hard) stop, or stop members individually |
| Stopped app, forgot to start it | No auto-restart after app stop | Always script stop+start as a pair; verify with `AdminApp.list()` / runtime state |

---

## 🎯 One-Line Summary

> **Cluster = everyone down (emergencies/maintenance). One server = rolling, zero downtime. Application = fast app-only reload. Pick the lowest level that fixes the problem — and never forget that JVM config needs a server restart, not an app restart.**
