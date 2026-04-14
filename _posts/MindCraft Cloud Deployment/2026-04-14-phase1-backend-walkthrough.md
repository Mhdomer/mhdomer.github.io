---
layout: post
title: Building a Secure MERN Backend From a Firebase App — Phase 1 Walkthrough
date: 2026-04-14T10:00:00
categories:
  - MindCraft Cloud Deployment
tags:
  - devops
  - cloud
  - mindcraft
  - jwt
  - mongodb
  - devsecops
  - nodejs
  - bcrypt
author: muhammed
description: Phase 1 backend walkthrough
toc: true
pin: false
math: false
mermaid: false
image: https://cdn.salla.sa/xArrjp/qKRJLW2AHJlt7ywZ1mDFaHBr3eMW2saQz1Nuxl0z.png
Link: "[[2026-03-24-Introduction-about-Application]]"
Link1:
---
In my [2026-03-24-Introduction-about-Application](2026-03-24-Introduction-about-Application.md), I wrote about why I decided to migrate MindCraft away from Firebase.
This post is the technical walkthrough of how Phase 1 actually happened — the decisions, the trade-offs, and the parts that took longer than expected.

Phase 1 had one job: replace Firebase entirely with a self-managed backend, and do it in a way

that a security or cloud engineer would find defensible.

Here's what that looked like in practice.

---

## The Starting Point

  

The original app had no backend in the traditional sense. It was a Next.js monolith — React pages

on one side, Firebase SDK calls on the other. No Express server, no REST API, no database queries.

Just `getDocs`, `addDoc`, `onSnapshot`, and `onAuthStateChanged` scattered across 30+ files.

  

Before I could build a new backend, I needed to understand what the old one was doing. I mapped

it out:

  

- **Auth:** `onAuthStateChanged` gave you a `user` object with a `uid`. Every Firestore query

  used that `uid` to filter documents.

- **Database:** Firestore collections — `user`, `course`, `module`, `lesson`, `enrollment`,

  `notification`, `post`, `submission`, `assignment`, `assessment`. Flat structure, string IDs.

- **API:** 16 Next.js Route Handlers in `app/api/` — thin wrappers around Firebase Admin SDK

  calls. They existed mainly to keep Firebase Admin credentials server-side.

  

That's what needed to be replaced.

  

---

  

## Decision 1: MongoDB over DynamoDB (or staying on Firestore)

  

The obvious move was to pick a database. I documented this in

[ADR-001](../adr/001-mongodb-over-firebase-firestore.md) — the short version:

  

DynamoDB is AWS-native but it's still a managed service. From a portfolio perspective,

DynamoDB and Firestore are the same story: AWS or Google manages the infrastructure, and you

just call an API. There are no Security Groups to configure, no port to isolate, no server to

maintain.

  

MongoDB on EC2 is different. It runs on a machine I provision, in a private subnet I design,

behind a Security Group I configure. The rule looks like this:

  

```

Inbound: TCP 27017 — Source: sg-app-tier only

```

  

That's one line of Terraform, but it tells a story in an interview: the database is unreachable

from the internet, unreachable from the web tier, and only accessible from the application tier.

That's real network isolation.

  

The document model also helped. Firestore data is hierarchical documents — MongoDB maps to that

directly. A `course` document with an embedded `modules` array translated almost one-to-one.

Switching to PostgreSQL would have meant designing a relational schema from scratch.

  

---

  

## The Data Migration Problem Nobody Warned Me About

  

This was the part that took the most time.

  

Firestore uses arbitrary string IDs — something like `"KJh3mN8pQ2rT"`. MongoDB uses ObjectIds —

24-character hex strings like `"507f1f77bcf86cd799439011"`. Every relationship reference in the

database (`courseId`, `createdBy`, `studentId`) was a Firestore string ID.

  

You can't just copy the data across. Every reference needs to be translated consistently —

if course `"KJh3mN8pQ2rT"` becomes ObjectId `"507f..."`, then every enrollment that references

`courseId: "KJh3mN8pQ2rT"` needs to be updated to `courseId: ObjectId("507f...")`.

  

The solution was an in-memory ID map built during migration:

  

```javascript

const idMap = new Map();

  

function getOrCreateId(firestoreId) {

  if (!idMap.has(firestoreId)) {

    idMap.set(firestoreId, new mongoose.Types.ObjectId());

  }

  return idMap.get(firestoreId);

}

```

  

Every Firestore document gets a consistent ObjectId on first encounter. When you hit a reference

field, you look it up in the map instead of generating a new one. Run users first, then courses,

then enrollments — by the time you're resolving `studentId` in an enrollment, the user's ObjectId

is already in the map.

  

The full script is at `scripts/migrate-firebase-to-mongo.js`. It migrates users, courses,

modules, lessons, enrollments, forum posts, and notifications in dependency order.

  

---

  

## 13 Mongoose Schemas

  

I wrote 13 schemas covering everything the app needed:

  

`User`, `Course`, `Module`, `Lesson`, `LessonExercise`, `Enrollment`, `Assignment`,

`Assessment`, `Submission`, `Notification`, `Post` (forum), `AuditLog`, `GameLevel`

  

A few design decisions worth mentioning:

  

**Indexes on every query field.** If a route filters by `userId`, the field has an index.

Mongoose lets you declare these inline:

  

```javascript

const NotificationSchema = new Schema({

  userId: { type: String, required: true, index: true },

  read:   { type: Boolean, default: false, index: true },

  // ...

});

```

  

**No plain passwords stored anywhere.** The `User` schema stores `passwordHash` only — bcrypt

output. There is no `password` field. The migration script generates a temporary hash for

existing users; they reset on first login.

  

**Soft references vs embedded documents.** Courses reference modules by ObjectId array rather

than embedding them. This lets a module exist independently and be updated without rewriting the

parent course document.

  

---

  

## Decision 2: JWT in an httpOnly Cookie

  

JWT is standard. The interesting decision is where you put it.

  

The two options are localStorage and cookies. I went with `httpOnly` cookies and documented

why in [ADR-002](../adr/002-jwt-over-firebase-auth.md).

  

The reason: JavaScript cannot read `httpOnly` cookies. That means an XSS vulnerability — an

injected script running on your page — cannot steal the token. localStorage has no such

protection. If your site has an XSS hole and you're storing JWTs in localStorage, an attacker

gets permanent access until the token expires.

  

The Express login route sets it like this:

  

```javascript

res.cookie('auth_token', token, {

  httpOnly: true,

  secure: process.env.NODE_ENV === 'production',

  sameSite: 'lax',

  maxAge: 24 * 60 * 60 * 1000, // 24 hours

});

```

  

`sameSite: 'lax'` blocks the cookie from being sent in cross-site POST requests (CSRF

mitigation). `secure: true` in production means it's only sent over HTTPS. The 24-hour expiry

is short enough to limit damage from a compromised token, long enough that users don't get

logged out constantly.

  

**bcrypt with 12 salt rounds.** The default for most tutorials is 10. The difference between

10 and 12 is that 12 is ~4x slower to compute. That's irrelevant for login (milliseconds), but

it means cracking a stolen hash database takes ~4x longer. For a portfolio project, 12 is the

right call — it signals you thought about it.

  

```javascript

const passwordHash = await bcrypt.hash(password, 12);

```

  

---

  

## Decision 3: Extracting the Express API

  

The original Next.js app used `app/api/` route handlers. That's fine for a monolith, but it

means the frontend and backend are the same process — you can't put them on separate EC2

instances or apply different Security Groups.

  

[ADR-003](../adr/003-express-api-tier-separation.md) covers this. The result was a standalone

`server/` directory — a full Express application that runs on port 3001.

  

The structure ended up being:

  

```

server/

├── config/

│   ├── database.js      # mongoose.connect with retry logic

│   └── env.js           # validates required env vars on startup

├── middleware/

│   ├── auth.js          # requireAuth, requireRole, optionalAuth

│   └── ...

├── models/              # 13 Mongoose schemas

├── routes/              # 14 route modules, 40+ endpoints

└── index.js             # Express app entry point

```

  

The frontend communicates with it via a centralized `lib/api.js` fetch wrapper:

  

```javascript

const BASE = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3001';

  

async function request(path, options = {}) {

  const res = await fetch(`${BASE}${path}`, {

    ...options,

    credentials: 'include', // sends the httpOnly cookie

    headers: { 'Content-Type': 'application/json', ...options.headers },

  });

  if (!res.ok) {

    const body = await res.json().catch(() => ({}));

    throw new Error(body.error || `Request failed: ${res.status}`);

  }

  return res.json();

}

```

  

`credentials: 'include'` is the key line — it tells the browser to attach the `auth_token`

cookie to every request, including cross-origin ones to `localhost:3001`.

  

---

  

## Security Baseline: Helmet + Rate Limiting + Morgan

  

Three things I added to every Express app I write now:

  

**Helmet.js** sets security headers in one call. The defaults cover:

`X-Content-Type-Options`, `X-Frame-Options`, `Strict-Transport-Security`, `X-XSS-Protection`,

and several others. One line:

  

```javascript

app.use(helmet());

```

  

**Rate limiting** on auth endpoints. Without it, someone can try passwords until they find

one that works. The auth routes get a strict limit:

  

```javascript

const authLimiter = rateLimit({

  windowMs: 15 * 60 * 1000, // 15 minutes

  max: 20,

  message: { error: 'Too many requests, please try again later.' },

});

app.use('/api/auth', authLimiter);

```

  

20 attempts per 15 minutes per IP. Enough for normal users (who don't fail login 20 times),

tight enough to make brute-force attacks impractical.

  

**Morgan** for request logging. In development it shows coloured output — method, path, status,

response time. In production it writes combined Apache format, which CloudWatch can parse. The

switch is one line:

  

```javascript

app.use(morgan(isProd ? 'combined' : 'dev'));

```

  

**Environment validation on startup.** The server refuses to start if required environment

variables are missing. No silent failures where `JWT_SECRET` is undefined and tokens are signed

with an empty string:

  

```javascript

const REQUIRED = ['MONGODB_URI', 'JWT_SECRET'];

  

export function validateEnv() {

  const missing = REQUIRED.filter(k => !process.env[k]);

  if (missing.length) {

    console.error(`Missing required env vars: ${missing.join(', ')}`);

    process.exit(1);

  }

}

```

  

This catches misconfigured deployments immediately rather than letting the app run in a broken

state.

  

---

  

## The Frontend Migration

  

The backend was the cleaner part. Removing Firebase from 30+ frontend files was messier.

  

The pattern repeated across every page: `onAuthStateChanged` in a `useEffect`, with a dynamic

Firestore import to fetch the user's role. Every page had its own copy of this pattern.

  

```javascript

// Before — in almost every page

useEffect(() => {

  const unsub = onAuthStateChanged(auth, async (user) => {

    if (user) {

      const doc = await getDoc(doc(db, 'user', user.uid));

      setUserRole(doc.data()?.role);

    }

  });

  return () => unsub();

}, []);

```

  

The replacement was an `AuthContext` that polls `/api/auth/me` on mount and exposes `userData`

globally. Every page becomes:

  

```javascript

const { userData, loading } = useAuth();

```

  

The hardest component was `NotificationBell` — it had three simultaneous Firestore `onSnapshot`

listeners running. Real-time updates became 30-second polling:

  

```javascript

useEffect(() => {

  fetchNotifications();

  const interval = setInterval(fetchNotifications, 30000);

  return () => clearInterval(interval);

}, [userData]);

```

  

Is polling less elegant than real-time listeners? Yes. Does it run on infrastructure I control,

with no Google dependency? Also yes. That trade-off was worth it.

  

---

  

## By The Numbers

  

After Phase 1:

  

- **13** Mongoose schemas

- **14** Express route modules

- **40+** REST endpoints

- **16** Next.js API route handlers deleted

- **0** Firebase imports remaining in `app/` or `components/`

- **~700** lines of Firestore debug logging removed from the analytics dashboard

  

The server starts in under a second, connects to MongoDB, validates the environment, and refuses

to serve traffic until all three are confirmed healthy.

  

---

  

## What's Next

  

Phase 2 is Docker. The goal: `docker compose up` should start the full stack — frontend,

Express API, and MongoDB — with no local installs required.

  

The Dockerfiles already exist (`Dockerfile.frontend`, `Dockerfile.api`). The interesting part

of Phase 2 is the network configuration: making sure the three containers can only talk to the

services they're supposed to, mimicking the Security Group isolation that AWS will enforce in

production.

  

Phase 3 after that is Terraform — provisioning the actual AWS infrastructure.

  

Source: [github.com/Mhdomer/mindcraft-aws-migration](https://github.com/Mhdomer/mindcraft-aws-migration)

  




---

##  You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.comMhdomer)  
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/) 
- **Tryhackme:**  [nonlouy](https://tryhackme.com/p/nonlouy)
