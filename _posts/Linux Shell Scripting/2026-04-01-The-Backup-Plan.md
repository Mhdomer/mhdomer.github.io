---
layout: post
title: Chapter 6 The Backup Plan
date: 2026-04-01T10:00:00
categories:
  - Linux Shell Scripting
tags:
  - linuxcli
  - linux
  - bash
  - scripting
  - backup
  - tar
  - rsync
  - gzip
author: muhammed
description: Chapter 6 of Linux Shell Scripting Cookbook — archiving, compression, and backup strategies using tar, rsync, gzip, and more
toc: true
pin: false
math: false
mermaid: false
image: https://media.licdn.com/dms/image/v2/D4D12AQG85GEyaMcdzg/article-cover_image-shrink_720_1280/article-cover_image-shrink_720_1280/0/1715198365363?e=2147483647&v=beta&t=zSE3QFqa_pLtDmQbUv_4HvOAvufhMvrOVzRy6lWimHE
Link: "[[Shell Scripting Notes]]"
---

# Chapter Overview

Backups are not optional — they're what separates a recoverable incident from a catastrophe. This chapter covers the full stack of Linux backup and archiving tools: `tar`, `gzip`, `zip`, `rsync`, `cpio`, `pbzip2`, and disk imaging with `fsarchiver`. Each solves a slightly different problem, and knowing when to use which one matters.

---

## Archiving with tar

`tar` (tape archive) bundles files into a single archive. It doesn't compress by default — it just packs. Compression is a separate step, though tar can do both at once.

### Basic syntax

```bash
tar [options] [archive] [files/dirs]
```

### Create an archive

```bash
tar -cvf archive.tar /path/to/dir        # create, verbose, file
tar -cf archive.tar file1 file2 dir/     # create without verbose
tar -cvf backup.tar /etc /home /var/log  # archive multiple targets
```

- `-c` = create
- `-v` = verbose (list files as they're added)
- `-f` = the next argument is the archive filename

### Create and compress in one step

```bash
tar -czvf archive.tar.gz  /path/   # gzip  (.tar.gz or .tgz)
tar -cjvf archive.tar.bz2 /path/   # bzip2 (.tar.bz2) — smaller, slower
tar -cJvf archive.tar.xz  /path/   # xz    (.tar.xz)  — smallest, slowest
```

### Extract

```bash
tar -xvf archive.tar                     # extract here
tar -xvf archive.tar -C /target/dir/     # extract to specific directory
tar -xzvf archive.tar.gz                 # extract gzip
tar -xjvf archive.tar.bz2               # extract bzip2
tar -xJvf archive.tar.xz                # extract xz
```

### List contents without extracting

```bash
tar -tvf archive.tar                     # list all files
tar -tvf archive.tar.gz | grep ".conf"   # search inside archive
```

### Extract specific files

```bash
tar -xvf archive.tar path/to/file.txt   # extract one file
tar -xvf archive.tar --wildcards "*.conf"  # extract by pattern
```

### Exclude files or directories

```bash
tar -czvf backup.tar.gz /home/ --exclude="/home/omar/.cache"
tar -czvf backup.tar.gz /var/ \
  --exclude="*.log" \
  --exclude="*.tmp" \
  --exclude="/var/cache"
```

### Incremental backup with tar

```bash
# Full backup (snapshot file records state)
tar -czvf full_backup.tar.gz \
  --listed-incremental=snapshot.file \
  /home/

# Incremental — only changed files since last run
tar -czvf incremental_backup.tar.gz \
  --listed-incremental=snapshot.file \
  /home/
```

### Append to an existing archive

```bash
tar -rvf archive.tar newfile.txt        # append (only works on uncompressed .tar)
```

### Verify an archive

```bash
tar -tvf archive.tar.gz > /dev/null && echo "OK" || echo "CORRUPT"
```

---

## Archiving with cpio

`cpio` is older than `tar` and less common now, but it's still used in Linux initial ramdisks (initramfs) and some backup workflows. It reads a list of files from stdin.

### Create an archive

```bash
find /path -type f | cpio -ov > archive.cpio
```

- `-o` = output (create)
- `-v` = verbose

### Extract an archive

```bash
cpio -idv < archive.cpio                 # extract in current directory
cpio -idv --no-absolute-filenames < archive.cpio  # strip leading /
```

- `-i` = input (extract)
- `-d` = create directories as needed

### List contents

```bash
cpio -tv < archive.cpio
```

### Copy a directory tree (pass-through mode)

```bash
find /source -depth | cpio -pdv /destination
```

`-p` = pass-through (copy directly, no archive file).

### Compare: cpio vs tar

| Feature | tar | cpio |
|---|---|---|
| Ease of use | Easier | More complex |
| Handles special files | Good | Excellent |
| Initramfs format | No | Yes |
| Append files | Yes (uncompressed) | No |
| Common usage | General backup | Kernel/initrd |

---

## Compressing Data with gzip

`gzip` compresses individual files — it replaces the original file with a `.gz` version by default.

### Basic usage

```bash
gzip file.txt                     # compress → file.txt.gz (original deleted)
gzip -k file.txt                  # keep original
gzip -d file.txt.gz               # decompress (same as gunzip)
gunzip file.txt.gz                # decompress

gzip -l file.txt.gz               # list compression ratio and sizes
gzip -t file.txt.gz               # test integrity
```

### Compression levels

```bash
gzip -1 file.txt                  # fastest, least compression
gzip -9 file.txt                  # slowest, best compression
gzip -6 file.txt                  # default (balanced)
```

### Compress multiple files

```bash
gzip *.log                        # compress all .log files in place
gzip -r /path/to/dir/             # recursively compress all files in directory
```

### View compressed file without decompressing

```bash
zcat file.txt.gz                  # like cat but for .gz
zless file.txt.gz                 # like less but for .gz
zgrep "pattern" file.txt.gz       # grep inside .gz without extracting
zdiff file1.txt.gz file2.txt.gz   # diff two .gz files
```

### stdin/stdout (for piping)

```bash
cat file.txt | gzip > file.txt.gz        # compress from stdin
gzip -d < file.txt.gz | grep "pattern"   # decompress and search
```

### Related tools

```bash
bzip2 file.txt        # better compression than gzip, slower (.bz2)
bunzip2 file.txt.bz2  # decompress bzip2
xz file.txt           # best compression of the three (.xz)
unxz file.txt.xz      # decompress xz
lz4 file.txt          # extremely fast, moderate compression (.lz4)
zstd file.txt         # modern: fast + good compression (.zst)
```

---

## Archiving and Compressing with zip

`zip` is the standard for cross-platform archives — primarily for sharing with Windows users. Unlike `tar+gzip`, zip compresses each file individually inside the archive.

### Create a zip archive

```bash
zip archive.zip file1 file2 file3          # add specific files
zip archive.zip *.txt                       # add by pattern
zip -r archive.zip /path/to/dir/           # recursive (include directories)
zip -j archive.zip /path/*.txt             # -j = junk paths (no directory structure)
```

### Compression level

```bash
zip -0 archive.zip files    # store only (no compression)
zip -9 archive.zip files    # maximum compression
zip -6 archive.zip files    # default
```

### Password protection

```bash
zip -e archive.zip files    # prompt for password (weak encryption)
zip -P "password" archive.zip files  # inline password (visible in history)
```

### Extract

```bash
unzip archive.zip                          # extract here
unzip archive.zip -d /target/directory/    # extract to directory
unzip -l archive.zip                       # list contents
unzip -t archive.zip                       # test integrity
unzip archive.zip "*.conf"                 # extract specific files
```

### Update an existing archive

```bash
zip -u archive.zip newfile.txt             # add/update files
zip -d archive.zip oldfile.txt             # delete a file from archive
```

### zip vs tar.gz

| | zip | tar.gz |
|---|---|---|
| Cross-platform | Yes (Windows friendly) | Linux/Mac primarily |
| Random access | Yes (per-file) | No (sequential) |
| Compression | Per file | Whole archive |
| Preserves Unix permissions | Partially | Fully |
| Best for | Sharing files | System backups |

---

## Faster Archiving with pbzip2

`pbzip2` is a parallel implementation of bzip2 — it uses all CPU cores, making compression significantly faster on multi-core machines.

```bash
pbzip2 file.txt                   # compress using all cores → file.txt.bz2
pbzip2 -d file.txt.bz2            # decompress
pbzip2 -k file.txt                # keep original
pbzip2 -p 4 file.txt              # use 4 CPU cores explicitly
pbzip2 -9 file.txt                # maximum compression
```

### With tar (parallel bzip2)

```bash
tar -c /path/ | pbzip2 > archive.tar.bz2        # create
pbzip2 -d < archive.tar.bz2 | tar -x           # extract
```

### Benchmark: bzip2 vs pbzip2

```bash
time bzip2 -k largefile
time pbzip2 -k largefile
```

On a 4-core machine, pbzip2 is typically 3–4x faster.

### pigz (parallel gzip)

Same idea but for gzip:

```bash
tar -c /path/ | pigz > archive.tar.gz           # parallel gzip compress
tar -c /path/ | pigz -9 > archive.tar.gz        # max compression, parallel
pigz -d archive.tar.gz                          # decompress
```

Use pigz/pbzip2 for large backups where speed matters.

---

## Creating Filesystems with Compression

Compressed filesystems store data compressed at the block level — reads are transparent, and data is always compressed on disk.

### SquashFS — read-only compressed filesystem

Used in live CDs, embedded systems, and container layers.

```bash
# Create a SquashFS image from a directory
mksquashfs /path/to/dir output.squashfs

# With specific compression
mksquashfs /path/to/dir output.squashfs -comp xz   # best compression
mksquashfs /path/to/dir output.squashfs -comp lz4   # fastest

# Mount it (read-only)
mount -t squashfs output.squashfs /mnt/sq

# List contents without mounting
unsquashfs -l output.squashfs

# Extract
unsquashfs -d /output/dir output.squashfs
```

### Btrfs with transparent compression

Btrfs supports per-filesystem or per-directory transparent compression:

```bash
# Mount with compression
mount -o compress=zstd /dev/sdb1 /mnt/data

# Enable on existing Btrfs filesystem
btrfs property set /mnt/data compression zstd

# Check compression ratio
compsize /mnt/data    # requires compsize package
```

### NTFS compressed files (for cross-platform)

```bash
ntfs-3g -o compression /dev/sdb1 /mnt/ntfs
```

---

## Backup Snapshots with rsync

`rsync` is the gold standard for incremental backups. It only transfers changed data (delta sync), making it fast and bandwidth-efficient.

### Basic syntax

```bash
rsync [options] source destination
```

### Local sync

```bash
rsync -av /source/ /destination/       # archive mode + verbose
rsync -av --delete /source/ /dest/     # delete files in dest not in source
rsync -av --dry-run /source/ /dest/    # preview what would change
```

**Trailing slash matters:**
- `/source/` — sync the *contents* of source
- `/source` — sync the source *directory itself*

### Common flags

| Flag | Meaning |
|---|---|
| `-a` | Archive mode: preserves permissions, timestamps, symlinks, owner, group |
| `-v` | Verbose |
| `-z` | Compress during transfer |
| `-P` | Show progress + keep partial transfers |
| `--delete` | Delete files in dest that don't exist in source |
| `--exclude` | Exclude pattern |
| `--backup` | Keep backup of overwritten files |
| `--dry-run` | Simulate without making changes |
| `-n` | Same as `--dry-run` |

### Remote sync over SSH

```bash
rsync -avz /local/dir/ user@server:/remote/dir/   # local → remote
rsync -avz user@server:/remote/dir/ /local/dir/   # remote → local
rsync -avz -e "ssh -p 2222" /local/ user@host:/backup/  # custom SSH port
```

### Exclude patterns

```bash
rsync -av --exclude="*.log" --exclude=".cache/" /source/ /dest/
rsync -av --exclude-from="exclude.txt" /source/ /dest/
```

`exclude.txt`:
```
*.log
*.tmp
.cache/
node_modules/
__pycache__/
```

### Automated daily backup script

```bash
#!/bin/bash
SRC="/home/omar/"
DEST="/backup/omar/"
LOG="/var/log/rsync_backup.log"
DATE=$(date +"%Y-%m-%d %H:%M:%S")

echo "[$DATE] Starting backup" >> "$LOG"

rsync -avz --delete \
  --exclude=".cache/" \
  --exclude="*.tmp" \
  --backup \
  --backup-dir="/backup/snapshots/$(date +%Y%m%d)" \
  "$SRC" "$DEST" >> "$LOG" 2>&1

echo "[$DATE] Backup complete" >> "$LOG"
```

### Snapshot backups with hardlinks

```bash
#!/bin/bash
BACKUP_DIR="/backup"
SRC="/home/"
DATE=$(date +%Y-%m-%d)
LATEST="$BACKUP_DIR/latest"

rsync -avz --delete \
  --link-dest="$LATEST" \
  "$SRC" "$BACKUP_DIR/$DATE/"

# Update the "latest" symlink
ln -snf "$BACKUP_DIR/$DATE" "$LATEST"
```

`--link-dest` creates hardlinks for unchanged files — each snapshot looks complete but only stores the differences. Disk usage is minimal.

---

## Version Control Based Backup with Git

Git isn't just for code — it can back up any text-based configuration or document with full history.

### Basic git backup workflow

```bash
cd /etc
git init
git add .
git commit -m "Initial backup $(date +%Y-%m-%d)"

# After any changes
git add -A
git commit -m "Config update $(date +%Y-%m-%d)"
git log --oneline                # see history
git diff HEAD~1                  # what changed since last backup
```

### Push to a remote (offsite backup)

```bash
git remote add origin git@github.com:user/config-backup.git
git push -u origin main

# Scheduled push
git add -A && git commit -m "Auto backup $(date +%Y-%m-%d %H:%M)" && git push
```

### etckeeper — automated /etc version control

```bash
apt install etckeeper
etckeeper init                   # initialise /etc as a git repo
etckeeper commit "Initial"       # manual commit
# Automatically commits before apt installs packages
```

### Restore a file from history

```bash
git log --oneline -- /etc/nginx/nginx.conf    # history for one file
git show HEAD~3:etc/nginx/nginx.conf          # view old version
git checkout HEAD~3 -- etc/nginx/nginx.conf  # restore old version
```

### Backup a directory to a bare repo

```bash
# Create a bare repo on the backup server
ssh backup-server "git init --bare /backup/myrepo.git"

# Push from the machine you're backing up
cd /path/to/data
git init
git remote add backup ssh://backup-server/backup/myrepo.git
git add . && git commit -m "Backup"
git push backup main
```

---

## Creating Disk Images with fsarchiver

`fsarchiver` creates filesystem images — it understands the filesystem structure (unlike `dd`) so it can compress and restore efficiently, and even restore to a filesystem of a different size.

### Save a filesystem to an archive

```bash
fsarchiver savefs /backup/root.fsa /dev/sda1      # save root partition
fsarchiver savefs /backup/all.fsa /dev/sda1 /dev/sda2  # multiple partitions
fsarchiver savefs -z9 /backup/root.fsa /dev/sda1  # maximum compression
fsarchiver savefs -j4 /backup/root.fsa /dev/sda1  # use 4 threads
```

**Important:** The source filesystem must be unmounted or mounted read-only. Run from a live environment for the system partition.

### Restore a filesystem

```bash
fsarchiver restfs /backup/root.fsa id=0,dest=/dev/sda1
# id=0 = first filesystem in the archive
# dest = target partition (will be formatted and restored)
```

**Restore to a different sized partition:**

```bash
fsarchiver restfs /backup/root.fsa id=0,dest=/dev/sdb1
# fsarchiver handles the resize automatically — unlike dd
```

### Inspect an archive

```bash
fsarchiver archinfo /backup/root.fsa       # show archive details
```

### fsarchiver vs dd

| | fsarchiver | dd |
|---|---|---|
| Understands filesystem | Yes | No (raw blocks) |
| Compressed | Yes | Only with pipes |
| Restore to different size | Yes | No |
| Speed | Faster (skips empty space) | Copies everything |
| Handles bad sectors | Better | Can fail |
| Cross-filesystem restore | Yes | No |

### Full disk backup workflow

```bash
# Boot from live USB, then:
fsarchiver savefs -z6 -j$(nproc) /mnt/backup/system.fsa /dev/sda1
fsarchiver savefs -z6 -j$(nproc) /mnt/backup/home.fsa /dev/sda2

# Verify the archives
fsarchiver archinfo /mnt/backup/system.fsa
fsarchiver archinfo /mnt/backup/home.fsa

echo "Backup complete."
```

---

## 📚 References

<div class="references">
<ul>
  <li><a href="https://www.packtpub.com/product/linux-shell-scripting-cookbook/9781785881985" target="_blank">Linux Shell Scripting Cookbook — Packt</a></li>
  <li><a href="https://rsync.samba.org/documentation.html" target="_blank">rsync Documentation</a></li>
  <li><a href="https://www.fsarchiver.org/" target="_blank">fsarchiver Official Site</a></li>
  <li><a href="https://www.gnu.org/software/tar/manual/" target="_blank">GNU tar Manual</a></li>
</ul>
</div>

---

##  You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer )
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:**  [nonlouy](https://tryhackme.com/p/nonlouy)
