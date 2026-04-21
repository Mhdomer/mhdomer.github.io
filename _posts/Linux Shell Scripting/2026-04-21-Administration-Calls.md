---
layout: post
title: Chapter 9 Administration Calls
date: 2026-04-21T10:00:00
categories:
  - Linux Shell Scripting
tags:
  - linuxcli
  - linux
  - bash
  - scripting
  - sysadmin
  - cron
author: muhammed
description: Chapter 9 of Linux Shell Scripting Cookbook — process management, signals, system info, /proc, cron scheduling, MySQL from Bash, user admin, and terminal multiplexing
toc: true
pin: false
math: false
mermaid: false
image: https://media.licdn.com/dms/image/v2/D4D12AQG85GEyaMcdzg/article-cover_image-shrink_720_1280/article-cover_image-shrink_720_1280/0/1715198365363?e=2147483647&v=beta&t=zSE3QFqa_pLtDmQbUv_4HvOAvufhMvrOVzRy6lWimHE
Link: "[[Shell Scripting Notes]]"
---

# Chapter Overview

This chapter covers the sysadmin side of shell scripting — managing processes, scheduling tasks, querying databases, administering users, and keeping everything running cleanly from the terminal.

---

## Gathering Information About Processes

### ps — process snapshot

```bash
ps aux                         # all processes, BSD style
ps -ef                         # all processes, POSIX style
ps -eo pid,ppid,%cpu,%mem,comm # custom columns
ps -u username                 # processes owned by a user
ps --forest                    # tree view showing parent/child
ps -p 1234                     # info for a specific PID
```

**Useful custom formats:**

```bash
ps -eo pid,ppid,stat,comm,args --sort=-%cpu | head -15
```

| Column | Meaning |
|---|---|
| `pid` | process ID |
| `ppid` | parent process ID |
| `stat` | process state (R=running, S=sleeping, Z=zombie, D=uninterruptible) |
| `comm` | command name (truncated) |
| `args` | full command line |
| `%cpu` | CPU usage |
| `%mem` | memory usage |
| `etime` | elapsed time since process started |

### pgrep / pstree

```bash
pgrep nginx                    # PIDs of all nginx processes
pgrep -l nginx                 # PIDs + names
pgrep -u root                  # all root-owned processes
pgrep -a python                # full command line
pstree                         # visual tree of all processes
pstree -p                      # include PIDs
pstree -u                      # include usernames
pstree 1234                    # tree from a specific PID
```

### lsof — what a process has open

```bash
lsof -p 1234                   # all files/sockets open by PID 1234
lsof -c nginx                  # all files opened by nginx processes
lsof -i                        # all network connections
lsof -i TCP:80                 # processes on TCP port 80
lsof +D /var/www               # all processes with files under /var/www
```

### /proc filesystem

Every running process has a directory `/proc/<PID>/`:

```bash
cat /proc/1234/status          # process state, memory, UID/GID
cat /proc/1234/cmdline         # full command line (null-delimited)
cat /proc/1234/environ         # environment variables
cat /proc/1234/fd              # file descriptors (symlinks)
ls -la /proc/1234/fd           # see what files are open
cat /proc/1234/maps            # memory map
cat /proc/1234/net/tcp         # TCP connections for this process
```

---

## Killing Processes and Sending Signals

### kill / killall / pkill

```bash
kill 1234                      # send SIGTERM to PID 1234 (graceful)
kill -9 1234                   # send SIGKILL (force kill, no cleanup)
kill -HUP 1234                 # send SIGHUP (reload config)
kill -l                        # list all signal names and numbers

killall nginx                  # SIGTERM to all processes named nginx
killall -9 firefox             # force kill all firefox processes
killall -HUP sshd              # reload sshd config

pkill nginx                    # kill by name (like killall but uses pgrep logic)
pkill -u username              # kill all processes of a user
pkill -f "python script.py"    # match against full command line
```

### Common signals

| Signal | Number | Default action | Use case |
|---|---|---|---|
| `SIGHUP` | 1 | Terminate | Reload config (daemons) |
| `SIGINT` | 2 | Terminate | Ctrl+C from keyboard |
| `SIGQUIT` | 3 | Core dump | Ctrl+\\ — quit with dump |
| `SIGKILL` | 9 | Terminate (force) | Cannot be caught or ignored |
| `SIGTERM` | 15 | Terminate | Default kill — graceful shutdown |
| `SIGSTOP` | 19 | Stop | Pause process (cannot be caught) |
| `SIGCONT` | 18 | Continue | Resume a stopped process |
| `SIGUSR1/2` | 10/12 | User-defined | App-specific (e.g., nginx log rotation) |

### Trapping signals in scripts

```bash
#!/bin/bash
cleanup() {
    echo "Cleaning up before exit..."
    rm -f /tmp/myapp.lock
}
trap cleanup EXIT          # run cleanup on any exit
trap cleanup INT TERM      # run cleanup on Ctrl+C or kill

echo "Running..."
sleep 100
```

`trap` is essential for scripts that create temp files or hold locks.

### Zombie processes

A zombie (`Z` state in `ps`) is a process that has exited but whose parent hasn't called `wait()` to collect its exit status. Fix:

```bash
ps aux | grep 'Z'                  # find zombies
kill -CHLD <parent_pid>            # ask parent to collect children
# if parent ignores it — kill the parent
```

---

## Sending Messages to User Terminals

### write — send a message to a logged-in user

```bash
write username                 # opens interactive session to that user's terminal
write username pts/1           # target a specific terminal
```

Type your message, then press Ctrl+D to send.

### wall — broadcast to all logged-in users

```bash
wall "System rebooting in 5 minutes for maintenance"
echo "Scheduled downtime at 23:00" | wall
```

### mesg — control whether others can write to your terminal

```bash
mesg n                         # block write/wall messages
mesg y                         # allow messages
mesg                           # check current state
```

### notify-send — desktop notification (GUI sessions)

```bash
notify-send "Backup complete" "All files synced successfully"
notify-send -u critical "Disk Full" "Less than 1GB remaining on /var"
```

Urgency levels: `low`, `normal`, `critical`

---

## Gathering System Information

### uname

```bash
uname -a                       # all info: kernel, hostname, arch
uname -r                       # kernel version only
uname -m                       # machine architecture (x86_64, aarch64)
uname -s                       # OS name (Linux)
uname -n                       # hostname
```

### hostname and domainname

```bash
hostname                       # current hostname
hostname -f                    # fully qualified domain name (FQDN)
hostname -I                    # all IP addresses
hostnamectl                    # detailed hostname info (systemd)
hostnamectl set-hostname newname   # change hostname permanently
```

### uptime and load average

```bash
uptime                         # current time, uptime, load averages
uptime -p                      # pretty format ("up 3 days, 4 hours")
cat /proc/loadavg              # raw load averages (1, 5, 15 min + running/total)
```

Load average: number of processes wanting CPU time. On a 4-core system, a load of 4.0 means fully utilized; 8.0 means double-loaded.

### free — memory usage

```bash
free -h                        # human-readable (KB/MB/GB)
free -m                        # in megabytes
free -s 2                      # update every 2 seconds
```

**Reading the output:**
- `total` — physical RAM
- `used` — actually used by processes
- `free` — completely unused
- `buff/cache` — used by kernel buffer/cache (can be reclaimed)
- `available` — what's actually available for new processes (free + reclaimable)

### lshw, lscpu, lspci, lsusb

```bash
lscpu                          # CPU details (cores, threads, cache, freq)
lshw -short                    # summary hardware list (requires root)
lshw -class disk               # disk-specific hardware info
lspci                          # PCI devices (GPU, NIC, etc.)
lspci -v                       # verbose PCI info
lsusb                          # USB devices
lsusb -v                       # verbose USB info
```

---

## Using /proc for Gathering Information

`/proc` is a virtual filesystem — it's not real files on disk, it's the kernel exposing live data.

### Key /proc files

```bash
cat /proc/cpuinfo              # CPU details (model, cores, flags)
cat /proc/meminfo              # detailed memory stats
cat /proc/version              # kernel version + build info
cat /proc/uptime               # seconds since boot (two values: uptime + idle time)
cat /proc/loadavg              # load averages + process counts
cat /proc/filesystems          # supported filesystem types
cat /proc/mounts               # currently mounted filesystems
cat /proc/net/dev              # network interface stats (bytes/packets RX/TX)
cat /proc/net/tcp              # TCP connection table (hex format)
cat /proc/sys/kernel/hostname  # current hostname
cat /proc/sys/net/ipv4/ip_forward  # IP forwarding state (0 or 1)
```

### Tuning kernel parameters via /proc

```bash
echo 1 > /proc/sys/net/ipv4/ip_forward        # enable IP forwarding
echo 65535 > /proc/sys/net/core/somaxconn     # increase socket backlog
```

For persistence across reboots, use `/etc/sysctl.conf`:

```bash
sysctl -w net.ipv4.ip_forward=1               # set at runtime
sysctl -p                                      # reload /etc/sysctl.conf
sysctl -a                                      # show all kernel parameters
sysctl net.ipv4.tcp_keepalive_time            # read a specific parameter
```

### Extract CPU core count from /proc

```bash
grep -c "^processor" /proc/cpuinfo            # number of logical CPUs
grep "model name" /proc/cpuinfo | head -1     # CPU model
grep "MemTotal" /proc/meminfo                 # total RAM in kB
```

---

## Scheduling with cron

### crontab syntax

```
MIN  HOUR  DOM  MON  DOW  COMMAND
 *    *     *    *    *   /path/to/script.sh
```

| Field | Range | Special |
|---|---|---|
| Minute | 0–59 | `*/5` = every 5 min |
| Hour | 0–23 | `0` = midnight |
| Day of Month | 1–31 | `1` = first of month |
| Month | 1–12 | `jan`–`dec` also work |
| Day of Week | 0–7 | 0 and 7 = Sunday |

**Special strings:**

| String | Equivalent |
|---|---|
| `@reboot` | once at startup |
| `@hourly` | `0 * * * *` |
| `@daily` | `0 0 * * *` |
| `@weekly` | `0 0 * * 0` |
| `@monthly` | `0 0 1 * *` |

### Managing crontabs

```bash
crontab -e                     # edit current user's crontab
crontab -l                     # list current user's crontab
crontab -r                     # delete current user's crontab
crontab -u username -e         # edit another user's crontab (root only)
```

### Example cron jobs

```
# Backup every day at 2:30 AM
30 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1

# Clear temp files every Sunday at midnight
0 0 * * 0 find /tmp -mtime +7 -delete

# Check disk every 15 minutes and alert if > 90%
*/15 * * * * /usr/local/bin/disk_alert.sh

# Run at reboot
@reboot /usr/local/bin/start_services.sh

# First day of every month
0 9 1 * * /usr/local/bin/monthly_report.sh
```

Always redirect both stdout and stderr: `>> /var/log/job.log 2>&1`

### systemd timers (modern cron alternative)

```bash
systemctl list-timers          # show all active timers
systemctl status mytimer.timer
systemctl enable --now mytimer.timer
```

A `.timer` unit + a `.service` unit replaces a cron job, with better logging and dependency handling.

---

## Writing and Reading MySQL from Bash

### Connect and run queries

```bash
mysql -u username -p database_name          # interactive
mysql -u username -pPASSWORD database_name  # password inline (no space after -p)
mysql -u root -p -e "SHOW DATABASES;"       # run a single query and exit
```

### Bash script pattern

```bash
#!/bin/bash
DB_HOST="localhost"
DB_USER="appuser"
DB_PASS="secret"
DB_NAME="myapp"

mysql -h "$DB_HOST" -u "$DB_USER" -p"$DB_PASS" "$DB_NAME" <<'EOF'
SELECT id, username, created_at
FROM users
WHERE active = 1
ORDER BY created_at DESC
LIMIT 10;
EOF
```

### Capture query output into a variable

```bash
result=$(mysql -u root -pPASS mydb -sN -e "SELECT COUNT(*) FROM users;")
echo "Total users: $result"
```

`-s` — silent (no table borders), `-N` — no column headers.

### Loop over query results in Bash

```bash
mysql -u root -pPASS mydb -sN -e "SELECT id, name FROM products;" | \
while IFS=$'\t' read -r id name; do
    echo "Processing product $id: $name"
done
```

MySQL separates columns with tabs by default in `-s` mode — match with `IFS=$'\t'`.

### Dump and restore

```bash
mysqldump -u root -p mydb > mydb_backup.sql          # dump
mysqldump -u root -p mydb table1 table2 > tables.sql # specific tables
mysqldump --all-databases -u root -p > all_dbs.sql   # all databases

mysql -u root -p mydb < mydb_backup.sql              # restore
```

---

## User Administration Script

### Common user management commands

```bash
useradd -m -s /bin/bash username       # create user with home dir and bash shell
useradd -m -G sudo,docker username     # add to groups at creation
passwd username                        # set password interactively
usermod -aG sudo username              # add to group (never omit -a!)
usermod -s /bin/zsh username           # change shell
usermod -L username                    # lock account
usermod -U username                    # unlock account
userdel username                       # delete user (keep home dir)
userdel -r username                    # delete user and home dir

groupadd devteam                       # create a group
groupdel devteam                       # delete a group
groups username                        # show groups for a user
id username                            # UID, GID, and all groups
```

### Bulk user creation script

```bash
#!/bin/bash
# users.csv format: username,fullname,group
CSV_FILE="$1"

while IFS=',' read -r username fullname group; do
    if id "$username" &>/dev/null; then
        echo "User $username already exists, skipping"
        continue
    fi

    useradd -m -s /bin/bash -c "$fullname" -G "$group" "$username"
    # Generate a random 12-char password
    pass=$(< /dev/urandom tr -dc 'A-Za-z0-9!@#$' | head -c 12)
    echo "$username:$pass" | chpasswd
    echo "Created: $username / $pass"
    chage -d 0 "$username"    # force password change on first login
done < "$CSV_FILE"
```

### User activity and info

```bash
id username                    # UID, GID, supplementary groups
finger username                # login info, shell, last login (if installed)
chage -l username              # password expiry info
lastlog                        # last login for all users
lastlog -u username            # last login for specific user
getent passwd username         # user info from /etc/passwd (or LDAP/AD)
```

---

## Bulk Image Resizing and Format Conversion

Uses **ImageMagick** (`convert` / `mogrify`).

### convert — create a new file

```bash
convert input.png output.jpg                      # format conversion
convert input.jpg -resize 800x600 output.jpg      # resize (maintain aspect if one dim omitted)
convert input.jpg -resize 800x output.jpg         # resize width to 800, height auto
convert input.jpg -resize 50% output.jpg          # resize by percentage
convert input.jpg -quality 85 output.jpg          # set JPEG quality (1-100)
convert input.png -strip output.jpg               # remove EXIF/metadata
```

### mogrify — in-place batch processing

```bash
mogrify -resize 1024x768 *.jpg                    # resize all JPEGs in place
mogrify -format png *.jpg                         # convert all JPEGs to PNG (keeps originals)
mogrify -quality 80 -strip *.jpg                  # compress and strip metadata
mogrify -resize 800x -path ./resized/ *.jpg       # output to different dir
```

### Batch script with a loop

```bash
#!/bin/bash
mkdir -p resized

for img in *.{jpg,jpeg,png,JPG,PNG}; do
    [[ -f "$img" ]] || continue
    name="${img%.*}"
    ext="${img##*.}"
    convert "$img" -resize 1200x -quality 85 "resized/${name}_web.jpg"
    echo "Processed: $img"
done
```

### identify — inspect image info

```bash
identify image.jpg                    # format, dimensions, bit depth
identify -verbose image.jpg           # full metadata dump
identify -format "%f: %wx%h\n" *.jpg  # filename and dimensions for all
```

---

## Taking Screenshots from the Terminal

### scrot — lightweight screenshot tool

```bash
scrot screenshot.png                  # full screen screenshot
scrot -d 5 screenshot.png             # 5-second delay before capture
scrot -s screenshot.png               # select a region with mouse
scrot -u screenshot.png               # capture focused window
scrot '%Y-%m-%d_%H%M%S.png'          # timestamp in filename
```

### import (ImageMagick)

```bash
import -window root screenshot.png    # full desktop
import -window 0x123456 window.png    # specific window by ID
import screen.png                     # click to select a window
```

### gnome-screenshot

```bash
gnome-screenshot                      # interactive
gnome-screenshot -f output.png        # save to file
gnome-screenshot -a -f region.png     # select area
gnome-screenshot -w -f window.png     # active window
gnome-screenshot -d 3 -f delayed.png  # 3-second delay
```

### Automated screenshot script

```bash
#!/bin/bash
# Take a screenshot every 5 minutes (monitoring / kiosk use)
OUTPUT_DIR="$HOME/screenshots"
mkdir -p "$OUTPUT_DIR"

while true; do
    filename="$OUTPUT_DIR/$(date '+%Y-%m-%d_%H%M%S').png"
    scrot "$filename"
    echo "Captured: $filename"
    sleep 300
done
```

---

## Managing Multiple Terminals with tmux

`tmux` lets you run multiple terminal sessions inside one — and keep them alive when you disconnect from SSH.

### Core concepts

- **Session** — a collection of windows (survives disconnection)
- **Window** — like a browser tab (one per pane layout)
- **Pane** — a split view within a window

### Essential commands

```bash
tmux                           # start a new session
tmux new -s mysession          # start named session
tmux ls                        # list sessions
tmux attach -t mysession       # attach to a session
tmux kill-session -t mysession # kill a session
```

### Key bindings (default prefix: Ctrl+B)

| Binding | Action |
|---|---|
| `Ctrl+B d` | detach from session |
| `Ctrl+B c` | new window |
| `Ctrl+B n` | next window |
| `Ctrl+B p` | previous window |
| `Ctrl+B ,` | rename current window |
| `Ctrl+B %` | split vertically (side by side) |
| `Ctrl+B "` | split horizontally (top/bottom) |
| `Ctrl+B arrow` | move between panes |
| `Ctrl+B z` | zoom/unzoom current pane |
| `Ctrl+B x` | close current pane |
| `Ctrl+B [` | enter scroll/copy mode |
| `Ctrl+B ?` | show all key bindings |

### Scripted tmux session setup

```bash
#!/bin/bash
SESSION="dev"

tmux new-session -d -s "$SESSION" -n editor
tmux send-keys -t "$SESSION:editor" "vim ." Enter

tmux new-window -t "$SESSION" -n server
tmux send-keys -t "$SESSION:server" "npm run dev" Enter

tmux new-window -t "$SESSION" -n logs
tmux split-window -h -t "$SESSION:logs"
tmux send-keys -t "$SESSION:logs.left"  "tail -f /var/log/app.log" Enter
tmux send-keys -t "$SESSION:logs.right" "htop" Enter

tmux attach -t "$SESSION"
```

### screen — the older alternative

```bash
screen                         # start a new screen session
screen -S sessionname          # named session
screen -ls                     # list sessions
screen -r sessionname          # reattach
```

Key bindings (prefix: Ctrl+A):
- `Ctrl+A d` — detach
- `Ctrl+A c` — new window
- `Ctrl+A n` / `p` — next/previous window
- `Ctrl+A |` — vertical split
- `Ctrl+A S` — horizontal split

tmux is preferred over screen — better scripting support and active development.

---

## Quick Reference

| Task | Command |
|---|---|
| List all processes | `ps aux` |
| Find PID by name | `pgrep nginx` |
| Kill process gracefully | `kill <pid>` |
| Force kill | `kill -9 <pid>` |
| Kill by name | `pkill nginx` |
| Message a user | `write username` |
| Broadcast message | `wall "message"` |
| CPU info | `lscpu` |
| Memory info | `free -h` |
| Kernel params | `sysctl -a` |
| Edit crontab | `crontab -e` |
| Run MySQL query | `mysql -u user -p db -e "SELECT..."` |
| Create user | `useradd -m username` |
| Resize images | `mogrify -resize 800x *.jpg` |
| Take screenshot | `scrot output.png` |
| Start tmux session | `tmux new -s name` |
| Detach tmux | `Ctrl+B d` |
| List tmux sessions | `tmux ls` |
