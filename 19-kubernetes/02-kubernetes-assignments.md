# Kubernetes — Assignments

## Assignment 1: Get Started with kubectl

```bash
# Setup local cluster (choose one):
# Option A: minikube
minikube start --driver=docker --cpus=2 --memory=4g
minikube addons enable metrics-server
minikube addons enable ingress

# Option B: kind
kind create cluster --name dev --config kind-config.yaml

# Option C: Use EKS (see Module 12)

# Explore the cluster
kubectl cluster-info
kubectl get nodes -o wide
kubectl get namespaces
kubectl get pods -A   # What's running?

# Create your first pod
kubectl run nginx-pod --image=nginx:alpine --port=80
kubectl get pods
kubectl describe pod nginx-pod
kubectl logs nginx-pod

# Access it
kubectl port-forward pod/nginx-pod 8080:80
curl http://localhost:8080  # In another terminal

# Clean up
kubectl delete pod nginx-pod
```

---

## Assignment 2: Deploy a Stateless Application

**Deploy a multi-replica web application:**

```yaml
# Task: Create all these resources
# 1. Namespace: workshop
# 2. Deployment: nginx with 3 replicas
# 3. Service: ClusterIP
# 4. ConfigMap: custom nginx config
# 5. HPA: scale between 2-10 based on CPU 70%

# web-app.yaml — write this yourself with:
# - Deployment with 3 replicas
# - Resource requests: cpu=100m memory=128Mi
# - Resource limits: cpu=500m memory=256Mi
# - Liveness probe on /health every 10s
# - Readiness probe on / every 5s
# - Labels: app=web, tier=frontend
# - Rolling update: maxUnavailable=1, maxSurge=1
```

**Tasks:**
```bash
kubectl apply -f web-app.yaml

# Verify deployment
kubectl get all -n workshop
kubectl rollout status deployment/web -n workshop

# Scale manually
kubectl scale deployment web -n workshop --replicas=5
kubectl get pods -n workshop -w  # Watch pods come up

# Test rolling update
kubectl set image deployment/web web=nginx:1.25 -n workshop
kubectl rollout status deployment/web -n workshop
kubectl rollout history deployment/web -n workshop

# Rollback
kubectl rollout undo deployment/web -n workshop
kubectl rollout status deployment/web -n workshop
```

---

## Assignment 3: Persistent Storage

**Deploy a stateful database:**

```yaml
# Deploy PostgreSQL with persistent storage
# Requirements:
# - StatefulSet with 1 replica
# - PVC: 10Gi gp2 storage
# - Secret for DB password
# - Service: ClusterIP for internal access
# - Liveness probe on postgres port
# - Resource limits

# Create the Secret first
kubectl create secret generic postgres-secret \
    --from-literal=POSTGRES_PASSWORD=mypassword \
    --from-literal=POSTGRES_USER=myuser \
    --from-literal=POSTGRES_DB=mydb \
    --namespace workshop
```

**Write `postgres-statefulset.yaml` and:**
```bash
kubectl apply -f postgres-statefulset.yaml

# Verify
kubectl get statefulset -n workshop
kubectl get pvc -n workshop
kubectl describe pvc -n workshop

# Connect to postgres
kubectl exec -it postgres-0 -n workshop -- \
    psql -U myuser -d mydb

# Inside postgres, create a table
CREATE TABLE test_data (id SERIAL, name VARCHAR(100));
INSERT INTO test_data (name) VALUES ('test record');
SELECT * FROM test_data;
\q

# Delete and recreate pod — data should persist!
kubectl delete pod postgres-0 -n workshop
kubectl get pods -n workshop -w
# Wait for pod to restart

kubectl exec -it postgres-0 -n workshop -- \
    psql -U myuser -d mydb -c "SELECT * FROM test_data;"
# Data should still be there!
```

---

## Assignment 4: Ingress and Service Mesh Basics

**Expose services externally with Ingress:**

```yaml
# Deploy two services and route via Ingress:
# - Service A: api-service → handles /api/*
# - Service B: web-service → handles /*
```

**Deploy and configure:**
```bash
# Install nginx ingress controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml

# Create two deployments
kubectl create deployment api --image=hashicorp/http-echo -- \
    --listen=:8080 --text='{"service":"api"}'
kubectl create deployment web --image=nginx

# Create services
kubectl expose deployment api --port=80 --target-port=8080
kubectl expose deployment web --port=80

# Create the Ingress (write this YAML):
# - Host: myapp.local
# - /api → api service port 80
# - / → web service port 80
# - TLS with self-signed cert

# Test
echo "127.0.0.1 myapp.local" >> /etc/hosts
curl http://myapp.local/api/test
curl http://myapp.local/
```

---

## Assignment 5: ConfigMaps and Secrets

**Practice environment configuration:**

```bash
# 1. Create ConfigMap from literal values
kubectl create configmap app-config \
    --from-literal=APP_ENV=production \
    --from-literal=LOG_LEVEL=INFO \
    --from-literal=MAX_CONNECTIONS=100

# 2. Create ConfigMap from file
cat > nginx.conf << 'EOF'
server {
    listen 80;
    server_name _;
    location /health {
        return 200 "healthy\n";
    }
}
EOF
kubectl create configmap nginx-config --from-file=nginx.conf

# 3. Create Secret
kubectl create secret generic app-secret \
    --from-literal=API_KEY=sk-abc123 \
    --from-literal=DB_PASSWORD=supersecret

# 4. Deploy an app using both ConfigMap and Secret
# Write a deployment that:
# - Mounts nginx-config as a volume
# - Injects app-config as env vars
# - Injects app-secret as env vars (API_KEY, DB_PASSWORD)

# 5. Verify env vars in running pod
kubectl exec -it <pod-name> -- env | grep -E 'APP_ENV|LOG_LEVEL|API_KEY'

# 6. Update ConfigMap without pod restart
kubectl edit configmap app-config
# Change LOG_LEVEL to DEBUG
# Note: Volume-mounted files update within ~60s; env vars do NOT update
```

---

## Assignment 6: RBAC and Security

**Implement role-based access control:**

```bash
# Scenario: Set up RBAC for two teams
# - Team A (developers): read-only access to pods and services in "dev" namespace
# - Team B (ops): full access to "dev" namespace, read-only in "prod"

# 1. Create namespaces
kubectl create namespace dev
kubectl create namespace prod

# 2. Create ServiceAccounts
kubectl create serviceaccount dev-team -n dev
kubectl create serviceaccount ops-team -n dev

# 3. Create Roles (write the YAML):
# dev-reader: get, list, watch on pods and services in "dev"
# ops-admin: full access in "dev" namespace

# 4. Create RoleBindings

# 5. Test RBAC
# Create kubeconfig for dev-team and verify they can't delete pods:
kubectl auth can-i delete pods --namespace=dev \
    --as=system:serviceaccount:dev:dev-team

kubectl auth can-i list pods --namespace=dev \
    --as=system:serviceaccount:dev:dev-team

# 6. Create NetworkPolicy that only allows pods with label app=api
# to accept traffic from pods with label app=frontend
```

---

## Interview Assignment: Production-Grade Deployment

**Deploy a complete production-grade application:**

**Requirements:**
- 3-tier app: nginx → api → postgres
- All pods run as non-root
- Resource requests and limits on all containers
- Liveness and readiness probes on all containers
- Secrets managed with External Secrets Operator (or manually)
- HPA on api deployment (min: 2, max: 10, CPU: 70%)
- Pod Disruption Budget: at least 2 api pods always available
- Anti-affinity: api pods spread across different nodes
- Zero-downtime deployment strategy

**Write all manifests and then verify:**
```bash
# Verify zero-downtime during rolling update
while true; do curl -s http://myapp/health | jq .status; sleep 1; done &
# Then trigger deployment update
kubectl set image deployment/api api=myapi:v2 -n production
# Watch — no failures should occur during update
```

---

## Workflow: Complete Kubernetes CI/CD

```bash
# Build → Test → Push → Deploy pattern

# 1. Build and push (in CI)
docker build -t $ECR_URL/myapp:$GIT_SHA .
docker push $ECR_URL/myapp:$GIT_SHA

# 2. Update image in deployment
kubectl set image deployment/myapp \
    myapp=$ECR_URL/myapp:$GIT_SHA \
    --record -n production

# 3. Watch rollout
kubectl rollout status deployment/myapp -n production --timeout=5m

# 4. Verify new pods are running
kubectl get pods -n production -l app=myapp -o wide

# 5. Run smoke test
kubectl run smoke-test --image=curlimages/curl --rm -it --restart=Never \
    -- curl -f http://myapp-service/health

# 6. If something goes wrong — rollback
kubectl rollout undo deployment/myapp -n production
kubectl rollout status deployment/myapp -n production
```
