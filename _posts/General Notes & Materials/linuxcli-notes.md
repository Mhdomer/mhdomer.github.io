---
title: linuxcli
date: 2026-02-14 10:00:00 +0800
categories:
  - General Notes & Materials 
tags:
  - Notes
  - bash
  - scripting
  - linuxcli
author: muhammed
description: linux cli notes through time
toc: true
pin: false
math: false
mermaid: false
image:
---
---


## Append and overwrite files in Linux 


In Linux, the difference is just one extra character: `>` overwrites, while `>>` appends.

|**Command**|**Action**|**Result**|
|---|---|---|
|`echo "text" > file`|**Overwrite**|Deletes old content, writes "text".|
|`echo "text" >> file`|**Append**|Keeps old content, adds "text" to the bottom.|
|`tee -a file`|**Append with Sudo**|Safest way to append to system files like `/etc/hosts`.|
1) Using the Append Operator (`>>`)

```
echo "10.10.10.0 lookup.thm" >> /etc/hosts
```

2) Using `tee -a` (The "Permission" Way)

If I'm editing  `/etc/hosts`, it will  will likely run into a "Permission denied" error if i aren't logged in as root. Even using `sudo echo ... >> /etc/hosts` often fails because the _redirect_ (`>>`) is handled by your user's shell, not the sudo command.

The professional way to handle this is with the `tee` command using the `-a` (append) flag:

```
echo "10.10.10.0 lookup.thm" | sudo tee -a /etc/hosts

```

---


































---

##  You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer )
- **GitHub:** [Mhdomer](https://github.comMhdomer)  
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/) 
- **Tryhackme:**  [nonlouy](https://tryhackme.com/p/nonlouy)
