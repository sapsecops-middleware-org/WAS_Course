# DMZ Architecture — Explained From Zero

> [!NOTE]
> This document explains DMZ (Demilitarized Zone) network architecture from first principles, using a real-world banking example with IBM HTTP Server (IHS) and WebSphere Application Server (WAS).

---

## 1. What Problem Does DMZ Solve?

Think of a bank office:

- Customers walk in through the **front door**.
- They talk to the **receptionist**.
- The **vault**? It is behind three locked doors. Customers never touch it.

Now imagine the opposite — customers walking straight into the vault. Disaster.

Computers have the same problem:

- A bank's **website** must be reachable by the outside world.
- The **database** — holding everyone's money and account details — must **never** be reachable from outside.

The DMZ is the **"reception area"** between the street and the vault.

**The problem in one line:**

> How do we let strangers talk to our website WITHOUT letting them anywhere near our sensitive systems?

**DMZ solves this.**

---

## 2. What Is a DMZ?

The name comes from war. A DMZ (Demilitarized Zone) is a **"no man's land"** between two armies — nobody fully owns it.

In networking:

- A DMZ is a small, **separate network zone** where we place servers that the internet must talk to.
- Internet traffic comes **INTO** the DMZ.
- But it **stops there**. It cannot go further in.
- If a hacker breaks into a DMZ server, they are stuck in the "reception area." The vault is still safe.

---

## 3. The Three Zones of a Bank Network

```text
╔══════════════════╗     ╔══════════════════╗     ╔══════════════════╗
║    ZONE 1        ║     ║    ZONE 2        ║     ║    ZONE 3        ║
║    INTERNET      ║     ║    DMZ     ║  INTERNAL NET    ║
║                  ║     ║                  ║     ║                  ║
║  • Customers     ║     ║  • IHS lives     ║     ║  • WAS lives     ║
║  • Hackers       ║──▶  ║    here          ║──▶  ║  • DB lives      ║
║  • Bots          ║     ║  • Only 80/443   ║     ║  • Core Banking  ║
║  • Anyone        ║     ║    allowed in    ║     ║  • Strictly      ║
║                  ║     ║                  ║     ║    internal      ║
╚══════════════════╝     ╚══════════════════╝     ╚══════════════════╝
         │                        │                        │
    [Outer Firewall]         [Inner Firewall]
    Blocks everything        Blocks everything
    except 80/443            except 9080 from IHS
```

### Zone 1 — The Internet (Danger Zone)

- Anyone: customers, hackers, bots, scanners.
- Assume everything coming from here is **hostile**.
- Zero trust.

### Zone 2 — The DMZ (Semi-Safe Zone)

- Public-facing servers live here.
- In our world: **IHS (IBM HTTP Server)** sits here.
- It receives customer requests and forwards them inside.

### Zone 3 — Internal Network (The Vault)

- **WAS (WebSphere Application Server)** — logic.
- **Database** — holds account data, balances, transactions.
- Core banking systems.
- **No direct internet access. Ever.**

---

## 4. The Two Firewalls — The Guards

A firewall is like a security guard with a checklist. Traffic arrives, the guard checks:

1. "Where are you coming from?" (source IP)
2. "Where are you going?" (destination IP)
3. "Which door?" (port number)

If it's not on the approved list → **blocked**.

### 🛡️ Outer Firewall (Internet → DMZ)

**Allows:**

| Port  | Protocol | Source   |
|-------|----------|----------|
| 80    | HTTP     | Anywhere |
| 443   | HTTPS    | Anywhere |

**Blocks everything else:**

- Port 22 (SSH) ❌
- Port 9080 (WAS) ❌
- Port 1521 (Oracle DB) ❌
- Port 3389 (Remote Desktop) ❌

Meaning: the outside world can only "talk web" to your DMZ. They cannot log in, cannot probe the database, cannot do anything else.

### 🛡️ Inner Firewall (DMZ → Internal Network)

**Allows:**

- Port 9080 (HTTP traffic to WAS) — but **ONLY**:
  - **FROM:** the IHS server's IP
  - **TO:** the WAS server's IP
- Nothing else. Not even another DMZ server.

**Blocks:**

- Internet IPs ❌
- Other DMZ servers ❌
- Database ports ❌

### 🎯 The Golden Result

Even if a hacker **fully takes over** the IHS server in the DMZ:

- They **cannot reach** WAS.
- They **cannot reach** the database.

The inner firewall simply doesn't allow it. The hacker is trapped in the reception area.

> [!TIP]
> This is called **defense in depth** — multiple layers of security, so one failure doesn't sink you.

---

## 5. Why IHS Is Perfect for the DMZ

IHS is basically a web server (Apache-based, tuned by IBM). It is built to face the internet.

| Feature | Why It Matters in the DMZ |
|---------|---------------------------|
| Serves only HTTP/HTTPS | Needs only ports 80/443 → minimal firewall openings |
| No business logic inside | If hacked, attacker finds nothing valuable |
| Can filter/block bad requests | Attacks die in the DMZ, before touching WAS |
| SSL termination | Handles HTTPS in the DMZ; WAS talks plain HTTP internally |

### SSL Termination (Very Common Interview Question)

1. Customer → browser uses **HTTPS** (encrypted).
2. The encryption must be "unwrapped" somewhere so the server can read the request.
3. **IHS does this unwrapping in the DMZ.**
4. From IHS → WAS, traffic goes as **plain HTTP over port 9080**.
5. This is safe because the inner firewall protects that path — it is inside our walls.

**So: HTTPS outside, HTTP inside. IHS is the handshake point.**

---

## 6. Full Flow of a Banking Request (Follow the Money 💰)

A customer checks their balance:

1. Customer opens browser → `https://www.mybank.com/balance`
2. Outer firewall → "Port 443? Allowed. Come in."
3. IHS in DMZ receives it → **unwraps SSL** → checks the request isn't garbage.
4. Inner firewall → "Source is IHS, destination is WAS, port 9080? Allowed."
5. WAS runs the banking logic → needs balance → asks the database (internal, no firewall crossing to internet).
6. DB replies → WAS replies → IHS wraps in HTTPS → customer sees balance.

**The customer never directly touches WAS or the DB. Ever.**

---

## 7. Real Banking Reality — How Firewall Changes Actually Work

At a bank like Citibank, you **cannot** just open a firewall port. Here is the process, lived through many times:

1. You're adding a new JVM/clone to your WAS cluster.
2. IHS must now route traffic to this new server → **port 9080 must be opened on the inner firewall**.
3. You raise a **firewall change request ticket**.
4. It goes to the **network security team** for review.
5. Then to the **CAB (Change Advisory Board)** — a meeting where all changes are approved.
6. Only after approval → the network team implements the rule.

> [!IMPORTANT]
> **Takeaway as admin:** Adding a new member to a cluster isn't just clicking buttons. It involves tickets, approvals, and coordination with security and network teams. This is exactly why banks plan cluster expansions carefully.

---

## Summary

| Concept | Role |
|---------|------|
| DMZ | Buffer zone between internet and internal network |
| Outer Firewall | Allows only 80/443 into the DMZ |
| Inner Firewall | Allows only IHS → WAS on 9080 |
| IHS | Internet-facing web server; terminates SSL |
| WAS | Runs business logic inside the vault |
| Defense in Depth | Multiple layers so one breach doesn't sink you |
