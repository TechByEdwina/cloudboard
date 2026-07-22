# 🐳 CloudBoard — Docker Compose Workshop

> A hands-on educational project for students learning Docker and Docker Compose.
> Build, run, and explore a real multi-container application in 60–90 minutes.

---

## Table of Contents

1. [Project Scenario](#1-project-scenario)
2. [Architecture Overview](#2-architecture-overview)
3. [Service Breakdown](#3-service-breakdown)
4. [Prerequisites](#4-prerequisites)
5. [Quick Start](#5-quick-start)
6. [Environment Variables](#6-environment-variables)
7. [Docker Networking & Service Discovery](#7-docker-networking--service-discovery)
8. [Named Volumes & Persistence](#8-named-volumes--persistence)
9. [Docker Compose Command Reference](#9-docker-compose-command-reference)
10. [Exploring Redis Live](#10-exploring-redis-live)
11. [Troubleshooting](#11-troubleshooting)
12. [Hands-On Exercises](#12-hands-on-exercises)
13. [Project Structure](#13-project-structure)

---

## 1. Project Scenario

You are helping a lecturer run a digital attendance board for their Docker workshop.

Students arrive, visit **CloudBoard**, and register their names. The lecturer can see a live list of attendees, view statistics about visits, and reset the board between sessions — all without touching a database GUI or writing SQL.

Under the hood, three Docker containers work together:

- A **Flask web application** serves the UI and handles requests.
- A **Redis** instance stores every piece of data — visitor counts, student names, timestamps.
- A **Redis Commander** web interface lets you inspect Redis keys in real time.

The entire stack starts with **one command**: `docker compose up --build`.

---

## 2. Architecture Overview

```
┌────────────────────────────────────────────────────────────┐
│                     Docker Host (your machine)             │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              cloudboard-net  (bridge network)        │  │
│  │                                                      │  │
│  │   ┌─────────────┐     ┌─────────────┐               │  │
│  │   │     app     │────▶│    redis    │               │  │
│  │   │  Flask:5000 │     │  port 6379  │               │  │
│  │   └─────────────┘     └──────┬──────┘               │  │
│  │                              │ (redis-data volume)   │  │
│  │   ┌─────────────────┐        │                       │  │
│  │   │ redis-commander │────────┘                       │  │
│  │   │   port 8081     │                                │  │
│  │   └─────────────────┘                                │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  Exposed to host:                                          │
│    localhost:5000  →  app (Flask UI)                       │
│    localhost:8081  →  redis-commander (Redis GUI)          │
│    Redis (6379) is NOT exposed — internal only             │
└────────────────────────────────────────────────────────────┘
```

All three containers communicate over a private Docker bridge network named `cloudboard-net`. They discover each other using **service names** as hostnames — no IP addresses required.

---

## 3. Service Breakdown

### `app` — Flask Web Application

| Property | Value |
|---|---|
| Built from | `./app/Dockerfile` |
| Base image | `python:3.12-slim` |
| Exposed port | `5000` (mapped to host `5000`) |
| Depends on | `redis` |

This is the only service we **build ourselves**. The `Dockerfile` in `./app/`:

1. Starts from an official Python 3.12 slim image.
2. Installs dependencies from `requirements.txt` (Flask, redis-py, Gunicorn).
3. Copies the application source code.
4. Starts the app with **Gunicorn**, a production WSGI server.

The Flask app reads all its configuration from environment variables, making it easy to customise without changing code.

**Redis operations used:**

| Route | Redis Command | Purpose |
|---|---|---|
| `GET /` | `INCR stats:visitor_count` | Increment visit counter atomically |
| `GET /` | `SET stats:last_visit` | Store current timestamp |
| `POST /register` | `LPUSH students:names` | Add name to list |
| `POST /register` | `SET stats:last_student` | Track last registrant |
| `POST /register` | `INCR stats:student_count` | Increment student counter |
| `GET /students` | `LRANGE students:names 0 -1` | Read full list |
| `GET /stats` | `GET stats:*` | Read all stat keys |
| `POST /reset` | `FLUSHALL` | Wipe all data |

---

### `redis` — Data Store

| Property | Value |
|---|---|
| Image | `redis:alpine` |
| Internal port | `6379` (not exposed to host) |
| Volume | `redis-data:/data` |

Redis is our **only data store**. No SQLite, no JSON files. Every piece of application state — visitor counts, student names, timestamps — lives in Redis.

We use the `alpine` tag for a smaller image. Redis is configured with persistence enabled:

```
redis-server --save 60 1 --appendonly yes
```

- `--save 60 1` — write an RDB snapshot every 60 seconds if at least 1 key changed.
- `--appendonly yes` — also maintain an append-only file (AOF) for durability.

**Note:** The `redis` service intentionally has **no `ports:` mapping** to the host. Redis is internal — only containers on `cloudboard-net` can reach it. This is a security best practice.

---

### `redis-commander` — Redis Web GUI

| Property | Value |
|---|---|
| Image | `rediscommander/redis-commander` |
| Exposed port | `8081` (mapped to host `8081`) |
| Purpose | Browse and inspect Redis keys live |

Redis Commander gives you a visual window into Redis. During the workshop, keep it open at [http://localhost:8081](http://localhost:8081) and watch keys appear as you interact with CloudBoard.

It connects to Redis using the service name: `REDIS_HOSTS=local:redis:6379`.

---

## 4. Prerequisites

- **Docker Desktop** (includes Docker Engine and Docker Compose)  
  Download: https://www.docker.com/products/docker-desktop/
- Git (optional, for cloning)

Verify your installation:

```bash
docker --version
docker compose version
```

You should see version numbers for both commands.

---

## 5. Quick Start

```bash
# 1. Clone or download the project
git clone <repo-url> cloudboard
cd cloudboard

# 2. (Optional) customise settings
cp .env.example .env
# Edit .env with your preferred editor

# 3. Build and start all three containers
docker compose up --build

# 4. Open in your browser
#    CloudBoard UI      →  http://localhost:5000
#    Redis Commander    →  http://localhost:8081
```

To stop the stack:

```bash
# Stop containers (keeps the redis-data volume intact)
docker compose down

# Stop AND delete the data volume (full reset)
docker compose down --volumes
```

---

## 6. Environment Variables

All configurable values are passed to containers via environment variables. This follows the [12-Factor App](https://12factor.net/config) methodology — configuration is separated from code.

| Variable | Default | Description |
|---|---|---|
| `APP_NAME` | `CloudBoard` | Application name shown in UI |
| `APP_GREETING` | `Welcome to the Docker Compose Workshop!` | Hero greeting text |
| `REDIS_HOST` | `redis` | Hostname of the Redis service |
| `REDIS_PORT` | `6379` | Port Redis listens on |
| `FLASK_PORT` | `5000` | Port the Flask app listens on |
| `FLASK_DEBUG` | `0` | Set to `1` to enable Flask debug mode |

### How to customise

**Option A — `.env` file (recommended)**

```bash
cp .env.example .env
```

Edit `.env`:

```env
APP_NAME=My Workshop Board
APP_GREETING=Hello, students!
FLASK_PORT=8080
```

Docker Compose automatically loads `.env` from the project root.

**Option B — inline override**

```bash
APP_NAME="Lab 3 Board" docker compose up --build
```

**Option C — `compose.yaml` directly**

Edit the `environment:` block under the `app` service.

---

## 7. Docker Networking & Service Discovery

### What is a Docker network?

When you run `docker compose up`, Compose creates a **bridge network** named `cloudboard-net` and attaches all three services to it. This is a private, isolated virtual network that exists only on the Docker host.

Containers on the same network can communicate freely. Containers on different networks (or external systems) cannot reach them unless ports are explicitly published.

### Service Discovery — How containers find each other

Inside `cloudboard-net`, each container is reachable using its **service name** as a hostname. Docker runs an internal DNS server that resolves these names.

In `app.py`:

```python
REDIS_HOST = os.environ.get("REDIS_HOST", "localhost")
# When running in Docker Compose, REDIS_HOST="redis"
# Docker DNS resolves "redis" → 172.18.0.x (the Redis container's IP)
```

This is **service discovery**. You never need to know a container's IP address.

**Try it yourself:**

```bash
# "Ping" the Redis container from inside the Flask container
docker compose exec app ping redis

# Look up the IP address Docker assigned
docker compose exec app nslookup redis
```

### Published vs. internal ports

| Service | Internal port | Published to host? |
|---|---|---|
| app | 5000 | ✅ Yes — `5000:5000` |
| redis | 6379 | ❌ No — internal only |
| redis-commander | 8081 | ✅ Yes — `8081:8081` |

Redis is intentionally not published. Only containers inside `cloudboard-net` can reach it. This is a security best practice — you don't want Redis exposed to the network.

---

## 8. Named Volumes & Persistence

### The problem without volumes

Docker containers are **ephemeral** — their filesystem is temporary. When you run `docker compose down`, container filesystems are destroyed, and any data Redis wrote to disk is gone.

### The solution: Named volumes

In `compose.yaml`:

```yaml
volumes:
  redis-data:
    driver: local
```

And mounted on the Redis service:

```yaml
redis:
  volumes:
    - redis-data:/data
```

A **named volume** is a piece of storage managed by Docker, stored outside any container. When the Redis container writes to `/data`, Docker writes to the `redis-data` volume instead. The volume persists independently of the container's lifecycle.

### Lifecycle

| Command | Containers | Volume |
|---|---|---|
| `docker compose up` | Created / started | Created if new |
| `docker compose down` | Stopped and removed | **Preserved** ✅ |
| `docker compose down --volumes` | Stopped and removed | **Deleted** ❌ |

### Inspect the volume

```bash
# List all volumes on your system
docker volume ls

# Show detailed information about the redis-data volume
docker volume inspect cloudboard_redis-data
```

---

## 9. Docker Compose Command Reference

```bash
# ── Start ──────────────────────────────────────────────────────────────────

# Build images and start all services (foreground — shows all logs)
docker compose up --build

# Start in the background (detached mode)
docker compose up --build -d

# Start only specific services
docker compose up app redis

# ── Stop ───────────────────────────────────────────────────────────────────

# Stop and remove containers (volumes preserved)
docker compose down

# Stop, remove containers AND volumes (full wipe)
docker compose down --volumes

# ── Logs ───────────────────────────────────────────────────────────────────

# Follow all logs
docker compose logs -f

# Follow logs for one service
docker compose logs -f app

# ── Exec (run commands inside a running container) ─────────────────────────

# Open an interactive shell in the app container
docker compose exec app sh

# Run redis-cli inside the Redis container
docker compose exec redis redis-cli

# ── Inspect ────────────────────────────────────────────────────────────────

# List running services and their status
docker compose ps

# Show port mappings
docker compose port app 5000

# Inspect the named volume
docker volume inspect cloudboard_redis-data

# ── Build ──────────────────────────────────────────────────────────────────

# Rebuild the app image without cache (use after code changes)
docker compose build --no-cache app

# ── Scale (advanced) ───────────────────────────────────────────────────────
# Run 3 instances of the Flask app (requires a load balancer in front)
docker compose up --scale app=3
```

---

## 10. Exploring Redis Live

Open a `redis-cli` session inside the Redis container:

```bash
docker compose exec redis redis-cli
```

Useful commands to try while interacting with CloudBoard:

```
# List ALL keys in Redis
KEYS *

# Watch the visitor counter change in real time (visit the home page)
WATCH stats:visitor_count
GET stats:visitor_count

# See the full student list
LRANGE students:names 0 -1

# Count how many students are in the list
LLEN students:names

# Read the last registered student
GET stats:last_student

# Get information about all keys and their types
TYPE stats:visitor_count
TYPE students:names

# Manually set a value
SET stats:visitor_count 100

# Delete a specific key
DEL stats:visitor_count

# Wipe everything (same as the Reset page)
FLUSHALL
```

You can also use the **Redis Commander** web GUI at [http://localhost:8081](http://localhost:8081) for a visual view.

---

## 11. Troubleshooting

### Port already in use

```
Error: Bind for 0.0.0.0:5000 failed: port is already allocated
```

**Fix:** Another process is using port 5000 (common on macOS: AirPlay Receiver).

Option A — change the host port in `.env`:
```env
FLASK_PORT=8080
```
Then access CloudBoard at `http://localhost:8080`.

Option B — stop the process using port 5000:
```bash
# macOS/Linux
lsof -ti :5000 | xargs kill

# Windows (PowerShell)
netstat -ano | findstr :5000
taskkill /PID <PID> /F
```

---

### App shows "Redis Disconnected"

The Flask app cannot reach Redis. Common causes:

1. **Redis container hasn't started yet** — wait a few seconds and refresh.
2. **Wrong `REDIS_HOST`** — make sure it's `redis` (the service name), not `localhost`.
3. **Redis container crashed** — check logs:
   ```bash
   docker compose logs redis
   ```
4. **Containers are on different networks** — check that both `app` and `redis` are in `compose.yaml` under `cloudboard-net`.

---

### Redis Commander shows "Error connecting"

Redis Commander takes a few seconds to connect after startup. Wait 10 seconds and refresh. If it still fails:

```bash
docker compose logs redis-commander
docker compose restart redis-commander
```

---

### Changes to Python code don't appear

The image must be rebuilt after code changes:

```bash
docker compose build app
docker compose up -d app
```

Or do both at once:

```bash
docker compose up --build app
```

---

### Container exits immediately

Check the logs for the error:

```bash
docker compose logs app
```

Common causes:
- Syntax error in `app.py` — Python will print the error and exit.
- Missing package in `requirements.txt`.
- Port conflict (see above).

---

### Clear everything and start fresh

```bash
# Stop all containers and remove volumes
docker compose down --volumes

# Remove built images
docker compose down --rmi local

# Remove dangling images/containers from other projects (optional)
docker system prune
```

---

## 12. Hands-On Exercises

Work through these exercises in order. Each one reinforces a different Docker Compose concept.

---

### Exercise 1 — First Launch (5 min)

**Goal:** Get the stack running and verify all three services are healthy.

1. Run `docker compose up --build` and watch the output.
2. Open [http://localhost:5000](http://localhost:5000) — verify the home page loads.
3. Open [http://localhost:8081](http://localhost:8081) — verify Redis Commander loads.
4. Run `docker compose ps` and record the status of each service.

**Questions:**
- Which service started last? Why?
- What does the `--build` flag do? What happens if you omit it?

---

### Exercise 2 — Service Discovery (10 min)

**Goal:** Understand how containers find each other by name.

1. Open a shell inside the Flask container:
   ```bash
   docker compose exec app sh
   ```
2. Run the following commands and note the output:
   ```sh
   ping redis -c 3
   nslookup redis
   ```
3. What IP address does `redis` resolve to?
4. Exit the shell (`exit`), then open a shell in the Redis container:
   ```bash
   docker compose exec redis sh
   ```
5. Can the Redis container reach the Flask app? Try: `ping app -c 3`

**Questions:**
- What does Docker use to resolve service names to IP addresses?
- Why don't we use `localhost` instead of `redis` for `REDIS_HOST`?
- What would happen if you renamed the `redis` service to `cache` in `compose.yaml`?

---

### Exercise 3 — Redis Data Exploration (10 min)

**Goal:** Observe Redis state change as you interact with CloudBoard.

1. Open Redis Commander at [http://localhost:8081](http://localhost:8081).
2. Open CloudBoard at [http://localhost:5000](http://localhost:5000).
3. Visit the home page 5 times and watch the visitor counter change.
4. Register 3 student names via the Register page.
5. In Redis Commander (or `redis-cli`), run `KEYS *` and record all keys.
6. Check the type of each key: `TYPE <key>`

**Questions:**
- What Redis type is `students:names`? What type is `stats:visitor_count`?
- What command is used to read all elements of a list?
- Run `GET stats:visitor_count` then visit the home page and run it again. What changed?

---

### Exercise 4 — Volume Persistence (10 min)

**Goal:** Prove that data survives container restarts.

1. Register your name on the Register page.
2. Note the visitor count on the home page.
3. Stop and remove the containers (but **not** the volume):
   ```bash
   docker compose down
   ```
4. Check the volume still exists:
   ```bash
   docker volume ls
   ```
5. Bring the stack back up:
   ```bash
   docker compose up --build
   ```
6. Visit the home page — is your data still there?

Now repeat, but this time delete the volume:
```bash
docker compose down --volumes
docker compose up --build
```

**Questions:**
- What happened to the data the second time?
- What is the difference between `docker compose down` and `docker compose down --volumes`?
- Where does Docker store named volume data on your machine? (`docker volume inspect cloudboard_redis-data`)

---

### Exercise 5 — Environment Variables (10 min)

**Goal:** Customise the application without changing source code.

1. Create a `.env` file:
   ```bash
   cp .env.example .env
   ```
2. Edit `.env` and change `APP_NAME` and `APP_GREETING` to something personalised.
3. Restart the stack:
   ```bash
   docker compose up --build
   ```
4. Verify your changes appear on the home page.

Now change the port:
1. In `.env`, set `FLASK_PORT=8080`
2. Restart: `docker compose up --build`
3. Access CloudBoard at [http://localhost:8080](http://localhost:8080)

**Questions:**
- Where is the `.env` file read? Does it need to be specified anywhere?
- What does the `${FLASK_PORT:-5000}` syntax mean in `compose.yaml`?
- Which environment variable controls where Flask looks for Redis?

---

### Exercise 6 — Stopping Individual Services (5 min)

**Goal:** Observe what happens when Redis becomes unavailable.

1. While the stack is running, stop only the Redis service:
   ```bash
   docker compose stop redis
   ```
2. Visit [http://localhost:5000](http://localhost:5000) — what do you see?
3. Navigate to the Register and Students pages — what happens?
4. Restart Redis:
   ```bash
   docker compose start redis
   ```
5. Refresh CloudBoard — does it recover automatically?

**Questions:**
- How does the Flask app communicate that Redis is unavailable?
- Why does the app not crash when Redis goes down?
- What is the `restart: unless-stopped` policy doing in `compose.yaml`?

---

### Exercise 7 — Inspecting the Built Image (5 min)

**Goal:** Understand what `docker compose build` actually produces.

1. Build the image and then inspect it:
   ```bash
   docker compose build app
   docker images | grep cloudboard
   ```
2. List the layers of the image:
   ```bash
   docker history cloudboard-app
   ```
3. Look at the image size — what contributes most to it?

**Questions:**
- Why do we `COPY requirements.txt` separately before `COPY . .`?
- What does `--no-cache` do when building?
- Why is the base image `python:3.12-slim` instead of `python:3.12`?

---

### Exercise 8 — Logs & Debugging (5 min)

**Goal:** Learn to use logs to debug container issues.

1. Follow all logs in real time:
   ```bash
   docker compose logs -f
   ```
2. In another terminal, visit CloudBoard several times. Watch the log entries appear.
3. Filter to just the Flask app logs:
   ```bash
   docker compose logs -f app
   ```
4. Introduce a deliberate error: in `compose.yaml`, change `REDIS_HOST` from `redis` to `wrong-host`, then restart:
   ```bash
   docker compose up --build -d
   docker compose logs app
   ```
5. Fix the hostname and restart.

**Questions:**
- What HTTP method and path appear in the Flask logs for each page visit?
- How does the app behave vs. how do the logs look when Redis is unreachable?

---

### Bonus Exercise — Scale the App (15 min)

**Goal:** Run multiple Flask instances and observe load distribution.

> ⚠️ This requires an understanding of Compose scaling. It works without a load balancer for demonstration purposes — only the last instance bound to port 5000 will receive traffic.

1. Scale the app service:
   ```bash
   docker compose up --scale app=3 --no-recreate
   ```
2. Run `docker compose ps` — how many `app` containers are running?
3. Notice the container `hostname` on the CloudBoard home page. Refresh several times.
4. Open a `redis-cli` session and run `GET stats:visitor_count` — every Flask instance writes to the same Redis.

**Questions:**
- Why does the hostname change on every refresh when scaling?
- How does Redis handle simultaneous `INCR` calls from multiple app instances?
- What would you need to add to properly load balance across multiple Flask instances?

---

## 13. Project Structure

```
cloudboard/
│
├── compose.yaml            ← Defines all three services, networks, and volumes
├── .env.example            ← Template for environment variable customisation
├── .gitignore
├── README.md               ← You are here
│
└── app/                    ← Flask application (built by compose.yaml)
    ├── Dockerfile          ← Builds the Flask image
    ├── requirements.txt    ← Python dependencies (Flask, redis-py, Gunicorn)
    ├── app.py              ← Main Flask application
    │
    ├── templates/          ← Jinja2 HTML templates
    │   ├── base.html       ← Master layout (navigation, footer)
    │   ├── home.html       ← Home page with live status cards
    │   ├── register.html   ← Student registration form
    │   ├── students.html   ← Registered students list
    │   ├── stats.html      ← Statistics dashboard
    │   └── reset.html      ← Data reset page
    │
    └── static/
        └── style.css       ← Responsive CSS stylesheet
```

---

## License

This project is released for educational use. Fork it, break it, rebuild it — that's the point.

---

*Built for Docker Compose workshops. Happy containerising!* 🐳
