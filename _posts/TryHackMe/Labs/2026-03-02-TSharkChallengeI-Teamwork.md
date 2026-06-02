---
layout: post
title: SharkChallengeI
date: 2026-03-02T12:12:00
categories:
  - TryHackMe
  - writeUps 
tags:
  - thm
  - network
  - labs
author: muhammed
description: Put your TShark skills into practice and analyse some network traffic.
toc: true
pin: false
math: false
mermaid: false
image:
Link: "[[../LabsNotes/2026-03-02-TShark-The-Basics|2026-03-02-TShark-The-Basics]]"
Link1: "[[TryHackme]]"
---

# Case: Directory Curiosity!

**An alert has been triggered:** "A user came across a poor file index, and their curiosity led to problems".

The case was assigned to you. Inspect the provided **directory-curiosity.pcap** located in `~/Desktop/exercise-files` and retrieve the artefacts to confirm that this alert is a true positive.




---

Given a pcap file we need to filter based on the questions below 

### What is the name of the malicious/suspicious domain? 

```shell
tshark -r directory-curiosity.pcap -Y " dns " | grep 'com'
```


![f](/assets/thm/Pasted%20image%2020260304205412.png)

testing domains in virus total and found that 

![g](/assets/thm/Pasted%20image%2020260304205533.png)

**Answer: jx2-bavuong.com**

---

### What is the total number of HTTP requests sent to the malicious domain?

``

```shell
tshark -r directory-curiosity.pcap -Y "http.request and http.host contains jx2-bavuong.com" | wc -l
```


**Answer: 14** 

---

### What is the IP address associated with the malicious domain?

```shell
tshark -r directory-curiosity.pcap -Y "http.request and http.host contains jx2-bavuong.com" 
```

**Answer: 141[.]164[.]41[.]174**

---

### What is the server info of the suspicious domain?

```
tshark -r directory-curiosity.pcap  -T fields -e "http.server"| awk NF

```

**Answer: Apache/2.2.11 (Win32) DAV/2 mod_ssl/2.2.11 OpenSSL/0.9.8i PHP/5.2.9**


---


Follow the "first TCP stream" in "ASCII".  
Investigate the output carefully.

### What is the number of listed files?

```shell
tshark -r directory-curiosity.pcap -Y "tcp.stream eq 0" -z follow,tcp,ascii,0
```

**Answer: 3** 


### What is the filename of the first file?


**Answer: 123[.]php**


Export all HTTP traffic objects.  

### What is the name of the downloaded executable file?


**Answer: vlauto[.]exe**


---

### What is the SHA256 value of the malicious file?

to get this we need to export the file to external folder using this command 

```
tshark -r directory-curiosity.pcap --export-object http,./**output**
```


after this 

```
sha256sum output/vlauto.exe
```


**Answer: b4851333efaf399889456f78eac0fd532e9d8791b23a86a19402c1164aed20de** 

---


Search the SHA256 value of the file on VirtusTotal.  

### What is the "PEiD packer" value?


**Answer: .NET executable**

---

Search the SHA256 value of the file on VirtusTotal.  

### What does the "Lastline Sandbox" flag this as?


**Answer: MALWARE TROJAN**


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
