# ChkWAN-merlin Setup Log

## Overview

This is a fork of [MartineauUK/Chk-WAN](https://github.com/MartineauUK/Chk-WAN) with an added uptime safety check to prevent the script from running during router boot.

**Setup Date:** 2026-01-31
**Router:** ASUS RT-AX88U
**Firmware:** ASUSWRT-Merlin 3004.388.11_0

---

## Modification Made

Added uptime check at the beginning of `ChkWAN.sh` (after shebang):

```sh
# Uptime Check: skip if router has been up for less than 5 mins (300 seconds)
if [ $(cat /proc/uptime | awk '{print $1}' | cut -d. -f1) -lt 300 ]; then
    logger -t ChkWAN "Router uptime < 5 mins. Exiting."
    exit 0
fi
```

**Purpose:** Prevents the script from triggering WAN restarts during boot when network interfaces may not be fully initialized.

---

## Router Installation

### Files Installed

| File | Location | Purpose |
|------|----------|---------|
| ChkWAN.sh | `/jffs/scripts/ChkWAN.sh` | Main monitoring script |
| services-start | `/jffs/scripts/services-start` | Cron persistence across reboots |

### Cron Job

```
*/5 * * * * /jffs/scripts/ChkWAN.sh wan nowait once
```

- Runs every 5 minutes
- `wan` = Restart WAN service on failure (not full reboot)
- `nowait` = Skip 10-second startup delay
- `once` = Exit after single check (for cron usage)

### services-start Content

```sh
#!/bin/sh
cru a ChkWAN "*/5 * * * * /jffs/scripts/ChkWAN.sh wan nowait once"
```

---

## How to Undo / Uninstall

SSH into the router and run:

```sh
# Remove the cron job
cru d ChkWAN

# Remove the script
rm /jffs/scripts/ChkWAN.sh

# Remove ChkWAN line from services-start (or delete file if nothing else in it)
sed -i '/ChkWAN/d' /jffs/scripts/services-start

# Verify removal
cru l | grep ChkWAN
ls /jffs/scripts/ChkWAN.sh
```

---

## Useful Commands

```sh
# Check if monitor is running
/jffs/scripts/ChkWAN.sh status

# View current cron jobs
cru l

# Manually test the script (verbose)
/jffs/scripts/ChkWAN.sh wan nowait once

# View logs
grep ChkWAN /tmp/syslog.log

# Stop a running monitor instance
/jffs/scripts/ChkWAN.sh stop
```

---

## Troubleshooting

### Script runs but doesn't restart WAN
- Check logs: `grep ChkWAN /tmp/syslog.log`
- Verify ping targets are reachable when internet is up
- Try running manually: `/jffs/scripts/ChkWAN.sh wan nowait once`

### Cron job not running after reboot
- Verify services-start exists: `cat /jffs/scripts/services-start`
- Verify it's executable: `ls -la /jffs/scripts/services-start`
- Check cron: `cru l`

### Want to change check frequency
Edit the cron schedule:
```sh
# Every 2 minutes instead of 5
cru d ChkWAN
cru a ChkWAN "*/2 * * * * /jffs/scripts/ChkWAN.sh wan nowait once"

# Update services-start for persistence
sed -i 's/\*\/5/\*\/2/' /jffs/scripts/services-start
```

### Want full reboot instead of WAN restart
Change `wan` to `reboot`:
```sh
cru d ChkWAN
cru a ChkWAN "*/5 * * * * /jffs/scripts/ChkWAN.sh reboot nowait once"

# Update services-start
sed -i 's/wan nowait/reboot nowait/' /jffs/scripts/services-start
```

---

## Original Script Documentation

See the original repository for full usage documentation:
https://github.com/MartineauUK/Chk-WAN

Key options:
- `reboot` - Full router reboot on failure
- `wan` - Restart WAN interface only
- `noaction` - Check only, no recovery action
- `force` - Include 10MB download test
- `forcesmall` - Include 500B download test
- `tries=N` - Number of ping attempts per host
- `fails=N` - Number of failed cycles before action
- `ping=x.x.x.x,y.y.y.y` - Custom ping targets
