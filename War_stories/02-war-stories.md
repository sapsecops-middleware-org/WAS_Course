# 🏆 WAS MIDDLEWARE WAR STORIES — Interview-Ready Bank
### 10-Year Middleware Admin | Situation → Action → Result → Lesson Format

---

## 🏆 STORY 2: "Zero cert-expiry outages in 4 years after automating certificate lifecycle"

### 📋 Situation (the pain — every middleware admin knows it)

> "Before automation, certificate renewal was **manual chaos**: 100+ SSL endpoints across WAS nodes, IHS, and plugin configs. Certificates lived in different keystores (NodeDefaultKeyStore, truststores, IHS kdb files), renewed by different people, tracked in nobody's memory. In my first year here, one expired mid-day — **2,000 failed logins in 20 minutes**, P1 raised, bank customers posting on Twitter before we knew what happened."

### ⚙️ Action (the build — this is your technical showcase)

1. **Inventory first** — discovered NOBODY knew the full list. Found 100+ certs across keystores, including **12 nobody knew existed** (some expired-but-unused, one live on a payment integration)
2. **Built a certificate inventory database** — endpoint, keystore path, alias, expiry date, owner, renewal runbook link
3. **Automated expiry monitoring** — daily script (bash + keytool/iKeyMan) scanning all keystores, outputting days-to-expiry per cert → **alerts at 30/14/7 days** into the monitoring system
4. **Automated the renewal itself** — Ansible playbook + wsadmin scripts: renew cert → update keystore → propagate → restart JVM rolling (cert change = restart required in classic WAS) → verify SSL handshake from the F5 → re-enable
5. **Drained-and-rolled, never big-bang** — certificate restarts followed the same zero-downtime pattern as patching: F5 drain → restart → verify → next node
6. **Evidence trail** — every renewal logged with change record + pre/post handshake verification output → audit-ready

### 📈 Result

- **Zero cert-expiry outages for 4 years running** (and counting at my exit)
- Renewal across the estate: **2 weeks of panic → 15 minutes automated**
- The daily scan caught **3 near-misses in year one alone** — certs nobody was tracking
- Security audit: certificate management moved from "finding" to "commendation"

### 🎓 Lesson (the senior punchline)

> "Certificate expiry is the most **predictable outage in IT** — the date is literally printed on the certificate. If you're having cert outages, the problem is never the certificate. It's the absence of inventory and automation."

### 💬 Interview follow-ups — ready answers

| Their Question | Your Answer Hook |
|---|---|
| "Why do cert changes need a restart in WAS?" | "SSL config is loaded at JVM/endpoint startup — dynamically reloading needs Liberty or careful config refresh; classic WAS = restart per node, so the design question becomes: how do you restart 48 JVMs without downtime? → drain-and-roll" |
| "Walk me through the renewal flow end-to-end" | "Inventory → new cert from CA (internal PKI) → import to keystore → propagate + IHS kdb → rolling restart with F5 drain → handshake verify → close change record" |
| "What's the risk during cert renewal?" | "Propagation mismatch — keystore updated but truststore/IHS kdb missed = handshake failures on half the path. My playbook verifies handshake from BOTH browser-side (F5) and backend before marking done" |
| "How do you monitor certs you don't know about?" | "That's why inventory came first — keytool -list swept every keystore on every server, diffed against known endpoints. The 12 unknown certs were the real finding" |

### ⏱️ 90-Second Delivery Version

> "We had 100+ SSL endpoints, certs scattered across keystores, tracked by nobody. One expired mid-day — 2,000 failed logins in 20 minutes. I built a full cert inventory (found 12 certs nobody knew existed), automated daily expiry scanning with alerts at 30/14/7 days, and automated renewal end-to-end with drain-and-roll restarts. Zero cert-expiry outages for 4 years, renewal went from 2 weeks of panic to 15 minutes, and the audit finding became a commendation. The lesson: cert expiry is the most predictable outage in IT — if you're having cert outages, the problem is inventory and automation, not certificates."

---

## ⚠️ Rule for YOUR Real Resume

Only use these bullets if **you can defend every number**. If the interviewer says *"show me the reference architecture"* or *"what triggered the automation?"* — you must have a story. Adapt to YOUR actual experience:

- Real estate size you worked on (even 15 nodes is fine — say the truth, frame it well)
- Real outages you lived (cert expiry, config drift, onboarding pain — everyone has these)

> **Truthful story, well-structured = senior. Inflated story = collapsed under follow-up question 3.**

---

## 🎯 The 90-Second Structure (Memorize the Shape)

```
1. SITUATION  — the pain + ONE number                        [15 sec]
2. HOOK       — the one-line insight                         [10 sec]
3. ACTION     — 3–4 steps, technical + specific              [45 sec]
4. RESULT     — the number flipped + bonus outcome           [15 sec]
5. LESSON     — one sentence that sounds earned              [5 sec]
```
