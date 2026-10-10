# WebSphere Application Server — SystemOut.log Rotation & Disk-Full Prevention Guide

## Overview

Unbounded growth of JVM logs (`SystemOut.log`, `SystemErr.log`) on WebSphere Application Server (WAS) can fill the disk partition hosting the profile, causing log write failures, transaction rollbacks, and application outages.

This document covers:
- Root cause of log-induced disk-full incidents
- Immediate remediation steps
- Permanent fix via log rotation configuration
- Automation and monitoring recommendations

---

## Incident Summary (Real-World Case)

| Item | Detail |
|------|--------|
| Environment | HDFC Bank — Payment Gateway server |
| Component | `SystemOut.log` on WAS profile |
| Issue | No log rotation / size limit configured |
| Duration of growth | ~8 months |
| Final log size | **48 GB** |
| Impact | `/opt` partition hit 100% → UPI & IMPS payment failures |

### Failure Chain

```
Log grows unbounded
      ↓
/opt partition reaches 100% capacity
      ↓
WAS cannot write to log files
      ↓
Log writing failure → transaction rollbacks
      ↓
UPI / IMPS payments fail
```

> [!NOTE]
> WAS treats JVM log write failures as critical. Transaction infrastructure may roll back in-flight work when the server cannot persist logging/tracing output, directly impacting payment processing.

---

## Detection & Diagnosis

### 1. Check disk usage

```bash
df -h
```

Example output:

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda3        50G   50G     0  100% /opt
```

### 2. Locate the oversized log

```bash
du -sh /opt/IBM/WebSphere/AppServer/profiles/*/logs/*/SystemOut.log
```

Example output:

```
48G  /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/server1/SystemOut.log
```

### 3. Check for other space hogs

```bash
du -sh /opt/IBM/WebSphere/AppServer/profiles/*/logs/* | sort -rh | head -20
```

---

## Immediate Fix (Incident Response)

> [!WARNING]
> Do **not** delete the log file if it is required for audit or compliance. Rename (archive) it instead.

```bash
cd /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs/server1

# Rename the huge log — preserve for audit
mv SystemOut.log SystemOut.log.archived_20241018

# Create an empty new log so WAS can resume writing
touch SystemOut.log
```

### Then restart the server

```bash
/opt/IBM/WebSphere/AppServer/profiles/AppSrv01/bin/stopServer.sh server1
/opt/IBM/WebSphere/AppServer/profiles/AppSrv01/bin/startServer.sh server1
```

> [!TIP]
> After restart, verify WAS is writing to the fresh log:
> ```bash
> tail -f SystemOut.log
> ```

### Free space from archived log (optional, post-audit)

```bash
# Compress instead of deleting outright
gzip SystemOut.log.archived_20241018
```

---

## Permanent Fix — Configure Log Rotation

Configure rotation via the **Admin Console**:

```
Troubleshooting → Logs and Trace → server1 → JVM Logs
```

### Recommended Settings

| Setting | Value | Purpose |
|---------|-------|---------|
| Maximum Size | `50 MB` | Limit each log file to 50 MB |
| Maximum Number of Files | `10` | Retain last 10 rotated files (500 MB max total) |
| Log Rollover | `By Size` | Auto-rotate when the size limit is hit |

### Equivalent `server.xml` Configuration

The rotation settings appear in the profile's `server.xml` under the `<logging>` element:

```xml
<logging
    enableRollover="true"
    rolloverSize="50"
    maxFileSize="50"
    maxNumOfFiles="10"
    rolloverType="SIZE"/>
```

> [!NOTE]
> `rolloverSize` / `maxFileSize` values are in **MB**. A maximum of 10 files × 50 MB caps total log usage at ~500 MB per log type.

### Repeat for all log types

Configure rotation for **each** of the following:

- `SystemOut.log`
- `SystemErr.log`
- Trace logs (if tracing is enabled)
- Activity logs / HTTP access logs

---

## Alternative — Log Rotation via `logrotate` (Linux)

If console access is limited, or you need OS-level control:

```bash
# /etc/logrotate.d/was-logs
/opt/IBM/WebSphere/AppServer/profiles/*/logs/*/SystemOut.log
/opt/IBM/WebSphere/AppServer/profiles/*/logs/*/SystemErr.log {
    daily
    rotate 10
    size 50M
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
}
```

> [!WARNING]
> `copytruncate` briefly copies then truncates the file — a small window of log lines may be lost. Prefer WAS-native rotation when possible.

---

## Monitoring & Prevention

### Disk Usage Alerts

Monitor the hosting partition and alert before it reaches capacity:

```bash
#!/bin/bash
# disk_alert.sh — alert when /opt usage exceeds 80%
THRESHOLD=80
USAGE=(df -h /opt | awk 'NR==2 {print int(5)}')
if [ "USAGE"−ge"USAGE" -ge "USAGE"−ge"THRESHOLD" ]; then
    echo "ALERT: /opt at USAGE{USAGE}% —USAGE(date)" | mail -s "WAS Disk Alert" ops-team@example.com
fi
```

Schedule via cron:

```bash
*/15 * * * * /opt/scripts/disk_alert.sh
```

### Log Size Monitoring

```bash
find /opt/IBM/WebSphere/AppServer/profiles/*/logs -name "*.log" -size +100M \
    -exec ls -lh {} \;
```

---

## Best Practices Checklist

- [ ] Enable rollover (`By Size`) on all JVM logs across all profiles
- [ ] Set `Maximum Size` = 50 MB, `Maximum Number of Files` = 10
- [ ] Apply the same settings to `SystemErr.log` and trace logs
- [ ] Add disk usage monitoring with alerts at 80% threshold
- [ ] Schedule periodic log archive cleanup (compress and move off-node)
- [ ] Document the archive/audit retention policy before deleting old logs
- [ ] Verify rotation behavior after patching or profile migration — settings can reset
- [ ] Review verbose logging flags; excessive `Trace`/`Debug` output accelerates growth

---

## Key Takeaways

1. **Never leave JVM logs unbounded** — a single 48 GB log can take down a payment platform.
2. **Archive, don't delete** during incidents — audit and compliance requirements usually apply.
3. **Configure rotation proactively** — 50 MB × 10 files caps log storage at ~500 MB.
4. **Monitor disk usage** — an alert at 80% would have prevented this outage weeks in advance.
5. **Log failures cascade** — in WAS, a full disk doesn't just stop logging; it can roll back transactions and break payment flows.
