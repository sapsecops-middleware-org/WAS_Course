# WebSphere DataSource & Connection Pooling — DigiBank

```
                        ┌──────────────────────────────────────┐
                        │           WEBSPHERE SERVER            │
                        │                                       │
   ┌──────────┐         │   ┌─────────────┐    ┌────────────┐  │
   │ DigiBank │  lookup │   │    JNDI     │    │ Connection │  │
   │   App    │────────────────▶ Phone book │    │    POOL    │  │
   │ (WAR)    │         │   │             │    │            │  │
   └──────────┘         │   │ jdbc/       │    │ ┌──┐┌──┐┌──┐  │
                        │   │ DigiBankDB  │    │ │c1││c2││c3│  │
                        │   └─────────────┘    │ └──┘└──┘└──┘  │
                        │                      └──────┬───────┘  │
                        └──────────────────────────────┼─────────┘
                                                       │
                                                       ▼
                                          ┌─────────────────────┐
                                          │      ORACLE DB      │
                                          │     (The Vault)     │
                                          └─────────────────────┘

```

## 3. What Happens When DigiBank Starts?

```

   START
     │
     ▼
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│  1. READ CONFIG │      │ 2. REGISTER     │      │ 3. WARM-UP      │
│  resources.xml  │─────▶│    IN JNDI      │─────▶│ Create min 10   │
│  name, URL,     │      │ jdbc/DigiBankDB │      │ connections     │
│  user, password │      │ is findable ✅  │      │ in ADVANCE      │
└─────────────────┘      └─────────────────┘      └────────┬────────┘
                                                           │
                                                           ▼
                                                  ┌─────────────────┐
                                                  │ 4. KEEP ALIVE   │
                                                  │ Connections sit │
                                                  │ READY & waiting │
                                                  └─────────────────┘

   🍞 Bakery analogy: Bread is baked BEFORE the shop opens

```

WebSphere does **4 things** at startup:

1. **Reads config** — from the config folder (`resources.xml` etc.). It finds your DataSource definition: name, Oracle URL, username, password.
2. **Registers in JNDI** — now `jdbc/DigiBankDB` is findable by name.
3. **Warm-up** — creates a few DB connections in advance (e.g., minimum 10). Not waiting for requests.
4. **Keeps them alive** — connections stay open and ready.

### Why warm-up?
- First customer of the morning should **not wait** for a DB connection to be created.
- Connections are ready **before the doors open** — like a bakery baking bread before the shop opens.

---

## 4. What Happens When a Customer Request Comes In?

```
 CUSTOMER                APP CODE              WEBSPHERE              POOL
    │                       │                       │                  │
    │  "Check balance"      │                       │                  │
    │──────────────────────▶│                       │                  │
    │                       │ 1. lookup(            │                  │
    │                       │  "jdbc/DigiBankDB")   │                  │
    │                       │──────────────────────▶│                  │
    │                       │                       │ 2. Check web.xml │
    │                       │                       │    mapping       │
    │                       │                       │ 3. Resolve DS    │
    │                       │                       │ 4. BORROW conn ──┼──▶ ┌────┐
    │                       │                       │                  │    │ c2 │
    │                       │  5. Run SQL ◀─────────┼──────────────────┼─── └────┘
    │                       │  (SELECT balance)     │                  │
    │                       │                       │ 6. RETURN conn ──┼──▶ ┌────┐
    │  Balance shown ✅     │                       │   (NOT closed!)  │    │ c2 │
    │◀──────────────────────│                       │                  │    └────┘

```


Step by step:

1. **App asks** — code calls `ctx.lookup("java:comp/env/jdbc/DigiBankDB")`.
2. **Resource reference check** — the app's `web.xml` says: *"I call it `jdbc/DigiBankDB` locally, map it to the real one."*
   - **Why?** So developers don't hardcode real names App code stays portable.
3. **WebSphere resolves it** — finds the real DataSource it configured.
4. **Borrows a connection** — from the pool (**not a new one!**).
5. **Uses it** — runs the SQL (fetch balance, transfer money, etc.).
6. **Returns it** — connection goes back to the pool. **Not closed — just returned.**

### Real-life example 📚
Library book. You borrow it, read it, return it. Next person borrows the same book.
**The library doesn't buy a new book for every reader.**

---

## 5. Connection Pooling — The Heart of It

- WebSphere keeps a **pool of connections**.
- Apps **borrow and return** — they never create or destroy.
- Pool size is configured: minimum, maximum, etc.

### Key settings (remember these names)

| Setting | Meaning |
|---|---|
| **Minimum connections** | Kept ready always (warm-up) |
| **Maximum connections** | Hard ceiling — pool can't grow beyond this |
| **Connection timeout** | How long a request waits for a free connection |
| **Unused timeout** | Idle connections get cleaned up |

---

## 6. ⚠️ The Golden Production Insight

> **WebSphere does NOT create a new Oracle connection per request. It REUSES pooled connections.**

### Why this matters
- Creating a DB connection is **expensive** (network handshake, login, memory).
- Doing that for every request would **kill performance**.
- **Pooling = reuse = fast.**

### The sizing math (interview gold 💰)
- ❌ 10,000 online users ≠ 10,000 DB connections
- A request uses a connection only for a **few milliseconds**
- So **50–100 connections can serve 10,000 users**
- This is how you size `Maximum connections` in production

### Real-life example 🏦
A bank with 10,000 customers doesn't hire 10,000 tellers.
At any moment, only ~50 customers are actually being served.
**50 tellers handle everyone** — customers just wait in line briefly.
That line = **connection timeout**.

---

## 7. Quick Memory Card 🧠

| Concept | Remember It As |
|---|---|
| **DataSource** | Named doorway to DB |
| **JNDI** | Phone directory of resources |
| **Startup** | read config → register JNDI → warm pool |
| **Request** | lookup → map via web.xml → borrow → use → return |
| **Pooling** | borrow/return, never create/destroy |
| **Sizing** | 10,000 users → maybe just 50 connections |
