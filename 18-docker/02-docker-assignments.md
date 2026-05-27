# Docker — Assignments

## Assignment 1: Your First Docker Container

```bash
# 1. Pull and run nginx
docker run -d --name my-nginx -p 8080:80 nginx
curl http://localhost:8080  # Should see nginx welcome page

# 2. Customize the index page
docker exec -it my-nginx bash
echo "<h1>Hello from Docker!</h1>" > /usr/share/nginx/html/index.html
exit
curl http://localhost:8080  # Should see custom page

# 3. Inspect the container
docker inspect my-nginx
docker stats my-nginx --no-stream
docker top my-nginx

# 4. View logs
docker logs my-nginx
docker logs -f my-nginx &  # Follow in background
curl http://localhost:8080  # Generate a log entry

# 5. Stop, start, and remove
docker stop my-nginx
docker start my-nginx
docker rm -f my-nginx
```

---

## Assignment 2: Build Your First Docker Image

**Build a Python Flask application:**

```python
# app.py
from flask import Flask, jsonify
import os, datetime, socket

app = Flask(__name__)

@app.route('/health')
def health():
    return jsonify({
        'status': 'healthy',
        'hostname': socket.gethostname(),
        'timestamp': datetime.datetime.utcnow().isoformat(),
        'version': os.environ.get('APP_VERSION', '1.0.0')
    })

@app.route('/api/info')
def info():
    return jsonify({
        'app': 'demo',
        'environment': os.environ.get('APP_ENV', 'development'),
        'memory_limit': os.environ.get('MEMORY_LIMIT', 'not set')
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=int(os.environ.get('PORT', 5000)))
```

```
# requirements.txt
flask==3.0.0
gunicorn==21.2.0
```

**Tasks:**
1. Write a Dockerfile following all best practices (non-root user, health check, .dockerignore)
2. Build the image: `docker build -t myflaskapp:v1 .`
3. Run it: `docker run -d -p 5000:5000 -e APP_ENV=production myflaskapp:v1`
4. Test health endpoint: `curl http://localhost:5000/health`
5. Check the image size: `docker images myflaskapp`
6. Analyze layers: `docker history myflaskapp:v1`
7. Optimize: switch to `python:3.11-slim` and compare sizes

---

## Assignment 3: Multi-Stage Build

**Build a Java Spring Boot application (or Go binary) with a multi-stage Dockerfile:**

**Option A — Go binary (smallest final image):**
```go
// main.go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "os"
)

func healthHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]string{
        "status": "healthy",
        "app": os.Getenv("APP_NAME"),
    })
}

func main() {
    http.HandleFunc("/health", healthHandler)
    fmt.Println("Starting on :8080")
    http.ListenAndServe(":8080", nil)
}
```

**Build multi-stage Dockerfile:**
```dockerfile
# Stage 1: Build (write this)
FROM golang:1.21 AS builder
WORKDIR /build
COPY go.mod ./
# ... your implementation

# Stage 2: Runtime (write this)  
FROM scratch
# ... copy only the binary
```

**Tasks:**
1. Write the complete multi-stage Dockerfile
2. Build and measure image sizes:
   - Without multi-stage: `FROM golang:1.21` only
   - With multi-stage: final FROM scratch
3. Document size comparison
4. Verify the app works: `curl http://localhost:8080/health`

---

## Assignment 4: Docker Compose — Full Application Stack

**Run a WordPress site with Docker Compose:**

```yaml
# docker-compose.yml (write this yourself based on requirements)
# Requirements:
# - WordPress latest
# - MySQL 8.0 database
# - nginx as reverse proxy on port 80
# - All data persisted in named volumes
# - Database credentials from .env file
# - Nginx only accessible on port 80 (not WordPress directly)
# - Health checks on all services
# - Restart policy: unless-stopped
```

**Tasks:**
1. Write the complete `docker-compose.yml`
2. Create `.env` file for secrets
3. Start the stack: `docker-compose up -d`
4. Access WordPress at `http://localhost`
5. Complete WordPress setup
6. Stop and restart: `docker-compose restart`
7. Verify data persists after: `docker-compose down && docker-compose up -d`
8. Scale: run 2 WordPress instances (challenge: what breaks?)

---

## Assignment 5: Docker Networking

**Understand container networking:**

```bash
# 1. Inspect the default bridge network
docker network inspect bridge

# 2. Create a custom network
docker network create myapp-net

# 3. Start a database
docker run -d --name db --network myapp-net \
    -e POSTGRES_DB=mydb \
    -e POSTGRES_USER=user \
    -e POSTGRES_PASSWORD=pass \
    postgres:16-alpine

# 4. Start an app that connects to the db by hostname
docker run -d --name app --network myapp-net \
    -e DATABASE_URL=postgresql://user:pass@db:5432/mydb \
    myflaskapp:v1

# 5. Verify app can reach db by name (not IP)
docker exec app ping -c 2 db

# 6. Verify db is NOT accessible from host (good security)
curl http://db:5432  # Should fail

# 7. Add a second network for frontend
docker network create frontend-net
docker network connect frontend-net app

# 8. Disconnect from a network
docker network disconnect myapp-net app

# Questions to answer:
# - Why can containers on custom network use hostnames?
# - What is the default bridge limitation for DNS?
# - How does host networking differ?
```

---

## Assignment 6: Docker Registry and ECR

**Push and manage images in AWS ECR:**

```bash
# 1. Create ECR repository
aws ecr create-repository \
    --repository-name myflaskapp \
    --region us-east-1

# 2. Tag your local image for ECR
ECR_URL=$(aws ecr describe-repositories \
    --repository-names myflaskapp \
    --query 'repositories[0].repositoryUri' \
    --output text)

docker tag myflaskapp:v1 ${ECR_URL}:v1
docker tag myflaskapp:v1 ${ECR_URL}:latest

# 3. Authenticate Docker to ECR
aws ecr get-login-password --region us-east-1 | \
    docker login --username AWS --password-stdin ${ECR_URL%/*}

# 4. Push to ECR
docker push ${ECR_URL}:v1
docker push ${ECR_URL}:latest

# 5. Verify in ECR
aws ecr list-images --repository-name myflaskapp

# 6. Create lifecycle policy (keep last 10 images)
aws ecr put-lifecycle-policy \
    --repository-name myflaskapp \
    --lifecycle-policy-text '{
        "rules": [{
            "rulePriority": 1,
            "selection": {"tagStatus": "any", "countType": "imageCountMoreThan", "countNumber": 10},
            "action": {"type": "expire"}
        }]
    }'

# 7. Pull from ECR on another machine
docker pull ${ECR_URL}:latest
```

---

## Interview Assignment: Debug a Broken Container

**Scenario:** A production container keeps crashing. Diagnose and fix.

```bash
# Run this broken container
docker run -d --name broken-app \
    -p 8090:8090 \
    -e DB_HOST=nonexistent-host \
    -e DB_PORT=5432 \
    --memory=50m \
    nginx

# Investigation tasks:
# 1. Check why the container stopped
docker ps -a
docker logs broken-app

# 2. Check resource constraints
docker inspect broken-app | jq '.[0].HostConfig.Memory'

# 3. Try to exec into it
docker exec -it broken-app bash  # What happens if it's stopped?

# 4. Check if there are any OOM kills
dmesg | grep -i oom

# For the actual Flask app broken scenario:
docker run -d --name flask-broken \
    -e DB_HOST=localhost \
    -e APP_PORT=abc \
    myflaskapp:v1

# Debug without starting the failing process:
docker run --entrypoint /bin/bash -it myflaskapp:v1

# Write a diagnosis report:
# - What failed
# - Root cause
# - How to fix
# - How to prevent
```

---

## Workflow: Docker in CI/CD Pipeline

**Build the following workflow manually:**

```bash
# Step 1: Build
docker build \
    --build-arg BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
    --build-arg VCS_REF=$(git rev-parse --short HEAD) \
    -t myapp:$(git rev-parse --short HEAD) .

# Step 2: Scan for vulnerabilities
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
    aquasec/trivy:latest image \
    --exit-code 0 \
    --severity HIGH,CRITICAL \
    myapp:$(git rev-parse --short HEAD)

# Step 3: Run container tests
docker-compose -f docker-compose.test.yml up \
    --abort-on-container-exit \
    --exit-code-from test-runner

# Step 4: Tag and push
docker tag myapp:$(git rev-parse --short HEAD) registry/myapp:latest
docker push registry/myapp:$(git rev-parse --short HEAD)
docker push registry/myapp:latest

# Step 5: Cleanup
docker rmi myapp:$(git rev-parse --short HEAD)
docker system prune -f
```

---

## Cheat Sheet

```bash
# Clean up everything (careful!)
docker system prune -a --volumes

# Run temporary debugging container
docker run --rm -it --network container:myapp nicolaka/netshoot

# Docker build with no cache
docker build --no-cache -t myapp:latest .

# Export/import image (transfer without registry)
docker save myapp:latest | gzip > myapp.tar.gz
docker load < myapp.tar.gz

# Copy files from stopped container
docker cp stopped-container:/app/logs ./local-logs

# Get container IP
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' mycontainer

# Follow logs from multiple containers
docker-compose logs -f app db redis

# Check image vulnerabilities (quick)
docker scout cves myapp:latest
```
