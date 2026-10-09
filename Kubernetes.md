# Kubernetes on Your Local Machine — Complete Notes

**Local Setup · Docker Desktop · MERN Stack · nginx Ingress**
*Sheryians Coding School*

A complete guide to understanding and running Kubernetes locally using Docker Desktop. Covers every core component with examples from a real MERN stack project, plus config, probes, storage, debugging, autoscaling, and tooling.

---

## Table of Contents

**Overview**
- [00. What is Kubernetes?](#00-what-is-kubernetes)
- [01. Architecture](#01-architecture)

**Core Components**
- [02. Pod](#02-pod)
- [03. Deployment](#03-deployment)
- [04. ReplicaSet](#04-replicaset)
- [05. Service](#05-service)
- [06. Ingress](#06-ingress)
- [07. Ingress Controller](#07-ingress-controller)

**Setup**
- [08. Enable Kubernetes on Docker Desktop](#08-enable-kubernetes-on-docker-desktop)
- [09. Writing YAML Files](#09-writing-yaml-files)
- [10. Labels & Selectors](#10-labels--selectors)

**Configuration & Reliability**
- [11. Namespaces](#11-namespaces)
- [12. ConfigMap](#12-configmap)
- [13. Secrets (Local)](#13-secrets-local)
- [14. Health Probes](#14-health-probes)
- [15. Rolling Updates & Rollbacks](#15-rolling-updates--rollbacks)
- [16. Volumes & Persistent Storage](#16-volumes--persistent-storage)

**Run & Debug**
- [17. Deploy Your App](#17-deploy-your-app)
- [18. Full MERN Stack on Local Kubernetes](#18-full-mern-stack-on-local-kubernetes)
- [19. kubectl Essentials & Port-Forward](#19-kubectl-essentials--port-forward)
- [20. Debug Commands & Troubleshooting](#20-debug-commands--troubleshooting)
- [21. Autoscaling (HPA)](#21-autoscaling-hpa)

**Beyond the Basics**
- [22. Other Workload Types](#22-other-workload-types)
- [23. Helm & Kustomize](#23-helm--kustomize)
- [24. Handy Tools](#24-handy-tools)
- [25. Cleanup & Reset](#25-cleanup--reset)
- [26. From Local to EKS](#26-from-local-to-eks)
- [27. Best Practices Checklist](#27-best-practices-checklist)

---

## 00. What is Kubernetes?

> **Core Concept** — **Kubernetes (K8s)** is a container orchestration system. It manages how your Docker containers run — making sure the right number are running, restarting them when they crash, routing traffic between them, and scaling them up when load increases. You describe what you want in YAML files and Kubernetes makes it happen.

### Without vs With Kubernetes

| Situation | Without Kubernetes | With Kubernetes |
|---|---|---|
| Container crashes | App goes down, manual restart needed | Kubernetes restarts it automatically |
| Traffic spike | App slows down or crashes | HPA adds more containers automatically |
| Multiple services | Manual ports, complex networking | Internal DNS — services talk by name |
| Deploy new version | Downtime during update | Rolling update — zero downtime |
| Multiple replicas | Manage each container manually | Set `replicas: 3` and Kubernetes handles the rest |

### Declarative model (the big idea)

You never say *"start a container"*. You say *"I want 3 copies of this app running"* (desired state). Kubernetes constantly compares **desired state** with **actual state** and fixes any difference. This loop is called **reconciliation**.

```
You write YAML (desired state)  →  API Server stores it in etcd
                                         ↓
Controllers watch: actual state ≠ desired state?
                                         ↓
Fix it: create / delete / restart pods
```

### Key Terms at a glance

| Term | Simple meaning |
|---|---|
| **Pod** | Smallest unit. One or more containers running together. |
| **Deployment** | Manages pods — keeps them running, handles updates and rollbacks. |
| **ReplicaSet** | Keeps N identical pods alive. Created by a Deployment. |
| **Service** | Stable network endpoint that finds pods using labels and routes traffic to them. |
| **Ingress** | HTTP routing rules — which domain/path goes to which service. |
| **Ingress Controller** | The actual pod (nginx) that reads Ingress rules and routes real traffic. |
| **Namespace** | Virtual cluster inside a cluster. Used to separate environments. |
| **ConfigMap** | Store non-secret config (env variables) separately from your image. |
| **Secret** | Store sensitive config (passwords, tokens) encoded in the cluster. |
| **Volume / PVC** | Storage that can outlive a pod (needed for databases). |
| **HPA** | Automatically scales pod count based on CPU/memory. |
| **Node** | A machine (VM or physical) that runs pods. On Docker Desktop: one node. |
| **Cluster** | Control plane + all nodes together. |

---

## 01. Architecture

> **Key Concept** — A Kubernetes cluster has two parts: the **Control Plane** (the brain — decides what runs where) and **Worker Nodes** (the muscle — where your pods actually run). On Docker Desktop both live on your local machine, inside a single node called `docker-desktop`.

### Full traffic flow

```
Browser / Client
   │  HTTP request to express.local or localhost
   ▼
Ingress Controller (nginx pod)
   │  Receives traffic. Reads Ingress rules to decide where to send it.
   ▼
Ingress Resource (your ingress.yaml rules)
   │  Defines which host/path maps to which Service
   ▼
Service (ClusterIP)
   │  Finds pods using label selector. Load balances across matching pods.
   ▼
Pod (your Express container)
      Handles the request and sends back a response
```

### Cluster architecture

```
┌─────────────────── KUBERNETES CLUSTER (Docker Desktop) ───────────────────┐
│                                                                            │
│  ┌─── MASTER NODE · Control Plane ───┐   ┌─── WORKER NODE · Pods run ────┐ │
│  │                                   │   │                               │ │
│  │  API Server   Scheduler           │   │  kubelet   kube-proxy         │ │
│  │  Controller Manager   etcd        │   │  Container Runtime            │ │
│  │                                   │   │                               │ │
│  │  System pods:                     │   │  Deployment                   │ │
│  │   coredns, kube-proxy,            │   │   └─ ReplicaSet               │ │
│  │   ingress-nginx, metrics-server   │   │       ├─ Pod 1 (app=express)  │ │
│  │                                   │   │       ├─ Pod 2 (app=express)  │ │
│  └───────────────────────────────────┘   │       └─ Pod 3 (HPA scaled)   │ │
│                                          └───────────────────────────────┘ │
│  Service (ClusterIP) · Ingress · HPA · Secrets / ConfigMap                 │
└────────────────────────────────────────────────────────────────────────────┘

Browser/curl → localhost → Ingress Controller → Service → Pod
kubectl → API Server → Scheduler/Controller → kubelet on Worker → Pod created
```

### Control plane components

| Component | Role |
|---|---|
| **API Server** | Entry point for every `kubectl` command and every internal component. |
| **Scheduler** | Decides which node a new pod should run on (based on CPU/memory availability). |
| **Controller Manager** | Runs the control loops (ReplicaSet, Deployment, Node controllers…) that keep actual state = desired state. |
| **etcd** | Key-value database that stores the whole cluster state. |

### Worker node components

| Component | Role |
|---|---|
| **kubelet** | Agent on each node. Talks to the API server and makes sure assigned pods are running. |
| **kube-proxy** | Maintains networking rules so Services can reach pods. |
| **Container runtime** | Actually runs containers (containerd / Docker). |

### What happens when you run `kubectl apply -f deployment.yaml`

1. `kubectl` sends the YAML to the **API Server**.
2. API Server validates it and stores it in **etcd**.
3. **Deployment controller** sees a new Deployment → creates a **ReplicaSet**.
4. **ReplicaSet controller** sees "need 2 pods, have 0" → creates **Pod objects**.
5. **Scheduler** assigns each pod to a node.
6. **kubelet** on that node pulls the image and starts the container.
7. Pod becomes `Running` → `Ready` → Service starts sending traffic to it.

### Project file structure

```
your-project/
├── Backend/
│   ├── server.js
│   ├── package.json
│   └── dockerfile
└── k8s/                  # all Kubernetes YAML files here
    ├── deployment.yaml   # how to run the app
    ├── service.yaml      # how to reach the app
    └── ingress.yaml      # how external traffic enters
```

---

## 02. Pod

> **Key Concept** — A **Pod** is a wrapper around one or more containers. All containers in a pod share the same network (same IP address) and storage. In practice, most pods contain a single container — your app. You rarely create pods directly — a Deployment creates and manages them for you.

### What a pod contains

| What | Description |
|---|---|
| **Container(s)** | One or more Docker containers. They share the same `localhost` inside the pod. |
| **IP Address** | Each pod gets its own IP inside the cluster. This IP changes every time a pod restarts — which is why you use a Service instead of the pod IP directly. |
| **Labels** | Key-value tags like `app: express`. Services and Deployments use these to find the pod. |
| **Resources** | CPU and memory limits and requests defined in the pod spec. |

### Pod lifecycle

```
Pending            → pod is scheduled, image is being pulled
   ↓
Running            → container started successfully
   ↓
Ready              → readiness probe passed, pod receives traffic ✅
   ↓
CrashLoopBackOff   → container keeps crashing, Kubernetes retries
   ↓
ImagePullBackOff   → cannot pull the Docker image
```

Other final states: `Succeeded` (job finished), `Failed`, `Terminating` (being deleted), `Evicted` (node ran out of resources), `OOMKilled` (container exceeded memory limit).

> 💡 You **never create pods directly** in production. Always use a Deployment. If a pod is created directly and crashes, nothing restarts it. A Deployment automatically replaces crashed pods.

### Minimal standalone pod (for quick experiments only)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  labels:
    app: test
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
```

### Multi-container pods (sidecar pattern)

Containers in one pod share network + volumes. Common pattern: **main app + sidecar** (log shipper, proxy, config reloader).

```
Pod
├── container: express      (main app, port 3000)
└── container: log-shipper  (reads shared volume, sends logs out)
```

### Check pod status

```bash
kubectl get pods                       # list all pods
kubectl get pods -o wide               # include pod IP and node
kubectl describe pod <pod-name>        # full details + events
kubectl logs <pod-name>                # app console output
kubectl exec -it <pod-name> -- sh      # shell inside pod
```

---

## 03. Deployment

> **Key Concept** — A **Deployment** is a manager for your pods. You tell it what image to run, how many replicas you want, and what resources each pod needs. It ensures that exact state is always maintained — if a pod crashes, Deployment creates a new one. If you update the image, it does a rolling update with zero downtime.

### What Deployment is responsible for

| Responsibility | What it means |
|---|---|
| **Replica management** | Always keeps the exact number of pods running. If one crashes, a new one is created immediately. |
| **Rolling update** | When you push a new image, Kubernetes replaces pods one by one. Old pods serve traffic until new ones are ready — zero downtime. |
| **Rollback** | If a new version has a bug, one command goes back to the previous working version. |
| **Scaling** | Change replica count manually or let HPA do it automatically based on CPU. |

### Full deployment.yaml with explanation

```yaml
apiVersion: apps/v1          # Deployment lives in apps group
kind: Deployment
metadata:
  name: express-deployment
spec:
  replicas: 2              # always keep 2 pods running
  selector:
    matchLabels:
      app: express          # (1) manage pods that have this label
  template:              # blueprint for each pod
    metadata:
      labels:
        app: express        # (2) stick this label on every pod created
        owner: ankur       # extra label — deployment ignores it
    spec:
      containers:
        - name: express
          image: express-k8s:latest
          imagePullPolicy: IfNotPresent  # use local image if it exists
          ports:
            - containerPort: 3000         # port your app listens on
          resources:
            limits:
              memory: "128Mi"            # max memory pod can use
              cpu: "500m"               # max CPU (500m = 0.5 core)
            requests:
              memory: "64Mi"             # memory reserved for this pod
              cpu: "250m"               # CPU reserved (required for HPA)
```

### Key fields explained

| Field | Value | What it does |
|---|---|---|
| `apiVersion` | `apps/v1` | Deployment belongs to the apps API group. Not the core group (`v1`) — that's for Pods and Services. |
| `replicas` | `2` | Always maintain 2 running pods. If one crashes, a new one is created immediately. |
| `selector.matchLabels` | `app: express` | The Deployment manages pods that have ALL these labels. Must match `template.metadata.labels`. |
| `template.metadata.labels` | `app: express` | Labels stamped on every pod this Deployment creates. Service uses these to find pods. |
| `imagePullPolicy` | `IfNotPresent` | Use local image if available. On EKS change this to `Always` so it pulls from ECR on every restart. |
| `resources.requests` | `cpu: 250m` | CPU/memory reserved for this pod. HPA requires requests to be set — it uses this to calculate utilization %. |
| `resources.limits` | `cpu: 500m` | Maximum CPU/memory the pod can use. Prevents one pod from starving others. |

> ⚠️ `spec.selector` is **immutable** after creation. If you need to change it, delete and recreate the Deployment.

### CPU units explained

| Value | Meaning |
|---|---|
| `1000m` | 1 full CPU core |
| `500m` | Half a CPU core |
| `250m` | Quarter of a CPU core |

### Memory units

| Value | Meaning |
|---|---|
| `64Mi` | 64 mebibytes (≈ 67 MB) |
| `128Mi` | 128 mebibytes |
| `1Gi` | 1 gibibyte |

### Requests vs Limits — what happens when exceeded

| Resource | Exceeding **limit** | Result |
|---|---|---|
| **CPU** | Throttled (slowed down) | Pod keeps running, but slower |
| **Memory** | Container is killed | Status `OOMKilled`, then restarted |

Requests are used by the **Scheduler** to decide where a pod fits; limits are enforced at **runtime**.

### Useful Deployment commands

```bash
kubectl get deployments
kubectl rollout status deployment/express-deployment
kubectl rollout undo deployment/express-deployment    # rollback
kubectl scale deployment express-deployment --replicas=5
kubectl rollout restart deployment/express-deployment # re-pull image
```

---

## 04. ReplicaSet

> **Key Concept** — A **ReplicaSet** is the component that guarantees *N* copies of your pod are always running. If a pod crashes, ReplicaSet creates a replacement. If you have too many pods, it deletes extras. You almost never create a ReplicaSet directly — a **Deployment creates and manages a ReplicaSet for you**, and also handles rolling updates and rollbacks on top of it.

### Deployment → ReplicaSet → Pods relationship

```
Deployment  →  owns and manages a ReplicaSet
    ↓
ReplicaSet  →  ensures replicas: 2 pods are always alive
    ↓
Pod 1 (running ✅)   Pod 2 (running ✅)
    ↓  Pod 1 crashes
ReplicaSet detects: 1 pod running, desired is 2
    ↓
ReplicaSet creates Pod 3 automatically ✅
```

### Why you don't write ReplicaSet YAML directly

| If you use | What you get |
|---|---|
| **Deployment** | ReplicaSet management + rolling updates + rollback history. Always use this in practice. |
| **ReplicaSet directly** | Pod count maintenance only. No rolling updates, no rollback. Not recommended. |

> 💡 When you run `kubectl get replicasets` you'll see an auto-generated ReplicaSet like `express-deployment-7d9f8b6c4` — the hash suffix is a fingerprint of the pod template. When you update the image, a **new ReplicaSet** is created and the old one is scaled down to zero (but kept for rollback).

### Viewing ReplicaSets

```bash
kubectl get replicasets                          # list all ReplicaSets
kubectl describe replicaset <rs-name>           # details + events

# After a rollout you will see TWO replicasets:
# express-deployment-7d9f8b6c4   2   2   2   (new — active)
# express-deployment-5c8a3d1b2   0   0   0   (old — kept for rollback)
```

### How ReplicaSet selects pods

ReplicaSet uses **label selectors** — it looks for pods with matching labels and counts them. If you manually create a pod with the same labels, the ReplicaSet will count it and may delete one of its own pods to maintain the desired count. This is why labels must be unique per Deployment.

### replicaset.yaml — for reference only, use Deployment in practice

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: express-rs
spec:
  replicas: 2                  # desired pod count
  selector:
    matchLabels:
      app: express            # manage pods with this label
  template:
    metadata:
      labels:
        app: express
    spec:
      containers:
        - name: express
          image: express-k8s:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3000
```

### Self-healing demo

```bash
kubectl get pods                         # note pod names
kubectl delete pod <one-pod-name>        # kill one
kubectl get pods -w                      # watch: a new pod appears instantly
```

---

## 05. Service

> **Key Concept** — Pods are temporary — they crash and restart with new IP addresses. A **Service** gives you a stable endpoint that never changes. It finds pods using **label selectors** and load balances traffic across all matching pods automatically. When a pod is added or removed, the Service updates its list instantly.

### How Service finds pods (Labels & Selectors)

```yaml
apiVersion: v1              # Service is a core resource
kind: Service
metadata:
  name: express-service
spec:
  selector:
    app: express            # find ALL pods with label app=express
  ports:
    - protocol: TCP
      port: 80              # port the service listens on (inside cluster)
      targetPort: 3000      # port your app listens on inside the container
  type: ClusterIP          # only accessible inside the cluster
```

### Three Service types

| Type | Accessible from | Use case |
|---|---|---|
| **ClusterIP** | Inside cluster only | Pod-to-pod communication. Default type. Used with Ingress. |
| **NodePort** | External via port number | Quick local testing. Access via `localhost:30001`. No Ingress needed. |
| **LoadBalancer** | External via cloud LB | Production on AWS/GCP. Creates real Load Balancer. Costs money. |

(There is also **ExternalName**, which maps a Service name to an external DNS name — useful for pointing at a managed DB.)

### port vs targetPort

| Field | Value | Meaning |
|---|---|---|
| `port` | `80` | The port the Service exposes inside the cluster. Other pods call this service on port 80. |
| `targetPort` | `3000` | The port your app is actually listening on inside the container. Must match `containerPort` in deployment. |

### NodePort for local testing

```yaml
type: NodePort
ports:
  - port: 80
    targetPort: 3000
    nodePort: 30001    # access via localhost:30001 in browser
```

NodePort range is **30000–32767**.

### How Service load balances

```
Service receives request (selector: app=express)
    ↓
Looks up Endpoints list (live pod IPs)
    ↓
Request 1 → Pod 1 (192.168.1.10)
Request 2 → Pod 2 (192.168.1.11)
Request 3 → Pod 3 (192.168.1.12)
Request 4 → Pod 1 (round robin) ✅
```

```bash
kubectl get services
kubectl get endpoints express-service   # see which pod IPs are in the pool
```

### Service DNS — pods talk by name

Every Service gets an internal DNS name via **CoreDNS**:

```
<service-name>.<namespace>.svc.cluster.local
```

| From where | How to call |
|---|---|
| Same namespace | `http://express-service` |
| Different namespace | `http://express-service.other-ns` |
| Full form | `http://express-service.default.svc.cluster.local` |

```js
// Inside another pod (e.g. frontend calling backend)
const res = await fetch("http://express-service/api/users"); // port 80 → targetPort 3000
```

> ⚠️ Inside a pod, `localhost` means *that pod only*. To call another service always use its **service name**, never `localhost`.

---

## 06. Ingress

> **Key Concept** — An **Ingress** resource is a set of HTTP routing rules written in YAML. It says things like: "traffic for `express.local/` should go to `express-service`" or "traffic for `express.local/auth` should go to `auth-service`". The Ingress itself does nothing — it only works when an **Ingress Controller** is installed to read and act on these rules.

### Full ingress.yaml with explanation

```yaml
apiVersion: networking.k8s.io/v1   # Ingress is in the networking group
kind: Ingress
metadata:
  name: express-ingress
spec:
  ingressClassName: nginx         # which controller handles this rule
  rules:
    - host: express.local          # domain to match
      http:
        paths:
          - path: /
            pathType: Prefix         # match / and everything after
            backend:
              service:
                name: express-service
                port:
                  number: 80
```

### Path based routing — multiple services on one domain

```yaml
rules:
  - host: express.local
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: main-service       # express.local/ → main app
              port: { number: 80 }
        - path: /auth
          pathType: Prefix
          backend:
            service:
              name: auth-service       # express.local/auth → auth service
              port: { number: 80 }
```

### Key fields explained

| Field | Value | What it does |
|---|---|---|
| `ingressClassName` | `nginx` | Tells Kubernetes which Ingress Controller should handle this rule. If you have multiple controllers (nginx, traefik), each only picks up rules meant for it. |
| `host` | `express.local` | Only traffic with this Host header matches this rule. Add it to `/etc/hosts` for local development. |
| `pathType: Prefix` | `Prefix` | Matches the path and everything after it. `/api` matches `/api/users`, `/api/data` etc. |
| `pathType: Exact` | `Exact` | Only matches that exact path. `/api` does NOT match `/api/users`. |

### For local development — skip the host

```yaml
rules:
  - http:                     # no host: field — matches all traffic
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: express-service
              port:
                number: 80
```

Access via `http://localhost` directly — no `/etc/hosts` edit needed.

### Useful nginx annotations

```yaml
metadata:
  name: express-ingress
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"        # allow larger uploads (default 1m)
    nginx.ingress.kubernetes.io/proxy-read-timeout: "120"     # seconds, for slow APIs
    nginx.ingress.kubernetes.io/rewrite-target: /$2           # strip a path prefix (see below)
    nginx.ingress.kubernetes.io/enable-cors: "true"           # CORS at ingress level
```

### Strip a path prefix (rewrite)

Want `localhost/api/users` to reach the backend as `/users`?

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /api(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: express-service
                port:
                  number: 80
```

### WebSockets (Socket.io)

nginx ingress supports WebSocket upgrade out of the box. For long-lived connections increase timeouts:

```yaml
annotations:
  nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
  nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
```

If you run **more than one backend replica with Socket.io**, you also need sticky sessions (or a Redis adapter):

```yaml
annotations:
  nginx.ingress.kubernetes.io/affinity: "cookie"
```

### TLS (HTTPS) locally — optional

```bash
# create a self-signed cert
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=express.local"

# store as a TLS secret
kubectl create secret tls express-tls --key tls.key --cert tls.crt
```

```yaml
spec:
  ingressClassName: nginx
  tls:
    - hosts: [express.local]
      secretName: express-tls
  rules:
    - host: express.local
      # ...
```

---

## 07. Ingress Controller

> **Key Concept** — The **Ingress Controller** is a pod running nginx inside your cluster. It watches all Ingress resources and configures itself to route traffic according to those rules. Without the controller, your ingress.yaml is just a config file sitting there doing nothing. The controller is what makes it actually work.

### Ingress vs Ingress Controller

| What | Ingress Resource | Ingress Controller |
|---|---|---|
| What it is | A YAML file with routing rules | An actual nginx pod running in your cluster |
| Created by | You — `kubectl apply -f ingress.yaml` | You install once — `kubectl apply -f nginx-url` |
| Does it route traffic? | No — it only defines rules | Yes — reads rules and routes real HTTP traffic |
| Namespace | `default` (your app namespace) | `ingress-nginx` (its own namespace) |

### Install nginx Ingress Controller

```bash
kubectl apply -f \
  https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.1/deploy/static/provider/cloud/deploy.yaml
```

### Verify controller is running

```bash
kubectl get pods -n ingress-nginx

# Expected output:
NAME                                        READY   STATUS
ingress-nginx-controller-xxxxxxxxx-xxxxx    1/1     Running ✅

kubectl get svc -n ingress-nginx
# ingress-nginx-controller   LoadBalancer   ...   EXTERNAL-IP: localhost
```

On Docker Desktop the controller's `LoadBalancer` Service gets `EXTERNAL-IP: localhost`, which is why `http://localhost` reaches your app on ports 80/443.

### Add express.local to hosts file

**Mac / Linux**

```bash
sudo nano /etc/hosts

# Add this line at the bottom:
127.0.0.1  express.local
```

**Windows** (open Notepad **as Administrator**)

```
File: C:\Windows\System32\drivers\etc\hosts
Line: 127.0.0.1  express.local
```

Now `http://express.local` in your browser will route to your nginx Ingress Controller on localhost.

> 💡 **ingressClassName: nginx** in your ingress.yaml is how the controller knows which rules are meant for it. If you had traefik installed too, it only picks up rules with `ingressClassName: traefik`.

### Check the controller logs (very useful for 404/502 issues)

```bash
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller
```

---

## 08. Enable Kubernetes on Docker Desktop

### Step 1 — Enable in Docker Desktop settings

Open Docker Desktop → click the **gear icon** (Settings) → click **Kubernetes** in the left sidebar → check **Enable Kubernetes** → click **Apply & Restart**.

Wait 2–3 minutes. When the bottom status bar shows a green Kubernetes icon, it is ready.

> 💡 If you have limited RAM, give Docker Desktop at least **4 GB** (Settings → Resources). Kubernetes + several pods + metrics-server will feel slow below that.

### Step 2 — Verify kubectl is working

```bash
kubectl version
kubectl get nodes

# Expected output:
NAME             STATUS   ROLES
docker-desktop   Ready    control-plane ✅
```

### Step 3 — Check current context

```bash
kubectl config current-context
# Should return: docker-desktop

kubectl config get-contexts
# Lists all clusters kubectl knows about
```

> 💡 The `~/.kube/config` file stores all cluster connection info. When you later create an EKS cluster with eksctl, it adds a new entry here and switches the current context to EKS automatically. Switch back with `kubectl config use-context docker-desktop`.

> ⚠️ **Always check your context before running `apply` or `delete`.** Running a command against the wrong cluster (e.g. production EKS instead of local) is a classic mistake.

### Optional: how local images work

Docker Desktop's Kubernetes shares the **same Docker image store** as your `docker` CLI. So an image built locally with `docker build -t express-k8s:latest .` is instantly visible to Kubernetes — no registry push needed. That's why `imagePullPolicy: IfNotPresent` works locally.

> ⚠️ If an image tag is `:latest` and `imagePullPolicy` is not set, Kubernetes defaults to `Always` and tries to pull from Docker Hub → `ImagePullBackOff`. Always set `IfNotPresent` explicitly for local images (or use a versioned tag like `v1`).

---

## 09. Writing YAML Files

> **Key Concept** — Every Kubernetes resource is defined in a YAML file. Each file must have **apiVersion**, **kind**, **metadata**, and **spec**. The `apiVersion` tells Kubernetes which API group the resource belongs to. Different resources live in different groups.

### The four mandatory top-level fields

```yaml
apiVersion: ...    # which API group/version
kind: ...          # what type of resource
metadata:          # name, labels, namespace, annotations
  name: ...
spec:              # desired state (shape depends on kind)
  ...
```

### apiVersion — which API group

| Resource | apiVersion | Why |
|---|---|---|
| Pod, Service, ConfigMap, Secret, Namespace, PVC | `v1` | Core group — the oldest, most fundamental Kubernetes resources. No group prefix needed. |
| Deployment, ReplicaSet, StatefulSet, DaemonSet | `apps/v1` | Apps group — higher-level workload resources that manage pods. |
| Ingress, NetworkPolicy | `networking.k8s.io/v1` | Networking group — resources that manage cluster networking. |
| HorizontalPodAutoscaler | `autoscaling/v2` | Autoscaling group — resources that control scaling behavior. |
| Job, CronJob | `batch/v1` | Batch group — run-to-completion and scheduled work. |

> 💡 Not sure which apiVersion? Run `kubectl explain deployment` or `kubectl api-resources`.

### Putting multiple resources in one file

```yaml
apiVersion: apps/v1
kind: Deployment
# ... deployment config

---                    # separator — new resource starts here

apiVersion: v1
kind: Service
# ... service config
```

> 💡 Keep files **separate** as your project grows. One file per resource makes it easier to find and edit things. Use `kubectl apply -f k8s/` to apply the entire folder at once.

### Generate YAML instead of typing it

```bash
# Dry-run prints YAML without creating anything
kubectl create deployment express --image=express-k8s:latest --dry-run=client -o yaml > deployment.yaml
kubectl create service clusterip express-service --tcp=80:3000 --dry-run=client -o yaml > service.yaml

# See every field available for a resource
kubectl explain deployment.spec.template.spec.containers
```

### Imperative vs Declarative

| Style | Example | Use when |
|---|---|---|
| **Imperative** | `kubectl create deployment ...`, `kubectl scale ...` | Quick experiments, learning |
| **Declarative** | `kubectl apply -f deployment.yaml` | Real projects — YAML in git is the source of truth |

### `apply` vs `create` vs `replace`

| Command | Behavior |
|---|---|
| `kubectl create -f` | Creates; **errors if it already exists** |
| `kubectl apply -f` | Creates or updates; safe to re-run. **Use this.** |
| `kubectl replace -f` | Replaces entirely; errors if it doesn't exist |

### YAML tips

- Indentation uses **spaces only** (2 spaces), never tabs.
- Lists use `-`. A list item that's an object: `- name: x` then keys aligned under `name`.
- Strings with special characters or that look like numbers/bools should be quoted: `"128Mi"`, `"true"`.
- Validate before applying: `kubectl apply -f file.yaml --dry-run=server`.

---

## 10. Labels & Selectors

> **Key Concept** — **Labels** are key-value tags on pods. **Selectors** are filters used by Deployments and Services to find pods with specific labels. This is how all three resources connect — the Deployment owns pods via selector, the Service routes to pods via selector. No hardcoded pod names or IPs anywhere.

### The three-way connection

```yaml
# deployment.yaml
selector:
  matchLabels:
    app: express      # (1) Deployment manages pods with this label
template:
  metadata:
    labels:
      app: express    # (2) Pod gets stamped with this label

---
# service.yaml
selector:
  app: express        # (3) Service routes to pods with this label
```

### Selector matching rules

| Pod Labels | Selector: `app=express` | Result |
|---|---|---|
| `app: express` | Matches | ✅ Selected |
| `app: express, env: prod` | Matches (extra label ignored) | ✅ Selected |
| `app: backend` | Does not match | ❌ Ignored |
| `app: express, env: staging` | Matches `app=express` only | ✅ Selected |

### Multiple deployments using labels

```yaml
# staging-deployment.yaml
selector:
  matchLabels:
    app: express
    env: staging     # only manages staging pods

# production-deployment.yaml
selector:
  matchLabels:
    app: express
    env: production  # only manages production pods
```

Both deployments exist in the same cluster but never interfere with each other because their selectors are different.

### Working with labels from the CLI

```bash
kubectl get pods --show-labels
kubectl get pods -l app=express                 # filter by label
kubectl get pods -l 'env in (staging,prod)'     # set-based selector
kubectl label pod <pod-name> env=debug          # add a label
kubectl label pod <pod-name> env-               # remove label "env"
```

### Labels vs Annotations

| | Labels | Annotations |
|---|---|---|
| Purpose | Identify and **select** objects | Attach extra **metadata** |
| Used by selectors? | Yes | No |
| Example | `app: express` | `nginx.ingress.kubernetes.io/proxy-body-size: "10m"` |

---

## 11. Namespaces

> **Key Concept** — A **Namespace** is a virtual cluster inside your real cluster. It groups resources (pods, services, secrets…) so names don't collide and you can separate environments like `dev`, `staging`, `prod` — or separate projects — on one cluster. If you don't specify one, everything goes in `default`.

### Default namespaces

| Namespace | What lives there |
|---|---|
| `default` | Your resources, if you don't say otherwise |
| `kube-system` | Kubernetes system pods (coredns, kube-proxy, metrics-server…) |
| `kube-public` | Publicly readable data (rarely used) |
| `kube-node-lease` | Node heartbeats |
| `ingress-nginx` | Created when you install the ingress controller |

### Create and use a namespace

```bash
kubectl create namespace dev
kubectl get namespaces

kubectl apply -f k8s/ -n dev          # apply into the dev namespace
kubectl get pods -n dev               # list pods in dev
kubectl get pods -A                   # pods in ALL namespaces

# Make dev the default for your current context
kubectl config set-context --current --namespace=dev
```

### As YAML

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

Or put it directly in a resource:

```yaml
metadata:
  name: express-deployment
  namespace: dev
```

### What is and isn't namespaced

| Namespaced (isolated per namespace) | Cluster-wide |
|---|---|
| Pod, Deployment, Service, ConfigMap, Secret, Ingress, PVC, HPA | Node, Namespace, PersistentVolume, StorageClass, IngressClass |

### Cross-namespace DNS

```
http://<service>.<namespace>.svc.cluster.local
e.g. http://express-service.dev.svc.cluster.local
```

### Limit a namespace's resource usage (optional)

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    pods: "10"
    requests.cpu: "2"
    requests.memory: 2Gi
    limits.cpu: "4"
    limits.memory: 4Gi
```

> ⚠️ Deleting a namespace **deletes everything inside it**. `kubectl delete namespace dev` is destructive.

---

## 12. ConfigMap

> **Key Concept** — A **ConfigMap** stores **non-sensitive** configuration (ports, URLs, feature flags, `NODE_ENV`) outside your Docker image. Same image can then run in dev/staging/prod with different configs. Passwords and tokens belong in a **Secret**, not here.

### Create a ConfigMap

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: express-config
data:
  NODE_ENV: "production"
  PORT: "3000"
  API_BASE_URL: "http://express-service"
  LOG_LEVEL: "info"
```

Or from the CLI:

```bash
kubectl create configmap express-config \
  --from-literal=NODE_ENV=production \
  --from-literal=PORT=3000

kubectl create configmap app-env --from-env-file=.env   # from a .env file (non-secret ones!)
```

### Use it in a Deployment — three ways

**1. One key at a time**

```yaml
env:
  - name: NODE_ENV
    valueFrom:
      configMapKeyRef:
        name: express-config
        key: NODE_ENV
```

**2. All keys at once (`envFrom`)** — easiest

```yaml
envFrom:
  - configMapRef:
      name: express-config
  - secretRef:
      name: express-secret      # secrets can be bulk-loaded the same way
```

**3. Mount as files**

```yaml
volumeMounts:
  - name: config-vol
    mountPath: /app/config
volumes:
  - name: config-vol
    configMap:
      name: express-config      # each key becomes a file in /app/config
```

### Updating a ConfigMap

| How it was consumed | After `kubectl apply` of changed ConfigMap |
|---|---|
| Env variables | **NOT updated** — restart pods: `kubectl rollout restart deployment/express-deployment` |
| Mounted file | Updated automatically after a short delay (up to ~1 min) |

### Inspect

```bash
kubectl get configmaps
kubectl describe configmap express-config
kubectl get configmap express-config -o yaml
```

---

## 13. Secrets (Local)

> **Key Concept** — A **Secret** stores sensitive data like database passwords, API keys, and JWT secrets inside the Kubernetes cluster — encoded in base64. Pods then access these values as environment variables or file mounts. Unlike a ConfigMap (which is for non-sensitive config), Secrets signal to Kubernetes that this data should be handled with care (not logged, RBAC-restricted). In a local Docker Desktop setup secrets are fine as-is; in production you would use external vaults.

> ⚠️ **base64 is NOT encryption.** It is just encoding. Anyone with `kubectl get secret` access can decode it. Never commit secret YAML files to git. Use `.gitignore` to exclude them, or use a tool like *sealed-secrets* or *external-secrets* for production.

### Step 1 — Create base64 values

```bash
# Encode any string to base64
echo -n 'mypassword' | base64
# → bXlwYXNzd29yZA==

echo -n 'mongodb://localhost:27017/mydb' | base64
# → bW9uZ29kYjovL2xvY2FsaG9zdDoyNzAxNy9teWRi

# Decode to verify
echo 'bXlwYXNzd29yZA==' | base64 --decode
```

> Always use `echo -n` (no newline). Without `-n` a hidden `\n` gets encoded and your password silently breaks.

### Step 2 — Write secret.yaml

```yaml
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: express-secret
type: Opaque                # Opaque = generic key-value secret
data:
  MONGO_URI: bW9uZ29kYjovL2xvY2FsaG9zdDoyNzAxNy9teWRi  # base64 encoded
  JWT_SECRET: bXlzdXBlcnNlY3JldGtleQ==
  DB_PASSWORD: bXlwYXNzd29yZA==
```

> 💡 **Tip — use `stringData` instead of `data`** to write plain text directly (Kubernetes encodes it for you). This is easier for local development but the result is identical in the cluster.

```yaml
# k8s/secret.yaml — using stringData (plaintext, easier locally)
apiVersion: v1
kind: Secret
metadata:
  name: express-secret
type: Opaque
stringData:                  # plain text — k8s base64-encodes automatically
  MONGO_URI: "mongodb://localhost:27017/mydb"
  JWT_SECRET: "mysupersecretkey"
  DB_PASSWORD: "mypassword"
```

### Create a secret straight from the CLI (no YAML file, nothing to commit)

```bash
kubectl create secret generic express-secret \
  --from-literal=JWT_SECRET=mysupersecretkey \
  --from-literal=DB_PASSWORD=mypassword

kubectl create secret generic app-env --from-env-file=.env.secret
```

### Step 3 — Apply the Secret

```bash
kubectl apply -f k8s/secret.yaml

# Verify it was created
kubectl get secrets
kubectl describe secret express-secret   # shows keys but NOT values

# Decode a specific value to verify
kubectl get secret express-secret -o jsonpath='{.data.MONGO_URI}' | base64 --decode
```

### Step 4 — Inject Secret into your Deployment as env vars

```yaml
spec:
  containers:
    - name: express
      image: express-k8s:latest
      env:
        - name: MONGO_URI          # name of env var inside the container
          valueFrom:
            secretKeyRef:
              name: express-secret # secret name from metadata.name
              key: MONGO_URI         # key inside the secret
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: express-secret
              key: JWT_SECRET
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: express-secret
              key: DB_PASSWORD
```

### Shortcut — load every key at once

```yaml
envFrom:
  - secretRef:
      name: express-secret     # all keys become env vars with the same names
```

### Mount a Secret as files (e.g. certificates)

```yaml
volumeMounts:
  - name: secret-vol
    mountPath: /etc/secrets
    readOnly: true
volumes:
  - name: secret-vol
    secret:
      secretName: express-secret   # each key → a file in /etc/secrets
```

### Access in your Node.js/Express code

```js
// Kubernetes injects secrets as environment variables
const mongoUri   = process.env.MONGO_URI;
const jwtSecret  = process.env.JWT_SECRET;
const dbPassword = process.env.DB_PASSWORD;

// Same code works locally with .env and in Kubernetes with Secrets
```

### Secret vs ConfigMap — when to use which

| Use case | ConfigMap | Secret |
|---|---|---|
| Database password | ❌ Never | ✅ Yes |
| API base URL | ✅ Yes | Not needed |
| JWT secret key | ❌ Never | ✅ Yes |
| `NODE_ENV = production` | ✅ Yes | Not needed |
| MongoDB connection string | ❌ if it has password | ✅ Yes |
| Port number | ✅ Yes | Not needed |

### Secret types

| Type | Used for |
|---|---|
| `Opaque` | Generic key-value (default) |
| `kubernetes.io/tls` | TLS cert + key (for Ingress HTTPS) |
| `kubernetes.io/dockerconfigjson` | Credentials to pull from a private registry (ECR, Docker Hub) |

```bash
# Private registry pull secret
kubectl create secret docker-registry regcred \
  --docker-server=<registry-url> \
  --docker-username=<user> \
  --docker-password=<password>
```
```yaml
spec:
  imagePullSecrets:
    - name: regcred
```

### Add secret.yaml to your project structure

```
your-project/
├── Backend/
│   ├── server.js
│   ├── package.json
│   └── dockerfile
└── k8s/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── hpa.yaml
    └── secret.yaml       # ← add to .gitignore !
```

> ⚠️ **Always add secret.yaml to .gitignore.** Committing secrets to git — even encoded base64 — is a serious security risk. Add `k8s/secret.yaml` to your `.gitignore` file. Share secrets with teammates via a secure channel (1Password, Bitwarden, private Slack DM), never via git.

### Apply order matters

```bash
# 1. Secret (and ConfigMap) must exist before pods that reference them start up
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/configmap.yaml

# 2. Then apply deployment (pods can now find the secret)
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/ingress.yaml

# Verify env vars were injected into the pod
kubectl exec -it <pod-name> -- sh
# Inside pod:
echo $MONGO_URI
echo $JWT_SECRET
```

> If a pod references a missing Secret/ConfigMap it gets stuck in **`CreateContainerConfigError`**. Create the secret, and the pod recovers on its own.

> Secrets mounted as env vars are **not** refreshed when the Secret changes — run `kubectl rollout restart deployment/express-deployment`.

---

## 14. Health Probes

> **Key Concept** — Kubernetes can't know if your app is actually healthy just because the container process is running. **Probes** are small checks (HTTP request, TCP connect, or command) that tell Kubernetes whether to restart a container or send it traffic.

### The three probes

| Probe | Question it answers | If it fails |
|---|---|---|
| **startupProbe** | Has the app finished starting? | Keep waiting; kill after too many failures. Disables the other two until it passes. |
| **readinessProbe** | Can this pod take traffic *right now*? | Pod removed from Service endpoints (**no restart**) |
| **livenessProbe** | Is the app stuck/dead? | Container is **restarted** |

### Add a health endpoint to Express

```js
app.get("/health", (req, res) => res.status(200).json({ status: "ok" }));
```

For readiness, check real dependencies:

```js
app.get("/ready", async (req, res) => {
  const dbOk = mongoose.connection.readyState === 1;   // 1 = connected
  res.status(dbOk ? 200 : 503).json({ db: dbOk });
});
```

### Probes in deployment.yaml

```yaml
containers:
  - name: express
    image: express-k8s:latest
    ports:
      - containerPort: 3000
    startupProbe:
      httpGet:
        path: /health
        port: 3000
      failureThreshold: 30       # 30 × 2s = up to 60s to start
      periodSeconds: 2
    readinessProbe:
      httpGet:
        path: /ready
        port: 3000
      initialDelaySeconds: 5
      periodSeconds: 10
      failureThreshold: 3
    livenessProbe:
      httpGet:
        path: /health
        port: 3000
      initialDelaySeconds: 15
      periodSeconds: 20
      failureThreshold: 3
```

### Probe fields

| Field | Meaning |
|---|---|
| `initialDelaySeconds` | Wait this long after container start before first check |
| `periodSeconds` | How often to check |
| `timeoutSeconds` | How long to wait for a response (default 1s) |
| `failureThreshold` | Consecutive failures before acting |
| `successThreshold` | Consecutive successes to be considered healthy again |

### Other probe types

```yaml
# TCP — just check the port accepts connections
livenessProbe:
  tcpSocket:
    port: 3000

# Command — exit code 0 = healthy
livenessProbe:
  exec:
    command: ["cat", "/tmp/healthy"]
```

> ⚠️ **Don't check external dependencies in the liveness probe.** If MongoDB goes down and liveness fails, Kubernetes restarts *all* your pods for no reason. Put dependency checks in **readiness**; keep liveness simple (is the process responsive?).

> 💡 Readiness is what makes **rolling updates zero-downtime** — a new pod only gets traffic once its readiness probe passes.

---

## 15. Rolling Updates & Rollbacks

> **Key Concept** — When you change the pod template (image, env, resources), the Deployment creates a **new ReplicaSet** and gradually shifts pods from old to new. You control how aggressive this is with the update strategy.

### Strategy settings

```yaml
spec:
  replicas: 4
  revisionHistoryLimit: 5          # how many old ReplicaSets to keep for rollback
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1                  # can create 1 extra pod above desired during update
      maxUnavailable: 0            # never drop below desired count → zero downtime
```

| Field | Meaning |
|---|---|
| `maxSurge` | How many pods *above* `replicas` are allowed during update (number or %) |
| `maxUnavailable` | How many pods may be *down* during update (number or %) |
| `type: Recreate` | Kill all old pods first, then start new ones (has downtime — used when two versions can't coexist) |

### What a rolling update looks like (replicas: 3, maxSurge 1, maxUnavailable 0)

```
Old: [v1][v1][v1]
Step 1: [v1][v1][v1] + [v2 starting]
Step 2: [v1][v1][v2 ready] → kill one v1
Step 3: [v1][v2][v2] + [v2 starting] → ...
Done:   [v2][v2][v2]
```

### Trigger, watch, rollback

```bash
# Update image (or just edit YAML and apply)
kubectl set image deployment/express-deployment express=express-k8s:v2

# Watch progress
kubectl rollout status deployment/express-deployment

# History
kubectl rollout history deployment/express-deployment
kubectl rollout history deployment/express-deployment --revision=2

# Rollback
kubectl rollout undo deployment/express-deployment                  # to previous
kubectl rollout undo deployment/express-deployment --to-revision=1  # to a specific one

# Pause / resume (batch several changes into a single rollout)
kubectl rollout pause deployment/express-deployment
kubectl rollout resume deployment/express-deployment
```

### Use version tags, not just `:latest`

```bash
docker build -t express-k8s:v1 .
docker build -t express-k8s:v2 .
```

With `:latest`, Kubernetes sees *no change* in the YAML, so `kubectl apply` does nothing — you must run `rollout restart`. With unique tags, changing the tag in YAML automatically triggers a rollout and makes rollback meaningful.

### Graceful shutdown (so in-flight requests don't die)

Kubernetes sends `SIGTERM`, waits `terminationGracePeriodSeconds` (default 30s), then `SIGKILL`.

```js
process.on("SIGTERM", () => {
  console.log("SIGTERM received, closing server...");
  server.close(() => {
    mongoose.connection.close(false).then(() => process.exit(0));
  });
});
```

```yaml
spec:
  terminationGracePeriodSeconds: 30
```

---

## 16. Volumes & Persistent Storage

> **Key Concept** — A container's filesystem disappears when the pod is deleted or restarted. For temporary scratch space or for data that must survive (MongoDB, uploads), you attach a **Volume**.

### Volume types you'll meet

| Type | Lifetime | Use case |
|---|---|---|
| `emptyDir` | Lives as long as the **pod** | Scratch space, sharing files between containers in a pod |
| `hostPath` | Lives on the **node's disk** | Local dev only (mount a folder from your machine). Avoid in production. |
| `configMap` / `secret` | Config as files | See ConfigMap / Secret sections |
| **PersistentVolumeClaim** | Lives **beyond the pod** | Databases, uploads — real persistence |

### emptyDir

```yaml
spec:
  containers:
    - name: express
      volumeMounts:
        - name: tmp-data
          mountPath: /tmp/work
  volumes:
    - name: tmp-data
      emptyDir: {}
```

### PV, PVC, StorageClass — how persistence works

```
Pod  →  PersistentVolumeClaim (PVC)  →  PersistentVolume (PV)  →  actual disk
        "I need 1Gi"                    "here's 1Gi"
                          ↑
                    StorageClass auto-creates the PV on demand (dynamic provisioning)
```

| Object | Who creates | Meaning |
|---|---|---|
| **PersistentVolume (PV)** | Admin or StorageClass (automatically) | A piece of storage in the cluster |
| **PersistentVolumeClaim (PVC)** | You | A request for storage: size + access mode |
| **StorageClass** | Cluster | Defines *how* storage gets provisioned. Docker Desktop has a default (`hostpath`) |

### PVC example

```yaml
# k8s/mongo-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pvc
spec:
  accessModes:
    - ReadWriteOnce          # mounted read-write by ONE node
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl get storageclass
kubectl get pvc
kubectl get pv
```

### Use the PVC in a pod

```yaml
containers:
  - name: mongo
    image: mongo:7
    volumeMounts:
      - name: mongo-data
        mountPath: /data/db
volumes:
  - name: mongo-data
    persistentVolumeClaim:
      claimName: mongo-pvc
```

### Access modes

| Mode | Meaning |
|---|---|
| `ReadWriteOnce` (RWO) | One node can mount read-write |
| `ReadOnlyMany` (ROX) | Many nodes, read-only |
| `ReadWriteMany` (RWX) | Many nodes read-write (needs special storage like NFS/EFS) |

> ⚠️ Deleting a PVC may delete the data (depends on the PV's reclaim policy; the dynamic default is usually `Delete`).

> 💡 For production databases prefer a **managed service** (MongoDB Atlas, RDS, ElastiCache). Running databases in Kubernetes is fine for local learning but operationally hard in production.

---

## 17. Deploy Your App

### Step 1 — Build Docker image

```bash
cd Backend/
docker build -t express-k8s:latest .
```

A minimal `dockerfile` for the Express app:

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

Add a `.dockerignore`:

```
node_modules
.env
.git
k8s
```

> The app must listen on `0.0.0.0` (or just `app.listen(3000)`), not only `127.0.0.1`, or the Service can't reach it.

### Step 2 — Apply all YAML files

```bash
kubectl apply -f k8s/    # applies all files in the k8s folder

# Or apply individually:
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/ingress.yaml
```

### Step 3 — Verify everything is running

```bash
kubectl get pods           # STATUS: Running, READY: 1/1
kubectl get services       # service exists with correct ports
kubectl get ingress        # ingress has an ADDRESS
```

### Step 4 — Access in browser

| Setup | URL |
|---|---|
| No host in ingress | `http://localhost` |
| `host: express.local` (with hosts file) | `http://express.local` |
| NodePort service | `http://localhost:30001` |

### Update your app

```bash
# 1. Rebuild image after code changes
docker build -t express-k8s:latest .

# 2. Restart deployment to use new image
kubectl rollout restart deployment/express-deployment
```

### Delete what you deployed

```bash
kubectl delete -f k8s/                              # delete everything defined in the folder
kubectl delete deployment express-deployment        # delete one resource
```

---

## 18. Full MERN Stack on Local Kubernetes

> **Key Concept** — Real apps have multiple parts. Each part gets its own Deployment + Service, and they talk to each other using **service names**. Only the Ingress is exposed outside.

### Target architecture

```
                       http://localhost
                              │
                    ┌─────────▼─────────┐
                    │  Ingress (nginx)  │
                    └───┬───────────┬───┘
                 /      │           │   /api
              ┌─────────▼──┐   ┌────▼──────────┐
              │ frontend   │   │ backend       │
              │ (React +   │   │ (Express)     │
              │  nginx)    │   │ svc: backend  │
              └────────────┘   └───┬───────┬───┘
                                   │       │
                          ┌────────▼──┐ ┌──▼────────┐
                          │ mongo     │ │ redis     │
                          │ + PVC     │ │           │
                          └───────────┘ └───────────┘
```

### Folder layout

```
your-project/
├── Backend/    (dockerfile)
├── Frontend/   (dockerfile)
└── k8s/
    ├── namespace.yaml
    ├── configmap.yaml
    ├── secret.yaml            # gitignored
    ├── mongo.yaml             # PVC + Deployment + Service
    ├── redis.yaml             # Deployment + Service
    ├── backend.yaml           # Deployment + Service
    ├── frontend.yaml          # Deployment + Service
    └── ingress.yaml
```

### MongoDB (PVC + Deployment + Service in one file)

```yaml
# k8s/mongo.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo
spec:
  replicas: 1                      # a single mongod — do NOT scale this
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
        - name: mongo
          image: mongo:7
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: data
              mountPath: /data/db
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: mongo-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: mongo                      # ← this becomes the hostname
spec:
  selector:
    app: mongo
  ports:
    - port: 27017
      targetPort: 27017
```

### Redis

```yaml
# k8s/redis.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          ports:
            - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  selector:
    app: redis
  ports:
    - port: 6379
      targetPort: 6379
```

### Connection strings inside the cluster

```
MONGO_URI = mongodb://mongo:27017/mydb       # "mongo" = Service name
REDIS_URL = redis://redis:6379               # "redis" = Service name
```

Put these in the ConfigMap / Secret — **not** `localhost`.

### Backend

```yaml
# k8s/backend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: express-k8s:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: express-config
            - secretRef:
                name: express-secret
          readinessProbe:
            httpGet: { path: /ready, port: 3000 }
            initialDelaySeconds: 5
          resources:
            requests: { cpu: 250m, memory: 64Mi }
            limits:   { cpu: 500m, memory: 256Mi }
---
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 3000
```

### Frontend (React build served by nginx)

```dockerfile
# Frontend/dockerfile — multi-stage build
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build            # outputs dist/ (Vite) or build/ (CRA)

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
# SPA fallback so React Router routes work on refresh
RUN printf 'server {\n  listen 80;\n  location / {\n    root /usr/share/nginx/html;\n    try_files $uri /index.html;\n  }\n}\n' > /etc/nginx/conf.d/default.conf
EXPOSE 80
```

```yaml
# k8s/frontend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: frontend-k8s:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
```

> ⚠️ React runs in the **user's browser**, not inside the cluster. It can't call `http://backend` (that name only exists inside the cluster). Have the frontend call a relative path like `/api/...` and let the Ingress route `/api` to the backend.

### Ingress — one entry point for both

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend
                port: { number: 80 }
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port: { number: 80 }
```

nginx picks the **longest matching path**, so `/api/...` goes to backend and everything else goes to frontend.

### Deploy it all in the right order

```bash
docker build -t express-k8s:latest ./Backend
docker build -t frontend-k8s:latest ./Frontend

kubectl apply -f k8s/namespace.yaml        # if using one
kubectl apply -f k8s/configmap.yaml -f k8s/secret.yaml
kubectl apply -f k8s/mongo.yaml -f k8s/redis.yaml
kubectl apply -f k8s/backend.yaml -f k8s/frontend.yaml
kubectl apply -f k8s/ingress.yaml

kubectl get all
```

> 💡 Pods start in any order. If the backend starts before Mongo is ready it may crash and restart a few times — that's normal. Make your app **retry the DB connection**, and use a readiness probe.

---

## 19. kubectl Essentials & Port-Forward

### Anatomy of a command

```
kubectl  <verb>   <resource>   <name>        <flags>
kubectl  get      pods                        -n dev -o wide
kubectl  delete   deployment   express-deployment
```

### Common verbs

| Verb | What it does |
|---|---|
| `get` | List resources |
| `describe` | Detailed info + events |
| `apply` / `create` | Create or update from file |
| `delete` | Remove a resource |
| `logs` | Container output |
| `exec` | Run a command in a container |
| `edit` | Open live resource in your editor |
| `scale` | Change replicas |
| `rollout` | Manage Deployment rollouts |
| `port-forward` | Tunnel a local port to a pod/service |
| `top` | CPU/memory usage (needs metrics-server) |
| `explain` | Documentation for any field |

### Output formats

```bash
kubectl get pods -o wide                 # extra columns (IP, node)
kubectl get pods -o yaml                 # full YAML
kubectl get pod <name> -o json
kubectl get pods -o name                 # just names
kubectl get pods -w                      # watch live
kubectl get pods --sort-by=.metadata.creationTimestamp
kubectl get svc express-service -o jsonpath='{.spec.clusterIP}'
```

### Port-forward — reach anything without Ingress

```bash
kubectl port-forward pod/<pod-name> 8080:3000          # localhost:8080 → pod:3000
kubectl port-forward svc/express-service 8080:80       # localhost:8080 → service:80
kubectl port-forward svc/mongo 27017:27017             # connect MongoDB Compass to localhost:27017
```

Great for debugging a service directly, or opening your in-cluster Mongo with Compass. Press `Ctrl+C` to stop.

### Copy files in and out of a pod

```bash
kubectl cp <pod-name>:/app/logs/app.log ./app.log
kubectl cp ./config.json <pod-name>:/app/config.json
```

### Temporary debug pod

```bash
# Throwaway pod with curl/nslookup — deleted when you exit
kubectl run debug --rm -it --image=busybox:1.36 -- sh
# inside:
nslookup express-service
wget -qO- http://express-service/health

# Or with curl available
kubectl run curl --rm -it --image=curlimages/curl -- sh
```

### Edit a live resource

```bash
kubectl edit deployment express-deployment
```

> Changes made this way are **not** in your YAML files — update the file too or the next `apply` will overwrite them.

### Shell shortcuts

```bash
# Short resource names
kubectl get po          # pods
kubectl get svc         # services
kubectl get deploy      # deployments
kubectl get rs          # replicasets
kubectl get ing         # ingress
kubectl get cm          # configmaps
kubectl get ns          # namespaces
kubectl get hpa         # horizontal pod autoscalers
kubectl get pvc         # persistent volume claims

# Alias k for kubectl (add to ~/.zshrc or ~/.bashrc)
alias k=kubectl
```

---

## 20. Debug Commands & Troubleshooting

### Common errors and what to do

| Error | Cause | Debug command |
|---|---|---|
| **ImagePullBackOff** / **ErrImagePull** | Cannot pull Docker image. Usually local image not built or wrong name/tag. | `kubectl describe pod <name>` → check Events section |
| **CrashLoopBackOff** | Container keeps crashing. App has a startup error. | `kubectl logs <pod-name> --previous` → see app error output |
| **CreateContainerConfigError** | Referenced Secret/ConfigMap/key doesn't exist. | `kubectl describe pod <name>` → Events; create the missing secret |
| **OOMKilled** | Container exceeded memory limit. | `kubectl describe pod <name>` → Last State: OOMKilled; raise `limits.memory` or fix leak |
| **503 Service Unavailable** | Ingress controller running but no pods found. Label mismatch or pods not ready. | `kubectl get endpoints <service>` → if `<none>` labels don't match |
| **502 Bad Gateway** | Ingress reached the pod but app crashed/refused, or wrong targetPort. | Check `targetPort`, pod logs, ingress controller logs |
| **404 Not Found** | nginx running but no matching ingress rule for that host/path. | `kubectl describe ingress <name>` → check rules |
| **Pending (pod)** | No node has enough CPU/memory to schedule the pod. | `kubectl describe pod <name>` → check Events |
| **Pending (PVC)** | No StorageClass / can't provision volume. | `kubectl describe pvc <name>` |
| **ECONNREFUSED** | Pod calling another service using `localhost` instead of service name. | Use `http://service-name` not `http://localhost:PORT` |
| **Evicted** | Node ran low on resources. | `kubectl describe pod <name>`; set proper requests/limits |
| **Running but 0/1 READY** | Readiness probe failing. | `kubectl describe pod <name>` → probe failure events |

### Essential debug commands

```bash
# Check what is running
kubectl get pods
kubectl get services
kubectl get ingress
kubectl get all                              # everything at once
kubectl get events --sort-by=.lastTimestamp  # recent cluster events

# Deep inspect
kubectl describe pod <pod-name>              # events + full config
kubectl describe ingress <ingress-name>      # routing rules + backend IPs
kubectl describe service <service-name>      # selector + endpoints

# Logs
kubectl logs <pod-name>                      # app output
kubectl logs <pod-name> -f                   # follow live
kubectl logs <pod-name> --previous           # logs from crashed container
kubectl logs <pod-name> -c <container>       # multi-container pod
kubectl logs -l app=express --tail=50        # logs from all pods with a label
kubectl logs deploy/express-deployment       # logs from a deployment's pod

# Network debug
kubectl get endpoints <service-name>         # pod IPs in service pool
kubectl exec -it <pod-name> -- sh            # shell inside pod
# Inside pod: test another service
wget -qO- http://some-service/api/data
```

### Systematic troubleshooting flow

```
App not reachable?
│
├─ 1. kubectl get pods
│     ├─ Pending            → describe pod (resources? PVC? image?)
│     ├─ ImagePullBackOff   → image name/tag? built locally? pullPolicy?
│     ├─ CrashLoopBackOff   → kubectl logs --previous (app error?)
│     ├─ Running 0/1        → readiness probe failing → describe pod
│     └─ Running 1/1 ✅     → go to step 2
│
├─ 2. kubectl get endpoints <service>
│     ├─ <none>             → Service selector ≠ Pod labels
│     └─ IPs listed ✅      → go to step 3
│
├─ 3. kubectl port-forward svc/<service> 8080:80 → curl localhost:8080
│     ├─ fails              → targetPort wrong / app listening on wrong port or 127.0.0.1
│     └─ works ✅           → Service is fine, problem is Ingress → step 4
│
└─ 4. kubectl describe ingress + ingress controller logs
      ├─ 404                → host/path mismatch, wrong ingressClassName
      ├─ 503                → backend service name/port wrong
      └─ no ADDRESS         → controller not installed/running
```

### Quick label/port sanity checklist

- Deployment `selector.matchLabels` == `template.metadata.labels`
- Service `selector` == Pod labels
- Service `targetPort` == container's listening port
- Ingress backend `service.name` and `port.number` == Service name and `port`
- Image exists locally: `docker images | grep express-k8s`

---

## 21. Autoscaling (HPA)

> **Key Concept** — The **HorizontalPodAutoscaler (HPA)** watches CPU usage across your pods and automatically scales the replica count up or down. When CPU exceeds your threshold it adds pods. When traffic drops it removes them. It requires **metrics-server** to read CPU data — on Docker Desktop this is *not* pre-installed and must be added manually.

### Step 0 — Install Metrics Server (required for HPA)

> ⚠️ **HPA will not work without metrics-server.** Running `kubectl top pods` will return *"Metrics API not available"* and HPA will show `<unknown>/50%` for CPU until metrics-server is installed and running.

**Option A: Official manifest with kubelet insecure TLS patch (recommended for Docker Desktop)**

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Docker Desktop uses a self-signed cert — patch metrics-server to skip TLS verify:
kubectl patch deployment metrics-server -n kube-system \
  --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

# Wait for it to be ready (~30 seconds)
kubectl rollout status deployment/metrics-server -n kube-system

# Verify it works — you should see CPU and Memory columns
kubectl top nodes
kubectl top pods
```

**Option B: Helm install (if you have Helm)**

```bash
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm repo update
helm upgrade --install metrics-server metrics-server/metrics-server \
  --namespace kube-system \
  --set args={--kubelet-insecure-tls}
```

> 💡 **Where does metrics-server run?** It runs as a pod in the `kube-system` namespace on the master/control-plane node. Verify with: `kubectl get pods -n kube-system | grep metrics`. You should see `metrics-server-xxxx   1/1   Running`.

> `--kubelet-insecure-tls` is for **local development only**. Never use it on a real cluster.

### Create HPA via command

```bash
kubectl autoscale deployment express-deployment \
  --min=1 \
  --max=5 \
  --cpu-percent=50
```

### Or as a YAML file (recommended)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: express-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: express-deployment    # must match deployment name
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50   # scale when CPU > 50%
```

### Add memory as a second metric (optional)

```yaml
metrics:
  - type: Resource
    resource:
      name: cpu
      target: { type: Utilization, averageUtilization: 50 }
  - type: Resource
    resource:
      name: memory
      target: { type: Utilization, averageUtilization: 70 }
```

HPA computes desired replicas for each metric and uses the **highest**.

### The formula

```
desiredReplicas = ceil( currentReplicas × currentUtilization / targetUtilization )

Example: 2 pods at 100% CPU, target 50%
         → ceil(2 × 100 / 50) = 4 pods
```

### HPA scaling flow

```
Traffic increases → CPU per pod goes above 50%
    ↓
HPA adds new pod (up to maxReplicas: 5)
    ↓
Requests spread across more pods
    ↓
CPU per pod drops — each pod handles less ✅
    ↓
Traffic drops → CPU goes low
    ↓
HPA removes extra pods (down to minReplicas: 1) ✅
```

> ⚠️ **resources.requests must be set** in your deployment for HPA to work. HPA calculates utilization as *current CPU / requested CPU × 100*. Without requests defined, HPA has no baseline to calculate from and will not scale.

### Scale-down is deliberately slow

By default HPA waits ~**5 minutes** of low usage before scaling down (stabilization window) to avoid flapping. You can tune it:

```yaml
spec:
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 60
```

### Generate load to test it

```bash
# Terminal 1 — watch HPA
kubectl get hpa -w

# Terminal 2 — blast the service with requests from inside the cluster
kubectl run load-gen --rm -it --image=busybox:1.36 -- \
  /bin/sh -c "while true; do wget -q -O- http://express-service/; done"

# Terminal 3 — watch pods appear
kubectl get pods -w
```

Stop the load generator (`Ctrl+C`) and watch replicas shrink after the cooldown.

```bash
kubectl get hpa              # see current CPU%, desired vs actual replicas
kubectl top pods             # real-time CPU and memory per pod
kubectl describe hpa express-hpa   # events + why it scaled
```

> If you use HPA, **remove `replicas:` from your Deployment YAML** (or accept that `kubectl apply` will reset the count each time).

---

## 22. Other Workload Types

Deployments are for **stateless** apps. Kubernetes has other controllers for other jobs.

| Kind | Use for | Key behavior |
|---|---|---|
| **Deployment** | Stateless apps (APIs, frontends) | Interchangeable pods, rolling updates |
| **StatefulSet** | Databases, Kafka, anything needing stable identity/storage | Pods named `db-0`, `db-1`…; each gets its own PVC; ordered start/stop |
| **DaemonSet** | One pod **per node** (log collectors, monitoring agents) | Auto-adds a pod when a node joins |
| **Job** | Run-to-completion task (migration, one-off script) | Retries until success, then stops |
| **CronJob** | Scheduled Jobs | Cron syntax |

### Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-seed
spec:
  backoffLimit: 3                  # retry up to 3 times
  template:
    spec:
      restartPolicy: Never         # Jobs must use Never or OnFailure
      containers:
        - name: seed
          image: express-k8s:latest
          command: ["node", "seed.js"]
          envFrom:
            - secretRef:
                name: express-secret
```

### CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-cleanup
spec:
  schedule: "0 2 * * *"            # every day at 02:00
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: cleanup
              image: express-k8s:latest
              command: ["node", "cleanup.js"]
```

```
Cron format:  ┌ minute (0-59)
              │ ┌ hour (0-23)
              │ │ ┌ day of month (1-31)
              │ │ │ ┌ month (1-12)
              │ │ │ │ ┌ day of week (0-6, Sun=0)
              * * * * *
```

```bash
kubectl get jobs
kubectl get cronjobs
kubectl create job manual-run --from=cronjob/nightly-cleanup   # trigger a CronJob right now
```

### StatefulSet (short example)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongo
spec:
  serviceName: mongo-headless      # needs a headless Service (clusterIP: None)
  replicas: 1
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
        - name: mongo
          image: mongo:7
          volumeMounts:
            - name: data
              mountPath: /data/db
  volumeClaimTemplates:            # each replica gets its OWN PVC automatically
    - metadata:
        name: data
      spec:
        accessModes: [ReadWriteOnce]
        resources:
          requests:
            storage: 1Gi
```

---

## 23. Helm & Kustomize

### Why they exist

Copy-pasting YAML for dev/staging/prod gets messy. Two common solutions:

| Tool | Idea |
|---|---|
| **Helm** | Package manager. YAML **templates** + a `values.yaml`. Install ready-made charts (nginx-ingress, mongodb, redis, prometheus…). |
| **Kustomize** | Built into kubectl. Keep a **base** YAML and apply small **overlays** (patches) per environment. No templating. |

### Helm basics

```bash
# Install Helm (Mac)
brew install helm

helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo redis

helm install my-redis bitnami/redis                      # install a chart
helm install my-redis bitnami/redis -f my-values.yaml    # with custom values
helm install my-redis bitnami/redis --set auth.enabled=false

helm list                                                # installed releases
helm upgrade my-redis bitnami/redis -f my-values.yaml
helm rollback my-redis 1
helm uninstall my-redis
```

Install nginx ingress via Helm instead of raw manifest:

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

### Kustomize basics

```
k8s/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    └── prod/
        └── kustomization.yaml
```

```yaml
# base/kustomization.yaml
resources:
  - deployment.yaml
  - service.yaml
```

```yaml
# overlays/prod/kustomization.yaml
resources:
  - ../../base
replicas:
  - name: express-deployment
    count: 4
images:
  - name: express-k8s
    newTag: v2
```

```bash
kubectl kustomize k8s/overlays/prod      # preview final YAML
kubectl apply -k k8s/overlays/prod       # apply it
```

---

## 24. Handy Tools

| Tool | What it does |
|---|---|
| **Docker Desktop Kubernetes dashboard / Containers view** | Quick look at what's running |
| **k9s** | Terminal UI for the cluster — browse pods, logs, shells with keyboard shortcuts. `brew install k9s` |
| **Lens / OpenLens** | Desktop GUI for clusters |
| **kubectx / kubens** | Switch context / namespace fast: `kubectx docker-desktop`, `kubens dev` |
| **stern** | Tail logs from many pods at once: `stern express` |
| **kubectl neat** | Strip noisy fields from `-o yaml` output |
| **Kubernetes Dashboard** | Official web UI (extra install) |
| **VS Code Kubernetes extension** | YAML validation, cluster explorer |
| **kubeconform / kubeval** | Validate YAML in CI before applying |
| **Skaffold / Tilt** | Auto rebuild + redeploy on code change (inner-loop dev) |

### kubectl autocomplete

```bash
# zsh
echo 'source <(kubectl completion zsh)' >> ~/.zshrc
echo 'alias k=kubectl' >> ~/.zshrc
echo 'compdef __start_kubectl k' >> ~/.zshrc

# bash
echo 'source <(kubectl completion bash)' >> ~/.bashrc
```

---

## 25. Cleanup & Reset

```bash
# Delete resources you created from YAML
kubectl delete -f k8s/

# Delete by type/name
kubectl delete deployment express-deployment
kubectl delete svc express-service
kubectl delete hpa express-hpa

# Delete everything in the current namespace (careful!)
kubectl delete all --all

# Delete a whole namespace and everything in it
kubectl delete namespace dev

# Remove the ingress controller
kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.1/deploy/static/provider/cloud/deploy.yaml

# Clean unused Docker images/containers to free disk
docker system prune -a
```

> `kubectl delete all --all` does **not** remove ConfigMaps, Secrets, PVCs or Ingress — delete them separately.

### Full reset of the local cluster

Docker Desktop → Settings → Kubernetes → **Reset Kubernetes Cluster**. This wipes every resource and gives you a fresh single-node cluster (you'll need to reinstall ingress-nginx and metrics-server).

### Pause Kubernetes to save RAM

Docker Desktop → Settings → Kubernetes → uncheck **Enable Kubernetes** → Apply. Re-enable later (your YAML files are safe, but cluster state is lost).

---

## 26. From Local to EKS

What changes when you move the same YAML from Docker Desktop to AWS EKS:

| Topic | Local (Docker Desktop) | EKS (AWS) |
|---|---|---|
| **Images** | Local Docker store, `imagePullPolicy: IfNotPresent` | Push to **ECR**, use full image URL, `imagePullPolicy: Always` |
| **Image name** | `express-k8s:latest` | `<account>.dkr.ecr.<region>.amazonaws.com/express-k8s:v1` |
| **Nodes** | 1 node | Multiple nodes (managed node group / Fargate) |
| **Ingress controller** | nginx, LoadBalancer = `localhost` | nginx or **AWS Load Balancer Controller (ALB)**; real public DNS |
| **Service type LoadBalancer** | Maps to localhost | Creates a real AWS ELB (costs money) |
| **Storage** | hostpath StorageClass | EBS (`gp3`) for RWO, EFS for RWX |
| **Secrets** | Plain k8s Secrets | AWS Secrets Manager + External Secrets / CSI driver |
| **Domain / TLS** | `/etc/hosts`, self-signed | Route 53 + ACM / cert-manager |
| **Metrics-server** | Install manually with `--kubelet-insecure-tls` | Install normally (no insecure flag) |
| **Context** | `docker-desktop` | Added by `aws eks update-kubeconfig` / eksctl |

```bash
# Switching contexts
kubectl config get-contexts
kubectl config use-context docker-desktop
kubectl config use-context <eks-context-name>
```

> Always run `kubectl config current-context` before `apply`/`delete`.

---

## 27. Best Practices Checklist

**Images & containers**
- [ ] Use specific image tags (`v1.2.3`), not just `:latest`
- [ ] Small base images (`node:20-alpine`), `.dockerignore` in place
- [ ] App listens on `0.0.0.0` and handles `SIGTERM` gracefully
- [ ] Run as non-root user where possible

**Deployments**
- [ ] `resources.requests` **and** `limits` set on every container
- [ ] `readinessProbe` (and a simple `livenessProbe`) configured
- [ ] `replicas >= 2` for anything that needs availability
- [ ] Labels consistent: `selector` == `template.labels` == Service `selector`
- [ ] Rolling update tuned (`maxUnavailable: 0` for zero downtime)

**Config & secrets**
- [ ] Non-secret config in **ConfigMap**, sensitive in **Secret**
- [ ] Secret YAML files in `.gitignore`
- [ ] Never hardcode connection strings/passwords in the image or YAML in git
- [ ] Restart pods after changing env-based ConfigMap/Secret

**Networking**
- [ ] Services talk via **service names**, never `localhost`
- [ ] One Ingress as the single external entry point
- [ ] `ingressClassName` set explicitly

**Operations**
- [ ] Check `kubectl config current-context` before destructive commands
- [ ] Use namespaces to separate environments / projects
- [ ] YAML files live in git (one file per resource, grouped in `k8s/`)
- [ ] `kubectl apply -f` (declarative) over ad-hoc `kubectl edit`
- [ ] Debug in order: pods → endpoints → port-forward → ingress

---

## Quick Reference Card

```bash
# --- SETUP ---
kubectl get nodes
kubectl config current-context

# --- DEPLOY ---
docker build -t express-k8s:latest .
kubectl apply -f k8s/
kubectl get all

# --- INSPECT ---
kubectl get pods -o wide
kubectl describe pod <name>
kubectl logs <name> -f
kubectl get endpoints <svc>
kubectl get events --sort-by=.lastTimestamp

# --- UPDATE / ROLLBACK ---
kubectl rollout restart deployment/<name>
kubectl rollout status  deployment/<name>
kubectl rollout undo    deployment/<name>
kubectl scale deployment <name> --replicas=3

# --- DEBUG ---
kubectl exec -it <pod> -- sh
kubectl port-forward svc/<svc> 8080:80
kubectl run debug --rm -it --image=busybox:1.36 -- sh

# --- SCALE ---
kubectl top pods
kubectl get hpa -w

# --- CLEAN ---
kubectl delete -f k8s/
```

---

*Sheryians Coding School · Kubernetes Local Setup Notes · Docker Desktop · MERN Stack · nginx Ingress*
