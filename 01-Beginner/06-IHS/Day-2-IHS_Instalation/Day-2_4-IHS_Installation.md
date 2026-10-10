# PART 3 — IHS Installation (Silent Install) — Explained Simply

---

## First — What is a Response of it like this:**

- Installing software by clicking buttons = **GUI install**
- Installing software by giving it a **written instruction sheetent install**

That instruction sheet is called a **response file** (an XML file).

**Why banks love silent installs:**

- No human clicking = no mistakes
- Same file used on 50 servers = identical setup everywhere
- Can be run by automation (scripts, Ansible)
- Works on servers with **no screen/desktop**

**Real-life example:** A bank has 80 IHS servers. Doing GUI installs 80 times = slow and human errors. One response file + one command each = done in a day.

---

## Step 1 — Create the Response File

```bash
vi /tmp/ihs_install_response.xml
```

Now let's read the file line by line so you understand every part:

```xml
<?xml version="1.0" encoding="UTF-8"?>
```

- Standard XML header. Every XML file starts like this. Just copy it.

```xml
<agent-input>
```

- This opens the "instruction sheet" for Installation Manager.
- Everything goes inside this tag.

```xml
<server>
  <repositoryHS9055/"/>
</server>
```

- **Repository = the folder where the installation files live.**
- Installation Manager will look here for IHS software.
- Like telling a chef: "ingredients are

```xml
<install modify="false">
  <offering id="com.ibm.websphere.IHS.v90"
            version="9.0.5.5"
            profile="IBM HTTP Server"
            features="core.feature"
            installFixes="none"/>
</install>
```

- **What** to install:
  - `offering id` = the product name (IHS v9.0)
  - `version` = exact fix level (9.0.5.5)
  - `features` = `core.feature` means the basic IHS parts
  - `installFixes="none"` = don't add extra fixes now (version need)

```xml
<profile id="IBM HTTP Server"
         installLocation="/opt/IBM/HTTPServer">
```

- **Where** to install: `/opt/IBM/HTTPServer`
- Standard location your own path — support teams know this path.

```xml
<data key="user.ihs.httpPort" value="80"/>
<data key="user.ihs.httpsPort" value="443"/>
```

- The ports IHS will listen on.
- **80** = normal web traffic. **443** = secure (HTTPS) traffic.
- These are the standard web ports. Browsers use them automatically.

```xml
<data key="user.ihs.serverUserId" value="ibmhttpd"/>
<data key="user.ihs.serverGroupId" value="ibmhttpd"/>
```

- **VERY IMPORTANT for banking.**
- IHS will **run as a normal user** (`ibmhttpd`), NOT as root.
- Why? **Security.** If IHS is hacked, the attacker only gets `ibmhttpd` rights — not full root control of the server.
- Banks audit this. Running web servers as root = audit finding.

> ⚠️ **Tip:** The user `ibmhttpd` must already exist (created earlier with ` it doesn't exist, the install can fail.

---

## Step 2 — Run the Silent Install

```bash
/opt/IBM/InstallationManager/eclipse/tools/imcl \
  input /tmp/ihs_install_response.xml \
  -log /tmp/ihs_install_log.xml \
  -acceptLicense
```

**Breaking it down:**

| Piece | Meaning |
|---|---|
| `imcl` | Installation Manager Command Line — the silent install tool |
| `input <file>` | "Here is my instruction sheet" |
| `-log <file>` | Write all activity to this log file |
| `-acceptLicense` | "I accept the IBM license" (you must include this in silent mode — there's no button to click) |

- The `\` at line end just means "command continues on next line." Easy to read.
- Wait a few minutes. Silent install shows little output — that's normal. **Don't panic.**

---

## Step 3 — Verify the Install

```bash
echo $?
```

- Checks the **exit code** of the last command.
- **0 = success.** Anything else = something failed.
- This is the FIRST thing every admin checks.

```bash
grep -i error /tmp/ihs_install_log.xml
```

- Searches the log for the word "error" (`-i` = ignore capital/small letters).
- No output = no errors found. Good sign.

```bash
ls -la /opt/IBM/HTTPServer/bin/httpd
ls -la /opt/IBM/HTTPServer/bin/apachectl
```

- Confirms the actual IHS programs exist:
  - `httpd` = the web server engine itself.
  - `apachectl` = the control tool (start/stop/restart IHS).
- If both files exist with details shown — install is real, not just "claimed."

---

## GUI Installation (When You Have a Desktop)

Only use this in **labs or learning**. Production = silent install.

1. `cd /software/IM/`
2. `./install` → GUI opens (window pops up)
3. **File → Preferences → Repositories** → add `/software/IHS9055/`
   - This tells the GUI where the software is (same as `<server>` in the XML)
4. Click **Install**
5. Select **IBM HTTP Server 9.0.5.x**
6. Accept license (in silent mode we typed `-acceptLicense`; here you click it)
7. Install location: `/opt/IBM/HTTPServer`
8. Ports: 80 and 443
9. Run-as user: `ibmhttpd`
10. Click **Install**, wait for finish

**Notice:** GUI and response file do the *exact same things*. GUI = clicking. Response file = typing. Same result.

---

## Quick Memory Card 📌

- **Response file** = written instructions for Installation Manager
- **Repository** = folder holding install files
- **imcl** = the command that runs silent installs
- **-acceptLicense** = mandatory in silent mode
- **echo $?** → 0 = success
- **Run as ibmhttpd, never root** = banking security rule
- Ports)** and **443 (HTTPS)**
- Verify: `grep` has no errors + `httpd` and `apachectl` exist

---

## Common Beginner Mistakes

- ❌ Forgetting `-acceptLicense` → install fails with license error
- ❌ `ibmhttpd` user not created before install → install fails
- ❌ Wrong repository path in XML → "no offerings found" error
- ❌ Panicking during install (no output = normal)
- ❌ Not checking `echo $?` and assuming success

---

