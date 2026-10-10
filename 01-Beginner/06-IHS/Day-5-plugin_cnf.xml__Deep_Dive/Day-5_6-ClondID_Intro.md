# WebSphere Clone ID, JSESSIONID & Sticky Sessions — The Complete Guide

> [!NOTE]
> This document explains how session affinity (sticky sessions) works in an IBM WebSphere Application Server (WAS) clustered environment. It covers the role of the **Clone ID**, the anatomy of the **JSESSIONID** cookie, the **Web Server Plugin's** routing logic, and the most common failure scenarios seen in production.

---

## Table of Contents

- [1. Background: Why Multiple JVMs?](#1-background-why-multiple-jvms)
- [2. The Problem: Sessions Are Not Shared](#2-the-problem-sessions-are-not-shared)
- [3. The Solution: A Two-Half Handshake](#3-the-solution-a-two-half-handshake)
- [4. Clone ID: The JVM's Name Tag](#4-clone-id-the-jvms-name-tag)
- [5. JSESSIONID: The Wristband, Dissected](#5-jsessionid-the-wristband-dissected)
- [6. Complete Request Lifecycle](#6-complete-request-lifecycle)
- [7. Plugin Decision Tree](#7-plugin-decision-tree)
- [8. Failure Scenarios](#8-failure-scenarios)
- [9. Round Robin vs. Session Affinity](#9-round-robin-vs-session-affinity)
- [10. Hands-On: Where Everything Lives](#10-hands-on-where-everything-lives)
- [11. Quick Reference Card](#11-quick-reference-card)

---

## 1. Background: Why Multiple JVMs?

A single JVM (one application server process on one machine) has three fundamental problems in production:

| Problem | Impact |
|---|---|
| **Single point of failure** | Machine crash, OOM kill, or hung thread takes the entire application down |
| **Overloading** | High traffic (e.g., salary day) overwhelms one JVM → timeouts |
| **No patching window** | Restarting the only JVM kicks out all users |

The solution is a **cluster**: multiple identical JVMs running the same application across different machines.

```text
PaymentCluster
├── was1 — server1  (JVM1)
├── was2 — server1  (JVM2)
├── was3 — server1  (JVM3)
└── was4 — server1  (JVM4)
```

**Benefits:**

- One member dies → the rest keep serving users.
- Load spreads across all members.
- Members can be patched one at a time; the cluster stays up.

---

## 2. The Problem: Sessions Are Not Shared

Each JVM stores session data **in its own RAM**. It is **not shared** across the cluster.

When a user logs in, the JVM holds in memory:

- Authentication state ("user is logged in")
- Account details
- In-flight transaction state (e.g., mid-transfer)
- Wizard step / navigation state

If request #1 lands on `JVM3` and request #2 lands on `JVM1`:

```text
JVM1: "Session not found. Please log in again."
```

The user's transaction dies. Without affinity, this happens on **every click** if routing is random.

---

## 3. The Solution: A Two-Half Handshake

Session affinity is a cooperation between the JVM and the web server plugin:

| Question | Answer | Implemented by |
|---|---|---|
| "Which JVM should this request go to?" | Route by **Clone ID** | The Plugin (on the web server) |
| "How does the plugin know which JVM?" | The JVM stamps its Clone ID into the **JSESSIONID** cookie | The JVM itself |

> [!TIP]
> **The JVM writes the address; the Plugin reads the address.** Understand this handshake and everything else follows.

---

## 4. Clone ID: The JVM's Name Tag

### 4.1 Definition

A **Clone ID** is a short, unique string that identifies **one specific JVM** inside a cluster.

Analogy: four doctors all named "Dr. Sharma" (all JVMs run the same app). The receptionist routes you by **cabin number**. The Clone ID is that cabin number.

### 4.2 Where It Comes From

- WAS generates it **automatically** when the server is created.
- It is stored in the server's configuration (`server.xml`, attribute `cloneID`).
- When `plugin-cfg.xml` is generated from the DMGR, each server's Clone ID is written into its `<Server>` element.

```xml
<Server Name="was1_PaymentCluster_server1" CloneID="1a2b3c" .../>
<Server Name="was2_PaymentCluster_server1" CloneID="4d5e6f" .../>
<Server Name="was3_PaymentCluster_server1" CloneID="7g8h9i" .../>
<Server Name="was4_PaymentCluster_server1" CloneID="ab1cd2" .../>
```

You don't choose it or memorize it — you just need to know it **exists** and where to find it.

### 4.3 Properties

| Property | Rule | Why |
|---|---|---|
| **Uniqueness** | Must be unique per JVM in the cluster | Duplicate IDs → the plugin cannot distinguish JVMs → random logouts |
| **Stability** | Constant while the profile/server exists | Browsers hold cookies referencing it; changing it breaks existing sessions |
| **Visibility** | Appears in `plugin-cfg.xml` and in the JSESSIONID cookie | That is the entire mechanism |
| **Purpose** | Used **only** for session affinity routing | Not for security or load-balancing math |

### 4.4 Why "Clone" ID?

In legacy WebSphere setups, additional JVMs were created by literally **cloning** an existing server definition. Each clone needed a distinct identity — hence "Clone ID." The name stuck; today every cluster member gets one.

---

## 5. JSESSIONID: The Wristband, Dissected

### 5.1 What Is a Cookie?

A cookie is a small piece of text that:

1. The **server** sends to the browser (response header).
2. The **browser** stores.
3. The browser **automatically attaches** to every future request to that site.

### 5.2 Anatomy

When a user logs in, WAS responds with:

```http
Set-Cookie: JSESSIONID=0000r8KG7dLM9Xpq4vNZ2wTR:7g8h9i; Path=/NetBanking; Secure; HttpOnly
```

The value splits at the colon (`:`) into two parts:

```text
0000r8KG7dLM9Xpq4vNZ2wTR : 7g8h9i
─────────────────────────   ──────
        PART 1              PART 2
      Session ID            Clone ID
   "WHO you are"       "WHERE you live"
```

| Part | Reader | Question it answers |
|---|---|---|
| Left of `:` (Session ID) | The **JVM** | "Which session data should I load from RAM?" |
| Right of `:` (Clone ID) | The **Plugin** | "Which JVM should I send this request to?" |

> [!TIP]
> Memory trick — say it out loud:
> **JSESSIONID = WHO_YOU_ARE : WHERE_YOUR_SESSION_LIVES**
>
> One cookie, two readers, two purposes.

Internally, the owning JVM keeps a map such as:

```text
0000r8KG7dLM9Xpq4vNZ2wTR → { user: Ravi, account: 12345, amount: 200000, step: 2 }
```

### 5.3 Cookie Flags (Security)

| Flag | Effect |
|---|---|
| `Path=/NetBanking` | Browser sends the cookie only for URLs under that path |
| `Secure` | Cookie travels **only** over HTTPS |
| `HttpOnly` | JavaScript **cannot** read the cookie (XSS protection) |

> [!WARNING]
> Banks mandate `Secure` + `HttpOnly` on JSESSIONID. A security audit finding these flags missing results in a raised finding and a ticket to your team.

---

## 6. Complete Request Lifecycle

Example: a user performs a ₹2,00,000 fund transfer in 5 requests.

### Request 1 — Open Login Page

```text
Browser → GET /NetBanking/login
```

1. Plugin finds **no JSESSIONID cookie** (first visit).
2. No cookie → no affinity hint → **Round Robin** → lands on JVM2.
3. Login page served. **No session created yet** — the request is anonymous.

> [!NOTE]
> Before login, requests bounce around JVMs freely because there is nothing to lose. **Affinity only matters once a session exists.**

### Request 2 — Submit Credentials

```text
Browser → POST /NetBanking/authenticate
```

1. Still no cookie → Round Robin → lands on JVM3 (pure chance).
2. Credentials validated. **The session is born.** JVM3:
   - Creates the session object in RAM.
   - Generates the session ID: `0000r8KG7dLM9Xpq4vNZ2wTR`.
   - Stamps its own Clone ID: `7g8h9i`.
3. Responds with:

```http
Set-Cookie: JSESSIONID=0000r8KG7dLM9Xpq4vNZ2wTR:7g8h9i; Path=/NetBanking; Secure; HttpOnly
```

4. The browser saves the cookie.

### Request 3 — Open Fund Transfer Page

```text
Browser → GET /NetBanking/fundtransfer
Cookie: JSESSIONID=0000r8KG7dLM9Xpq4vNZ2wTR:7g8h9i
```

The plugin now performs its core job, in exact order:

1. Read the cookie → found.
2. Locate the `:` separator.
3. Extract everything **after** the colon → `7g8h9i`.
4. Match against `plugin-cfg.xml` → `was3_PaymentCluster_server1`.
5. Health check → JVM3 is alive.
6. **Route directly to JVM3** — Round Robin is bypassed.
7. JVM3 looks up the session ID (left of colon) in RAM → found → serves the page with the user's real data.

**This is the sticky session: not magic, an address lookup.**

### Request 4 — Enter Amount, Click Next

```text
Browser → POST /NetBanking/fundtransfer/step2SESSIONID=0000r8KG7dLM9Xpq4vNZ2wTR:7g8h9i
```

Plugin: `7g8h9i` → JVM3. JVM3 saves the amount into the session → serves the next step.

### Request 5 — Confirm

```text
Browser → POST /NetBanking/fundtransfer/confirm
Cookie: JSESSIONID=0000r8KG7dLM9Xpq4vNZ2wTR:7g8h9i
```

Plugin: `7g8h9i` → JVM3. JVM3 debits, credits, writes the audit trail, clears the pending transfer, and returns the success page.

**Result: 5 requests, 5 lookups, 5 arrivals at the same JVM.**

If **any** of those requests had gone to JVM1/2/4 instead:

```text
JVM1: "Session 0000r8KG...? Not in my memory."
→ HTTP 500 / "Session expired" → user logged out mid-transaction → incident ticket.
```

---

## 7. Plugin Decision Tree

Every request passing through the plugin follows this exact logic:

```text
REQUEST ARRIVES AT PLUGIN
        │
        ▼
Q1: Does the request carry a JSESSIONID cookie?
        │
        ├── NO ──► Round Robin across healthy cluster members. Done.
        │
        ▼ YES
Q2: Split the cookie at ':'.
    Is there a CloneID after the colon?
        │
        ├── NO ──► Round Robin. Done.
        │
        ▼ YES
Q3: Does this CloneID exist in plugin-cfg.xml?
        │
        ├── NO ──► Round Robin. Done.
        │           (Stale cookie after plugin regeneration!)
        │
        ▼ YES
Q4: Is that JVM currently ALIVE and healthy?
        │
        ├── NO ──► Round Robin across remaining healthy JVMs.
        │           ⚠️ Session data was in the dead JVM's RAM!
        │           Session replication may save the user (else logout).
        │
        ▼ YES
Q5: Is that JVM accepting work? (not stopped / quiesced / weight > 0)
        │
        ├── NO ──► Round Robin / next available server.
        │
        ▼ YES
ROUTE TO THE JVM WITH THE MATCHING CloneID.
Affinity honored. Session found.
```
---
## 8. Failure Scenarios

### Scenario A — JVM Dies Mid-Session

1. User is mid-transaction on JVM3; JVM3 crashes (OOM, machine failure).
2. Next request: plugin finds Clone ID `7g8h9i` in config, but the **health check fails** (Q4).
3. Plugin Round-Robins to another JVM → session not found → user logged out.

| Setup | Outcome |
|---|---|
| **With** Memory-to-Memory Session Replication | Sessions were copied to a backup JVM → user continues, possibly unnoticed |
| **Without** replication | Session lost. Period. |

**Ops procedure:** restart the failed JVM, determine root cause, verify replication config so the blast radius stays minimal.

### Scenario B — Duplicate Clone IDs

Caused by manually cloning a server and copying the Clone ID:

```xml
<Server Name="was_..." CloneID="1a2b3c"/>
<Server Name="was2_..." CloneID="1a2b3c"/>   <!-- DUPLICATE -->
```

The plugin's match becomes ambiguous → routing is effectively random → **intermittent session loss** ("sometimes it works, sometimes it doesn't" — the worst kind of ticket).

**Fix:** correct the Clone ID on the cloned server, regenerate and propagate `plugin-cfg.xml`.

> [!TIP]
> Interview line: *"Duplicate Clone IDs cause intermittent session loss because the plugin cannot deterministically resolve affinity."*

### Scenario C — Stale Cookie After Plugin Regeneration

1. Cluster rebuilt: new JVMs, new Clone IDs; `plugin-cfg.xml` regenerated and propagated.
2. A user's browser still holds the **old** cookie (old Clone ID).
3. Plugin lookup → **not found** (Q3) → Round Robin → user logs in once more → receives a new cookie with a valid Clone ID.

Self-healing, but produces a wave of "random logout" complaints after every change window. Expected for roughly one session-timeout window (~30 minutes). Brief the service desk.

### Scenario D — User Disables Cookies

- No cookie → the session ID cannot persist across requests.
- WAS fallback: **URL rewriting** — the session ID is appended to every URL:

```text
/NetBanking/fundtransfer;jsessionid=0000r8KG7dLM9Xpq4vNZ2wTR:7g8h9i
```

- The plugin identifies this via `AffinityURLIdentifier` in `plugin-cfg.xml` (default `;jsessionid=`) and parses the URL the same way it parses the cookie.

> [!WARNING]
> Banks typically **disable URL rewriting** and force cookies. URLs containing session IDs leak into server logs, bookmarks, and `Referer` headers — a security finding.

### Scenario E — Clone ID Missing Entirely

A JVM can be configured with an empty Clone ID (it is optional). Consequences:

- Its JSESSIONID has **no colon suffix** — just the session ID.
- The plugin has no affinity hint for that JVM's sessions → every subsequent request Round-Robins → **constant logouts** for users who landed there.

> [!NOTE]
> Rule: in a clustered environment, **every member must have a unique Clone ID**. On a single standalone server, it is irrelevant (no routing choice exists).

---

## 9. Round Robin vs. Session Affinity

**Round Robin** = distributing incoming requests across available servers one-by-one in a fixed rotating order, ignoring session history.

```text
JVM1 → JVM2 → JVM3 → JVM4 → JVM1 → JVM2 → ...
```

| Aspect | Round Robin | Session Affinity |
|---|---|---|
| Goal | Load distribution | Session continuity |
| Session awareness | None | Routes by Clone ID |
| Same user across requests | May hit different JVMs | Always the same JVM |
| When used | Default / fallback | Overrides Round Robin whenever a valid Clone ID exists |

> [!TIP]
> One sentence for interviews: *"Round Robin balances load; session affinity preserves state. Affinity wins when a Clone ID exists; Round Robin is the fallback when it doesn't."*

---

## 10. Hands-On: Where Everything Lives

| Artifact | Location | What You'll See |
|---|---|---|
| `plugin-cfg.xml` | IHS box: `$IHS_HOME/Plugins/config/<cell>/plugin-cfg.xml` | `<Server Name="..." CloneID="..." Weight="..."/>` per cluster member |
| JSESSIONID cookie | Browser DevTools → **Application/Storage → Cookies** | `value:colonsuffix` |
| `Set-Cookie` header | DevTools → **Network** → request → **Response Headers** | `JSESSIONID=...:...` |
| Session management config | DMGR Console → **Application servers → server1 → Session management, replication mode |
| Clone ID in server config | `server.xml` of the server definition | `cloneID="xxxxxx"` attribute |

> [!TIP]
> **Exercise:** open `plugin-cfg.xml` and find the Clone ID lines. Then log into a test application and compare the cookie's suffix. Seeing the **same string in both places** makes this concept permanent.

---

## 11. Quick Reference Card

```text
COOKIE:        WHO_YOU_ARE : WHERE_YOU_LIVE
               (session ID)   (Clone ID)

READERS:       JVM reads LEFT    → finds session data in RAM
               PLUGIN reads RIGHT → picks the route to the JVM

PLUGIN LOGIC:  No cookie            → Round Robin
               Clone ID not found   → Round Robin
               Clone ID's JVM down  → Round Robin (+ risk of session loss)
               Clone ID matched
               and JVM alive        → STICKY ROUTE ✔

FAILURES:      Dead JVM             → replication saves you
               Duplicate Clone ID   → random logouts
               Regenerated plugin   → stale cookies, brief complaint wave
               No cookies           → URL rewriting (;jsessionid=)
```
---
# 🖥️ View CloneID of each JVM in a cluster
```
Admin Console
  → Servers
    → Server Types
      → WebSphere Application Servers
        → Click each server (e.g. was1_PaymentCluster_server1)
          → Container Settings (left panel)
            → Web Container
              → Session management
                → Look for "Clone ID" field
```
Write down the CloneID for each JVM. They must ALL be different.

Change a CloneID (if two JVMs have the same one)

```
Admin Console
  → Servers → WebSphere Application Servers
    → Click the server with the duplicate CloneID
      → Web Container → Session management
        → Uncheck "Inherit session management" (if checked)
          → Look for Clone ID field
            → Clear it and enter a new unique value
              → OK → Save
                → Sync all nodes
                  → Regenerate plugin → Propagate to IHS
```
⚠️ After changing CloneID and regenerating plugin — all users currently on that JVM will lose their sessions (their old cookie has the old CloneID, the new plugin can't match it). Do this only during a maintenance window or when the server has zero active users.
