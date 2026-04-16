# DevOps Lecture Guide — Docker Compose + CI/CD
## Total Time: ~2 Hours

---

## THE PROJECT

A **Tasks REST API** with three services:

```
Browser
  │
  ▼ :80
[ Nginx ]  ← reverse proxy
  │
  ▼ :3000
[ Node.js API ]  ← our application
  │
  ▼ :5432
[ PostgreSQL ]  ← database
```

All three containers are defined in `docker-compose.yml` and wired together automatically.

---

## FILE MAP

```
devops-lecture-project/
├── app/
│   ├── src/
│   │   ├── index.js          ← Express REST API + DB logic
│   │   └── health.test.js    ← automated test (used in CI)
│   ├── package.json
│   ├── Dockerfile            ← multi-stage build (READ THIS FIRST)
│   └── .dockerignore
│
├── nginx/
│   ├── nginx.conf            ← reverse proxy config
│   └── Dockerfile            ← custom nginx image
│
├── docker-compose.yml        ← dev environment (all 3 services)
├── docker-compose.prod.yml   ← production OVERRIDES
│
├── .github/
│   └── workflows/
│       └── ci-cd.yml         ← GitHub Actions pipeline
│
├── .env.example              ← copy → .env, fill in secrets
└── .gitignore
```

---

## HOUR 1 — DOCKER & DOCKER COMPOSE (60 min)

### Part A: The Dockerfile (20 min)

**Open: `app/Dockerfile`**

Key concepts to cover:

| Instruction | Why |
|-------------|-----|
| `FROM node:20-alpine` | Base image — alpine = tiny Linux |
| `WORKDIR /app` | Set working directory inside container |
| `COPY package*.json ./` | Copy package files BEFORE source code |
| `RUN npm ci --omit=dev` | Install deps (locked, reproducible) |
| `COPY src/ ./src/` | Copy code AFTER deps (cache optimization) |
| **Stage 2** | Fresh image, copy only built artifacts |
| `USER node` | Never run as root |
| `EXPOSE 3000` | Documentation only — not a firewall rule |
| `HEALTHCHECK` | Docker monitors app health |
| `CMD ["node", "src/index.js"]` | Exec form = PID 1 = receives signals |

**Live demo commands:**
```bash
cd app
docker build -t tasks-api:v1 .
docker images | grep tasks-api          # see the image + size
docker run --rm -p 3000:3000 tasks-api:v1   # run it (DB will fail, that's ok)
docker inspect tasks-api:v1             # inspect layers
docker history tasks-api:v1             # see every layer + size
```

**Show layer caching:**
```bash
# Change a line in src/index.js, rebuild
docker build -t tasks-api:v2 .
# Watch: COPY package*.json → CACHED  (fast!)
# Only the COPY src/ step rebuilds
```

---

### Part B: .dockerignore (5 min)

**Open: `app/.dockerignore`**

Demo:
```bash
# Without .dockerignore:
docker build --no-cache -t tasks-api:big .
# Docker sends node_modules (100k files) as context → slow

# Ask: what would happen if .env got into the image?
docker run tasks-api:v1 cat /app/.env    # they'd see the password!
```

---

### Part C: The Nginx Dockerfile & Config (10 min)

**Open: `nginx/Dockerfile` then `nginx/nginx.conf`**

Key concepts:
- Why a reverse proxy? (security, SSL, load balancing)
- `upstream nodejs_backend { server app:3000; }` — Docker DNS!
- `proxy_pass http://nodejs_backend` — where traffic goes
- Security headers (X-Frame-Options, X-Content-Type-Options)
- `daemon off;` — why containers need foreground processes

---

### Part D: Docker Compose (25 min)

**Open: `docker-compose.yml`**

Walk through each section:

#### Services
| Setting | Concept |
|---------|---------|
| `image: postgres:15-alpine` | Pull from Docker Hub (no build) |
| `build: context: ./app` | Build from local Dockerfile |
| `environment:` | Inject env vars (use `.env` for secrets) |
| `depends_on: condition: service_healthy` | Startup ordering |
| `volumes: postgres_data:/var/...` | Persist DB data |
| `volumes: ./app/src:/app/src` | Live code sync (dev only) |
| `ports: "80:80"` | host:container mapping |
| `networks: tasks_network` | Isolated private network |
| `restart: unless-stopped` | Auto-restart policy |

#### Live demo:
```bash
# Setup
cp .env.example .env
# (explain: never commit .env)

# Start everything
docker compose up --build

# In another terminal:
docker compose ps                        # see all running containers
docker compose logs -f app               # tail app logs
docker compose logs db                   # see DB logs

# Test the API through Nginx
curl http://localhost/health
curl -X POST http://localhost/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Learn Docker Compose"}'
curl http://localhost/tasks

# Show containers can find each other by name
docker compose exec app ping db          # DNS resolution!
docker compose exec app wget -qO- http://db:5432 || true

# Scale the app (load balancing concept)
docker compose up --scale app=2 -d

# Stop everything
docker compose down
docker compose down -v    # also delete the volume (data gone!)
```

#### Named Volumes vs Bind Mounts:
```bash
docker volume ls                         # see the postgres_data volume
docker volume inspect devops-lecture-project_postgres_data
# Show: it lives on the host filesystem, survives container removal
```

---

### Part E: Production Override (5 min)

**Open: `docker-compose.prod.yml`**

Diff from dev:
- Uses pre-built image (`image: user/tasks-api:abc1234`) not `build:`
- No `ports: 3000` (app hidden behind Nginx)
- No bind mounts (code is baked in)
- `NODE_ENV: production`
- `replicas: 2` (two app instances)
- Resource `limits` (CPU/memory caps)

```bash
# How to run in prod:
docker compose \
  -f docker-compose.yml \
  -f docker-compose.prod.yml \
  config    # preview the MERGED config (no surprises)
```

---

## HOUR 2 — CI/CD WITH GITHUB ACTIONS (60 min)

### Part A: Concepts (10 min)

Draw on the board:

```
Developer pushes to GitHub
          │
          ▼
   GitHub detects .github/workflows/ci-cd.yml
          │
          ▼
   Spins up fresh Ubuntu VM (runner)
          │
    ┌─────┴──────┐
    ▼            ▼
  Job: test    (waits for test)
  - checkout
  - install deps
  - run tests  ──── FAIL? Pipeline stops. No deploy.
    │
    ▼ (tests pass)
  Job: build-and-push
  - docker build app
  - docker build nginx
  - docker push to Docker Hub
    │
    ▼
  Job: deploy
  - SSH into server
  - docker compose pull
  - docker compose up -d
  - curl /health (verify)
```

---

### Part B: The Workflow File (25 min)

**Open: `.github/workflows/ci-cd.yml`**

Walk through each section:

#### Triggers (`on:`)
| Trigger | When |
|---------|------|
| `push: branches: [main]` | Every push to main |
| `pull_request: branches: [main]` | Every PR to main |
| `workflow_dispatch` | Manual button in UI |

#### Job: `test`
| Step | Why |
|------|-----|
| `actions/checkout@v4` | Clone the repo onto the runner |
| `actions/setup-node@v4` | Install correct Node version |
| `cache: npm` | Speed up: reuse downloaded packages |
| `npm ci` | Reproducible install |
| `npm test` | Run Jest — FAILS = pipeline stops |
| `services: postgres` | Real DB for integration tests |

#### Job: `build-and-push`
| Step | Why |
|------|-----|
| `needs: test` | Only runs if tests pass |
| `if: github.event_name == 'push'` | Skip on PRs |
| `docker/setup-buildx-action` | Enable BuildKit (faster builds) |
| `docker/login-action` | Auth to Docker Hub via secrets |
| `docker/build-push-action` | Build + tag + push in one step |
| `cache-from/cache-to: registry` | Layer cache stored in Docker Hub |

**Image tagging strategy:**
```
tasks-api:abc1234   ← git SHA — EXACT version (never changes)
tasks-api:main      ← latest on main branch
tasks-api:latest    ← latest overall
```

#### Job: `deploy`
| Step | Why |
|------|-----|
| `needs: [test, build-and-push]` | Both must pass |
| `if: github.ref == 'refs/heads/main'` | Only from main branch |
| `environment: production` | Protection rules, audit trail |
| `appleboy/ssh-action` | SSH into server + run commands |
| `docker compose pull` | Fetch new images |
| `docker compose up -d --no-deps` | Restart with new images |
| `curl -f /health` | Verify deploy succeeded |
| `docker image prune -f` | Clean up old images |

---

### Part C: GitHub Secrets (10 min)

**In GitHub UI: Settings → Secrets and variables → Actions**

Add these secrets:
```
DOCKER_HUB_USERNAME   = your Docker Hub username
DOCKER_HUB_TOKEN      = Docker Hub access token (not password!)
SERVER_HOST           = your server IP
SERVER_USER           = ubuntu  (or ec2-user, root, etc.)
SERVER_SSH_KEY        = contents of your ~/.ssh/id_rsa
```

**Why tokens not passwords?**
- Tokens can be revoked without changing your account password
- Tokens can have limited scope (read-only, repo-specific)
- GitHub encrypts them — even repo admins can't read them

---

### Part D: Live Demo (15 min)

```bash
# 1. Push a change and watch the pipeline
git add .
git commit -m "feat: add tasks API"
git push origin main

# 2. Go to GitHub → Actions tab
# Show the pipeline running in real time

# 3. Break a test, push, show the pipeline STOP at test job
# Edit health.test.js: expect(res.body.status).toBe("broken")
git add . && git commit -m "test: intentional failure" && git push
# Watch the pipeline fail — build-and-push SKIPPED, deploy SKIPPED

# 4. Fix the test, push again — full pipeline succeeds
```

---

## KEY TAKEAWAYS

1. **Dockerfile layers are cached** — order instructions from least to most frequently changed
2. **Multi-stage builds** — build tools stay out of production images
3. **Never hardcode secrets** — `.env` locally, GitHub Secrets in CI/CD
4. **Docker Compose = orchestration for one machine** — replaces multiple `docker run` commands
5. **Named volumes persist data** — bind mounts sync code in dev
6. **Container networking** — services find each other by service NAME (Docker DNS)
7. **CI/CD gate** — tests must pass before anything gets built or deployed
8. **Pin versions** — `node:20-alpine` not `node:latest`, `@v4` not `@latest`

---

## USEFUL COMMANDS CHEAT SHEET

```bash
# Docker
docker build -t myapp:v1 .           # build image
docker run --rm -p 3000:3000 myapp   # run container
docker images                         # list images
docker ps                             # list running containers
docker logs -f <container>            # tail logs
docker exec -it <container> sh        # shell into container
docker inspect <container/image>      # detailed info
docker system prune -a                # clean EVERYTHING (careful!)

# Docker Compose
docker compose up --build             # build + start all services
docker compose up -d                  # start in background
docker compose down                   # stop + remove containers
docker compose down -v                # also remove volumes
docker compose ps                     # status of all services
docker compose logs -f <service>      # logs of one service
docker compose exec <service> sh      # shell into a service
docker compose scale app=3            # run 3 app instances

# Production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
docker compose config                 # preview merged config
```
