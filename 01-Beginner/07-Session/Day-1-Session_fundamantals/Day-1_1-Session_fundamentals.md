# HTTP Sessions — Complete Guide

> [!NOTE]
> This document explains HTTP session management from first principles, using simple language, analogies, and diagrams. It is part of a WebSphere Application Server (WAS) learning series.

---

## 1. The Problem: HTTP Has No Memory

### 1.1 What is HTTP?

HTTP is the protocol (language) that browsers and servers use to communicate. Both sides understand each other — but the server has **no memory** between requests.

### 1.2 What Does "Stateless" Mean?

> **Stateless** = No memory = Every request is treated as a **new stranger**.

### 1.3 HTTP is a protocol with zero memory.

Consider a user logging into a banking application:

```text
Click 1: "Show my balance"   → Server: "Who are you?"
Click 2: "Transfer ₹50,000"  → Server: "Who are you?"
Click 3: "Pay my bill"       → Server: "Who are you?"
```

Every single click — the server forgets the user. This is HTTP's default behavior.

This is called HTTP statelessness. The protocol by design remembers nothing.

### 1.4 Which Applications Need Memory?

| Site Type     | Needs Memory? | Why                              |
|---------------|---------------|----------------------------------|
| News blog     | ❌ No         | Every page is public             |
| Weather site  | ❌ No         | Just displays data               |
| NetBanking    | ✅ Yes        | Must know it is *you* on every click |

### 1.5 Why This Matters

Without session memory, banking site would:

- 💸 Transfer money to the wrong account
- 🚨 Allow anyone to claim to be anyone
- 💥 Fail to function entirely

**Solution:** A memory system — the **HTTP Session**.

---

## 2. Real-World Analogies

### 2.1 Bank Branch 🏦

```text
1. Show ID at door     → Guard recognizes you
2. Counter 1           → Teller knows you
3. Counter 2           → Same memory follows you
4. Return in 10 min    → Still recognized
```

The branch gave you a **token number** and kept a **record** against it.

> **Token + Record = Session**

### 2.2 Railway PNR 🚂 (The Best Analogy)

| Railway World              | NetBanking World                    |
|----------------------------|-------------------------------------|
| You book a ticket          | You log in                          |
| Railway gives PNR number   | WAS gives `JSESSIONID`              |
| Gatekeeper checks PNR      | IHS/Plugin forwards cookie          |
| TC checks PNR on train     | WAS validates session each request  |
| You exit, PNR is done      | You log out, session is destroyed   |

> [!TIP]
> 🎫 **Memory hook:** One PNR = one journey. One `JSESSIONID` = one login session.

---

## 3. The Full Session Flow

### Step 1 — Login Request

User types username + password → browser sends credentials to the server.

### Step 2 — WAS Verifies

```text
Browser → IHS → Plugin → WAS (JVM)
```

The application checks credentials. Correct? ✅ User is authenticated.

### Step 3 — Session Is Created

WAS creates `HttpSession` object. It lives in **JVM RAM (heap memory)**.

### Step 4 — Session ID Is Generated

```text
JSESSIONID = A3F9X8B2:0001
```

> `:0001` is the **CloneID** — the JVM's name tag.

### Step 5 — Cookie Sent to Browser

WAS sends a special response header:

```http
Set-Cookie: JSESSIONID=A3F9X8B2:0001
```

The browser stores it automatically.

### Step 6 — Every Following Click

The browser automatically attaches the cookie:

```http
Cookie: JSESSIONID=A3F9X8B2:0001
```

WAS looks it up in memory: *"Ah — this is Ravi. Balance ₹2,40,000. Mid-NEFT transfer."*

The experience feels continuous to the user — underneath, every request carries the ID like a passport.

### Step 7 — Logout

User clicks Logout → WAS destroys the session from memory.

- Old `JSESSIONID` reused? → `"Session not found"` → user is kicked to the login page.

### Who Creates the JSESSIONID?

| Component | Creates JSESSIONID? | Role            |
|-----------|---------------------|-----------------|
| WAS       | ✅ Yes              | Creator/owner   |
| IHS       | ❌ No               | Carrier only    |
| Plugin    | ❌ No               | Carrier only    |
| Browser   | ❌ No               | Storage only    |

> IHS and Plugin = postal workers. They only **CARRY** it.

### Master Diagram

```text
      YOUR BROWSER                        BANK SERVER (WAS)
          │                                     │
 1. Login: "Ravi / 1234"
          │ ──────────────────────────────────► │ 2. Check password ✓
          │                                     │ 3. Create session box:
          │                                     │    ┌──────────────────┐
          │                                     │    │: RAVI_001 │
          │                                     │    │ Bal: 240000      │
          │                                     │    └──────────────────┘
          │                                     │ 4. Generate ID: A3F9X8B2:0001
          │ ◄──── Set-Cookie: JSESSIONID ────── │ 5. Send cookie
  (browser saves cookie)                       │
 6. "Transfer money"
          │ 🎫 Cookie: A3F9X8B2:0001 ────────► │ "A3F9X8B2 = Ravi!" ✓
          │ ◄──────── "Money sent" ──────────── │
 7. Logout
          │ ──────────────────────────────────► │ 🗑️ session destroyed
```

---

## 4. The Session Object

### 4.1 A Key-Value Locker

```java
HttpSession session = request.getSession();
session.setAttribute("userId", "RAVI_CITI_001");
session.setAttribute("accountBalance", 240000);
session.setAttribute("lastTransactionId", "NEFT_TX_98712");
session.setAttribute("loginTime", System.currentTimeMillis());
```

### 4.2 School Locker Analogy 🎒

| Concept                 | Meaning                          |
|-------------------------|----------------------------------|
| One locker per student  | One session per user             |
| Locker NUMBER           | `JSESSIONID`                     |
| Books/notes inside      | `userId`, `balance`, etc.        |
| The school building     | JVM heap                         |

### 4.3 What the JVM Heap Looks Like

```text
JVM1 HEAP MEMORY
┌────────────────────────────────────────────┐
│ Session A3F9X8B2:0001 → Ravi's data        │
│ Session B7K2P1M4:0001 → Priya's data       │
│ Session C9X2L5N8:0001 → Amit's data        │
│ ... one box per logged-in user             │
└────────────────────────────────────────────┘
```

### 4.4 The Big Number to Remember

> [!IMPORTANT]
> **50,000 users logged in = 50,000 session objects = RAM pressure.**
> Sessions eat memory. Bigger sessions = less room for the application itself. (Tuning is covered in Day 42.)

---

## 5. Reading the JSESSIONID

### 5.1 Anatomy

```text
JSESSIONID = A3F9X8B2 : 0001
             ────┬────   ──┬──
              random ID   CloneID
              (the key)   (which JVM made it)
```

| Piece        | Meaning                            |
|--------------|------------------------------------|
| `JSESSIONID` | Cookie name (Java Session ID)      |
| `A3F9X8B2`   | Random unique session key          |
| `:0001`      | CloneID — which JVM created it     |

### 5.2 Why the CloneID Matters

```text
Request comes with A3F9X8B2:0001
        │
Plugin reads ":0001"
        │
"Ravi's session lives in JVM 0001 → send him THERE"
        │
✅ Session affinity = user always lands on the JVM
   that holds his memory
```

> [!WARNING]
> If CloneID routing fails, the request goes to the **wrong JVM** → `"Who are you?"` → random logout. 😱
>
> 🔗 This connects **CloneIDs** with session behavior — that is why Phase 6 = Sessions + Integration.

---

## 6. Session Lifetime

```text
1. BIRTH 👶
   Created when app calls request.getSession()
   (usually at login)

2. LIFE 🏃
   Alive as long as user keeps clicking
   Every request updates "last access time"

3. TIMEOUT ⏰ (natural death)
   User walks away, no clicks
   Default: 30 minutes of inactivity
   Configurable in WAS console (Session Management)

4. DEATH BY LOGOUT 🗑️
   User clicks Logout → app destroys the session
```

### Why Timeout Matters

```text
User leaves PC open, goes to lunch 🍔
30 min later → session expired
Next click → "Session expired, please login again"
```

> [!NOTE]
 This is **not a bug**. It is security working as designed. ✅

---

## 7. Cookie vs Session

```text
┌─────────────────────┐       ┌─────────────────────┐
│   COOKIE 🍪          │       │   SESSION 📦         │
│   (Your Browser)     │       │   (Server JVM heap)  │
│                      │       │                      │
│  Only the ID:        │       │  The REAL data:      │
│  "A3F9X8B2:0001"     │       │  Ravi, ₹240000, NEFT │
└─────────────────────┘       └─────────────────────┘
   locker NUMBER 🗝️              locker CONTENTS 💎
```

| Question         | Cookie                 | Session                          |
|------------------|------------------------|----------------------------------|
| Lives where?     | Browser (your PC)      | Server (JVM heap)                |
| Contains?        | Just the ID            | Actual data (`userId`, balance)  |
| Size             | Small (few KB)         | Can hold real data               |
| Value if stolen  | Low alone              | High — real data!                |

### Security Insight

- 🔑 The browser carries only the **locker number**. Contents never leave the server.
- ⚠️ A stolen `JSESSIONID` lets an attacker impersonate the user = **session hijacking**.

**Defenses:**

- 🔒 **HTTPS** → ID travels encrypted
- ⏰ **Idle timeout** → stolen IDs expire

---

## 8. Quick Reference

| Concept          | One-line meaning                        |
|------------------|-----------------------------------------|
| HTTP stateless   | Every request = new stranger            |
| Session          | Server-side memory for one user         |
| `JSESSIONID`     | The unique ID (PNR) for your session    |
| Cookie           | Browser-side holder of the `JSESSIONID` |
| `HttpSession`    | Java object holding user data in JVM heap |
| CloneID (`:0001`)| Tag saying which JVM owns this session  |
| Session timeout  | Auto-expiry after 30 min idle           |
| Logout           | Explicit destruction of session         |

### The Whole Flow in 6 Lines

```text
1. Login        → WAS verifies credentials
2. WAS creates  → HttpSession in JVM heap
3. WAS makes ID → JSESSIONID → sent as Set-Cookie
4. Browser      → stores cookie → sends it back every click
5. WAS          → looks up session → "This is Ravi"
6. Logout/timeout → session destroyed
```

### Memory Hooks 🎯

1. HTTP has no memory → **Sessions ARE the memory**
2. `JSESSIONID` = railway PNR 🚂 for your web journey
3. Cookie lives in **BROWSER** 🍪 | Session lives in **JVM** 📦
4. **WAS creates** the ID. IHS/Plugin just **CARRY** it 📮
5. `:0001` = CloneID = session affinity to the right JVM 🏠
6. 50,000 users = 50,000 sessions = heap pressure 💪
---

---

# Why NetBanking Absolutely Needs Sessions

## The Core Problem

Without sessions, every page of NetBanking would require Ravi to re-enter his password on every click. Clearly impossible.

But beyond login, sessions carry the bank's **critical state**.

## What a NetBanking Session Stores

```text
What a NetBanking session stores for Ravi:
├── Authentication state       → "Ravi is verified, 2FA passed"
├── Authorization context      → "Ravi can see Account ****1234 only"
├── Transaction state          → "Mid-NEFT: ₹50,000 to HDFC, pending OTP"
├── Account snapshot           → Current balance for this session
├── Basket/cart (credit card)  → Items added to credit card order
└── Audit trail start          → Session start time (RBI compliance)
```

## Breakdown

| Stored Data            | Purpose                                        |
|------------------------|------------------------------------------------|
| Authentication state   | Confirms Ravi is verified, 2FA passed          |
| Authorization context  | Limits visibility to Account ****1234 only     |
| Transaction state      | Tracks mid-NEFT transfer pending OTP           |
| Account snapshot       | Current balance for this session               |
| Basket/cart            | Credit card order items                        |
| Audit trail start      | Session start time for RBI compliance          |

## Why Losing a Session Is Critical

> [!WARNING]
> If any of this is lost mid-session, Ravi's transaction could go to limbo — money debited, beneficiary not credited.

> [!IMPORTANT]
> This is a **P1 incident** in any bank.
---

---

# Part 6: Quick Revision — Cheat Sheet 📋

## Session

- Server's **temporary memory** about a logged-in user.
- Solves HTTP's "stateless amnesia" problem.

## Session ID (JSESSIONID)

- Random, unique token.
- Data stays **on the server**; only the ID travels.

## Two Tracking Methods

| Method        | Description                                  |
|---------------|----------------------------------------------|
| Cookie        | Automatic, invisible, standard. All banks use this. |
| URL rewriting | Fallback, session ID in URL. **Dangerous.**  |

## Why URL Rewriting is Banned in Banking

- Leaks via **browser history**
- Leaks via **server logs**
- Leaks via **Referer header**
- Visible **on screen**

## Compliance

- PCI-DSS + RBI → **cookies only**, with `Secure` + `HttpOnly` flags.

## Session Timeout

```xml
<session-timeout>30</session-timeout>
```

- `30` = 30 **minutes**.
- **Idle-based** — resets on every request.
- RBI-mandated to block shared-PC attacks, hijacking, abandoned tabs.
---
