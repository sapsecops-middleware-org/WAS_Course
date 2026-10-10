# WebSphere Application Server — Session Affinity (Sticky Sessions)

A technical reference explaining HTTP statelessness, sessions, the `JSESSIONID`, `CloneID`, and how the WebSphere Plugin enforces Session Affinity in a clustered environment.

---

## 1. Background: HTTP is Stateless

By design, **HTTP has no memory**. Every request is treated as independent; the server forgets the client the moment the request completes.

> [!NOTE]
> Analogy: Calling a bank's customer care — if the call is transferred, the new agent has no idea what you already told the previous one. That is exactly how plain HTTP behaves.

---

## 2. Why Statelessness Breaks Transactional Applications

A multi-step transaction (e.g., a NEFT funds transfer) requires the server to retain context across requests:

| Step | User Action | Server Must Remember |
|------|-------------|----------------------|
| 1 | Login | Who is this user? |
| 2 | Enter amount | Who is doing this? |
| 3 | Enter beneficiary | What amount was entered? |
| 4 | Confirm | All of the above |
| 5 | Complete | Where to send confirmation |

Without persistent state:

- Transfers fail mid-flow.
- Requests may be attributed to the **wrong user**.

---

## 3. The Solution: HTTP Sessions

A **session** is a server-side container (a "locker") holding user-specific state. Created at login, it stores identity and in-progress transaction data.

- The server creates the session and places user data inside.
- The client receives a key — the **`JSESSIONID`** cookie.
- Every subsequent request presents the key, allowing the server to locate the session.

---

## 4. The Problem: Sessions vs. Round-Robin Load Balancing

In a clustered WAS topology, the web server plugin distributes requests across JVMs using Round Robin:

```text
Request 1 (Login)    → Round Robin → JVM2 → session created, JSESSIONID issued
Request 2 (Transfer) → Round Robin → JVM3 → "Who are you? No session here!"
```

**Result:** the user is logged out mid-transaction.

> [!IMPORTANT]
> This is the **Session Affinity Problem**. Round Robin spreads load; sessions require all requests to land on **one** JVM. They conflict by default.

---

## 5. The Fix: Session Affinity (Sticky Sessions)

**Rule:** Once a user's first request lands on a JVM and a session is created there, **all subsequent requests from that user must go to the same JVM**.

The WebSphere Plugin enforces this by parsing the `JSESSIONID`:

```text
1. Login request → JVM2 (no JSESSIONID yet → Round Robin applies)
2. JVM2 creates session → responds: JSESSIONID=ABC123:1a2b3c
3. Next request carries that cookie
4. Plugin reads the value after ":" → "This session belongs to JVM2"
5. Plugin routes the request to JVM2, bypassing Round Robin
6. All future requests → JVM2 → JVM2 → JVM2 ... (until logout or timeout)
```

---

## 6. Teller Analogy

| Bank | WebSphere |
|------|-----------|
| Token issued by teller | `JSESSIONID` |
| Teller's number printed on the token | `CloneID` |
| Queue manager routing by token | WebSphere Plugin |

- **Without affinity:** each visit lands on a different, clueless teller — the transaction never completes.
- **With affinity:** the token always routes you back to the same teller.

---

## 7. JSESSIONID Anatomy and CloneID

```text
JSESSIONID = ABC123XYZ : 1a2b3c
             ↑            ↑
      Random session ID   CloneID of the owning JVM
```

- The `:` is the separator.
- Everything **after** the `:` is the **CloneID**.

**Plugin routing logic:**

1. Read the `JSESSIONID`.
2. Extract the text after `:`.
3. Locate the `<Server>` in `plugin-cfg.xml` with that `CloneID`.
4. Route the request to that server.

Example `plugin-cfg.xml` server entries:

```xml
<Server Name="was1_server1" CloneID="1a2b3c" .../>
<Server Name="was2_server1" CloneID="4d5e6f" .../>
<Server Name="was3_server1" CloneID="7g8h9i" .../>
<Server Name="was4_server1" CloneID="ab1cd2" .../>
```

> [!TIP]
> Memorize: **`JSESSIONID` second half = CloneID = the sticky routing target.**

---

## 8. Failure Scenario: Owning JVM Crashes

| Option | Behavior | Outcome |
|--------|----------|---------|
| 1 | Return `503` | User sees an error page |
| 2 | Fall back to Round Robin | Routed to a healthy JVM **without the session** → user logged out |

Option 2 is the typical default, but the session data does not exist on the target JVM.

> [!IMPORTANT]
> Affinity determines **where to send** requests. Affinity alone cannot survive a JVM crash — you also need **Session Replication** (copying session data to peer JVMs).
> **Affinity = routing. Replication = data availability. You need both.**

---

## 9. Configuration (Two Places — Must Align)

### 9.1 The `<UriGroup>` / `<Uri>` Element

```xml
<UriGroup Name="NetBanking_URIs">
    <Uri AffinityCookie="JSESSIONID"
         AffinityURLIdentifier="jsessionid"
         Name="/NetBanking/*"/>
</UriGroup>
```

| Attribute | Purpose |
|-----------|---------|
| `AffinityCookie="JSESSIONID"` | Cookie used to extract the CloneID |
| `AffinityURLIdentifier="jsessionid"` | Fallback: parse `;jsessionid=` from the URL when cookies are disabled |

### 9.2 Unique `CloneID` per `<Server>`

Every JVM in the cell must have a **unique** CloneID:

- ✅ `CloneID="1a2b3c"`
- ✅ `CloneID="4d5e6f"`
- ❌ Duplicate CloneIDs → the plugin cannot distinguish the JVMs; affinity breaks and users experience random logouts.

---

## 10. Production Lesson: Cloning JVMs

Cloning a WAS profile by copying it wholesale can duplicate the **CloneID**:

- Two JVMs with identical CloneIDs.
- Plugin routes users randomly between them.
- Users logged out every few clicks.

**Fix:**

1. Change the CloneID in the WAS Admin Console on the cloned JVM.
2. Regenerate `plugin-cfg.xml`.

> [!WARNING]
> Whenever you clone or copy a JVM — **always verify and change the CloneID**.

---

## 11. Quick Reference Card

| Concept | One-Line Meaning |
|---------|------------------|
| HTTP stateless | Server has no memory between requests |
| Session | Server-side locker holding user state |
| `JSESSIONID` | The key to the locker (cookie) |
| `CloneID` | Part of `JSESSIONID` after `:` — identifies the owning JVM |
| Session Affinity | Same user → always the same JVM (sticky) |
| Plugin's job | Read CloneID → route to the correct JVM |
| Affinity failure on crash | User logged out → fixed by Session Replication |
| Config locations | `AffinityCookie` on `<Uri>` + unique `CloneID` per `<Server>` |

---

## 12. Self-Test Questions

1. Why is HTTP statelessness a problem for a banking application?
2. What is the `JSESSIONID`, and what are its two parts?
3. What does the plugin do when it reads the CloneID?
4. What happens when the JVM owning the session crashes?
5. Why must CloneIDs be unique across all JVMs in the cell?

---
# View CloneID of a JVM
```
Admin Console
  → Servers → Server Types → WebSphere Application Servers
    → Click was1_PaymentCluster_server1
      → "Web Container" (under Container Settings in left panel)
        → Look for "Session management"
          → You will see the Clone ID used for this server
```
## Change CloneID (if two JVMs accidentally have the same one)
```
Admin Console
  → Servers → Server Types → WebSphere Application Servers
    → Click the server
      → Web Container → Session Management
        → Uncheck "Enable cookies"... wait — look for Clone ID field
          → Change it to a unique value
            → Save → Sync nodes → Regenerate and propagate plugin
```