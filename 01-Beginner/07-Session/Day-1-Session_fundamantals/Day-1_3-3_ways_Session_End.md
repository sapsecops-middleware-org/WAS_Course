# Three Ways a Session Ends

## Overview

An HTTP session can die in three different ways — by user action, by timeout, or by server restart.

## Comparison Table

| Method              | What Happens                          | Real Bank Scenario                     |
|---------------------|---------------------------------------|----------------------------------------|
| Explicit logout     | App calls `session.invalidate()`      | Ravi clicks "Logout"                   |
| Inactivity timeout  | WAS invalidates after N minutes       | Ravi left his laptop open in a café    |
| JVM restart         | All sessions in that JVM's memory wiped | Admin restarts JVM for deployment    |

## 1. Explicit Logout

- User intentionally ends the session
- Application calls `session.invalidate()`
- Session object is removed from JVM heap immediately

## 2. Inactivity Timeout

- No clicks from the user for N minutes (default: 30)
- WAS automatically invalidates the session
- Next request → `"Session expired, please login again"`

> [!NOTE]
> This is security working as designed — protects users who walk away without logging out.

## 3. JVM Restart

- All sessions live in JVM **heap memory**
- When the JVM stops, all sessions in memory are wiped
- Every logged-in user on that JVM is logged out

> [!WARNING]
> Users receive random logouts during deployments unless session persistence or replication is configured.
---
# Part 5: Session Timeout — The Auto-Logout Rule ⏱️

A session can't live forever. If it did:

- You log in at a cyber café, walk away.
- Next person sits down — **still logged into your bank account**. Nightmare.

So every session has an expiry timer. In WAS, it's set in `web.xml` or via the admin console:

```xml
<session-timeout>30</session-timeout>
```

**What this means:**

- `30` = **minutes**.
- The timer counts from your **last activity**, not from login.

## The 30-Minute RBI-Recommended Rule — Ravi's Day

| Time          | Action                 | Result                                        |
|---------------|------------------------|-----------------------------------------------|
| 10:00 AM      | Ravi logs in           | Session created                               |
| 10:05 AM      | Checks balance         | Session alive, timer resets to 0              |
| 10:05 – 10:36 | Ravi at lunch, idle    | Timer keeps running                           |
| 10:36 AM      | Ravi clicks "Transfer" | Server checks: *"31 minutes idle. Session dead."* |
|               |                        | *"Please log in again."*                      |

## Key Insight — Idle Timeout, Not Absolute Timeout

- Every click **resets** the timer.
- Only **inactivity** kills the session.
- If Ravi kept clicking every 10 minutes, he'd stay logged in for hours.

## Why 30 Minutes? Why Is This a Security Requirement, Not Just UX?

> [!NOTE]
> RBI mandates it to protect against:

1. **Shared computer attacks** — cyber cafés, office common PCs. Dead session = next user can't reuse it.
2. **Session hijacking** — a stolen session ID is useless after 30 idle minutes. Smaller window for attackers.
3. **Abandoned tabs** — you left the bank tab open at home/office. Auto-closed = safe.

> [!IMPORTANT]
> **Senior admin gold:** when a session dies, the server-side data is **deleted**. The login state is gone. That's why you must re-authenticate — the server genuinely doesn't remember you anymore. It's not a trick.

---
# Setting Session Timeout

## 1. Via Admin Console

### Login

```text
https://was-host:9043/ibm/console
```

### Navigation

```text
Servers
  └─► Server Types
        └─► WebSphere Application Servers
              └─► [your server name, e.g. server1]
                    └─► Container Settings
                          └─► Session Management
```

### Key Fields to Configure

| Field                  | Value        | Why                        |
|------------------------|--------------|----------------------------|
| Enable cookies         | ✅ Checked   | Industry standard          |
| Cookie name            | JSESSIONID   | Default — keep it          |
| Maximum session count  | 50000        | For 50K bank users         |
| Session timeout        | 30 minutes   | RBI requirement            |
| Allow overflow         | Unchecked    | Prevent unbounded growth   |

### Save & Apply

```text
Click Apply → Click Save → Restart JVM for it to take effect.
```

---

## 2. At Application Level (`web.xml` — overrides server)

Inside the WAR/EAR's `web.xml`:

```xml
-config>
    <session-timeout>30</session-timeout>
</session-config>
```

> [!IMPORTANT]
> **Application-level (`web.xml`) overrides server-level (Console).**
>
> In a bank, always align both. Disagreement = confusion in audits.
---
