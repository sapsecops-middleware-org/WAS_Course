# IBM WebSphere Application Server — Log Viewing & Management Guide

---

## 1. Viewing Logs via the Admin Console

You can view logs directly from the browser **without SSH access**.

### Navigation Path

```
Left menu:
Troubleshooting
  └── Logs and Trace
        └── server1             ← Click the server name
              └── JVM Logs      ← Click this
```

### Available Logs

| Log            | Purpose                                            |
| -------------- | -------------------------------------------------- |
| `SystemOut.log`| Standard output (application + WAS messages)       |
| `SystemErr.log`| Standard error output (stack traces, exceptions)   |

### Available Actions

| Action           | Description                                        |
| ---------------- | -------------------------------------------------- |
| **View log**     | Opens the log in the browser (last N lines only)   |
| **Download log** | Downloads the full log file                        |
| **Configure**    | Change log settings (rollover size, max files, etc.) |

> [!WARNING]
> The browser view only shows **recent lines**. For full log analysis — always use SSH and command-line tools.

---

## 2. Viewing Logs via SSH (Command Line)

Log files are located under the profile's logs directory, for example:

```
<WAS_HOME>/profiles/AppSrv01/logs/server1/SystemOut.log
<WAS_HOME>/profiles/AppSrv01/logs/server1/SystemErr.log
<WAS_HOME>/profiles/AppSrv01/logs/nodeagent/SystemOut.log
<WAS_HOME>/profiles/AppSrv01/logs/ffdc/
```

> [!TIP]
> FFDC (`First Failure Data Capture`) files are critical after a crash — always check `logs/ffdc/` when diagnosing unexpected failures.

---

## 3. Working with Logs via wsadmin (Jython)

Start `wsadmin` in Jython mode before running any of these scripts:

```bash
./wsadmin.sh -lang jython -username wasadmin -password <password>
```

### 3.1 View the Log File Path Configured for a Server

```python
# Connect to wsadmin first

# Get server config object
server = AdminConfig.getid(
    '/Cell:BankCell01/Node:BankNode01/Server:server1/')

# Get JVM log config
jvmLog = AdminConfig.list('OutputStreamRedirect', server)
print jvmLog
```

### 3.2 Change / List Log File Location via wsadmin

```python
# Get the server
server = AdminConfig.getid(
    '/Cell:BankCell01/Node:BankNode01/Server:server1/')

# Get all OutputStreamRedirect objects (SystemOut and SystemErr)
logList = AdminConfig.list('OutputStreamRedirect', server).splitlines()

for log in logList:
    name = AdminConfig.showAttribute(log, 'fileName')
    print "Current log path: " + str(name)
```

> [!TIP]
> To modify the path, use `AdminConfig.modify()` on the `fileName` attribute of each `OutputStreamRedirect` object, then save the configuration.

### 3.3 Get Server State via wsadmin (Monitoring Scripts)

```python
# Check if server1 is running
try:
    serverMBean = AdminControl.queryNames(
        'cell=BankCell01,node=BankNode01,name=server1,type=Server,*')

    if serverMBean:
        state = AdminControl.getAttribute(serverMBean, 'state')
        print "server1 state: " + state
    else:
        print "server1 is NOT running (MBean not found)"
except:
    print "Cannot connect to server1"
```

---

## 4. Useful Log Commands — Quick Reference

```bash
# ── WATCH LIVE ──
tail -f profiles/AppSrv01/logs/server1/SystemOut.log

# ── SEARCH FOR ERRORS ──
grep " E " profiles/AppSrv01/logs/server1/SystemOut.log | tail -20

# ── SEARCH FOR SPECIFIC ERROR ──
grep -i "OutOfMemory\|heap\|crash" \
  profiles/AppSrv01/logs/server1/SystemOut.log

# ── CHECK NODE AGENT SYNC ──
grep -i "sync\|synchroni" \
  profiles/AppSrv01/logs/nodeagent/SystemOut.log | tail -10

# ── FIND WHEN SERVER STARTED ──
grep "ADMU3000I" profiles/AppSrv01/logs/server1/SystemOut.log

# ── FIND WHEN SERVER STOPPED ──
grep "ADMU0211I\|ADMU4000I" \
  profiles/AppSrv01/logs/server1/SystemOut.log

# ── CHECK FFDC FOR NEW INCIDENTS ──
ls -lth profiles/AppSrv01/logs/ffdc/ | head -10

# ── LOG SIZE CHECK (important — large logs slow down the server) ──
du -sh profiles/AppSrv01/logs/server1/SystemOut.log
```

### Key WAS Message ID Cheat Sheet

| Message ID   | Meaning                          |
| ------------ | -------------------------------- |
| `ADMU3000I`  | Server started successfully      |
| `ADMU0211I`  | Server stop initiated            |
| `ADMU4000I`  | Server stop completed            |

> [!TIP]
> WAS message severity letters inside log lines: ` I ` = Info, ` W ` = Warning, ` E ` = Error, ` F ` = Fatal. Use `grep " E "` for fast error triage.

---

## 5. Best Practices

- **Prefer SSH over the browser view** for anything beyond a quick glance — grep/tail are far more powerful.
- **Enable log rollover** (size-based and time-based) in the Admin Console: *Troubleshooting → Logs and Trace → server1 → JVM Logs → Configure*.
- **Monitor log size** — very large `SystemOut.log` files can slow down server I/O and make triage painful.
- **Check FFDC after crashes** — `logs/ffdc/` contains structured incident data you should attach to PMRs.
- **Automate health checks** with the wsadmin server-state script above, scheduled via `cron` or a monitoring agent.
- **Never edit log config by hand** in XML files — always use the Admin Console or `wsadmin` to keep the master repository in sync.
