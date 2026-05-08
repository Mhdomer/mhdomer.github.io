---
layout: post
title: Chapter 1 Shell Something Out
date: 2026-02-25T10:00:00
categories:
  - Linux Shell Scripting
tags:
  - linuxcli
  - linux
  - bash
  - scripting
author: muhammed
description: First Chapter in bash scripting for Linux
toc: true
pin: false
math: false
mermaid: false
image: https://media.licdn.com/dms/image/v2/D4D12AQG85GEyaMcdzg/article-cover_image-shrink_720_1280/article-cover_image-shrink_720_1280/0/1715198365363?e=2147483647&v=beta&t=zSE3QFqa_pLtDmQbUv_4HvOAvufhMvrOVzRy6lWimHE
Link: "[[Shell Scripting Notes]]"
---

# Chapter Overview 


In this chapter, I will cover

- Printing in the terminal
    
- Playing with variables and environment variables
    
- Function to prepend to environment variables
    
- Math with the shell
    
- Playing with file descriptors and redirection
    
- Arrays and associative array
    
- Visiting aliases
    
- Grabbing information about the terminal
    
- Getting and setting dates and delays
    
- Debugging the script
    
- Functions and arguments
    
- Reading output of a sequence of commands in a variable
    
- Reading n characters without pressing the return key
    
- Running a command until it succeeds
    
- Field separators and iterators
    
- Comparisons and tests



---

## Printing in the Terminal

### echo

```bash
echo "Hello World"
echo -n "No newline"          # -n suppresses the trailing newline
echo -e "Tab\there\nNewline"  # -e enables escape sequences
```

Double quotes expand variables. Single quotes print literally.

```bash
name="Omar"
echo "Hello $name"   # Hello Omar
echo 'Hello $name'   # Hello $name
```

### printf

More control than echo — borrowed from C. Useful for aligned output.

```bash
printf "%-10s %5d\n" "Item" 42   # left-align string, right-align number
printf "Pi is %.2f\n" 3.14159    # 2 decimal places
```

`%s` = string, `%d` = integer, `%f` = float. The number before the letter sets the field width.

---

## Variables and Environment Variables

Variables are untyped in bash — everything is a string unless told otherwise. No spaces around `=`.

```bash
name="Omar"
echo $name        # access with $
echo ${name}      # braces — safer, avoids ambiguity
echo "${name}!"   # braces needed: $name! would fail
```

**Environment variables** are exported to child processes:

```bash
export MY_VAR="value"
printenv MY_VAR       # print one var
env                   # list all
```

Common ones to know:

| Variable | Meaning |
|---|---|
| `$HOME` | Home directory |
| `$PATH` | Directories bash searches for commands |
| `$USER` | Current user |
| `$PWD` | Current working directory |
| `$?` | Exit status of the last command |

---

## Prepending to Environment Variables

Add to `PATH` without wiping it:

```bash
export PATH="/new/bin:$PATH"
```

A safer function that avoids adding duplicates:

```bash
prepend_path() {
  case ":$PATH:" in
    *":$1:"*) ;;          # already present, skip
    *) PATH="$1:$PATH" ;;
  esac
  export PATH
}

prepend_path "/opt/custom/bin"
```

Put it in `~/.bashrc` and call it for any directory you want at the front.

---

## Math with the Shell

Bash only does integer math natively — use `$(( ))`:

```bash
echo $((10 + 5))     # 15
echo $((2 ** 8))     # 256 — exponentiation
echo $((17 % 3))     # 2  — modulo
result=$((100 / 4))
```

For decimals, pipe to `bc`:

```bash
echo "scale=2; 10 / 3" | bc      # 3.33
echo "scale=4; sqrt(2)" | bc -l  # 1.4142 — -l loads math library
```

---

## File Descriptors and Redirection

Every process starts with 3 file descriptors:

| FD | Name | Default |
|---|---|---|
| `0` | stdin | keyboard |
| `1` | stdout | terminal |
| `2` | stderr | terminal |

```bash
command > file.txt        # redirect stdout (overwrite)
command >> file.txt       # redirect stdout (append)
command 2> errors.txt     # redirect stderr
command &> all.txt        # redirect both stdout and stderr
command > /dev/null 2>&1  # discard all output
```

Read from a file instead of keyboard:

```bash
wc -l < file.txt
```

Custom file descriptors:

```bash
exec 3> custom.txt   # open fd 3 for writing
echo "data" >&3      # write to it
exec 3>&-            # close fd 3
```

---

## Arrays and Associative Arrays

**Indexed arrays:**

```bash
fruits=("apple" "banana" "cherry")

echo ${fruits[0]}     # apple
echo ${fruits[@]}     # all elements
echo ${#fruits[@]}    # length — 3
fruits+=("date")      # append

for f in "${fruits[@]}"; do
  echo "$f"
done
```

**Associative arrays** (bash 4+) — key-value pairs:

```bash
declare -A user
user[name]="Omar"
user[role]="admin"

echo ${user[name]}      # Omar
echo ${!user[@]}        # all keys
echo ${user[@]}         # all values

for key in "${!user[@]}"; do
  echo "$key = ${user[$key]}"
done
```

---

## Aliases

Shortcuts for commands — define in `~/.bashrc`:

```bash
alias ll='ls -lah'
alias gs='git status'
alias ..='cd ..'
alias grep='grep --color=auto'

unalias ll   # remove one
alias        # list all
```

Aliases don't take arguments. Use a function instead:

```bash
mkcd() {
  mkdir -p "$1" && cd "$1"
}
```

---

## Terminal Information

```bash
tput cols     # columns (width)
tput lines    # rows (height)
tput colors   # number of colors supported
echo $TERM    # terminal type e.g. xterm-256color
```

Center text dynamically based on terminal width:

```bash
width=$(tput cols)
text="Hello"
pad=$(( (width - ${#text}) / 2 ))
printf "%${pad}s%s\n" "" "$text"
```

Set colors:

```bash
tput setaf 1; echo "Red text"; tput sgr0   # sgr0 resets
```

---

## Dates and Delays

```bash
date                      # full date and time
date +"%Y-%m-%d"          # 2026-02-25
date +"%H:%M:%S"          # 14:30:00
date +%s                  # Unix timestamp
date -d @1706140800       # convert timestamp back to readable
```

Delays:

```bash
sleep 1      # 1 second
sleep 0.5    # half second
sleep 2m     # 2 minutes
```

Measure how long something takes:

```bash
start=$(date +%s)
your_command
echo "Elapsed: $(( $(date +%s) - start ))s"
```

Or just:

```bash
time your_command
```

---

## Debugging Scripts

Trace mode — prints every command before it runs:

```bash
bash -x script.sh
```

Toggle inside the script:

```bash
set -x   # start tracing
# commands
set +x   # stop tracing
```

Best practice — put this at the top of every script:

```bash
#!/bin/bash
set -euo pipefail
```

| Flag | Effect |
|---|---|
| `-e` | Exit immediately on any error |
| `-u` | Treat unset variables as errors |
| `-o pipefail` | Pipe fails if any command in it fails, not just the last |

---

## Functions and Arguments

```bash
greet() {
  echo "Hello, $1"
  echo "Got $# arguments"
}

greet "Omar"
```

Special variables inside functions:

| Variable | Meaning |
|---|---|
| `$1`, `$2` | Positional arguments |
| `$@` | All arguments as separate strings |
| `$#` | Number of arguments |
| `$0` | Script name |

Functions return exit codes (0–255), not strings. To return a value, echo it and capture:

```bash
get_date() {
  echo "$(date +%Y-%m-%d)"
}

today=$(get_date)
```

---

## Command Substitution

Capture the output of a command into a variable:

```bash
files=$(ls /etc)
today=$(date +"%Y-%m-%d")
line_count=$(wc -l < file.txt)
```

Use `$()` not backticks — backticks can't be nested cleanly:

```bash
# Nested — works fine with $()
result=$(wc -l < $(find /var/log -name "*.log" | head -1))
```

---

## Reading Input

```bash
read -p "Enter your name: " name
echo "Hello, $name"
```

Read n characters without pressing Enter:

```bash
read -n 1 -p "Press any key..." key
```

Silent input (passwords):

```bash
read -s -p "Password: " pass
echo ""
```

With a timeout:

```bash
read -t 5 -p "Answer in 5s: " answer || echo "Timed out"
```

---

## Running a Command Until It Succeeds

```bash
until command; do
  sleep 1
done
```

Wait for a server to be ready:

```bash
until curl -s http://localhost:3000/health > /dev/null; do
  echo "Waiting for server..."
  sleep 2
done
echo "Server is up"
```

With a max retry limit:

```bash
retries=0
until command || [[ $retries -eq 5 ]]; do
  retries=$((retries + 1))
  sleep 1
done
```

---

## Field Separators and Iterators

**IFS** (Internal Field Separator) controls how bash splits strings. Default: space, tab, newline.

Split on a custom delimiter:

```bash
IFS=',' read -ra parts <<< "one,two,three"
for part in "${parts[@]}"; do
  echo "$part"
done
```

Always restore IFS after changing it:

```bash
old_IFS=$IFS
IFS=':'
# work with colon-separated data
IFS=$old_IFS
```

Parse `/etc/passwd` line by line:

```bash
while IFS=':' read -r user _ uid _ _ home shell; do
  echo "User: $user | Shell: $shell"
done < /etc/passwd
```

---

## Comparisons and Tests

Use `[[ ]]` over `[ ]` — bash built-in, safer, supports regex.

**String tests:**

```bash
[[ "$a" == "$b" ]]       # equal
[[ "$a" != "$b" ]]       # not equal
[[ -z "$a" ]]            # empty string
[[ -n "$a" ]]            # non-empty
[[ "$a" =~ ^[0-9]+$ ]]   # regex match
```

**Integer tests:**

```bash
[[ $a -eq $b ]]   # equal
[[ $a -ne $b ]]   # not equal
[[ $a -lt $b ]]   # less than
[[ $a -gt $b ]]   # greater than
[[ $a -le $b ]]   # less or equal
[[ $a -ge $b ]]   # greater or equal
```

**File tests:**

```bash
[[ -e "$f" ]]   # exists
[[ -f "$f" ]]   # is a regular file
[[ -d "$f" ]]   # is a directory
[[ -r "$f" ]]   # readable
[[ -w "$f" ]]   # writable
[[ -x "$f" ]]   # executable
[[ -s "$f" ]]   # non-empty
```

**Combining:**

```bash
[[ -f "$f" && -r "$f" ]]   # AND
[[ -z "$a" || -z "$b" ]]   # OR
[[ ! -d "$dir" ]]           # NOT
```
















---

## 📚 References

<div class="references">
<ul>
  <li><a href="#" target="_blank">Official Documentation</a></li>
  <li><a href="#" target="_blank">Research Paper</a></li>
  <li><a href="#" target="_blank">Related Blog Post</a></li>
</ul>
</div>






---

##  You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer )
- **GitHub:** [Mhdomer](https://github.comMhdomer)  
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/) 
- **Tryhackme:**  [nonlouy](https://tryhackme.com/p/nonlouy)
