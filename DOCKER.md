# 🐳 Docker Complete Guide — Cheat Sheet + Compose Dev Setup

---

## 📚 Table of Contents

1. [Need to Know Docker Commands](#1-need-to-know-docker-commands)
2. [Good to Know Docker Commands](#2-good-to-know-docker-commands)
3. [Key Concepts Explained](#3-key-concepts-explained)
4. [Essential Dockerfile Instructions](#4-essential-dockerfile-instructions)
5. [Project Structure (MERN-style)](#5-project-structure-mern-style)
5.5. [Frontend Dockerfile + API Proxy Setup (Dev Mode)](#55-frontend-dockerfile--api-proxy-setup-dev-mode)
6. [Docker Compose File — Full Breakdown](#6-docker-compose-file--full-breakdown)
7. [Volumes Deep Dive](#7-volumes-deep-dive)
8. [Adding New npm Packages (Container Workflow)](#8-adding-new-npm-packages-container-workflow)
9. [Commands You'll Actually Use Daily](#9-commands-youll-actually-use-daily)
10. [When Do You Actually Need to Rebuild?](#10-when-do-you-actually-need-to-rebuild)
11. [Common Mistakes (aur unke fix)](#11-common-mistakes-aur-unke-fix)
12. [Concept Diagrams](#12-concept-diagrams)
12.5. [Beyond Basics — What a DevOps Engineer Should Know](#125-beyond-basics--what-a-devops-engineer-should-know)
13. [Interview Questions (Docker + Compose)](#13-interview-questions-docker--compose)
14. [Final Takeaway](#14-final-takeaway)

---

## 1. Need to Know Docker Commands

| Command | Description | Common Options |
|---|---|---|
| `docker build . -t <image-name>` | Builds a Docker image from a Dockerfile in the current directory | `-t <name>` tag image, `--no-cache` skip cached layers, `--build-arg <ARG=value>` set build-time vars |
| `docker run <image-name>` | Runs a container from an image | `-d` detached mode, `-p <host>:<container>` port mapping, `--name <name>`, `-v <host-path>:<container-path>` mount volume, `-e ENV_VAR=value`, `--rm` auto-remove on exit |
| `docker pull <image-name>` | Pulls image from registry (e.g. Docker Hub) | e.g. `node:latest`, `mongo:latest` |
| `docker push <image-name>` | Pushes local image to registry | Needs `docker login` first |
| `docker images` | Lists all local images | `-a` all layers, `-q` only IDs |
| `docker ps` | Lists running containers | `-a` all (incl. stopped), `-q` only IDs, `-f status=running` |
| `docker stop <container>` | Stops a running container | Use ID or name |
| `docker rm <container>` | Removes a stopped container | `-f` force remove even if running |
| `docker rmi <image>` | Removes an image | `-f` force even if in use |
| `docker logs <container>` | Shows container logs (debugging) | `-f` follow live, `--tail <n>` last N lines |

**Example Node.js flow:**
```bash
docker build . -t my-node-app
docker run -d -p 8080:3000 my-node-app
docker ps
docker logs <container-id>
docker stop <container-id>
docker rm <container-id>
docker rmi my-node-app
```

---

## 2. Good to Know Docker Commands

| Command | Description | Common Options |
|---|---|---|
| `docker exec -it <container> <command>` | Run a command inside a running container (debugging) | `-it` interactive TTY, `-u <user>` run as specific user |
| `docker inspect <container/image>` | Full JSON details of container/image | `-f '{{.NetworkSettings.IPAddress}}'` Go template extract |
| `docker stats` | Live resource usage (CPU/mem) | `--all`, `--no-stream`, `--format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}"` |
| `docker network ls` | List networks | `-q`, `--filter` |
| `docker network inspect <network>` | Details of a network | — |
| `docker volume ls` | List volumes | `-q`, `--filter` |
| `docker volume inspect <volume>` | Details of a volume | — |
| `docker-compose up` | Build/create/start/attach services from `docker-compose.yml` | `-d` detached, `--build` force rebuild, `--scale <svc>=<n>`, `-f <file>` |
| `docker-compose down` | Stop & remove containers/networks/volumes made by `up` | `--rmi <local\|all>`, `-v` remove named volumes, `-f <file>` |

**Common debugging move:** `docker exec -it <container-id> bash` → shell inside the container to inspect files/run commands.

---

## 3. Key Concepts Explained

- **Image** = blueprint/template. Contains code, runtime, libraries, env vars, config. Read-only, base for containers.
- **Container** = running instance of an image. Isolated, portable, consistent environment.
- **Dockerfile** = text file with instructions to build an image (base image, install steps, copy files, env vars, start command).
- **Docker Hub / Registry** = storage & distribution system for images (public: hub.docker.com, or private for org).
- **Docker Compose** = tool to define & run multi-container apps via `docker-compose.yml` (services, networks, volumes).
- **Networks** = allow containers to talk to each other. Default = bridge network; custom networks isolate/connect specific containers.
- **Volumes** = preferred way to persist data (vs bind mounts). Managed by Docker, easy to backup/restore/migrate. Used for DBs, app data, shared storage.

---

## 4. Essential Dockerfile Instructions

| Instruction | Description | Example |
|---|---|---|
| `FROM <image>` | Base image, starting point | `FROM node:20-alpine` |
| `RUN <command>` | Executes commands during build (install packages) | `RUN apt-get update && apt-get install -y python3` |
| `COPY <src> <dest>` | Copies files from host → container | `COPY . /app` |
| `WORKDIR <path>` | Sets working directory for later instructions | `WORKDIR /app` |
| `EXPOSE <port>` | Documents which port container listens on | `EXPOSE 3000` |
| `CMD ["exe","p1","p2"]` | Default command when container starts (only last CMD wins if multiple) | `CMD ["node","server.js"]` |
| `ENTRYPOINT ["exe","p1"]` | Like CMD but not easily overridden by `docker run` args | `ENTRYPOINT ["/app/start-script.sh"]` |
| `ENV <key> <value>` | Sets env variable inside container | `ENV NODE_ENV production` |
| `ARG <var>=<default>` | Build-time variable (passed via `--build-arg`) | `ARG NODE_VERSION=20` |

**Sample Dockerfile (Node.js app):**
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

---

## 5. Project Structure (MERN-style)

```
project-root/
├── client/                # Frontend (React/Vite)
│   ├── Dockerfile
│   └── src/
├── server/                # Backend (Node/Express/NestJS)
│   ├── Dockerfile
│   └── src/
├── docker-compose.yml     # ties everything together
```

Each service (client, server, db) has its own **Dockerfile**. `docker-compose.yml` orchestrates all of them together — networks, ports, volumes, env vars.

---

## 5.5 Frontend Dockerfile + API Proxy Setup (Dev Mode)

Frontend ka Dockerfile backend se thoda different hota hai kyunki Vite dev server hai, aur frontend ko backend API tak pahunchne ka rasta chahiye hota hai — ye rasta hi "proxy" kehlata hai. **Nginx abhi skip kar** — wo production-deploy ke time seekhna, abhi jo tu daily dev mein use karega wo hai **Vite ka apna built-in proxy**.

### A. Dev-mode frontend Dockerfile
```dockerfile
# client/Dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 5173
CMD ["npm", "run", "dev", "--", "--host"]
```
Ye Vite ka dev server chalata hai (hot reload ke sath) — isko hi tune compose mein `command: npm run dev -- --host` se override kiya tha. `--host` zaroori hai warna Vite sirf container ke andar hi bind hoga, tere host browser se accessible nahi hoga.

### B. Problem: frontend se backend API kaise call kare?

Compose mein frontend aur backend **do alag containers** hain, do alag ports pe (`5173` aur `5000`). Agar frontend se seedha `fetch('http://localhost:5000/api/users')` maaroge, ye kabhi kabhi kaam karega, kabhi CORS error dega — kyunki browser ki nazar mein ye do alag "origins" hain.

### C. Solution: Vite Proxy (`vite.config.js`)

Vite khud ek proxy feature deta hai — dev server ke andar hi ek rule likh dete hain ki `/api` se shuru hone wali koi bhi request automatically backend tak forward ho jaaye:

```js
// client/vite.config.js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    host: true,              // same as --host flag, allows access from outside container
      watch: {
      usePolling: true,
    },
    proxy: {
      '/api': {
        target: 'http://backend:5000',   // 'backend' = docker-compose service name
        changeOrigin: true,
        // secure: false,   // uncomment only if backend uses self-signed HTTPS cert
      },
    },
  },
})
```

**Ab tera frontend code mein:**
```js
// pehle (CORS issue prone):
fetch('http://localhost:5000/api/users')

// ab (Vite proxy ke through):
fetch('/api/users')
```
Bas `/api/users` likho — Vite dev server khud detect karega ki ye `/api` se start ho raha hai, aur background mein `http://backend:5000/api/users` ko forward kar dega. Browser ko lagta hai request same-origin hai, CORS issue hi nahi aata.

**`target: 'http://backend:5000'` mein `backend` kya hai?**
Ye tera Docker Compose service ka naam hai (jaisa `MONGO_URI` mein `db` hostname use hota tha). Docker Compose apna internal DNS resolve kar deta hai — `backend` naam se hi us container tak pahunch jaata hai, IP address nahi chahiye.

### D. Full compose setup (dev mode)
```yaml
services:
  backend:
    build: ./server
    ports:
      - "5000:5000"
    environment:
      - MONGO_URI=your_atlas_uri

  frontend:
    build: ./client
    ports:
      - "3000:5173"
    volumes:
      - ./client:/app
      - frontend-node-modules:/app/node_modules
    command: npm run dev -- --host
    depends_on:
      - backend

volumes:
  frontend-node-modules:
```
Vite proxy config (`vite.config.js`) automatically kaam karega jab tu `docker-compose up` chalayega — koi extra container ya extra setup nahi chahiye, bas ek config file.

---

> **Nginx / reverse proxy — abhi ke liye park kar de.** Jab tu production deploy karega (React build ko static files ke roop mein serve karna, aur ek single port se sab kuch expose karna), tab Nginx multi-stage build seekhna padega. Abhi dev stage mein Vite proxy hi kaafi hai, aur yehi industry mein bhi dev-time standard hai. Jab wo stage aayega, main tujhe step-by-step Nginx bhi sikha dunga.

---

## 6. Docker Compose File — Full Breakdown

```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8080:3000"
    environment:
      MONGODB_URI: mongodb://db:27017/mydb
    depends_on:
      - db

  db:
    image: mongo:latest
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db

volumes:
  mongodb_data:
```

**Line-by-line meaning:**

| Key | Meaning |
|---|---|
| `version` | Compose file schema version |
| `services` | Each service = one container blueprint |
| `build: .` | Build image from Dockerfile in current dir (vs `image:` which pulls a prebuilt image) |
| `ports: "8080:3000"` | host_port:container_port mapping |
| `environment` | Env vars injected into container at runtime |
| `depends_on` | Start order — `db` starts before `app` (does NOT wait for DB to be *ready*, just *started*) |
| `volumes` (service-level) | Mounts a named/bind volume into container path |
| `volumes` (top-level) | Declares named volumes Docker manages persistently |

**Full lifecycle:**
```bash
docker-compose up -d        # build+start everything in background
docker-compose ps           # check status of all services
docker-compose logs app     # view logs of one service
docker exec -it <db_id> bash   # go inside db container
docker-compose down         # stop & remove containers/networks
docker-compose down -v      # also remove volumes (data wiped!)
```

Compose auto-creates a network so `app` can reach `db` simply using hostname `db` — no need for IP addresses.

---

## 7. Volumes Deep Dive

Volumes are Docker's way of surviving container deletion — data outside the container's writable layer.

**Bind mount vs Named volume:**

| Bind Mount Syntax | Named Volume Syntax |
|---|---|
| `./local-folder:/app` (maps host path directly) | `mongodb_data:/data/db` (Docker-managed storage) |
| Good for **live code sync** in dev | Good for **persistent data** (DB, uploads) |
| Changes reflect instantly both ways | Isolated, portable, backup-friendly |

**Typical dev setup — bind mount + volume trick (avoid node_modules overwrite):**

There are **two ways** to do this. Both stop your bind mount from wiping out the container's installed `node_modules` — they just differ in whether the volume has a name.

**Option A — Anonymous volume (no name given):**
```yaml
services:
  app:
    build: .
    volumes:
      - ./server:/app          # bind mount source code for live reload
      - /app/node_modules      # anonymous volume — protects container's own node_modules
```
Docker auto-generates a random hash ID for this volume (visible as gibberish in `docker volume ls`). Quick to write, harder to track/clean up.

**Option B — Named volume (recommended, production-grade dev setup):**
```yaml
services:
  backend:
    build: ./server              # build image from server/Dockerfile
    ports:
      - "5000:5000"               # HOST:CONTAINER
    volumes:
      - ./server:/app                       # bind mount: your code → container
      - backend-node-modules:/app/node_modules  # named volume: protect deps
    environment:
      - MONGO_URI=your_atlas_uri
      - PORT=5000
    command: npx nodemon -L server.js                   # overrides Dockerfile CMD (nodemon)

  frontend:
    build: ./client
    ports:
      - "3000:5173"                         # host 3000 → vite's 5173
    volumes:
      - ./client:/app
      - frontend-node-modules:/app/node_modules
    command: npm run dev -- --host          # --host exposes vite outside container

# All named volumes must be declared here
volumes:
  backend-node-modules:
  frontend-node-modules:
```
Here `backend-node-modules` and `frontend-node-modules` are explicit names you choose, declared under the top-level `volumes:` key. Docker tracks them by that readable name (`docker volume ls` shows `backend-node-modules`, not a hash) — and each service gets its **own isolated volume** so backend and frontend deps never mix.

**Why this matters (both options):** without that second line, your host's `node_modules` (or a missing folder on host) would overwrite the container's installed dependencies at `/app/node_modules`, breaking the app. Docker's mount-overlap rule (more specific/longer path wins) is what lets the volume "win" over the bind mount for that one subfolder.

| | Anonymous (`- /app/node_modules`) | Named (`backend-node-modules:/app/node_modules`) |
|---|---|---|
| Naming | Docker generates a random hash | You choose the name |
| Tracking in `docker volume ls` | Hard to identify | Clear, readable |
| Reuse across rebuilds | Can spawn a new volume each recreate if inconsistent | Same name = same volume reliably reused |
| Cleanup | Prone to becoming a dangling/orphaned volume | Explicitly manageable (`docker volume rm backend-node-modules`) |
| Best for | Quick, throwaway setups | Real dev environments with multiple services (e.g. backend + frontend) |

**Extra details from the named-volume example above:**
- `command: npm run dev` overrides the Dockerfile's `CMD`, so nodemon/dev mode runs instead of a production start command.
- `command: npm run dev -- --host` on the frontend — Vite binds to `localhost` inside the container by default; `--host` binds it to `0.0.0.0` so it's reachable from outside the container.
- `"3000:5173"` maps host port 3000 to Vite's default dev port 5173 inside the container.

**Named volume for DB persistence:**
```yaml
services:
  db:
    image: mongo:latest
    volumes:
      - mongodb_data:/data/db
volumes:
  mongodb_data:
```
This way `docker-compose down` (without `-v`) keeps your DB data intact.

---

## 8. Adding New npm Packages (Container Workflow)

Since `node_modules` lives inside the container (not just your host), adding a package needs a specific flow:

**Option 1 — install inside the running container:**
```bash
docker exec -it <container_name> npm install <package-name>
```
Works but the change is lost if container is destroyed unless `package.json` is bind-mounted (it usually is, so `package.json` updates on host too).

**Option 2 — install locally, then rebuild:**
```bash
npm install <package-name>          # on host, updates package.json + lockfile
docker-compose up --build           # rebuilds image with new deps baked in
```
This is the **safer/recommended** approach for anything you want to persist long-term.

---

## 9. Commands You'll Actually Use Daily

```bash
docker-compose up -d          # start everything
docker-compose down           # stop everything
docker-compose logs -f app    # tail logs of one service
docker exec -it <container> bash   # jump inside a container
docker-compose up --build     # rebuild after Dockerfile/package.json changes
docker ps                     # see what's running right now
```

---

## 10. When Do You Actually Need to Rebuild?

| Change | Rebuild? |
|---|---|
| Edited source code (`.js`, `.jsx`, etc.) — with bind mount | ❌ No — live reload picks it up |
| Added new npm package | ✅ Yes — `docker-compose up --build` |
| Changed Dockerfile | ✅ Yes |
| Changed `docker-compose.yml` (ports/env) | ⚠️ Usually just `docker-compose up -d` again (recreate), not full rebuild |
| Changed base image version | ✅ Yes |

---

## 11. Common Mistakes (aur unke fix)

| Mistake | Fix |
|---|---|
| Port already in use / already allocated error | Kill process on that port (`lsof -i :PORT` → `kill -9 <pid>`) or change host port mapping |
| `.env` file not respected inside container | Not restarted/rebuilt after copy, missing `env_file:` in compose |
| Volumes wiping/overwriting `node_modules` | Add anonymous volume `/app/node_modules` in compose |
| `depends_on` doesn't mean "wait until ready" | Use healthchecks or retry logic in app, not just `depends_on` |
| Changes not reflecting | Bind mount missing, or built image cached — use `--build` or `--no-cache` |

---

## 12. Concept Diagrams

**A. Image → Container → Running App**
```
┌─────────────┐     docker run      ┌─────────────────┐
│   Image     │ ───────────────────▶│    Container     │
│ (blueprint) │                     │ (running instance)│
└─────────────┘                     └─────────────────┘
      ▲                                     │
      │ docker build .                      │ exposes port
      │                                     ▼
┌─────────────┐                     ┌─────────────────┐
│  Dockerfile │                     │  Host machine    │
│ (instructions)                    │  (port mapping)  │
└─────────────┘                     └─────────────────┘
```

**B. Docker Compose Multi-Container Setup**
```
                  ┌───────────────────────────┐
                  │      docker-compose.yml    │
                  └─────────────┬──────────────┘
                                 │
                 ┌───────────────┼───────────────┐
                 ▼                               ▼
         ┌───────────────┐               ┌───────────────┐
         │  app (Node)   │  ─network───▶ │   db (Mongo)  │
         │  port 8080    │  hostname:db  │   port 27017  │
         └───────┬───────┘               └───────┬───────┘
                 │ bind mount                     │ named volume
                 ▼                                 ▼
         ./server:/app                     mongodb_data:/data/db
        (live code sync)                   (persistent DB storage)
```

**C. Dev Workflow Flow**
```
Edit code (host) → bind mount syncs → container hot-reloads
     │
     ├─ Add npm package → npm install (host) → docker-compose up --build
     │
     └─ Debug issue → docker-compose logs / docker exec -it <c> bash
```

---

## 12.5 Beyond Basics — What a DevOps Engineer Should Know

Sab kuch upar tak **dev workflow** level tha (build/run/compose/volumes). Production/DevOps role mein iske aage ye sab bhi pata hona chahiye:

### A. Multi-stage builds (image size + security)
Ek hi Dockerfile mein multiple `FROM` stages — build stage bhaari hota hai (compilers, dev deps), final stage sirf runtime + built output leta hai. Final image chhota aur secure banta hai kyunki source code/dev-deps final image mein jaate hi nahi.
```dockerfile
# Stage 1: build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: production
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/server.js"]
```

### B. `.dockerignore`
`.gitignore` jaisa hi, par build context ke liye. Isse `node_modules`, `.env`, `.git`, logs wagera build ke andar copy hi nahi hote — build fast hota hai, image chhota aur secrets leak nahi hote.
```
node_modules
.env
.git
*.log
Dockerfile
```

### C. Non-root user (security)
Default container root user pe chalta hai — agar container compromise ho toh attacker ko root access mil jata hai. Production Dockerfile mein non-root user banana best practice hai:
```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

### D. Restart policies
Container crash ho jaaye toh kya kare — production mein zaroori hai:
```yaml
services:
  app:
    restart: unless-stopped   # options: no, always, on-failure, unless-stopped
```

### E. Healthchecks
Docker/orchestrator ko batana ki container "running" hai matlab wo "healthy/ready" bhi hai:
```yaml
services:
  app:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
```
Ye `depends_on` ki limitation (start order only, readiness nahi) ko fix karta hai — Compose v3.4+ mein `depends_on: { condition: service_healthy }` use kar sakte ho.

### F. Resource limits
Container ko unlimited CPU/RAM lene se rokna — production stability ke liye critical:
```yaml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
```

### G. Environment-specific compose files
Dev aur prod ke liye alag configs, base file + override merge hota hai:
```
docker-compose.yml           # base config
docker-compose.override.yml  # auto-merged in dev (bind mounts, hot reload)
docker-compose.prod.yml      # explicit: docker-compose -f docker-compose.yml -f docker-compose.prod.yml up
```

### H. Secrets management
`environment:` mein plaintext secrets daalna production mein bad practice hai (image layers/logs mein leak ho sakte hain). Better:
- `env_file:` (still not ideal for real secrets)
- Docker Secrets (Swarm) / Kubernetes Secrets
- External secret managers (AWS Secrets Manager, HashiCorp Vault) injected at runtime

### I. Image tagging & registries
- Kabhi bhi `latest` tag production mein use mat karo — deploy ka exact version untraceable ho jata hai
- Semantic/commit-based tags use karo: `myapp:v1.2.3`, `myapp:git-sha-abc123`
- Private registries: Docker Hub (private repo), AWS ECR, GCP Artifact Registry, GitHub Container Registry (GHCR)

### J. Image vulnerability scanning
Production images ko deploy se pehle scan karna chahiye known CVEs ke liye:
```bash
docker scout cves myapp:latest      # Docker's built-in scanner
trivy image myapp:latest            # popular open-source scanner
```

### K. Logging & signal handling
- Container logs by default `docker logs` mein jaate hain via **logging drivers** (`json-file`, `syslog`, `fluentd`, `awslogs`) — production mein centralized logging (ELK, CloudWatch, Datadog) ke liye configure karte hain
- **PID 1 problem**: container ke main process ko OS signals (SIGTERM on `docker stop`) properly handle karne chahiye, warna graceful shutdown nahi hota. Fix: `tini` init system use karo ya app khud signals handle kare.

### L. Cleanup / housekeeping
```bash
docker system prune -a         # remove unused images, containers, networks
docker volume prune            # remove unused volumes
docker image prune -a --filter "until=24h"   # remove old unused images
```

### M. CI/CD integration
Typical pipeline flow:
```
git push → CI builds image (docker build) → tags it → pushes to registry
        → CD pulls image on server/cluster → docker-compose up -d / kubectl apply
```
Key idea: **image built once in CI, same image promoted through dev → staging → prod** (never rebuilt per environment) — this is what makes Docker reliable for deployments.

### N. Beyond Compose — orchestration
Compose is single-host only. For multi-host/production scale:

| Tool | Use case |
|---|---|
| **Docker Compose** | Local dev, single-host small deployments |
| **Docker Swarm** | Simple multi-host orchestration, built into Docker |
| **Kubernetes (K8s)** | Industry-standard for production orchestration — auto-scaling, self-healing, rolling updates, service discovery |

Concepts that map from Compose → Kubernetes:
- `services:` → K8s **Deployments** + **Services**
- `volumes:` → K8s **PersistentVolumes**
- `environment:` → K8s **ConfigMaps/Secrets**
- `docker-compose up` → `kubectl apply -f`

### O. Docker networking (deeper)
| Driver | Use case |
|---|---|
| `bridge` (default) | Single-host container-to-container communication |
| `host` | Container shares host's network directly (no isolation, faster) |
| `overlay` | Multi-host networking (Swarm/K8s) |
| `none` | No networking (fully isolated) |

---

## 13. Interview Questions (Docker + Compose)

**Basics**
1. Difference between an **image** and a **container**?
2. What does `docker build` actually do internally (layers, caching)?
3. Why is Docker image caching layer-based? How do you optimize a Dockerfile for better cache hits (COPY package.json before COPY . .)?
4. What's the difference between `CMD` and `ENTRYPOINT`?
5. What happens if you have multiple `CMD` instructions in a Dockerfile?

**Networking & Compose**
6. How do containers in the same `docker-compose.yml` communicate with each other?
7. What does `depends_on` guarantee — and what does it NOT guarantee?
8. Difference between bridge, host, and none network drivers?
9. How would you expose a service only internally (not to host) in Compose?

**Volumes & Data**
10. Difference between a **bind mount** and a **named volume**? When would you use each?
11. Why do we add an anonymous volume like `/app/node_modules` in a dev compose setup?
12. How do you make sure DB data survives a `docker-compose down`?

**Debugging**
13. How do you get a shell inside a running container? Inside a stopped one?
14. How do you check why a container exited immediately after `docker run`?
15. `docker logs` vs `docker inspect` — when do you use which?

**Production/Advanced**
16. What's a multi-stage Docker build and why use it?
17. How do you reduce final image size (alpine base, multi-stage, .dockerignore)?
18. What is `.dockerignore` and why is it important?
19. How do secrets/env vars get managed safely in Docker (vs hardcoding in Dockerfile)?
20. Difference between `docker-compose up --build` vs just `docker-compose up`?

**Scenario-based**
21. Your container keeps restarting in a loop — how do you debug it?
22. You changed a package.json but `npm install` inside container isn't reflecting — why, and how do you fix it?
23. How would you set up a MERN stack (React + Node + MongoDB) entirely with Docker Compose for local dev?

**DevOps / Production-level**
24. What is a multi-stage build and why does it reduce final image size?
25. Why should you avoid using the `latest` tag in production deployments?
26. How do you handle secrets in Docker without hardcoding them in the Dockerfile or image?
27. What's the "PID 1 problem" in containers, and how do you solve it?
28. Difference between `docker-compose.yml`, `docker-compose.override.yml`, and a prod-specific compose file — how do they merge?
29. How would you set up a healthcheck for a service, and how is it different from `depends_on`?
30. What's the difference between Docker Swarm and Kubernetes? When would you pick one over the other?
31. How do you scan a Docker image for vulnerabilities before deploying it?
32. What are Docker logging drivers, and how would you send container logs to a centralized system (e.g. CloudWatch/ELK)?
33. Explain a typical CI/CD flow using Docker — where is the image built, and how does it move through environments?
34. Why do we run containers as a non-root user in production?
35. What's the difference between `bridge`, `host`, and `overlay` network drivers?
36. How do you limit CPU/memory usage for a container, and why does it matter in production?

**Frontend / API Proxy (Dev)**
37. Why can't the frontend container directly call `http://localhost:5000/api/...` reliably from inside a Compose setup?
38. How does Vite's `server.proxy` config solve the CORS problem in dev mode?
39. In `target: 'http://backend:5000'`, why does `backend` work as a hostname instead of an IP address?
40. What does the `--host` flag do for Vite inside a Docker container, and why is it required?

**Advanced / Future (once you learn Nginx)**
41. Why does a production frontend setup usually switch from Vite's dev proxy to an Nginx reverse proxy?
42. What is `try_files $uri $uri/ /index.html` used for in an Nginx config serving a React SPA?
43. Why should only the frontend's port be exposed to the host in production, with backend using `expose` instead of `ports`?

---

## 14. Final Takeaway

Docker ka sabse important mental model:

> **Image = frozen blueprint. Container = living instance. Compose = orchestrator that wires multiple containers (app + db) together over a shared network, with volumes for persistence and bind mounts for live dev.**

Daily loop for you (MERN/NestJS dev workflow):
```bash
docker-compose up -d --build   # first time / after deps change
docker-compose logs -f app     # watch logs while coding
docker exec -it <container> bash   # debug inside
docker-compose down            # end of day (data safe unless -v used)
```
