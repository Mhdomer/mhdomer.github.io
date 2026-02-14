---
layout: post
title: Linux CLI Notes
date: 2026-02-14 10:00:00 +0800
categories:
  - General Notes & Materials
tags:
  - Notes
  - bash
  - scripting
  - linuxcli
author: muhammed
description: A growing reference of Linux CLI commands, concepts, and patterns picked up over time
toc: true
pin: false
math: false
mermaid: false
image:
---

---

## Append and Overwrite Files

In Linux, the difference between overwriting and appending is just one extra character.
`>` replaces the file entirely, while `>>` adds to the bottom of it.

| Command | Action | Result |
|---|---|---|
| `echo "text" > file` | Overwrite | Deletes old content, writes "text" |
| `echo "text" >> file` | Append | Keeps old content, adds "text" at the bottom |
| `tee -a file` | Append with sudo | Safest way to append to system files like `/etc/hosts` |

**Using `>>`:**

```bash
echo "10.10.10.0 lookup.thm" >> /etc/hosts
```

**Using `tee -a` (the permission-safe way):**

Editing `/etc/hosts` directly will give a "Permission denied" error for non-root users.
Even `sudo echo ... >> /etc/hosts` fails because the redirect `>>` is handled by your shell, not by sudo.
The correct approach is to pipe through `tee` with the `-a` flag:

```bash
echo "10.10.10.0 lookup.thm" | sudo tee -a /etc/hosts
```

---

## Namespaces and cgroups

**Namespaces** isolate a process so it only sees its own version of system resources — its own process list, network stack, filesystem root, and so on.
Normally all processes share the same global view of the system.
Namespaces carve out a private "bubble" for a process or group of processes.
This is the core technology that makes **containers** (like Docker) possible.

**cgroups** (control groups) limit how much of a resource a process is allowed to use — CPU time, RAM, disk I/O, network bandwidth.

The key distinction:
- **Namespaces** → control what a process *can see*
- **cgroups** → control what a process *can consume*

Together they are the foundation of Linux containers.

### Namespace Types

| Namespace | Isolates |
|---|---|
| `PID` | Process IDs — processes only see their own PID tree |
| `Network` | Network interfaces, IPs, routing tables, ports |
| `Mount` | Filesystem mount points |
| `UTS` | Hostname and domain name |
| `IPC` | Shared memory and semaphores |
| `User` | UID/GID mappings |
| `Cgroup` | cgroup root directory visibility |
| `Time` | System clocks |

---

## Navigation and File Management

### Moving Around

```bash
pwd                        # print current directory
cd /etc                    # go to absolute path
cd ..                      # go up one level
cd ~                       # go to home directory
cd -                       # go back to previous directory
ls                         # list files
ls -la                     # long format, show hidden files
ls -lh                     # human-readable file sizes
ls -lt                     # sort by modification time (newest first)
```

### Creating and Removing

```bash
mkdir projects             # create a directory
mkdir -p a/b/c             # create nested directories (no error if exists)
touch file.txt             # create empty file or update timestamp
cp file.txt backup.txt     # copy a file
cp -r dir/ dir_copy/       # copy a directory recursively
mv old.txt new.txt         # rename or move a file
rm file.txt                # delete a file
rm -r dir/                 # delete directory recursively
rm -rf dir/                # force delete without prompts (be careful)
```

### Finding Files

```bash
find / -name "passwd"               # find by name
find /home -type f -name "*.log"    # files only, by pattern
find /var -size +100M               # files larger than 100MB
find . -mtime -1                    # modified in the last 24 hours
find . -perm 777                    # files with exact permissions
find / -user root -type f           # files owned by root
find . -name "*.conf" -exec cat {} \;   # find and run a command on results
```

---

## Viewing Files

```bash
cat file.txt               # print entire file
cat -n file.txt            # print with line numbers
less file.txt              # scroll through file (q to quit)
head file.txt              # first 10 lines
head -n 20 file.txt        # first 20 lines
tail file.txt              # last 10 lines
tail -f file.txt           # follow (live updates — great for logs)
tail -n 50 file.txt        # last 50 lines
wc -l file.txt             # count lines
wc -w file.txt             # count words
```

---

## File Permissions

Every file in Linux has three permission sets: **owner**, **group**, and **others**.
Each set has three bits: **read (r=4)**, **write (w=2)**, **execute (x=1)**.

```
-rwxr-xr--
 ^^^         owner:  rwx = 7
    ^^^      group:  r-x = 5
       ^^^   others: r-- = 4
```

### chmod — change permissions

```bash
chmod 755 script.sh          # rwxr-xr-x (owner full, group/others read+execute)
chmod 644 file.txt           # rw-r--r-- (owner read+write, others read only)
chmod +x script.sh           # add execute for everyone
chmod -w file.txt            # remove write from everyone
chmod u+x,g-w file.txt       # add execute for owner, remove write from group
chmod -R 755 /var/www/html   # apply recursively
```

### chown — change owner

```bash
chown user file.txt           # change owner
chown user:group file.txt     # change owner and group
chown -R user:group /data/    # change recursively
```

### Special permissions

```bash
chmod u+s /usr/bin/prog     # setuid — runs as file owner, not caller
chmod g+s /shared/          # setgid — new files inherit the directory's group
chmod +t /tmp/              # sticky bit — only owner can delete their files
```

### umask — default permissions

`umask` subtracts from the maximum permission (666 for files, 777 for directories).
A umask of `022` means new files are `644` and new directories are `755`.

```bash
umask            # show current umask
umask 027        # set new umask (files=640, dirs=750)
```

---

## Text Processing

### grep — search inside files

```bash
grep "error" logfile.txt             # search for pattern
grep -i "error" logfile.txt          # case-insensitive
grep -r "TODO" ./src/                # recursive search
grep -n "error" logfile.txt          # show line numbers
grep -v "debug" logfile.txt          # invert — lines that do NOT match
grep -c "error" logfile.txt          # count matching lines
grep -l "TODO" *.py                  # show only filenames that match
grep -E "error|warn|fail" log.txt    # extended regex (OR)
grep -o "[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}" file   # extract IPs
```

### cut — extract columns

```bash
cut -d: -f1 /etc/passwd             # extract field 1, delimiter ":"
cut -d, -f2,4 data.csv              # fields 2 and 4 from CSV
cut -c1-10 file.txt                 # first 10 characters of each line
```

### sort and uniq

```bash
sort file.txt                       # alphabetical sort
sort -n numbers.txt                 # numeric sort
sort -r file.txt                    # reverse sort
sort -k2 file.txt                   # sort by second column
sort file.txt | uniq                # remove duplicate lines
sort file.txt | uniq -c             # count occurrences of each line
sort file.txt | uniq -d             # show only duplicate lines
```

### sed — stream editor

```bash
sed 's/old/new/' file.txt           # replace first match per line
sed 's/old/new/g' file.txt          # replace all matches per line
sed -i 's/old/new/g' file.txt       # in-place edit
sed '/pattern/d' file.txt           # delete matching lines
sed -n '10,20p' file.txt            # print lines 10 to 20
sed '5a\new line' file.txt          # append text after line 5
```

### awk — column processing

```bash
awk '{print $1}' file.txt           # print first column
awk -F: '{print $1, $3}' /etc/passwd    # custom delimiter, print cols 1 and 3
awk '{sum += $1} END {print sum}' file  # sum a column
awk 'NR > 1' file.txt               # skip first line (header)
awk '$3 > 100' data.txt             # filter rows where column 3 > 100
awk '{print NR, $0}' file.txt       # print line numbers
```

### Pipelines — chaining commands

The pipe `|` passes the output of one command as input to the next.
You can chain as many commands as you need.

```bash
cat /etc/passwd | cut -d: -f1 | sort             # sorted list of usernames
ps aux | grep nginx | grep -v grep               # find nginx processes
cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -10   # top IPs
ls -la | grep "^d"                               # list only directories
```

---

## Process Management

### Viewing processes

```bash
ps aux                              # all processes, BSD style
ps -ef                              # all processes, POSIX style
ps aux | grep nginx                 # find a process by name
pgrep nginx                         # get PID by name
pstree                              # visual process tree
top                                 # live process monitor (q to quit)
htop                                # better top (if installed)
```

### Signals and killing

```bash
kill 1234                           # send SIGTERM (graceful stop) to PID
kill -9 1234                        # SIGKILL (force kill — no cleanup)
kill -HUP 1234                      # SIGHUP — reload config
killall nginx                       # kill all processes named nginx
pkill -f "python app.py"            # kill by matching command line
```

| Signal | Number | Meaning |
|---|---|---|
| SIGTERM | 15 | Polite stop — default `kill` |
| SIGKILL | 9 | Force kill — cannot be caught |
| SIGHUP | 1 | Hangup — reload config |
| SIGINT | 2 | Interrupt — same as Ctrl+C |
| SIGSTOP | 19 | Pause — cannot be caught |
| SIGCONT | 18 | Resume a paused process |

### Background jobs

```bash
command &                           # run in background immediately
Ctrl+Z                              # suspend current foreground process
bg                                  # resume suspended process in background
fg                                  # bring background process to foreground
jobs                                # list all background jobs
nohup command &                     # run in background, survive logout
```

---

## Disk and Storage

```bash
df -h                               # filesystem disk usage, human-readable
df -hT                              # include filesystem type
du -sh /var/log                     # total size of a directory
du -sh *                            # size of each item in current dir
du -h --max-depth=1 /               # one level deep from root
lsblk                               # list block devices (disks and partitions)
fdisk -l                            # list partition tables (root only)
mount                               # show all mounted filesystems
mount /dev/sdb1 /mnt/data           # mount a device
umount /mnt/data                    # unmount
```

---

## Networking

### Interface and routing

```bash
ip addr                             # show all interfaces and IP addresses
ip addr show eth0                   # show specific interface
ip link set eth0 up                 # bring interface up
ip route                            # show routing table
ip route show default               # show default gateway
```

### Connectivity and diagnostics

```bash
ping -c 4 8.8.8.8                   # send 4 ICMP packets
traceroute google.com               # trace route to host
mtr google.com                      # live traceroute
nslookup example.com                # DNS query
dig example.com                     # detailed DNS query
dig +short example.com              # just the IP
host example.com                    # quick DNS lookup
```

### Ports and connections

```bash
ss -tlnp                            # listening TCP ports with PID
ss -tunap                           # all TCP/UDP connections
netstat -tlnp                       # same (older command)
lsof -i :80                         # what process is using port 80
lsof -i TCP                         # all open TCP connections
```

### curl and wget

```bash
curl https://example.com                         # fetch a URL
curl -o file.html https://example.com            # save to file
curl -I https://example.com                      # headers only
curl -X POST -d '{"key":"val"}' -H "Content-Type: application/json" URL
curl -u user:pass https://api.example.com        # basic auth
curl -L https://example.com                      # follow redirects
wget https://example.com/file.zip                # download file
wget -r -np https://example.com/files/           # recursive download
```

---

## SSH

SSH is the standard way to connect to remote Linux machines securely.

```bash
ssh user@host                       # connect to remote host
ssh -p 2222 user@host               # non-standard port
ssh -i ~/.ssh/id_rsa user@host      # use a specific key
ssh -L 8080:localhost:80 user@host  # local port forwarding
ssh -R 9090:localhost:22 user@host  # remote port forwarding
ssh -D 1080 user@host               # SOCKS proxy (dynamic forwarding)
ssh -J jumphost user@target         # connect via a jump host
```

### Key management

```bash
ssh-keygen -t ed25519 -C "email@example.com"    # generate a key pair
ssh-copy-id user@host                            # copy public key to remote
ssh-add ~/.ssh/id_ed25519                        # add key to SSH agent
eval $(ssh-agent)                                # start SSH agent
```

### SCP and rsync

```bash
scp file.txt user@host:/remote/path/            # copy file to remote
scp user@host:/remote/file.txt ./               # copy file from remote
scp -r dir/ user@host:/remote/                  # copy directory

rsync -av ./local/ user@host:/remote/           # sync local to remote
rsync -av --delete ./local/ user@host:/remote/  # mirror (delete missing)
rsync -avz -e ssh ./local/ user@host:/remote/   # compressed over SSH
```

---

## User Management

```bash
whoami                              # current username
id                                  # UID, GID, and groups
groups username                     # groups a user belongs to
who                                 # logged-in users
w                                   # logged-in users + what they're running
last                                # login history
su - username                       # switch user (full login shell)
sudo command                        # run command as root
sudo -i                             # open root shell
```

### Managing users and groups

```bash
useradd -m -s /bin/bash username    # create user with home dir and bash
passwd username                     # set password
usermod -aG sudo username           # add user to sudo group (keep -a!)
usermod -s /bin/zsh username        # change shell
userdel -r username                 # delete user and home directory
groupadd devteam                    # create group
groupdel devteam                    # delete group
```

---

## Package Management

### Debian / Ubuntu (apt)

```bash
apt update                          # refresh package index
apt upgrade                         # upgrade installed packages
apt install nginx                   # install a package
apt remove nginx                    # remove package (keep config)
apt purge nginx                     # remove package and config
apt autoremove                      # remove unused dependencies
apt search keyword                  # search for packages
apt show nginx                      # package info
dpkg -l                             # list all installed packages
dpkg -i package.deb                 # install a local .deb file
```

### RHEL / CentOS / Fedora (dnf/yum)

```bash
dnf update                          # update all packages
dnf install nginx                   # install
dnf remove nginx                    # remove
dnf search keyword                  # search
dnf info nginx                      # package info
rpm -qa                             # list all installed packages
rpm -ivh package.rpm                # install a local .rpm file
```

---

## Archives and Compression

### tar

```bash
tar -czvf archive.tar.gz dir/       # create compressed archive
tar -xzvf archive.tar.gz            # extract compressed archive
tar -xzvf archive.tar.gz -C /dest/  # extract to a specific directory
tar -tzvf archive.tar.gz            # list contents without extracting
tar -cjvf archive.tar.bz2 dir/      # create bzip2 compressed archive
```

### gzip and zip

```bash
gzip file.txt                       # compress (creates file.txt.gz, removes original)
gzip -d file.txt.gz                 # decompress
gzip -k file.txt                    # compress but keep original
zcat file.txt.gz                    # view compressed file without extracting

zip -r archive.zip dir/             # zip a directory
unzip archive.zip                   # extract
unzip -l archive.zip                # list contents
```

---

## System Information

```bash
uname -a                            # kernel version, hostname, architecture
uname -r                            # kernel version only
hostname                            # machine hostname
hostname -I                         # all IP addresses
uptime                              # uptime and load averages
free -h                             # RAM usage
lscpu                               # CPU information
lsblk                               # disk layout
lspci                               # PCI devices (GPU, NIC, etc.)
lsusb                               # USB devices
dmesg | tail -20                    # recent kernel messages
dmesg -T | grep -i error            # kernel errors with timestamps
journalctl -b                       # all logs from current boot
journalctl -u nginx                 # logs for a specific service
journalctl --since "1 hour ago"     # recent logs
```

---

## Environment and Shell

### Variables

```bash
echo $PATH                          # print PATH variable
echo $HOME                          # print home directory
MY_VAR="hello"                      # set local variable
export MY_VAR="hello"               # export to child processes
env                                 # show all environment variables
unset MY_VAR                        # remove a variable
printenv PATH                       # print a specific env var
```

### PATH management

```bash
export PATH="$HOME/.local/bin:$PATH"    # prepend to PATH
export PATH="$PATH:/opt/tool/bin"       # append to PATH
which python3                           # find where a command lives
whereis nginx                           # find binary, source, and man page
type ls                                 # show how a command is resolved
```

### Aliases

```bash
alias ll='ls -la'                   # create alias
alias gs='git status'               # git shortcut
unalias ll                          # remove alias
alias                               # list all aliases
```

Put permanent aliases in `~/.bashrc` or `~/.zshrc` and run `source ~/.bashrc` to reload.

### History

```bash
history                             # show command history
history | grep ssh                  # search history
!!                                  # repeat last command
!ssh                                # repeat last command starting with "ssh"
Ctrl+R                              # interactive reverse history search
history -c                          # clear history
```

---

## Redirection

```bash
command > file.txt                  # redirect stdout to file (overwrite)
command >> file.txt                 # redirect stdout to file (append)
command < file.txt                  # read stdin from file
command 2> error.txt                # redirect stderr to file
command 2>&1                        # redirect stderr to stdout
command &> file.txt                 # redirect both stdout and stderr
command > /dev/null                 # discard stdout
command > /dev/null 2>&1            # discard all output
command | tee file.txt              # write to file AND show on screen
command | tee -a file.txt           # append to file AND show on screen
```

---

## Scheduling with cron

```bash
crontab -e                          # edit cron jobs for current user
crontab -l                          # list cron jobs
crontab -r                          # remove all cron jobs
```

Crontab format:

```
MIN  HOUR  DOM  MON  DOW  COMMAND
 *    *     *    *    *   /path/to/script.sh
```

Examples:

```bash
0 2 * * *    /backup.sh            # daily at 2 AM
*/15 * * * * /check.sh             # every 15 minutes
0 9 * * MON  /report.sh            # every Monday at 9 AM
@reboot      /startup.sh           # once at boot
```

---

## Useful One-Liners

```bash
# Find and kill a process by port
kill $(lsof -t -i:8080)

# Show top 10 largest files in current directory
du -ah . | sort -rh | head -10

# Watch disk usage in real time
watch -n 5 df -h

# Count lines in all Python files
find . -name "*.py" | xargs wc -l | tail -1

# Replace text in all files in a directory
grep -rl "old_text" . | xargs sed -i 's/old_text/new_text/g'

# Extract all IP addresses from a file
grep -oE '\b([0-9]{1,3}\.){3}[0-9]{1,3}\b' file.txt

# Show all open ports
ss -tlnp

# Test if a port is open
nc -zv host 443

# Generate a random password
openssl rand -base64 16

# Check certificate expiry
echo | openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -dates

# Monitor a log file and highlight errors
tail -f /var/log/syslog | grep --line-buffered -i "error\|warn\|fail"
```

---

##  You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer)
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:** [nonlouy](https://tryhackme.com/p/nonlouy)
