# Docker — Complete Guide

## Table of Contents
1. [What is Docker?](#1-what-is-docker)
2. [Docker Architecture](#2-docker-architecture)
3. [Docker Installation](#3-docker-installation)
4. [Docker Images](#4-docker-images)
5. [Dockerfile Reference](#5-dockerfile-reference)
6. [Docker Containers](#6-docker-containers)
7. [Docker Volumes](#7-docker-volumes)
8. [Docker Networking](#8-docker-networking)
9. [Docker Compose](#9-docker-compose)
10. [Docker Registry](#10-docker-registry)
11. [Multi-Stage Builds](#11-multi-stage-builds)
12. [Docker Security](#12-docker-security)
13. [Docker in Production](#13-docker-in-production)
14. [Real-World Use Cases](#14-real-world-use-cases)
15. [Common Interview Questions](#15-common-interview-questions)

---

## 1. What is Docker?

Docker is a platform for packaging, distributing, and running applications in containers.

```
Traditional Deployment:          Containerized Deployment:
─────────────────────────────    ──────────────────────────────────────
App A  App B  App C              App A      App B      App C
  │      │      │                Container  Container  Container
  └──────┴──────┘                   │          │          │
     OS (shared)                    └──────────┴──────────┘
     Hardware                         Docker Engine
                                       OS (shared)
                                       Hardware

Virtual Machines:                Containers:
─────────────────────────────    ──────────────────────────────────────
App  App  App                    App  App  App
 │    │    │                      │    │    │
OS   OS   OS                    Bins Bins Bins  (shared kernel)
 │    │    │                      │    │    │
Hypervisor (thick)               Docker Engine (thin)
Hardware                         Hardware

VMs: Full OS per VM (~GBs)       Containers: Share OS kernel (~MBs)
VMs: Minutes to start            Containers: Milliseconds to start
VMs: Strong isolation            Containers: Process isolation
```

**"Works on my machine" → solved by containers:**
- Package code + dependencies + runtime + config together
- Same container runs identically on any machine with Docker
- Eliminate environment drift between dev/staging/production

---

## 2. Docker Architecture

```
┌────────────────────────────────────────────────────────────┐
│                   Docker Client (CLI)                       │
│            docker build / run / push / pull                 │
└──────────────────────────┬─────────────────────────────────┘
                           │ REST API (unix socket or TCP)
┌──────────────────────────▼─────────────────────────────────┐
│                   Docker Daemon (dockerd)                    │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │   Images    │  │  Containers  │  │    Networks       │  │
│  │  (layers)   │  │  (running    │  │  (bridge, host,   │  │
│  │             │  │   processes) │  │   overlay, none)  │  │
│  └─────────────┘  └──────────────┘  └───────────────────┘  │
│  ┌─────────────┐  ┌──────────────┐                         │
│  │   Volumes   │  │    Plugins   │                         │
│  └─────────────┘  └──────────────┘                         │
└──────────────────────────┬─────────────────────────────────┘
                           │
               ┌───────────▼───────────┐
               │    Container Runtime   │
               │    (containerd, runc)  │
               └───────────────────────┘

Docker Registries (Image Storage):
  Docker Hub      → docker.io/ubuntu:22.04
  AWS ECR         → 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
  GitHub Container Registry → ghcr.io/org/app:v1.0
  Private Registry → registry.company.com/app:latest
```

**Image Layers:**
```
ubuntu:22.04                    (base layer — 30MB)
   + apt-get install python3    (new layer — 50MB)
   + pip install requirements   (new layer — 100MB)
   + COPY app/ /app/            (new layer — 5MB)
   + CMD ["python", "app.py"]   (new layer — tiny)
─────────────────────────────
Final image: ~185MB

Layers are cached and shared:
If two images share ubuntu:22.04, it's stored ONCE on disk
```

---

## 3. Docker Installation

```bash
# Ubuntu
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER   # Add current user to docker group
newgrp docker                   # Reload group (or log out/in)
docker --version
docker run hello-world          # Test installation

# Verify with detailed info
docker info

# Start and enable daemon
sudo systemctl enable --now docker
```

---

## 4. Docker Images

```bash
# List images
docker images                   # or: docker image ls
docker images -a                # Include intermediate layers

# Pull images
docker pull ubuntu:22.04        # Specific tag
docker pull nginx:latest        # Latest tag (avoid in prod!)
docker pull node:18-alpine      # Alpine-based (smaller)

# Image details
docker inspect nginx:latest     # Full JSON metadata
docker history nginx:latest     # Layer history and sizes
docker image inspect --format='{{.Config.Entrypoint}}' nginx

# Remove images
docker rmi nginx:latest         # Remove image
docker rmi -f image-id          # Force remove
docker image prune              # Remove dangling images
docker image prune -a           # Remove all unused images

# Tag images
docker tag myapp:latest registry.company.com/myapp:v1.2.3
docker tag myapp:latest registry.company.com/myapp:latest

# Build
docker build -t myapp:latest .
docker build -t myapp:1.0.0 -f Dockerfile.prod .
docker build --no-cache -t myapp:latest .         # Skip cache
docker build --build-arg VERSION=1.2.3 -t myapp . # Pass build args
```

---

## 5. Dockerfile Reference

### Python Application

```dockerfile
# Dockerfile
FROM python:3.11-slim            # Base image (slim = smaller)

# Set metadata
LABEL maintainer="team@company.com"
LABEL version="1.0"

# Set working directory
WORKDIR /app

# Create non-root user (security best practice)
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

# Install system dependencies (separate layer for caching)
RUN apt-get update && apt-get install -y \
    gcc \
    libpq-dev \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies FIRST (changes less often → cached)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code (changes most often → last)
COPY --chown=appuser:appgroup . .

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PORT=8000

# Expose port (documentation — doesn't actually open port)
EXPOSE $PORT

# Switch to non-root user
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:$PORT/health || exit 1

# Run application
CMD ["python", "-m", "gunicorn", "--bind", "0.0.0.0:8000", "app:app"]
```

### Node.js Application

```dockerfile
FROM node:18-alpine

WORKDIR /app

# Copy package files first (caching!)
COPY package*.json ./
RUN npm ci --only=production     # Use ci for reproducible installs

COPY . .

RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 3000
HEALTHCHECK CMD wget -qO- http://localhost:3000/health || exit 1
CMD ["node", "server.js"]
```

### Java Application (Multi-Stage)

```dockerfile
# Stage 1: Build
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:go-offline -q    # Cache dependencies
COPY src/ src/
RUN mvn clean package -DskipTests

# Stage 2: Runtime (much smaller image)
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy only the built JAR from build stage
COPY --from=builder --chown=appuser:appgroup /build/target/app.jar app.jar

USER appuser
EXPOSE 8080
HEALTHCHECK CMD curl -f http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Dockerfile Best Practices

```dockerfile
# ✅ Order layers from least to most changing (cache optimization)
FROM node:18-alpine
COPY package.json .        # Changes rarely
RUN npm install            # Cached if package.json unchanged
COPY . .                   # Changes often

# ✅ Combine RUN commands to reduce layers
RUN apt-get update \
    && apt-get install -y package1 package2 \
    && rm -rf /var/lib/apt/lists/*

# ❌ DON'T do this (each RUN is a separate layer)
RUN apt-get update
RUN apt-get install -y package1
RUN apt-get install -y package2

# ✅ Use .dockerignore
# .dockerignore file:
# .git/
# node_modules/
# *.log
# .env
# Dockerfile
# README.md

# ✅ Use specific tags (not latest)
FROM node:18.20.3-alpine3.19   # Specific, reproducible

# ✅ Non-root user
USER appuser

# ✅ Read-only filesystem where possible
# In docker-compose or k8s: readOnlyRootFilesystem: true
```

---

## 6. Docker Containers

```bash
# Run containers
docker run nginx                          # Run in foreground
docker run -d nginx                       # Detached (background)
docker run -d --name my-nginx nginx       # Named container
docker run -d -p 8080:80 nginx           # Port mapping (host:container)
docker run -d -p 127.0.0.1:8080:80 nginx  # Bind to localhost only
docker run -it ubuntu bash               # Interactive TTY
docker run --rm ubuntu ls /             # Auto-remove after exit
docker run -e DB_HOST=localhost app     # Environment variable
docker run --env-file .env app          # From env file
docker run -v /host/path:/container/path app  # Bind mount
docker run --memory="512m" --cpus="1.5" app   # Resource limits
docker run --restart unless-stopped app  # Auto-restart policy

# Container management
docker ps                    # Running containers
docker ps -a                 # All containers (including stopped)
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
docker stop container-id     # Graceful stop (SIGTERM → SIGKILL)
docker start container-id    # Start stopped container
docker restart container-id  # Restart
docker kill container-id     # Force kill (SIGKILL)
docker rm container-id       # Remove stopped container
docker rm -f container-id    # Force remove running container

# Exec into running container
docker exec -it container-id bash      # Interactive shell
docker exec container-id ls /app       # Run command
docker exec -u root container-id bash  # As root

# Logs
docker logs container-id                # All logs
docker logs -f container-id            # Follow (like tail -f)
docker logs --tail 100 container-id    # Last 100 lines
docker logs --since 1h container-id    # Last 1 hour
docker logs --since "2024-01-15T10:00:00" container-id

# Inspect and stats
docker inspect container-id            # Full JSON info
docker stats                           # Live resource usage (all)
docker stats container-id              # Specific container
docker top container-id                # Running processes

# Copy files
docker cp file.txt container-id:/app/  # Host → container
docker cp container-id:/app/file.txt . # Container → host

# Prune
docker container prune                 # Remove stopped containers
docker system prune                    # Remove all unused resources
docker system prune -a                 # Also remove unused images
docker system df                       # Docker disk usage
```

---

## 7. Docker Volumes

```bash
# Volume types:
# Named Volume: managed by Docker, lives in /var/lib/docker/volumes/
# Bind Mount: specific host path mapped to container path
# tmpfs: stored in memory (Linux only)

# Named volumes
docker volume create mydata          # Create
docker volume ls                     # List
docker volume inspect mydata         # Details
docker volume rm mydata              # Remove
docker volume prune                  # Remove unused

# Use named volume
docker run -d \
    --name postgres \
    -e POSTGRES_PASSWORD=secret \
    -v postgres-data:/var/lib/postgresql/data \   # Named volume
    postgres:16

# Bind mount
docker run -d \
    -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \  # Read-only
    -v $(pwd)/html:/usr/share/nginx/html \
    nginx

# Share volume between containers
docker run -d --name app1 -v shared:/data myapp
docker run -d --name app2 -v shared:/data myapp
```

---

## 8. Docker Networking

```
Docker Network Types:
─────────────────────────────────────────────────────────────
bridge    → Default. Containers on same bridge can communicate by IP
            (use --name and custom network for DNS resolution)
host      → Container shares host network stack. No port mapping needed.
none      → No network. Isolated container.
overlay   → Multi-host networking (Docker Swarm, Kubernetes)
macvlan   → Container gets its own MAC/IP (appears as physical device)
```

```bash
# Network management
docker network ls                              # List networks
docker network create myapp-network            # Create bridge network
docker network create --driver bridge myapp    # Explicit driver
docker network inspect myapp-network           # Details
docker network rm myapp-network                # Remove

# Connect containers via custom network (enables DNS)
docker network create backend
docker run -d --name db --network backend postgres:16
docker run -d --name api --network backend myapp
# Now 'api' can reach 'db' by hostname: ping db

# Connect to multiple networks
docker network connect frontend api    # api now on both networks

# DNS in custom networks
# Containers can reach each other by container name
# In code: DB_HOST=postgres (not an IP address!)
```

---

## 9. Docker Compose

Docker Compose manages multi-container applications:

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Web application
  app:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        VERSION: ${APP_VERSION:-1.0.0}
    image: myapp:${APP_VERSION:-latest}
    container_name: myapp
    ports:
      - "8080:8000"
    environment:
      - DATABASE_URL=postgresql://postgres:secret@db:5432/myapp
      - REDIS_URL=redis://redis:6379/0
      - SECRET_KEY=${SECRET_KEY}
    env_file:
      - .env
    volumes:
      - ./uploads:/app/uploads
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - frontend
      - backend
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          memory: 256M

  # Database
  db:
    image: postgres:16-alpine
    container_name: postgres
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Cache
  redis:
    image: redis:7-alpine
    container_name: redis
    volumes:
      - redis-data:/data
    networks:
      - backend
    command: redis-server --appendonly yes

  # Reverse proxy
  nginx:
    image: nginx:alpine
    container_name: nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - app
    networks:
      - frontend
    restart: unless-stopped

volumes:
  postgres-data:
  redis-data:

networks:
  frontend:
  backend:
    internal: true    # Not accessible from host network
```

```bash
# Compose commands
docker-compose up -d              # Start all services (detached)
docker-compose up -d app          # Start specific service
docker-compose down               # Stop and remove containers
docker-compose down -v            # Also remove volumes
docker-compose ps                 # Status
docker-compose logs -f app        # Follow logs for service
docker-compose logs -f            # Follow all logs
docker-compose restart app        # Restart service
docker-compose exec app bash      # Shell into service
docker-compose build              # Build images
docker-compose build --no-cache   # Rebuild without cache
docker-compose pull               # Pull latest images
docker-compose scale app=3        # Scale service (replicas)
docker-compose config             # Validate and display config

# Multiple compose files (override pattern)
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## 10. Docker Registry

```bash
# Docker Hub
docker login                                    # Docker Hub
docker push myusername/myapp:v1.0.0

# AWS ECR
aws ecr create-repository --repository-name myapp --region us-east-1
aws ecr get-login-password --region us-east-1 | \
    docker login --username AWS \
    --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com

docker tag myapp:latest 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:latest

# ECR image lifecycle policy (auto-delete old images)
aws ecr put-lifecycle-policy \
    --repository-name myapp \
    --lifecycle-policy-text '{
        "rules": [{
            "rulePriority": 1,
            "description": "Keep last 10 images",
            "selection": {"tagStatus": "any", "countType": "imageCountMoreThan", "countNumber": 10},
            "action": {"type": "expire"}
        }]
    }'

# Private Registry
docker run -d \
    -p 5000:5000 \
    --restart always \
    --name registry \
    -v /mnt/registry:/var/lib/registry \
    registry:2
```

---

## 11. Multi-Stage Builds

```dockerfile
# Golang example — build binary, run in scratch (zero OS)
FROM golang:1.21 AS builder
WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o app ./cmd/server

# Final image — just the binary, no Go toolchain
FROM scratch
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /build/app /app
EXPOSE 8080
ENTRYPOINT ["/app"]
# Final image: ~10MB vs 800MB with Go toolchain
```

---

## 12. Docker Security

```bash
# Security best practices:

# 1. Scan images for vulnerabilities
docker scan myapp:latest
trivy image myapp:latest
# Or: snyk container test myapp:latest

# 2. Run as non-root (in Dockerfile)
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# 3. Read-only root filesystem
docker run --read-only myapp

# 4. Drop Linux capabilities
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE nginx

# 5. Limit resources
docker run --memory="512m" --cpus="1.0" myapp

# 6. No privileged mode (except when necessary)
docker run --privileged myapp   # AVOID — gives root-equivalent access

# 7. Use secrets, not env vars for sensitive data
docker secret create my-password ./password.txt
# Or use Docker BuildKit secrets for Dockerfile:
# RUN --mount=type=secret,id=mysecret,target=/run/secrets/mysecret ...

# 8. Keep base images updated
docker pull ubuntu:22.04  # Pull latest patch
docker build --no-cache .  # Rebuild from fresh base

# 9. Docker Bench Security (automated security scan)
docker run --net host --pid host --userns host --cap-add audit_control \
    -v /etc:/etc:ro -v /usr/bin/containerd:/usr/bin/containerd:ro \
    -v /var/lib:/var/lib:ro -v /var/run/docker.sock:/var/run/docker.sock:ro \
    docker/docker-bench-security
```

---

## 13. Docker in Production

```bash
# Health checks and restart policies
docker run -d \
    --restart unless-stopped \
    --health-cmd "curl -f http://localhost/health" \
    --health-interval 30s \
    --health-retries 3 \
    --health-timeout 10s \
    myapp

# Resource limits
docker run -d \
    --memory 512m \
    --memory-swap 1g \
    --cpus 1.5 \
    --pids-limit 100 \
    myapp

# Logging configuration
docker run -d \
    --log-driver json-file \
    --log-opt max-size=10m \
    --log-opt max-file=3 \
    myapp

# Send logs to CloudWatch
docker run -d \
    --log-driver=awslogs \
    --log-opt awslogs-region=us-east-1 \
    --log-opt awslogs-group=/app/production \
    myapp
```

---

## 14. Real-World Use Cases

### Use Case: Local Dev Environment

```yaml
# docker-compose.dev.yml — full dev environment in one command
version: '3.8'
services:
  app:
    build: .
    volumes:
      - .:/app         # Hot reload — code changes reflected instantly
    ports: ["8000:8000"]
    command: python manage.py runserver 0.0.0.0:8000
    
  db:
    image: postgres:16
    ports: ["5432:5432"]   # Exposed for local debugging
    environment:
      POSTGRES_DB: myapp_dev
      POSTGRES_PASSWORD: devpassword
    
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
    
  mailhog:            # Capture all emails locally
    image: mailhog/mailhog
    ports: ["8025:8025"]
```

```bash
# Developer workflow
docker-compose -f docker-compose.dev.yml up -d
# Everything running: app, postgres, redis, mailhog
# Code changes live-reload
# Emails captured at localhost:8025
```

---

## 15. Common Interview Questions

**Q1: What is the difference between a Docker image and a container?**
> An image is a read-only template (blueprint) built from a Dockerfile, stored in a registry. A container is a running instance of an image — it adds a writable layer on top. Multiple containers can be created from one image. Image:Container :: Class:Object.

**Q2: What is a Docker layer and how does layer caching work?**
> Each instruction in a Dockerfile creates a layer. Layers are cached — if a layer's instruction and context haven't changed, Docker reuses the cached layer. Order instructions from least-changing to most-changing: base OS → system packages → app dependencies → application code.

**Q3: Explain Docker networking — bridge, host, and none.**
> Bridge (default): creates a virtual network; containers communicate by IP; use custom named bridge for DNS. Host: container uses host's network stack directly — better performance, no NAT. None: no network access — for security-critical isolated containers.

**Q4: What is the difference between COPY and ADD in Dockerfile?**
> COPY: copies files/dirs from build context. ADD: does everything COPY does plus auto-extracts TAR files and can fetch from URLs. Prefer COPY for simplicity and predictability. Use ADD only if you specifically need tar extraction.

**Q5: What is the difference between CMD and ENTRYPOINT?**
> ENTRYPOINT: defines the executable that always runs. CMD: provides default arguments to ENTRYPOINT (or default command if no ENTRYPOINT). `docker run myimage arg` replaces CMD but not ENTRYPOINT. Pattern: ENTRYPOINT ["python", "app.py"] + CMD ["--port", "8000"] (port overridable).

**Q6: How do you reduce Docker image size?**
> (1) Use Alpine/slim base images. (2) Multi-stage builds — only copy build artifacts. (3) Combine RUN commands (fewer layers). (4) Clean apt caches in same RUN layer. (5) Use .dockerignore to exclude unnecessary files. (6) Remove dev dependencies.

**Q7: What is Docker Compose and when do you use it?**
> Docker Compose defines and runs multi-container applications via YAML. Use for: local development environments, integration testing, simple deployments. Compose v3 with Swarm for production. For production at scale, prefer Kubernetes. `docker-compose up` starts entire stack.

**Q8: What is a Docker volume and why is it needed?**
> Containers are ephemeral — data is lost when container is removed. Volumes persist data outside the container lifecycle. Named volumes: managed by Docker, best for databases. Bind mounts: host path — best for development (live code changes). Always use volumes for stateful services (databases, uploads).

**Q9: How do you handle secrets in Docker containers?**
> Docker Secrets (Swarm): `docker secret create`. Kubernetes Secrets. External vault (HashiCorp Vault, AWS Secrets Manager) — fetch at startup. Environment variables — NOT recommended for sensitive data (visible in `docker inspect`). Never bake secrets into images.

**Q10: What is the difference between `docker stop` and `docker kill`?**
> `stop`: sends SIGTERM (graceful shutdown signal), waits 10 seconds, then sends SIGKILL. Allows application to cleanup (close DB connections, flush logs). `kill`: sends SIGKILL immediately (or specified signal). Use stop for production; kill for hanging containers.

---

*Next: [Docker Assignments](02-docker-assignments.md)*
