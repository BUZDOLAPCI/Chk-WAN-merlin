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

### Connection Details

```
Host: 192.168.1.1
Port: 22
Username: cemocan2334
```

### Installation Method

Installed via SSH using expect script automation. The following commands were executed on the router:

```sh
# Download script from this GitHub repo
curl -o /jffs/scripts/ChkWAN.sh https://raw.githubusercontent.com/BUZDOLAPCI/Chk-WAN-merlin/master/ChkWAN.sh

# Make executable
chmod +x /jffs/scripts/ChkWAN.sh

# Add cron job (runs every 5 minutes)
cru a ChkWAN "*/5 * * * * /jffs/scripts/ChkWAN.sh wan nowait once"

# Create services-start for persistence (if it didn't exist)
test -f /jffs/scripts/services-start || (echo '#!/bin/sh' > /jffs/scripts/services-start && chmod +x /jffs/scripts/services-start)

# Add cron job to services-start for reboot persistence
grep -q ChkWAN /jffs/scripts/services-start || echo 'cru a ChkWAN "*/5 * * * * /jffs/scripts/ChkWAN.sh wan nowait once"' >> /jffs/scripts/services-start
```

### Files Installed

| File | Location | Purpose |
|------|----------|---------|
| ChkWAN.sh | `/jffs/scripts/ChkWAN.sh` | Main monitoring script |
| services-start | `/jffs/scripts/services-start` | Cron persistence across reboots |

### Cron Job Configuration

```
*/5 * * * * /jffs/scripts/ChkWAN.sh wan nowait once
```

**Why these parameters:**

| Parameter | Value | Reason |
|-----------|-------|--------|
| Schedule | `*/5 * * * *` | Check every 5 minutes - frequent enough to catch drops quickly |
| Action | `wan` | Restart WAN service only (faster recovery than full reboot) |
| `nowait` | enabled | Skip 10-second startup delay since cron handles timing |
| `once` | enabled | Exit after single check cycle (required for cron usage) |

**Default behavior with this config:**
- Pings: gateway → DNS servers → 9.9.9.9 → 1.1.1.1
- Tries: 3 ping attempts per host
- Fails: 3 complete cycles before taking action
- Time to action: ~90 seconds of confirmed failure
- Recovery: Restarts WAN interface via `service restart_wan`

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
