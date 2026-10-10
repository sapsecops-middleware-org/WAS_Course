# WebSphere Deployment — Interview Q&A (Easy + Memorize)

---

## Q1: What is rolling deployment and how do you do it in WebSphere?

**Simple meaning:** Update servers **ONE AT A TIME** so customers never see downtime.

### The Story in 6 Steps (for a 2-node cluster)

1. **Stop** AppServer01
2. IHS plugin **auto-routes all traffic** to AppServer02 ✅ (customers unaffected)
3. **Update** AppServer01 → sync Node01 → start it
4. **Verify** AppServer01 is healthy
5. Only THEN **repeat** the same for AppServer02
6. Result: **zero downtime** 🎉

> 🔑 **Key point:** Session replication = customers on AppServer01 don't lose their sessions when it stops.

🧠 **Memory trick:** *"One up, one down, never both"* — like changing tires on a moving car: you never lift the whole car at once.

---

## Q2: What is the difference between AdminApp.update and doing an uninstall followed by install?

**Simple meaning:**

| AdminApp.update | Uninstall + Install |
|---|---|
| Replaces only the EAR file | Removes EVERYTHING, then rebuilds |
| Keeps: name, cluster mapping, virtual host, context roots, JNDI bindings | Loses all config — must redo it all |
| Fast, low risk | Slow, error-prone, but "clean" |

> **When to use uninstall+install:** Only when something fundamental changed — renamed modules, new app name, or suspected config corruption.

🧠 **Memory trick:**
- `update` = **renovate the house** (new furniture, same address)
- `uninstall/install` = **demolish and rebuild** (use only if the foundation is broken)

---

## Q3: During a rolling deployment, you updated Node01 and a critical issue was discovered. AppServer02 is still on the old version. What do you do?

**Simple meaning:** Don't panic — the old server (AppServer02) is still running, so customers are safe. Just undo Node01.

### The Recovery Plan (5 Steps)

1. **Check** AppServer02 is healthy → customers are fine, you have time
2. **Don't touch** AppServer02 (leave it on the old, working version)
3. **Roll back** AppServer01: restore old EAR (backup) → AdminApp.update → Full Resync → start → verify
4. **Paperwork:** change ticket = "rolled back," raise a defect ticket
5. **Redeploy** later after the fix is confirmed

🧠 **Memory trick:** *"Stop, Swap back, Sync, Start, Ticket"* — the **5 S's of rollback**.

> 🏆 **Golden principle to say in the interview:**
> *"Rolling deployment means you always have a safety net — the other node never stopped."*

---

## One-Line Summary of All Three

- **Q1:** Update one node at a time = no downtime.
- **Q2:** update = keeps config; uninstall+install = fresh start (only for big changes).
- **Q3:** Issue found? Other node still serves users → roll back Node01 safely.
