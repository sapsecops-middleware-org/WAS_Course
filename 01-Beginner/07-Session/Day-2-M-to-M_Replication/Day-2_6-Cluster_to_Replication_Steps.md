# 📘 Day 34 Full Console Configuration: Cluster → Replication Domain

> **"Your First Week at a New Bank — Do This from Scratch"**

Pure hands-on guide: configure Memory-to-Memory (M-to-M) session replication **from absolute zero** in the WebSphere Admin Console — exactly as you would on Day 1 at a new bank.

---

## 🧠 Quick Recap — Day 33 (60 Seconds)

You learned the 3 replication modes:

| Mode | Behavior | When to Use |
|---|---|---|
| **BOTH** | JVM sends AND receives | ✅ Always use this in production |
| **CLIENT** | JVM only sends | Memory-constrained JVMs |
| **SERVER** | JVM only receives | Dedicated backup node |

You also learned:

- **Timeout** = 5 seconds (wait for backup JVM confirmation)
- **Frequency** = every 2 seconds (batched replication interval)
- **Disk Offload** = emergency safety net — **never rely on it**

**Today — no new theory. Pure hands-on.**

---

## 🏦 Scenario Setup (Read This First)

You just joined as **WebSphere Admin at DigiBank India**. Your manager calls you on Day 1:

> *"We have a 4-JVM cluster called PaymentCluster. Sessions are being lost every time we patch a JVM. Set up Memory-to-Memory replication by tomorrow morning. Zero session loss during next maintenance window."*

**Your environment:**

| Item | Value |
|---|---|
| Deployment Manager (DMGR) | `dmgr.digibank.in` |
| Admin Console port | `9043` |
| Cluster | `PaymentCluster` |
| JVMs | `server1`, `server2`, `server3`, `server4` |
| Login | `wasadmin / password` |

This is Day 34. You are going to do exactly what this scenario requires.

---

## PART 1: The Full Picture — What You Are About to Build

Before touching anything, understand the complete picture:

### What You Will Build

```
Step 1: Create Replication Domain "PaymentClusterDRS"
        (the named config — the rules of the photocopy machine)

Step 2: Link Replication Domain → PaymentCluster Session Manager
        (tell the cluster to USE this domain)

Step 3: Set Mode = BOTH, Timeout = 5000ms, Frequency = 2 sec
        (how the copying happens)

Step 4: Save + Restart cluster
        (apply everything)

Step 5: Verify DRS is running
        (prove it works before going to manager)
```

> [!NOTE]
> Every single click is shown below. Nothing is assumed.

---

## PART 2: Step 1 — Create the Replication Domain

1. Log in to the Admin Console:

   ```
   https://dmgr.digibank.in:9043/ibm/console
   Login: wasadmin / password
   ```

2. Navigate to:

   ```
   Environment → Replication Domain
   ```

3. Click **New...**

4. Fill in the fields:

   | Field | Value |
   |---|---|
   | **Name** | `PaymentClusterDRS` |
   | **Domain Type** | None (default) |

5. Click **Apply**.

6. Under **Additional Properties**, click **Single row topology settings** and set:

   | Field | Value | Meaning |
   |---|---|---|
   | **Number of replicas** | `1` (entire domain) | 1 backup copy per session |

7. Click **OK** → **Save** to the master configuration.

> [!TIP]
> The replication domain is just a **named rule set**. At this point nothing uses it yet — Step 2 connects it to the cluster.

---

## PART 3: Step 2 — Link the Domain to the Cluster's Session Manager

1. Navigate to:

   ```
   Servers → Clusters → WebSphere Application Server clusters
   ```

2. Click **PaymentCluster**.

3. Under **Configuration**, open the tabs:

   ```
   Container Services → Session Management (or Distributed Environment Settings depending on version)
   ```

4. Check **Enable session management** (if not already enabled).

5. Under **Additional Properties**, click:

   ```
   Distributed Environment Settings
   ```

6. Under **Memory-to-Memory Replication**, set:

   | Field | Value |
   |---|---|
   | **Replication Domain** | `PaymentClusterDRS` (select from dropdown) |
   | **Enable replication** | Checked |

7. Click **Apply**.

---

## PART 4: Step 3 — Set Mode, Timeout, Frequency

Still inside **Distributed Environment Settings → Memory-to-Memory Replication**, configure:

| Field | Value | Why |
|---|---|---|
| **Replication Mode** | `Both` | JVM sends AND receives — production standard |
| **Request Timeout** | `5000` ms | 5 seconds wait for backup JVM confirmation |
| **Replication Interval** | `2` seconds | Batch replication every 2 seconds |
| **Replication Trigger** | `Time Based` | Most common; balances traffic vs. safety |

Click **OK**.

> [!WARNING]
> Mode **MUST** be `Both`. If you accidentally leave it as `Client`, your JVM sends sessions but holds no backups — the cluster thinks replication exists when it does not.

---

## PART 5: Step 4 — Save + Restart the Cluster

1. Click **Save** in the taskbar (master configuration).

2. Synchronize configuration to all nodes:

   ```
   System Administration → Nodes
   → Select all nodes → Full Resynchronize
   ```

3. Perform a **rolling restart** of the cluster (zero downtime):

   ```
   Servers → Clusters → PaymentCluster
   → Stop (ripplestart if available, or stop/start members one at a time)
   ```

   > [!TIP]
   > A **ripplestart** stops and starts each cluster member one-by-one — the rolling restart pattern. Users never see downtime because other members keep serving requests.

4. Wait until all 4 members show **Started** state:

   ```
   Servers → Clusters → PaymentCluster → Cluster members
   server1 → Started ✅
   server2 → Started ✅
   server3 → Started ✅
   server4 → Started ✅
   ```

---

## PART 6: Step 5 — Verify DRS Is Running

### 6.1 Check via Admin Console

```
Servers → Clusters → PaymentCluster → Cluster members
→ server1 → Session Management → Distributed Environment Settings
```

Confirm:

- Replication Domain: `PaymentClusterDRS` ✅
- Replication Mode: `Both` ✅

### 6.2 Check via wsadmin

```python
drsSettings = AdminConfig.list('DRSSettings',
    AdminConfig.list('SessionManager',
        AdminConfig.getid('/ServerCluster:PaymentCluster/')))

print AdminConfig.showall(drsSettings)
```

**Expected output:**

```
[drsMode BOTH]
[requestTimeout 5000]
[messageThreshold 2]
[messageBrokerDomainName PaymentClusterDRS]
```

> [!TIP]
> If `drsMode` shows **BOTH** and `messageBrokerDomainName` shows your domain → ✅ configured correctly.

### 6.3 Runtime Proof (SystemOut.log)

Check any member's `SystemOut.log` for DRS transport messages:

```
grep -i "DRS\|replication" /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/server1/SystemOut.log
```

Look for lines indicating DRS readers/writers started and the domain name.

### 6.4 Functional Test

1. Log into the banking app through `server1` (note the session).
2. Stop `server1`.
3. Refresh the browser → session must **survive** (served by backup on another JVM).

If the user stays logged in after the member goes down → **zero session loss achieved** ✅

---

## ✅ Final Checklist — Report Back to Your Manager

- [ ] Replication domain `PaymentClusterDRS` created
- [ ] Domain linked to `PaymentCluster` session manager
- [ ] Mode = **Both**, Timeout = **5000 ms**, Interval = **2 s**
- [ ] Config saved, synchronized to all nodes
- [ ] Cluster rolled (ripplestart), all 4 members Started
- [ ] `AdminConfig.showall` confirms DRS settings
- [ ] DRS active in `SystemOut.log`
- [ ] Failover test passed — session survived JVM stop

> [!NOTE]
> **Tomorrow's maintenance window:** patch JVMs one at a time (rolling). With M-to-M replication active, sessions failover to backups — users keep working, and your manager gets the "zero session loss" you promised.
---
# 📘 Day 34 — Part 2: Admin Console — Full Step-by-Step

Every click, every field, every screen — from login to verification. Nothing is assumed.

---

## 🔐 Step 0: Login to Admin Console

Open your browser. Type:

```
https://dmgr.digibank.in:9043/ibm/console
```

You will see the WebSphere login page.

| Field | Value |
|---|---|
| **Username** | `wasadmin` |
| **Password** | (your password) |

Click: **[Log in]**

You are now on the Welcome page of Admin Console.

> [!NOTE]
> On the left side, you see a navigation panel. **Everything we do today is from this left panel.**

---

## 📦 Step 1: Create the Replication Domain

> Think of this as: *"Create the photocopy machine and give it a name."*

In the left navigation panel, click:

```
Resources
  ↓
Replication
  ↓
Replication Domains
```

You will see a table. It is probably empty (no replication domains yet).

Click the **[New]** button.

A form opens. Fill it in exactly like this:

```
┌─────────────────────────────────────────────────────────┐
│  Create Replication Domain                              │
│                                                         │
│  Name:                  PaymentClusterDRS               │
│                         ↑ Use cluster name + DRS        │
│                           Makes it easy to identify     │
│                                                         │
│  Number of Replicas:    1                               │
│                         ↑ One backup copy per session   │
│                           Start with 1, increase later  │
│                           if needed                     │
│                                                         │
│  Request Timeout:       5                               │
│                         ↑ Seconds. Wait 5 sec for       │
│                           backup JVM to confirm receipt │
│                                                         │
│  Encrypt Replication:   [ ] (leave unchecked)           │
│                         ↑ Sessions travel inside the    │
│                           bank's internal network       │
│                           Encryption only needed for    │
│                           cross-DC replication          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

Click **[OK]**

You will be taken back to the Replication Domains list.
You should now see **PaymentClusterDRS** in the table.

Now click the **[Save]** link at the top of the page.

> [!WARNING]
> **Always click Save after every change.**
> If you close the browser without saving → change is lost.
> The Save link appears at the top — it looks like a small text link. Easy to miss.
> Senior admins click it without thinking. **Build that habit now.**

---

## 🔗 Step 2: Navigate to PaymentCluster

Now you need to go to your cluster and tell it to **USE** this replication domain.

In the left navigation panel, click:

```
Servers
  ↓
Clusters
  ↓
WebSphere Application Server Clusters
```

You will see a table of clusters. Find **PaymentCluster** and click on its name.

The cluster detail page opens. You will see many sections.

Scroll down until you see the **"Additional Properties"** section at the bottom.

You will see several links:

```
  → Application servers
  → Session Management        ← CLICK THIS
  → Messaging engines
  → ...
```

Click **Session Management**.

---

## ⚙️ Step 3: Navigate to Distributed Environment Settings

You are now on the **Session Management** page for PaymentCluster.

You will see several settings. Look for the **"Additional Properties"** section.

You will see:

```
  → Distributed environment settings    ← CLICK THIS
```

Click **Distributed environment settings**.

A new page opens. You will see **three radio button choices**:

| Option | Meaning |
|---|---|
| ○ **None** | No session sharing — sessions stay only on one JVM |
| ○ **Memory-to-Memory Replication** ← **SELECT THIS** | Copy sessions between JVMs — what we want |
| ○ **Database** | Store sessions in a database — different topic, Day 38 |

Select the radio button next to **Memory-to-Memory Replication**.

Now click **[Apply]** (NOT OK yet — **Apply first** to see the sub-options).

---

## 📋 Step 4: Fill in the M-to-M Settings

After clicking **Apply**, a new sub-section appears below the radio buttons.

You will see a link: **Memory-to-Memory Replication**

Click on that link.

A detailed form opens. Fill it in exactly like this:

```
┌─────────────────────────────────────────────────────────────┐
│  Memory-to-Memory Replication Settings                      │
│                                                             │
│  Replication Domain:                                        │
│  [PaymentClusterDRS ▼]                                      │
│  ↑ Dropdown — select the domain you just created            │
│    If you don't see it → you forgot to Save in Step 1       │
│                                                             │
│  Replication Mode:                                          │
│  ● Both                                                     │
│  ○ Client                                                   │
│  ○ Server                                                   │
│  ↑ Select BOTH — full replication both directions           │
│                                                             │
│  Replication Interval:    2                                 │
│  ↑ Seconds — replicate every 2 seconds (time-based)         │
│                                                             │
│  Replication Trigger:                                       │
│  ● Time-Based                                               │
│  ○ End of Service                                           │
│  ↑ Select Time-Based                                        │
│                                                             │
│  Allow Overflow:          [ ] (leave unchecked)             │
│  ↑ Overflow = use DB if M-to-M fails                        │
│    We do not want this. Keep unchecked.                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

Click **[OK]**

Click **[Save]** at the top of the page.

---

## 🔄 Step 5: Restart the Cluster

Configuration changes to Session Manager need a **full cluster restart** to activate.

Go to:

```
Servers
  ↓
Clusters
  ↓
WebSphere Application Server Clusters
```

Find **PaymentCluster**. Click the checkbox next to it.

1. Click **[Stop]** button.
2. Wait for all 4 JVMs to stop. Status will show **"Stopped"**.
3. Click **[Start]** button.
4. Wait for all 4 JVMs to start. Status will show **"Started"**.

> [!CAUTION]
> **In PRODUCTION:** Never do full stop/start during business hours.
> Use **Rolling Restart** — stop one JVM at a time. Day 63 covers this in detail.
> **In LAB:** full stop/start is perfectly fine.

---

## ✅ Step 6: Verify in Admin Console

Go to:

```
Resources → Replication → Replication Domains → PaymentClusterDRS
```

Click on **PaymentClusterDRS**.

Scroll down. You should see a section called **"Runtime"** or **"Runtime Status"**.

If DRS is working, you will see:

| Field | Expected Value |
|---|---|
| **Status** | `Running` ✅ |
| **Connected Peers** | `server1, server2, server3, server4` ✅ |

> [!WARNING]
> If it shows **"Stopped"** or no peers → something went wrong. Go back and check **Step 2 onwards**.

---

## ✅ Part 2 Complete — Where You Stand

- [x] Logged into Admin Console (`dmgr.digibank.in:9043`)
- [x] Created replication domain `PaymentClusterDRS` (Step 1)
- [x] Linked domain to `PaymentCluster` Session Management (Steps 2–3)
- [x] Mode = **Both**, Interval = **2 s**, Trigger = **Time-Based**, Timeout = **5 s** (Step 4)
- [x] Cluster restarted (Step 5)
- [x] Runtime status verified: **Running**, 4 peers connected (Step 6)

**Next:** Runtime verification via wsadmin + the failover test (Part 3) — prove zero session loss before reporting to your manager. 🎯
