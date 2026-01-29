# Kubernetes Complete Guide: From Fundamentals to Production

A comprehensive, deep-dive repository covering everything about Kubernetes—from basic concepts to advanced production patterns. This guide combines the 5-week DevOps Unplugged newsletter series with extended topics for professionals building production systems.

## Table of Contents

1. [Week 1: Understanding the Problem](#week-1-understanding-the-problem)
2. [Week 2: Kubernetes Architecture](#week-2-kubernetes-architecture)
3. [Week 3: Core Objects & Manifests](#week-3-core-objects--manifests)
4. [Week 4: Reliability & Self-Healing](#week-4-reliability--self-healing)
5. [Week 5: CI/CD & GitOps](#week-5-cicd--gitops)
6. [Advanced: Networking & Ingress](#advanced-networking--ingress)
7. [Advanced: Storage & StatefulSets](#advanced-storage--statefulsets)
8. [Advanced: Security & RBAC](#advanced-security--rbac)
9. [Advanced: Service Mesh](#advanced-service-mesh)
10. [Production Best Practices](#production-best-practices)
11. [Troubleshooting Guide](#troubleshooting-guide)

---

## Week 1: Understanding the Problem

### Why Kubernetes Exists

Docker Compose works beautifully on a single machine. But production systems don't fit on single machines.

**The Wall You Hit:**
- Server dies → entire application offline
- Traffic spikes 10x → manual scaling required
- New deployment → minutes of downtime
- Multi-machine infrastructure → manual networking, load balancing, failover

**What You Need:**
- Automatic recovery when servers fail
- Automatic scaling with traffic
- Zero-downtime deployments
- Multi-machine orchestration

### Docker Compose vs Kubernetes

| Aspect | Docker Compose | Kubernetes |
|--------|---|---|
| Scope | Single host | Multi-node cluster |
| Scaling | Manual | Automatic |
| Self-healing | No | Yes |
| Rolling updates | Limited | Excellent |
| Multi-cloud | No | Yes |
| Complexity | Simple | Complex |
| Best for | Development, small apps | Production, large scale |

### The Three Core Ideas Behind Kubernetes

**1. Declarative State Over Procedural Commands**
```
Procedural (Docker Compose):
  docker-compose down
  docker-compose up
  
Declarative (Kubernetes):
  "I want 3 copies running. Kubernetes, make it so."
```

**2. System Self-Heals Automatically**
```
Container crashes → Kubernetes restarts it
Server dies → Kubernetes reschedules workload
App is unhealthy → Kubernetes restarts it
```

**3. Scale Should Be Automatic**
```
Define: "Keep CPU at 70% usage"
Kubernetes: Adds/removes pods to maintain that target
```

---

## Week 2: Kubernetes Architecture

### The High-Level Structure

```
Kubernetes Cluster
├── Control Plane (The Brain)
│   ├── API Server (Front door)
│   ├── etcd (Database)
│   ├── Scheduler (Resource allocation)
│   └── Controller Manager (Reconciliation)
├── Worker Node 1 (The Muscle)
│   ├── kubelet (Node agent)
│   ├── Container Runtime
│   └── kube-proxy (Networking)
├── Worker Node 2
│   ├── kubelet
│   ├── Container Runtime
│   └── kube-proxy
└── Worker Node N
    ├── kubelet
    ├── Container Runtime
    └── kube-proxy
```

### Control Plane Components

#### API Server
- **Purpose**: Single entry point to the cluster
- **Function**: Validates requests, stores objects in etcd, processes kubectl commands
- **Key**: Everything goes through here

```
kubectl apply → API Server → validates → stores in etcd
```

#### etcd
- **Purpose**: Distributed database of cluster state
- **Stores**: Every Pod, Deployment, Service, ConfigMap, Node status
- **Criticality**: Lose etcd, lose entire cluster state

```yaml
# Example etcd entry
/kubernetes.io/pods/default/nginx-abc123:
  apiVersion: v1
  kind: Pod
  metadata:
    name: nginx-abc123
  status:
    phase: Running
```

#### Scheduler
- **Purpose**: Decides where Pods should run
- **Process**: 
  1. Watch for unscheduled Pods
  2. Filter nodes (CPU, memory, labels, taints)
  3. Score remaining nodes
  4. Pick winner, assign to Pod

```
Unscheduled Pod → Scheduler → "Run on worker-3" → Pod now has nodeName
```

#### Controller Manager
- **Purpose**: Runs controllers that maintain desired state
- **Key Controllers**:
  - Deployment Controller: Manages replicas
  - ReplicaSet Controller: Maintains pod count
  - Node Controller: Handles node failures
  - Job Controller: Ensures one-time tasks complete

```
Deployment spec: replicas=3
Current state: 2 pods running
Deployment Controller detects mismatch → Creates 1 more pod
```

### Worker Node Components

#### kubelet
- **Purpose**: Node's connection to cluster
- **Function**: Watches API server for assigned Pods, ensures they run
- **Reaction**: Starts containers, monitors health, reports status

```
Scheduler assigns Pod → kubelet sees it → Pulls image → Starts container
```

#### Container Runtime
- **Purpose**: Actually runs containers
- **Options**: containerd, CRI-O, Docker (deprecated)
- **Interface**: Communicates with kubelet via CRI (Container Runtime Interface)

#### kube-proxy
- **Purpose**: Implements Services and network routing
- **How**: Programs iptables/IPVS rules to distribute traffic

```
Service IP:Port → kube-proxy rules → Route to Pod:Port
```

### The Reconciliation Loop: How Everything Works

```
1. Desired State (in etcd):
   Deployment: 3 replicas of app:1.0
   
2. Current State (in etcd):
   2 replicas running, 1 crashed
   
3. Controller observes mismatch:
   "Current != Desired"
   
4. Controller takes action:
   Creates replacement Pod
   
5. Loop repeats every few seconds
   Always working to match desired ↔ current
```

### Complete Flow: From YAML to Running App

```
1. kubectl apply deployment.yaml
   ↓
2. API Server receives, validates, stores in etcd
   ↓
3. Deployment Controller sees new Deployment
   Creates ReplicaSet and 3 Pod objects
   ↓
4. Scheduler sees 3 unscheduled Pods
   Assigns each to a worker node
   ↓
5. kubelet on each node sees assigned Pod
   Pulls image, starts container
   ↓
6. Pod status flows back to etcd
   Controllers monitor for failures
   ↓
7. Service load balances traffic to running Pods
   ↓
8. System continuously works to maintain declared state
```

---

## Week 3: Core Objects & Manifests

### Pods: The Atomic Unit

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    env: production
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
```

**Key Concepts:**
- **Single unit**: Usually one container per Pod
- **Ephemeral**: Pods die, they're replaced
- **Shared network**: Multiple containers in a Pod share localhost
- **Never use standalone**: Always managed by Deployment/StatefulSet

### Deployments: The Real Unit of Management

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: myapp:1.0.0
        ports:
        - containerPort: 3000
        resources:
          requests:
            cpu: "250m"
            memory: "512Mi"
          limits:
            cpu: "500m"
            memory: "1Gi"
```

**Features:**
- **Replication**: Maintains exact number of Pods
- **Self-healing**: Restarts failed Pods
- **Rolling updates**: Zero-downtime deployments
- **Rollback**: One command to go back

**Common Operations:**

```bash
# Create/update
kubectl apply -f deployment.yaml

# View status
kubectl get deployments
kubectl describe deployment web-app

# Scale
kubectl scale deployment/web-app --replicas=5

# Update image
kubectl set image deployment/web-app web=myapp:2.0.0

# Check rollout status
kubectl rollout status deployment/web-app

# Rollback
kubectl rollout undo deployment/web-app
```

### Services: Stable Networking

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000
```

**Service Types:**

```yaml
# ClusterIP: Internal only (default)
type: ClusterIP
# Access: web-service:80 (from within cluster)

# NodePort: External via node IP
type: NodePort
ports:
- port: 80
  targetPort: 3000
  nodePort: 30000  # Access: nodeIP:30000

# LoadBalancer: External via cloud load balancer
type: LoadBalancer
# Access: cloud-lb-ip:80 (from outside)
```

**Endpoints Example:**

```bash
# Service finds all matching Pods
kubectl get service web-service
kubectl get endpoints web-service
# Shows: 10.244.1.5:3000, 10.244.2.7:3000, 10.244.3.2:3000
```

### ConfigMaps: External Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_URL: "postgres://db:5432/app"
  LOG_LEVEL: "info"
  FEATURE_FLAGS: "new-ui=true,analytics=false"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
      - name: app
        image: myapp:1.0
        envFrom:
        - configMapRef:
            name: app-config
```

**Update config without redeploying:**

```bash
kubectl edit configmap app-config
kubectl rollout restart deployment/app
```

### Secrets: Sensitive Data

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  username: dbuser
  password: secure-password
---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:1.0
        env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
```

**Important**: Secrets are base64-encoded, not encrypted. Use sealed-secrets or external secret operators for production.

### Complete Application Stack

```yaml
---
# Namespace for organization
apiVersion: v1
kind: Namespace
metadata:
  name: production
---
# Configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  DATABASE_URL: "postgres://db:5432/prod"
  LOG_LEVEL: "info"
---
# Secrets
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
stringData:
  db_password: "secure-pass"
---
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
      - name: app
        image: ghcr.io/myorg/app:1.2.0
        ports:
        - containerPort: 3000
        envFrom:
        - configMapRef:
            name: app-config
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: db_password
        resources:
          requests:
            cpu: "250m"
            memory: "512Mi"
          limits:
            cpu: "500m"
            memory: "1Gi"
---
# Service
apiVersion: v1
kind: Service
metadata:
  name: app-service
  namespace: production
spec:
  type: LoadBalancer
  selector:
    app: app
  ports:
  - port: 80
    targetPort: 3000
```

---

## Week 4: Reliability & Self-Healing

### Probes: Teaching Kubernetes What "Healthy" Means

Three probe types protect your application:

#### Startup Probe
```yaml
spec:
  containers:
  - name: app
    image: myapp:1.0
    startupProbe:
      httpGet:
        path: /health
        port: 3000
      periodSeconds: 10
      failureThreshold: 18  # 3 minutes max startup
```

**Use**: Apps that take time to boot (migrations, warm-up, JIT)

#### Readiness Probe
```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 3000
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 2
```

**Meaning**: Pod is NOT ready to receive traffic
- Service removes it from load balancer
- Pod is NOT restarted

#### Liveness Probe
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 3000
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3
```

**Meaning**: Pod is stuck, restart it
- kubelet kills and recreates the Pod
- Use cautiously; only for truly broken apps

**Complete Example:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: myapi:1.0.0
        ports:
        - containerPort: 3000
        resources:
          requests:
            cpu: "300m"
            memory: "512Mi"
          limits:
            cpu: "700m"
            memory: "1Gi"
        startupProbe:
          httpGet:
            path: /health
            port: 3000
          periodSeconds: 10
          failureThreshold: 18
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 2
        livenessProbe:
          httpGet:
            path: /healthz
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
```

### Rolling Updates

Control how Pods are replaced during deployment:

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2           # Can go up to 5 pods (3+2)
      maxUnavailable: 1     # At least 2 pods always serving (3-1)
```

**Deployment Flow:**

```
1. kubectl set image deployment/api api=myapi:2.0.0
2. Kubernetes starts new pod with v2.0.0
3. Waits for readiness probe to pass
4. Starts draining old pod (no new traffic)
5. Routes traffic to new pod
6. Removes old pod
7. Repeats until all 3 are v2.0.0
```

**Monitoring:**

```bash
kubectl rollout status deployment/api
kubectl rollout history deployment/api
kubectl rollout undo deployment/api
```

### Horizontal Pod Autoscaler (HPA)

Automatically scale based on metrics:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 65
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

**How It Works:**

```
Every 15 seconds:
  1. Check average CPU/memory across all Pods
  2. Compare to target (65% CPU, 80% memory)
  3. If above: scale up
  4. If below: scale down
  5. Never go below minReplicas (3)
  6. Never go above maxReplicas (20)
```

### Pod Disruption Budgets (PDB)

Protect availability during voluntary disruptions (maintenance, node drains):

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
spec:
  minAvailable: 3
  selector:
    matchLabels:
      app: api
```

**Effect:**

```
Running 6 api pods
Maintenance drains a node
PDB says: "At least 3 must stay up"
Result: Only 3 pods are evicted, not all
```

---

## Week 5: CI/CD & GitOps

### Image Versioning Strategy

**Never use `latest`:**

```bash
# Bad
docker build -t myapp:latest .

# Good: Semantic versioning
docker build -t myapp:1.2.3 .

# Better: Git commit reference too
docker build -t myapp:1.2.3-abc123 .
```

### CI Pipeline (GitHub Actions Example)

```yaml
name: Build & Push
on:
  push:
    tags:
      - "v*"

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Get version
        id: version
        run: echo "tag=${GITHUB_REF#refs/tags/}" >> $GITHUB_OUTPUT
      
      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          push: true
          tags: |
            ghcr.io/myorg/myapp:${{ steps.version.outputs.tag }}
            ghcr.io/myorg/myapp:latest
      
      - name: Update dev manifest
        run: |
          git clone https://x-access-token:${{ secrets.GH_TOKEN }}@github.com/myorg/k8s-manifests
          cd k8s-manifests/dev
          sed -i 's|image:.*|image: ghcr.io/myorg/myapp:${{ steps.version.outputs.tag }}|g' deployment.yaml
          git config user.name "CI"
          git config user.email "ci@example.com"
          git add deployment.yaml
          git commit -m "Promote myapp to ${{ steps.version.outputs.tag }}"
          git push
```

### Promotion Through Environments

```
Repository: k8s-manifests

dev/
├── deployment.yaml (image: myapp:1.2.3)
staging/
├── deployment.yaml (image: myapp:1.2.3)
prod/
└── deployment.yaml (image: myapp:1.2.0)  # Older version
```

**Promotion Process:**

```bash
# After testing in dev passes
# Merge same commit to staging
git cherry-pick <dev-commit>
git push origin staging

# After staging tests pass
# Merge to prod
git cherry-pick <staging-commit>
git push origin prod
```

**Key Principle**: Same binary promoted, not rebuilt.

### GitOps with ArgoCD

Install ArgoCD:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Create Application:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/k8s-manifests
    targetRevision: prod
    path: prod/
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

**How It Works:**

```
1. Git repository changes
   ↓
2. ArgoCD detects change (polling or webhook)
   ↓
3. ArgoCD applies new manifests
   ↓
4. Kubernetes reconciles desired state
   ↓
5. Application updates happen automatically
```

**Advantages:**

- Git is source of truth
- Entire deployment history is audited
- Rollback is one Git revert
- Multiple environments easily managed
- No manual kubectl apply

---

## Advanced: Networking & Ingress

### Services Deep Dive

Services provide stable access to Pods. Understand the difference:

```yaml
# Type 1: ClusterIP (Internal)
kind: Service
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 3000
# Access: http://service-name:80 (from within cluster)

# Type 2: NodePort (Node IP)
kind: Service
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 3000
    nodePort: 30000
# Access: http://node-ip:30000 (from outside)

# Type 3: LoadBalancer (Cloud LB)
kind: Service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 3000
# Access: http://cloud-lb-ip:80 (from internet)
```

### Ingress: Advanced HTTP Routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls-cert
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-v1
            port:
              number: 80
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-v2
            port:
              number: 80
```

**Ingress Handles:**
- HTTPS/TLS termination
- Host-based routing (api.example.com, www.example.com)
- Path-based routing (/api, /admin)
- Load balancing
- Name-based virtual hosting

### Service Mesh (Istio/Linkerd)

For advanced use cases (not recommended for beginners):

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-vs
spec:
  hosts:
  - api
  http:
  - match:
    - uri:
        prefix: "/admin"
    route:
    - destination:
        host: api
        port:
          number: 80
        subset: v2       # Route to v2
      weight: 100
  - route:
    - destination:
        host: api
        port:
          number: 80
        subset: v1       # Route to v1
      weight: 100
```

**Service Mesh Provides:**
- mTLS between services
- Advanced traffic routing (canary, A/B testing)
- Observability (metrics, tracing)
- Circuit breaking and retries
- Better than Service for production microservices

---

## Advanced: Storage & StatefulSets

### Persistent Volumes (PV) & Persistent Volume Claims (PVC)

```yaml
# 1. Administrator creates PersistentVolume
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-database
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: fast
  gcePersistentDisk:
    pdName: gce-disk
    fsType: ext4

# 2. Developer creates PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-database
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast
  resources:
    requests:
      storage: 5Gi
```

### StatefulSets: Stateful Applications

For databases, message queues, anything needing stable identity and storage:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
          name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: standard
      resources:
        requests:
          storage: 20Gi
---
# Headless Service (no load balancing)
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None  # Headless
  selector:
    app: mysql
  ports:
  - port: 3306
    name: mysql
```

**Key Differences from Deployment:**

```
StatefulSet vs Deployment

Pod Identity:
  Deployment: nginx-abc123 (random)
  StatefulSet: mysql-0, mysql-1, mysql-2 (stable)

Storage:
  Deployment: Shared or transient
  StatefulSet: Each pod has own PVC (persistent)

Scaling:
  Deployment: All at once
  StatefulSet: One at a time (ordered)

Access:
  Deployment: service-name:port
  StatefulSet: pod-name.service-name:port
```

---

## Advanced: Security & RBAC

### Role-Based Access Control (RBAC)

```yaml
# 1. Create a Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["pods", "logs"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list"]
---
# 2. Bind Role to User
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-binding
  namespace: production
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: developer
subjects:
- kind: User
  name: "alice@company.com"
  apiGroup: rbac.authorization.k8s.io
```

**Common Roles:**

```yaml
# View-only
- apiGroups: [""]
  resources: ["*"]
  verbs: ["get", "list", "watch"]

# Admin
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]

# Create/update only (no delete)
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["create", "update", "patch"]
```

### Network Policies: Firewall Rules

```yaml
# Default deny all traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# Allow specific traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: mysql
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 3306
```

**Best Practice:**

```
1. Create default deny-all policy
2. Add specific allow policies for needed traffic
3. Test to ensure nothing breaks
4. Monitor for denied connections
```

### Pod Security Policies

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'MustRunAs'
  fsGroup:
    rule: 'MustRunAs'
  readOnlyRootFilesystem: false
```

---

## Production Best Practices

### 1. Resource Requests and Limits

```yaml
resources:
  requests:          # Scheduler uses this for placement
    cpu: "250m"      # 0.25 CPU cores
    memory: "512Mi"
  limits:            # Hard limit; container killed if exceeded
    cpu: "500m"
    memory: "1Gi"
```

**Rule of Thumb:**
- Requests: What your app *usually* uses
- Limits: Absolute max you're willing to give

### 2. Health Checks Configuration

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 3000
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 2
  successThreshold: 1
```

### 3. Update Strategy

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%
    maxUnavailable: 25%
```

### 4. Namespace Organization

```
default/              # Don't use for real apps
kube-system/          # Kubernetes internals
kube-public/          # Public data
monitoring/           # Prometheus, Grafana
logging/              # ELK, Loki
production/           # Production apps
staging/              # Staging apps
development/          # Dev apps
```

### 5. Resource Quotas

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: prod-quota
  namespace: production
spec:
  hard:
    requests.cpu: "100"
    requests.memory: "200Gi"
    limits.cpu: "200"
    limits.memory: "400Gi"
    pods: "500"
    services: "50"
```

### 6. Pod Disruption Budgets for All Critical Services

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: critical-app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      tier: critical
```

### 7. Monitoring and Observability

```yaml
# Prometheus ServiceMonitor
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: app-metrics
spec:
  selector:
    matchLabels:
      app: app
  endpoints:
  - port: metrics
    interval: 30s
```

### 8. Logging Strategy

```yaml
# Sidecar pattern for logging
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:1.0
        volumeMounts:
        - name: log-volume
          mountPath: /var/log
      - name: log-forwarder
        image: filebeat:latest
        volumeMounts:
        - name: log-volume
          mountPath: /var/log
      volumes:
      - name: log-volume
        emptyDir: {}
```

---

## Troubleshooting Guide

### Pod Pending

```bash
kubectl describe pod <name>
# Look for "Unschedulable" events

# Causes:
# 1. Not enough CPU/memory
# 2. Node selectors not matching
# 3. Taints on nodes
# 4. PVC not bound

# Solution:
kubectl describe nodes  # Check available resources
```

### Pod CrashLoopBackOff

```bash
kubectl logs <pod-name>          # Current logs
kubectl logs <pod-name> --previous  # Previous attempt

# Causes:
# 1. App crashing on startup
# 2. Missing config/secrets
# 3. Database unavailable
# 4. Port already in use

# Check readiness probe thresholds
kubectl describe pod <name> | grep -A 5 "readiness"
```

### Pod ImagePullBackOff

```bash
# Check image exists and is accessible
docker pull <image>

# Check secret for private registry
kubectl get secrets
kubectl get secret <name> -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d

# Check image pull policy
kubectl get pod <name> -o yaml | grep imagePullPolicy
```

### Service Not Reachable

```bash
# Check endpoints (backing pods)
kubectl get endpoints <service-name>
# Should show pod IPs, not empty

# If empty:
# 1. Labels don't match
# 2. All pods unhealthy
# 3. Pods in different namespace

# Test connectivity from another pod
kubectl run debug --image=busybox --rm -it -- wget <service>:80
```

### Deployment Stuck

```bash
kubectl describe deployment <name>
# Look for "ProgressDeadlineExceeded"

# Common causes:
# 1. New pods can't become ready
# 2. Insufficient resources
# 3. Image pull issues

# Check replica status
kubectl describe rs <deployment>-<hash>
```

### Node Not Ready

```bash
kubectl describe node <node-name>
# Look for conditions and events

# Check kubelet
ssh <node>
sudo systemctl status kubelet
sudo journalctl -u kubelet -n 100

# Common fixes:
# 1. Restart kubelet
# 2. Check disk space (df -h)
# 3. Check network connectivity
```

### Persistent Volume Not Mounting

```bash
kubectl get pvc
kubectl describe pvc <name>
# Check "Status" and "Events"

# If Pending:
# 1. No PV available
# 2. StorageClass doesn't exist
# 3. Access mode mismatch

kubectl get pv
kubectl describe pv <name>
```

---

## Commands Reference

### Cluster Management

```bash
kubectl cluster-info                    # Cluster info
kubectl get nodes                       # List nodes
kubectl describe node <name>            # Node details
kubectl top nodes                       # Node resource usage
kubectl drain <node>                    # Drain node for maintenance
kubectl cordon <node>                   # Prevent new pods
kubectl uncordon <node>                 # Re-enable pod scheduling
```

### Namespace

```bash
kubectl create namespace <name>
kubectl get namespaces
kubectl delete namespace <name>
kubectl config set-context --current --namespace=<name>  # Set default
```

### Deployment

```bash
kubectl create deployment <name> --image=<image>
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl describe deployment <name>
kubectl scale deployment/<name> --replicas=5
kubectl set image deployment/<name> <container>=<image>
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name>
```

### Pods

```bash
kubectl get pods
kubectl get pods -o wide                # Show node assignment
kubectl get pods --all-namespaces       # All namespaces
kubectl describe pod <name>
kubectl logs <pod>
kubectl logs <pod> --previous           # Previous crashed attempt
kubectl exec -it <pod> -- bash          # Enter container
kubectl port-forward pod/<pod> 3000:3000  # Port forward
kubectl delete pod <name>
```

### Services

```bash
kubectl create service clusterip <name> --tcp=80:8080
kubectl get services
kubectl get endpoints <service>
kubectl describe service <name>
kubectl delete service <name>
```

### ConfigMaps & Secrets

```bash
kubectl create configmap <name> --from-literal=key=value
kubectl create secret generic <name> --from-literal=key=value
kubectl get configmaps
kubectl get secrets
kubectl describe configmap <name>
kubectl edit configmap <name>
kubectl delete configmap <name>
```

### Debugging

```bash
kubectl describe <resource> <name>      # Full details
kubectl logs <pod>                      # Container logs
kubectl logs <pod> -c <container>       # Specific container
kubectl logs <pod> --tail=100           # Last 100 lines
kubectl events <namespace>              # Recent events
kubectl top pods                        # Resource usage
kubectl get events --all-namespaces --sort-by='.lastTimestamp'
```

### Apply & Manage

```bash
kubectl apply -f file.yaml              # Create/update
kubectl apply -f dir/                   # Apply all in directory
kubectl delete -f file.yaml             # Delete
kubectl get -f file.yaml                # Check status
kubectl diff -f file.yaml               # Preview changes
kubectl replace -f file.yaml            # Replace (no merge)
```

---

## Architecture Diagrams

### Complete Kubernetes Flow

```
┌──────────────────────────────────────────────────────────┐
│                    Developer                             │
│         (pushes code to Git)                             │
└──────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────┐
│               GitHub CI Pipeline                         │
│    ├── Build Docker image                               │
│    ├── Push to registry                                 │
│    └── Update k8s manifest in Git                       │
└──────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────┐
│          Git Repository (Source of Truth)                │
│    ├── dev/deployment.yaml (v1.2.3)                     │
│    ├── staging/deployment.yaml (v1.2.3)                │
│    └── prod/deployment.yaml (v1.2.2)                   │
└──────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────┐
│        ArgoCD (watches Git repo)                         │
│    └── Detects changes → applies to cluster            │
└──────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────┐
│          Kubernetes Control Plane                        │
│    ├── API Server (stores in etcd)                      │
│    ├── Scheduler (assigns to nodes)                     │
│    ├── Controllers (reconcile state)                    │
│    └── etcd (source of truth)                           │
└──────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────┐
│           Kubernetes Worker Nodes                        │
│    ├── kubelet (executes Pods)                          │
│    ├── Container Runtime (runs containers)             │
│    ├── kube-proxy (routes traffic)                      │
│    └── Running Pods (the actual application)            │
└──────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────┐
│               Service & Ingress                          │
│    ├── Internal: Service ClusterIP                      │
│    └── External: LoadBalancer or Ingress               │
└──────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────┐
│            Users/External Traffic                        │
└──────────────────────────────────────────────────────────┘
```

---

## Interview Preparation

### Common Questions

**Q: How does Kubernetes handle failures?**
A: Through controllers and the reconciliation loop. Controllers continuously compare desired state (in etcd) to actual state, taking corrective action. If a pod crashes, the deployment controller recreates it. If a node dies, the scheduler reschedules pods to healthy nodes.

**Q: What's the difference between a Pod and a Deployment?**
A: A Pod is the smallest deployable unit (usually one container). A Deployment manages multiple Pods, ensuring a desired number stay running, handling updates, and managing ReplicaSets. You never use standalone Pods in production.

**Q: Explain rolling updates.**
A: Gradually replace old pods with new ones. `maxSurge` controls how many extra pods can exist (for speed), `maxUnavailable` ensures minimum availability. Kubernetes waits for readiness probes before marking pods as ready. If new pods fail, the rollout pauses, protecting the old version.

**Q: What are probes and why do you need them?**
A: Kubernetes has no built-in app knowledge. Probes teach it:
- Startup: "Is this app done booting?"
- Readiness: "Can it handle traffic?"
- Liveness: "Is it stuck beyond recovery?"

**Q: How does a Service work?**
A: Services select pods by label and provide a stable IP. kube-proxy programs iptables rules to load-balance traffic to selected pods. When pods die/restart, the Service endpoint automatically updates.

**Q: What's etcd and why is it critical?**
A: Distributed database storing entire cluster state (all objects). Lose etcd = lose cluster configuration. It's why production clusters run etcd with multiple replicas for high availability.

**Q: Explain the scheduler's job.**
A: Watches for unscheduled Pods. Filters nodes (CPU, memory, labels, taints), scores remaining nodes, picks best match, assigns pod to node by setting `.spec.nodeName`.

---

## Kubernetes Certification Paths

### CKA (Certified Kubernetes Administrator)
- Focus: Cluster setup, management, troubleshooting
- Topics: Workloads, scheduling, networking, storage, security
- Format: Hands-on exam (2 hours, 67% pass)

### CKAD (Certified Kubernetes Application Developer)
- Focus: Building and deploying applications
- Topics: Pods, deployments, services, config, observability
- Format: Hands-on exam (2 hours, 66% pass)

### CKS (Certified Kubernetes Security Specialist)
- Focus: Securing Kubernetes
- Topics: RBAC, network policies, secrets, PSP, admission control

---

## Next Steps After This Guide

1. **Hands-on Practice**
   - Deploy an application locally with Minikube
   - Experiment with different probe configurations
   - Practice rolling updates and rollbacks
   - Set up GitOps with ArgoCD

2. **Monitoring & Observability**
   - Deploy Prometheus for metrics
   - Set up Grafana for dashboards
   - Learn distributed tracing (Jaeger)
   - Implement log aggregation (ELK, Loki)

3. **Advanced Topics**
   - Service mesh (Istio/Linkerd)
   - Custom resource definitions (CRDs)
   - Operators pattern
   - Multi-cluster deployments

4. **Certifications**
   - CKAD (easiest starting point)
   - CKA (comprehensive)
   - CKS (security focus)

---

## Resources

- **Official**: kubernetes.io docs
- **Practice**: Play with Kubernetes (katakoda.com/kubernetes)
- **Courses**: Linux Academy, Pluralsight
- **Community**: Kubernetes Slack, local meetups
- **Books**: "Kubernetes in Action" by Marko Lukša

---
