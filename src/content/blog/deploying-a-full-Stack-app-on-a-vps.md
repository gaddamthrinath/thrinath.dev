---
title: "Deploying a Full-Stack App on a VPS — The Right Way"
description: "If you're deploying your first real project, or you've deployed before but never felt confident about *why* you're doing what you're doing — this is for you"
category: "backend"
tags: ["docker", "nginx", "full-stack", "deployment", "vps"]
publishedAt: 2025-04-22
---

This is the complete walkthrough. Not a sanitized tutorial — the actual thing, with the real decisions, the security reasoning, and the mistakes worth warning you about.

## Before We Touch the Terminal — Understanding the Architecture

I have Choonse this delpoyment Architecture bcoz i want the security and isolation of environments and also cheap to maintain. exmaple if the one node Package get compromized and give shell to container then it cant access the other container or host or mongoDB. this is called isolation of environment. or one server crash no impact on other servers. its a very good practise in production level. if you want to scale you can scale this architecture easly. for this Archietecture we need minimal of 4GB ram 2 Core Cpu. but here i have used 8GB Ram and 4 Core Cpu.

Most deployment guides jump straight into commands. That's a mistake. If you don't understand what you're building, you can't debug it when something breaks at 2 AM.

So let's talk about the architecture first.

### What We're Building

```
Internet
    │
    ▼
[Nginx]  ← the only door into your server
    │
    ├── yourapp.in          → React frontend (production)
    ├── admin.yourapp.in    → Admin panel (production)
    └── staging.yourapp.in  → React frontend (staging)

Frontends and Backedns are running in Docker containers not in vps directly as i want the isolation of environments Nginx will forward the requests to the containers acting as reverse proxy.

MongoDB is never directly reachable from outside. MongoDB is running in docker containers on internal ports with docker networks so only the Node backend container can access the mongoDB container through docker network.
```

## Why Docker?

When you run your app directly on the VPS without Docker, you're one bad `npm install` away from breaking your server. Your app's Node.js version conflicts with another app's. A misconfigured environment variable leaks into a process it wasn't meant for. Dependencies accumulate on the host OS and you lose track of what's actually needed.

Docker gives each application a completely isolated box. The box has its own filesystem, its own environment variables, its own process space. Your frontend container has no idea the backend container exists, and the backend container has no idea what OS version the host is running.

It also means deployment becomes predictable. The same Docker image that works in staging will work in production — same Node version, same dependencies, same everything.

### Why Nginx in Front of Everything?

Your containers run on internal ports — 3000, 5000, 3001, 5001. None of those ports are publicly accessible.

Nginx is the only process that listens on port 443 (HTTPS). When a request comes in for `yourapp.in`, Nginx reads the domain name from the request header and decides which container to forward it to. This is called a **reverse proxy**.

This gives you several things for free:

- **SSL in one place.** You configure HTTPS once in Nginx. Your containers don't need to know about certificates at all — they just serve plain HTTP on internal ports.
- **Multiple domains on one server.** You can host ten different apps on the same VPS. Nginx routes each domain to the right container.
- **A security boundary.** Your containers are only reachable through Nginx. You can control exactly what's exposed.

### Why Production AND Staging on the Same Server?

For a small project or early-stage startup, running a separate staging server costs money you probably don't need to spend. The smarter move is to isolate environments at the container level — staging gets its own containers, its own database, its own environment variables — all on the same machine.

The only shared resources are the VPS hardware and the Nginx process. Everything else is completely separate.

**What staging gives you:**

- You can push breaking changes to staging without touching production
- You can test migrations against a real database before running them in production
- You can share a staging URL with clients or QA without giving them access to production data
- basically before going to production you can test your changes on production like environment on same server. if any error or issue found then you can fix it on staging environment. then after you can deploy it to production environment.

The key rule: **staging and production must never share a database.** Ever. Run a migration that wipes a table in staging and it's annoying. Run the same migration in production and it's a disaster.

### Security Architecture — Why This Layout Protects You

Let's be specific about what this setup protects against.

**MongoDB is never exposed to the internet.**

In the Docker Compose file, MongoDB has no `ports` mapping. That means there is no host port — you can't reach MongoDB from outside the server at all. Not from the internet, not through UFW, not through Nginx. The only way to talk to MongoDB is from another container on the same internal Docker network.

This matters because MongoDB's default configuration has historically had weak authentication. There are bots constantly scanning the internet for exposed MongoDB ports. With this setup, they'll never find yours.

**Containers bind to `127.0.0.1`, not `0.0.0.0`.**

Your frontend and backend containers are configured like this:

```yaml
ports:
  - "127.0.0.1:3000:3000"
```

That `127.0.0.1` prefix means the port only binds to the localhost network interface — it's only reachable from the same machine. Without it, Docker would bind to `0.0.0.0`, which means every network interface, which means the internet can reach your container directly on port 3000, bypassing Nginx and your SSL entirely.

**UFW blocks everything except what you explicitly allow.**

Your server accepts connections on exactly three ports: 22 (SSH), 80 (HTTP, needed for SSL certificate verification), and 443 (HTTPS). Nothing else. Port 27017 (MongoDB), port 5000 (backend), port 3000 (frontend) — all blocked at the firewall level.

**Fail2Ban watches for brute-force attacks.**

Every server on the internet gets SSH login attempts within minutes of being created. Bots try thousands of username/password combinations. Fail2Ban watches these attempts and blocks the attacking IP address after a few failures. It's not fancy, but it eliminates a huge class of automated attacks.

**Containers run as a non-root user.**

The VPS itself is accessed via the `ubuntu` user, not `root`. Docker operations run as `ubuntu`. This means even if an attacker somehow gained code execution inside a container, they'd be running as `ubuntu` — a user with limited privileges — not as `root` with access to everything.

---

## Let's Build It

Now that you understand why each piece exists, let's set it up.

---

## Phase 1 — VPS Provisioning

### The Server

```
RAM      : 8 GB
CPU      : 4 vCPU
OS       : Ubuntu 24.04 LTS
```

Ubuntu 24.04 LTS is the right choice here. LTS means Long Term Support — five years of security patches until 2029. You're not going to rebuild your server every year to stay on a supported OS version.

### First Login

```bash
ssh ubuntu@YOUR_SERVER_IP
```

Hostinger provisions the server with an `ubuntu` user. You'll use `sudo` for anything that needs elevated privileges.

### Update First, Always

```bash
sudo apt update
sudo apt upgrade -y
```

The package list on a fresh server is days old. `apt update` downloads the current list of available packages — it doesn't install anything. `apt upgrade -y` then upgrades everything to the latest version. Do this before installing anything else.

---

## Phase 2 — Security Hardening

Don't skip this. Don't do it later. Do it before anything else is running.

### Firewall

```bash
sudo apt install ufw -y

sudo ufw allow OpenSSH    # SSH — allow this BEFORE enabling, or you lock yourself out
sudo ufw allow 80/tcp     # HTTP — needed for SSL certificate verification
sudo ufw allow 443/tcp    # HTTPS — your actual application traffic

sudo ufw enable
sudo ufw status
```

The order matters. Allow SSH first, then enable the firewall. If you enable first, your existing SSH session stays open but new connections are blocked — and you've locked yourself out permanently.

Expected output after `ufw status`:

```
Status: active

To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere
80/tcp                     ALLOW       Anywhere
443/tcp                    ALLOW       Anywhere
```

Three rules. Nothing else.

### Brute-Force Protection

```bash
sudo apt install fail2ban -y
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
sudo fail2ban-client status
```

`systemctl enable` means "start this service automatically when the server reboots." `systemctl start` means "start it right now." Always do both.

---

## Phase 3 — Docker

We install Docker from Docker's own official repository, not Ubuntu's default package repo. Ubuntu's version is often months behind.

```bash
# Required for HTTPS package sources
sudo apt install ca-certificates curl -y

# Create directory for package signing keys
sudo install -m 0755 -d /etc/apt/keyrings

# Download Docker's GPG signing key
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add Docker's repository
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

### Add Your User to the Docker Group

```bash
sudo usermod -aG docker ubuntu
```

Without this, every `docker` command requires `sudo`. Adding `ubuntu` to the `docker` group lets you run Docker without it. This also means your GitHub Actions runner (which runs as `ubuntu`) can execute Docker commands without needing elevated privileges.

**This only takes effect after you log out and back in:**

```bash
exit
ssh ubuntu@YOUR_SERVER_IP  # reconnect
docker compose version     # should work without sudo
```

### Enable Docker on Boot

```bash
sudo systemctl enable docker
```

Without this, Docker stops when the server reboots. Your containers won't restart. Don't forget this step.

---

## Phase 4 — Nginx

Same approach — use the official Nginx repository for the latest stable version.

```bash
sudo apt install curl gnupg2 ca-certificates lsb-release ubuntu-keyring -y

curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor \
  | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null

echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
https://nginx.org/packages/mainline/ubuntu $(lsb_release -cs) nginx" \
  | sudo tee /etc/apt/sources.list.d/nginx.list

# Pin nginx.org as preferred source so apt upgrade doesn't downgrade you
echo -e "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900\n" \
  | sudo tee /etc/apt/preferences.d/99nginx

sudo apt update
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

Visit your server's IP in a browser — you should see the Nginx welcome page.

---

## Phase 5 — Domain DNS

In your domain registrar's DNS panel, create these A records — all pointing to the same server IP:

```
A    yourapp.in           →  YOUR_SERVER_IP
A    www.yourapp.in       →  YOUR_SERVER_IP
A    admin.yourapp.in     →  YOUR_SERVER_IP
A    staging.yourapp.in   →  YOUR_SERVER_IP
```

They all point to the same IP. Nginx figures out which container to route to based on the domain name in the request — that's the whole point of a reverse proxy.

Check propagation:

```bash
dig yourapp.in +short
```

DNS can take anywhere from 2 minutes to 48 hours. In practice, most registrars propagate within 5–10 minutes.

---

## Phase 6 — Directory Structure

A clean directory structure is worth spending 5 minutes on. You'll thank yourself the first time you need to debug something at midnight.

```bash
mkdir -p ~/apps/yourapp/prod
mkdir -p ~/apps/yourapp/staging

# Production service directories
mkdir -p ~/apps/yourapp/prod/fe
mkdir -p ~/apps/yourapp/prod/be
mkdir -p ~/apps/yourapp/prod/admin

# Staging service directories
mkdir -p ~/apps/yourapp/staging/fe
mkdir -p ~/apps/yourapp/staging/be
mkdir -p ~/apps/yourapp/staging/admin
```

Your final layout:

```
~/apps/
└── yourapp/
    ├── prod/
    │   ├── fe/                ← frontend repo lives here
    │   ├── be/                ← backend repo lives here
    │   ├── admin/             ← admin panel repo lives here
    │   ├── docker-compose.yml ← defines all prod containers
    │   └── .env.backend       ← production secrets (never in git)
    └── staging/
        ├── fe/
        ├── be/
        ├── admin/
        ├── docker-compose.yml
        └── .env.backend       ← staging secrets, different values
```

The `docker-compose.yml` and `.env.backend` live at the environment root, not inside the individual service directories. Compose needs to see all services from one location.

---

## Phase 7 — Clone Your Repositories

### Set Up GitHub SSH Access

```bash
ssh-keygen -t ed25519 -C "your@email.com"
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub
```

Copy that public key. Add it in GitHub under **Settings → SSH and GPG Keys → New SSH Key**.

Test it:

```bash
ssh -T git@github.com
# Hi yourusername! You've successfully authenticated...
```

### Clone Into Each Directory

```bash
# Production (main branch)
cd ~/apps/yourapp/prod/fe
git clone git@github.com:YOUR_ORG/yourapp-frontend.git .

cd ~/apps/yourapp/prod/be
git clone git@github.com:YOUR_ORG/yourapp-backend.git .

cd ~/apps/yourapp/prod/admin
git clone git@github.com:YOUR_ORG/yourapp-admin.git .

# Staging (develop branch)
cd ~/apps/yourapp/staging/fe
git clone -b develop git@github.com:YOUR_ORG/yourapp-frontend.git .

cd ~/apps/yourapp/staging/be
git clone -b develop git@github.com:YOUR_ORG/yourapp-backend.git .

cd ~/apps/yourapp/staging/admin
git clone -b develop git@github.com:YOUR_ORG/yourapp-admin.git .
```

The `.` at the end clones into the current directory instead of creating a new subdirectory inside it.

---

> ### ⚠️ Warning: Dockerfile Git Conflicts
>
> Here's something that will catch you if you're not careful.
>
> You create `Dockerfile` and `.dockerignore` directly on the server inside the cloned repo directory. Then you do a `git pull` — and Git throws an error because those files exist locally but don't exist in the remote branch. Git won't overwrite them and won't know what to do.
>
> **Why this happens:** You created the files on the server before committing them to the repo. The server's copy of the repo now has untracked or modified files that conflict with what GitHub is trying to pull.
>
> **The fix for next time:** Always create `Dockerfile` and `.dockerignore` on your local machine first, commit and push them to the repo, then clone on the server. The server should only ever receive files via `git pull` — never create deployment-related files manually inside a cloned repo on the server.
>
> **If you're already in this situation:**
>
> ```bash
> # Check what's conflicting
> git status
>
> # If the files on the server are the correct ones, add them to git
> git add Dockerfile .dockerignore
> git commit -m "Add docker configuration"
> git push origin main
>
> # Then on the server, pull again — it'll be clean
> git pull origin main
> ```
>
> The cleaner long-term habit: treat the server's repo directories as read-only except for `git pull`. Configuration, Dockerfiles, CI workflows — all of that lives in the repo and arrives via git.

---

## Phase 8 — Environment Variables

### Production `.env.backend`

```bash
nano ~/apps/yourapp/prod/.env.backend
```

```env
NODE_ENV=production
PORT=5000

# MongoDB connection string
# See note below about special characters in passwords
MONGO_URI=mongodb://admin:YOUR_PASSWORD@yourapp-prod-mongodb:27017/yourapp?authSource=admin

JWT_SECRET=your-minimum-64-character-random-secret-here
JWT_EXPIRES_IN=7d

SMTP_HOST=smtp.mailgun.org
SMTP_PORT=587
SMTP_USER=postmaster@yourapp.in
SMTP_PASSWORD=your-smtp-password

FRONTEND_URL=https://yourapp.in
ADMIN_URL=https://admin.yourapp.in
```

### Staging `.env.backend`

```bash
nano ~/apps/yourapp/staging/.env.backend
```

```env
NODE_ENV=staging
PORT=5000

MONGO_URI=mongodb://admin:YOUR_PASSWORD@yourapp-staging-mongodb:27017/yourapp_staging?authSource=admin

JWT_SECRET=completely-different-secret-for-staging
JWT_EXPIRES_IN=7d

SMTP_HOST=smtp.mailgun.org
SMTP_PORT=587
SMTP_USER=postmaster@yourapp.in
SMTP_PASSWORD=your-smtp-password

FRONTEND_URL=https://staging.yourapp.in
```

Notice the MongoDB hostname changes between environments — `yourapp-prod-mongodb` vs `yourapp-staging-mongodb`. That's the container name. Docker's internal DNS resolves container names to their IP addresses automatically, so your app can just use the container name as the hostname.

### Lock Down the Files

```bash
chmod 600 ~/apps/yourapp/prod/.env.backend
chmod 600 ~/apps/yourapp/staging/.env.backend
```

`chmod 600` means only the owner can read or write the file. Other users on the system cannot read it.

### The Special Character Trap in MongoDB URIs

This one will silently break your connection and drive you insane trying to debug it.

A MongoDB connection string has this structure:

```
mongodb://USERNAME:PASSWORD@HOST:PORT/DATABASE
```

The `@` symbol is the separator between credentials and host. If your password contains `@`, the connection string parser splits in the wrong place so it better to use password without special characters or use percent-encode special characters in passwords i.e in url encode the special characters.

**Example — password `password@123`:**

```
mongodb://admin:password@123@yourapp-prod-mongodb:27017/yourapp
                 ↑
         Parser sees this @ as the credential/host separator
         "cm@yourapp-prod-mongodb" becomes the host — connection fails
```

**Fix: percent-encode special characters in passwords**

```
@  →  %40
#  →  %23
$  →  %24
!  →  %21
```

Your URI becomes:

```env
MONGO_URI=mongodb://admin:password%40cm@yourapp-prod-mongodb:27017/yourapp?authSource=admin
```

If your backend can't connect to MongoDB and the error isn't obviously about credentials, check this first.

---

## Phase 9 — Dockerfiles

### Frontend and Admin Panel

Both the user-facing frontend and admin panel are React apps. Same Dockerfile pattern for both.

```dockerfile
# Stage 1: Build
# Full Node.js environment to compile the React app
FROM node:22-alpine AS builder

WORKDIR /app

# Copy package files first — Docker caches this layer separately
# If package.json hasn't changed, npm ci is skipped on the next build
COPY package*.json ./

# npm ci = exact install from package-lock.json
# Faster and more reproducible than npm install in CI/CD and Docker builds
RUN npm ci

COPY . .

# Compile React → static HTML/CSS/JS in /app/dist
RUN npm run build


# Stage 2: Runtime
# Discard everything from Stage 1 — source code, node_modules, all of it
# Only keep the compiled output
FROM node:22-alpine

WORKDIR /app

RUN npm install -g serve

COPY --from=builder /app/dist ./dist

EXPOSE 3000

# -s: single-page app mode — 404s redirect to index.html for React Router
CMD ["serve", "-s", "dist", "-l", "3000"]
```

If your admin panel has peer dependency conflicts during build, use `--legacy-peer-deps` in the build stage:

```dockerfile
RUN npm install --legacy-peer-deps
```

### Backend

TypeScript backend — needs compilation before running.

```dockerfile
# Stage 1: Build
FROM node:22-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

# tsc compiles TypeScript to JavaScript → /app/dist
RUN npm run build


# Stage 2: Production runtime
FROM node:22-alpine

WORKDIR /app

COPY package*.json ./

# --omit=dev skips typescript, ts-node, jest, nodemon etc.
# Production only needs the compiled JS — no TypeScript tooling
RUN npm ci --omit=dev

COPY --from=builder /app/dist ./dist
COPY public_media ./public_media

ENV NODE_ENV=production

EXPOSE 5000

CMD ["npm", "start"]
```

### `.dockerignore`

Create this in each service directory. It tells Docker what to exclude when building — without it, your entire `node_modules` folder (potentially 300MB) gets sent to the Docker daemon every build.

```
node_modules
npm-debug.log
dist
build
.git
.gitignore
.env
.env.*
.DS_Store
.vscode
```

---

## Phase 10 — Docker Compose

### Production

```bash
nano ~/apps/yourapp/prod/docker-compose.yml
```

```yaml
services:

  mongodb:
    image: mongo:8
    container_name: yourapp-prod-mongodb
    restart: unless-stopped

    environment:
      # Creates the root user on first startup only
      # If the data volume already exists, these are ignored
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: YOUR_MONGO_PASSWORD

    volumes:
      - mongodb_data:/data/db

    networks:
      - backend-network

    # No ports: section here
    # MongoDB is completely unreachable from outside this Docker network
    # This is intentional — it's the most important security decision in this file

    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s

  backend:
    build:
      context: ./be

    container_name: yourapp-prod-backend
    restart: unless-stopped

    depends_on:
      mongodb:
        condition: service_healthy
        # Waits until MongoDB passes its health check before starting
        # Without this, backend can start before MongoDB is ready and crash

    env_file:
      - .env.backend

    ports:
      # 127.0.0.1 prefix = localhost only
      # Only Nginx (running on the same machine) can reach this port
      # Without the prefix, Docker exposes it to 0.0.0.0 = the internet
      - "127.0.0.1:5000:5000"

    volumes:
      - ./be/public_media:/app/public_media

    networks:
      - backend-network

    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:5000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 20s

  frontend:
    build:
      context: ./fe

    container_name: yourapp-prod-frontend
    restart: unless-stopped

    ports:
      - "127.0.0.1:3000:3000"

    networks:
      - frontend-network

    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s

  admin:
    build:
      context: ./admin

    container_name: yourapp-prod-admin
    restart: unless-stopped

    ports:
      - "127.0.0.1:3002:3000"

    networks:
      - frontend-network

    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s

volumes:
  mongodb_data:

networks:
  backend-network:
    driver: bridge

  frontend-network:
    driver: bridge
```

### A Note on Health Checks

The `healthcheck` blocks tell Docker how to verify each container is actually working — not just running. There's a big difference.

A container can be in `Up` state but completely broken — the process started but the app crashed, or the database isn't accepting connections yet. Without health checks, `docker ps` shows green and you have no idea something is wrong.

With health checks:

- `docker ps` shows `healthy` or `unhealthy` status
- `depends_on: condition: service_healthy` means the backend waits for MongoDB to be genuinely ready before starting, not just running
- Docker restarts `unhealthy` containers if you configure it to

For the backend health check to work, you need a `/api/health` endpoint in your Express app:

```javascript
app.get('/api/health', (req, res) => {
  res.status(200).json({ status: 'ok' });
});
```

### Staging

```bash
nano ~/apps/yourapp/staging/docker-compose.yml
```

```yaml
services:

  mongodb:
    image: mongo:8
    container_name: yourapp-staging-mongodb
    restart: unless-stopped
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: YOUR_MONGO_PASSWORD
    volumes:
      - mongodb_staging_data:/data/db
    networks:
      - backend-network
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s

  backend:
    build:
      context: ./be
    container_name: yourapp-staging-backend
    restart: unless-stopped
    depends_on:
      mongodb:
        condition: service_healthy
    env_file:
      - .env.backend
    ports:
      - "127.0.0.1:5001:5000"   # 5001 on host — no conflict with prod's 5000
    volumes:
      - ./be/public_media:/app/public_media
    networks:
      - backend-network
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:5000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 20s

  frontend:
    build:
      context: ./fe
    container_name: yourapp-staging-frontend
    restart: unless-stopped
    ports:
      - "127.0.0.1:3001:3000"   # 3001 on host
    networks:
      - frontend-network
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s

  admin:
    build:
      context: ./admin
    container_name: yourapp-staging-admin
    restart: unless-stopped
    ports:
      - "127.0.0.1:3003:3000"   # 3003 on host
    networks:
      - frontend-network
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s

volumes:
  mongodb_staging_data:

networks:
  backend-network:
    driver: bridge
  frontend-network:
    driver: bridge
```

**Port allocation at a glance:**

```
Production:
  frontend  →  127.0.0.1:3000
  backend   →  127.0.0.1:5000
  admin     →  127.0.0.1:3002
  mongodb   →  not exposed

Staging:
  frontend  →  127.0.0.1:3001
  backend   →  127.0.0.1:5001
  admin     →  127.0.0.1:3003
  mongodb   →  not exposed
```

---

## Phase 11 — MongoDB Authentication

When a MongoDB container starts for the first time with an empty volume, it initializes using the `MONGO_INITDB_ROOT_*` variables and creates the root user.

On every subsequent restart, the volume already has an initialized database — those environment variables are ignored. Changing the password in `docker-compose.yml` after first run does nothing. You'd need to change it inside MongoDB directly.

Your backend's connection string needs the root credentials and the `authSource` parameter:

```env
MONGO_URI=mongodb://admin:password@yourapp-prod-mongodb:27017/yourapp?authSource=admin
```

`authSource=admin` is required because the root user is stored in MongoDB's `admin` database, not in `yourapp`. Without it, MongoDB looks for the user in the wrong database and authentication fails.

Verify it's working:

```bash
docker exec -it yourapp-prod-mongodb mongosh \
  -u admin -p YOUR_PASSWORD --authenticationDatabase admin

# Inside mongosh
show dbs
```

---

## Phase 12 — First Deployment

```bash
cd ~/apps/yourapp/prod

# Build all images
docker compose build

# Start everything in the background
docker compose up -d
```

Then check:

```bash
docker ps
```

You want to see all four containers with `Up` status. After a minute or two, once health checks run, they should show `(healthy)`.

```
CONTAINER ID   IMAGE           STATUS                    NAMES
ca6da3ba97db   prod-admin      Up 2 min (healthy)        yourapp-prod-admin
b757323c089e   prod-backend    Up 2 min (healthy)        yourapp-prod-backend
4cb403b27950   prod-frontend   Up 2 min (healthy)        yourapp-prod-frontend
3d1c3230d2a5   mongo:8         Up 2 min (healthy)        yourapp-prod-mongodb
```

Test locally before touching Nginx:

```bash
curl http://127.0.0.1:3000    # should return HTML
curl http://127.0.0.1:5000/api/health  # should return {"status":"ok"}
curl http://127.0.0.1:3002    # should return admin HTML
```

If any of these fail, look at the logs before moving on:

```bash
docker compose logs -f backend
```

---

## Phase 13 — Nginx Configuration

All Nginx configuration for yourapp goes in a single file:

```bash
sudo nano /etc/nginx/conf.d/yourapp.conf
```

```nginx
# ─── yourapp.in — User frontend + production API ─────────────────────────
server {
    server_name yourapp.in www.yourapp.in;

    client_max_body_size 50M;

    # All /api/* requests go to the backend container
    location /api/ {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Everything else goes to the frontend container
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/yourapp.in/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourapp.in/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}


# ─── admin.yourapp.in — Admin frontend + same production API ──────────────
server {
    server_name admin.yourapp.in;

    client_max_body_size 50M;

    # Admin uses the same backend as the user frontend
    # Your backend differentiates admin vs user via JWT roles/claims
    location /api/ {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location / {
        proxy_pass http://127.0.0.1:3002;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/yourapp.in/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourapp.in/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}


# ─── staging.yourapp.in — Staging frontend + staging API ─────────────────
server {
    server_name staging.yourapp.in;

    client_max_body_size 50M;

    location /api/ {
        proxy_pass http://127.0.0.1:5001;  # staging backend
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location / {
        proxy_pass http://127.0.0.1:3001;  # staging frontend
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/yourapp.in/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourapp.in/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}


# ─── HTTP → HTTPS redirects ───────────────────────────────────────────────────
server {
    listen 80;
    server_name yourapp.in www.yourapp.in;
    if ($host = www.yourapp.in) { return 301 https://$host$request_uri; }
    if ($host = yourapp.in) { return 301 https://$host$request_uri; }
    return 404;
}

server {
    listen 80;
    server_name admin.yourapp.in;
    if ($host = admin.yourapp.in) { return 301 https://$host$request_uri; }
    return 404;
}

server {
    listen 80;
    server_name staging.yourapp.in;
    if ($host = staging.yourapp.in) { return 301 https://$host$request_uri; }
    return 404;
}
```

Test and reload:

```bash
sudo nginx -t       # always test before reloading
sudo systemctl reload nginx
```

`reload` vs `restart` — always use `reload` for config changes. It gracefully replaces worker processes without dropping active connections. `restart` kills everything immediately.

---

## Phase 14 — SSL Certificates

```bash
sudo apt install certbot python3-certbot-nginx -y

sudo certbot --nginx \
  -d yourapp.in \
  -d www.yourapp.in \
  -d admin.yourapp.in \
  -d staging.yourapp.in
```

Certbot will ask for your email (for renewal notices), agree to terms, and then verify each domain by placing a challenge file at `/.well-known/acme-challenge/` and having Let's Encrypt fetch it over port 80. That's why we allowed port 80 in UFW.

After it finishes, it automatically modifies your Nginx config to add the SSL certificate paths and HTTP→HTTPS redirect blocks.

Test auto-renewal:

```bash
sudo certbot renew --dry-run
```

If the dry run succeeds, your certificates will renew automatically every 60 days. You never need to touch this again.

---

## Phase 15 — Self-Hosted GitHub Actions Runners

This is the part that surprises people coming from SSH-based deployment tutorials.

Instead of GitHub Actions SSHing into your server to run commands, you run a small **runner process** directly on the VPS. The runner connects outbound to GitHub, listens for jobs, and executes them locally when triggered.

**Why this is better than the SSH approach:**

| Self-hosted runner | SSH action |
|---|---|
| No private key stored in GitHub secrets | Requires storing your server's private key |
| Runner has direct filesystem access | Commands get sent as strings over SSH |
| Reuses local Docker layer cache | Slightly slower per connection |
| Runs as `ubuntu` user — correct permissions already set | Need to configure SSH user permissions separately |

You need one runner process per repository. Three repos (fe, be, admin) means three runner processes, each in its own directory.

### Runner Directory Structure

```bash
mkdir -p ~/actions-runner                          # frontend runner
mkdir -p ~/yourapp-be-runner/actions-runner    # backend runner
mkdir -p ~/yourapp-admin-runner/actions-runner # admin runner
```

### Register Each Runner

The steps are identical for each. Get a fresh registration token for each runner from GitHub: **Repo → Settings → Actions → Runners → New self-hosted runner**.

Tokens expire after 1 hour — generate a new one each time you register.

**Frontend runner:**

```bash
cd ~/actions-runner

curl -o actions-runner-linux-x64-2.334.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.334.0/actions-runner-linux-x64-2.334.0.tar.gz

# Verify the download wasn't corrupted
echo "048024cd2c848eb6f14d5646d56c13a4def2ae7ee3ad12122bee960c56f3d271  actions-runner-linux-x64-2.334.0.tar.gz" \
  | shasum -a 256 -c

tar xzf ./actions-runner-linux-x64-2.334.0.tar.gz

./config.sh \
  --url https://github.com/YOUR_ORG/yourapp-frontend \
  --token YOUR_TOKEN_HERE
# When prompted for a name, use something descriptive: yourapp-frontend-vps

sudo ./svc.sh install ubuntu   # install as a systemd service running as ubuntu user
sudo ./svc.sh start
sudo ./svc.sh status
```

**Backend runner:**

```bash
cd ~/yourapp-be-runner/actions-runner

curl -o actions-runner-linux-x64-2.334.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.334.0/actions-runner-linux-x64-2.334.0.tar.gz

echo "048024cd2c848eb6f14d5646d56c13a4def2ae7ee3ad12122bee960c56f3d271  actions-runner-linux-x64-2.334.0.tar.gz" \
  | shasum -a 256 -c

tar xzf ./actions-runner-linux-x64-2.334.0.tar.gz

./config.sh \
  --url https://github.com/YOUR_ORG/yourapp-backend \
  --token YOUR_TOKEN_HERE

sudo ./svc.sh install ubuntu
sudo ./svc.sh start
```

**Admin runner:**

```bash
cd ~/yourapp-admin-runner/actions-runner

curl -o actions-runner-linux-x64-2.334.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.334.0/actions-runner-linux-x64-2.334.0.tar.gz

echo "048024cd2c848eb6f14d5646d56c13a4def2ae7ee3ad12122bee960c56f3d271  actions-runner-linux-x64-2.334.0.tar.gz" \
  | shasum -a 256 -c

tar xzf ./actions-runner-linux-x64-2.334.0.tar.gz

./config.sh \
  --url https://github.com/YOUR_ORG/yourapp-admin \
  --token YOUR_TOKEN_HERE

sudo ./svc.sh install ubuntu
sudo ./svc.sh start
```

After registering all three, go to each repo's **Settings → Actions → Runners** — you should see all three showing as **Idle** (green).

---

## Phase 16 — GitHub Actions Workflows

Create a workflow file in each repository at `.github/workflows/deploy.yml`.

### Frontend (`yourapp-frontend` repo)

```yaml
name: Deploy Frontend

on:
  push:
    branches:
      - main      # triggers prod deploy
      - develop   # triggers staging deploy

jobs:
  deploy:
    runs-on: self-hosted  # uses YOUR runner on the VPS, not GitHub's cloud

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Deploy
        run: |
          set -e

          # Choose environment based on which branch was pushed
          if [ "${{ github.ref_name }}" = "main" ]; then
            COMPOSE_DIR="$HOME/apps/yourapp/prod"
          else
            COMPOSE_DIR="$HOME/apps/yourapp/staging"
          fi

          cd "$COMPOSE_DIR"

          # Pull latest code
          cd fe && git pull origin ${{ github.ref_name }} && cd ..

          # Rebuild and restart only the frontend container
          # --no-deps: don't touch mongodb, backend, or admin
          # --build: force a fresh image build
          docker compose up -d --no-deps --build frontend

          # Clean up old images to save disk space
          docker image prune -f

          echo "✓ Frontend deployed to $COMPOSE_DIR"
```

### Backend (`yourapp-backend` repo)

```yaml
name: Deploy Backend

on:
  push:
    branches:
      - main
      - develop

jobs:
  deploy:
    runs-on: self-hosted

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Deploy
        run: |
          set -e

          if [ "${{ github.ref_name }}" = "main" ]; then
            COMPOSE_DIR="$HOME/apps/yourapp/prod"
          else
            COMPOSE_DIR="$HOME/apps/yourapp/staging"
          fi

          cd "$COMPOSE_DIR"
          cd be && git pull origin ${{ github.ref_name }} && cd ..
          docker compose up -d --no-deps --build backend
          docker image prune -f

          echo "✓ Backend deployed to $COMPOSE_DIR"
```

### Admin (`yourapp-admin` repo)

```yaml
name: Deploy Admin

on:
  push:
    branches:
      - main
      - develop

jobs:
  deploy:
    runs-on: self-hosted

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Deploy
        run: |
          set -e

          if [ "${{ github.ref_name }}" = "main" ]; then
            COMPOSE_DIR="$HOME/apps/yourapp/prod"
          else
            COMPOSE_DIR="$HOME/apps/yourapp/staging"
          fi

          cd "$COMPOSE_DIR"
          cd admin && git pull origin ${{ github.ref_name }} && cd ..
          docker compose up -d --no-deps --build admin
          docker image prune -f

          echo "✓ Admin deployed to $COMPOSE_DIR"
```

Push these workflow files to your repos. The next time you push to `main` or `develop`, the corresponding environment deploys automatically.

---

## Phase 17 — Backups

### Backup Script

```bash
mkdir -p ~/backups
nano ~/backups/backup-mongo.sh
```

```bash
#!/bin/bash
set -e

CONTAINER="yourapp-prod-mongodb"
DB_NAME="yourapp"
BACKUP_DIR="$HOME/backups"
DATE=$(date +%Y-%m-%d_%H-%M-%S)

echo "[$DATE] Starting backup..."

docker exec "$CONTAINER" mongodump \
  -u admin \
  -p YOUR_MONGO_PASSWORD \
  --authenticationDatabase admin \
  --db "$DB_NAME" \
  --out "/tmp/mongodump-$DATE"

docker cp "$CONTAINER:/tmp/mongodump-$DATE" "$BACKUP_DIR/mongo-$DB_NAME-$DATE"

cd "$BACKUP_DIR"
tar -czf "mongo-$DB_NAME-$DATE.tar.gz" "mongo-$DB_NAME-$DATE"
rm -rf "mongo-$DB_NAME-$DATE"

# Delete backups older than 7 days
find "$BACKUP_DIR" -name "mongo-*.tar.gz" -mtime +7 -delete

echo "[$DATE] Backup complete: mongo-$DB_NAME-$DATE.tar.gz"
```

```bash
chmod +x ~/backups/backup-mongo.sh
~/backups/backup-mongo.sh  # test it manually first
```

### Schedule with Cron

```bash
crontab -e
```

```cron
# Daily MongoDB backup at 2:00 AM
0 2 * * * /home/ubuntu/backups/backup-mongo.sh >> /home/ubuntu/backups/backup.log 2>&1
```

---

## Phase 18 — Monitoring

```bash
# Container status + health
docker ps

# Live resource usage
docker stats

# Stream logs from a specific container
docker logs -f yourapp-prod-backend

# Last 100 lines
docker logs --tail 100 yourapp-prod-backend

# Nginx errors
sudo tail -f /var/log/nginx/error.log

# Disk usage
df -h
docker system df
```

---

## What's Running Now

```
Push to main (any repo)
        │
        ▼
GitHub sends job to self-hosted runner on VPS
        │
        ▼
Runner pulls latest code + rebuilds container
        │
        ▼
New container starts, old one stops
        │
        ▼
Nginx keeps routing traffic — no downtime, no manual steps
```

---

## Things Worth Adding (But Not Required to Start)

You have a working production deployment. Everything below is optional — things to consider as your project grows.

---

### Log Rotation

Docker logs every container's stdout/stderr to JSON files on disk. By default, there's no limit. Leave a busy app running for a few months and you might wake up to a full disk.

Fix it globally by creating `/etc/docker/daemon.json`:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

This caps each container's logs at 10MB × 3 files = 30MB max. When a file hits 10MB, Docker rotates to a new one and deletes the oldest.

Apply it:

```bash
sudo systemctl restart docker
# Note: this restarts Docker — containers will stop briefly
```

For an already-running container to pick up the new log settings, you need to recreate it:

```bash
docker compose up -d --force-recreate backend
```

---

### Secrets Management

Right now your secrets live in `.env.backend` files on the server — `JWT_SECRET`, `SMTP_PASSWORD`, MongoDB credentials. For a small project or early-stage startup, this is fine. The files are `chmod 600`, only accessible by the `ubuntu` user.

As your project scales or your team grows, this approach has limitations — you can't audit who accessed a secret, you can't rotate secrets without SSHing into the server, and if someone gets your server access they get all secrets at once.

**The next step up: Docker Secrets**

Docker has a built-in secrets mechanism that mounts secrets as files inside containers rather than environment variables. Environment variables can accidentally leak in logs, crash reports, or child processes. Files are more controlled.

```yaml
# In docker-compose.yml
services:
  backend:
    secrets:
      - jwt_secret
      - smtp_password

secrets:
  jwt_secret:
    file: ./secrets/jwt_secret.txt   # file on host, only readable by container
  smtp_password:
    file: ./secrets/smtp_password.txt
```

Inside the container, secrets are available at `/run/secrets/jwt_secret`.

**The step after that: a dedicated secrets manager**

For teams or compliance requirements, tools like **HashiCorp Vault** or cloud-native options like **AWS Secrets Manager** give you centralized secrets storage, access audit logs, automatic rotation, and fine-grained permissions. Vault in particular works well with Docker and is self-hostable if you want to keep everything on your own infrastructure.

For most indie projects and early-stage startups, `.env` files with proper permissions are completely acceptable. Know the limitation and upgrade when the risk justifies it.

---

That's the complete setup. Production is live, staging is isolated, deployments are automated, and you understand why every piece is there.