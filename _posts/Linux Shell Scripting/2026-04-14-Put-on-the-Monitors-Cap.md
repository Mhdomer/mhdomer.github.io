---
layout: post
title: Chapter 8 Put on the Monitor's Cap
date: 2026-04-14T10:00:00
categories:
  - Linux Shell Scripting
tags:
  - linuxcli
  - linux
  - bash
  - scripting
  - monitoring
  - sysadmin
author: muhammed
description: Chapter 8 of Linux Shell Scripting Cookbook — disk usage, process monitoring, logging, power measurement, and filesystem health from the shell
toc: true
pin: false
math: false
mermaid: false
image: https://media.licdn.com/dms/image/v2/D4D12AQG85GEyaMcdzg/article-cover_image-shrink_720_1280/article-cover_image-shrink_720_1280/0/1715198365363?e=2147483647&v=beta&t=zSE3QFqa_pLtDmQbUv_4HvOAvufhMvrOVzRy6lWimHE
Link: "[[Shell Scripting Notes]]"
---

# Chapter Overview

This chapter is about keeping an eye on your system — disk usage, running processes, login activity, power consumption, and filesystem health. The tools here are what sysadmins and CTF players both reach for when they need situational awareness.

---

## Monitoring Disk Usage

### du — disk usage of files and directories

```bash
du -sh /var/log            # human-readable total for a directory
du -sh *                   # size of each item in current directory
du -ah /home/user          # all files recursively, human-readable
du -h --max-depth=1 /      # one level deep from root
```

**Find the 10 largest directories:**

```bash
du -h /var | sort -rh | head -10
```

`sort -rh` — reverse, human-readable sort (handles K/M/G correctly).

### df — disk free (filesystem level)

```bash
df -h                      # all mounted filesystems, human-readable
df -hT                     # include filesystem type (ext4, tmpfs, etc.)
df -i                      # inode usage instead of block usage
df -h /home                # only the filesystem containing /home
```

**Watch for inode exhaustion** — a partition can be 0% block-full but 100% inode-full and still reject new files.

### Finding large files

```bash
find / -type f -size +100M 2>/dev/null          # files over 100 MB
find /var/log -name "*.log" -size +50M           # large log files
find / -type f -printf '%s %p\n' | sort -rn | head -10   # top 10 by bytes
```

---

## Calculating Execution Time

### time — measure command duration

```bash
time sleep 2
time find / -name "*.conf" 2>/dev/null
```

Output:
```
real    0m2.004s     # wall-clock time (what you actually wait)
user    0m0.001s     # CPU time in user space
sys     0m0.003s     # CPU time in kernel space
```

`real > user + sys` means the command was waiting (I/O, sleep, network).

### Manual timing with date

```bash
start=$(date +%s%N)          # nanoseconds since epoch
some_command
end=$(date +%s%N)
echo "Elapsed: $(( (end - start) / 1000000 )) ms"
```

Useful when you want to embed timing inside a script.

---

## Logged-in Users, Boot Logs, and Boot Failures

### who and w

```bash
who                    # currently logged-in users
who -b                 # last system boot time
w                      # logged-in users + what they're running
```

### last — login history

```bash
last                   # full login history (reads /var/log/wtmp)
last reboot            # all reboot events
last -n 10             # last 10 logins
last username          # logins for a specific user
last -F                # full timestamps
```

### lastb — failed login attempts

```bash
lastb                  # failed logins (reads /var/log/btmp)
lastb -n 20            # last 20 failures
```

`lastb` requires root — it reads `/var/log/btmp`.

### journalctl — systemd boot logs

```bash
journalctl -b                  # logs from current boot
journalctl -b -1               # logs from previous boot
journalctl -b --list-boots     # list all recorded boots
journalctl -p err -b           # only errors from current boot
journalctl -u ssh              # logs for the SSH service
journalctl --since "1 hour ago"
journalctl --since "2026-04-14 08:00" --until "2026-04-14 09:00"
```

### dmesg — kernel ring buffer

```bash
dmesg                        # all kernel messages since boot
dmesg | tail -20             # latest kernel messages
dmesg -T                     # human-readable timestamps
dmesg --level=err,warn       # only errors and warnings
dmesg | grep -i "fail\|error\|warn"
```

---

## Top 10 CPU-Consuming Processes in an Hour

The idea: sample `ps` repeatedly, accumulate CPU time per PID, sort at the end.

```bash
#!/bin/bash
declare -A cpu_map

for i in $(seq 1 60); do
    while IFS= read -r line; do
        pid=$(echo "$line" | awk '{print $1}')
        cpu=$(echo "$line" | awk '{print $2}')
        name=$(echo "$line" | awk '{print $3}')
        cpu_map[$pid]+=$(echo "$cpu" | awk '{printf "%.2f", $1}')
        # store name for last seen PID
        name_map[$pid]=$name
    done < <(ps -eo pid,%cpu,comm --no-headers --sort=-%cpu | head -20)
    sleep 60
done

# Print top 10 by accumulated CPU
for pid in "${!cpu_map[@]}"; do
    echo "${cpu_map[$pid]} $pid ${name_map[$pid]}"
done | sort -rn | head -10
```

Simpler one-liner snapshot (not accumulated — just a point-in-time top 10):

```bash
ps -eo pid,%cpu,%mem,comm --no-headers --sort=-%cpu | head -10
```

---

## Monitoring Command Outputs with watch

`watch` re-runs a command at a fixed interval and refreshes the terminal.

```bash
watch -n 2 df -h              # refresh disk usage every 2 seconds
watch -n 1 'ps -eo pid,%cpu,comm --sort=-%cpu | head -10'
watch -n 5 'ss -tnp | grep ESTAB'   # established TCP connections
watch -d free -h              # highlight differences between runs (-d)
watch -n 1 date               # basic clock in the terminal
```

`-d` / `--differences` — highlight what changed since the last refresh.

---

## Logging Access to Files and Directories

### inotifywait — filesystem event monitoring

```bash
inotifywait -m /etc/passwd            # monitor a single file
inotifywait -m -r /home/user/         # recursive directory watch
inotifywait -m -e modify,create,delete /var/www/html
```

**Log file access to a file:**

```bash
inotifywait -m -r --format '%T %w %f %e' --timefmt '%F %T' \
    /sensitive/dir >> /var/log/access.log &
```

| Event flag | Meaning |
|---|---|
| `ACCESS` | file was read |
| `MODIFY` | file was written |
| `CREATE` | file/dir created |
| `DELETE` | file/dir deleted |
| `ATTRIB` | permissions/ownership changed |
| `MOVED_FROM/TO` | rename or move |

### auditd — kernel-level audit

```bash
auditctl -w /etc/sudoers -p rwxa -k sudoers_watch    # watch sudoers
auditctl -w /home/user/secret.txt -p rw              # watch a file
auditctl -l                                           # list rules
ausearch -k sudoers_watch                             # search by key
aureport --summary                                    # audit summary
```

`auditd` survives reboots when rules are saved to `/etc/audit/rules.d/`.

---

## Logfile Management with logrotate

`logrotate` prevents logs from filling the disk by rotating, compressing, and deleting old log files.

**Config file: `/etc/logrotate.d/myapp`**

```
/var/log/myapp/*.log {
    daily               # rotate every day
    rotate 7            # keep 7 rotated copies
    compress            # gzip old logs
    delaycompress       # compress previous rotation (not the just-rotated one)
    missingok           # don't error if log file is missing
    notifempty          # don't rotate if log is empty
    create 0640 www-data adm    # create new file with these perms/owner
    postrotate
        systemctl reload nginx   # reload service after rotation
    endscript
}
```

**Run logrotate manually (for testing):**

```bash
logrotate -d /etc/logrotate.d/myapp    # dry run (debug mode)
logrotate -f /etc/logrotate.d/myapp    # force rotation now
logrotate /etc/logrotate.conf          # run all configs
```

Common rotation frequencies: `daily`, `weekly`, `monthly`, `yearly`.

---

## Logging with syslog

### logger — write to syslog from scripts

```bash
logger "Backup completed successfully"
logger -p local0.err "Disk usage exceeded 90%"
logger -t myapp "Service started"
logger -s "This also prints to stderr"
```

Format: `logger -p <facility>.<level> <message>`

| Facility | Use |
|---|---|
| `auth` | authentication messages |
| `cron` | cron daemon |
| `daemon` | system daemons |
| `kern` | kernel messages |
| `local0–local7` | custom application use |
| `mail` | mail system |
| `syslog` | syslog internal |

Levels (high to low): `emerg`, `alert`, `crit`, `err`, `warning`, `notice`, `info`, `debug`

### rsyslog / journald forwarding

Messages written via `logger` appear in:
- `/var/log/syslog` (Debian/Ubuntu)
- `/var/log/messages` (RHEL/CentOS)
- `journalctl` output (systemd systems)

**Embed logging in a script:**

```bash
#!/bin/bash
log() {
    logger -t "$(basename "$0")" "$*"
    echo "[$(date '+%F %T')] $*"
}

log "Starting backup"
rsync -av /data /backup && log "Backup successful" || log "Backup FAILED"
```

---

## Monitoring User Logins to Find Intruders

### Detect multiple failed logins (brute force indicator)

```bash
lastb | awk '{print $3}' | sort | uniq -c | sort -rn | head -10
```

This prints: count, then IP/hostname — most-attempted hosts at the top.

**From journalctl (SSH failures):**

```bash
journalctl -u ssh --since "24 hours ago" | grep "Failed password" \
    | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head -20
```

### Watch for logins from unusual IPs

```bash
last | awk '{print $3}' | grep -E '^[0-9]+\.[0-9]+' | sort -u
```

Filters login records to only those from IP addresses (not `tty`/`pts`).

### Real-time login alert (add to crontab or systemd timer)

```bash
#!/bin/bash
# Run every minute, alert on new logins
NEW=$(last -n 5 | head -1)
echo "$NEW" | logger -t login-monitor
```

---

## Remote Disk Usage Health Monitor

Check free space on multiple hosts and alert if below a threshold:

```bash
#!/bin/bash
HOSTS=("server1" "server2" "server3")
THRESHOLD=90    # alert if usage >= 90%

for host in "${HOSTS[@]}"; do
    ssh "$host" "df -h --output=pcent,target" | tail -n +2 | while read -r pct mount; do
        usage=${pct//%/}    # strip the % sign
        if (( usage >= THRESHOLD )); then
            echo "ALERT: $host $mount is at ${pct} usage"
            logger -p local0.warn "ALERT: $host $mount at ${pct}"
        fi
    done
done
```

Run via cron every 15 minutes:

```
*/15 * * * * /usr/local/bin/disk_health_monitor.sh
```

---

## Finding Out Active User Hours

Build a report of when each user was active using `last`:

```bash
last | awk 'NF > 6 {print $1, $5, $6, $7}' | head -40
```

**Hour-of-day activity breakdown:**

```bash
last | grep -v "^$\|wtmp\|reboot" \
    | awk '{print $1, $5}' \
    | awk -F: '{print $1}' \
    | sort | uniq -c | sort -rn
```

This groups by user and login hour, showing when each user is most active. Useful for spotting odd-hours logins.

---

## Measuring and Optimizing Power Usage

### powertop — interactive power monitor

```bash
powertop                    # interactive TUI (requires root)
powertop --auto-tune        # apply all suggested tunables
powertop --html=report.html # generate HTML report
powertop --calibrate        # calibrate for more accurate readings
```

powertop shows per-process wakeup rates, C/P-state usage, and device power consumption.

### cpupower — CPU frequency scaling

```bash
cpupower frequency-info               # current frequency and governor
cpupower frequency-set -g powersave   # set governor to powersave
cpupower frequency-set -g performance # set governor to performance
cpupower idle-info                    # C-state (idle) information
```

Governors:
- `performance` — always max frequency (best for benchmarks)
- `powersave` — always min frequency (best for battery)
- `ondemand` / `schedutil` — scale with load (default on most distros)

### upower — battery and power source info

```bash
upower -e                         # list power devices
upower -i /org/freedesktop/UPower/devices/battery_BAT0
upower --monitor                  # watch for power events
```

### Quick power snapshot

```bash
cat /sys/class/power_supply/BAT0/capacity       # battery % (laptops)
cat /sys/class/power_supply/BAT0/status         # Charging/Discharging
cat /sys/class/power_supply/BAT0/power_now      # current power draw (µW)
```

---

## Monitoring Disk Activity

### iostat — I/O statistics

```bash
iostat                         # one-shot snapshot
iostat -x 2 5                  # extended stats, every 2s, 5 times
iostat -d sda 1                # only sda, every 1 second
```

Key columns in `iostat -x`:
- `r/s`, `w/s` — reads and writes per second
- `rMB/s`, `wMB/s` — throughput in MB/s
- `await` — average wait time per I/O request (ms)
- `%util` — how busy the device is (100% = saturated)

### iotop — per-process I/O monitor (like top for disks)

```bash
iotop                          # interactive, requires root
iotop -o                       # only show processes doing I/O (-o = only)
iotop -b -n 5                  # batch mode, 5 iterations (for scripts)
```

### lsof — files currently open

```bash
lsof                           # all open files (massive output)
lsof -u username               # files opened by a user
lsof /var/log/syslog           # who has this file open
lsof -i :80                    # processes using port 80
lsof +D /var/www               # all open files under a directory
```

---

## Checking Disks and Filesystems for Errors

### fsck — filesystem check

```bash
fsck /dev/sdb1                 # check a partition (must be unmounted)
fsck -n /dev/sdb1              # dry run (read-only check)
fsck -y /dev/sdb1              # auto-yes to all fixes
fsck -t ext4 /dev/sdb1         # specify filesystem type
```

**Never run fsck on a mounted filesystem** — it can corrupt data. Boot from live media or use `tune2fs -l` to schedule a check on next boot.

### tune2fs — ext filesystem info and settings

```bash
tune2fs -l /dev/sda1           # detailed filesystem info
tune2fs -c 30 /dev/sda1        # check every 30 mounts
tune2fs -C 0 /dev/sda1         # reset mount count (triggers check on next boot)
```

### smartctl — drive health (SMART)

```bash
smartctl -a /dev/sda           # all SMART data
smartctl -H /dev/sda           # health summary (PASSED / FAILED)
smartctl -t short /dev/sda     # run a short self-test
smartctl -t long /dev/sda      # run a long self-test
smartctl -l selftest /dev/sda  # show test results
```

**Key SMART attributes to watch:**

| Attribute | What it means |
|---|---|
| `Reallocated_Sector_Ct` | bad sectors remapped — should be 0 |
| `Current_Pending_Sector` | sectors waiting to be reallocated |
| `Offline_Uncorrectable` | unrecoverable read errors |
| `Spin_Retry_Count` | drive struggling to spin up |
| `Temperature_Celsius` | drive temperature |

### badblocks — low-level block scan

```bash
badblocks -v /dev/sdb          # read-only scan (safe on mounted)
badblocks -w /dev/sdb          # destructive write test (unmounted only!)
badblocks -sv /dev/sdb         # show progress
```

`badblocks -w` overwrites the disk — use only on empty drives or for diagnosis.

---

## Quick Reference

| Task | Command |
|---|---|
| Disk usage of directory | `du -sh /path` |
| Filesystem free space | `df -h` |
| Time a command | `time <command>` |
| Currently logged-in users | `w` or `who` |
| Login history | `last` |
| Failed logins | `lastb` |
| Kernel logs | `dmesg -T` |
| Boot logs | `journalctl -b` |
| Watch command output | `watch -n 2 <cmd>` |
| Monitor file access | `inotifywait -m /path` |
| Log from script | `logger -t tag "message"` |
| Per-process I/O | `iotop -o` |
| Disk I/O stats | `iostat -x 2` |
| Drive health | `smartctl -H /dev/sda` |
| Filesystem check | `fsck /dev/sdb1` (unmounted) |
| CPU power governor | `cpupower frequency-set -g powersave` |
