---
layout: post
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
## Namespaces and cgroups 

**namespaces**: is the concept of isolating a process from other process and make it in a " bubble " that it sees only its own isolated version of those resources 


Normally, all processes share the same system resources — the same network, same process list, same filesystem root, etc. Namespaces create a "bubble" around a process so it only sees its own isolated version of those resources.


This is the core technology behind **containers** (like Docker).

## Types of Namespaces

|Namespace|Flag|Isolates|
|---|---|---|
|**PID**|`CLONE_NEWPID`|Process IDs — processes see only their own PID tree|
|**Network**|`CLONE_NEWNET`|Network interfaces, IP addresses, routing tables, ports|
|**Mount**|`CLONE_NEWNS`|Filesystem mount points|
|**UTS**|`CLONE_NEWUTS`|Hostname and domain name|
|**IPC**|`CLONE_NEWIPC`|Inter-process communication (shared memory, semaphores)|
|**User**|`CLONE_NEWUSER`|User and group IDs (UID/GID mapping)|
|**Cgroup**|`CLONE_NEWCGROUP`|Cgroup root directory visibility|
|**Time**|`CLONE_NEWTIME`|System clocks (boot time, monotonic)|

## Namespaces vs. cgroups

These two features are often used together but do different things:

- **Namespaces** → control what a process _can see_ < التحكم فالي البروسيس يقدر يشوفه >
- **cgroups** → control how many _resources_ (CPU, RAM) a process can use < التحكم فالي البروسيس يقدر يستخدمه (ram, CPU)

Together they form the foundation of **Linux containers**.

---





















---

##  You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer )
- **GitHub:** [Mhdomer](https://github.comMhdomer)  
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/) 
- **Tryhackme:**  [nonlouy](https://tryhackme.com/p/nonlouy)
