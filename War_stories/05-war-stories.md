# 🏆 WAS MIDDLEWARE WAR STORIES — Interview-Ready Bank
### 10-Year Middleware Admin | Situation → Action → Result → Lesson Format
---

## 🏆 STORY 5: DR FAILOVER — 4 HOURS → 45 MINUTES

### 📋 Situation

> "Regulator-mandated DR drill coming up. Previous drill: **4 hours 10 minutes** to restore the payments estate in DR — and only 'worked' because 8 people heroically improvised. The DR document was 5 years old, never tested, referenced servers that no longer existed, and assumed manual steps by people who'd left. My director's words: 'If a real DR happens tonight, we're done. Fix it.' Untested DR = no DR."

### ⚙️ Action

#### 1. Root-cause the 4 hours — where did the time actually go?

| Time Sink in Old Drill | Root Cause |
|---|---|
| ~90 min | Figuring out WHAT to build — config reconstructed from memory/old docs |
| ~60 min | Manual DMGR/node installs & config per server |
| ~50 min | Datasource/JDBC/security config hunting for correct values |
| ~45 min | IHS/plugin/F5 cutover confusion |
| ~45 min | Waiting on network/firewall/change approvals mid-drill |

> **Insight: the DR failure wasn't technical — it was that prod config lived only in prod.**

#### 2. Rebuilt DR as automation, not documentation
- **Config-as-code**: all WAS cell configs (clusters, datasources, JVM, security, SSL) exported to **Ansible + wsadmin scripts in Git** — prod source of truth, version-controlled
- **Golden builds**: DR cell built from the same playbooks that build prod nodes — identical, no drift
- **Deployments from Artifactory**: DR never stores WARs — it pulls exact prod versions by build number
- **Network pre-staged**: firewall rules, F5 DR pool, DNS entries — requested ONCE during setup, tested quarterly, not requested during a disaster
- **Runbook = checklist, not tutorial**: because automation does the work, the runbook is a 1-page sequence of playbook invocations + verification gates with owners and times

#### 3. Verification-first design
- Each phase ends with a **gate**: configs verified via wsadmin reads, datasource connectivity tested, one full transaction test through the payment API before declaring "restored"
- Defined **RTO 1 hour / RPO 15 min** formally; sequenced the runbook to hit it: infra build (25 min) → app deploy (10 min) → cutover (5 min) → verify (5 min)

#### 4. Drilled it — twice — before the regulator drill
- First internal drill: **52 min** — found DNS entry missed. Fixed. Second drill: **41 min**

### 📈 Result

- Regulator drill: payments estate restored in **45 minutes** — **RTO met with 15 min margin**
- 4 hours → 45 min = **~80% reduction**; 8-person heroics → **2 people + runbook**
- DR went from annual terror to a **routine half-day exercise, twice a year, with evidence pack**
- The Git config-as-code side benefit: prod config drift detection + audit trail came free

### 🎓 Lesson

> "DR is not a document — it's a **rehearsed, automated capability**. The old DR failed not because the team was slow but because prod config existed nowhere except prod. Once config lived in Git and builds were automated, DR became 'run the same playbooks you run every day, pointed at DR.' If your DR has steps only one person knows, you don't have DR — you have a lottery ticket."

### 💬 Follow-up traps — ready answers

| They Ask | You Answer |
|---|---|
| "What about the DATA? You restored apps, not the DB" | "Correct — DR covers the middleware layer. DB failover via storage replication/Oracle Dataguard owned by DBA team, but coordinated: my runbook's datasource verification gate runs only after DBA declares DB up. Cross-team dependency = explicit gate with owner, not hope" |
| "RPO 15 min — how?" | "DB-side replication (DBA owned) + our transaction logs; for config, Git itself is the RPO — config is continuously versioned, effectively RPO zero" |
| "What failed in your first internal drill?" | "A DNS entry for the plugin cutover — 10 minutes hunting. That's the point of rehearsing: the runbook catches what the design review can't" |
| "How do you keep DR current after the drill?" | "Because DR uses the SAME playbooks as daily builds, any prod change flows through Git — DR stays current by construction. Plus quarterly mini-drills. Decay-proofing is the design, not discipline" |
| "Difference between RTO and RPO?" | "RTO = how fast you're BACK (45 min). RPO = how much DATA you may lose (15 min of replication lag). Executives conflate them; I present both numbers with owners for each" |

### ⏱️ 90-Second Delivery Version

> "Last DR drill took 4 hours 10 minutes — 8 people improvising off a 5-year-old document. I timed every step and found the real problem: prod config existed nowhere except prod. So I rebuilt DR as automation — all cell config to Ansible scripts in Git as the source of truth, DR built from the same playbooks as prod, deployments pulled from Artifactory, network pre-staged and tested quarterly. Runbook became a one-page gate checklist. Drilled it twice internally first — first run was 52 minutes and caught a missing DNS entry. Regulator drill: restored in 45 minutes, RTO met, 2 people instead of 8. The lesson: untested DR is no DR — and if a step exists only in one person's head, you have a lottery ticket, not a runbook."

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
