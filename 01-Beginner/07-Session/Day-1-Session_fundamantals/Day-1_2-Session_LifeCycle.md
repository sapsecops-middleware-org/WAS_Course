# 📊 Complete Session Birth-to-Death Visual

## Full Request Flow Diagram

```text
BROWSER          IHS              PLUGIN            JVM (WAS)
   │              │                  │                   │
   │─GET /login──►│                  │                   │
   │              │──forward────────►│                   │
   │              │                  │──route to JVM1───►│
   │              │                  │                   │
   │              │                  │            [Credentials OK]
   │              │                  │            [session = new HttpSession]
   │              │                  │            [JSESSIONID = A3F9X8B2:0001]
   │              │                  │            [session.setAttribute(userId)]
   │              │                  │◄──────────────────│
   │              │◄─Set-Cookie──────│                   │
   │◄─Set-Cookie──│                  │                   │
   │              │                  │                   │
   │ (User browses, each request carries cookie)         │
   │              │                  │                   │
   │─GET /balance─►│                 │                   │
   │ Cookie:A3F9..│──forward────────►│                   │
   │              │                  │──CloneID=0001────►│
   │              │                  │   (Sticky! Always JVM1)
   │              │                  │                   │
   │              │                  │            [session.getAttribute(userId)]
   │              │                  │            [Returns ₹2,40,000]
   │              │                  │            [lastAccess timer RESET]
   │◄─200 balance─│◄────────────────│◄──────────────────│
   │              │                  │                   │
   │─POST /logout─►│                 │                   │
   │              │                  │──────────────────►│
   │              │                  │            [session.invalidate()]
   │              │                  │            [Session DESTROYED]
   │              │                  │            [Memory FREED]
   │◄─302 /login──│◄────────────────│◄──────────────────│
```

## Phase Breakdown

### Phase 1 — Birth (Login)

| Step | Component | Action |
|------|-----------|--------|
| 1 | Browser | Sends `GET /login` with credentials |
| 2 | IHS → Plugin | Forwards request to WAS |
| 3 | Plugin | Routes request to JVM1 |
| 4 | WAS | Credentials verified → `HttpSession` created |
| 5 | WAS | `JSESSIONID = A3F9X8B2:0001` generated, user data stored via `setAttribute()` |
| 6 | WAS → Browser | `Set-Cookie` sent back through the chain |

### Phase 2 — Life (Browsing)

| Step | Component | Action |
|------|-----------|--------|
| 1 | Browser | Sends `GET /balance` with `Cookie: A3F9...` |
| 2 | Plugin | Reads CloneID `0001` → sticky routing to **JVM1 always** |
| 3 | WAS | `getAttribute(userId)` → returns ₹2,40,000 |
| 4 | WAS | **lastAccess timer RESET** (idle timeout restarts) |
| 5 | Browser | Receives `200 OK` with balance |

### Phase 3 — Death (Logout)

| Step | Component | Action |
|------|-----------|--------|
| 1 | Browser | Sends `POST /logout` |
| 2 | WAS | `session.invalidate()` called |
| 3 | WAS | Session **DESTROYED**, memory **FREED** |
| 4 | Browser | Redirected (`302`) to `/login` |
---
# JSESSIONID Lifecycle

## Overview

The `JSESSIONID` follows a simple lifecycle: it is **born** at login, stays **active** while the user browses, and is **destroyed** at logout or timeout.

## Lifecycle Diagram

```text
         LOGIN                  BROWSING               LOGOUT / TIMEOUT
           │                       │                         │
    WAS creates session      Browser sends cookie      Session destroyed
    JSESSIONID born          WAS finds session         Memory freed
    Set-Cookie sent          Data retrieved            Cookie deleted
           │                       │                         │
           ▼                       ▼                         ▼
       ACTIVE                   ACTIVE               INVALIDATED
```

## Stage 1 — Login (Birth) 👶

- WAS verifies user credentials
- WAS creates an `HttpSession` object in JVM heap
- WAS generates a unique `JSESSIONID`
- WAS sends the ID to the browser:

```http
Set-Cookie: JSESSIONID=A3F9X8B2:0001
```

**State:** `ACTIVE`

## Stage 2 — Browsing (Active Life) 🏃

- Browser automatically attaches the cookie on every request:

```http
Cookie: JSESSIONID=A3F9X8B2:0001
```

- WAS looks up the session by ID
- User data (userId, balance, etc.) is retrieved and used
- Every request updates the **last access time**

**State:** `ACTIVE`

## Stage 3 — Logout / Timeout (Death) 🗑️

The session ends in one of two ways:

| Cause   | Trigger                          | Result                          |
|---------|----------------------------------|---------------------------------|
| Logout  | User explicitly clicks Logout    | Session destroyed immediately   |
| Timeout | 30 min of inactivity (default)   | Session auto-expired by WAS     |

After destruction:

- Session object removed from JVM heap → **memory freed**
- Cookie is deleted / becomes invalid
- Reusing the old `JSESSIONID` → `"Session not found"` → user redirected to login

**State:** `INVALIDATED`

> [!TIP]
> Session timeout is not a bug — it is security working as designed.