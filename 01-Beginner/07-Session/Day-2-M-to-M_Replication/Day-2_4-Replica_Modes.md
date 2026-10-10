# Tomcat Cluster Replication Modes — Deep Dive in Simple English

This document explains the three session **replication modes** used in a Tomcat/JVM cluster: `BOTH`, `CLIENT`, and `SERVER`.

---

## The Core Question

Every replication mode answers **ONE** question:

> **"What is this JVM ALLOWED to do with session copies?"**

There are only two possible actions:

1. **SEND** my sessions to someone else → backup **OUT**
2. **RECEIVE** other people's sessions and hold them → backup **IN**

The three modes are just the three possible combinations:

| SEND | RECEIVE | Mode |
|:----:|:-------:|:-----|
| ✅ | ✅ | `BOTH` |
| ✅ | ❌ | `CLIENT` |
| ❌ | ✅ | `SERVER` |

---

## Mode 1: BOTH ✅ (The Full Partnership)

### What It Means

The JVM does **BOTH** jobs:

- It **sends** its own sessions to other JVMs (it is a **SOURCE**).
- It **receives** and stores other JVMs' sessions (it is a **RECEIVER**).

### Picture It

```
        JVM1                     JVM2
   ┌──────────┐             ┌──────────┐
   │ Own real │──copies────►│ Backup of│
   │ sessions │             │ JVM1     │
   │          │             │          │
   │ Backup of│◄─copies─────│ Own real │
   │ JVM2     │             │ sessions │
   └──────────┘             └──────────┘
```

Both boxes have:

- Real sessions (their own customers)
- Backup copies (the other guy's customers)

### What Happens Internally (Step by Step)

1. Rahul logs in. JVM1 creates his session in its RAM.
2. Immediately (or within milliseconds), JVM1 packages that session.
3. JVM1 sends the copy over the network to JVM2.
4. JVM2 stores it in its RAM, marked as: *"This is a COPY, not mine. Owner = JVM1."*
5. Meanwhile, Meena logs in on JVM2. JVM2 copies HER session to JVM1 the same way.
6. Every time Rahul does something (updates cart, transfers money), the **updated copy is re-sent**. The backup stays fresh.

> [!IMPORTANT]
> The copy is updated **continuously**, not just once. Otherwise the backup would be old and useless.

### Failover in BOTH Mode

```
Rahul's session on JVM1 → copy exists on JVM2
💥 JVM1 crashes
Load balancer: "JVM1 is dead. Route Rahul to JVM2."
JVM2: "I have Rahul's copy. I'll activate it."
Rahul keeps working. No logout. No data loss.
```

The copy on JVM2 "wakes up" and becomes a real session. This is called **failover**.

### Real-Life Analogy

Two bank tellers, Tara and Mohan:

- Tara makes photocopies of her customer files and gives them to Mohan every hour.
- Mohan does the same for Tara.
- If Tara goes on leave, Mohan opens his drawer and — there are all of Tara's files. He serves her customers like it's nothing.
- Neither teller is a burden on the other. Both share the work equally.

### Pros

- ✅ Maximum protection — every JVM has a buddy
- ✅ No single point of failure
- ✅ Load is shared evenly — everyone holds roughly the same amount of backup data
- ✅ Simple to manage — same setting on all servers

### Cons

- ⚠️ Every JVM needs extra memory (to hold backups)
- ⚠️ Network traffic — copies constantly flying between JVMs

### When to Use

- Almost always. Production banking clusters.
- When all JVMs are similar in size and memory.
- When you want no weak link in the chain.

> [!TIP]
> **One-line memory hook:** `BOTH` = "You watch my back, I watch yours."

---

## Mode 2: CLIENT ❌✅ (The Giver Who Won't Keep)

### What It Means

The JVM does only **ONE** job:

- It **sends** its own sessions to other JVMs. (**SOURCE** only)
- It **refuses** to store anyone (**NOT** a receiver)

### Picture It

```
        JVM1                     JVM2
   ┌──────────┐             ┌──────────┐
   │ Own real │──copies────►│ Backup of│
   │ sessions │             │ JVM1     │
   │          │             │          │
   │ (empty   │◄─tries──────│ Own real │
   │ backup   │  to send    │ sessions │
   │  space)  │   its copies│          │
   └──────────┘   ❌ REJECTED└──────────┘
```

Notice: JVM2 still sends copies toward JVM1 — but JVM1 **throws them away**. JVM1 only pushes **OUT**, never takes **IN**.

### What Happens Internally

1. Rahul logs in on JVM1.
2. JVM1 sends his session copy to JVM2. JVM2 stores it. ✅
3. Meena logs in on JVM2.
4. JVM2 sends her session copy to JVM1.
5. JVM1 says: *"No. I'm CLIENT mode. I don't keep backups."* ❌
6. Meena's copy goes... nowhere (or to another willing receiver in the cluster).

### The BIG Risk — Read Carefully ⚠️

> [!WARNING]
> A `CLIENT`-mode JVM's sessions **ARE protected** (they're copied out somewhere else).
>
> But the sessions of others are **NOT protected** by this JVM. If the receiving JVM dies, someone must still hold those backups. So `CLIENT` mode only works safely when **OTHER** JVMs in the cluster are receiving (`BOTH` or `SERVER` mode).

```
CLIENT mode alone             = dangerous (nowhere to store backups)
CLIENT + BOTH/SERVER servers  = safe ✅
```

### What Happens at Failover

- If JVM1 (`CLIENT`) crashes → Rahul's session is safe on JVM2. Rahul fails over fine. ✅
- But nobody was failing over **TO** JVM1. JVM1 was never a safety net for anyone.

### Real-Life Analogy

**Teller with a tiny desk.**

- His desk can barely fit his own files.
- So he photocopies his files and sends them to the big-desk teller next door.
- When the big teller offers **HIM** copies to hold, he says: *"Sorry, look at my desk. No space. Give them to someone else."*
- He's protected. But he protects no one.

### Why "CLIENT"? (The Logic of the Name)

Think of a customer at a bank counter:

- A **customer GIVES** his documents to the bank (submits for safe keeping in the locker).
- The **bank KEEPS** them.
- The customer doesn't keep the bank's documents.

`CLIENT` = the one who **hands over** data. `SERVER` = the one who **keeps** it. Classic client-server thinking applied to session copies.

### Pros

- ✅ Saves memory on the JVM — no backup data eating RAM
- ✅ Good for small, weak, or busy JVMs
- ✅ Its own sessions are still safe (backed up elsewhere)

### Cons

- ❌ It helps nobody — it holds zero backups for others
- ❌ Must be paired with willing receivers, or backups have nowhere to go
- ❌ Creates uneven load — other JVMs carry the extra burden

### When to Use

- JVM has very limited RAM (small box, cheap VM).
- JVM is a light front-end — just handles login pages or routing; you don't want backup data slowing it down.
- Other bigger JVMs in the cluster can absorb the backups.

> [!TIP]
> **One-line memory hook:** `CLIENT` = "I hand over my files for safekeeping, but don't give me yours — I have no space."

---

## Mode 3: SERVER ✅❌ (The Keeper Who Gives Nothing)

### What It Means

The JVM does only the **OTHER** job:

- It **receives** and stores other JVMs' sessions. (**RECEIVER** only)
- It does **NOT** send its own sessions anywhere. (**NOT** a source of backups)

### Picture It

```
        JVM1                     JVM2
   ┌──────────┐             ┌──────────┐
   │ Own real │──copies────►│ Backup of│
   │ sessions │             │ JVM1     │
   │          │             │          │
   │ Own real │◄─copies─────│ Own real │
   │ sessions │  of JVM2    │ sessions │
   │ (NOT     │             │          │
   │ backed   │             │          │
   │ up!) ⚠️  │             │          │
   └──────────┘             └──────────┘
```

Wait — look at the second arrow reversed. JVM1 **receives** JVM2's copies ✅. But **nobody receives JVM1's copies** ❌.

### What Happens Internally

1. Other JVMs push their session copies to JVM1. JVM1 accepts and stores them all. ✅
2. JVM1 has its **OWN** sessions too (if it serves users).
3. But JVM1 **never** sends copies of its own sessions out.
4. If JVM1 dies → its own sessions are gone forever. ⚠️

### The BIG Risk — Read Carefully ⚠️

> [!WARNING]
> **The backup holder itself is NOT backed up.**

If the `SERVER`-mode JVM crashes:

- Everyone else's backups on it are fine (originals still live on their owner JVMs).
- But the `SERVER` JVM's **OWN** sessions die with it.
- Any customer working on **THAT** JVM gets kicked out. Session expired. Start again.

So rule: only use `SERVER` mode on a JVM whose **own sessions don't matter**.

### Real-Life Analogy

**The big filing cabinet room.**

- One employee has a huge, mostly-empty desk.
- Everyone sends their photocopies to him. His desk becomes the bank's filing cabinet.
- Nobody backs up **HIS** files — because his own files are just rough notes and scratch paper. If they're lost, nobody cares.
- But imagine he kept an important customer's original file... and his desk caught fire. That customer's file is gone. **That's the risk.**

### Why "SERVER"? (The Logic of the Name)

Again, the bank counter:

- The bank/**server KEEPS** the customer's documents in the locker.
- It receives, it stores, it protects.
- But a locker doesn't hand over **ITS OWN** documents for someone else to keep.

`SERVER` = the **receiver and keeper** of other people's data.

### Pros

- ✅ Perfect for a dedicated backup node — one big machine holds everyone's copies
- ✅ Its big memory is put to good use
- ✅ Other JVMs stay light — they don't carry each other's backups
- ✅ Easy to plan capacity — you know exactly which machine holds backups

### Cons

- ❌ Its own sessions have **ZERO protection**
- ❌ If it dies, its own users lose their sessions
- ❌ Can become a single point of failure for the backup data (if it's the only receiver)

### When to Use

- A dedicated backup node — its whole purpose is holding copies.
- A JVM running **stateless or throwaway work** (batch reports, nightly summaries) — sessions there are disposable.
- A JVM with lots of spare memory and unimportant sessions of its own.

> [!TIP]
> **One-line memory hook:** `SERVER` = "Give me everyone's files — I'll keep them. My own files? Nobody needs to back those up."

---

## The Deepest Understanding — One Picture for All Three

Think of it as a two-way street with **toll gates**:

```
                SEND gate          RECEIVE gate
                (backup OUT)       (backup IN)
               ┌──────────┐       ┌──────────┐
    BOTH       │   OPEN   │       │   OPEN   │
               └──────────┘       └──────────┘
               ┌──────────┐       ┌──────────┐
    CLIENT     │   OPEN   │       │  CLOSED  │
               └──────────┘       └──────────┘
               ┌──────────┐       ┌──────────┐
    SERVER     │  CLOSED  │       │   OPEN   │
               └──────────┘       └──────────┘
```

> [!NOTE]
> Every mode = just a combination of **open/closed gates**.

---

## Master Comparison Table (Memorize)

| Question | BOTH | CLIENT | SERVER |
|:---|:---:|:---:|:---:|
| Does it own real sessions? | Yes | Yes | Yes (usually) |
| Sends copies of its sessions? | ✅ Yes | ✅ Yes | ❌ No |
| Holds others' backups? | ✅ Yes | ❌ No | ✅ Yes |
| Are **ITS** sessions protected? | ✅ Yes | ✅ Yes (elsewhere) | ❌ **NO — biggest risk** |
| Does it protect **OTHERS**? | ✅ Yes | ❌ No | ✅ Yes |
| Memory needed for backups? | Medium | None (saves RAM) | High |
| Role name | Source + Receiver | Source only Receiver only |

## Key Pattern

- `CLIENT`'s own sessions = **SAFE** (they're sent out).
- `SERVER`'s own sessions = **UNSAFE** (nobody takes them).
- `BOTH` = everyone safe.

> [!IMPORTANT]
> **The irony people miss:** `CLIENT` is safe for itself, `SERVER` is not.
> The names sound like the opposite should be true — don't fall for it.

---

# How a Real Bank Mixes All Three

A smart production design doesn't use just one mode everywhere:

```

┌─────────────────────────────────────────────────┐
│ │
│ JVM-A (BOTH) ◄────► JVM-B (BOTH) │
│ │ protect each other │
│ │ │
│ ▼ sends copies │
│ │ │
│ JVM-C (CLIENT) ── pushes its backups to A & B │
│ (small front-end box, holds nothing) │
│ │
│ JVM-D (SERVER) ◄── A & B also push extra │
│ copies to D │
│ (big backup box, its own sessions unprotected │
│ — but it runs batch reports, so who cares) │
│ │
└─────────────────────────────────────────────────┘

```

### Why This Mix Is Smart

- `JVM-A` and `JVM-B` are the main workhorses → they **protect each other** (`BOTH`).
- `JVM-C` is tiny → it can't hold backups, but its own sessions are safe because A & B receive them (`CLIENT`).
- `JVM-D` is a big machine with throwaway sessions → it acts as an **extra filing cabinet** (`SERVER`).

**Result:** Every important session has at least one backup. No machine
wastes memory it doesn't have. This is what a 25-year admin designs.

---

## The Two Golden Safety Rules

These rules come from real production incidents:

### Rule 1: A cluster of only CLIENT-mode JVMs = disaster

Everyone pushes backups, nobody stores them. The copies have nowhere to go.
You need at least one `BOTH` or `SERVER` **receiver** in the mix.

### Rule 2: Never put important user sessions on a SERVER-mode JVM

If it crashes, those sessions die with **zero backup**. `SERVER` mode is only
for JVMs whose own sessions are disposable.

---

## Quick Revision Summary

| Aspect | BOTH | CLIENT | SERVER |
|:---|:---:|:---:|:---:|
| Gates open | SEND + RECEIVE | SEND only | RECEIVE only |
| Memory hook | "You watch my back, I watch yours." | "I have no space for your files." | "Give me everyone's files." |
| Own sessions | Safe ✅ | Safe ✅ (elsewhere) | Unsafe ❌ |
| Protects others | Yes ✅ | No ❌ | Yes ✅ |

> [!TIP]
> If you remember only ONE thing: **the mode tells you which gates are
> open — SEND, RECEIVE, or both. Everything else follows from that.**
