# 🏆 WAS MIDDLEWARE WAR STORIES — Interview-Ready Bank
### 10-Year Middleware Admin | Situation → Action → Result → Lesson Format

---

## 🏆 STORY 4: 65% GC REDUCTION — JVM TUNING DEEP-DIVE

### 📋 Situation

> "Payment API p99 latency was **2.1 seconds**, SLA was 800ms. Business complaints: 'payments feel slow every minute or so.' App team blamed infrastructure, we blamed the code. Pattern I noticed: complaints came in **roughly every 90 seconds**, not randomly. That periodicity is a fingerprint — I've seen it before. I suspected GC."

### ⚙️ Action

#### 1. Enabled verbose GC on 2 of 4 nodes (canary)
- IBM JDK: `-verbose:gc` + GC logs to a separate location; left prod untouched on other 2 nodes for safe comparison

#### 2. Correlated latency with GC logs — smoking gun
- GC analysis: **full GCs every ~90 seconds, 1.2 second pauses**
- Overlaid Grafana latency graph with GC log timestamps: **every latency spike sat exactly on a full GC**. Case closed on causation
- Deeper read: heap was 2GB, but **live data set after each full GC was ~1.5GB** — old gen nearly full → full GC running constantly just to survive. Heap was undersized for the workload

#### 3. Fixed in layers (tuning is math, not magic)

| Change | Rationale |
|---|---|
| Heap 2GB → 4GB (`-Xms=-Xmx`) | Live dataset 1.5GB needs headroom; equal Xms/Xmx removes resize pauses & heap churn |
| GC policy: default → `-Xgcpolicy:gencon` with tuned nursery (`-Xmn` sized so young GCs complete < 50ms) | Application = short-lived request objects → gencon lets them die in nursery; nursery sized by allocation rate from GC logs |
| PMI verification: allocation rate & nursery collection frequency | Sized nursery so minor GCs happen often but cheap — nothing promotes prematurely |

#### 4. Verified the fix with data, not feelings
- 1 week post-change: **full GCs eliminated entirely** (young collections only, ~20–40ms)
- p99: **2.1s → 0.8s (65% reduction)**, meeting SLA for the first time since launch
- Rolled the validated config to remaining 2 nodes via the standard (this fed the Reference Architecture golden JVM settings)

### 📈 Result

- p99 latency **2.1s → 0.8s (−65%)**, SLA met, business complaints stopped
- **Zero full GCs** on that app ever again
- JVM sizing became part of the bank's sizing sheet: *heap ≥ 2.5× live dataset* — every future app sized with this rule, not guesswork

### 🎓 Lesson

> "GC tuning isn't flag-memorizing — it's **reading the heap's story**. The full GC interval told me the workload; the live-data size told me the heap. Change one number without knowing your live dataset and you're guessing. And 'slow every 90 seconds' is a GC fingerprint — latency patterns that repeat on an interval are ALWAYS worth checking against GC logs first."

### 💬 Follow-up traps — ready answers

| They Ask | You Answer |
|---|---|
| "How do you know what heap size is right?" | "You don't pick a number — you measure the **live data set after full GC** and size heap at 2.5–3× it, then validate with verbose GC: young collections frequent but cheap, old gen never fills" |
| "gencon vs balanced vs optavgpause?" | "gencon for transactional apps with short-lived objects — most web apps; balanced for huge heaps with fragmented promotion; optavgpause when pause-sensitive but throughput-light. I choose from GC log evidence, not fashion" |
| "Why -Xms=-Xmx?" | "Heap resize = full GCs + native memory churn. Fixed heap = predictable. The only cost is committing RAM upfront — which you sized correctly anyway" |
| "How did you enable GC logging without restart impact?" | "It does need a restart — so I canaried 2 of 4 nodes, drained-rolled them. Any change, even logging, follows drain-and-roll. No exceptions to my own procedure" |
| "Couldn't the app team just fix the object churn?" | "Churn is NORMAL for web apps — nursery exists for exactly that. The pathology was old-gen pressure from an undersized heap, not the app. Both can be true: tuning fixed it; I also flagged allocation-heavy endpoints to the app team from the logs" |

### ⏱️ 90-Second Delivery Version

> "Payment API p99 was 2.1s against an 800ms SLA, and complaints came in every 90 seconds — that periodicity is a GC fingerprint. Enabled verbose GC on two canary nodes: full GCs every 90 seconds with 1.2-second pauses, and the live dataset was 1.5GB on a 2GB heap — the JVM was suffocating. Fixed with evidence: heap to 4GB with -Xms=-Xmx, GC policy to gencon with nursery sized from the allocation rate. Full GCs eliminated entirely, p99 dropped to 0.8s — a 65% reduction — and the heap-sizing rule went into the bank's standard. The lesson: GC tuning isn't memorizing flags — it's reading the heap's story."

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
