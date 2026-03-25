---
layout: post
title: Chapter 5 Tangled Web Not At All
date: 2026-03-25T10:00:00
categories:
  - Linux Shell Scripting
tags:
  - linuxcli
  - linux
  - bash
  - scripting
  - curl
  - wget
  - web
author: muhammed
description: Chapter 5 of Linux Shell Scripting Cookbook — web interaction from the command line using curl, wget, and shell scripting
toc: true
pin: false
math: false
mermaid: false
image: https://media.licdn.com/dms/image/v2/D4D12AQG85GEyaMcdzg/article-cover_image-shrink_720_1280/article-cover_image-shrink_720_1280/0/1715198365363?e=2147483647&v=beta&t=zSE3QFqa_pLtDmQbUv_4HvOAvufhMvrOVzRy6lWimHE
Link: "[[Shell Scripting Notes]]"
---

# Chapter Overview

This chapter covers interacting with the web from the command line — downloading files, scraping pages, making HTTP requests, parsing responses, and automating web tasks. `curl` and `wget` are the two main tools, and they're used in virtually every DevOps and security workflow.

---

## Downloading from a Web Page

### wget

`wget` is built for downloading — it handles retries, resuming, and recursive downloads automatically.

```bash
wget https://example.com/file.tar.gz           # download a file
wget -O output.tar.gz https://example.com/f    # save with a specific name
wget -c https://example.com/largefile.iso      # resume interrupted download
wget -q https://example.com/file              # quiet mode (no output)
wget -b https://example.com/file              # background download
wget --limit-rate=500k https://example.com/f  # limit speed to 500KB/s
```

**Download multiple files from a list:**

```bash
wget -i urls.txt                               # read URLs from a file
```

**Mirror an entire website:**

```bash
wget --mirror --convert-links --adjust-extension \
  --page-requisites --no-parent \
  https://example.com/
```

- `--mirror` = recursive + timestamps
- `--convert-links` = fix links for offline use
- `--page-requisites` = download CSS, images, JS
- `--no-parent` = don't go above the starting URL

**Recursive download with depth limit:**

```bash
wget -r -l 2 https://example.com/docs/        # 2 levels deep
```

### curl for downloading

```bash
curl -O https://example.com/file.tar.gz        # save with original name
curl -o output.tar.gz https://example.com/f    # save with custom name
curl -C - -O https://example.com/large.iso     # resume download
curl -L https://example.com/file              # follow redirects (-L is important)
```

---

## Downloading a Web Page as Plain Text

### wget

```bash
wget -q -O - https://example.com | cat         # dump raw HTML to stdout
```

### curl

```bash
curl -s https://example.com                    # -s = silent (no progress bar)
curl -s https://example.com | grep "<title>"   # extract title
```

### Convert HTML to plain text

**Using lynx:**

```bash
lynx -dump https://example.com                 # render page as text (no HTML tags)
lynx -dump -nolist https://example.com         # suppress link list at bottom
```

**Using w3m:**

```bash
w3m -dump https://example.com                  # text rendering
```

**Using html2text:**

```bash
curl -s https://example.com | html2text        # convert HTML to markdown-like text
```

**Strip HTML tags with sed (quick and dirty):**

```bash
curl -s https://example.com | sed 's/<[^>]*>//g' | sed '/^[[:space:]]*$/d'
```

---

## A Primer on cURL

`curl` (Client URL) is the Swiss Army knife of HTTP requests. Essential for testing APIs, web scraping, and automation.

### Basic requests

```bash
curl https://example.com                       # GET request
curl -s https://example.com                    # silent (no progress)
curl -v https://example.com                    # verbose (show headers)
curl -I https://example.com                    # HEAD request (headers only)
curl -L https://example.com                    # follow redirects
```

### Request methods

```bash
curl -X GET https://api.example.com/users
curl -X POST https://api.example.com/users
curl -X PUT https://api.example.com/users/1
curl -X DELETE https://api.example.com/users/1
curl -X PATCH https://api.example.com/users/1
```

### Sending data (POST)

```bash
# Form data
curl -X POST -d "user=omar&pass=secret" https://example.com/login

# JSON body
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"username": "omar", "password": "secret"}' \
  https://api.example.com/login

# JSON from a file
curl -X POST \
  -H "Content-Type: application/json" \
  -d @payload.json \
  https://api.example.com/data
```

### Headers

```bash
curl -H "Authorization: Bearer TOKEN" https://api.example.com/data
curl -H "Accept: application/json" https://api.example.com/
curl -H "X-Custom-Header: value" https://api.example.com/
```

### Authentication

```bash
curl -u username:password https://api.example.com/     # Basic auth
curl -H "Authorization: Bearer TOKEN" https://api.com  # Bearer token
curl --digest -u user:pass https://example.com          # Digest auth
```

### Cookies

```bash
curl -c cookies.txt https://example.com          # save cookies to file
curl -b cookies.txt https://example.com          # send cookies from file
curl -b "session=abc123" https://example.com     # send cookie directly
```

### Response handling

```bash
curl -o output.html https://example.com          # save body to file
curl -D headers.txt https://example.com          # save headers to file
curl -w "%{http_code}\n" -s -o /dev/null https://example.com  # print status code only
curl -w "%{time_total}\n" -s -o /dev/null https://example.com # print response time
```

### TLS/SSL

```bash
curl -k https://self-signed.example.com          # skip certificate verification
curl --cacert ca.pem https://example.com         # use custom CA
curl --cert client.pem --key key.pem https://   # client certificate
```

### Useful write-out variables

```bash
curl -w "Status: %{http_code}\nTime: %{time_total}s\nSize: %{size_download} bytes\n" \
  -s -o /dev/null https://example.com
```

---

## Accessing Gmail from the Command Line

**Note:** Gmail now requires OAuth2 — plain password access is disabled. These approaches use `mutt` or `msmtp` with app passwords or OAuth2.

### mutt with Gmail (app password)

Configure `~/.muttrc`:

```bash
set imap_user = "you@gmail.com"
set imap_pass = "your-app-password"
set folder = "imaps://imap.gmail.com/"
set spoolfile = "+INBOX"
set ssl_force_tls = yes
```

```bash
mutt -f imaps://imap.gmail.com/INBOX    # open Gmail inbox
```

### Send email from command line with msmtp

Configure `~/.msmtprc`:

```
account gmail
host smtp.gmail.com
port 587
auth on
tls on
user you@gmail.com
password your-app-password
```

```bash
echo "Message body" | msmtp -a gmail recipient@example.com
echo -e "Subject: Test\n\nHello" | msmtp recipient@example.com
```

### Send with curl (SMTP)

```bash
curl --ssl-reqd \
  --url 'smtps://smtp.gmail.com:465' \
  --user 'you@gmail.com:app-password' \
  --mail-from 'you@gmail.com' \
  --mail-rcpt 'to@example.com' \
  --upload-file email.txt
```

`email.txt` format:
```
From: you@gmail.com
To: to@example.com
Subject: Test

Message body here.
```

---

## Parsing Data from a Website

### grep for quick extraction

```bash
curl -s https://example.com | grep -oE 'href="[^"]*"'      # all links
curl -s https://example.com | grep -oE '<title>[^<]*</title>'  # page title
curl -s https://example.com | grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+'  # IPs
```

### sed for structured extraction

```bash
curl -s https://example.com | sed -n 's/.*<title>\(.*\)<\/title>.*/\1/p'
```

### awk for table data

```bash
curl -s https://example.com | \
  awk '/<table/,/<\/table/' | \
  awk '/<td/,/<\/td/' | \
  sed 's/<[^>]*>//g' | \
  sed '/^[[:space:]]*$/d'
```

### pup — HTML parser (cleaner approach)

```bash
curl -s https://example.com | pup 'a[href] attr{href}'    # extract all hrefs
curl -s https://example.com | pup 'title text{}'           # page title
curl -s https://example.com | pup 'h2 text{}'              # all h2 text
curl -s https://example.com | pup '.classname text{}'      # by CSS class
```

Install: `go install github.com/ericchiang/pup@latest`

### python (when shell isn't enough)

```bash
curl -s https://example.com | python3 -c "
import sys
from html.parser import HTMLParser

class LinkParser(HTMLParser):
    def handle_starttag(self, tag, attrs):
        if tag == 'a':
            for attr, val in attrs:
                if attr == 'href':
                    print(val)

LinkParser().feed(sys.stdin.read())
"
```

---

## Image Crawler and Downloader

### Extract all image URLs from a page

```bash
curl -s https://example.com | \
  grep -oE 'src="[^"]*\.(jpg|jpeg|png|gif|webp)"' | \
  sed 's/src="//;s/"//'
```

### Download all images from a page

```bash
# Extract and download
curl -s https://example.com | \
  grep -oE '(https?://[^"]*\.(jpg|jpeg|png|gif))' | \
  xargs -P 4 wget -q
```

### wget recursive image download

```bash
wget -r -A "*.jpg,*.png,*.gif" -nd -P ./images/ https://example.com/gallery/
```

- `-A` = accept only these extensions
- `-nd` = no directories (flat download)
- `-P` = save to `./images/`

### Full image crawler script

```bash
#!/bin/bash
url="$1"
output_dir="./downloaded_images"
mkdir -p "$output_dir"

echo "Crawling: $url"

curl -s "$url" | \
  grep -oE '(https?://[^"'\''<>[:space:]]*\.(jpg|jpeg|png|gif|webp))' | \
  sort -u | \
  while read img_url; do
    filename=$(basename "$img_url" | cut -d'?' -f1)
    echo "Downloading: $img_url"
    curl -s -L -o "$output_dir/$filename" "$img_url"
  done

echo "Done. Images saved to $output_dir"
```

Usage: `bash crawler.sh https://example.com/gallery`

---

## Web Photo Album Generator

Generate a simple HTML gallery from a directory of images:

```bash
#!/bin/bash
dir="${1:-.}"
output="album.html"

cat > "$output" << 'HEADER'
<!DOCTYPE html>
<html>
<head>
  <title>Photo Album</title>
  <style>
    body { font-family: sans-serif; background: #111; color: #fff; }
    .gallery { display: flex; flex-wrap: wrap; gap: 10px; padding: 20px; }
    .gallery img { width: 200px; height: 150px; object-fit: cover; border-radius: 4px; }
    .gallery a:hover img { opacity: 0.8; }
  </style>
</head>
<body>
<h1>Photo Album</h1>
<div class="gallery">
HEADER

find "$dir" -maxdepth 1 -type f \( -iname "*.jpg" -o -iname "*.png" -o -iname "*.jpeg" \) | sort | \
while read img; do
  filename=$(basename "$img")
  echo "  <a href=\"$filename\"><img src=\"$filename\" alt=\"$filename\"></a>"
done >> "$output"

cat >> "$output" << 'FOOTER'
</div>
</body>
</html>
FOOTER

echo "Album generated: $output"
echo "Images included: $(grep -c '<img' $output)"
```

Usage: `bash album.sh /path/to/photos`

---

## Twitter / X Command-Line Client

The official Twitter v1.1 API is now restricted. For modern usage, the approach is the Twitter v2 API with Bearer token auth.

### Basic tweet fetch with curl

```bash
BEARER_TOKEN="your_bearer_token_here"

# Get recent tweets from a user
curl -s \
  -H "Authorization: Bearer $BEARER_TOKEN" \
  "https://api.twitter.com/2/tweets/search/recent?query=from:username&max_results=10" | \
  python3 -m json.tool
```

### twurl (official Twitter curl wrapper)

```bash
gem install twurl
twurl authorize --consumer-key KEY --consumer-secret SECRET
twurl /1.1/statuses/home_timeline.json | python3 -m json.tool
```

### t (Ruby Twitter CLI)

```bash
gem install t
t authorize
t timeline                    # home timeline
t mentions                    # mentions
t search "keyword"            # search
t update "Hello from terminal!"  # post a tweet
```

---

## Creating a Define Utility

Build a command-line dictionary lookup using free web APIs.

### Using the Free Dictionary API

```bash
define() {
  word="$1"
  curl -s "https://api.dictionaryapi.dev/api/v2/entries/en/$word" | \
    python3 -c "
import sys, json
data = json.load(sys.stdin)
if isinstance(data, list):
    for entry in data:
        print(f'Word: {entry[\"word\"]}')
        for meaning in entry.get('meanings', []):
            print(f'  [{meaning[\"partOfSpeech\"]}]')
            for d in meaning.get('definitions', [])[:2]:
                print(f'    - {d[\"definition\"]}')
else:
    print(data.get('message', 'Not found'))
"
}

define "ephemeral"
```

### Simple version with grep/sed

```bash
define() {
  curl -s "https://api.dictionaryapi.dev/api/v2/entries/en/$1" | \
    grep -oP '"definition":"\K[^"]+' | \
    head -3 | \
    nl
}
```

Add either version to `~/.bashrc` for permanent use.

---

## Finding Broken Links in a Website

### wget spider mode

```bash
wget --spider -r -nd -nv --delete-after \
  -o wget_log.txt https://example.com

grep -i "broken\|404\|error" wget_log.txt
```

`--spider` = don't download, just check links.

### curl in a loop

```bash
#!/bin/bash
url="$1"

# Extract all links
links=$(curl -s "$url" | grep -oE 'href="(https?://[^"]*)"' | sed 's/href="//;s/"//')

while read -r link; do
  status=$(curl -s -o /dev/null -w "%{http_code}" -L --max-time 10 "$link")
  if [[ "$status" =~ ^[45] ]]; then
    echo "BROKEN [$status]: $link"
  else
    echo "OK [$status]: $link"
  fi
done <<< "$links"
```

### linkchecker (dedicated tool)

```bash
pip install linkchecker
linkchecker https://example.com
linkchecker --no-warnings https://example.com    # errors only
linkchecker -r 2 https://example.com             # limit recursion depth
```

### Check a list of URLs

```bash
while read url; do
  code=$(curl -s -o /dev/null -w "%{http_code}" --max-time 5 "$url")
  echo "$code $url"
done < urls.txt | grep -v "^200"
```

---

## Tracking Changes to a Website

### Basic change detection with diff

```bash
#!/bin/bash
url="$1"
snapshot_file="snapshot_$(echo $url | md5sum | cut -c1-8).txt"

new_content=$(curl -s "$url" | \
  sed 's/<[^>]*>//g' | \  # strip HTML
  sed '/^[[:space:]]*$/d')  # remove blank lines

if [[ -f "$snapshot_file" ]]; then
  if diff -q <(echo "$new_content") "$snapshot_file" > /dev/null; then
    echo "No changes detected."
  else
    echo "CHANGES DETECTED:"
    diff "$snapshot_file" <(echo "$new_content")
    echo "$new_content" > "$snapshot_file"
  fi
else
  echo "First run — saving snapshot."
  echo "$new_content" > "$snapshot_file"
fi
```

### Run on a schedule with cron

```bash
# Check every hour and email if changed
0 * * * * /path/to/check_changes.sh https://example.com | mail -s "Site Changed" you@gmail.com
```

### Hash-based detection (lightweight)

```bash
#!/bin/bash
url="$1"
hash_file=".site_hash"

current_hash=$(curl -s "$url" | md5sum | cut -d' ' -f1)

if [[ -f "$hash_file" ]]; then
  saved_hash=$(cat "$hash_file")
  if [[ "$current_hash" != "$saved_hash" ]]; then
    echo "$(date): CHANGED — $url"
    echo "$current_hash" > "$hash_file"
  else
    echo "$(date): No change"
  fi
else
  echo "$current_hash" > "$hash_file"
  echo "Baseline saved."
fi
```

---

## Posting to a Web Page and Reading the Response

### POST with curl

```bash
# Form submission
curl -X POST -d "username=omar&password=secret" https://example.com/login

# URL-encoded (same as above, explicit)
curl -X POST --data-urlencode "query=hello world" https://example.com/search

# JSON API
curl -s -X POST \
  -H "Content-Type: application/json" \
  -d '{"title": "New Post", "body": "Content here", "userId": 1}' \
  https://jsonplaceholder.typicode.com/posts | python3 -m json.tool
```

### Read the full response (headers + body)

```bash
curl -i https://example.com                    # headers and body together
curl -D - https://example.com                  # headers to stdout, body to stdout
curl -v https://example.com 2>&1               # verbose — everything
```

### Check status code

```bash
code=$(curl -s -o /dev/null -w "%{http_code}" -X POST -d "data=val" https://example.com)
echo "Response: $code"

if [[ "$code" == "200" || "$code" == "201" ]]; then
  echo "Success"
else
  echo "Failed with $code"
fi
```

### API interaction script

```bash
#!/bin/bash
API="https://jsonplaceholder.typicode.com"
TOKEN="your_token_here"

# GET
get_posts() {
  curl -s -H "Authorization: Bearer $TOKEN" "$API/posts" | python3 -m json.tool
}

# POST
create_post() {
  curl -s -X POST \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"title\": \"$1\", \"body\": \"$2\", \"userId\": 1}" \
    "$API/posts"
}

# DELETE
delete_post() {
  curl -s -X DELETE -o /dev/null -w "%{http_code}" "$API/posts/$1"
}

case "$1" in
  get)    get_posts ;;
  create) create_post "$2" "$3" ;;
  delete) delete_post "$2" ;;
  *)      echo "Usage: $0 {get|create|delete}" ;;
esac
```

---

## 📚 References

<div class="references">
<ul>
  <li><a href="https://www.packtpub.com/product/linux-shell-scripting-cookbook/9781785881985" target="_blank">Linux Shell Scripting Cookbook — Packt</a></li>
  <li><a href="https://curl.se/docs/manpage.html" target="_blank">curl Man Page</a></li>
  <li><a href="https://www.gnu.org/software/wget/manual/" target="_blank">GNU Wget Manual</a></li>
  <li><a href="https://dictionaryapi.dev/" target="_blank">Free Dictionary API</a></li>
</ul>
</div>

---

##  You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer )
- **GitHub:** [Mhdomer](https://github.com/Mhdomer)
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/)
- **Tryhackme:**  [nonlouy](https://tryhackme.com/p/nonlouy)
