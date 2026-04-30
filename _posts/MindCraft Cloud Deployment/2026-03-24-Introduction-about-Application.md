---
layout: post
title: "Phase 0: MindCraft Overview"
date: 2026-03-24T10:00:00
categories:
  - MindCraft Cloud Deployment
tags:
  - aws
  - cloud
  - deployment
author: muhammed
description: In this Writeup i will explain what is MindCraft and why I took the steps to deploy it on Cloud
toc: true
pin: false
math: false
mermaid: false
image: https://mind-craft-k21f.vercel.app/logoMindCraft.jpg
Link: "[[]]"
Link1:
---

# From Course Project to Cloud Deployment

## The Beginning

This application started as a university course project  built as part of a team to fulfill academic requirements.

We designed and developed a full-stack learning platform with features like:

- Course and content management
- Student progress tracking
- Assessments and assignments
- AI-assisted learning features

At the time, the goal was simple: **complete the project and pass the course.** 

Ps to view the current deployed version, you can find it here [MindCraft ](https://mind-craft-k21f.vercel.app/)


## The Turning Point

After the semester ended, the project just sat there.

Finished… but not really used.

That’s when a question came to mind:

> What if I actually treated this like a real product?

Instead of leaving it as just another academic submission, I decided to take it a step further.


## Why Take It to the Cloud?

Building an app is one thing.

Deploying it, securing it, and making it production-ready is a completely different challenge.

So this blog series is about:

- Taking a completed project and deploying it to the cloud
- Improving it beyond its academic version
- Gradually hardening it with better security and practices
- Turning it into something closer to a real world system

## What This Series Will Be

- Figuring things out
- Breaking things and fixing them
- Learning what actually matters in production

This is my attempt to push the project further  beyond grades, into something practical as well as it will act as a " trail " for my upcoming FYP to enhance my knowledge. 

Next, I’ll start preparing the project for deployment and setting up the environment to get it running outside my local machine.


## The Problem: Firebase Looked Great Until It Didn't

  

When we built MindCraft as a team, Firebase made perfect sense. One SDK, auth and database handled, deployed in minutes. For a 7-person team with a semester deadline, that was the right call.
  

Firebase is a Backend-as-a-Service. Google manages everything below the application layer. That means:

- No Security Groups to configure

- No private subnets to reason about

- No port isolation between services

- No Linux server to harden

For a frontend role, Firebase is fine. For a cloud/security role at a company that runs its own infrastructure, 
  
## What the Migration Actually Involves


I mapped out exactly what "removing Firebase" means in practice:
  

| What Firebase Gave Us | What We Replace It With |

|Service / Feature|Replacement / Implementation|
|---|---|
|Firestore (database)|MongoDB on EC2 in a private subnet|
|Firebase Auth|JWT with bcrypt, `httpOnly` cookies|
|Firebase Hosting|Nginx + Next.js Docker container on EC2|
|Firebase Admin SDK|Express.js + Mongoose|

That's essentially a full MERN stack migration wrapped inside a DevSecOps project.

## The Part Nobody Talks About: The Data Migration

  
The trickiest part wasn't the Express routes or the JWT implementation. It was the ID mapping problem.


Firestore uses arbitrary string IDs. MongoDB uses ObjectIds. Every relationship reference (`courseId`, `createdBy`, etc.) needed to be translated consistently. The solution: build a `Map` in memory during migration that tracks `firestoreId → ObjectId` and resolve references as you go.

  

```javascript

// Every Firestore ID gets a consistent MongoDB ObjectId

const idMap = new Map();

  

function getOrCreateId(firebaseId) {

  if (!idMap.has(firebaseId)) {

    idMap.set(firebaseId, new mongoose.Types.ObjectId());

  }

  return idMap.get(firebaseId);

}

```

  
## Security Decisions I'm Proud Of

  
#### JWT in an `httpOnly` Cookie (Not localStorage)
XSS attacks can't steal httpOnly cookies, localStorage is a common mistake]
#### Rate Limiting on Auth Endpoints
#### Helmet.js: One Line, 10 Security Headers

#### bcrypt with 12 Salt Rounds


  

## Architecture Decision Records

  

One practice I picked up during this project: writing ADRs (Architecture Decision Records). Every major technical choice gets a short document explaining: 

- Why this decision was needed

- What was decided

- What trade-offs were accepted

- What alternatives were rejected

to find it go through the ADR section under the MindCraft Deployment
  
This turned out to be the most useful documentation I wrote — not just for the blog, but for interview prep. When an interviewer asks "why MongoDB over DynamoDB?", I have a three-paragraph answer ready.

  

## What's Next

  

- [ ] Docker Compose to run the full stack locally

- [ ] Frontend auth context rewrite (Firebase Auth → JWT)

- [ ] Terraform for the AWS 3-tier network

- [ ] GitHub Actions pipeline with Trivy + SonarCloud

  

  

---

  

*The full source is on [GitHub](https://github.com/Mhdomer/mindcraft-aws-migration). If you're doing something similar, feel free to reach out on [LinkedIn](https://linkedin.com/in/Mhd3omar).*





---

##  You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **X (Twitter):** [Md3omer](https://x.com/Md3omer )
- **GitHub:** [Mhdomer](https://github.comMhdomer)  
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/) 
- **Tryhackme:**  [nonlouy](https://tryhackme.com/p/nonlouy)
