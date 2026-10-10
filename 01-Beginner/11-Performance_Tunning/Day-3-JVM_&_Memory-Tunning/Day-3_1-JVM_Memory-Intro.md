# WebSphere Application Server 9 — JVM Memory Management for Production

A practical, production-focused guide to the three memory regions of a WAS 9 JVM (IBM J9 / OpenJ9), how they fail, and how to size them for banking-grade workloads.

---

## 1. The Big Picture: Three Separate Rooms

A WAS server is a single Java process, but its memory is divided into **three independent regions**. Filling one does **not** release or reuse space in another — and each can fail independently.

| Region | Analogy | What it holds | Typical failure |
|---|---|---|---|
| **Java Heap** | Guest rooms | Application objects (`new Foo()`, sessions, carts, lists) | `OutOfMemoryError: Java heap` |
| **Native Memory** | Staff area, kitchen, machinery | JIT code, thread stacks, NIO/direct buffers, JNI/OS libraries | Silent death, core dump, native OOM |
| **Metaspace** | Filing room with blueprints | Class definitions (bytecode, method metadata) | Classloader leak → unbounded growth |

> [!IMPORTANT]
> **Heap being healthy ≠ process being healthy.** Monitoring that only tracks heap usage will happily report "all green" while the process dies from native memory exhaustion.

---

## The Full JVM Memory Picture
```
┌─────────────────────────────────────────────────────────────────────┐
│                    OPERATING SYSTEM MEMORY (RAM)                    │
│                        (The whole building / land)                  │
│                                                                     │
│  ┌──────────────────────────────────────────────────┐              │
│  │              JVM PROCESS MEMORY                  │              │
│  │              (The hotel building)                │              │
│  │                                                  │              │
│  │  ┌────────────────────────────┐                 │              │
│  │  │        JAVA HEAP           │ ← -Xms / -Xmx  │              │
│  │  │      (The guest rooms)     │                │              │
│  │  │                            │                 │              │
│  │  │  ┌──────────┬──────────┐   │                 │              │
│  │  │  │  NURSERY │  TENURE  │   │                 │              │
│  │  │  │ (Young)  │  (Old)   │   │                 │              │
│  │  │  │ New      │ Surviving│   │                 │              │
│  │  │  │ objects  │ objects  │   │                 │              │
│  │  │  └──────────┴──────────┘   │                 │              │
│  │  └────────────────────────────┘                 │              │
│  │                                                  │              │
│  │  ┌────────────────────────────┐                 │              │
│  │  │      NATIVE MEMORY         │ ← No -X flag    │              │
│  │  │    (The hidden machinery)  │   (the danger!) │              │
│  │  │  • JIT compiled code       │                 │              │
│  │  │  • Thread stacks (-Xss)    │                 │              │
│  │  │  • NIO buffers (MQ, JDBC)  │                 │              │
│  │  │  • OS libraries / JNI      │                 │              │
│  │  └────────────────────────────┘                 │              │
│  │                                                  │              │
│  │  ┌────────────────────────────┐                 │              │
│  │  │    METASPACE / CLASS DATA  │ ← -XX:MaxMeta   │              │
│  │  │    (The filing room with   │   or            │              │
│  │  │     class blueprints)      │  -Xscmx (J9)   │              │
│  │  │  • Class definitions       │                 │              │
│  │  │  • Method bytecode         │                 │              │
│  │  │  • NOT on heap!            │                 │              │
│  │  └────────────────────────────┘                 │              │
│  └──────────────────────────────────────────────────┘              │
└─────────────────────────────────────────────────────────────────────┘

```

### 🧠 How to Read It these 3 Level
```
Level 1:  OS RAM                    → the land
Level 2:   └─ JVM Process           → the building on that land
Level 3:       ├─ Heap              → guest rooms (YOUR objects)
               ├─ Native Memory     → machinery (JVM's own stuff)
               └─ Metaspace         → filing room (class blueprints)

```
### ⚠️ The 3 Warning Signs (pin this to your wall)
```
Heap full        →  OutOfMemoryError: Java heap space      → fix -Xmx
Native exhausted →  OOM native / core dump, heap looks FINE → check threads, JIT, buffers
Metaspace leak   →  grows after hot-deploys, never shrinks → classloader leak
```
---
## 2. Region 1 — Java Heap

### 2.1 What Lives Here

Every object your application creates at runtime:

- User sessions and shopping carts
- DTOs, collections, cached entities
- Every `new SomeObject()` allocation

### 2.2 The Two Core Flags

| Flag | Meaning |
|---|---|
| `-Xms` | Initial heap size (allocated at startup) |
| `-Xmx` | Maximum heap size the JVM may grow to |

### 2.3 Golden Rule for Production: `-Xms = -Xmx`

If the heap starts small and grows under load, the JVM must **stop, resize, and reshuffle memory** — causing pauses and CPU spikes exactly when you least want them (salary day, midnight batch, 3 AM peaks).

```text
BAD  (Dev):     -Xms256m  -Xmx2048m   # heap keeps resizing = unpredictable
GOOD (Prod):    -Xms4096m -Xmx4096m   # fixed size = predictable = stable
```

> [!TIP]
> In production, always set `-Xms` equal to `-Xmx`. This is the single highest-value, lowest-cost tuning decision you can make.

### 2.4 Inside the Heap: Nursery and Tenure

```text
┌─────────────────────────────────┐
│            JAVA HEAP            │
│  ┌────────────┬───────────────┐ │
│  │  NURSERY   │    TENURED    │ │
│  │  (Young)   │    (Old)      │ │
│  └────────────┴───────────────┘ │
└─────────────────────────────────┘
```

- **Nursery (Young):** Where new objects are born. Most objects die young (temporary computations). GC here is frequent but fast.
- **Tenured (Old):** Objects that survive multiple young-GC cycles get promoted here (e.g., long-lived sessions). GC here is slow and expensive.

> [!NOTE]
> Memory trick: *Babies are born in the Nursery. Survivors move to the Tenured area.*

---

## 3. Region 2 — Native Memory (The Hidden Killer)

### 3.1 Why It Is Dangerous

- **No single flag controls it.**
- Most administrators don't know it exists — until it kills the server at 2 AM.
- Monitoring often shows "heap is fine!" because the pressure is **outside the heap**.

### 3.2 What Lives in Native Memory

| Component | Purpose |
|---|---|
| **JIT-compiled code** | Machine code produced by the JIT compiler (code cache) |
| **Thread stacks** | Each thread gets a private stack (~256–512 KB each, set via `-Xss`) |
| **NIO direct buffers** | Used by MQ, web connectors, and JDBC drivers for fast I/O |
| **OS libraries / JNI** | Native C libraries WAS uses internally |

### 3.3 Quick Math: Thread Stacks

```text
-Xss512k                       → 512 KB per thread
200 web container threads      → 200 × 512 KB = ~100 MB
+ JIT code cache, class data,
  MQ/NIO buffers               → easily 2–4 GB of native memory
                                 ON TOP of your heap
```

### 3.4 The Deadly Scenario (Memorize This)

```text
1. Heap looks fine → verbose:gc shows plenty of free heap
2. But the process RSS keeps climbing steadily
3. Finally:
   - OutOfMemoryError: native memory exhausted, OR
   - The process just dies — no Java stack trace, only a CORE dump
```

> [!WARNING]
> Always monitor **total process RSS** (e.g., via `top`, `ps`, or `native_memory` tooling), not just heap utilization. Set OS-level alerts on process size, not only on GC logs.

---

## 4. Region 3 — Metaspace

### 4.1 What It Stores

Class definitions — the "blueprints" of your code: class structure, method bytecode, and runtime metadata.

- **Objects → Heap**
- **Class definitions → Metaspace**

### 4.2 History: PermGen vs Metaspace

| Era | Storage | Behavior | Classic Failure |
|---|---|---|---|
| **Before Java 8** (WAS 7/8) | PermGen — fixed size, inside the heap | Fixed ceiling | `OutOfMemoryError: PermGen space` |
| **Java 8+** (WAS 9, IBM J9) | Metaspace — in native memory | Grows dynamically | Classloader leaks → unbounded native growth |

### 4.3 IBM J9 Specific: Shared Class Cache (SCC)

```text
-Xscmx256m    # Shared Class Cache size — IBM's own class storage
```

### 4.4 Why Banks Care

- **Hot-deploys** and **classloader leaks** keep loading fresh copies of classes.
- Metaspace grows and grows → consumes native memory → process death.
- Every old WAS 7/8 administrator has PermGen nightmare stories.

---

## 5. The 64-bit Reality

### 5.1 Then vs Now

| Architecture | Max Practical Heap | Consequence |
|---|---|---|
| 32-bit | ~1.4 GB hard ceiling | Forced many small JVMs per box |
| 64-bit (WAS 9) | 8–32 GB+ achievable | Fewer, larger JVMs possible |

### 5.2 Bigger Is NOT Better

- A larger heap means **longer Full GC pauses**.
- A 32 GB heap Full GC can pause **30–60 seconds**.
- 60 seconds of a frozen payment server = a payment outage.

### 5.3 Sizing Rule of Thumb

```text
Live set (objects surviving GC) × 3 = starting -Xmx

Example:
  Live set = 1.5 GB
  -Xmx     = 1.5 × 3 = 4.5 GB → round to 4096m or 6144m
```

> [!TIP]
> **Philosophy:** Use the *smallest* heap that runs comfortably with breathing room — not the biggest one you can afford.

---

## 6. IBM J9 vs Oracle Them Up

WAS 9 ships with **IBM J9** (now OpenJ9). It is **not** Oracle HotSpot.

| Feature | IBM J9 (WAS 9) | Oracle HotSpot |
|---|---|---|
| GC policies | `gencon`, `balanced`, `optavgpause`, `optthruput` | G1GC, ZGC, Shenandoah |
| Verbose GC format | IBM XML | HotSpot text |
| Heap dump trigger | `-Xdump:heap:events=systhrow` | `-XX:+HeapDumpOnOutOfMemoryError` |
| Shared class cache | `-Xscmx` | Not available |

> [!WARNING]
> **Interview trap:** Saying *"I use G1GC on WAS 9"* signals you've never actually administered WAS. The correct answer: WAS 9 uses **IBM J9 GC policies** (e.g., `gencon`), not HotSpot collectors.

---

## 7. The Total Memory Formula — Sizing a Box

A JVM needs **far more RAM than its heap**. Sum everything:

```text
Total RAM per JVM =
     Java Heap          (-Xmx)
   + Thread Stacks      (max threads × -Xss)
   + JIT code cache     (~256 MB)
   + Metaspace / SCC    (~256–512 MB)
   + MQ/JDBC buffers    (~256–512 MB)
   + OS overhead        (~512 MB)
```

### Worked Example — `PaymentServer`

```text
  4096 MB   heap
+  100 MB   (200 threads × 512 KB)
+  256 MB   JIT code cache
+  256 MB   Metaspace / SCC
+  256 MB   MQ/JDBC native buffers
+  512 MB   OS overhead
─────────
≈ 5.5 GB per JVM
```

**2 JVMs on one box → 11 GB minimum → provision a 16 GB box for headroom.**

---

## 8. GC Policy Quick Reference (IBM J9)

| Policy | Behavior | When to Use |
|---|---|---|
| `gencon` | Generational concurrent (default) | Best general-purpose choice; low pause + good throughput |
| `balanced` | Region-based, partial GC | Large heaps (multi-GB) on NUMA hardware |
| `optthruput` | Stop-the-world, max throughput | Batch systems where pause time doesn't matter |
| `optavgpause` | Concurrent mark/sweep | Legacy use; mostly superseded by `gencon`/`balanced` |

```text
-Xgcpolicy:gencon    # the standard choice for OLTP banking workloads
```

---

## 9. One-Page Cheat Sheet

| Region | Key Flags | Failure Mode | Monitoring Focus |
|---|---|---|---|
| Heap | `-Xms` / `-Xmx` (always equal in prod) | `OutOfMemoryError: Java heap` | Verbose GC, heap utilization |
| Native | `-Xss` (per-thread stacks only) | Silent death / core dump / native OOM | Process RSS, native memory tracking |
| Metaspace | `-Xscmx` (J9 SCC) | Classloader leak → unbounded growth | Class count, metaspace usage |

### Three Golden Lines

1. **`-Xms = -Xmx` in production. Always.**
2. **Heap healthy ≠ process healthy. Watch native memory.**
3. **Bigger heap ≠ better. Longer GC pauses = outage risk.**

---

## 10. Production Checklist

- [ ] `-Xms` equals `-Xmx`
- [ ] Heap sized as `live set × 3` (verified via verbose GC, not guessed)
- [ ] GC policy is a J9 policy (`gencon` unless you have a documented reason otherwise)
- [ ] Thread stack size (`-Xss`) reviewed against max thread count
- [ ] Process RSS monitored and alerted — not just heap
- [ ] Heap dump trigger configured: `-Xdump:heap:events=systhrow`
- [ ] Shared class cache sized: `-Xscmx`
- [ ] Box RAM covers **heap + native + OS**, with ≥ 25% headroom
- [ ] Verbose GC enabled and shipped to a log aggregator
- [ ] Classloader leak detection in place for hot-deploy environments
