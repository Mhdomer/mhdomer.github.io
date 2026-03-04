---
layout: post
title: Chapter 2 Have a Good Command
date: 2026-03-04T10:00:00
categories:
  - Linux Shell Scripting
tags:
  - linuxcli
  - linux
  - bash
  - scripting
author: muhammed
description: Chapter 2 of Linux Shell Scripting Cookbook — core command-line tools every scripter needs
toc: true
pin: false
math: false
mermaid: false
image: https://media.licdn.com/dms/image/v2/D4D12AQG85GEyaMcdzg/article-cover_image-shrink_720_1280/article-cover_image-shrink_720_1280/0/1715198365363?e=2147483647&v=beta&t=zSE3QFqa_pLtDmQbUv_4HvOAvufhMvrOVzRy6lWimHE
Link: "[[Shell Scripting Notes]]"
---

# Chapter Overview

This chapter covers the essential command-line tools that turn a basic shell user into someone who can actually get things done fast — file operations, searching, filtering, checksums, parallelism, and more.

---

## Concatenating with cat

`cat` (concatenate) reads files and writes them to stdout. Simple but used everywhere.

```bash
cat file.txt                  # print a file
cat file1.txt file2.txt       # concatenate and print both
cat file1.txt file2.txt > combined.txt  # merge into one file
```

**Useful flags:**

```bash
cat -n file.txt    # number every line
cat -b file.txt    # number only non-blank lines
cat -s file.txt    # squeeze multiple blank lines into one
cat -A file.txt    # show hidden chars — tabs as ^I, line endings as $
```

**Create a file from stdin** (type content, Ctrl+D to finish):

```bash
cat > newfile.txt
```

**Append to a file:**

```bash
cat >> file.txt
```

**Pipe into other commands:**

```bash
cat access.log | grep "404" | wc -l
```

---

## Recording and Playing Back Terminal Sessions

`script` records everything printed to the terminal — commands and output. Useful for documentation, demos, and audit trails.

**Record a session:**

```bash
script session.log         # start recording to session.log
# do your work
exit                       # stop recording
```

The timing file records exactly when each output occurred:

```bash
script --timing=timing.log session.log
```

**Replay the session at the original speed:**

```bash
scriptreplay timing.log session.log
```

**Replay faster or slower:**

```bash
scriptreplay -d 2 timing.log session.log   # 2x speed
scriptreplay -d 0.5 timing.log session.log # half speed
```

---

## Finding Files and File Listing

### ls

```bash
ls -l      # long format — permissions, size, date
ls -a      # show hidden files (starting with .)
ls -h      # human-readable sizes (KB, MB)
ls -t      # sort by modification time (newest first)
ls -S      # sort by size (largest first)
ls -r      # reverse order
ls -lhtr   # combine: long, human, time, reversed — most useful combo
```

### find

`find` is the real power tool for locating files.

```bash
find /path -name "*.log"          # find by name (case-sensitive)
find /path -iname "*.log"         # case-insensitive
find /path -type f                # files only
find /path -type d                # directories only
find /path -size +10M             # files larger than 10MB
find /path -size -1k              # files smaller than 1KB
find /path -mtime -7              # modified in the last 7 days
find /path -mtime +30             # modified more than 30 days ago
find /path -user omar             # owned by user omar
find /path -perm 644              # exact permissions
find /path -perm /u+x             # executable by owner
```

**Execute a command on each result:**

```bash
find . -name "*.log" -exec rm {} \;          # delete all .log files
find . -name "*.py" -exec chmod +x {} \;     # make all .py executable
find . -type f -exec wc -l {} +              # count lines in all files
```

**Combine conditions:**

```bash
find . -name "*.sh" -size +1k -mtime -7      # AND (default)
find . -name "*.log" -o -name "*.txt"         # OR
find . -not -name "*.bak"                     # NOT
```

---

## Playing with xargs

`xargs` takes stdin and passes it as arguments to another command. Essential for combining `find` with other tools.

```bash
find . -name "*.log" | xargs rm           # delete all found files
find . -name "*.txt" | xargs wc -l        # count lines in each
find . -name "*.py" | xargs grep "import" # search inside each
```

**Why not just use pipes?** Some commands don't read from stdin — they only take arguments. `xargs` bridges that gap.

**Handle filenames with spaces** — use null delimiter:

```bash
find . -name "*.txt" -print0 | xargs -0 rm
```

`-print0` outputs null-separated names, `-0` tells xargs to split on null. Always use this combo when filenames might have spaces.

**Limit arguments per command:**

```bash
find . -name "*.log" | xargs -n 2 echo    # pass 2 args at a time
```

**Show what would be executed (dry run):**

```bash
find . -name "*.log" | xargs -p rm        # -p asks confirmation for each
```

**Run in parallel:**

```bash
find . -name "*.png" | xargs -P 4 -I {} convert {} {}.jpg
```

`-P 4` runs 4 processes at once. `-I {}` defines a placeholder for the filename.

---

## Translating with tr

`tr` translates (replaces) or deletes characters. Works on stdin only — no file arguments.

```bash
echo "hello world" | tr 'a-z' 'A-Z'    # lowercase to uppercase
echo "HELLO" | tr 'A-Z' 'a-z'          # uppercase to lowercase
echo "hello 123" | tr -d '0-9'          # -d delete digits
echo "hello   world" | tr -s ' '        # -s squeeze repeated spaces to one
echo "hello:world" | tr ':' '\n'         # replace : with newline
```

**Common character classes:**

```bash
tr '[:lower:]' '[:upper:]'   # portable way to uppercase
tr '[:digit:]' '*'           # replace all digits with *
tr -d '[:space:]'            # remove all whitespace
tr -d '\r'                   # remove Windows carriage returns (fix CRLF files)
```

**Remove non-printable characters from a file:**

```bash
cat file.txt | tr -cd '[:print:]\n' > clean.txt
```

---

## Checksum and Verification

Checksums verify file integrity — confirm a file wasn't corrupted or tampered with.

**Generate checksums:**

```bash
md5sum file.txt                  # MD5 (fast, not secure for crypto)
sha1sum file.txt                 # SHA-1
sha256sum file.txt               # SHA-256 (standard for verification)
sha512sum file.txt               # SHA-512 (stronger)
```

**Verify a file against a known checksum:**

```bash
echo "expectedhash  file.txt" | sha256sum --check
sha256sum --check checksums.txt   # verify a whole list
```

**Generate checksums for multiple files:**

```bash
sha256sum *.iso > checksums.txt   # save all checksums
sha256sum --check checksums.txt   # verify later
```

**Compare two files quickly:**

```bash
sha256sum file1.txt file2.txt
# if the hashes match, the files are identical
```

---

## Cryptographic Tools and Hashes

### OpenSSL

```bash
# Generate a random password
openssl rand -base64 32

# Hash a string
echo -n "password" | openssl dgst -sha256

# Encrypt a file (AES-256)
openssl enc -aes-256-cbc -salt -in plain.txt -out encrypted.enc

# Decrypt
openssl enc -aes-256-cbc -d -in encrypted.enc -out plain.txt

# Generate RSA key pair
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem
```

### gpg

```bash
# Generate a key pair
gpg --gen-key

# Encrypt a file for a recipient
gpg --encrypt --recipient "user@example.com" file.txt

# Decrypt
gpg --decrypt file.txt.gpg

# Sign a file
gpg --sign file.txt

# Verify a signature
gpg --verify file.txt.gpg
```

### bcrypt / password hashing

```bash
# Quick password hash with openssl (for scripts)
openssl passwd -6 "mypassword"    # SHA-512 crypt format
```

---

## Sorting, Unique, and Duplicates

### sort

```bash
sort file.txt                  # alphabetical sort
sort -r file.txt               # reverse
sort -n numbers.txt            # numeric sort
sort -k 2 file.txt             # sort by 2nd field
sort -t ',' -k 3 -n file.csv  # sort CSV by 3rd column numerically
sort -u file.txt               # sort and remove duplicates
```

**Sort by multiple keys:**

```bash
sort -k 1,1 -k 2,2n file.txt  # sort by field 1 alpha, then field 2 numeric
```

### uniq

`uniq` only removes *adjacent* duplicates — always sort first.

```bash
sort file.txt | uniq           # remove duplicates
sort file.txt | uniq -c        # count occurrences of each line
sort file.txt | uniq -d        # show only duplicated lines
sort file.txt | uniq -u        # show only unique lines (not duplicated)
```

**Find the most common lines in a log:**

```bash
sort access.log | uniq -c | sort -rn | head -10
```

### comm

Compare two sorted files:

```bash
comm file1.txt file2.txt
# column 1: lines only in file1
# column 2: lines only in file2
# column 3: lines in both
```

```bash
comm -12 file1.txt file2.txt   # only lines in BOTH files
comm -23 file1.txt file2.txt   # only lines in file1 (not file2)
```

---

## Temporary File Naming and Random Numbers

### mktemp

Never manually name temp files — use `mktemp` to avoid race conditions and collisions.

```bash
tmpfile=$(mktemp)                    # /tmp/tmp.XxXxXx
tmpfile=$(mktemp /tmp/myapp.XXXXXX)  # custom prefix + random suffix
tmpdir=$(mktemp -d)                  # create a temp directory
```

Always clean up:

```bash
tmpfile=$(mktemp)
trap "rm -f $tmpfile" EXIT           # auto-delete when script exits
```

### $RANDOM

Bash built-in — generates a random integer between 0 and 32767.

```bash
echo $RANDOM                         # random number
echo $(( RANDOM % 100 ))             # random 0-99
echo $(( RANDOM % 50 + 1 ))          # random 1-50
```

**Random filename:**

```bash
name="backup_$(date +%Y%m%d)_$RANDOM.tar.gz"
```

**Shuffle lines in a file:**

```bash
sort -R file.txt                     # random sort (shuffle)
```

---

## Splitting Files and Data

### split

Split a large file into smaller pieces:

```bash
split -l 1000 bigfile.txt part_      # 1000 lines per piece, prefix "part_"
split -b 10M bigfile.tar part_       # 10MB per piece
split -n 5 bigfile.txt part_         # split into exactly 5 pieces
```

Reassemble:

```bash
cat part_* > reassembled.txt
```

### csplit

Split on a pattern (context-aware split):

```bash
csplit log.txt '/ERROR/' '{*}'       # split every time ERROR appears
csplit file.txt 10 25                # split at lines 10 and 25
```

### head and tail

```bash
head -n 100 file.txt                 # first 100 lines
tail -n 100 file.txt                 # last 100 lines
tail -f log.txt                      # follow (live updates — great for logs)
tail -n +50 file.txt                 # from line 50 to end
```

---

## Slicing Filenames Based on Extension

### basename and dirname

```bash
basename /home/omar/docs/report.pdf         # report.pdf
basename /home/omar/docs/report.pdf .pdf    # report (strip extension)
dirname /home/omar/docs/report.pdf          # /home/omar/docs
```

### Parameter expansion

The most efficient way — no subshell needed:

```bash
filepath="/home/omar/docs/report.pdf"

echo "${filepath##*/}"       # report.pdf     (filename only)
echo "${filepath%/*}"        # /home/omar/docs (directory only)
echo "${filepath##*.}"       # pdf            (extension only)
echo "${filepath%.*}"        # /home/omar/docs/report (strip extension)
echo "${filepath##*.}"       # pdf
```

**Change extension in a loop:**

```bash
for f in *.txt; do
  mv "$f" "${f%.txt}.md"    # rename .txt to .md
done
```

---

## Renaming and Moving Files in Bulk

### mv in a loop

```bash
# Add prefix to all .txt files
for f in *.txt; do
  mv "$f" "backup_$f"
done

# Change extension
for f in *.jpeg; do
  mv "$f" "${f%.jpeg}.jpg"
done
```

### rename (Perl rename)

More powerful — uses regex:

```bash
rename 's/\.txt$/.md/' *.txt          # change extension
rename 's/ /_/g' *                     # replace spaces with underscores
rename 's/^/prefix_/' *                # add prefix to all files
rename 'y/A-Z/a-z/' *                  # lowercase all filenames
```

**Dry run first** (check before applying):

```bash
rename -n 's/\.txt$/.md/' *.txt        # -n = dry run, shows what would happen
```

### find + mv for subdirectories

```bash
find . -name "*.log" -exec mv {} /archive/ \;
```

---

## Spell Checking and Dictionary Manipulation

### look

Search for words starting with a prefix — uses `/usr/share/dict/words`:

```bash
look uni          # all dictionary words starting with "uni"
look "uni" /usr/share/dict/words
```

### aspell

Interactive spell checker:

```bash
aspell check report.md         # interactive correction
aspell list < file.txt         # output only misspelled words
aspell -l en list < file.txt   # specify language
```

**Check a file and get only the bad words:**

```bash
cat essay.txt | aspell list | sort | uniq
```

### /usr/share/dict/words

The system word list — useful in scripts:

```bash
# Check if a word is valid
grep -w "^hello$" /usr/share/dict/words && echo "valid" || echo "invalid"

# Count total words in the dictionary
wc -l /usr/share/dict/words
```

---

## Automating Interactive Input

### echo / printf piping

The simplest approach — pipe answers directly:

```bash
echo "y" | apt-get install package    # auto-answer yes
echo -e "user\npassword\n" | ftp host # automate login prompts
```

### yes

Continuously outputs "y" (or any string) — useful for confirming prompts:

```bash
yes | apt-get install package         # answer yes to everything
yes "no" | command                    # answer "no" to everything
yes "" | command                      # send empty lines
```

### Here-doc for multi-line input

```bash
command << EOF
answer1
answer2
answer3
EOF
```

Real example — automate an FTP session:

```bash
ftp -n host << EOF
user username password
cd /upload
put file.txt
bye
EOF
```

### expect

When timing matters — `expect` waits for specific output before sending input:

```bash
expect << EOF
spawn ssh user@host
expect "password:"
send "mypassword\r"
expect "$ "
send "ls -la\r"
expect "$ "
EOF
```

Install: `apt install expect`

---

## Parallel Processes

### Background with &

Run a command in the background and continue:

```bash
command &
pid=$!          # $! holds the PID of the last background process
```

**Wait for all background jobs:**

```bash
process1 &
process2 &
process3 &
wait            # blocks until all background jobs finish
```

### xargs -P

Easiest way to parallelize a loop:

```bash
# Process 4 files at a time
find . -name "*.png" | xargs -P 4 -I {} convert {} {}.jpg

# Run 8 parallel jobs
cat urls.txt | xargs -P 8 -I {} curl -O {}
```

### GNU parallel

More powerful — handles progress, logging, job control:

```bash
parallel gzip ::: *.log                    # compress all .log files in parallel
parallel -j 4 wget {} ::: url1 url2 url3   # 4 parallel downloads
parallel "echo processing {}; sleep 1" ::: a b c d
```

**From a file:**

```bash
parallel -a urls.txt wget {}
```

**With progress bar:**

```bash
parallel --progress gzip ::: *.log
```

Install: `apt install parallel`

### Timing comparison

```bash
# Sequential
time for f in *.log; do gzip "$f"; done

# Parallel (4 jobs)
time find . -name "*.log" | xargs -P 4 gzip
```

The parallel version is typically `N` times faster where `N` is your CPU core count.

---

## 📚 References

<div class="references">
<ul>
  <li><a href="https://www.packtpub.com/product/linux-shell-scripting-cookbook/9781785881985" target="_blank">Linux Shell Scripting Cookbook — Packt</a></li>
  <li><a href="https://www.gnu.org/software/bash/manual/" target="_blank">GNU Bash Manual</a></li>
  <li><a href="https://www.gnu.org/software/parallel/" target="_blank">GNU Parallel</a></li>
</ul>
</div>

---

##  You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer )
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:**  [nonlouy](https://tryhackme.com/p/nonlouy)
