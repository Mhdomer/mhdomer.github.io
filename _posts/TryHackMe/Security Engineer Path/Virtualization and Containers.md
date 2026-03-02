---
layout: post
title: Virtualization and Containers
date: 2026-02-14 10:00:00 +0800
categories:
  - TryHackMe
  -  Security Engineer Path 
tags:
  - Container
  - docker
  - thm
author: muhammed
description: Short summary of the post for previews and SEO.
toc: true
pin: false
math: false
mermaid: false
image: https://tryhackme-images.s3.amazonaws.com/room-icons/ad84791e77907798bf92424b643d0023.png
Link: "[[Security Engineer Path/Security-Engineer-Path-and-Notes]]"
Link1: "[[TryHackme]]"
---

# Introduction

**Cloud Computing** is basically just "someone else’s computer." Instead of buying a heavy, expensive server to sit under my desk, I rent power from a giant data center. It’s like using a laundry service instead of buying the washing machine—I only pay for what I use.

**Containers** (like Docker) are the "suitcases" for my code. Before, if I moved my code from my laptop to the Cloud, it would often break because the settings were different. Now, I pack the code and all its settings into a container.

If it works in the container on my laptop, it will work in the Cloud. No more "it works on my machine" excuses!



---
## What is Virtualization 

virtualization is the concept of encapsulating the capabilities and features of a physical machine in a virtual environment, known as a **virtual machine**.

But why is virtualization needed? For most organizations and individuals, virtualization comes from a need of the following:

- **Decrease expenses:** Physical servers can be expensive, and virtualization can decrease the number of servers or other hardware, or even completely remove physical hardware from a company's infrastructure.
- **Scale:** Without properly implemented DevOps, it may be hard for a company to scale resources as server usage increases. Virtualization makes this process easier and can delegate a server's resources to virtual machines as needed based on usage.
- **Efficiency:** Like scaling, virtualization can also make it easier to decrease the resources allocated to a virtual machine if there is reduced usage.

ormally, virtualization abstracts or creates an **abstraction layer** over computer hardware. An abstraction layer allows a single device to be divided into multiple virtual computers, also known as virtual machines (VMs).  

In simpler terms, this means that the virtual machine will have access to _logical resources_ that are abstracted away from the _physical resources_.

### Virtualization Structure

Virtualization is implemented using an engine-machine format, which means that a software or system creates an abstraction layer and allocates resources, while an operating system or application can then be installed on top of this virtualized environment. The operating system installed in a virtual machine is known as a **guest OS**, as opposed to the **host OS** on which the virtualization engine is running. 






---

## Hypervisor 

A hypervisor provides the ability to create the abstraction layer between hardware and software. A hypervisor will also generally include some form of management application or software to provide an interface between the end user and the abstraction layer to create or load virtual machines. 

Hypervisors are separated into **two** categories that are determined by their position relative to the hardware. They can either directly create a lightweight operating system on top of the hardware that is the hypervisor **or** add a hypervisor as an application on top of a pre-existing operating system.

### Type 1 Hypervisors 

**Type 1 hypervisors**, also known as **bare metal hypervisors**, create an abstraction layer directly between hardware and virtual machines without a common operating system between them. Instead, the hypervisor is the operating system and is often _headless_, with only a web-based management portal remotely accessed. These hypervisors are designed for scale and to deploy a large number of virtual machines at once. They are extremely lightweight to dedicate the most resources to virtual machines. Below is a diagram of a type 1 hypervisor architecture.

![gg](/assets/Pasted%20image%2020260227000015.png)

Examples of type 1 hypervisors include *VMware ESXi, Proxmox, VMware vSphere, Xen, and KVM.*


### Type 2 Hypervisors

**Type 2 hypervisors**, also known as **hosted hypervisors**, create an abstraction layer from a software application built on top of a pre-existing operating system. Unlike type 1 hypervisors, type 2 hypervisors are often managed directly from the application through a GUI. These hypervisors are often designed for end-users or individuals such as developers and are not strictly designed to run a large number of virtual machines for scale. Below is a diagram of a type 2 hypervisor architecture.

Examples of type 2 hypervisors include *VMware Workstation, VMware Fusion, VirtualBox, Parallels, and QEMU.*



---





---



#### Heading 4 



---
















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
