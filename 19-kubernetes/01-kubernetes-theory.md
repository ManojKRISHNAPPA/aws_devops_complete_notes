# Kubernetes — Complete Guide

## Table of Contents
1. [What is Kubernetes?](#1-what-is-kubernetes)
2. [Kubernetes Architecture](#2-kubernetes-architecture)
3. [Core Concepts](#3-core-concepts)
4. [Workloads](#4-workloads)
5. [Services and Networking](#5-services-and-networking)
6. [Storage](#6-storage)
7. [Configuration Management](#7-configuration-management)
8. [RBAC and Security](#8-rbac-and-security)
9. [Scheduling and Resource Management](#9-scheduling-and-resource-management)
10. [Helm](#10-helm)
11. [kubectl Commands](#11-kubectl-commands)
12. [EKS — Kubernetes on AWS](#12-eks--kubernetes-on-aws)
13. [Real-World Deployments](#13-real-world-deployments)
14. [Common Interview Questions](#14-common-interview-questions)

---

## 1. What is Kubernetes?

**Kubernetes (K8s)** is an open-source container orchestration platform that automates deploying, scaling, and managing containerized applications.

```
Without Kubernetes:                With Kubernetes:
────────────────────────────────   ────────────────────────────────────
Manual deployment of containers    Declarative desired state management
Manual scaling (hire someone)      Auto-scaling based on CPU/memory
Manual recovery from crashes       Self-healing — restarts failed pods
Manual load balancing              Built-in load balancing & discovery
Different configs per env          Uniform config management
Manual rolling updates             Zero-downtime rolling deployments
Hard to manage 100s of containers  Designed for thousands of containers
```

**Kubernetes Solves:**
- Service discovery and load balancing
- Storage orchestration (EBS, EFS, NFS)
- Automated rollouts and rollbacks
- Self-healing (restart failed containers)
- Secret and config management
- Horizontal scaling
- Batch job execution

---

## 2. Kubernetes Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         CONTROL PLANE                            │
│                                                                  │
│  ┌───────────────┐  ┌──────────────┐  ┌───────────────────────┐ │
│  │  API Server   │  │   Scheduler  │  │  Controller Manager   │ │
│  │  (kube-       │  │  (kube-      │  │  (kube-controller-    │ │
│  │  apiserver)   │  │  scheduler)  │  │   manager)            │ │
│  │               │  │              │  │  - Node Controller    │ │
│  │  All K8s API  │  │  Assign pods │  │  - Deployment Ctrl   │ │
│  │  calls go here│  │  to nodes    │  │  - ReplicaSet Ctrl   │ │
│  └───────┬───────┘  └──────────────┘  │  - Service Ctrl      │ │
│          │                             └───────────────────────┘ │
│  ┌───────▼───────┐  ┌──────────────────────────────────────────┐ │
│  │     etcd      │  │              Cloud Controller            │ │
│  │  (distributed │  │    (cloud-controller-manager)             │ │
│  │   key-value   │  │  - Load Balancer provisioning             │ │
│  │    store)     │  │  - EBS/EFS volume creation               │ │
│  └───────────────┘  └──────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                               │ (kubelets poll API server)
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
┌────────▼────────┐   ┌────────▼────────┐   ┌───────▼─────────┐
│    Worker Node 1 │   │    Worker Node 2 │   │   Worker Node 3  │
│                  │   │                  │   │                  │
│  ┌────────────┐  │   │  ┌────────────┐  │   │ ┌────────────┐  │
│  │  Pod: app  │  │   │  │  Pod: app  │  │   │ │  Pod: db   │  │
│  └────────────┘  │   │  └────────────┘  │   │ └────────────┘  │
│  ┌────────────┐  │   │  ┌────────────┐  │   │                  │
│  │  Pod: api  │  │   │  │  Pod: redis│  │   │                  │
│  └────────────┘  │   │  └────────────┘  │   │                  │
│  ──────────────  │   │  ──────────────  │   │ ──────────────   │
│  kubelet         │   │  kubelet         │   │ kubelet          │
│  kube-proxy      │   │  kube-proxy      │   │ kube-proxy       │
│  container       │   │  container       │   │ container        │
│  runtime         │   │  runtime         │   │ runtime          │
└──────────────────┘   └──────────────────┘   └──────────────────┘
```

**Component Roles:**
| Component | Role |
|-----------|------|
| **kube-apiserver** | Front door to K8s — all API calls go here |
| **etcd** | Cluster state database — stores all K8s objects |
| **kube-scheduler** | Decides which node to place each pod on |
| **kube-controller-manager** | Runs control loops (desired state → actual state) |
| **kubelet** | Node agent — runs pods, reports to API server |
| **kube-proxy** | Network rules — enables pod-to-pod communication |
| **container runtime** | Actually runs containers (containerd, CRI-O) |

---

## 3. Core Concepts

### Pod

The smallest deployable unit — wraps one or more containers:

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  namespace: production
  labels:
    app: myapp
    version: "1.0"
spec:
  containers:
  - name: myapp
    image: myapp:1.0.0
    ports:
    - containerPort: 8080
    env:
    - name: DB_HOST
      value: "postgres-service"
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    resources:
      requests:
        cpu: "100m"       # 0.1 CPU cores
        memory: "128Mi"
      limits:
        cpu: "500m"       # 0.5 CPU cores
        memory: "512Mi"
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'until nc -z postgres-service 5432; do sleep 2; done']
  restartPolicy: Always
```

### Namespace

```bash
kubectl get namespaces
kubectl create namespace production
kubectl get pods -n production
kubectl get pods --all-namespaces    # or -A
kubectl config set-context --current --namespace=production
```

---

## 4. Workloads

### Deployment

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1    # Max pods unavailable during update
      maxSurge: 1          # Max extra pods during update
    # type: Recreate       # Kill all then create (downtime!)
  template:
    metadata:
      labels:
        app: myapp
        version: "2.0"
    spec:
      containers:
      - name: myapp
        image: myapp:2.0.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: myapp
              topologyKey: kubernetes.io/hostname  # Spread across nodes
```

### StatefulSet (for stateful apps — databases)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:16
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:                  # Each pod gets its own PVC
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: gp3
      resources:
        requests:
          storage: 100Gi
```

### DaemonSet (run on every node)

```yaml
# Use for: log collectors, monitoring agents, network plugins
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:latest
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

### HorizontalPodAutoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70   # Scale up if CPU > 70%
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5min before scaling down
```

### Job and CronJob

```yaml
# Job — run to completion
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  backoffLimit: 3    # Retry 3 times on failure
  completions: 1
  template:
    spec:
      containers:
      - name: migrate
        image: myapp:latest
        command: ["python", "manage.py", "migrate"]
      restartPolicy: OnFailure

---
# CronJob — scheduled execution
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-cleanup
spec:
  schedule: "0 2 * * *"        # Daily at 2AM
  concurrencyPolicy: Forbid     # Don't run if previous still running
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup
            image: myapp:latest
            command: ["python", "cleanup.py"]
          restartPolicy: OnFailure
```

---

## 5. Services and Networking

```yaml
# ClusterIP — internal only (default)
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
  - port: 80          # Service port
    targetPort: 8080  # Container port

---
# NodePort — expose on every node's IP:port
apiVersion: v1
kind: Service
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080   # Range: 30000-32767

---
# LoadBalancer — creates cloud load balancer (ELB on AWS)
apiVersion: v1
kind: Service
metadata:
  name: myapp-lb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
  - port: 443
    targetPort: 8080
```

### Ingress

```yaml
# Ingress — HTTP routing rules (requires Ingress Controller like nginx, ALB)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: tls-secret
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /api/v1
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /auth
        pathType: Prefix
        backend:
          service:
            name: auth-service
            port:
              number: 80
```

---

## 6. Storage

```yaml
# PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
  - ReadWriteOnce          # RWO: one node; RWX: many nodes (EFS)
  storageClassName: gp3    # AWS EBS gp3
  resources:
    requests:
      storage: 50Gi

---
# StorageClass (EBS gp3 for EKS)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  fsType: ext4
  encrypted: "true"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer  # Create EBS in same AZ as pod
allowVolumeExpansion: true
```

---

## 7. Configuration Management

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  LOG_LEVEL: "INFO"
  MAX_CONNECTIONS: "100"
  nginx.conf: |
    server {
        listen 80;
        location / {
            proxy_pass http://app:8000;
        }
    }
```

### Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:                        # Kubernetes auto-encodes to base64
  DB_PASSWORD: "supersecret"
  DB_HOST: "postgres.internal"
# data:                            # Or provide base64 manually
#   DB_PASSWORD: c3VwZXJzZWNyZXQ=

# Better: Use External Secrets Operator or Vault
# ExternalSecret from AWS Secrets Manager:
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: SecretStore
  target:
    name: db-secret
  data:
  - secretKey: DB_PASSWORD
    remoteRef:
      key: /prod/myapp/db-password
```

---

## 8. RBAC and Security

```yaml
# ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/myapp-role  # IRSA

---
# Role (namespace-scoped)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods", "pods/logs"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "update", "patch"]

---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-pod-reader
  namespace: production
subjects:
- kind: ServiceAccount
  name: myapp-sa
  namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### PodSecurityContext

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 2000
  containers:
  - name: myapp
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE
    volumeMounts:
    - name: tmp
      mountPath: /tmp                # Writable temp dir
  volumes:
  - name: tmp
    emptyDir: {}
```

---

## 9. Scheduling and Resource Management

```yaml
# Resource Quotas (per namespace)
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"
    services: "20"

---
# LimitRange (defaults per pod)
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
spec:
  limits:
  - default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    max:
      cpu: "2"
      memory: "2Gi"
    type: Container

---
# Taints and Tolerations
# On node: kubectl taint nodes node1 dedicated=gpu:NoSchedule
# On pod:
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"

# NodeSelector
spec:
  nodeSelector:
    disktype: ssd
    topology.kubernetes.io/zone: us-east-1a
```

---

## 10. Helm

Helm is the package manager for Kubernetes:

```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Repository management
helm repo add stable https://charts.helm.sh/stable
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update                              # Update indexes

# Install charts
helm install my-nginx ingress-nginx/ingress-nginx
helm install my-release bitnami/nginx --namespace production
helm install my-app ./my-chart -f values.prod.yaml

# Chart management
helm list                                     # List installed releases
helm list -A                                  # All namespaces
helm status my-release                        # Release status
helm history my-release                       # Release history
helm upgrade my-release bitnami/nginx --set replicaCount=3
helm rollback my-release 1                    # Rollback to revision 1
helm uninstall my-release

# Create custom chart
helm create myapp                             # Scaffold chart structure
# myapp/
# ├── Chart.yaml           # Chart metadata
# ├── values.yaml          # Default values
# ├── charts/              # Dependencies
# └── templates/           # Kubernetes manifests with Go templates
#     ├── deployment.yaml
#     ├── service.yaml
#     ├── ingress.yaml
#     └── _helpers.tpl     # Template helpers

# Template rendering (debug)
helm template myapp ./myapp -f values.prod.yaml
helm lint ./myapp                             # Validate chart

# Package and share
helm package ./myapp                          # Creates myapp-0.1.0.tgz
helm push myapp-0.1.0.tgz oci://registry.example.com/charts
```

---

## 11. kubectl Commands

```bash
# Cluster info
kubectl cluster-info
kubectl get nodes                      # Node status
kubectl get nodes -o wide              # With IP and OS info
kubectl describe node worker-1         # Node details
kubectl top nodes                      # Node resource usage

# Resources
kubectl get pods                       # Pods in current namespace
kubectl get pods -n production         # Specific namespace
kubectl get pods -A                    # All namespaces
kubectl get pods -o wide               # With node and IP
kubectl get pods -l app=myapp          # Label selector
kubectl get pods --watch               # Watch for changes
kubectl get all -n production          # All resource types

# Describe (detailed info)
kubectl describe pod myapp-abc123      # Pod events, conditions
kubectl describe deployment myapp
kubectl describe service myapp-svc

# Logs
kubectl logs pod-name                  # Pod logs
kubectl logs pod-name -c container    # Multi-container pod
kubectl logs -f pod-name              # Follow
kubectl logs --previous pod-name      # Previous (crashed) container
kubectl logs -l app=myapp             # All pods with label
kubectl logs pod-name --since=1h      # Last 1 hour

# Exec
kubectl exec -it pod-name -- bash     # Shell
kubectl exec pod-name -- ls /app      # Run command
kubectl exec -it pod-name -c sidecar -- sh  # Specific container

# Apply and delete
kubectl apply -f manifest.yaml        # Create or update
kubectl apply -f ./k8s/               # Apply directory
kubectl delete -f manifest.yaml       # Delete
kubectl delete pod myapp-abc123       # Delete specific pod
kubectl delete pods -l app=myapp      # Delete by label

# Deployment management
kubectl rollout status deployment/myapp         # Watch rollout
kubectl rollout history deployment/myapp        # History
kubectl rollout undo deployment/myapp           # Rollback
kubectl rollout undo deployment/myapp --to-revision=2
kubectl rollout restart deployment/myapp        # Force restart

# Scaling
kubectl scale deployment myapp --replicas=5
kubectl autoscale deployment myapp --min=2 --max=10 --cpu-percent=70

# Port forwarding (debug)
kubectl port-forward pod/myapp-abc123 8080:8080
kubectl port-forward service/myapp-svc 8080:80

# Copy files
kubectl cp pod-name:/app/logs ./local-logs
kubectl cp ./local-file.txt pod-name:/tmp/

# Config management
kubectl config view                    # Show kubeconfig
kubectl config get-contexts            # List contexts
kubectl config use-context prod-eks    # Switch context
kubectl config set-context --current --namespace=production  # Set default ns

# Resource editing
kubectl edit deployment myapp          # Open in editor
kubectl patch deployment myapp -p '{"spec":{"replicas":3}}'
kubectl set image deployment/myapp myapp=myapp:2.0.0  # Update image

# Debugging
kubectl debug pod-name --image=busybox --target=myapp  # Ephemeral container
kubectl run debug --image=nicolaka/netshoot --rm -it  # Debug networking
```

---

## 12. EKS — Kubernetes on AWS

```bash
# Install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Create cluster
eksctl create cluster \
  --name production \
  --region us-east-1 \
  --nodegroup-name workers \
  --node-type m5.xlarge \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 10 \
  --with-oidc \
  --managed

# Configure kubectl
aws eks update-kubeconfig --region us-east-1 --name production

# Node groups
eksctl create nodegroup \
  --cluster production \
  --name gpu-workers \
  --node-type p3.2xlarge \
  --nodes 2

# IRSA — IAM Roles for Service Accounts (secure AWS API access from pods)
eksctl create iamserviceaccount \
  --name myapp-sa \
  --namespace production \
  --cluster production \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve

# Cluster autoscaler (auto-scale EC2 nodes)
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm install aws-cluster-autoscaler autoscaler/cluster-autoscaler \
  --set autoDiscovery.clusterName=production \
  --set awsRegion=us-east-1

# ALB Ingress Controller
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --set clusterName=production \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

---

## 13. Real-World Deployments

### Complete Application Stack

```yaml
# Full production deployment with all best practices
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: production
  labels:
    app: api
    version: "2.1.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0      # Zero downtime
      maxSurge: 1
  template:
    metadata:
      labels:
        app: api
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
    spec:
      serviceAccountName: api-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      topologySpreadConstraints:        # Spread across AZs
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: api
      containers:
      - name: api
        image: 123456789.dkr.ecr.us-east-1.amazonaws.com/api:2.1.0
        ports:
        - containerPort: 8080
        envFrom:
        - configMapRef:
            name: api-config
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]  # Drain connections before stop
      terminationGracePeriodSeconds: 60
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: api
            topologyKey: kubernetes.io/hostname
```

---

## 14. Common Interview Questions

**Q1: What is the difference between a Pod, ReplicaSet, and Deployment?**
> Pod: smallest unit, runs containers. ReplicaSet: ensures N pod replicas running, handles pod failures. Deployment: manages ReplicaSets, enables rolling updates and rollbacks. In practice, always use Deployment — it creates and manages ReplicaSets automatically.

**Q2: What is the difference between ClusterIP, NodePort, and LoadBalancer services?**
> ClusterIP: internal only, pods communicate via service name (DNS). NodePort: exposes service on every node's IP at static port (30000-32767) — for dev/testing. LoadBalancer: provisions cloud load balancer (ELB on AWS) — for production external access. Use Ingress for HTTP routing with one LB.

**Q3: How does Kubernetes handle pod restarts and self-healing?**
> Kubelet monitors pod health via liveness probes. If liveness fails: container restarted. ReplicaSet controller monitors pod count — if pod dies, creates new one. Node failure: controller creates pods on other nodes. RestartPolicy: Always (default for Deployments), OnFailure (Jobs), Never.

**Q4: What is the difference between liveness and readiness probes?**
> Liveness: Is the container alive? Failure → restart container. Readiness: Is the container ready to receive traffic? Failure → remove from service endpoints (no restart). Use readiness during startup and when temporarily unavailable (cache warming). Use liveness for deadlock detection.

**Q5: How do you perform a zero-downtime deployment in Kubernetes?**
> (1) RollingUpdate strategy with `maxUnavailable: 0`. (2) Proper readiness probes — new pods must pass before old ones are terminated. (3) `preStop` hook with sleep for connection draining. (4) Sufficient `terminationGracePeriodSeconds`. (5) Multiple replicas so traffic shifts gradually.

**Q6: What is a DaemonSet and when do you use it?**
> DaemonSet ensures one pod runs on every node (or selected nodes). Use for: log collectors (Fluentd), monitoring agents (Datadog, Prometheus node-exporter), network plugins (Calico, Weave), storage drivers. Automatically deploys to new nodes when added to cluster.

**Q7: How do you persist data in Kubernetes?**
> PersistentVolumeClaim (PVC) requests storage. StorageClass provisions it dynamically (EBS, EFS). For databases: StatefulSet with volumeClaimTemplates (each pod gets dedicated PVC). For shared storage across pods: ReadWriteMany (EFS/NFS). Volumes outlive pods — data survives pod restarts.

**Q8: What is RBAC in Kubernetes?**
> Role-Based Access Control: defines who can do what. ServiceAccount = identity for pods. Role/ClusterRole = permissions (resources + verbs). RoleBinding/ClusterRoleBinding = assign Role to ServiceAccount/User/Group. Namespace-scoped: Role+RoleBinding. Cluster-wide: ClusterRole+ClusterRoleBinding.

**Q9: What is a ConfigMap vs a Secret?**
> ConfigMap: non-sensitive config data (env vars, config files). Secret: sensitive data (passwords, tokens, keys) — base64 encoded (not encrypted by default!). For real security: enable etcd encryption, use external secrets (Vault, AWS Secrets Manager). Can be consumed as env vars or volume mounts.

**Q10: How does Kubernetes scheduling work?**
> Scheduler filters nodes (can pod fit? taints/tolerations? node affinity?) then scores remaining nodes (balanced resource usage, pod affinity, spread). Assigns pod to highest-scoring node. Influence: nodeSelector (must match), nodeAffinity (preferred or required), pod affinity/anti-affinity, taints/tolerations, resource requests.

---

*Next: [Kubernetes Assignments](02-kubernetes-assignments.md)*
