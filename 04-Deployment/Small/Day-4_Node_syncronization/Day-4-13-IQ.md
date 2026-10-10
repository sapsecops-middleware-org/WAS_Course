# PART 14 — Interview Answers Explained (Beginner-Friendly): Node Sync Q&A

---

## Q1: What is node synchronization in WebSphere ND and why is it important?

### Simple explanation

Think of WebSphere ND like a **head office (DMGR)** and **branch offices (Nodes)**.

- The head office writes the **official rulebook** (the master configuration).
- Every branch must have an **exact copy** of that rulebook.
- **Sync = the courier** that delivers updated pages of the rulebook to each branch.

### What travels during sync?

| Item | Example at DigiStack Bank |
|------|---------------------------|
| Deployed EAR files | `digistack-bank-v8.ear` |
| DataSources | The database connection settings |
| JVM settings | Heap sizes, etc. |

### The golden rule

> **DMGR is always the source of truth.** Whatever DMGR has is what every node
> *should* have. Sync is what makes that happen.

### Why it matters (the Part 10 story)

- If you deploy v8 but Node02 never syncs → Node02 still runs **v7**.
- IHS keeps sending customers to Node02 → wrong results or errors.
- **That is exactly the Part 10 disaster.**

### The senior habit 💎

- Don't just *trigger* sync — **verify** it with `isNodeSynchronized=true`
  **before** starting the app.
- **Trigger ≠ completed. Ever.**

---

## Q2: What is the difference between Synchronize and Full Resync?

### Simple explanation — two ways to fix a rulebook

| | **Synchronize (delta)** | **Full Resync** |
|---|-------------------------|-----------------|
| **What it does** | Compares node vs DMGR, pushes only changed files | Deletes everything on the node, re-downloads everything |
| **Speed** | Fast (seconds) | Slower |
| **Analogy** | Send only the pages that changed 📄 | Throw away the whole book, get a brand-new one 📕 |
| **Risk** | Small risk of leftover corrupted/partial files | None — clean slate |

### Real-life example 🏠

> Your house is messy in one room.
>
> - **Delta sync:** tidy that one room. (Fast, usually enough.)
> - **Full resync:** empty the entire house and refurnish it. (Slow, but guaranteed clean.)

### When to use which at DigiStack Bank

- **Regular sync** → every routine deployment (default choice).
- **Full Resync** → only when:
  1. A node has been **offline for a long time** (missed many updates).
  2. You **suspect config corruption**.
  3. After big changes like a **WAS fix pack installation**.

> 💡 **Rule of thumb:** Delta sync first. Full Resync is your "break glass" option.

---

## Q3: Node02 shows 'Sync needed' even after you clicked Synchronize. What do you check?

### Simple explanation — a troubleshooting flow, like a doctor 🩺

You don't guess. You check the **most common causes first**, then go deeper.

### Step 1 — Is the NodeAgent even running? (most common cause!)

- The NodeAgent is the **friendly helper** on each node — DMGR's **channel**
  to push config.
- No NodeAgent = DMGR has no way to reach the node.
  Like trying to deliver a parcel to a house with **no door**. 📦🚪

```bash
ssh to Node02
ps -ef | grep nodeagent
```

- Not running? → `startNode.sh`, then retry the sync.

### Step 2 — If NodeAgent is up, read its logs

- Look in the nodeagent `SystemOut.log` for **ADSY error codes**.
- Specifically **ADSY0012E** = connection failure.

### Step 3 — Check the network path

- DMGR talks to the NodeAgent over **port 8878** (NodeAgent SOAP port).
- Check connectivity + firewall rules between DMGR and Node02 on that port.

### Step 4 — The banking extra: SSL certificates 🏦

- In a bank, inter-node communication is **SSL-encrypted**.
- Expired certificates → sync fails **silently** with an SSL handshake error.
- Nothing obviously "broken", sync just quietly fails — very sneaky!

### The full flow as a picture

```text
Sync failed on Node02
   │
   ├─ 1. NodeAgent running? ── No → startNode.sh → retry
   │
   ├─ 2. ADSY0012E in logs? ── Yes → connection problem
   │
   ├─ 3. Port 8878 open? ──── No → fix firewall/network
   │
   └─ 4. SSL certs valid? ─── Expired → renew certs
```

---

## The Big Lessons 📚

1. **DMGR is the source of truth** — sync is how nodes copy it.
2. **Trigger ≠ verified** — always confirm `isNodeSynchronized=true`.
3. **Delta sync first, Full Resync only for real trouble.**
4. **Troubleshoot in order of probability:** NodeAgent → logs → network → SSL.
5. **Silent failures are the enemy** (expired SSL certs, unsynced Node02) —
   verification catches what hope misses.

---

## Quick Memory Card 🎯

```text
Q1: DMGR = truth → sync delivers it → ALWAYS verify true
Q2: Delta = fast pages | Full = wipe & restore (rare, for corruption/offline)
Q3: NodeAgent → ADSY logs → port 8878 → SSL certs
```