# How IHS Work Internally explain 


```text
                          🏦 DIGIBANK — ONE PICTURE, TOTAL MEMORY 🏦

 Step 1: DNS                          Step 2: Load Balancer
┌─────────────────────┐              ┌──────────────────────────┐
│  DNS = Phone Book   │              │  VIP: 203.0.113.50       │
│  www.digibank.com   │── name→IP ──▶│  One public door.        │
│  → 203.0.113.50     │              │  Two IHS behind it.      │
└─────────────────────┘              │  (No single point of     │
                                     │   failure)               │
                                     └────────────┬─────────────┘
                                                  │
                       ┌──────────────────────────┴───────────┐
                       ▼                                      ▼
              ┌─────────────────┐                   ┌─────────────────┐
              │     IHS01       │                   │     IHS02       │
              │  192.168.1.10   │                   │  192.168.1.11   │
              │                 │                   │                 │
              │ ┌─────────────┐ │                   │ ┌─────────────┐ │
              │ │ PARENT      │ │  root             │ │ PARENT      │ │
              │ │ (Manager)   │ │  reads httpd.conf │ │ (Manager)   │ │
              │ │ supervises  │ │  grabs ports      │ │ supervises  │ │
              │ └──────┬──────┘ │                   │ └──────┬──────┘ │
              │        │        │                   │        │        │
              │ ┌──────┴──────┐ │                   │ ┌──────┴──────┐ │
              │ │ CHILDREN    │ │  ibmhttpd user    │ │ CHILDREN    │ │
              │ │ (Tellers)   │ │  threads serve    │ │ (Tellers)   │ │
              │ │  serve      │ │  real requests    │ │  serve      │ │
              │ └─────────────┘ │                   │ └─────────────┘ │
              └────────┬────────┘                   └────────┬────────┘
                       │                                     │
     Port 80 ──────────┤  ← redirect door only ("go to https")
     Port 443 ◀────────┘  ← REAL customer traffic + certificate
                       │
                       ▼
        ┌─────────────────────────────────────┐
        │  VIRTUAL HOSTS (apartment building) │
        │  IHS reads Host header → picks flat │
        │                                     │
        │  www.digibank.com       → Retail    │
        │  payments.digibank.com  → Payments  │
        │  corporate.digibank.com → Corporate │
        └──────────────────┬──────────────────┘
                           │
                           ▼  IHS Plugin (receptionist)
        ┌─────────────────────────────────────┐
        │  INTERNAL CORRIDOR — Port 9080/9443 │
        │  🔥 NEVER open to internet.         │
        └───────┬─────────────────────┬───────┘
                ▼                     ▼
        ┌──────────────┐      ┌──────────────┐
        │ WAS Server01 │      │ WAS Server02 │
        │  (9080)      │      │  (9080)      │
        └──────────────┘      └──────────────┘
```

---

## How IHS Work Internally  — IHS Processes (Parent and Child)

### What is a Process?

- A **process** = a running program.
- When you start IHS, it does **not** run as ONE.
- It runs as a **family** → one **parent** + many **children**.

---

### The Parent Process — "The Manager"

Think of the parent as a **bank branch manager**.
The manager does **NOT** serve customers. The manager **supervises**.

**The parent:**

- Starts first when you type `apachectl start` or when IHS starts.
- Reads the config file (`httpd.conf`). All rules come from here.
- Creates child processes.
- Watches the children. If one dies → it creates a new one **automatically**.
- Runs as **root**.

**Why root?**

> Only root can use ports below 1024. Ports **80 and 443** are below 1024.
> So the parent needs root **only to grab these ports**.

---

### The Child Processes — "The Tellers"

Think of children as **bank tellers**.
They actually serve customers.

**The children:**

- Handle **all real customer requests**.
- Each child has many **threads** (one thread = one customer at a time).
- Run as **ibmhttpd** user — a **low-privilege** user.

**Why low-privilege user?**

> Security. If a hacker attacks a child process, the hacker only gets the
> `ibmhttpd` user — not root. Damage is limited.
> **Golden rule in banking: never serve customers as root.**

---

### Simple Diagram

```text
Parent () — Manager: reads config, supervises
   ├── Child 1 (ibmhttpd) — Teller: serves customer A, B, C
   ├── Child 2 (ibmhttpd) — Teller
   └── Child 3 (ibmhttpd) — Teller
```

---

### Why This Design? (Interview Answer)

| Reason | Explanation |
|--------|-------------|
| **Stability** | One child crashes → only that child dies. Parent restarts it in seconds. Customers don't notice Bank stays online. |
| **Security** | Root manages only. Children (customer-facing) run with least privilege. |
| **Performance** | Threads handle many requests using less memory than full processes. |

---

### Commands You Should Know

```bash
ps -ef | grep httpd    # See parent + child processes
# Parent shows:  root     httpd
# Children show: ibmhttpd httpd
```

> ⚠️ If you ever see IHS children running as **root** — that is a security problem. Fix it.

---

### One Practical Tip

> - **Parent dies → everything dies.**
> - **Children die → parent heals itself.**
>
> So **monitor the parent process**. That is the heart.

---

## 6 — IHS Ports

### What is a Port?

- A **port** = a **door number** on the server.
- Traffic comes in through specific doors.

---

### Port Table

| Port | | Used For |
|------|----------|----------|
 **80**   | HTTP  | Plain, unencrypted. **Redirect door only.** |
| **443**  | HTTPS | Encrypted. **Real customer traffic.** |
| **9080** | HTTP  | Internal. **IHS Plugin → WAS.** |
| **9443** | HTTPS | Internal encrypted to WAS. |

---

### The Golden Rule

- **Port 80** — Customer types `digibank.com` (no https). IHS and immediately says:
  *"Go to https instead."* → **Redirect. Nothing real happens here.**
- **Port 443** — All real customer traffic. **SSL/TLS encrypted.** Your **certificate** lives here.
- **Port 9080** — Secret **internal** door. IHS Plugin sends requests to WebSphere here.
  **Never open to the internet. Firewall it off.**

---

 DigiBank Flow (Memorize This)

```text
Customer Browser
      │
      ▼
Port 443 (IHS)  ← encrypted, certificate checked
      │
      ▼  (IHS Plugin decides which server)
Port 9080 → WAS Server01
Port 9080 → WAS Server02
```

---

### Story Version 🏦

> - Customer walks into the bank through the **secure front door443)**.
> - The **receptionist (Plugin)** sends them to the correct **back-office officer (WAS Server01 or Server02)** through an **internal corridor (9080)**.
> - Customers **never touch the internal corridor directly**.

---

### Why Two WAS Servers?

- **Load balancing.** Plugin spreads requests between Server01 and Server02.
- If one dies, the other keeps serving.
- Same idea as child processes → **never a single point of failure.**

---

## Quick Revision (5 Lines) ✅

1. **Parent** = manager = root = reads config + supervises.
2. **Children** = tellers = ibmhttpd = serve requests using threads.
3. Child crashes → parent restarts it → bank stays up.
4. **80** = redirect only. **443** = real traffic. **9080/9443** = internal only.
5. **Never expose 9080 to the internet. Firewall it.**
---
# PART 7 & 8 — Virtual Hosts and DNS (Simple Training)

> **Senior WAS Notes** — Explained for fresh admins in banking. Simple. Step by step.

---

## PART 7 — Virtual Hosts

### What is a Virtual Host?

- **One IHS server** (one machine, one IP) can host **many websites**.
- Each website = one **Virtual Host** block in `httpd.conf`.
- Think of it like **one apartment building, many flats**. One address (IP). Many families (websites) living inside.

---

### DigiBank Example

```text
One IHS Server — one machine, one IP
   ├── www.digibank.com        → Retail banking portal
   ├── payments.digibank.com   → Payment portal
   └── corporate.digibank.com  → Corporate banking portal
```

Three portals. One server. Same IP. Same ports. Still separate.

---

### How Does IHS Know Which Site the Customer Wants?

**Answer: the Host Header.**

Every browser request carries a small line called the **Host header**. It tells the server *which name the customer typed*.

```text
GET /internetbanking HTTP/1.1
Host: www.digibank.com
```

IHS reads `Host:` → matches it to the right **VirtualHost block** → serves that site.

```text
GET /payments HTTP/1.1
Host: payments.digibank.com
```

IHS sees `payments.digibank.com` → payments VirtualHost → Plugin forwards to `payments.war` on WAS.

---

### Bank Story 🏦

> One bank building. Three departments inside: Retail, Payments, Corporate.
> Customer walks in and says **"I'm here for retail banking"** (the Host header).
> Receptionist (IHS) sends them to the right counter (VirtualHost).

---

### Why This Matters in Production

- **Save money** — one server serves many portals.
- **Separate configs** — each portal can have its own rules, logs, certificates.
- **Easy maintenance** — change payments site without touching retail site.

---

### Quick Interview Tip

> **Q: How does name-based virtual hosting work?**
> "IHS reads the **Host header** in the HTTP request and matches it against the
> `ServerName` in the VirtualHost blocks. Name-based hosting allows many sites on one IP."

---

## PART 8 — DNS Relationship

### What is DNS?

- **DNS = the internet's phone book.**
- Humans remember names (`www.digibank.com`). Computers talk using **IP addresses**.
- DNS translates: **name → IP**.

---

### What Happens When a Customer Types the URL?

Customer types:

```text
https://www.digibank.com
```

Step by step:

1. Browser asks DNS: *"What is the IP of www.digibank.com?"*
2. DNS answers: `203.0.113.50` (Load Balancer IP).
3. Browser sends the request to that IP.
4. Load Balancer forwards to a real IHS server.

```text
www.digibank.com → 203.0.113.50 (Load Balancer VIP)
                        → IHS01 (192.168.1.10)
                        → IHS02 (192.168.1.11)
```

---

### In Your DigiBank Lab (Simple Setup)

No load balancer in the lab, so DNS points straight to IHS:

```text
www.digibank.com → 192.168.1.10 (IHS VM1)
```

For lab practice, you can test without real DNS by editing the **hosts file**:

- Linux: `/etc/hosts`
- Windows: `C:\Windows\System32\drivers\etc\hosts`

Add a line:

```text
192.168.1.10   www.digibank.com
```

Now your browser finds your lab IHS without any real DNS. Every WAS admin does this in labs.

---

### Why Two IHS Servers Behind a Load Balancer?

- **Load Balancer VIP** = one public IP that customers always use.
- Behind it: IHS01 + IHS02.
- One IHS dies → Load Balancer sends traffic to the other. **Customers never notice.**
- Same pattern as everything else in this course: **no single point of failure.**

---

### DigiBank Full Picture So Far

```text
Customer types www.digibank.com
        │
        ▼
     DNS  (name → IP)
        │
        ▼
Load Balancer VIP (203.0.113.50)
        │
   ┌────┴────┐
   ▼         ▼
 IHS01     IHS02      (Port 443)
   └────┬────┘
        ▼  (Plugin)
WAS Server01 / Server02   (Port 9080)
```

---

## Quick Revision (5 Lines) ✅

1. **Virtual Host** = many websites on one IHS server.
2. IHS picks the right site by reading the **Host header**.
3. **DNS** = phone book: translates `www.digibank.com` → IP address.
4. Lab trick: use the **hosts file** instead of real DNS.
5. Load Balancer VIP + two IHS servers = **no single point of failure**.
