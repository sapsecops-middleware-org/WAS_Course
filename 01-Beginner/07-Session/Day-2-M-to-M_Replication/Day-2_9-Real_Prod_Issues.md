# 📘 Day 34 — Part 4: What Can Go Wrong (Real Production Issues)

These are the top mistakes that happen in real banks. Learn them **now** — not at 2 AM during your first maintenance window.

---

## ❌ MISTAKE 1: Forgot to Save after Creating Replication Domain

```
─────────────────────────────────────────────────────────
What happens: Domain disappears after browser close.
              Cluster restart finds no domain → DRS does not start.
Fix:          Always Save immediately after every config change.
              Verify in Replication Domains list before moving to next step.
─────────────────────────────────────────────────────────
```

> [!WARNING]
> This is the #1 beginner mistake. The **[Save]** link at the top of the console is tiny and easy to miss. Build the habit: **every change → OK → Save → verify in the list.**

---

## ❌ MISTAKE 2: Linked Wrong Replication Domain to Cluster

```
─────────────────────────────────────────────────────────
What happens: 2 clusters exist — PaymentCluster and CreditCluster.
              Admin accidentally links PaymentClusterDRS to CreditCluster.
              CreditCluster sessions replicate. PaymentCluster sessions do NOT.
Fix:          After save — go back and verify the Session Manager shows
              the correct domain name.
Prevention:   Name domains to match clusters: PaymentClusterDRS, CreditClusterDRS.
              Makes mistakes obvious.
─────────────────────────────────────────────────────────
```

> [!TIP]
> **Naming convention is your safety net.** If every domain name matches its cluster name exactly, a wrong link jumps out the moment you look at the Session Manager page.

---

## ❌ MISTAKE 3: Did Not Restart the Cluster After Configuration

```
─────────────────────────────────────────────────────────
What happens: Config is saved. DRS MBeans show 0.
              Admin panics, thinks config is broken.
              Sessions still being lost.
Fix:          Simply restart the cluster. DRS activates on startup.
              Config changes to Session Manager ALWAYS need restart.
─────────────────────────────────────────────────────────
```

> [!NOTE]
> **Rule to memorize:** Saved ≠ Active. Session Manager changes only take effect on JVM restart. If DRS MBeans show 0, don't debug the config — restart first.

---

## ❌ MISTAKE 4: replicaCount Set to 0 Accidentally

```
─────────────────────────────────────────────────────────
What happens: Replication Domain exists, DRS is running,
              but no backup copies are made.
              JVM dies → sessions lost.
              Looks like replication is configured — it is, but useless.
Fix:          AdminConfig.showall on the domain → verify numberOfReplicas = 1 or 2.
              Then restart cluster.
─────────────────────────────────────────────────────────
```

This is the most dangerous mistake because **everything looks healthy**:

- ✅ Domain exists
- ✅ DRS running
- ✅ Peers connected
- ❌ Zero backup copies — replication theater

**Verification via wsadmin:**

```python
domain = AdminConfig.getid('/ReplicationDomain:PaymentClusterDRS/')
print AdminConfig.showall(domain)
```

**Expected output must include:**

```
[numberOfReplicas 1]
```

> [!CAUTION]
> If `numberOfReplicas` shows **0** → every JVM is "replicating" to nobody. Fix it, restart, and re-verify.

---

## ❌ MISTAKE 5: JVMs Not Finding Each Other (Firewall Blocking DRS Port)

```
─────────────────────────────────────────────────────────
What happens: DRS tries to connect JVM-to-JVM on port 7272 (default).
              Firewall between servers blocks this port.
              DRS MBeans start but show 0 peers connected.
              Sessions not replicated.
Fix:          Open port 7272 between all JVM servers (internal firewall rule).
              grep SystemOut.log for "7272" to confirm the port being used.
              Firewall change request to network team.
─────────────────────────────────────────────────────────
```

**Confirm the port in logs:**

```bash
grep -i "7272" /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/server1/SystemOut.log
```

**Test connectivity between every pair of servers:**

```bash
telnet server2.digibank.in 7272
telnet server3.digibank.in 7272
telnet server4.digibank.in 7272
```

> [!TIP]
> For a 4-JVM cluster, you need port 7272 open **in both directions between all pairs** — that's 6 server-to-server connections. Test all of them.

---
