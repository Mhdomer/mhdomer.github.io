---
layout: post
title: Network Security Protocols
date: 2026-02-24T13:00:00
categories:
  - TryHackMe
  -  Security Engineer Path 
tags:
  - linux
  - thm
author: muhammed
description: Learn about secure network protocols at the different layers of the OSI model.
toc: true
pin: false
math: false
mermaid: false
image: https://tryhackme-images.s3.amazonaws.com/room-icons/847a4406931cf36d675800e1a95eb08f.png
Link: "[[TryHackme]]"
Link1: "[[Security-Engineer-Path-and-Notes]]"
---



# Introduction

A network protocol specifies how two devices, or more precisely processes, communicate with each other.

A network protocol is a pre-defined set of rules and processes to determine how data is transmitted between devices, such as end-user devices, networking devices, and servers. The fundamental objective of all protocols is to allow machines to connect and communicate seamlessly, regardless of any difference in their internal design, structure, logic, or operation. In analogy, a networking protocol is like a “common language” that helps make communication possible among people with different native languages and from various parts of the globe.



---

## Application Layer

- Each protocol represents specific layers of the OSI or TCP/IP model and operates as per the functionality of that layer.
- TCP and UDP-based protocols operate on specific network ports.

### HTTPS Protocol


Hypertext Transfer Protocol Secure (HTTPS) is a client-server protocol; responsible for securely sending data between a web server (website) and a web browser (client side). It is an encrypted variant of HTTP which sends data in an unencrypted format.


HTTP would be enough to browse a website to learn about a company product; however, HTTPS is a must if you want to provide your credit card details to place an online order. HTTPS was developed to securely share sensitive information, including passwords, contact information and financial information, between web browsers and websites. Without HTTPS, secure online banking and online payment wouldn’t have been possible.


#### Workflow - HTTPS

HTTPS uses its unencrypted counterpart, i.e., HTTP, and adds a layer of encryption. In this case, it is SSL/TLS (Secure Sockets Layer/Transport Layer Security); the rest of the workflow remains the same. So before proceeding forward, we will review how HTTP requests and responses operate in a typical client and server environment.

#### Request and Response - HTTP

An HTTP request is made by a user agent (a browser or any other application sending requests through a web API (Application Programming Interface)). It is vice versa in the case of response. This request aims to access some resources on the remote web server, which is then responded to by the web server. The figure below shows a web browser sending an HTTP request to a web server, listening at TCP port 80.




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
