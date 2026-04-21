---
layout: post
title: Containerizing a 3-Tier MERN Stack With Docker Compose — Phase 2 Walkthrough
date: 2026-04-21T10:00:00
categories:
  - MindCraft Cloud Deployment
tags:
  - Container
  - docker
  - docker-compose
  - mongodb
author: muhammed
description: docker-walkthrough
toc: true
pin: false
math: false
mermaid: false
image: https://cdn.corenexis.com/files/c/9945582720.png
Link: "[[2026-03-24-Introduction-about-Application]]"
Link1:
---



Phase 1 gave us a working MERN stack running locally — Next.js on port 3000, Express API on
port 3001, MongoDB on port 27017, all running as bare Node processes on the host machine.

Phase 2 goal: wrap all three into Docker containers so the entire stack starts with one command
and runs identically on any machine, no local installs required.

```bash
docker compose up
```

That's the finish line. Here's what it took to get there.

---

## What We're Containerizing

Three services, three containers:

| Container | Image | Purpose | Port |
|---|---|---|---|
| `mindcraft-frontend` | Custom build | Next.js — serves the React app | 3000 (public) |
| `mindcraft-api` | Custom build | Express — REST API + JWT auth | 3001 (public for local dev) |
| `mindcraft-db` | `mongo:7` | MongoDB — database tier | 27017 (internal only) |

The key constraint: MongoDB should never be reachable from outside the stack. In production on
AWS, a Security Group enforces this at the network level. In Docker Compose, we enforce it with
Docker networks.

---

## The Dockerfiles

### Express API — `Dockerfile.api`

```dockerfile
# Stage 1 — install production dependencies only
FROM node:20-alpine AS deps
WORKDIR /app
COPY server/package*.json ./
RUN npm ci --omit=dev

# Stage 2 — runtime image
FROM node:20-alpine AS runner
WORKDIR /app

RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 express
USER express

COPY --from=deps --chown=express:nodejs /app/node_modules ./node_modules
COPY --chown=express:nodejs server/ .

EXPOSE 3001
ENV NODE_ENV=production

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget -qO- http://localhost:3001/health || exit 1

CMD ["node", "index.js"]
```

**Two stages.** Stage 1 installs only production dependencies (`--omit=dev`). Stage 2 copies
those into a fresh image. The build tools (`npm`, `package.json` devDeps) never make it into
the final image — smaller attack surface, smaller image size.

**Non-root user.** The process runs as `express` (uid 1001), not root. If an attacker gets
code execution inside the container, they can't write to the filesystem or escalate to root.

**Health check.** Docker Compose uses this to know when the API is actually ready — not just
started, but responding. The `depends_on` chain waits for `healthy` status before starting the
next container.

### Next.js Frontend — `Dockerfile.frontend`

```dockerfile
# Stage 1 — build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build

# Stage 2 — slim runtime
FROM node:20-alpine AS runner
WORKDIR /app

RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs
USER nextjs

ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
ENV PORT=3000

COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
COPY --from=builder --chown=nextjs:nodejs /app/public ./public

EXPOSE 3000
CMD ["node", "server.js"]
```

**The `standalone` output.** Next.js has a build mode that produces a self-contained bundle
with only what's needed to run — no `node_modules`, no source files. It's enabled in
`next.config.mjs`:

```javascript
export default { output: 'standalone' };
```

This cut the final image from ~1.2GB (full `node_modules`) to ~200MB.

**Issue we hit:** The config file was named `next.config.cjs` — Next.js only recognizes
`next.config.js` and `next.config.mjs`. The build succeeded and compiled all 36 pages, but
`.next/standalone` was never created because the config was silently ignored. The fix was
renaming to `next.config.mjs` and converting `module.exports` to `export default`.

---

## The Docker Compose File

```yaml
services:

  mongodb:
    image: mongo:7
    container_name: mindcraft-db
    restart: unless-stopped
    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_ROOT_USER}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_ROOT_PASSWORD}
      MONGO_INITDB_DATABASE: mindcraft
    volumes:
      - mongo-data:/data/db
    networks:
      - backend-net
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 20s

  api:
    build:
      context: .
      dockerfile: Dockerfile.api
    container_name: mindcraft-api
    depends_on:
      mongodb:
        condition: service_healthy
    environment:
      NODE_ENV: production
      MONGODB_URI: mongodb://${MONGO_ROOT_USER}:${MONGO_ROOT_PASSWORD}@mongodb:27017/mindcraft?authSource=admin
      JWT_SECRET: ${JWT_SECRET}
      FRONTEND_URL: ${FRONTEND_URL:-http://localhost:3000}
    ports:
      - "3001:3001"
    networks:
      - frontend-net
      - backend-net
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:3001/health"]

  frontend:
    build:
      context: .
      dockerfile: Dockerfile.frontend
    container_name: mindcraft-frontend
    depends_on:
      api:
        condition: service_healthy
    environment:
      NEXT_PUBLIC_API_URL: ${NEXT_PUBLIC_API_URL:-http://localhost:3001}
    ports:
      - "3000:3000"
    networks:
      - frontend-net

volumes:
  mongo-data:
    driver: local

networks:
  frontend-net:
    driver: bridge
  backend-net:
    driver: bridge
    internal: true   # ← MongoDB is unreachable from outside this network
```

**Three things worth explaining:**

### 1. Network Isolation

Two Docker networks:

- `frontend-net` — frontend and API both connect here. Frontend talks to API.
- `backend-net` — API and MongoDB both connect here. API talks to MongoDB.

MongoDB is **only** on `backend-net`. The frontend container is only on `frontend-net`.
That means the frontend container cannot reach MongoDB directly — ever. It has to go through
the API. This mirrors the AWS Security Group setup:

```
Browser → port 3000 → Frontend container
                           ↓
                      port 3001 (frontend-net)
                           ↓
                      API container
                           ↓
                      port 27017 (backend-net)
                           ↓
                      MongoDB container (not reachable from anywhere else)
```

`internal: true` on `backend-net` means Docker won't create a route out of that network to
the host or the internet.

### 2. Startup Order with Health Checks

`depends_on` with `condition: service_healthy` means:

1. MongoDB starts → waits until `mongosh ping` succeeds → marked healthy
2. API starts → waits until `/health` returns 200 → marked healthy
3. Frontend starts

Without this, the API would try to connect to MongoDB before MongoDB is ready and crash.
Without the API health check, the frontend would try to reach an API that isn't accepting
requests yet.

### 3. Environment Variables

Nothing is hardcoded. Every secret comes from `.env` at runtime:

```bash
# .env (never committed — in .gitignore)
MONGO_ROOT_USER=admin
MONGO_ROOT_PASSWORD=changeme
JWT_SECRET=your-32-char-secret-here
GEMINI_API_KEY=your-key-here
```

The compose file references them as `${VARIABLE_NAME}`. This is the same pattern used in
production with AWS Secrets Manager — the application code never sees where the secret comes
from, only its value.

---

## What Went Wrong (And Why)

### Problem 1: `next.config.cjs` not recognized

**Symptom:** Frontend built successfully — 36 pages compiled — but the Docker image failed with:
```
"/app/.next/standalone": not found
```

**Root cause:** `package.json` has `"type": "module"`, so `.js` files are ESM. The config was
named `.cjs` to use CommonJS syntax, but Next.js only looks for `next.config.js` or
`next.config.mjs`. It silently skipped the file, `output: 'standalone'` was never applied,
and the standalone directory was never generated.

**Fix:** Rename to `next.config.mjs`, replace `module.exports =` with `export default`.

### Problem 2: MongoDB auth mismatch

**Symptom:** API container logs showed `MongoDB connection error: Authentication failed` in a
restart loop.

**Root cause:** The `mongo-data` Docker volume had been created in an earlier `docker compose up`
run before we set `MONGO_ROOT_USER` and `MONGO_ROOT_PASSWORD` in `.env`. MongoDB only runs its
initialization scripts (setting the root user/password) when the data directory is empty. Since
the volume already existed, it kept the old state — which had no auth configured at all.

**Fix:**
```bash
docker compose down -v   # -v removes the named volumes
docker compose up -d     # fresh volume → MongoDB initializes with correct credentials
```

> **Warning:** `down -v` deletes all data in the volume. In production, you'd restore from
> a backup instead of destroying the volume.

---

## The Build Commands

```bash
# Build both images
docker compose build

# Build a single service
docker compose build api
docker compose build frontend

# Start the full stack (detached)
docker compose up -d

# View logs for a specific container
docker logs mindcraft-api
docker logs mindcraft-frontend
docker logs mindcraft-db

# Check container status and health
docker compose ps

# Stop everything (keep volumes)
docker compose down

# Stop everything AND delete volumes (fresh start)
docker compose down -v

# Rebuild and restart a single service after a code change
docker compose up -d --build api
```

---

## Answering the Sync Question

**"Shouldn't the Docker MongoDB be synced with the existing local database?"**

No — and that's intentional. The Docker MongoDB is a completely separate database from the
local MongoDB installation on the host machine. They don't share data, and they shouldn't.

Here's why:

- The local MongoDB (`localhost:27017`, no auth) is your **development** database — you connect
  to it directly with Compass, run the server with `npm run dev`, and it has no password.
- The Docker MongoDB (`mongodb` container, auth required) is your **integration testing**
  database — it runs with authentication, inside the network isolation, exactly as it will
  in production.

These are two separate environments. Data in one doesn't affect the other.

If you want to copy your local data into Docker, you can dump and restore:

```bash
# Export from local MongoDB
mongodump --uri="mongodb://localhost:27017/mindcraft" --out=./backup

# Import into Docker MongoDB
mongorestore --uri="mongodb://admin:changeme@localhost:27017/mindcraft?authSource=admin" ./backup/mindcraft
```

But for testing purposes, starting fresh is actually better — it verifies that the registration
and login flow works end-to-end without relying on pre-existing data.

---

## Result

```
$ docker compose up -d
✓ Container mindcraft-db       Healthy
✓ Container mindcraft-api      Healthy
✓ Container mindcraft-frontend Started

$ curl http://localhost:3001/health
{"status":"ok","timestamp":"...","env":"production"}
```

The full MindCraft stack — Next.js frontend, Express API, MongoDB — runs in Docker with a
single command. The database is isolated behind an internal network. Credentials come from
environment variables. Each container runs as a non-root user.

Phase 3 takes this same stack and provisions the AWS infrastructure to run it in production.

Source: [github.com/Mhdomer/mindcraft-aws-migration](https://github.com/Mhdomer/mindcraft-aws-migration)


##  You can find me online at:

![My signature image](/assets/img/footer-signature.png)

- **GitHub:** [Mhdomer](https://github.comMhdomer)  
- **LinkedIn:** [mhd3omar](https://www.linkedin.com/in/mhd3omar/) 
- **Tryhackme:**  [nonlouy](https://tryhackme.com/p/nonlouy)
