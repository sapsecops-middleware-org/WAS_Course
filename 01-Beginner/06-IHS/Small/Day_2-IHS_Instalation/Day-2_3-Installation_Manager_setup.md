# PART 2 — IBM Installation Manager (IM)

---

## 1. What is Installation Manager?

**Simple answer:** It is IBM's main installer tool.

Think of it like this:

- Google Play Store installs apps on your phone.
- **Installation Manager installs IBM software on your server.**

You don't run separate installers for each product. You use ONE tool (IM) for everything.

### What IM installs in our DigiBank project:

- IBM HTTP Server (IHS) 9.0.5.x
- WebSphere Application Server ND 9.0.5.x
- Web Server Plugins
- Fix Packs (updates)

---

## 2. Why Do We Need IM? (Real-Life Comparison)

### ❌ Without IM — the painful way:

- Download IHS installer → run it manually
- Download Plugin installer → run it manually
- Download Fix Pack → run it manually
- **Problem:** You must track versions yourself. Easy to make mistakes.

### ✅ With IM — the smart way:

- Open IM once
- Select what you need — IM knows the correct order, correct versions, correct paths
- Fix Packs: IM **remembers what is installed** and applies the right update

**Banking rule:** Mistakes = outages = money lost. IM reduces mistakes. That's why we use it.

---

## 3. Headless Servers — Why Silent Install?

**Question:** What is a headless server?

**Answer:** A server with **no monitor, no mouse, no keyboard**. You only reach it through the network (SSH).

So:

- No GUI (graphical screens) possible
- Everything is done with commands = **"Silent Install"**

In banking production, **100% silent installs**. Remember this.

---

## 4. Installing IM Itself — Step by Step

### Step 1 — Go to the IM software folder

```bash
cd /software/IM/
```

*(Your team usually keeps downloaded IBM software in `/software` or similar)*

### Step 2 — Run the silent install

```bash
./installc -acceptLicense -log /tmp/im_install.log
```

**Break it down:**

| Part | Meaning |
|------|---------|
| `./installc` | The silent (command-line) installer program. "c" = console mode |
| `-acceptLicense` | You accept IBM's license automatically (no GUI to click "I Agree") |
| `-log /tmp/im_install.log` | Writes the install log here — use it if anything fails |

### Step 3 — Verify IM is installed

```bash
/opt/IBM/InstallationManager/eclipse/tools/imcl version
```

**What is `imcl`?**

- IM CommandLine tool
- This is the command you will use to install IHS, WAS, Plugins, Fix Packs later
- If this command shows a version number → **IM is installed successfully** ✅

---

## 5. Default Installation Path (Memorize This)

```text
/opt/IBM/InstallationManager
```

Most IBM products go under `/opt/IBM/...`. If a senior asks **"where is IM installed?"** — now you know.

---

## 6. Key Terms — Quick Glossary

| Term | Plain English |
|------|---------------|
| **IM** | Installation Manager — IBM's universal installer |
| **imcl** | The command-line tool inside IM used to install everything |
| **Silent install** | Installing with commands only, no screens |
| **Fix Pack** | A patch/update for an IBM product |
| **Repository** | The location (folder or URL) where IM finds the install files |
| **-acceptLicense** | Flag that says "I agree to the license" |

---

## 7. Golden Rules (From 25 Years of Experience)

1. **Always check the log** after every install. The log tells you success or failure.
2. **Never install with root GUI tools in production** — silent install only.
3. **Note down versions** — IM helps, but you must still document.
4. **Disk space first** — check space (`df -h`) before installing.
5. **IM is the foundation** — IHS, WAS, Plugins all depend on it. If IM install is bad, everything after is bad.

---

## 8. Quick Recap (Exam-Style)

- ✅ IM = one tool to install all IBM products
- ✅ Production = silent install (headless servers)
- ✅ Install IM with: `./installc -acceptLicense -log /tmp/im_install.log`
- ✅ Verify with: `imcl version`
- ✅ `imcl` = the command you'll use for ALL future installs
- ✅ Always read the log
