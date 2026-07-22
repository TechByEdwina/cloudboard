# Docker Compose Hands-on Lab
## CloudBoard — CloudSpace Academy

---

## Background

CloudSpace Academy needs to deploy a web application called **CloudBoard** — a simple student registration board used during workshops.

The application has two components: a Python Flask web app, and a Redis database that stores all application data. There is also a third tool, Redis Commander, that gives you a visual view of what is stored inside Redis.

Rather than starting each piece manually with individual `docker run` commands, your job is to wire everything together using a single **Docker Compose** configuration.

---

## What You Are Working With

The repository already contains the Flask application code inside the `app/` folder. A `Dockerfile` is provided — it describes how to build the Flask application into a Docker image. You do not need to modify it, but read through it to understand what it does.

```
cloudboard/
├── app/
│   ├── Dockerfile          ← builds the Flask image (already written)
│   ├── requirements.txt    ← Python dependencies
│   ├── app.py              ← the Flask application
│   ├── templates/          ← HTML page templates
│   └── static/             ← CSS stylesheet
├── .env.example            ← environment variable reference
└── LAB.md                  ← this file
```

Your task is to create the missing piece: **`compose.yaml`** in the project root.

---

## Objective

Deploy the complete CloudBoard application stack using Docker Compose, so that all three services start with a single command and communicate with each other correctly.

---

## The Three Services

| Service | What it does | How to run it |
|---|---|---|
| `app` | The Flask web application | Build from `./app` using the provided Dockerfile |
| `redis` | Stores all application data | Use the `redis:alpine` image |
| `redis-commander` | Web UI for browsing Redis data | Use the `rediscommander/redis-commander` image |

---

## Your Tasks

### 1. Create `compose.yaml`

In the project root, create a file called `compose.yaml` and define the three services above.

Your configuration should cover all of the following:

**Services**
- Build the `app` service from the local `./app` directory
- Pull `redis` and `redis-commander` from their respective images
- Set `app` and `redis-commander` to depend on `redis` starting first

**Ports**
- Expose the Flask app on port `5000`
- Expose Redis Commander on port `8081`
- Redis does not need to be exposed to the host

**Environment variables**

The Flask app reads its configuration from environment variables. Set the following on the `app` service:

| Variable | Value to set |
|---|---|
| `APP_NAME` | `CloudBoard` |
| `APP_GREETING` | A welcome message of your choice |
| `REDIS_HOST` | The hostname the app uses to reach Redis |
| `REDIS_PORT` | `6379` |
| `FLASK_PORT` | `5000` |

Redis Commander needs one variable to know where Redis is:

| Variable | Value |
|---|---|
| `REDIS_HOSTS` | `local:redis:6379` |

**Volumes**
- Create a named volume called `redis-data`
- Mount it on the `redis` service at `/data`

**Networks** *(optional but recommended)*
- Define a custom bridge network and attach all three services to it

---

### 2. Start the Stack

Once your `compose.yaml` is ready:

```bash
docker compose up --build
```

Open the application in your browser:
- Flask app → http://localhost:5000
- Redis Commander → http://localhost:8081

---

### 3. Validate Your Setup

Work through this checklist before moving on:

- [ ] All three containers are running (`docker compose ps`)
- [ ] The home page loads and shows **Redis: Connected**
- [ ] You can register a student name on the Register page
- [ ] The student appears on the Students page
- [ ] The Stats page shows correct counts
- [ ] Redis Commander shows the keys your app has created
- [ ] Run `docker compose down`, then `docker compose up` — your data is still there

---

## Things to Keep in Mind

**Do not hardcode IP addresses.** Docker assigns container IPs dynamically. Use service names instead — when services share a network, Docker resolves names like `redis` to the correct container automatically.

**`REDIS_HOST` should be the service name**, not `localhost`. Setting it to `localhost` means "this container", not "the Redis container".

**Named volumes persist across restarts.** Running `docker compose down` removes containers but keeps the volume. Running `docker compose down --volumes` removes everything including the data.

**`depends_on` controls start order, not readiness.** It ensures Redis starts before the Flask app, but does not guarantee Redis is ready to accept connections yet. For this lab that is fine — the app handles the connection gracefully and recovers on its own.

---

## Checking Redis Directly

You can open a Redis CLI session inside the running Redis container at any point:

```bash
docker compose exec redis redis-cli
```

Useful commands once inside:

```
KEYS *                          list all keys
GET stats:visitor_count         read the visitor counter
LRANGE students:names 0 -1      list all registered students
TYPE students:names             check the data type of a key
```

---

## Validation Checklist

Before asking for help, go through these:

| Check | How to verify |
|---|---|
| All containers running | `docker compose ps` |
| App loads in browser | http://localhost:5000 |
| Redis connected | Home page status card shows Connected |
| Data persists on restart | `down` then `up` without `--volumes` |
| Redis Commander works | http://localhost:8081 shows your keys |

---

## Docker Compose Features Used in This Lab

Your finished `compose.yaml` should demonstrate all of the following:

- `build` — building a local image from a Dockerfile
- `image` — pulling a pre-built image from Docker Hub
- `ports` — publishing container ports to the host
- `environment` — passing configuration into containers
- `depends_on` — controlling service start order
- `volumes` — persisting data with a named volume

---

## Troubleshooting

**The app shows Redis as disconnected**
Check that `REDIS_HOST` is set to `redis` (the service name), not `localhost`. Also confirm both services are on the same network.

**Port already in use**
Something else on your machine is already using port 5000 or 8081. Either stop the other process, or change the host-side port mapping in your `compose.yaml` (e.g., `8080:5000`).

**Redis Commander shows a connection error**
It sometimes takes a few seconds on first start. Wait and refresh. If it still fails, check that the `REDIS_HOSTS` variable is set correctly and that Redis Commander is on the same network as Redis.

**Changes to app code are not showing up**
Rebuild the image: `docker compose build app`, then restart: `docker compose up -d app`.

**Starting fresh**
```bash
docker compose down --volumes   # removes containers and the data volume
docker compose up --build       # rebuild and start clean
```
