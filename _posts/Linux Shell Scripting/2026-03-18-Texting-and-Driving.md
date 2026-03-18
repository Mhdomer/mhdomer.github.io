---
layout: post
title: Chapter 4 Texting and Driving
date: 2026-03-18T10:00:00
categories:
  - Linux Shell Scripting
tags:
  - linuxcli
  - linux
  - bash
  - scripting
  - regex
  - awk
  - sed
  - grep
author: muhammed
description: Chapter 4 of Linux Shell Scripting Cookbook — text processing with grep, sed, awk, cut, and regular expressions
toc: true
pin: false
math: false
mermaid: false
image: https://media.licdn.com/dms/image/v2/D4D12AQG85GEyaMcdzg/article-cover_image-shrink_720_1280/article-cover_image-shrink_720_1280/0/1715198365363?e=2147483647&v=beta&t=zSE3QFqa_pLtDmQbUv_4HvOAvufhMvrOVzRy6lWimHE
Link: "[[Shell Scripting Notes]]"
---

# Chapter Overview

This chapter is the core of shell text processing — `grep`, `sed`, `awk`, `cut`, and regular expressions. These four tools together can parse, transform, extract, and reformat almost any structured text. Every sysadmin, developer, and security engineer uses them daily.

---

## Regular Expressions

Regular expressions (regex) are patterns used to match text. Before diving into the tools, you need to know the syntax.

### Basic (BRE) vs Extended (ERE)

Most tools support both. ERE is generally cleaner — use `grep -E`, `sed -E`, or `awk` (which uses ERE by default).

### Character classes

```
.        any single character (except newline)
[abc]    a, b, or c
[^abc]   anything NOT a, b, or c
[a-z]    lowercase letters
[0-9]    digits
[a-zA-Z0-9]  alphanumeric
```

### Anchors

```
^        start of line
$        end of line
\b       word boundary
\B       not a word boundary
```

### Quantifiers

```
*        0 or more
+        1 or more (ERE)
?        0 or 1 (ERE)
{n}      exactly n times (ERE)
{n,}     n or more (ERE)
{n,m}    between n and m (ERE)
```

### Special sequences

```
\d       digit (same as [0-9]) — works in some tools
\w       word character [a-zA-Z0-9_]
\s       whitespace
\D \W \S negations of above
```

### Groups and alternation

```
(abc)    group (ERE)
a|b      a or b (ERE)
```

### Examples

```bash
^ERROR           # lines starting with ERROR
\.log$           # lines ending with .log
[0-9]{1,3}       # 1 to 3 digits
\b\w+@\w+\.\w+   # rough email pattern
https?://        # http:// or https://
```

---

## Searching with grep

`grep` searches for patterns in files or stdin.

```bash
grep "pattern" file.txt              # basic search
grep -i "pattern" file.txt           # case-insensitive
grep -v "pattern" file.txt           # invert — lines NOT matching
grep -n "pattern" file.txt           # show line numbers
grep -c "pattern" file.txt           # count matching lines
grep -l "pattern" *.txt              # list files that match
grep -L "pattern" *.txt              # list files that DON'T match
grep -r "pattern" /path/             # recursive search
grep -w "word" file.txt              # whole word match only
grep -x "exact line" file.txt        # whole line match
grep -A 3 "pattern" file.txt         # 3 lines after match
grep -B 3 "pattern" file.txt         # 3 lines before match
grep -C 3 "pattern" file.txt         # 3 lines before and after
```

### Extended regex (-E)

```bash
grep -E "error|warn|fatal" log.txt          # OR
grep -E "^[0-9]{4}-[0-9]{2}-[0-9]{2}" log  # date pattern
grep -E "\b[A-Z]{2,}\b" file.txt            # all-caps words
```

### Fixed string (-F) — no regex, faster for literal searches

```bash
grep -F "192.168.1.1" access.log     # literal IP, no regex overhead
```

### Useful combos

```bash
# Count errors per log file
grep -rc "ERROR" /var/log/

# Find files containing a pattern, then open them
grep -rl "TODO" . | xargs vim

# Search only specific file types
grep -r "pattern" . --include="*.py"
grep -r "pattern" . --exclude="*.log"
```

---

## Cutting Columns with cut

`cut` extracts columns or fields from each line. Fast and simple for structured text.

### By character position

```bash
cut -c 1-5 file.txt          # characters 1 to 5
cut -c 1,3,5 file.txt        # characters 1, 3, and 5
cut -c 10- file.txt          # from character 10 to end
cut -c -20 file.txt          # first 20 characters
```

### By field (delimiter-separated)

```bash
cut -d ':' -f 1 /etc/passwd        # first field (username)
cut -d ':' -f 1,3 /etc/passwd      # fields 1 and 3
cut -d ':' -f 1-3 /etc/passwd      # fields 1 through 3
cut -d ',' -f 2 data.csv           # second column of CSV
cut -d $'\t' -f 3 file.tsv         # third column of TSV
```

**Extract IPs from an access log:**

```bash
cut -d ' ' -f 1 access.log | sort | uniq -c | sort -rn | head
```

**Limitation:** `cut` can't handle multiple consecutive delimiters as one. For that, use `awk`.

---

## Text Replacement with sed

`sed` (stream editor) applies edits to each line of input without opening a file interactively.

### Substitution — the most used command

```bash
sed 's/old/new/' file.txt           # replace first match per line
sed 's/old/new/g' file.txt          # replace all matches per line (global)
sed 's/old/new/i' file.txt          # case-insensitive
sed 's/old/new/2' file.txt          # replace only 2nd occurrence
sed 's/old/new/gi' file.txt         # global + case-insensitive
```

**Edit file in-place:**

```bash
sed -i 's/old/new/g' file.txt       # modify file directly
sed -i.bak 's/old/new/g' file.txt   # in-place with backup (.bak)
```

### Address ranges (which lines to act on)

```bash
sed '5s/old/new/' file.txt          # only line 5
sed '5,10s/old/new/g' file.txt      # lines 5 to 10
sed '/pattern/s/old/new/g' file.txt # lines matching a pattern
sed '5,/end/s/old/new/g' file.txt   # from line 5 to first match of "end"
```

### Delete lines

```bash
sed '5d' file.txt                   # delete line 5
sed '5,10d' file.txt                # delete lines 5-10
sed '/pattern/d' file.txt           # delete lines matching pattern
sed '/^$/d' file.txt                # delete blank lines
sed '/^[[:space:]]*$/d' file.txt    # delete whitespace-only lines
```

### Print specific lines

```bash
sed -n '5p' file.txt                # print only line 5
sed -n '5,10p' file.txt             # print lines 5-10
sed -n '/start/,/end/p' file.txt    # print between patterns
```

### Append, insert, change

```bash
sed '5a\new line after' file.txt    # append after line 5
sed '5i\new line before' file.txt   # insert before line 5
sed '5c\replacement line' file.txt  # replace line 5 entirely
```

### Extended regex

```bash
sed -E 's/[0-9]+/NUM/g' file.txt    # replace all numbers
sed -E 's/(error|warn)/[\1]/gi'     # wrap matched word in brackets
```

**Backreferences** — reference captured groups with `\1`, `\2`:

```bash
sed -E 's/([0-9]{4})-([0-9]{2})-([0-9]{2})/\3\/\2\/\1/' file.txt
# 2026-03-18 → 18/03/2026
```

---

## Advanced Text Processing with awk

`awk` is a full programming language for text processing. It processes each line as a record and splits it into fields automatically.

### Basic structure

```bash
awk 'pattern { action }' file.txt
```

- `pattern` — which lines to act on (omit to match all)
- `action` — what to do (omit to print the line)

### Built-in variables

| Variable | Meaning |
|---|---|
| `$0` | Entire current line |
| `$1`, `$2` | First, second field |
| `NF` | Number of fields in current line |
| `NR` | Current line number (record number) |
| `FS` | Field separator (default: whitespace) |
| `OFS` | Output field separator |
| `RS` | Record separator (default: newline) |
| `ORS` | Output record separator |

### Examples

```bash
awk '{print $1}' file.txt              # print first field of every line
awk '{print $NF}' file.txt             # print last field
awk '{print NR, $0}' file.txt          # number every line
awk 'NR==5' file.txt                   # print only line 5
awk 'NR>=5 && NR<=10' file.txt         # print lines 5-10
awk '/pattern/' file.txt               # print lines matching pattern
awk '!/pattern/' file.txt              # print lines NOT matching
awk 'NF > 3' file.txt                  # lines with more than 3 fields
```

### Custom delimiter

```bash
awk -F ':' '{print $1}' /etc/passwd        # use : as separator
awk -F ',' '{print $2, $4}' data.csv       # CSV — print columns 2 and 4
awk -F '\t' '{print $3}' file.tsv          # tab-separated
```

### Calculations

```bash
awk '{sum += $1} END {print sum}' nums.txt           # sum a column
awk '{sum += $1} END {print sum/NR}' nums.txt        # average
awk 'BEGIN{max=0} $1>max{max=$1} END{print max}' f   # max value
```

### BEGIN and END blocks

```bash
awk 'BEGIN {print "Start"} {print $0} END {print "Done"}' file.txt
```

`BEGIN` runs once before any lines are processed. `END` runs once after all lines.

### Conditionals and loops

```bash
awk '{
  if ($3 > 100) {
    print $1, "high"
  } else {
    print $1, "low"
  }
}' file.txt
```

### Output formatting with printf

```bash
awk '{printf "%-20s %5d\n", $1, $2}' file.txt
```

### Practical examples

```bash
# Sum the size column in ls -l output
ls -l | awk '{sum += $5} END {print sum " bytes"}'

# Print lines where field 3 is greater than 50
awk -F ',' '$3 > 50' data.csv

# Count occurrences of each value in column 2
awk -F ',' '{count[$2]++} END {for (k in count) print k, count[k]}' data.csv

# Print duplicate lines
awk 'seen[$0]++' file.txt

# Remove duplicate lines (keep first occurrence)
awk '!seen[$0]++' file.txt
```

---

## Word Frequency in a File

Count how often each word appears:

```bash
tr -s '[:space:]' '\n' < file.txt | tr '[:upper:]' '[:lower:]' | \
  sort | uniq -c | sort -rn | head -20
```

Step by step:
1. `tr -s '[:space:]' '\n'` — replace all whitespace with newlines (one word per line)
2. `tr '[:upper:]' '[:lower:]'` — lowercase everything
3. `sort` — sort alphabetically (required for uniq)
4. `uniq -c` — count consecutive duplicates
5. `sort -rn` — sort by count descending
6. `head -20` — top 20

**With awk (handles punctuation better):**

```bash
awk '{
  gsub(/[^a-zA-Z]/, " ")   # replace non-letters with spaces
  for (i=1; i<=NF; i++) {
    word = tolower($i)
    if (length(word) > 0) freq[word]++
  }
} END {
  for (w in freq) print freq[w], w
}' file.txt | sort -rn | head -20
```

---

## Compressing and Decompressing JavaScript

Minifying JS removes whitespace and comments to reduce file size for production.

### Using uglifyjs

```bash
npm install -g uglify-js

uglifyjs script.js -o script.min.js                      # minify
uglifyjs script.js -o script.min.js --compress --mangle  # compress + mangle names
uglifyjs script.min.js --beautify -o script.readable.js  # decompress/beautify
```

### Quick sed minification (basic — not production-grade)

```bash
sed 's/\/\/.*$//g' script.js |   # remove single-line comments
sed 's/[[:space:]]\+/ /g' |      # squeeze whitespace
sed 's/^ //; s/ $//'             # trim leading/trailing spaces
```

### Using python for pretty-printing JSON (related)

```bash
cat data.json | python3 -m json.tool          # pretty print
cat data.json | python3 -m json.tool --compact # compact/minify
```

---

## Merging Files as Columns

### paste

`paste` merges files side by side, line by line:

```bash
paste file1.txt file2.txt               # tab-separated by default
paste -d ',' file1.txt file2.txt        # comma-separated
paste -d ':' file1.txt file2.txt file3.txt  # three files, colon-separated
paste -s file.txt                       # serial — merge lines of ONE file into one line
paste -s -d ',' file.txt               # comma-separated single line
```

**Create a CSV from separate column files:**

```bash
paste -d ',' names.txt ages.txt emails.txt > people.csv
```

**Combine with process substitution:**

```bash
paste <(cut -d: -f1 /etc/passwd) <(cut -d: -f7 /etc/passwd)
# username   shell (side by side)
```

### column

Format output into aligned columns:

```bash
column -t file.txt                      # auto-align columns
paste file1.txt file2.txt | column -t   # merge then align
cat /etc/passwd | column -t -s ':'      # align by delimiter
```

---

## Printing the nth Word or Column

### awk — most reliable

```bash
awk '{print $3}' file.txt          # 3rd field of every line
awk 'NR==5 {print $3}' file.txt    # 3rd field of line 5 only
awk -F ',' '{print $2}' file.csv   # 2nd column of CSV
```

### cut

```bash
cut -d ' ' -f 3 file.txt           # 3rd space-separated field
cut -d ',' -f 2 file.csv           # 2nd CSV column
```

**Note:** `cut` treats consecutive delimiters as separate fields. `awk` collapses whitespace.

### From a single line (not a file)

```bash
echo "one two three four" | awk '{print $3}'     # three
echo "a:b:c:d" | cut -d ':' -f 2                 # b
```

### Last field

```bash
awk '{print $NF}' file.txt              # last field (NF = number of fields)
rev file.txt | cut -d ' ' -f 1 | rev   # reverse trick
```

---

## Printing Text Between Line Numbers or Patterns

### By line numbers

```bash
sed -n '10,20p' file.txt              # lines 10 to 20
awk 'NR>=10 && NR<=20' file.txt       # same with awk
head -20 file.txt | tail -11          # lines 10-20 (head then tail)
```

### Between patterns

```bash
sed -n '/START/,/END/p' file.txt      # from START to END (inclusive)
awk '/START/,/END/' file.txt          # same with awk

# Exclusive (don't include the pattern lines themselves)
sed -n '/START/,/END/{/START/!{/END/!p}}' file.txt
awk '/START/{f=1; next} /END/{f=0} f' file.txt
```

### After a pattern to end of file

```bash
sed -n '/pattern/,$p' file.txt        # from pattern to EOF
awk '/pattern/,0' file.txt            # awk equivalent
```

### Between Nth and Mth occurrence of a pattern

```bash
awk '/SECTION/{count++} count==2,count==3' file.txt
```

---

## Printing Lines in Reverse Order

### tac

`tac` is `cat` backwards — reverses line order:

```bash
tac file.txt                          # reverse all lines
tac file.txt | head -5                # last 5 lines in forward order
```

### awk

```bash
awk '{lines[NR]=$0} END {for(i=NR;i>=1;i--) print lines[i]}' file.txt
```

### Reverse character order within a line

```bash
rev file.txt                          # reverse characters on each line
echo "hello" | rev                    # olleh
```

**Combine for full reversal (lines and characters):**

```bash
tac file.txt | rev
```

---

## Parsing Email Addresses and URLs

### Extract emails with grep

```bash
grep -Eo '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' file.txt
```

`-E` = extended regex, `-o` = print only the matched part (not the full line).

### Extract URLs with grep

```bash
grep -Eo 'https?://[^[:space:]"]+' file.txt
grep -Eo '(http|https|ftp)://[^[:space:]]*' file.txt
```

### Extract with awk

```bash
# Extract all emails
awk '{
  while (match($0, /[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/)) {
    print substr($0, RSTART, RLENGTH)
    $0 = substr($0, RSTART + RLENGTH)
  }
}' file.txt
```

### Validate an email (basic)

```bash
echo "user@example.com" | grep -Eq '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$' \
  && echo "valid" || echo "invalid"
```

---

## Removing Sentences Containing a Word

### sed

```bash
sed '/keyword/d' file.txt                    # delete lines containing keyword
sed '/error/Id' file.txt                     # case-insensitive delete
sed '/^.*keyword.*$/d' file.txt              # explicit full-line match
```

### grep -v (inverse)

```bash
grep -v "keyword" file.txt                   # print all lines EXCEPT those with keyword
grep -vi "keyword" file.txt                  # case-insensitive
```

### awk

```bash
awk '!/keyword/' file.txt                    # print lines not matching
awk 'tolower($0) !~ /keyword/' file.txt      # case-insensitive
```

**Remove multiple keywords:**

```bash
grep -vE "error|warning|debug" file.txt
sed '/error\|warning\|debug/d' file.txt
```

**Edit in-place:**

```bash
sed -i '/keyword/d' file.txt
```

---

## Replacing a Pattern in All Files in a Directory

### grep + sed combo

```bash
grep -rl "old_pattern" /path/ | xargs sed -i 's/old_pattern/new_text/g'
```

`-r` = recursive, `-l` = list files only (not matches), then pipe to `xargs sed -i`.

### find + sed

```bash
find /path -type f -name "*.txt" -exec sed -i 's/old/new/g' {} \;
```

**Safer — preview first:**

```bash
grep -rl "old" . | xargs grep -l "old"        # confirm which files
grep -rl "old" . | xargs sed -n 's/old/new/gp' # dry run (print changes, don't save)
grep -rl "old" . | xargs sed -i.bak 's/old/new/g'  # apply with backup
```

### Specific file types only

```bash
find . -name "*.py" | xargs sed -i 's/import old/import new/g'
find . -name "*.conf" | xargs sed -i 's/localhost/production.host.com/g'
```

---

## Text Slicing and Parameter Operations

Bash parameter expansion lets you slice and manipulate strings without spawning subshells.

### Length

```bash
str="hello world"
echo ${#str}            # 11
```

### Substring extraction

```bash
echo ${str:6}           # world (from position 6)
echo ${str:0:5}         # hello (positions 0-4)
echo ${str:(-5)}        # world (last 5 characters)
```

### Remove prefix/suffix

```bash
filename="report_2026.txt"

echo ${filename#report_}       # 2026.txt  (remove shortest prefix match)
echo ${filename##*_}           # 2026.txt  (remove longest prefix match)
echo ${filename%.txt}          # report_2026 (remove shortest suffix)
echo ${filename%%.*}           # report_2026 (remove longest suffix)
```

### Find and replace

```bash
str="hello hello world"
echo ${str/hello/hi}           # hi hello world  (first match)
echo ${str//hello/hi}          # hi hi world     (all matches)
echo ${str/#hello/hi}          # hi hello world  (prefix match only)
echo ${str/%world/earth}       # hello hello earth (suffix match only)
```

### Case conversion (bash 4+)

```bash
str="Hello World"
echo ${str^^}           # HELLO WORLD (all uppercase)
echo ${str,,}           # hello world (all lowercase)
echo ${str^}            # Hello World (capitalize first char)
echo ${str,}            # hello World (lowercase first char)
```

### Default values

```bash
echo ${var:-default}    # use "default" if var is unset or empty
echo ${var:=default}    # set var to "default" if unset, then use it
echo ${var:+other}      # use "other" if var IS set (opposite)
echo ${var:?error msg}  # exit with error if var is unset
```

### Practical example — batch rename with slicing

```bash
for f in IMG_*.jpg; do
  date_part="${f:4:8}"        # extract 8 chars starting at position 4
  mv "$f" "photo_${date_part}.jpg"
done
```

---

## 📚 References

<div class="references">
<ul>
  <li><a href="https://www.packtpub.com/product/linux-shell-scripting-cookbook/9781785881985" target="_blank">Linux Shell Scripting Cookbook — Packt</a></li>
  <li><a href="https://www.gnu.org/software/gawk/manual/" target="_blank">GNU awk Manual</a></li>
  <li><a href="https://www.gnu.org/software/sed/manual/" target="_blank">GNU sed Manual</a></li>
  <li><a href="https://www.regular-expressions.info/" target="_blank">Regular-Expressions.info</a></li>
</ul>
</div>

---

##  You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer )
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:**  [nonlouy](https://tryhackme.com/p/nonlouy)
