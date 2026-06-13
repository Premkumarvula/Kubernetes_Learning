# Kubernetes_Learning


# ☸️ Kubernetes From Scratch — Mapped to AWS ECS

> **For Prem** — You already know AWS ECS (Task Definitions, Services, Fargate, ALB, Service Discovery). This course teaches Kubernetes from the ground up, constantly mapping every concept back to what you already know in ECS. Think of ECS as AWS's opinionated orchestrator and K8s as the open-source, cloud-agnostic orchestrator that does the same job — but with more knobs to turn.

---

## 🧭 Quick Mental Model: Kubernetes ↔ ECS Mapping

| Kubernetes Concept | AWS ECS Equivalent | What It Means |
|---|---|---|
| Pod | ECS Task (single container) | Smallest deployable unit |
| Deployment | ECS Service (with desired count) | Manages replica count & rolling updates |
| ReplicaSet | ECS Service desired count | Ensures N replicas are running |
| Service (ClusterIP) | ECS Service Connect / CloudMap | Internal DNS & load balancing |
| Service (LoadBalancer) | ECS Service + ALB/NLB | External traffic to pods |
| Service (NodePort) | ECS on EC2 with host port | Expose on worker node port |
| Ingress | ALB + Listener Rules | HTTP-level routing (path/host based) |
| ConfigMap | Environment variables in task def | Non-sensitive configuration |
| Secret | AWS Secrets Manager / SSM | Sensitive configuration |
| Namespace | ECS Cluster (logical grouping) | Resource isolation & quota |
| Node | EC2 Instance (in ECS cluster) | Worker machine running pods |
| Cluster | ECS Cluster | Full K8s environment |
| kubelet | ECS Agent | Agent on each node managing pods |
| kubectl | AWS CLI / ECS Console | CLI to manage the cluster |
| PersistentVolume (PV) | EBS Volume | Cluster-level storage |
| PersistentVolumeClaim (PVC) | Volume mount in task def | App requesting storage |
| StorageClass | EBS volume type (gp2, io1) | Dynamic storage provisioning |
| Liveness Probe | Health check in task def | Is the app alive? Restart if not |
| Readiness Probe | Target Group health check | Is the app ready to serve traffic? |
| Startup Probe | — (ECS doesn't have this) | Has the app finished starting? |
| Resource Requests | Task CPU/Memory reservation | Guaranteed minimum resources |
| Resource Limits | Task CPU/Memory hard limit | Maximum resources allowed |
| Horizontal Pod Autoscaler | ECS Service Auto Scaling | Auto-scale based on metrics |
| Pod Disruption Budget | — (ECS doesn't have this) | Voluntary disruption protection |
| DaemonSet | ECS Service on every EC2 instance | Run a pod on every node |
| StatefulSet | ECS with EFS + ordered deploy | Stateful apps with stable identity |
| Job | ECS RunTask (one-off) | Run to completion |
| CronJob | EventBridge → ECS RunTask | Scheduled runs |
| Label | ECS task tags | Key-value pairs for filtering |
| Selector | ECS service tag filters | Query resources by labels |
| Taint / Tolerations | Placement constraints | Node-level scheduling rules |
| Node Affinity | Task placement constraints | Pod → Node scheduling |
| NetworkPolicy | Security Group | Pod-level network firewall |
| RBAC | IAM policies for ECS | Who can do what |
| Helm | CloudFormation / CDK | Package manager for K8s apps |
| kustomize | — (no direct equivalent) | Template-free YAML customization |
| Rollout | ECS rolling deployment | Zero-downtime deployment strategy |
| Rollback | ECS CodeDeploy rollback | Revert to previous version |
| Init Container | Container dependencies in task def | Containers that run before main |
| Sidecar Container | Multiple containers in task def | Helper container alongside main |
| kube-proxy | ENI + Security Groups | Network routing on each node |
| etcd | — (AWS manages this) | Cluster state database |

---

## 📦 Module 1: What Is Kubernetes & Why It Exists

### 1.1 The Problem Kubernetes Solves

```
Without an orchestrator:
┌──────────────────────────────────────────────┐
│  "My app crashed at 3 AM"                      │
│  "We need 10 more instances for Black Friday"  │
│  "Container A can't reach Container B"          │
│  "We need zero-downtime deployments"            │
│  "Which node has capacity for this container?"  │
│                                                 │
│  You'd have to do ALL of this manually!        │
└──────────────────────────────────────────────┘

With an orchestrator (ECS or K8s):
┌──────────────────────────────────────────────┐
│  ✅ Auto-restarts crashed containers           │
│  ✅ Auto-scales based on demand                │
│  ✅ Built-in service discovery & networking     │
│  ✅ Rolling updates & rollbacks                │
│  ✅ Smart scheduling across nodes              │
└──────────────────────────────────────────────┘
```

### 1.2 ECS vs Kubernetes — Same Job, Different Approach

```
ECS:  "AWS manages the orchestrator, you just define tasks"
K8s:  "YOU manage the orchestrator, but you get full control"

ECS = iPhone (simple, managed, opinionated)
K8s  = Android (flexible, powerful, more complexity)

Both run containers. Both schedule workloads. Both auto-heal.
The difference is WHO manages the control plane.
```

### 1.3 ECS Connection 🧠

> **You already understand container orchestration!** ECS Service maintaining desired count = K8s Deployment. ECS task placement = K8s scheduling. ALB routing to ECS tasks = K8s Service + Ingress. The concepts are the same — K8s just gives you more granular control and is cloud-agnostic.

---

## 📦 Module 2: Kubernetes Architecture

### 2.1 The Big Picture

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                         │
│                                                               │
│  ┌─────────────────────────────────────────────────────┐     │
│  │              Control Plane (Master)                   │     │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐  │     │
│  │  │ API      │ │ Scheduler│ │ Controller│ │  etcd   │  │     │
│  │  │ Server   │ │          │ │  Manager  │ │         │  │     │
│  │  └──────────┘ └──────────┘ └──────────┘ └────────┘  │     │
│  └─────────────────────────────────────────────────────┘     │
│                           │                                   │
│            ┌──────────────┼──────────────┐                    │
│            ▼              ▼              ▼                    │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐         │
│  │  Worker Node  │ │  Worker Node  │ │  Worker Node  │         │
│  │  ┌──────────┐ │ │  ┌──────────┐ │ │  ┌──────────┐ │         │
│  │  │ kubelet  │ │ │  │ kubelet  │ │ │  │ kubelet  │ │         │
│  │  ├──────────┤ │ │  ├──────────┤ │ │  ├──────────┤ │         │
│  │  │kube-proxy│ │ │  │kube-proxy│ │ │  │kube-proxy│ │         │
│  │  ├──────────┤ │ │  ├──────────┤ │ │  ├──────────┤ │         │
│  │  │ Pod  Pod │ │ │  │ Pod  Pod │ │ │  │ Pod  Pod │ │         │
│  │  └──────────┘ │ │  └──────────┘ │ │  └──────────┘ │         │
│  └──────────────┘ └──────────────┘ └──────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Control Plane Components

| Component | What It Does | ECS Equivalent |
|---|---|---|
| **API Server** | The front door — all kubectl commands hit this | ECS Control Plane (AWS managed) |
| **etcd** | Stores ALL cluster state (like a database) | AWS manages this for you in ECS |
| **Scheduler** | Decides which node runs which pod | ECS Task Placement Engine |
| **Controller Manager** | Runs control loops (deployment, replica, etc.) | ECS Service Controller |
| **Cloud Controller** | Integrates with AWS (ELB, EBS, etc.) | AWS's ECS integration |

### 2.3 Worker Node Components

| Component | What It Does | ECS Equivalent |
|---|---|---|
| **kubelet** | Agent on each node — starts/stops pods | ECS Agent on EC2 |
| **kube-proxy** | Network routing on each node | ENI + Security Groups |
| **Container Runtime** | Runs containers (containerd, Docker) | Docker/containerd on ECS |

### 2.4 ECS Connection 🧠

> **In ECS, AWS runs the control plane for you — you never see it.** In K8s, the control plane is visible and YOU manage it (or EKS manages it for you). The kubelet = ECS Agent. The scheduler = ECS task placement. etcd = the state store AWS manages behind the scenes. When you use EKS, AWS manages the control plane just like ECS — you get the best of both worlds.

---

## 📦 Module 3: Pods — The Smallest Unit

### 3.1 What Is a Pod?

A Pod is the **smallest deployable unit** in Kubernetes. It's a group of one or more containers that:
- Share the same network namespace (same IP, same ports)
- Share the same storage volumes
- Are always scheduled on the same node
- Are always started and stopped together

```
┌─────────────────────────────────────┐
│                Pod                    │
│  IP: 10.0.1.5 (shared)              │
│                                       │
│  ┌──────────────┐  ┌──────────────┐  │
│  │  Main App     │  │  Sidecar     │  │
│  │  :3000        │  │  :8080       │  │
│  │               │  │  (log agent) │  │
│  └──────────────┘  └──────────────┘  │
│                                       │
│  ┌──────────────────────────────────┐ │
│  │  Shared Volume                   │ │
│  └──────────────────────────────────┘ │
└─────────────────────────────────────┘
```

### 3.2 Pod YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  labels:
    app: my-app
    env: production
spec:
  containers:
  - name: api
    image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-api:1.0
    ports:
    - containerPort: 3000
    env:
    - name: NODE_ENV
      value: "production"
    - name: DB_HOST
      value: "my-rds.cluster-xxxxx.us-east-1.rds.amazonaws.com"
    resources:
      requests:
        memory: "256Mi"
        cpu: "250m"
      limits:
        memory: "512Mi"
        cpu: "500m"
    livenessProbe:
      httpGet:
        path: /health
        port: 3000
      initialDelaySeconds: 15
      periodSeconds: 30
    readinessProbe:
      httpGet:
        path: /ready
        port: 3000
      initialDelaySeconds: 5
      periodSeconds: 10
  restartPolicy: Always
```

### 3.3 Pod with Sidecar (Multi-Container)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app-with-sidecar
spec:
  containers:
  - name: api                    # Main container
    image: my-api:1.0
    ports:
    - containerPort: 3000
    volumeMounts:
    - name: logs
      mountPath: /app/logs

  - name: log-agent              # Sidecar container
    image: fluent/fluentd:latest
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
    # Both containers share the "logs" volume!
    # api writes → log-agent reads & ships

  volumes:
  - name: logs
    emptyDir: {}
```

### 3.4 Pod with Init Container

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app-with-init
spec:
  initContainers:
  - name: wait-for-db            # Runs FIRST, must complete before main
    image: busybox
    command: ['sh', '-c', 'until nc -z db-host 5432; do echo waiting; sleep 2; done']

  containers:
  - name: api
    image: my-api:1.0
    ports:
    - containerPort: 3000
```

### 3.5 Pod Lifecycle

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Pending  │───▶│ Running  │───▶│ Succeeded│    │ Failed   │
│          │    │          │    │(exit 0)  │    │(exit ≠0) │
└──────────┘    └────┬─────┘    └──────────┘    └──────────┘
                     │
              CrashLoopBackOff
              (keeps crashing & restarting)
```

### 3.6 ECS Connection 🧠

> **A Pod ≈ an ECS Task.** In ECS, a task can have multiple containers (main + sidecar) — that's exactly what a Pod is. The key difference: ECS tasks don't share a network namespace by default (each container gets its own), but in a Pod, all containers share the same IP and port space. Init containers in K8s = `dependsOn` in ECS task definition. The `restartPolicy` on a Pod = ECS service maintaining desired count.

---

## 📦 Module 4: Deployments — Managing Pod Lifecycle

### 4.1 What Is a Deployment?

A Deployment manages a **ReplicaSet** which manages **Pods**. It gives you:
- Desired state (I want 3 replicas)
- Rolling updates (zero-downtime deploys)
- Rollback capability (undo a bad deploy)
- Self-healing (restart crashed pods)

```
Deployment
    │
    └──▶ ReplicaSet (manages replica count)
              │
              └──▶ Pod Pod Pod (actual running instances)
```

### 4.2 Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-api
  labels:
    app: my-api
spec:
  replicas: 3                      # Like ECS desired count
  selector:                        # Which pods belong to this deployment
    matchLabels:
      app: my-api
  strategy:                        # Deployment strategy
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1                  # Can create 1 extra pod during update
      maxUnavailable: 0            # Never go below 3 pods
  template:                        # Pod template (like task definition)
    metadata:
      labels:
        app: my-api
    spec:
      containers:
      - name: api
        image: my-api:1.0
        ports:
        - containerPort: 3000
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        env:
        - name: NODE_ENV
          value: "production"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 15
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 10
```

### 4.3 Deployment Strategies

```yaml
# Strategy 1: Rolling Update (default, zero-downtime)
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1          # Create 1 new pod before killing old
    maxUnavailable: 0     # Never drop below desired count

# Strategy 2: Recreate (kill all, then start new — HAS DOWNTIME)
strategy:
  type: Recreate
  # All old pods killed before new ones start
  # Use only for apps that can't run two versions simultaneously
```

### 4.4 Rolling Update Visualization

```
Before update:  [v1] [v1] [v1]

Rolling update (maxSurge=1, maxUnavailable=0):
Step 1:  [v1] [v1] [v1] [v2]     ← New v2 pod created
Step 2:  [v1] [v1] [v2] [v2]     ← Another v2, one v1 terminating
Step 3:  [v1] [v2] [v2] [v2]     ← Continue...
Step 4:  [v2] [v2] [v2]          ← All v2, one extra removed

Result: Zero downtime! Traffic gradually shifts from v1 → v2
```

### 4.5 Deployment Commands

```bash
# Create deployment
kubectl apply -f deployment.yaml

# Check deployment status
kubectl get deployments
kubectl describe deployment my-api

# Check rollout status
kubectl rollout status deployment/my-api

# Check rollout history
kubectl rollout history deployment/my-api

# Rollback to previous version
kubectl rollout undo deployment/my-api

# Rollback to specific revision
kubectl rollout undo deployment/my-api --to-revision=2

# Scale deployment
kubectl scale deployment my-api --replicas=5

# Update image (triggers rolling update)
kubectl set image deployment/my-api api=my-api:2.0

# Edit deployment live
kubectl edit deployment my-api

# Delete deployment
kubectl delete deployment my-api
```

### 4.6 ECS Connection 🧠

> **Deployment = ECS Service.** The `replicas` field = ECS Service `desiredCount`. Rolling update = ECS rolling deployment. `maxSurge` = ECS `maximumPercent`. `maxUnavailable` = ECS `minimumHealthyPercent`. Rollback = ECS CodeDeploy rollback. The key difference: K8s gives you more control over the rollout strategy, while ECS keeps it simpler.

---

## 📦 Module 5: Services — Networking & Discovery

### 5.1 Why Services?

Pods are **ephemeral** — they get created, destroyed, and get new IPs. A Service provides a **stable IP and DNS name** that routes to healthy pods.

```
Without Service:
Client → Pod IP (10.0.1.5)  ← Pod dies, new IP = 10.0.1.9  ← BROKEN!

With Service:
Client → Service (stable IP + DNS: my-api.default.svc.cluster.local)
              │
              ├──▶ Pod 10.0.1.5 (healthy)
              ├──▶ Pod 10.0.1.7 (healthy)
              └──✕  Pod 10.0.1.9 (unhealthy — skipped)
```

### 5.2 Service Types

```yaml
# ─── ClusterIP (default) — Internal only ───
# Like ECS Service Connect / CloudMap
apiVersion: v1
kind: Service
metadata:
  name: my-api-internal
spec:
  type: ClusterIP              # Only reachable within cluster
  selector:
    app: my-api
  ports:
  - port: 80                   # Service port
    targetPort: 3000            # Container port
```

```yaml
# ─── NodePort — Expose on worker node port ───
# Like ECS on EC2 with host port mapping
apiVersion: v1
kind: Service
metadata:
  name: my-api-nodeport
spec:
  type: NodePort
  selector:
    app: my-api
  ports:
  - port: 80
    targetPort: 3000
    nodePort: 30080            # Accessible on <NodeIP>:30080
```

```yaml
# ─── LoadBalancer — External load balancer ───
# Like ECS Service + ALB/NLB
apiVersion: v1
kind: Service
metadata:
  name: my-api-lb
spec:
  type: LoadBalancer
  selector:
    app: my-api
  ports:
  - port: 80
    targetPort: 3000
# Creates a Cloud Load Balancer (ELB on AWS)
```

### 5.3 Service Discovery

```
Within the cluster, services resolve by DNS:

Full DNS:    my-api.default.svc.cluster.local
Short DNS:   my-api.default
Shorter:     my-api          (same namespace)

Other services can call:
  curl http://my-api:80      ← That's it!

This is EXACTLY like ECS Service Connect / CloudMap!
```

### 5.4 Headless Service (for StatefulSets)

```yaml
# Headless — no ClusterIP, returns pod IPs directly
# Used for StatefulSets where each pod needs a stable identity
apiVersion: v1
kind: Service
metadata:
  name: my-db-headless
spec:
  clusterIP: None              # ← Makes it headless
  selector:
    app: my-db
  ports:
  - port: 5432
    targetPort: 5432
# DNS: my-db-0.my-db-headless, my-db-1.my-db-headless, etc.
```

### 5.5 ECS Connection 🧠

> **K8s Service = ECS Service + ALB/CloudMap.** ClusterIP = ECS Service Connect (internal). LoadBalancer = ECS Service with ALB. NodePort = ECS host port mapping. DNS-based service discovery = CloudMap. The concept is identical — stable endpoint that load-balances to dynamic pod/task IPs.

---

## 📦 Module 6: Ingress — HTTP-Level Routing

### 6.1 What Is Ingress?

An Ingress is an **HTTP-level router** — it routes traffic based on host and path. It's the K8s equivalent of an **ALB with listener rules**.

```
Internet
    │
    ▼
┌──────────────────────────────────────────┐
│           Ingress Controller (ALB)        │
│                                           │
│  api.example.com/*    ──▶ api Service     │
│  app.example.com/*    ──▶ web Service     │
│  example.com/api/*    ──▶ api Service     │
│  example.com/web/*    ──▶ web Service     │
└──────────────────────────────────────────┘
```

### 6.2 Ingress YAML

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-api
            port:
              number: 80

  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-web
            port:
              number: 80

  - host: example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: my-api
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-web
            port:
              number: 80
```

### 6.3 Ingress vs Service LoadBalancer

```
Service LoadBalancer:
  One LB per service → 5 services = 5 LBs = expensive!

Ingress:
  One LB for ALL services → 5 services = 1 LB = cost effective!
  + Path-based routing
  + Host-based routing
  + TLS termination
```

### 6.4 ECS Connection 🧠

> **Ingress = ALB with Listener Rules.** In ECS, you configure ALB listener rules to route `/api/*` to one target group and `/web/*` to another. In K8s, you write an Ingress resource that does the same thing. On AWS, the Ingress Controller (like AWS Load Balancer Controller) actually creates an ALB for you!

---

## 📦 Module 7: ConfigMaps & Secrets

### 7.1 ConfigMap — Non-Sensitive Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-app-config
data:
  NODE_ENV: "production"
  DB_HOST: "my-rds.cluster-xxxxx.us-east-1.rds.amazonaws.com"
  DB_PORT: "5432"
  LOG_LEVEL: "info"
  # Or mount as a file
  app.properties: |
    server.port=3000
    cache.ttl=300
    feature.flag=true
```

### 7.2 Using ConfigMap in a Pod

```yaml
# As environment variables
spec:
  containers:
  - name: api
    envFrom:
    - configMapRef:
        name: my-app-config        # All keys become env vars

# Or specific keys only
spec:
  containers:
  - name: api
    env:
    - name: NODE_ENV
      valueFrom:
        configMapKeyRef:
          name: my-app-config
          key: NODE_ENV

# Or mount as files
spec:
  containers:
  - name: api
    volumeMounts:
    - name: config
      mountPath: /app/config
  volumes:
    - name: config
      configMap:
        name: my-app-config
```

### 7.3 Secret — Sensitive Configuration

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-app-secrets
type: Opaque
data:
  DB_PASSWORD: c3VwZXJzZWNyZXQ=    # base64 encoded!
  API_KEY: YXBpLWtleS0xMjM0NQ==   # base64 encoded!
```

```bash
# Create secrets imperatively
kubectl create secret generic my-app-secrets \
  --from-literal=DB_PASSWORD=supersecret \
  --from-literal=API_KEY=api-key-12345

# Or from a file
kubectl create secret generic db-creds \
  --from-file=credentials=./db-creds.json
```

### 7.4 Using Secrets in a Pod

```yaml
spec:
  containers:
  - name: api
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: my-app-secrets
          key: DB_PASSWORD
```

### 7.5 ECS Connection 🧠

> **ConfigMap = Environment variables in ECS task definition.** Secret = AWS Secrets Manager / SSM Parameter Store (via `secrets` in task definition). The key difference: K8s Secrets are just base64-encoded (NOT encrypted by default) — you need to enable encryption at rest. AWS Secrets Manager is more secure out of the box. In production K8s, use external secret managers (like AWS Secrets Manager via External Secrets Operator).

---

## 📦 Module 8: Namespaces — Multi-Tenancy

### 8.1 What Are Namespaces?

Namespaces are **logical clusters within a cluster** — like having multiple virtual clusters.

```
┌──────────────────────────────────────────┐
│            Kubernetes Cluster              │
│                                            │
│  ┌──────────────┐  ┌──────────────┐       │
│  │  default      │  │  production  │       │
│  │  ┌──┐ ┌──┐   │  │  ┌──┐ ┌──┐  │       │
│  │  │P1│ │P2│   │  │  │P1│ │P2│  │       │
│  │  └──┘ └──┘   │  │  └──┘ └──┘  │       │
│  └──────────────┘  └──────────────┘       │
│                                            │
│  ┌──────────────┐  ┌──────────────┐       │
│  │  staging      │  │  monitoring │       │
│  │  ┌──┐        │  │  ┌──┐ ┌──┐  │       │
│  │  │P1│        │  │  │P1│ │P2│  │       │
│  │  └──┘        │  │  └──┘ └──┘  │       │
│  └──────────────┘  └──────────────┘       │
└──────────────────────────────────────────┘
```

### 8.2 Working with Namespaces

```bash
# List namespaces
kubectl get namespaces

# Create namespace
kubectl create namespace production

# Or via YAML
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
```

```bash
# Deploy to a namespace
kubectl apply -f deployment.yaml -n production

# Set default namespace
kubectl config set-context --current --namespace=production

# Resource quotas per namespace
```

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
    services: "10"
```

### 8.3 ECS Connection 🧠

> **Namespace ≈ ECS Cluster (for logical grouping).** In ECS, you might have separate clusters for dev/staging/prod. In K8s, you can use namespaces within one cluster. Namespaces also support ResourceQuotas (like how you'd limit resources per ECS cluster) and RBAC (like IAM policies per cluster).

---

## 📦 Module 9: Volumes & Persistent Storage

### 9.1 The Storage Problem (Again!)

Same as Docker — pods are ephemeral. When a pod dies, its data dies. K8s solves this with **PersistentVolumes**.

### 9.2 Storage Hierarchy

```
PersistentVolume (PV)          — Cluster-level storage resource (like an EBS volume)
PersistentVolumeClaim (PVC)    — App's request for storage (like mounting EBS in task def)
StorageClass                   — Dynamic provisioning (like EBS volume type)

Flow:
1. App creates PVC: "I need 10Gi of storage"
2. StorageClass provisions a PV automatically (creates an EBS volume on AWS)
3. App mounts the PVC in its pod spec
```

### 9.3 PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-app-data
spec:
  accessModes:
  - ReadWriteOnce              # RWO — one node can mount read-write
  resources:
    requests:
      storage: 10Gi
  storageClassName: gp3        # AWS EBS gp3 volume type
```

### 9.4 Using PVC in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-db
spec:
  containers:
  - name: postgres
    image: postgres:15
    volumeMounts:
    - name: data
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-app-data    # References the PVC above
```

### 9.5 StorageClass (AWS EBS Example)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com     # AWS EBS CSI Driver
parameters:
  type: gp3
  iopsPerGB: "50"
  throughput: "250"
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

### 9.6 Access Modes

| Mode | Abbreviation | Description | AWS Equivalent |
|---|---|---|---|
| ReadWriteOnce | RWO | One node read-write | EBS (default) |
| ReadOnlyMany | ROX | Many nodes read-only | EBS snapshot shared |
| ReadWriteMany | RWX | Many nodes read-write | EFS |

### 9.7 ECS Connection 🧠

> **PV = EBS Volume. PVC = Volume mount in ECS task definition. StorageClass = EBS volume type (gp2, gp3, io1).** The big difference: K8s separates the storage admin (who creates PVs/StorageClasses) from the app developer (who creates PVCs). In ECS, you just specify the volume in the task definition. Also, for RWX (ReadWriteMany), K8s uses EFS — same as ECS.

---

## 📦 Module 10: Health Checks (Probes)

### 10.1 Three Types of Probes

```
┌──────────────────────────────────────────────────────────┐
│                    K8s Probes                              │
├──────────────┬───────────────────────────────────────────┤
│ Liveness     │ "Is the app ALIVE?"                        │
│ Probe        │ If fails → KILL & RESTART the pod          │
│              │ Like: healthCheck in ECS task definition   │
├──────────────┼───────────────────────────────────────────┤
│ Readiness    │ "Is the app READY to serve traffic?"        │
│ Probe        │ If fails → REMOVE from Service (no traffic)│
│              │ Like: ALB target group health check        │
├──────────────┼───────────────────────────────────────────┤
│ Startup      │ "Has the app FINISHED starting?"            │
│ Probe        │ If fails → KILL & RESTART the pod          │
│              │ ECS doesn't have this — it's K8s-only!     │
└──────────────┴───────────────────────────────────────────┘
```

### 10.2 Probe Configuration

```yaml
spec:
  containers:
  - name: api
    image: my-api:1.0

    # Liveness: "Kill and restart if unhealthy"
    livenessProbe:
      httpGet:
        path: /health
        port: 3000
      initialDelaySeconds: 15    # Wait 15s before first check
      periodSeconds: 30          # Check every 30s
      timeoutSeconds: 3          # Timeout after 3s
      failureThreshold: 3       # Fail 3 times before restarting
      successThreshold: 1        # 1 success to mark healthy

    # Readiness: "Stop sending traffic if not ready"
    readinessProbe:
      httpGet:
        path: /ready
        port: 3000
      initialDelaySeconds: 5
      periodSeconds: 10
      failureThreshold: 3

    # Startup: "Wait for slow-starting apps"
    startupProbe:
      httpGet:
        path: /health
        port: 3000
      initialDelaySeconds: 0
      periodSeconds: 5           # Check every 5s
      failureThreshold: 30       # Allow up to 150s (5*30) to start
    # While startup probe is running, liveness & readiness are disabled!
```

### 10.3 Probe Types

```yaml
# HTTP GET (most common)
httpGet:
  path: /health
  port: 3000

# TCP Socket
tcpSocket:
  port: 5432

# Command (exec inside container)
exec:
  command:
  - cat
  - /tmp/healthy

# gRPC (K8s 1.24+)
grpc:
  port: 3000
```

### 10.4 ECS Connection 🧠

> **Liveness Probe = ECS task definition healthCheck** (container-level, restart if unhealthy). **Readiness Probe = ALB target group health check** (remove from load balancer if not ready). **Startup Probe = ECS doesn't have this** — it's a K8s feature for slow-starting apps. In ECS, you'd use `startPeriod` in the health check, which is similar but less flexible.

---

## 📦 Module 11: Resource Management

### 11.1 Requests vs Limits

```
┌──────────────────────────────────────────────────────┐
│                Resource Management                      │
├──────────────┬───────────────────────────────────────┤
│ Requests      │ GUARANTEED minimum. Scheduler uses    │
│               │ this to place pods on nodes.           │
│               │ Like: ECS task CPU/memory reservation   │
├──────────────┼───────────────────────────────────────┤
│ Limits        │ MAXIMUM allowed. Pod is killed/throttled│
│               │ if it exceeds this.                    │
│               │ Like: ECS task CPU/memory hard limit   │
└──────────────┴───────────────────────────────────────┘
```

### 11.2 Resource YAML

```yaml
spec:
  containers:
  - name: api
    resources:
      requests:                    # Guaranteed minimum
        memory: "256Mi"           # 256 MiB guaranteed
        cpu: "250m"               # 0.25 CPU guaranteed (250 millicores)
      limits:                      # Maximum allowed
        memory: "512Mi"           # OOM killed if exceeds
        cpu: "500m"               # CPU throttled if exceeds
```

### 11.3 CPU & Memory Units

```
CPU:
  1 CPU = 1000m (millicores)
  500m = 0.5 CPU
  250m = 0.25 CPU
  1 full core = "1" or "1000m"

Memory:
  Mi = Mebibytes (2^20 bytes) — USE THIS
  Gi = Gibibytes (2^30 bytes) — USE THIS
  M  = Megabytes (10^6) — less common
  G  = Gigabytes (10^9) — less common
```

### 11.4 QoS Classes

```
┌──────────────────────────────────────────────────────────┐
│ If requests == limits → Guaranteed QoS (highest priority) │
│ If requests < limits  → Burstable QoS (medium priority)   │
│ If no requests/limits → BestEffort QoS (lowest priority)  │
│                                                            │
│ When a node runs out of memory:                            │
│   BestEffort pods killed FIRST                             │
│   Then Burstable pods (by usage)                           │
│   Guaranteed pods killed LAST                               │
└──────────────────────────────────────────────────────────┘
```

### 11.5 LimitRange (Default Resources per Namespace)

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
  - default:              # Default limits if not specified
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:        # Default requests if not specified
      cpu: "100m"
      memory: "128Mi"
    type: Container
```

### 11.6 ECS Connection 🧠

> **Requests = ECS task CPU/memory reservation. Limits = ECS task CPU/memory hard limit.** In ECS, you set one value per task (e.g., 512 CPU units, 1024 MiB memory). In K8s, you can set both requests AND limits independently — this is more flexible. Requests affect scheduling; limits affect enforcement. ECS essentially uses requests == limits (no overcommit).

---

## 📦 Module 12: Labels, Selectors & Annotations

### 12.1 Labels

```yaml
metadata:
  labels:
    app: my-api              # What app is this?
    env: production           # What environment?
    tier: backend             # What tier?
    version: v2.0            # What version?
    team: platform            # Who owns it?
```

### 12.2 Selectors

```yaml
# Equality-based selector
selector:
  matchLabels:
    app: my-api
    env: production

# Set-based selector
selector:
  matchExpressions:
  - key: env
    operator: In
    values: ["production", "staging"]
  - key: tier
    operator: NotIn
    values: ["frontend"]
```

### 12.3 Annotations

```yaml
metadata:
  annotations:
    # Non-identifying metadata — used by tools & automation
    kubernetes.io/ingress.class: alb
    prometheus.io/scrape: "true"
    prometheus.io/port: "3000"
    description: "Main API service for the platform"
    contact: "platform-team@company.com"
```

### 12.4 ECS Connection 🧠

> **Labels = ECS task tags.** Selectors = ECS service tag filters. Annotations = ECS task metadata that tools use. Same concept, different syntax. Labels are for identifying/selecting; annotations are for storing tool-specific metadata.

---

## 📦 Module 13: ReplicaSets, DaemonSets, StatefulSets

### 13.1 ReplicaSet (Usually managed by Deployment)

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-api-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-api
  template:
    # ... pod template
```

> You rarely create ReplicaSets directly — Deployments manage them for you.

### 13.2 DaemonSet — Run on EVERY Node

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-agent
spec:
  selector:
    matchLabels:
      app: log-agent
  template:
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd:latest
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

**Use cases:** Log agents, monitoring agents, node exporters, CNI plugins

### 13.3 StatefulSet — Stateful Apps with Stable Identity

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless     # Required! Headless service
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    spec:
      containers:
      - name: postgres
        image: postgres:15
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:              # Each pod gets its OWN PVC!
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 50Gi
```

**Key differences from Deployment:**
- Pods get stable names: `postgres-0`, `postgres-1`, `postgres-2`
- Pods get stable DNS: `postgres-0.postgres-headless`, `postgres-1.postgres-headless`
- Each pod gets its own PersistentVolume
- Ordered start/stop: 0→1→2 (start), 2→1→0 (stop)
- Pod always gets the same PVC back after reschedule

### 13.4 ECS Connection 🧠

> **ReplicaSet = ECS Service desired count (you don't manage this directly).** DaemonSet = running a task on every EC2 instance in your ECS cluster (you'd use ECS capacity providers or custom scripts). StatefulSet = ECS with EFS + ordered deployment — but K8s gives you stable pod identity and per-pod volumes, which ECS doesn't natively support. For databases, use RDS — don't run StatefulSets unless you need to.

---

## 📦 Module 14: Jobs & CronJobs

### 14.1 Job — Run to Completion

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  completions: 1              # Run 1 time successfully
  parallelism: 1              # Run 1 pod at a time
  backoffLimit: 3             # Retry up to 3 times on failure
  template:
    spec:
      containers:
      - name: migrate
        image: my-api:1.0
        command: ["npm", "run", "migrate"]
      restartPolicy: Never
```

### 14.2 CronJob — Scheduled Runs

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-cleanup
spec:
  schedule: "0 2 * * *"      # Every day at 2 AM (cron format)
  concurrencyPolicy: Forbid   # Don't run overlapping jobs
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup
            image: my-api:1.0
            command: ["npm", "run", "cleanup"]
          restartPolicy: Never
```

### 14.3 ECS Connection 🧠

> **Job = ECS RunTask (one-off task).** CronJob = EventBridge rule → ECS RunTask. In ECS, you'd set up an EventBridge scheduled rule that triggers an ECS task. In K8s, CronJob is built-in. Same concept, different mechanism.

---

## 📦 Module 15: Horizontal Pod Autoscaler (HPA)

### 15.1 What Is HPA?

HPA automatically scales the number of pod replicas based on metrics (CPU, memory, custom metrics).

```
┌──────────────────────────────────────────────┐
│              HPA in Action                     │
│                                                │
│  CPU at 80% → Scale UP (add more pods)        │
│  CPU at 20% → Scale DOWN (remove pods)        │
│  Min: 2 pods, Max: 10 pods                    │
│                                                │
│  [Pod] [Pod] → [Pod] [Pod] [Pod] → [Pod] [Pod]│
│   2 replicas     3 replicas       2 replicas   │
└──────────────────────────────────────────────┘
```

### 15.2 HPA YAML

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70     # Scale when CPU > 70%
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80     # Scale when memory > 80%
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100                 # Can double pods in one step
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scaling down
      policies:
      - type: Percent
        value: 10                  # Remove 10% of pods at a time
        periodSeconds: 60
```

### 15.3 ECS Connection 🧠

> **HPA = ECS Service Auto Scaling.** In ECS, you create a scaling policy with target tracking (e.g., scale when CPU > 70%). HPA does the same thing. The key difference: HPA scales pods within a deployment; ECS auto scaling scales tasks within a service. Both need a metrics source (CloudWatch for ECS, Metrics Server for K8s).

---

## 📦 Module 16: Networking Deep Dive

### 16.1 K8s Networking Model

```
3 Fundamental Rules of K8s Networking:
────────────────────────────────────────
1. Every Pod gets its own IP address
2. Every Pod can reach every other Pod (no NAT)
3. Nodes can reach every Pod (no NAT)

This is EXACTLY like ECS awsvpc mode:
  Each task gets its own ENI + IP
  All tasks can reach each other
  No NAT needed within the VPC
```

### 16.2 CNI Plugins (Container Network Interface)

```
┌──────────────────────────────────────────────────────┐
│                  CNI Plugins on AWS                    │
├──────────────┬───────────────────────────────────────┤
│ AWS VPC CNI  │ Default on EKS. Each pod gets an ENI. │
│              │ Like ECS awsvpc mode. Native AWS net.   │
├──────────────┼───────────────────────────────────────┤
│ Calico       │ Popular CNI + NetworkPolicy enforcement│
├──────────────┼───────────────────────────────────────┤
│ Cilium       │ eBPF-based. Fast. GKE default.        │
├──────────────┼───────────────────────────────────────┤
│ Flannel      │ Simple overlay. Good for learning.     │
└──────────────┴───────────────────────────────────────┘
```

### 16.3 NetworkPolicy — Pod-Level Firewall

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: my-api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: my-web        # Only web pods can talk to api
    ports:
    - port: 3000
      protocol: TCP
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres      # api can only reach postgres
    ports:
    - port: 5432
      protocol: TCP
  - to:                      # Allow DNS resolution
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - port: 53
      protocol: UDP
```

### 16.4 ECS Connection 🧠

> **NetworkPolicy = Security Group.** In ECS, you use Security Groups to control which tasks can talk to each other. In K8s, NetworkPolicy does the same thing at the pod level. The key difference: NetworkPolicy is namespace-aware and label-based (more flexible), while Security Groups are IP/VPC-based. On EKS with AWS VPC CNI, Security Groups can also be attached to pods directly.

---

## 📦 Module 17: Scheduling — Taints, Tolerations & Affinity

### 17.1 Taints & Tolerations

```
Taint:    "This node repels pods that don't tolerate the taint"
Toleration: "This pod can tolerate that taint"

Like: ECS placement constraints (memberOf, distinctInstance)
```

```bash
# Taint a node (e.g., dedicate a node for GPU workloads)
kubectl taint nodes node-1 gpu=true:NoSchedule
# NoSchedule = don't schedule pods that don't tolerate this taint

# Taint effects:
# NoSchedule      — Don't schedule new pods
# PreferNoSchedule — Try to avoid, but not guaranteed
# NoExecute       — Don't schedule AND evict existing pods
```

```yaml
# Pod that tolerates the GPU taint
spec:
  tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  # This pod CAN be scheduled on node-1
```

### 17.2 Node Affinity

```yaml
# Prefer to schedule on specific nodes
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-type
            operator: In
            values: ["high-memory"]
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 50
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values: ["us-east-1a"]
```

### 17.3 Pod Anti-Affinity (Spread Pods Across Nodes)

```yaml
# Ensure pods are spread across different nodes
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: my-api
        topologyKey: kubernetes.io/hostname
        # Don't put two my-api pods on the same node!
```

### 17.4 ECS Connection 🧠

> **Taints/Tolerations = ECS placement constraints.** Node affinity = ECS task placement constraints (memberOf). Pod anti-affinity = ECS spread strategy across AZs. In ECS, you use `placementConstraints` and `placementStrategies`. In K8s, you use taints, tolerations, and affinity rules. Same goal — control where pods/tasks run.

---

## 📦 Module 18: kubectl — The K8s CLI

### 18.1 Essential kubectl Commands

```bash
# ─── Cluster Info ────────────────────────────
kubectl cluster-info
kubectl get nodes
kubectl get nodes -o wide

# ─── Get Resources ──────────────────────────
kubectl get pods                          # List pods
kubectl get pods -A                       # All namespaces
kubectl get pods -o wide                  # With node info
kubectl get pods -w                       # Watch for changes
kubectl get deployments
kubectl get services
kubectl get ingress
kubectl get configmaps
kubectl get secrets
kubectl get namespaces
kubectl get all                          # All resources in namespace

# ─── Describe Resources ─────────────────────
kubectl describe pod my-app-xxxxx
kubectl describe deployment my-api
kubectl describe service my-api
kubectl describe ingress my-app-ingress

# ─── Create & Apply ─────────────────────────
kubectl apply -f deployment.yaml          # Create/update from file
kubectl apply -f ./manifests/            # Apply all YAMLs in directory
kubectl apply -k ./kustomization/        # Apply with kustomize

# ─── Delete ─────────────────────────────────
kubectl delete -f deployment.yaml
kubectl delete pod my-app-xxxxx
kubectl delete deployment my-api

# ─── Exec & Logs ────────────────────────────
kubectl exec -it my-app-xxxxx -- /bin/sh  # Shell into pod
kubectl logs my-app-xxxxx                 # View logs
kubectl logs my-app-xxxxx -f             # Follow logs
kubectl logs my-app-xxxxx -c sidecar     # Specific container logs

# ─── Port Forward (local debugging) ────────
kubectl port-forward pod/my-app-xxxxx 8080:3000
# Now access http://localhost:8080

# ─── Scale ──────────────────────────────────
kubectl scale deployment my-api --replicas=5

# ─── Labels ─────────────────────────────────
kubectl get pods --show-labels
kubectl get pods -l app=my-api
kubectl label pod my-app-xxxxx env=staging

# ─── Debugging ──────────────────────────────
kubectl get events --sort-by='.lastTimestamp'
kubectl top nodes                        # Node resource usage
kubectl top pods                         # Pod resource usage
kubectl api-resources                    # List all resource types
```

### 18.2 Imperative vs Declarative

```bash
# Imperative (quick, not versioned)
kubectl run nginx --image=nginx
kubectl expose pod nginx --port=80 --type=LoadBalancer
kubectl scale deployment nginx --replicas=3

# Declarative (GitOps-friendly, versioned) — PREFER THIS
kubectl apply -f deployment.yaml
# Store YAML in Git → apply from Git → reproducible, auditable
```

### 18.3 ECS Connection 🧠

> **kubectl = AWS CLI for ECS.** `kubectl get pods` = `aws ecs list-tasks` + `describe-tasks`. `kubectl logs` = CloudWatch Logs. `kubectl exec` = ECS Exec. `kubectl apply` = `aws ecs register-task-definition` + `update-service`. The key difference: kubectl is declarative (describe desired state), AWS CLI is imperative (tell what to do).

---

## 📦 Module 19: Helm — Package Manager for K8s

### 19.1 What Is Helm?

Helm is the **package manager for Kubernetes** — like `npm` for Node.js, `pip` for Python, or **CloudFormation/CDK for AWS**.

```
Without Helm:
  10 YAML files for one app (deployment, service, ingress, configmap, etc.)
  Copy-paste for each environment (dev, staging, prod)
  Hard to version and share

With Helm:
  One chart = one packaged app
  Values files for each environment
  Easy to install, upgrade, rollback
  Share via Helm repositories
```

### 19.2 Helm Chart Structure

```
my-app-chart/
├── Chart.yaml              # Chart metadata (name, version)
├── values.yaml             # Default values
├── values-prod.yaml        # Production overrides
├── values-staging.yaml     # Staging overrides
├── templates/
│   ├── deployment.yaml     # Deployment template
│   ├── service.yaml        # Service template
│   ├── ingress.yaml        # Ingress template
│   ├── configmap.yaml      # ConfigMap template
│   ├── secrets.yaml        # Secrets template
│   ├── hpa.yaml            # HPA template
│   ├── _helpers.tpl        # Template helpers
│   └── NOTES.txt           # Post-install instructions
└── .helmignore
```

### 19.3 values.yaml

```yaml
# Default values — can be overridden per environment
replicaCount: 2

image:
  repository: 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-api
  tag: "1.0"
  pullPolicy: IfNotPresent

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

service:
  type: ClusterIP
  port: 80
  targetPort: 3000

ingress:
  enabled: true
  className: alb
  host: api.example.com
  path: /

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

### 19.4 values-prod.yaml (Override)

```yaml
replicaCount: 5

resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: "1"
    memory: 1Gi

autoscaling:
  minReplicas: 5
  maxReplicas: 20
```

### 19.5 Helm Commands

```bash
# Install a chart
helm install my-api ./my-app-chart
helm install my-api ./my-app-chart -f values-prod.yaml

# Upgrade (with zero downtime)
helm upgrade my-api ./my-app-chart -f values-prod.yaml

# Rollback
helm rollback my-api 1

# List releases
helm list
helm list -A                    # All namespaces

# Uninstall
helm uninstall my-api

# Dry run (see what gets created without applying)
helm install my-api ./my-app-chart --dry-run --debug

# Template (render YAML without applying)
helm template my-api ./my-app-chart > rendered.yaml

# Add a repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm search repo nginx
helm install my-nginx bitnami/nginx
```

### 19.6 ECS Connection 🧠

> **Helm = CloudFormation / CDK for K8s.** In ECS, you use CloudFormation/CDK/Terraform to define your infrastructure as code. In K8s, Helm does the same — it packages all your YAML manifests into a versioned, parameterized chart. Values files = CloudFormation parameters. `helm upgrade` = CloudFormation stack update. `helm rollback` = CloudFormation stack rollback.

---

## 📦 Module 20: ECS → EKS Migration Playbook

### 20.1 ECS Task Definition → K8s Deployment

This is the **most practical module** — translating what you know to K8s.

```
ECS Task Definition          →    K8s Deployment
─────────────────────────        ─────────────────────────
family: my-api-task              metadata.name: my-api
networkMode: awsvpc             (handled by CNI)
cpu: 512                        resources.requests.cpu: 250m
memory: 1024                    resources.requests.memory: 256Mi
containerDefinitions:           spec.containers:
  image: ...                      image: ...
  portMappings:                    ports:
  environment:                     env: (or ConfigMap)
  secrets:                         env: (from Secret)
  healthCheck:                     livenessProbe:
  logConfiguration:                (handled by EKS add-ons)
executionRoleArn:               serviceAccountName:
taskRoleArn:                    serviceAccountName:
```

### 20.2 ECS Service → K8s Deployment + Service

```
ECS Service                  →    K8s Resources
─────────────────────────        ─────────────────────────
serviceName: my-api             Deployment: my-api
desiredCount: 3                 replicas: 3
launchType: FARGATE             (EKS Fargate profile)
loadBalancers:                  Service (LoadBalancer) + Ingress
  targetGroupArn                service + ingress
  containerPort                 targetPort
deploymentConfiguration:        strategy:
  maximumPercent: 200             maxSurge: 1
  minimumHealthyPercent: 50       maxUnavailable: 0
healthCheckGracePeriod:         readinessProbe:
serviceConnectConfiguration:    Service (ClusterIP)
```

### 20.3 Step-by-Step Migration

```bash
# Step 1: Create EKS Cluster
eksctl create cluster \
  --name my-cluster \
  --region us-east-1 \
  --nodes 3 \
  --node-type m5.large

# Step 2: Create namespace
kubectl create namespace production

# Step 3: Create ConfigMap (from ECS env vars)
kubectl apply -f configmap.yaml -n production

# Step 4: Create Secrets (from ECS Secrets Manager refs)
kubectl apply -f secrets.yaml -n production

# Step 5: Create ServiceAccount (from ECS task role)
kubectl apply -f serviceaccount.yaml -n production

# Step 6: Create Deployment (from ECS task definition)
kubectl apply -f deployment.yaml -n production

# Step 7: Create Service (from ECS service + ALB)
kubectl apply -f service.yaml -n production

# Step 8: Create Ingress (from ALB listener rules)
kubectl apply -f ingress.yaml -n production

# Step 9: Verify
kubectl get all -n production
kubectl port-forward deployment/my-api 8080:3000 -n production
curl http://localhost:8080/health
```

### 20.4 ECS vs EKS Decision Guide

```
When to use ECS:
├── You're all-in on AWS
├── You want simplicity & managed experience
├── You don't need K8s-specific tools
├── Your team knows AWS, not K8s
└── You want less operational overhead

When to use EKS (K8s on AWS):
├── You need cloud portability
├── You want the K8s ecosystem (Helm, ArgoCD, etc.)
├── You have K8s expertise on the team
├── You need advanced scheduling (affinity, taints)
├── You need StatefulSets, DaemonSets
└── You want multi-cloud or hybrid deployments

When to use ECS Fargate:
├── You don't want to manage nodes AT ALL
├── Simple workloads (web APIs, workers)
└── Cost is predictable per-task

When to use EKS Fargate:
├── You want K8s without node management
├── You need K8s features + serverless
└── Slightly more expensive but zero node ops
```

### 20.5 ECS Connection 🧠

> **This is the bridge between your ECS world and K8s.** Every ECS concept has a K8s equivalent. You're not learning from scratch — you're learning new syntax for concepts you already understand. The mental model is: Task Definition → Deployment YAML, ECS Service → K8s Service + Deployment, ALB → Ingress, CloudMap → K8s DNS, Security Groups → NetworkPolicy.

---

## 🏆 kubectl Cheat Sheet (Quick Reference)

### Cluster & Node
```bash
kubectl cluster-info                        # Cluster info
kubectl get nodes                           # List nodes
kubectl describe node <node>                # Node details
kubectl top nodes                           # Node resource usage
kubectl cordon <node>                       # Mark node unschedulable
kubectl drain <node>                        # Evict all pods from node
```

### Pods
```bash
kubectl get pods                            # List pods
kubectl get pods -A                         # All namespaces
kubectl get pods -o wide                    # With node/IP info
kubectl describe pod <pod>                  # Pod details & events
kubectl logs <pod> -f                       # Follow logs
kubectl exec -it <pod> -- sh               # Shell into pod
kubectl port-forward <pod> 8080:3000       # Local port forward
kubectl delete pod <pod>                    # Delete pod
```

### Deployments
```bash
kubectl get deployments                     # List deployments
kubectl apply -f deployment.yaml            # Create/update
kubectl scale deployment <name> --replicas=5  # Scale
kubectl rollout status deployment/<name>    # Rollout status
kubectl rollout history deployment/<name>   # Rollout history
kubectl rollout undo deployment/<name>      # Rollback
kubectl set image deployment/<name> <c>=<img>  # Update image
```

### Services & Ingress
```bash
kubectl get services                        # List services
kubectl get ingress                         # List ingress
kubectl describe service <name>            # Service details
kubectl describe ingress <name>            # Ingress details
```

### ConfigMaps & Secrets
```bash
kubectl get configmaps                      # List configmaps
kubectl get secrets                         # List secrets
kubectl create configmap <name> --from-literal=key=val
kubectl create secret generic <name> --from-literal=key=val
```

### Namespaces
```bash
kubectl get namespaces                      # List namespaces
kubectl apply -f file.yaml -n <ns>         # Apply to namespace
kubectl config set-context --current --namespace=<ns>
```

### Debugging
```bash
kubectl get events --sort-by='.lastTimestamp'  # Recent events
kubectl top pods                                # Pod resource usage
kubectl api-resources                           # All resource types
kubectl explain pod.spec.containers             # API documentation
```

---

## 🎯 Interview Questions — Kubernetes + ECS

### Beginner
1. What is a Pod and how does it relate to a container?
2. What is the difference between a Deployment and a ReplicaSet?
3. What is a Kubernetes Service and why do we need it?
4. How does K8s service discovery work?
5. What is the difference between a ConfigMap and a Secret?

### Intermediate
6. Explain the difference between liveness, readiness, and startup probes.
7. What is the difference between resource requests and resource limits?
8. How does a rolling update work in Kubernetes?
9. What is a Namespace and when would you use one?
10. How does HPA decide when to scale up or down?

### Advanced
11. How does Kubernetes networking compare to ECS networking? (Explain CNI, awsvpc, etc.)
12. What is a StatefulSet and when would you use it instead of a Deployment?
13. Explain taints, tolerations, and node affinity — how do they affect scheduling?
14. How would you migrate an ECS workload to Kubernetes?
15. What is a NetworkPolicy and how does it compare to AWS Security Groups?

### Scenario-Based
16. Your pod keeps going into CrashLoopBackOff. How do you debug it?
17. You need zero-downtime deployments. How do you configure the Deployment?
18. Your K8s cluster is running out of resources. How do you investigate and fix?
19. You need to run a database in K8s. What resources would you use and why?
20. Compare ECS Fargate vs EKS Fargate for a new microservice — which do you pick and why?

---

## 📚 Recommended Learning Path

| Week | Focus | What to Do |
|------|-------|-----------|
| **1** | K8s Basics | Architecture, Pods, kubectl (Modules 1-3, 18) |
| **2** | Core Resources | Deployments, Services, Ingress (Modules 4-6) |
| **3** | Config & Storage | ConfigMaps, Secrets, Volumes, Probes (Modules 7-10) |
| **4** | Advanced | Resources, Scheduling, HPA, Networking (Modules 11-17) |
| **5** | Production | Helm, ECS→EKS migration, Labs (Modules 19-20) |

---

## 🔑 The One-Liner That Ties It All Together

> **ECS is Kubernetes with AWS managing the control plane and simplifying the API.** Every ECS concept has a K8s equivalent. You already know orchestration — now you're just learning the K8s dialect.

---

*Built with ❤️ by Cline SR for Prem — Kubernetes from scratch, mapped to what you already know*
