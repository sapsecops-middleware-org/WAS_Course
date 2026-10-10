# 🏆 WAS MIDDLEWARE WAR STORIES — Interview-Ready Bank
### 10-Year Middleware Admin | Situation → Action → Result → Lesson Format

---

## 🏆 STORY 1: ZERO-DOWNTIME FIXPACK UPGRADE — 48 NODES

### 📋 Situation

> "Bank mandate: all WAS estate on **9.0.5.14+ within one quarter** (security baseline + IBM end-of-support date for our old Fixpack). 48 nodes, 4 cells, 120+ JVMs hosting payments, digital lending, internet banking — **10M+ transactions a day**. Business refused any maintenance window. Every prior upgrade in the bank's history was done in a 2 AM weekend window with downtime. The question I was asked: *'Can you patch without downtime?'* Nobody had done it here before."

### ⚙️ Action

#### Phase 1 — Design the procedure (2 weeks)
- Designed the **drain-and-roll pattern**: F5 drain node → quiesce → stop → patch → restart → verify → next node. With 4+ node clusters, users never notice one node gone — capacity drops 25%, headroom absorbs it
- Validated **WebSphere Plugin + F5 behavior during drain**: confirmed in-flight HTTP sessions complete before node stops (graceful stop via `wsadmin` shutdown with wait-time, not kill)
- **Session replication already on** (memory-to-memory) → a drained node's users fail over cleanly to remaining nodes

#### Phase 2 — Prove it in UAT (2 weeks)
- Replicated prod topology in UAT: patched all 48-node equivalent, ran full regression + a **load test with real session traffic** while nodes drained
- Found 2 gotchas in UAT (this is WHY you rehearse):
  - **Plugin refresh timing** — after each node restart, plugin config needed propagation before re-enabling traffic, else stale routing
  - One app's **session serialization failed** on the new Fixpack's session format → caught in UAT, never reached prod
- Built the **verification checklist per node**: server started OK → plugin propagated → health check green → handshake verify → live traffic smoke test → then F5 enable

#### Phase 3 — Automate the repeatable part
- Wrote Ansible + wsadmin scripts: drain → stop → patch apply → start → verify sequence per node. Human only approves each gate. **48 nodes = 48 identical executions**, no fatigue mistakes

#### Phase 4 — Prod rollout (3 nights, phased)
- **Cell 1 (least critical): 2 nodes/night.** Cell 2–4 (payments): next week, 4 nodes/night
- Each night: 30-min dry run on a standby node → execute → evidence pack (patch logs, restart times, health checks) per change record
- **Rollback plan ready per node**: `IBM Installation Manager` rollback tested in UAT; if a node failed to come up clean in 15 min → drain it permanently, continue with remaining nodes, never stall the wave

### 📈 Result

- **48 nodes upgraded, zero user impact, zero rollbacks, zero P1s**
- 6+ hours of weekend maintenance windows **eliminated permanently**
- Procedure documented → became bank standard; later reused for the emergency CVE patching (48 servers in 48 hours)
- **CISO commendation** — and the business stopped fearing upgrades

### 🎓 Lesson

> "Zero-downtime isn't a tool — it's a **designed procedure rehearsed in UAT**. The two gotchas we caught in UAT (plugin propagation, session serialization) would have been prod outages. You don't get zero downtime by being brave; you get it by rehearsing failure."

### 💬 Follow-up traps — ready answers

| They Ask | You Answer |
|---|---|
| "What if a node fails mid-upgrade?" | "15-minute gate: fail to come clean → stays drained, I continue with remaining nodes, cluster absorbs it. Fix off-path. Never gamble the window on one node" |
| "How do you handle in-flight sessions?" | "Graceful wsadmin shutdown lets threads finish; memory-to-memory replication carries active sessions to surviving nodes; verified with real logged-in traffic in UAT load test" |
| "Why 3 nights and not one?" | "Blast radius control. Cell 1 first — 2 nodes = smallest blast radius, prove the procedure, then scale. Big-bang on night 1 is how legends become incidents" |
| "What about apps on single-node clusters?" | "Good catch — 3 apps were single-node. Those got the traditional window. The zero-downtime pattern REQUIRES redundancy; I never pretend a procedure covers topology it doesn't" |
| "Plugin propagation — why does it matter?" | "Plugin routes based on its XML config; restart a backend node but stale plugin config still thinks old state = routing errors. Verify propagation before re-enable — always" |

### ⏱️ 90-Second Delivery Version

> "Bank mandated 48 nodes on 4 cells patched to the new Fixpack in one quarter — 10M transactions a day, no downtime allowed. I designed a drain-and-roll pattern, rehearsed the full rollout in UAT with real session load — which caught two gotchas that would've been prod outages — then automated the sequence with Ansible so all 48 nodes ran identically, phased over 3 nights with a 15-minute failure gate per node. Result: zero user impact, zero rollbacks, zero P1s, and 6 hours of weekend maintenance eliminated permanently. The lesson: you don't get zero downtime by being brave — you get it by rehearsing failure in UAT."

---

## 🎯 The 90-Second Structure (Memorize the Shape)

```
1. SITUATION  — the pain + ONE number                        [15 sec]
2. HOOK       — the one-line insight                         [10 sec]
3. ACTION     — 3–4 steps, technical + specific              [45 sec]
4. RESULT     — the number flipped + bonus outcome           [15 sec]
5. LESSON     — one sentence that sounds earned              [5 sec]
```

## ⚠️ Rule for YOUR Real Stories

- Only claim what you can defend through **3 follow-up questions**
- Adapt numbers to YOUR real estate (even 15 nodes — truth, framed well)
- **Truthful story, well-structured = senior. Inflated story = collapsed under follow-up question 3.**
