# 📝 Answer Review — Q1/Q2/Q3 (Server vs. Cluster Deploy)

---

## Q1: Server vs. Cluster — ✅ Strong Pass (9/10)

### What you nailed:

- ✅ **"Only that one JVM runs the application"** — correct, and you framed it in terms of **blast radius**
- ✅ **"No failover"** — the key operational consequence
- ✅ **"WAS distributes the application to every cluster member... same EAR"** — correct
- ✅ **"If one member fails the plugin automatically reroutes"** — correct plugin failover behavior
- ✅ **Tied it to the business context (banking, zero downtime)** — this is what separates an operator from an engineer in an interview

### Minor gap (the missing 1 point):

> You didn't mention the **single point of administration**.
> With a cluster target, one install operation updates all members and **future members added to the cluster automatically get the app**.
> With a per-server target, every new server needs a **manual deploy**.
> That's the second half of "why clusters" beyond HA — **operational consistency**.

### 💡 One-liner upgrade:

> *"Cluster = one deployment, N servers, automatic failover, and any future member inherits the app automatically. Single server = one deployment, one JVM, one point of failure, and manual work every time you scale."*

---

## Q2: Sequence of Events After Cluster Deploy — ✅ Strong Pass (9/10)

### What you nailed — and in the right order:

| Your Step | Correct? |
|-----------|----------|
| DMGR saves config to master repository | ✅ |
| NodeSync pushes EAR to each node via NodeAgent | ✅ |
| AppServers load from **local copy, not DMGR directly** | ✅⭐ — this is the detail most juniors get wrong |
| JVM reads EAR → classloaders → JNDI/DataSources → servlet init → RUNNING | ✅ |
| plugin-cfg.xml regenerated so IHS routes to new context roots | ✅⭐ — often forgotten |

### What impressed me:

- ⭐ **"They load from their local copy, not from DMGR directly"** — this is a **senior-level distinction**. It's why sync is a hard prerequisite and why a stale node serves a *stale version*.
- ⭐ **You ended with the verification sequence** — mirroring the runbook habit from Part 8.

### Minor gaps:

1. **Who triggers the sync?**
   You said *"Node synchronization then pushes..."* — passive voice.
   In an interview, say:
   > *"Sync can be automatic (sync interval) or manual — and for production deploys we force it manually via the NodeSync MBean so we control timing."*

2. **Plugin nuance:**
   `plugin-cfg.xml` regen isn't always automatic on app start — it's generated at the DMGR and must be **propagated to IHS**.
   Say:
   > *"Regenerate and propagate plugin-cfg.xml to the web server, or IHS may still 404 on the new context root."*

---

## Q3: App on Node01 Only, Intermittent 404s — ✅ Strong Pass (10/10)

This is essentially a **perfect reproduction of the Part 8 incident response**:

- ✅ **Pattern recognition first:** *"Intermittent 404 in a cluster immediately tells me one member is missing the app"* — you led with reasoning, not tooling. Interviewers love this.
- ✅ **Layer isolation:** direct `curl :9080` per node, bypassing IHS — proves it's WAS, not the plugin
- ✅ **Root cause confirmation:** Manage Modules showing single-server mapping
- ✅ **The right fix:** `AdminApp.edit` → save → **sync both nodes** → restart app — the exact low-risk sequence
- ✅ **Quantified impact:** *"under 10 minutes"* — speaks to production pressure awareness
- ✅ **Process fix, not just technical fix:** the mandatory checklist. This closes the loop on *"why did it happen and how do we stop it recurring."*

### One small addition that would make it flawless:

> Mention **how you'd verify the fix**:
>
> ```bash
> curl Node01:9080/digistack   → 200 OK
> curl Node02:9080/digistack   → 200 OK
> curl via IHS                 → 200 OK
> ```
>
> Plus confirm `CWWEB0001I` appears in **Node02's** `SystemOut.log`.
> **Logs as evidence closes the incident properly.**

---
