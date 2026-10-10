# PART 1 — Pre-Installation Checklist (IHS Installation)

> **Think of it like building a house.**
> Before construction, you check the land, water, electricity.
> Same idea here — before installing IHS, we check the server is ready.
>
> **Miss one check → installation fails → bank incident → angry managers.**

---

## 1. What is IHS? (Quick Intro)

- **IHS = IBM HTTP Server.**
- It's like a **reception desk** of a bank.
- Customers (browser traffic) come to the reception first.
- Reception checks them, then forwards them inside (to WebSphere/WAS).
- It sits at the **front** and handles HTTPS (secure) traffic.

**DigiBank setup:**

```
Customer Browser → IHS (VM1) → WAS (Node01 / Node02)
   (reception)         (back-office staff)
```

---

## 2. OS and Hardware Check

> **Why?** Installing software on a weak server is like loading a truck more than its capacity. It will crash.

### Commands — What Each Does

| Command | What it does |
|---|---|
| `cat /etc/os-release` | Shows **which Linux** you have. IHS only supports specific OS versions. Wrong version = install fails. |
| `free -h` | Shows **RAM**. `-h` = human readable (GB instead of ugly numbers). Min: **2 GB**, Production: **8 GB+** |
| `df -h` | Shows **disk space** per folder/partition. No space = installation stops halfway. |

```bash
cat /etc/os-release
free -h
df -h
```

### Minimum Space Needed

| Location | Space | Why |
|---|---|---|
| `/opt` | 2 GB | Software itself installs here |
| `/tmp` | 1 GB | Installer uses temp space |
| `/var` | 1 GB | System logs |

### DigiBank Real Numbers

- `/opt/IBM/HTTPServer` → **5 GB**
- `/opt/IBM/HTTPServer/logs` → **10 GB** (banking traffic = logs grow FAST)
- `/opt/IBM/WebSphere/Plugins` → **2 GB**

> 🧠 **Memory trick:** Think **"SRT"** → **S**pace, **R**AM, **T**ype of OS. Check all three.

---

## 3. OS User and Group

> **Rule: IHS must NOT run as root.**

### Why This Is a Big Deal

- **root = the boss account with ALL powers.**
- If a hacker breaks into IHS running as root → they own the whole server.
- If running as a normal user → hacker gets limited access. Damage controlled.

### Banking Angle

- RBI / auditors will **flag root-running processes as a security violation**.
- This can **fail your audit**. Auditors love to check this.

### Commands

```bash
# Create group
groupadd ibmhttpd

# Create user
useradd -g ibmhttpd -d /home/ibmhttpd -m -s /bin/bash ibmhttpd

# Verify
id ibmhttpd
```

### `useradd` Flags Explained

| Flag | Meaning |
|---|---|
| `-g ibmhttpd` | Put user in that group |
| `-d /home/ibmhttpd` | Home folder location |
| `-m` | Actually create the home folder |
| `-s /bin/bash` | Shell the user can use |

### ⭐ Interview Favorite — Parent vs Child Process

- IHS **parent process runs as root** — only to grab port 443 (low ports need root).
- **Child processes run as ibmhttpd** — these do the actual work.
- So it's a **hybrid**. Auditors accept this because the working processes are non-root.

> 🧠 **Memory trick:** "Root opens the door, ibmhttpd does the job."

---

## 4. Port Check (Ports Must Be FREE)

### What is a Port?

- A **port = a door number** on the server.
- Traffic arrives at a specific door:
  - **80** → HTTP (normal web)
  - **443** → HTTPS (secure web) ← banking uses this

### The Rule

- **Only ONE program can use a door at a time.**
- If another program already occupies 443 → IHS install/config will fail.

### Commands

```bash
ss -tlnp | grep :80
ss -tlnp | grep :443
```

### Flags Explained

| Flag | Meaning |
|---|---|
| `ss` | Shows sockets (network connections) |
| `-t` | TCP only |
| `-l` | Listening doors |
| `-n` | Show numbers, not names |
| `-p` | Show which process is using it |
| `\| grep :80` | Filter — only show line with :80 |

### Expected Result

- ✅ **Nothing prints = port is FREE.** That's what we want.
- ❌ **Something prints = port is BUSY.** Find and stop that process first.

> 🧠 **Memory trick:** "Empty output = happy output."

---

## 5. Hostname and DNS Check

### Why?

- Other machines must **find VM1 by name**, not just IP.
- Example: config uses names like `ihs01.digibank.internal`.
- If the name doesn't resolve → **plugin fails** → customers can't reach the bank.

### Commands

```bash
# Show machine's own name
hostname

# Ask DNS for the IP of this name
nslookup ihs01.digibank.internal

# Local "phone book" file (use if DNS is not configured)
cat /etc/hosts
```

- `hostname` → should show `ihs01` (or similar).
- `nslookup` → DNS answers the IP → good. ✅ If no DNS → use `/etc/hosts`.
- Add a line in `/etc/hosts` like:

```
10.10.10.5   ihs01.digibank.internal   ihs01
```

> 🧠 **Memory trick:** DNS = public phone book. `/etc/hosts` = your personal phone book.

---

## 6. Firewall Rules

### What is a Firewall?

- A **security guard** at the server's gate.
- It decides who can come in and through which door (port).

### DigiBank Rules (Memorize This Table)

| From | To | Port | Purpose |
|---|---|---|---|
| Internet / Load Balancer | IHS VM1 | 443 | Customer HTTPS traffic |
| Internet / LB | IHS VM1 | 80 | HTTP → redirect to 443 |
| IHS VM1 | WAS Node01 | 9080 | Plugin forwards to WAS |
| IHS VM1 | WAS Node02 | 9080 | Plugin forwards to WAS |
| Admin team | IHS VM1 | 22 | SSH (your admin login) |

### Read It Like This

- Traffic must be allowed **BOTH ways**:
  - **IN to IHS:** 443, 80 (customers coming in)
  - **OUT from IHS:** 9080 to WAS nodes (IHS passing work inside)
  - **22 in:** so YOU can SSH and manage it (forget this = you lock yourself out!)

### Commands

```bash
firewall-cmd --list-ports
# or
iptables -L -n | grep 443
```

- `firewall-cmd --list-ports` → shows ports the firewall (firewalld) allows.
- `iptables -L -n | grep 443` → older firewall tool; lists rules, grep finds 443.

### ⚠️ Real-Life Pain Point

- In banks, **you don't open firewall ports yourself** — the **security team** does it.
- You must **raise a request** with the exact table above.
- Port not open + you didn't check = hours wasted debugging. **Always verify first.**

---

## 📝 Final Summary — Pre-Install Checklist

| # | Check | Key Point |
|---|---|---|
| 1 | **SRT check** | Space, RAM, Type of OS ✅ |
| 2 | **Non-root user** | Create `ibmhttpd`, run as non-root (audit requirement) ✅ |
| 3 | **Ports free** | 80 and 443 empty — empty output = good ✅ |
| 4 | **Name resolves** | Hostname + DNS/hosts working ✅ |
| 5 | **Firewall open** | 443/80 in, 9080 out to WAS, 22 for you ✅ |

> 💡 **Golden rule:** *10 minutes of checking saves 10 hours of troubleshooting.*
