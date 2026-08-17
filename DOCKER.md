# 🐳 Docker Complete Guide — Cheat Sheet + Compose Dev Setup

> Anand, ye dono PDFs (Docker Cheat Sheet + Docker Compose Dev Environment Setup) combine karke ek hi master file bana di hai. Sab commands, Dockerfile instructions, compose breakdown, volumes, common mistakes, concept diagrams aur interview questions — sab ek jagah.

---

## 📚 Table of Contents

1. [Need to Know Docker Commands](#1-need-to-know-docker-commands)
2. [Good to Know Docker Commands](#2-good-to-know-docker-commands)
3. [Key Concepts Explained](#3-key-concepts-explained)
4. [Essential Dockerfile Instructions](#4-essential-dockerfile-instructions)
5. [Project Structure (MERN-style)](#5-project-structure-mern-style)
6. [Docker Compose File — Full Breakdown](#6-docker-compose-file--full-breakdown)
7. [Volumes Deep Dive](#7-volumes-deep-dive)
8. [Adding New npm Packages (Container Workflow)](#8-adding-new-npm-packages-container-workflow)
9. [Commands You'll Actually Use Daily](#9-commands-youll-actually-use-daily)
10. [When Do You Actually Need to Rebuild?](#10-when-do-you-actually-need-to-rebuild)
11. [Common Mistakes (aur unke fix)](#11-common-mistakes-aur-unke-fix)
12. [Concept Diagrams](#12-concept-diagrams)
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
| `FROM <image>` | Base image, starting point | `FROM node:16-alpine` |
| `RUN <command>` | Executes commands during build (install packages) | `RUN apt-get update && apt-get install -y python3` |
| `COPY <src> <dest>` | Copies files from host → container | `COPY . /app` |
| `WORKDIR <path>` | Sets working directory for later instructions | `WORKDIR /app` |
| `EXPOSE <port>` | Documents which port container listens on | `EXPOSE 3000` |
| `CMD ["exe","p1","p2"]` | Default command when container starts (only last CMD wins if multiple) | `CMD ["node","server.js"]` |
| `ENTRYPOINT ["exe","p1"]` | Like CMD but not easily overridden by `docker run` args | `ENTRYPOINT ["/app/start-script.sh"]` |
| `ENV <key> <value>` | Sets env variable inside container | `ENV NODE_ENV production` |
| `ARG <var>=<default>` | Build-time variable (passed via `--build-arg`) | `ARG NODE_VERSION=16` |

**Sample Dockerfile (Node.js app):**
```dockerfile
FROM node:16-alpine
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

**Typical dev setup — bind mount + anonymous volume trick (avoid node_modules overwrite):**
```yaml
services:
  app:
    build: .
    volumes:
      - ./server:/app          # bind mount source code for live reload
      - /app/node_modules      # anonymous volume — protects container's own node_modules
```

**Why this matters:** without the second line, your host's `node_modules` (or missing folder) would overwrite the container's installed dependencies, breaking the app.

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
