---
layout: post
title: Chapter 3 File In, File Out
date: 2026-03-11T10:00:00
categories:
  - Linux Shell Scripting
tags:
  - linuxcli
  - linux
  - bash
  - scripting
  - filesystem
author: muhammed
description: Chapter 3 of Linux Shell Scripting Cookbook — file creation, permissions, comparison, navigation, and filesystem operations
toc: true
pin: false
math: false
mermaid: false
image: https://media.licdn.com/dms/image/v2/D4D12AQG85GEyaMcdzg/article-cover_image-shrink_720_1280/article-cover_image-shrink_720_1280/0/1715198365363?e=2147483647&v=beta&t=zSE3QFqa_pLtDmQbUv_4HvOAvufhMvrOVzRy6lWimHE
Link: "[[Shell Scripting Notes]]"
---

# Chapter Overview

This chapter is all about files — creating them, comparing them, protecting them, navigating around them, and understanding what the filesystem is actually doing. Most of this is foundational for any serious shell scripting or sysadmin work.

---

## Generating Files of Any Size

### dd

`dd` reads and writes raw data block by block. The go-to tool for creating files of an exact size.

```bash
dd if=/dev/zero of=testfile.img bs=1M count=100   # 100MB file filled with zeros
dd if=/dev/urandom of=random.bin bs=1M count=10   # 10MB of random data
dd if=/dev/zero of=swap.img bs=1M count=1024      # 1GB swap file
```

- `if` = input file (`/dev/zero` = infinite zeros, `/dev/urandom` = random bytes)
- `of` = output file
- `bs` = block size
- `count` = number of blocks

### truncate

Creates a sparse file instantly — doesn't actually write data, just sets the size in the filesystem metadata. Much faster than `dd` when you just need a placeholder.

```bash
truncate -s 1G bigfile.img     # create a 1GB sparse file
truncate -s 100M test.img
truncate -s +500M file.img     # extend an existing file by 500MB
```

**Sparse vs real:** `ls -lh` shows the declared size. `du -sh` shows actual disk usage. A sparse file shows 1G in `ls` but 0 in `du`.

### fallocate

Actually allocates disk space (not sparse) — faster than `dd`:

```bash
fallocate -l 1G testfile.img   # allocate 1GB immediately
```

---

## Intersection and Set Difference on Text Files

These are set operations on sorted text files — each line is treated as an element.

### comm

`comm` compares two **sorted** files and outputs three columns:

- Column 1: lines only in file1
- Column 2: lines only in file2
- Column 3: lines in both

```bash
comm file1.txt file2.txt
```

Suppress columns to get specific operations:

```bash
comm -12 file1.txt file2.txt    # intersection — lines in BOTH
comm -23 file1.txt file2.txt    # difference A-B — lines only in file1
comm -13 file1.txt file2.txt    # difference B-A — lines only in file2
```

**Always sort first:**

```bash
comm -12 <(sort file1.txt) <(sort file2.txt)
```

### grep for set operations

```bash
# Lines in file1 that are NOT in file2
grep -vxFf file2.txt file1.txt

# Lines in file1 that ARE in file2
grep -xFf file2.txt file1.txt
```

`-x` = whole line match, `-F` = fixed string (no regex), `-f` = read patterns from file.

---

## Finding and Deleting Duplicate Files

### fdupes

Purpose-built for finding duplicate files by content:

```bash
fdupes -r /path                 # recursive search
fdupes -r -d /path              # delete duplicates (interactive)
fdupes -r -N -d /path           # delete automatically (keep first, no prompt)
fdupes -r -S /path              # show size of duplicates
```

Install: `apt install fdupes`

### Manual approach using checksums

```bash
# Find all duplicate files in current directory
find . -type f | xargs md5sum | sort | uniq -w32 -D
```

`-w32` compares only the first 32 characters (the hash), `-D` shows all duplicate lines.

**Delete duplicates keeping one copy:**

```bash
find . -type f -exec md5sum {} \; | sort | \
awk 'seen[$1]++ {print $2}' | xargs rm
```

---

## File Permissions, Ownership, and the Sticky Bit

### Understanding permissions

```
-rwxr-xr--  1  omar  staff  4096  Mar 11 10:00  script.sh
 ^^^         ^  ^^^^  ^^^^^
 |           |  |     group
 |           |  owner
 |           link count
 |
 [type][owner][group][others]
```

Each set of 3: `r` (read=4), `w` (write=2), `x` (execute=1)

### chmod

```bash
chmod 755 script.sh          # rwxr-xr-x
chmod 644 file.txt           # rw-r--r--
chmod 600 private.key        # rw------- (only owner can read)
chmod 777 file               # rwxrwxrwx (everyone — avoid this)

chmod +x script.sh           # add execute for everyone
chmod -x script.sh           # remove execute
chmod u+x script.sh          # add execute for owner only
chmod go-w file.txt          # remove write from group and others
chmod a+r file.txt           # add read for all (a = all)
```

**Recursive:**

```bash
chmod -R 755 /var/www/html
```

### chown

```bash
chown omar file.txt              # change owner
chown omar:staff file.txt        # change owner and group
chown :staff file.txt            # change group only
chown -R omar:staff /var/www     # recursive
```

### Special bits

**Setuid (4)** — file runs as its owner, not the caller:

```bash
chmod u+s /usr/bin/passwd    # passwd runs as root regardless of caller
chmod 4755 file              # numeric: 4 = setuid
```

**Setgid (2)** — on a directory, new files inherit the directory's group:

```bash
chmod g+s /shared/           # new files in /shared get the directory's group
chmod 2775 /shared/
```

**Sticky bit (1)** — on a directory, only the file owner can delete their own files:

```bash
chmod +t /tmp                # classic sticky bit use case
chmod 1777 /tmp              # /tmp permissions — anyone writes, only owner deletes
ls -ld /tmp                  # shows as drwxrwxrwt (t at the end)
```

---

## Making Files Immutable

Even root can't modify an immutable file. Used for protecting critical config files.

```bash
chattr +i file.txt           # set immutable
chattr -i file.txt           # remove immutable
lsattr file.txt              # check attributes
```

**Other chattr flags:**

| Flag | Meaning |
|---|---|
| `+i` | Immutable — no write, delete, rename, link |
| `+a` | Append-only — can only add to the file, not modify or delete |
| `+u` | Undeletable — data preserved when deleted (for recovery) |
| `+c` | Compressed automatically by the kernel |

**Protect a directory (recursively):**

```bash
chattr -R +i /etc/critical/
```

**Note:** Requires root. Even `sudo rm` will fail on an immutable file.

---

## Generating Blank Files in Bulk

### touch

`touch` creates empty files or updates timestamps:

```bash
touch file.txt               # create empty file (or update timestamp if exists)
touch file1 file2 file3      # create multiple
touch -t 202602251200 file   # set specific timestamp: YYYYMMDDhhmm
```

**Bulk creation:**

```bash
# Create 100 numbered files
for i in $(seq 1 100); do touch "file_$i.txt"; done

# Shorter with brace expansion
touch file_{1..100}.txt

# Create files with today's date
touch report_$(date +%Y%m%d).txt
```

**Create a directory tree with files in one line:**

```bash
touch logs/{access,error,debug}.log
touch tests/{unit,integration,e2e}.sh
```

---

## Finding Symbolic Links and Their Targets

### find symlinks

```bash
find /path -type l                      # find all symlinks
find /path -type l -name "*.conf"       # symlinks matching a pattern
find /path -xtype l                     # broken symlinks (target doesn't exist)
```

### readlink

```bash
readlink symlink.txt                    # show target (one level)
readlink -f symlink.txt                 # fully resolved path (follows chains)
readlink -e symlink.txt                 # like -f but fails if target doesn't exist
```

### ls -l

```bash
ls -la /path | grep "^l"               # list only symlinks
```

### Find and remove broken symlinks

```bash
find /path -xtype l -delete            # delete all broken symlinks
find /path -xtype l -exec rm {} \;    # same with explicit rm
```

**List all symlinks with their targets:**

```bash
find . -type l | while read link; do
  echo "$link -> $(readlink -f $link)"
done
```

---

## Enumerating File Type Statistics

### file

Detects the actual type of a file — ignores the extension, reads the magic bytes:

```bash
file image.png          # PNG image data, 1920 x 1080
file script.sh          # Bourne-Again shell script, ASCII text executable
file archive.tar.gz     # gzip compressed data
file binary             # ELF 64-bit LSB executable
file unknown            # ASCII text / data / etc.
```

**On multiple files:**

```bash
file *                  # check all files in current dir
find . -type f | xargs file
```

### Count files by type

```bash
find . -type f | xargs file | awk -F: '{print $2}' | sort | uniq -c | sort -rn
```

**Count by extension:**

```bash
find . -type f | sed 's/.*\.//' | sort | uniq -c | sort -rn
```

**Disk usage by file type:**

```bash
find . -name "*.log" -exec du -sh {} + | sort -h
```

---

## Using Loopback Files

A loopback file is a regular file treated as a block device — you can format it and mount it as a filesystem. Useful for creating disk images, testing, or portable encrypted containers.

**Create and mount a loopback filesystem:**

```bash
# 1. Create a 500MB file
dd if=/dev/zero of=disk.img bs=1M count=500

# 2. Format it as ext4
mkfs.ext4 disk.img

# 3. Mount it
mkdir /mnt/disk
mount -o loop disk.img /mnt/disk

# 4. Use it like a normal directory
cp files/* /mnt/disk/

# 5. Unmount when done
umount /mnt/disk
```

**Mount automatically using losetup:**

```bash
losetup /dev/loop0 disk.img       # attach to loop device
losetup -l                        # list all loop devices
losetup -d /dev/loop0             # detach
```

**Encrypted loopback container:**

```bash
dd if=/dev/urandom of=secure.img bs=1M count=200
cryptsetup luksFormat secure.img
cryptsetup luksOpen secure.img secure_vol
mkfs.ext4 /dev/mapper/secure_vol
mount /dev/mapper/secure_vol /mnt/secure
```

---

## Creating ISO Files and Hybrid ISO

### Create ISO from a directory

```bash
genisoimage -o output.iso -R -J /path/to/directory
```

- `-R` = Rock Ridge extensions (preserves Unix permissions)
- `-J` = Joliet extensions (Windows-compatible filenames)
- `-V "Label"` = set volume label

Or with `mkisofs` (same tool, different name on some distros):

```bash
mkisofs -o output.iso -R -J -V "MyDisk" /path/to/directory
```

### Create a bootable hybrid ISO

A hybrid ISO works both as a CD/DVD image and can be written directly to a USB drive:

```bash
isohybrid output.iso            # make it USB-bootable
```

### Write ISO to USB

```bash
dd if=output.iso of=/dev/sdb bs=4M status=progress && sync
```

**Always double-check the target device** with `lsblk` before running this — wrong device = data loss.

### Mount an ISO without burning

```bash
mkdir /mnt/iso
mount -o loop output.iso /mnt/iso
```

---

## Finding the Difference Between Files and Patching

### diff

```bash
diff file1.txt file2.txt           # basic diff
diff -u file1.txt file2.txt        # unified format (most readable, used in patches)
diff -i file1.txt file2.txt        # ignore case
diff -w file1.txt file2.txt        # ignore whitespace
diff -r dir1/ dir2/                # recursive directory diff
```

Unified format output:

```
--- file1.txt  (original)
+++ file2.txt  (new)
@@ -1,4 +1,4 @@   (line numbers)
 unchanged line
-removed line
+added line
```

### Creating and applying patches

**Create a patch:**

```bash
diff -u original.txt modified.txt > changes.patch
```

**Apply a patch:**

```bash
patch original.txt < changes.patch         # apply to single file
patch -p1 < changes.patch                  # apply to directory tree (-p strips path prefix)
patch --dry-run -p1 < changes.patch        # test without applying
patch -R original.txt < changes.patch      # reverse (undo) a patch
```

### vimdiff / colordiff

```bash
vimdiff file1.txt file2.txt        # side-by-side in vim
colordiff file1.txt file2.txt      # color-coded diff output
```

---

## Head and Tail

### head

```bash
head file.txt              # first 10 lines (default)
head -n 20 file.txt        # first 20 lines
head -c 100 file.txt       # first 100 bytes
head -n -5 file.txt        # everything EXCEPT the last 5 lines
```

### tail

```bash
tail file.txt              # last 10 lines (default)
tail -n 20 file.txt        # last 20 lines
tail -c 100 file.txt       # last 100 bytes
tail -n +50 file.txt       # from line 50 to end (skip first 49)
tail -f log.txt            # follow — live updates as file grows
tail -F log.txt            # follow even if file is rotated (reopens on rename)
```

**Combine head and tail to extract a range:**

```bash
# Lines 20 to 30
head -n 30 file.txt | tail -n 11

# Or with sed (cleaner)
sed -n '20,30p' file.txt
```

**Monitor multiple log files at once:**

```bash
tail -f /var/log/syslog /var/log/auth.log
```

---

## Listing Only Directories

Several ways to list just directories — each has tradeoffs:

```bash
ls -d */                      # glob: directories in current dir only
ls -la | grep "^d"            # filter ls output by permission string
find . -maxdepth 1 -type d    # find: precise, handles edge cases
find . -maxdepth 1 -type d -not -name "."  # exclude current dir itself
```

**With details:**

```bash
ls -lhd */                    # long format, directories only
```

**Recursive — all directories in the tree:**

```bash
find . -type d
find . -type d | sort         # sorted
```

**Count directories:**

```bash
find . -maxdepth 1 -type d | wc -l
```

---

## Fast Navigation with pushd and popd

`cd` is one-way. `pushd`/`popd` maintain a **directory stack** so you can jump between locations instantly.

```bash
pushd /var/log          # go to /var/log AND push current dir to stack
pushd /etc/nginx        # go to /etc/nginx AND push /var/log to stack
pushd /tmp              # stack is now: /tmp /etc/nginx /var/log ~

dirs                    # show the stack
popd                    # go back to /etc/nginx, remove from stack
popd                    # go back to /var/log
popd                    # go back to home
```

**Jump to a specific stack position:**

```bash
dirs -v                 # show stack with index numbers
pushd +2               # rotate stack to bring index 2 to front
```

**Practical use in scripts:**

```bash
pushd /tmp > /dev/null        # suppress output
# do work in /tmp
popd > /dev/null              # return to original directory
```

This is cleaner than saving `$(pwd)` and `cd`-ing back — the stack handles multiple levels automatically.

---

## Counting Lines, Words, and Characters

### wc

```bash
wc file.txt              # lines, words, characters (all three)
wc -l file.txt           # line count only
wc -w file.txt           # word count only
wc -c file.txt           # byte count
wc -m file.txt           # character count (differs from -c for multibyte)
```

**Multiple files:**

```bash
wc -l *.txt              # count lines in each, plus total
wc -l *.log | sort -n    # sorted by line count
```

**From stdin:**

```bash
cat file.txt | wc -l
echo "hello world" | wc -w    # 2
```

**Count files in a directory:**

```bash
ls | wc -l
find . -type f | wc -l         # count all files recursively
find . -name "*.py" | wc -l    # count specific file type
```

**Count unique lines:**

```bash
sort file.txt | uniq | wc -l
```

---

## Printing the Directory Tree

### tree

```bash
tree                           # current directory tree
tree /path/to/dir              # specific directory
tree -L 2                      # limit to 2 levels deep
tree -d                        # directories only
tree -a                        # include hidden files
tree -h                        # show file sizes in human-readable format
tree -f                        # show full path for each file
tree -I "node_modules|*.pyc"   # exclude patterns (pipe-separated)
tree --dirsfirst               # show directories before files
```

**Output to a file:**

```bash
tree > structure.txt
tree -H . > structure.html     # HTML output
```

Install: `apt install tree`

### Without tree — using find

```bash
find . | sed -e 's/[^\/]*\//│   /g' -e 's/│   \([^│]\)/└── \1/'
```

Or a cleaner version:

```bash
find . -type d | sort | sed 's|[^/]*/|  |g'
```

### ls recursively

```bash
ls -R                          # recursive list (messy for large trees)
ls -R | grep ":$" | sed 's/:$//' | sed 's/[^\/]*\//  /g'  # directories only
```

---

## 📚 References

<div class="references">
<ul>
  <li><a href="https://www.packtpub.com/product/linux-shell-scripting-cookbook/9781785881985" target="_blank">Linux Shell Scripting Cookbook — Packt</a></li>
  <li><a href="https://www.gnu.org/software/bash/manual/" target="_blank">GNU Bash Manual</a></li>
  <li><a href="https://man7.org/linux/man-pages/" target="_blank">Linux Man Pages</a></li>
</ul>
</div>

---

##  You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer )
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:**  [nonlouy](https://tryhackme.com/p/nonlouy)
