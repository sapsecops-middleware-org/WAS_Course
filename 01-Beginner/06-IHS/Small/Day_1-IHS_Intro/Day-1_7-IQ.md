# Part 12 — Quiz: IHS Fundamentals (Q&A)

> **How to use this:** Cover the answer. Read the question. Answer out loud in your own words. Then compare with the model answer.

1: What is IBM HTTP Server and why does DigiBank use it instead of exposing WebSphere directly to the internet?

### Model Answer

IBM HTTP Server (IHS) is a web server based on Apache HTTP Server, enhanced by IBM with:

- **GSKit** — for SSL certificate and encryption management.
- **WebSphere Plugin** — for seamless integration with WebSphere Application Server (WAS).

In DigiBank, IHS sits in the **DMZ** as the first point of contact for customer requests.

**We never expose WebSphere directly to the internet because:**

- WAS is not designed to handle raw internet traffic.
- It lacks the security hardening required at the perimeter.
- It would create a direct attack surface against our application server.

**IHS responsibilities:**

- SSL termination (handles encryption/decryption).
- Serves static content.
- Load balancing between WAS cluster members.
- Acts as the secure entry point.
- The WebSphere Plugin inside IHS routes dynamic requests to the correct WAS cluster members.

### Simple Explanation 💡

- Think of IHS as the **reception desk** of the bank building.
- Customers (internet traffic) always talk to the reception desk first.
- You never let strangers walk straight into the vault room (WebSphere).
- The reception desk checks the visitor, handles the secure door (SSL), and passes the request to the right office (WAS cluster member).

**Memory hook:** *IHS = front door guard. WAS = back office. Never mix them.*

---

## Q2: Explain the complete request flow when a DigiBank customer logs in.

### Model Answer

When a customer opens `https://www.digibank.com/internetbanking`:

1. **DNS lookup** — browser resolves the domain to the load balancer VIP.
2. **Load balancer** — forwards the request to the active IHS server on port 443.
3. ** presents the DigiBank SSL certificate and decrypts the request.
4. **VirtualHost match** — IHS reads the Host header and matches it to the correct VirtualHost block in `httpd.conf`.
5. **Plugin routing** — the WebSphere Plugin reads `plugin-cfg.xml`, sees that `/internetbanking/` maps to the DigiBank cluster, applies the load balancing algorithm, and selects Server01 or Server02.
6. **Internal forward** — request is forwarded to WAS on port 9080.
7. **WAS processing** — request reaches `internetbanking.war`, runs the Java business logic, queries the database, and builds the response.
8. **Return path** — response travels back through WAS → Plugin → IHS (which re-encrypts it) → customer's browser.

### Simple Explanation 💡

Follow the letter, step by step:

1. You write the address (DNS finds the building).
2. Guard at the gate lets you in (load balancer).
3. Reception checks your ID badge (TLS handshake with certificate).
4. Reception reads your form and picks the right office (VirtualHost + plugin-cfg.xml).
5. Reception sends you to Office 1 or Office 2 (Server01 or Server02, port 9080).
6. The office does the work and answers (WAR file + database).
7. The answer goes back the same way, sealed in an envelope again (re-encrypted by IHS).

**Memory hook:** *DNS → LB → IHS (SSL) → Plugin (plugin-cfg.xml) → WAS :9080 → WAR → DB → back again.*

---

## Q3: What is the difference between the parent process and child processes in IHS?

### Model Answer

IHS uses a **parent-child process model** for stability and security.

**Parent process:**

- Starts first, reads `httpd.conf`, and spawns child processes.
- Does **NOT** handle customer requests.
- Runs as **root** because it must bind to privileged ports 80 and 443.

**Child processes:**

- Actually handle customer requests.
- Each child runs as a **low-privilege user** (typically `ibmhttpd`).
- Each child contains multiple **worker threads** that process requests concurrently.

**Why this design:**

- If a child process crashes, the parent immediately detects it and spawns a replacement.
- A single crashed connection or process does not take down the entire IHS server.
- In DigiBank production, we tune the number of child processes and threads based on expected concurrent user load for peak banking hours.

### Simple Explanation 💡

- Parent = the **manager**. Does the paperwork (reads config), hires staff (spawns children), replaces staff who quit (restarts crashed children).
- Children = the **workers**. They actually serve the customers.
- Workers wear restricted badges (low-privilege user `ibmhttpd`) — even if one is compromised, damage is limited.
- The manager holds the keys to doors 80 and 443 (root) — only the manager can open those doors.

**Memory hook:** *Parent = root + no customers. Children = no root + all customers. Crash a child → parent replaces it instantly.*

---

## Quick Recall Card 🔁

| Question | One-Line Answer |
|----------|-----------------|
| What is IHS? | Apache-based web server + GSKit (SSL) + WebSphere Plugin — the secure front door for WAS. |
| Why not expose WAS directly? | Not hardened for internet, direct attack surface, IHS handles SSL/static/load balancing. |
| Request flow? | DNS → LB → IHS:443 (TLS) → VirtualHost → Plugin (plugin-cfg.xml) → WAS:9080 → WAR → DB → back. |
| Parent process? | Root, reads httpd.conf, spawns children, never touches customer requests. |
| Child processes? | Low-privilege (ibmhttpd), worker threads, serve customers, auto-replaced if they crash. |
