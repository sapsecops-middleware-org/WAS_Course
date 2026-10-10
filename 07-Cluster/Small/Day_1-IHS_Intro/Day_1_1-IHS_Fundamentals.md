# 🏦 WebSphere Administrator Training — DigiBank

## LESSON 1 — IBM IHS Fundamentals + Complete Request Flow

---

## PART 1 — What is IBM HTTP Server (IHS)?

### The Real-Life Problem

DigiBank has thousands of customers.

A customer opens their browser and types:

```bash
https://www.digibank.com/internetbanking
```

This request must reach the **DigiBank application** running inside a **Java server** (WebSphere Application Server — WAS).

---

### ❌ The Golden Rule: Never Expose WAS Directly to the Internet

Why not?

- Not designed to handle raw internet traffic
- No SSL certificate management
- No security hardening
- Cannot serve static files efficiently
- Direct exposure = huge security risk

**Solution:** Put **IBM HTTP Server (IHS)** in front of WAS.

---

### What is IHS?

**IBM HTTP Server** is a **web server** — a program that:

1. **Listens** for incoming HTTP/HTTPS requests from browsers
2. **Decides** what to do with each request:
   - Static file (HTML, CSS, image)? → **IHS serves it itself**
   - Dynamic request (login, balance, transfer)? → **IHS forwards it to WAS**
3. **Protects** — handles SSL, security, hides the backend

---

### 🚪 The Front Door Analogy (Remember This Forever!)

```
Customer (Browser)
      ↓
Security Gate (IHS)        ← Front door of the building
      ↓
Bank Teller Counter (WAS)  ← Where real work happens
      ↓
DigiBank Application       ← Java business logic
```

| Role | Does | Does NOT |
|------|------|----------|
| **Security Gate (IHS)** | Checks every request, handles HTTPS, routes traffic | Does NOT do banking work |
| **Teller Counter (WAS)** | Runs Java app, processes accounts/payments | Not exposed to internet |

> 💡 **One line to remember:** *IHS is the front door. WAS is the worker behind the door.*

---

## PART 2 — IHS vs Apache HTTP Server

### Key Fact

**IHS is built ON TOP of Apache HTTP Server.**

IBM took Apache → added enterprise features → sold it as IHS with full support.

### 📊 Comparison Table (Memorize This!)

| Feature | Apache HTTP Server | IBM HTTP Server (IHS) |
|---------|-------------------|----------------------|
| **Base** | Open source | Apache + IBM additions |
| **SSL/TLS** | OpenSSL | **IBM GSKit** (IBM's own SSL engine) |
| **WebSphere Plugin** | ❌ Not included | ✅ Included and supported |
| **IBM Support** | ❌ No | ✅ Full IBM support contract |
| **Certificates** | Standard format | **IBM KDB** (Key Database) format |
| **Enterprise features** | Limited | Full enterprise features |

---

### Why Does DigiBank Care? (Real Banking Reasons)

#### 1️⃣ SSL is Different

- Apache → uses **OpenSSL** commands
- IHS → uses **IBM GSKit** tools
- Certificates stored in **KDB files** (IBM's special format)
- ⚠️ Learn GSKit — OpenSSL knowledge alone is not enough!

#### 2️⃣ The WebSphere Plugin

- Special component that **connects IHS to WAS**
- Apache does NOT have it — IHS does
- Without it, IHS cannot forward requests to WAS

#### 3️⃣ Support Contract (Bank Rule!)

- Everything in a bank MUST have vendor support
- IHS breaks at 2 AM → 📞 Call IBM → they fix it
- Apache breaks at 2 AM → 😅 Community forum, no answer

#### 4️⃣ Audit & Compliance

- Banking regulators accept IBM products easily
- *"It's an IBM supported product"* → audit closed ✅

---

## 📝 EXAM-READY SUMMARY

1. **IHS = IBM's web server**, based on Apache + IBM additions
2. **Purpose:** Front door — receives requests, serves static content, forwards dynamic requests to WAS
3. **Never expose WAS directly** — 5 reasons: traffic, SSL, hardening, static files, security
4. **SSL in IHS uses IBM GSKit**, certificates in **KDB format** — NOT OpenSSL
5. **WebSphere Plugin** connects IHS to WAS
6. **Banks choose IHS** for IBM support + audit compliance

---

## 🧠 Quick Self-Test

1. Why can't customers reach WAS directly?
2. What are the 3 jobs of IHS?
3. Which tool handles SSL in IHS — OpenSSL or GSKit?
4. What file format holds IHS certificates?
5. What connects IHS to WAS?