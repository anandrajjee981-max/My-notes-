# Kubernetes on Your Local Machine — Complete Guide

**Docker Desktop · MERN Stack · nginx Ingress**


---

## How to read this guide

The guide follows one story from start to finish:

```
What is the problem?  →  The Kubernetes mental model  →  Setup  →  Pod  →  Deployment  →  Service
   →  (Checkpoint 1: run your app)  →  Ingress  →  (Checkpoint 2: open it in the browser)
   →  Config and Secrets  →  Probes, Rolling updates, Storage
   →  Full MERN stack  →  Debugging and Scaling  →  Extras
```

Every chapter uses the same layout, so you always know where to look:

| Section | What it gives you |
|---|---|
| **In one line** | The whole chapter in 10 seconds |
| **Why you need it** | The problem you face without it |
| **How it works** | The concept, with a diagram |
| **Hands-on** | YAML and commands you can run |
| **Common mistakes** | Where people usually get stuck |
| **Remember** | A short summary |

**Running example:** the whole guide uses one app: an **Express backend** with the Docker image `express-k8s:latest`, listening on port **3000**. Later we add MongoDB, Redis and a React frontend.

---

## Table of Contents

**Part 1 — Understanding**
- [1. What is Kubernetes and why use it](#1-what-is-kubernetes-and-why-use-it)
- [2. Architecture: what runs inside](#2-architecture-what-runs-inside)

**Part 2 — Setup**
- [3. Enable Kubernetes in Docker Desktop](#3-enable-kubernetes-in-docker-desktop)
- [4. Learn to write YAML](#4-learn-to-write-yaml)

**Part 3 — Core Building Blocks**
- [5. Pod](#5-pod)
- [6. Deployment (and the ReplicaSet inside it)](#6-deployment-and-the-replicaset-inside-it)
- [7. Labels and Selectors](#7-labels-and-selectors)
- [8. Service](#8-service)
- [Checkpoint 1: Deploy and run your app](#checkpoint-1-deploy-and-run-your-app)

**Part 4 — Traffic From Outside**
- [9. Ingress Controller](#9-ingress-controller)
- [10. Ingress](#10-ingress)
- [Checkpoint 2: Open it in the browser](#checkpoint-2-open-it-in-the-browser)

**Part 5 — Configuration**
- [11. Namespaces](#11-namespaces)
- [12. ConfigMap](#12-configmap)
- [13. Secrets](#13-secrets)

**Part 6 — Reliability**
- [14. Health Probes](#14-health-probes)
- [15. Rolling Updates and Rollbacks](#15-rolling-updates-and-rollbacks)
- [16. Volumes and Persistent Storage](#16-volumes-and-persistent-storage)

**Part 7 — A Real Project**
- [17. The full MERN stack on Kubernetes](#17-the-full-mern-stack-on-kubernetes)

**Part 8 — Operating Your Cluster**
- [18. Everyday kubectl](#18-everyday-kubectl)
- [19. Debugging and Troubleshooting](#19-debugging-and-troubleshooting)
- [20. Autoscaling (HPA)](#20-autoscaling-hpa)

**Part 9 — Extras**
- [21. Other Workload Types](#21-other-workload-types)
- [22. Helm and Kustomize](#22-helm-and-kustomize)
- [23. Useful Tools](#23-useful-tools)
- [24. Cleanup and Reset](#24-cleanup-and-reset)
- [25. From Local to EKS](#25-from-local-to-eks)
- [26. Best Practices Checklist](#26-best-practices-checklist)
- [Quick Reference Card](#quick-reference-card)

---

# PART 1 — UNDERSTANDING

## 1. What is Kubernetes and why use it

> **In one line:** Kubernetes is a **container manager**. You say "I want 3 copies of my app", and it keeps 3 copies running. It restarts crashed containers, adds more when load grows, and updates to a new version with no downtime.

### Why you need it

Say you built a Docker container for your Express app. On your laptop, `docker run` is enough. Now think about production:

- The container crashes at 3 AM. **Who restarts it?**
- Traffic grows 10 times. **Who starts more containers?**
- You release a new version. **How do you avoid downtime?**
- You have 5 services (frontend, backend, auth, mongo, redis). **How do they find each other?** IP addresses change on every restart.

Doing all this by hand is painful. Kubernetes (short name: **K8s**) does it automatically.

### Without vs with Kubernetes

| Situation | Without Kubernetes | With Kubernetes |
|---|---|---|
| Container crashes | App is down until someone restarts it | Restarted automatically |
| Traffic spike | App slows down or crashes | HPA adds more containers |
| Many services | Manual ports and complex networking | Internal DNS: services talk by name |
| Deploy a new version | Downtime during the update | Rolling update with zero downtime |
| Many copies | Manage each container by hand | Write `replicas: 3` and K8s does the rest |

### Key terms you will meet

| Term | Simple meaning |
|---|---|
| **Pod** | The smallest unit. A wrapper around one or more containers. |
| **Deployment** | The manager of your pods. Keeps them running, handles updates and rollbacks. |
| **ReplicaSet** | Makes sure N identical pods are always alive. Lives inside a Deployment. |
| **Service** | A stable address for reaching your pods. |
| **Ingress** | HTTP routing rules: which domain or path goes to which service. |
| **Ingress Controller** | The nginx pod that reads Ingress rules and routes real traffic. |
| **Namespace** | A virtual cluster inside a cluster. Used to separate environments. |
| **ConfigMap** | Stores non-secret config (environment variables). |
| **Secret** | Stores sensitive config (passwords, tokens). |
| **Volume / PVC** | Storage that survives when a pod dies (needed for databases). |
| **HPA** | Adds or removes pods automatically based on load. |
| **Node** | A machine (VM or physical) that runs pods. Docker Desktop has 1 node. |
| **Cluster** | The control plane plus all nodes. |

### The big idea: the declarative model

You never tell Kubernetes *"start a container"*. You tell it the **desired state**: *"I want 3 copies running."* Kubernetes keeps checking whether the **actual state** matches the desired state. If not, it fixes the difference. This loop is called **reconciliation**.

```
You write YAML (desired state: "3 pods")
        ↓
The API Server saves it in etcd
        ↓
Controllers keep checking: actual ≠ desired ?
        ↓
If different → create / delete / restart pods
```

> **Analogy:** A restaurant manager. You say "there must always be 3 chefs in the kitchen." If one chef goes on leave, the manager calls another one at once. You do not have to say it again.

### Remember
- Kubernetes is an automatic manager for containers.
- You write **what you want** (YAML). Kubernetes works out **how to do it**.
- The four big benefits: self-healing, scaling, rolling updates, and service discovery.

---

## 2. Architecture: what runs inside

> **In one line:** A cluster has a **Control Plane** (the brain, which decides) and **Worker Nodes** (the muscle, where pods run). On Docker Desktop both live on your own machine, in one node called `docker-desktop`.

### The full picture

```
┌──────────────────── KUBERNETES CLUSTER (Docker Desktop) ────────────────────┐
│                                                                              │
│  ┌─── MASTER / Control Plane ───┐     ┌─── WORKER NODE ───────────────────┐ │
│  │  API Server   Scheduler      │     │  kubelet   kube-proxy             │ │
│  │  Controller Manager   etcd   │     │  Container Runtime                │ │
│  │                              │     │                                   │ │
│  │  System pods:                │     │  Deployment                       │ │
│  │   coredns, kube-proxy,       │     │   └─ ReplicaSet                   │ │
│  │   ingress-nginx,             │     │       ├─ Pod 1 (app=express)      │ │
│  │   metrics-server             │     │       ├─ Pod 2 (app=express)      │ │
│  └──────────────────────────────┘     │       └─ Pod 3 (added by HPA)     │ │
│                                       └───────────────────────────────────┘ │
│  Service · Ingress · HPA · ConfigMap / Secret                                │
└──────────────────────────────────────────────────────────────────────────────┘
```

### The 4 parts of the Control Plane

| Component | Job | Analogy |
|---|---|---|
| **API Server** | The entry point for every `kubectl` command | Reception desk |
| **Scheduler** | Decides which node a new pod runs on | The person who assigns seats |
| **Controller Manager** | Runs control loops (ReplicaSet, Deployment...) | Floor manager who keeps checking for gaps |
| **etcd** | Key-value database holding the whole cluster state | The record book |

### The 3 parts of a Worker Node

| Component | Job |
|---|---|
| **kubelet** | The agent on the node. Takes work from the API server and keeps pods running |
| **kube-proxy** | Manages the network rules that make Services work |
| **Container runtime** | Actually runs the containers (containerd / Docker) |

### What happens when you run `kubectl apply -f deployment.yaml`

1. `kubectl` sends the YAML to the **API Server**.
2. The API Server validates it and saves it in **etcd**.
3. The **Deployment controller** sees a new Deployment and creates a **ReplicaSet**.
4. The **ReplicaSet controller** sees "need 2 pods, have 0" and creates **Pod objects**.
5. The **Scheduler** assigns each pod to a node.
6. The **kubelet** on that node pulls the image and starts the container.
7. The pod becomes `Running`, then `Ready`, and the Service starts sending traffic to it.

### The journey of a request

```
Browser
   │  http://express.local or http://localhost
   ▼
Ingress Controller (nginx pod)       ← real traffic arrives here
   │  reads the Ingress rules
   ▼
Ingress rules (your ingress.yaml)    ← "where should this path go?"
   ▼
Service (ClusterIP)                  ← finds pods by label, balances load
   ▼
Pod (Express container)              ← handles the request and replies
```

Keep this journey in mind. In Parts 3 and 4 we build it one step at a time.

### Remember
- The Control Plane **decides**. The Worker Node **runs**.
- `kubectl` always talks to the API Server, never directly to a pod.
- Docker Desktop gives a single-node cluster. It is perfect for learning, but not the same as a multi-node production cluster.

---

# PART 2 — SETUP

## 3. Enable Kubernetes in Docker Desktop

> **In one line:** One checkbox in Settings, a 2–3 minute wait, and you have a full Kubernetes cluster on your laptop.

### Step 1 — Enable it

Open Docker Desktop → **Settings (gear icon)** → **Kubernetes** → tick **Enable Kubernetes** → click **Apply & Restart**.

Wait 2–3 minutes. When the status bar at the bottom shows a **green Kubernetes icon**, it is ready.

> 💡 If your RAM is low, give Docker Desktop at least **4 GB** (Settings → Resources). Below that, Kubernetes with a few pods and metrics-server will feel slow.

### Step 2 — Check that kubectl works

```bash
kubectl version
kubectl get nodes

# Expected output:
NAME             STATUS   ROLES
docker-desktop   Ready    control-plane
```

### Step 3 — Check the current context

```bash
kubectl config current-context
# Should print: docker-desktop

kubectl config get-contexts
# Lists every cluster kubectl knows about
```

The file `~/.kube/config` stores the connection details of all your clusters. When you create an EKS cluster later, a new entry is added and the context switches to EKS automatically. To come back:

```bash
kubectl config use-context docker-desktop
```

> ⚠️ **An expensive mistake:** always run `kubectl config current-context` before `apply` or `delete`. Running a command against a production cluster instead of your local one can cause real damage.

### How local images work

Docker Desktop's Kubernetes uses the **same Docker image store** as your `docker` CLI. So after `docker build -t express-k8s:latest .`, Kubernetes can see the image right away. You do not need to push it to a registry. This is why `imagePullPolicy: IfNotPresent` works locally.

> ⚠️ If the tag is `:latest` and you do not set `imagePullPolicy`, Kubernetes assumes `Always` and tries to pull from Docker Hub. That gives **`ImagePullBackOff`**. For local images, always set `IfNotPresent` (or use a versioned tag like `v1`).

### Remember
- `kubectl get nodes` should show `docker-desktop  Ready`.
- Make it a habit to check the context.
- Use `imagePullPolicy: IfNotPresent` for local images.

---

## 4. Learn to write YAML

> **In one line:** Every Kubernetes resource is a YAML file with 4 required fields: `apiVersion`, `kind`, `metadata` and `spec`.

### Why learn YAML first
In every later chapter you will write YAML. Learn the structure once, and only the `spec` part changes from resource to resource.

### The four required fields

```yaml
apiVersion: ...    # which API group and version
kind: ...          # what type of resource
metadata:          # name, labels, namespace, annotations
  name: ...
spec:              # the desired state (shape depends on kind)
  ...
```

### Choosing the right apiVersion

Different resources live in different **API groups**:

| Resource | apiVersion | Why |
|---|---|---|
| Pod, Service, ConfigMap, Secret, Namespace, PVC | `v1` | Core group: the oldest, most basic resources. No group prefix. |
| Deployment, ReplicaSet, StatefulSet, DaemonSet | `apps/v1` | Apps group: higher-level resources that manage pods. |
| Ingress, NetworkPolicy | `networking.k8s.io/v1` | Networking group. |
| HorizontalPodAutoscaler | `autoscaling/v2` | Autoscaling group. |
| Job, CronJob | `batch/v1` | Batch group: one-time or scheduled work. |

> 💡 If you forget, run `kubectl explain deployment` or `kubectl api-resources`.

### Several resources in one file

Separate them with `---`:

```yaml
apiVersion: apps/v1
kind: Deployment
# ... deployment config

---                    # separator: a new resource starts here

apiVersion: v1
kind: Service
# ... service config
```

> 💡 As the project grows, keep files **separate** (one resource per file) and apply the whole folder at once: `kubectl apply -f k8s/`.

### Project structure (we will build this)

```
your-project/
├── Backend/
│   ├── server.js
│   ├── package.json
│   └── dockerfile
└── k8s/                  # all Kubernetes YAML files go here
    ├── deployment.yaml   # how the app runs
    ├── service.yaml      # how to reach the app
    └── ingress.yaml      # how outside traffic comes in
```

### Generate YAML instead of typing it

```bash
# A dry run prints the YAML and creates nothing
kubectl create deployment express --image=express-k8s:latest --dry-run=client -o yaml > deployment.yaml
kubectl create service clusterip express-service --tcp=80:3000 --dry-run=client -o yaml > service.yaml

# See every available field of a resource
kubectl explain deployment.spec.template.spec.containers
```

### Imperative vs declarative

| Style | Example | Use it when |
|---|---|---|
| **Imperative** | `kubectl create deployment ...`, `kubectl scale ...` | Quick experiments and learning |
| **Declarative** | `kubectl apply -f deployment.yaml` | Real projects: YAML in git is the source of truth |

### `apply` vs `create` vs `replace`

| Command | What it does |
|---|---|
| `kubectl create -f` | Creates the resource. **Fails if it already exists** |
| `kubectl apply -f` | Creates or updates. Safe to run again. **Use this one.** |
| `kubectl replace -f` | Replaces the resource completely. Fails if it does not exist |

### Common mistakes
- Use **spaces only** in YAML (2 spaces), never tabs.
- Quote strings that look like numbers or booleans: `"128Mi"`, `"true"`.
- Check before applying: `kubectl apply -f file.yaml --dry-run=server`.

### Remember
- Every YAML has `apiVersion + kind + metadata + spec`.
- `apply` is the default command.
- When unsure, use `kubectl explain`.

---

# PART 3 — CORE BUILDING BLOCKS

## 5. Pod

> **In one line:** A **Pod** wraps one or more containers. It is the smallest unit in Kubernetes. You rarely create pods directly; a Deployment creates them for you.

### Why you need it
Kubernetes does not run a container directly. It runs it inside a pod. All containers in a pod share **one IP address and one network (`localhost`)**, and they can share storage. Most pods have just one container: your app.

### What a pod contains

| Item | Meaning |
|---|---|
| **Container(s)** | One or more Docker containers. They share `localhost` inside the pod. |
| **IP address** | Every pod has its own IP. **It changes on every restart**, which is why you use a Service and not the pod IP. |
| **Labels** | Tags like `app: express`. Services and Deployments use them to find the pod. |
| **Resources** | CPU and memory limits and requests. |

### Pod lifecycle

```
Pending            → the pod is scheduled, the image is being pulled
   ↓
Running            → the container started
   ↓
Ready              → the readiness probe passed, traffic can arrive
   ↓
CrashLoopBackOff   → the container keeps crashing, Kubernetes keeps retrying
   ↓
ImagePullBackOff   → the Docker image cannot be pulled
```

Other states: `Succeeded` (job finished), `Failed`, `Terminating` (being deleted), `Evicted` (the node ran out of resources), `OOMKilled` (the container used more memory than its limit).

### Hands-on: a test pod

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

```bash
kubectl apply -f pod.yaml
kubectl get pods
kubectl delete pod test-pod
```

After you delete it, the pod **does not come back**, because nothing was managing it. This is the main weakness of a standalone pod, and a Deployment fixes it (next chapter).

> 💡 In production, never create pods directly. If a standalone pod crashes, nothing restarts it.

### Multi-container pods (sidecar pattern)

Containers in one pod share the network and volumes. A common pattern is **main app + sidecar** (a log shipper, a proxy, a config reloader).

```
Pod
├── container: express      (main app, port 3000)
└── container: log-shipper  (reads logs from a shared volume and sends them out)
```

### Inspecting a pod

```bash
kubectl get pods                       # list pods
kubectl get pods -o wide               # include pod IP and node
kubectl describe pod <pod-name>        # full details and events
kubectl logs <pod-name>                # app console output
kubectl exec -it <pod-name> -- sh      # open a shell inside the pod
```

### Remember
- A pod wraps containers and has its own IP, which changes on restart.
- A standalone pod is not restarted if it dies, so **use a Deployment**.
- For debugging use `describe`, `logs` and `exec`.

---

## 6. Deployment (and the ReplicaSet inside it)

> **In one line:** A **Deployment** manages your pods. You say which image, how many copies and how many resources. It keeps exactly that running, and also handles updates and rollbacks.

### Why you need it
In the last chapter, a lone pod died and stayed dead. A Deployment is the manager that says: "2 pods must always be running." If one crashes, it creates a new one.

### What a Deployment is responsible for

| Job | Meaning |
|---|---|
| **Replica management** | Always keeps the exact number of pods running. If one crashes, a new one starts at once. |
| **Rolling update** | When you push a new image, pods are replaced one by one. Old pods keep serving until the new ones are ready, so there is no downtime. |
| **Rollback** | If the new version has a bug, one command returns to the last working version. |
| **Scaling** | Change the replica count by hand, or let HPA do it from CPU use. |

### A full deployment.yaml, explained line by line

```yaml
apiVersion: apps/v1          # Deployment belongs to the 'apps' group
kind: Deployment
metadata:
  name: express-deployment
spec:
  replicas: 2              # always keep 2 pods running
  selector:
    matchLabels:
      app: express          # (1) manage the pods that have this label
  template:              # the blueprint for every pod
    metadata:
      labels:
        app: express        # (2) put this label on every pod created
        owner: ankur       # extra label; the Deployment ignores it
    spec:
      containers:
        - name: express
          image: express-k8s:latest
          imagePullPolicy: IfNotPresent  # use the local image if it exists
          ports:
            - containerPort: 3000         # the port your app listens on
          resources:
            limits:
              memory: "128Mi"            # the most memory the pod can use
              cpu: "500m"               # the most CPU it can use (500m = half a core)
            requests:
              memory: "64Mi"             # memory reserved for this pod
              cpu: "250m"               # CPU reserved (required for HPA)
```

### Key fields explained

| Field | Value | What it does |
|---|---|---|
| `apiVersion` | `apps/v1` | A Deployment is in the `apps` group, not the core group (`v1`), which is for Pods and Services. |
| `replicas` | `2` | Keep 2 pods running. If one crashes, a new one starts at once. |
| `selector.matchLabels` | `app: express` | The Deployment manages pods that have **all** these labels. It must match `template.metadata.labels`. |
| `template.metadata.labels` | `app: express` | The labels put on every pod this Deployment creates. The Service uses them to find pods. |
| `imagePullPolicy` | `IfNotPresent` | Use the local image if available. On EKS use `Always` so it pulls from ECR. |
| `resources.requests` | `cpu: 250m` | CPU and memory reserved for the pod. **HPA needs requests**, because it uses them to calculate utilization %. |
| `resources.limits` | `cpu: 500m` | The most the pod may use. Stops one pod from starving the others. |

> ⚠️ `spec.selector` is **immutable** (you cannot change it after creation). To change it, delete the Deployment and create it again.

### CPU and memory units

| CPU | Meaning | Memory | Meaning |
|---|---|---|---|
| `1000m` | 1 full core | `64Mi` | 64 mebibytes (about 67 MB) |
| `500m` | Half a core | `128Mi` | 128 mebibytes |
| `250m` | A quarter core | `1Gi` | 1 gibibyte |

### Requests vs limits: what happens when a limit is exceeded

| Resource | Going over the **limit** | Result |
|---|---|---|
| **CPU** | Throttled (slowed down) | The pod keeps running, only slower |
| **Memory** | The container is killed | Status `OOMKilled`, then it restarts |

The Scheduler uses **requests** to decide where a pod fits. **Limits** are enforced at runtime.

### The ReplicaSet inside: how a Deployment really works

A Deployment does not create pods itself. It creates a **ReplicaSet**, and the ReplicaSet creates the pods.

```
Deployment  →  manages a ReplicaSet
    ↓
ReplicaSet  →  "replicas: 2" means 2 pods are always alive
    ↓
Pod 1 (running)   Pod 2 (running)
    ↓  Pod 1 crashes
The ReplicaSet sees: 1 running, 2 wanted
    ↓
The ReplicaSet creates Pod 3 automatically
```

**Why a Deployment and not just a ReplicaSet?**

| If you use | You get |
|---|---|
| **Deployment** | ReplicaSet management + rolling updates + rollback history. Always use this in practice. |
| **ReplicaSet directly** | Only pod count. No rolling updates and no rollback. Not recommended. |

When you run `kubectl get replicasets`, you will see an auto-generated name like `express-deployment-7d9f8b6c4`. The hash is a fingerprint of the pod template. When you update the image, a **new ReplicaSet** is created and the old one is scaled down to 0 (kept for rollback):

```bash
kubectl get replicasets

# After a rollout you will see TWO ReplicaSets:
# express-deployment-7d9f8b6c4   2   2   2   (new, active)
# express-deployment-5c8a3d1b2   0   0   0   (old, kept for rollback)
```

A ReplicaSet finds its pods using **label selectors**. If you create a pod by hand with the same labels, the ReplicaSet counts it and may delete one of its own pods. So keep labels unique per Deployment.

For reference, here is a ReplicaSet YAML (in practice, use a Deployment):

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: express-rs
spec:
  replicas: 2
  selector:
    matchLabels:
      app: express
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

### Hands-on: watch self-healing

```bash
kubectl get pods                         # note the pod names
kubectl delete pod <one-pod-name>        # kill one
kubectl get pods -w                      # watch: a new pod appears at once
```

### Useful Deployment commands

```bash
kubectl get deployments
kubectl rollout status deployment/express-deployment
kubectl rollout undo deployment/express-deployment    # rollback
kubectl scale deployment express-deployment --replicas=5
kubectl rollout restart deployment/express-deployment # pull the image again
```

### Remember
- Deployment → ReplicaSet → Pods (three layers).
- `selector.matchLabels` and `template.labels` must match.
- `requests` are needed for HPA. `limits` keep a pod under control.
- Going over the memory limit means OOMKilled. Going over the CPU limit only slows the pod.

---

## 7. Labels and Selectors

> **In one line:** **Labels** are key-value tags on pods. **Selectors** are filters that find pods by those tags. Deployments and Services both use them, so no pod name or IP is ever hardcoded.

### Why you need it
Pods die and get replaced, with new names and new IPs. So "that specific pod" makes no sense. Kubernetes says instead: "whichever pods have the label `app=express` belong to me."

### The three-way connection

```yaml
# deployment.yaml
selector:
  matchLabels:
    app: express      # (1) the Deployment manages pods with this label
template:
  metadata:
    labels:
      app: express    # (2) the pod gets this label

---
# service.yaml
selector:
  app: express        # (3) the Service sends traffic to pods with this label
```

(1) and (3) must match (2). That is the whole trick.

### Selector matching rules

| Pod labels | Selector: `app=express` | Result |
|---|---|---|
| `app: express` | Matches | ✅ Selected |
| `app: express, env: prod` | Matches (extra label ignored) | ✅ Selected |
| `app: backend` | Does not match | ❌ Ignored |
| `app: express, env: staging` | Matches `app=express` | ✅ Selected |

### Separating staging and production with labels

```yaml
# staging-deployment.yaml
selector:
  matchLabels:
    app: express
    env: staging     # manages only staging pods

# production-deployment.yaml
selector:
  matchLabels:
    app: express
    env: production  # manages only production pods
```

Both live in the same cluster but never interfere with each other.

### Working with labels from the CLI

```bash
kubectl get pods --show-labels
kubectl get pods -l app=express                 # filter by label
kubectl get pods -l 'env in (staging,prod)'     # set-based selector
kubectl label pod <pod-name> env=debug          # add a label
kubectl label pod <pod-name> env-               # remove the label "env"
```

### Labels vs annotations

| | Labels | Annotations |
|---|---|---|
| Purpose | Identify and **select** objects | Attach extra **metadata** |
| Used by selectors? | Yes | No |
| Example | `app: express` | `nginx.ingress.kubernetes.io/proxy-body-size: "10m"` |

### Common mistakes
- A typo in a label (`app: expres`) means the Service finds no pods, and `kubectl get endpoints` shows `<none>`.
- Two Deployments with the same labels start counting each other's pods.

### Remember
- Label = tag. Selector = filter.
- A Deployment (to own pods) and a Service (to route traffic) both work through selectors.

---

## 8. Service

> **Key idea:** Pods are temporary and their IPs change. A **Service** gives you a **stable address** that never changes, and it spreads traffic across the pods behind it.

### Why you need it
Say the frontend wants to call the backend. The backend pod had IP `10.1.0.5`, then it crashed and the new pod got `10.1.0.9`. What does the frontend call now? A Service solves this. It gives a fixed name and IP, and behind the scenes it keeps finding live pods using a **label selector**. When a pod is added or removed, the list updates at once.

### A basic service.yaml

```yaml
apiVersion: v1              # Service is a core resource
kind: Service
metadata:
  name: express-service
spec:
  selector:
    app: express            # find ALL pods with the label app=express
  ports:
    - protocol: TCP
      port: 80              # the port the Service listens on inside the cluster
      targetPort: 3000      # the port your app listens on inside the container
  type: ClusterIP          # reachable only inside the cluster
```

### port vs targetPort

| Field | Value | Meaning |
|---|---|---|
| `port` | `80` | The Service's own port. Other pods call `express-service:80`. |
| `targetPort` | `3000` | The real port inside the container. Must match `containerPort` in the Deployment. |

### The three Service types

| Type | Reachable from | Use it for |
|---|---|---|
| **ClusterIP** | Inside the cluster only | Pod-to-pod traffic. The default. Used with Ingress. |
| **NodePort** | Outside, through a port number | Local testing. `localhost:30001`. No Ingress needed. |
| **LoadBalancer** | Outside, through a cloud load balancer | Production on AWS/GCP. Creates a real load balancer and costs money. |

(There is also **ExternalName**. It maps a Service name to an external DNS name, for example a managed database.)

### NodePort (for local testing)

```yaml
type: NodePort
ports:
  - port: 80
    targetPort: 3000
    nodePort: 30001    # open localhost:30001 in the browser
```

The NodePort range is **30000–32767**.

### How a Service balances load

```
A request arrives (selector: app=express)
    ↓
The Service looks at its Endpoints list (live pod IPs)
    ↓
Request 1 → Pod 1 (192.168.1.10)
Request 2 → Pod 2 (192.168.1.11)
Request 3 → Pod 3 (192.168.1.12)
Request 4 → Pod 1 (round robin)
```

```bash
kubectl get services
kubectl get endpoints express-service   # see which pod IPs are in the pool
```

### Service DNS: pods call each other by name

Every Service gets an internal DNS name from CoreDNS:

```
<service-name>.<namespace>.svc.cluster.local
```

| Calling from | How to call |
|---|---|
| Same namespace | `http://express-service` |
| Another namespace | `http://express-service.other-ns` |
| Full name | `http://express-service.default.svc.cluster.local` |

```js
// Inside another pod (for example, frontend calling backend)
const res = await fetch("http://express-service/api/users"); // port 80 → targetPort 3000
```

> ⚠️ Inside a pod, `localhost` means **only that pod**. To call another service, use the **service name**, never `localhost`. (This is the cause of `ECONNREFUSED`.)

### Remember
- A Service is a stable address plus a load balancer. It finds pods by label.
- `targetPort` is the container's port. `port` is the Service's port.
- Inside the cluster, call a service by name. From outside, use NodePort or Ingress.

---

## Checkpoint 1: Deploy and run your app

So far you have learned Pod, Deployment, Labels and Service. Now let's run them.

### Step 1 — Build the Docker image

```bash
cd Backend/
docker build -t express-k8s:latest .
```

A minimal `dockerfile` for the backend:

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

`.dockerignore`:

```
node_modules
.env
.git
k8s
```

> Your app must listen on `0.0.0.0` (or just `app.listen(3000)`), not only `127.0.0.1`. Otherwise the Service cannot reach it.

### Step 2 — Apply the YAML

```bash
kubectl apply -f k8s/    # apply every file in the k8s folder

# Or one by one:
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

### Step 3 — Verify

```bash
kubectl get pods           # STATUS: Running, READY: 1/1
kubectl get services       # the service shows the right ports
kubectl get endpoints express-service   # pod IPs should be listed
```

### Step 4 — Open the app (without Ingress, for now)

Ingress comes in the next part. For now, there are two easy ways:

```bash
# Option A: port-forward (works on any service)
kubectl port-forward svc/express-service 8080:80
# Now open http://localhost:8080
```

Or set `type: NodePort` with `nodePort: 30001` in the Service and open `http://localhost:30001`.

If this works, Part 3 is complete: Pod → Deployment → Service all work.

### Updating after a code change

```bash
# 1. Rebuild the image after changing code
docker build -t express-k8s:latest .

# 2. Restart the deployment so it picks up the new image
kubectl rollout restart deployment/express-deployment
```

### Deleting what you deployed

```bash
kubectl delete -f k8s/                              # delete everything in the folder
kubectl delete deployment express-deployment        # delete one resource
```

---

# PART 4 — TRAFFIC FROM OUTSIDE

## 9. Ingress Controller

> **In one line:** The **Ingress Controller** is an **nginx pod** running in your cluster. It reads Ingress rules and routes real HTTP traffic. Without it, an Ingress YAML is just a config file that does nothing.

### Why the Controller comes before the Ingress
An Ingress holds only **rules** ("send this path to that service"). Someone still has to **apply** those rules. That is the Controller's job. So you install the controller once.

> **Analogy:** The Ingress is a chart on the wall showing who goes to which floor. The Ingress Controller is the receptionist who reads the chart and sends people. Without a receptionist, the chart just hangs there.

### Ingress vs Ingress Controller

| | Ingress Resource | Ingress Controller |
|---|---|---|
| What it is | A YAML file with routing rules | An nginx pod running in your cluster |
| Who creates it | You, with `kubectl apply -f ingress.yaml` | You, once, with `kubectl apply -f nginx-url` |
| Does it route traffic? | No, it only defines rules | Yes, it reads the rules and routes real traffic |
| Namespace | `default` (your app's) | `ingress-nginx` (its own) |

### Install it

```bash
kubectl apply -f \
  https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.1/deploy/static/provider/cloud/deploy.yaml
```

### Verify it

```bash
kubectl get pods -n ingress-nginx

# Expected:
NAME                                        READY   STATUS
ingress-nginx-controller-xxxxxxxxx-xxxxx    1/1     Running

kubectl get svc -n ingress-nginx
# ingress-nginx-controller   LoadBalancer   ...   EXTERNAL-IP: localhost
```

On Docker Desktop, the controller's `LoadBalancer` Service gets `EXTERNAL-IP: localhost`. That is why `http://localhost` (ports 80/443) reaches your app.

### Controller logs (very useful for 404 and 502 problems)

```bash
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller
```

### Remember
- You install the controller once, in its own namespace (`ingress-nginx`).
- A controller with no Ingress YAML routes nothing. An Ingress YAML with no controller does nothing.

---

## 10. Ingress

> **In one line:** An **Ingress** is a YAML file of HTTP routing rules, such as "send `express.local/` to `express-service`" and "send `/auth` to `auth-service`". One entry point, many services.

### Why you need it
Creating a separate NodePort or LoadBalancer for every service is costly and messy. With Ingress you use **one entry point** (port 80/443) and reach many services by host and path.

### A full ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1   # Ingress is in the networking group
kind: Ingress
metadata:
  name: express-ingress
spec:
  ingressClassName: nginx         # which controller handles this rule
  rules:
    - host: express.local          # the domain to match
      http:
        paths:
          - path: /
            pathType: Prefix         # match / and everything after it
            backend:
              service:
                name: express-service
                port:
                  number: 80
```

### Key fields

| Field | Value | What it does |
|---|---|---|
| `ingressClassName` | `nginx` | Tells Kubernetes which controller should pick up this rule. If you have both nginx and traefik, each takes only its own rules. |
| `host` | `express.local` | Only traffic with this Host header matches. Locally you must add it to the hosts file. |
| `pathType: Prefix` | `Prefix` | Matches the path and everything after it. `/api` matches `/api/users`, `/api/data` and so on. |
| `pathType: Exact` | `Exact` | Matches only that exact path. `/api` does **not** match `/api/users`. |

### Easiest for local use: leave out the host

```yaml
rules:
  - http:                     # no host: field, so it matches all traffic
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: express-service
              port:
                number: 80
```

Open `http://localhost` directly. You do not need to edit the hosts file.

### With a host: add `express.local` to the hosts file

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

Now `http://express.local` in your browser reaches the nginx Ingress Controller.

### Path-based routing: many services on one domain

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

nginx always picks the **longest matching path**, so `/auth/...` goes to auth-service and everything else goes to main-service.

### Useful nginx annotations

```yaml
metadata:
  name: express-ingress
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"        # allow bigger uploads (default 1m)
    nginx.ingress.kubernetes.io/proxy-read-timeout: "120"     # seconds, for slow APIs
    nginx.ingress.kubernetes.io/rewrite-target: /$2           # strip a path prefix (see below)
    nginx.ingress.kubernetes.io/enable-cors: "true"           # CORS at the ingress level
```

### Stripping a path prefix (rewrite)

To send `localhost/api/users` to the backend as `/users`:

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

nginx ingress supports WebSocket upgrades out of the box. For long-lived connections, raise the timeouts:

```yaml
annotations:
  nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
  nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
```

If you run **more than one backend replica** with Socket.io, you also need sticky sessions (or a Redis adapter):

```yaml
annotations:
  nginx.ingress.kubernetes.io/affinity: "cookie"
```

### HTTPS locally (optional)

```bash
# create a self-signed certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=express.local"

# create the TLS secret
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

### Common mistakes
- Forgetting `ingressClassName` means no controller picks up the rule, and `ADDRESS` stays empty.
- A wrong backend `service.name` or `port.number` gives **503**.
- A host or path that does not match gives **404**.

### Remember
- Ingress = the rules. Controller = the thing that applies them.
- Locally, either leave out the host (`localhost`) or add `express.local` to the hosts file.
- The longest matching path wins.

---

## Checkpoint 2: Open it in the browser

```bash
kubectl apply -f k8s/ingress.yaml
kubectl get ingress           # an ADDRESS should appear (localhost)
```

| Setup | URL |
|---|---|
| No host in the Ingress | `http://localhost` |
| `host: express.local` + hosts file | `http://express.local` |
| NodePort service | `http://localhost:30001` |

The whole journey now works: **Browser → Ingress Controller → Ingress rule → Service → Pod**. If it does not, follow the flowchart in [Chapter 19](#19-debugging-and-troubleshooting).

---

# PART 5 — CONFIGURATION

## 11. Namespaces

> **In one line:** A **Namespace** is a virtual cluster inside your real cluster. It groups resources so names do not clash, and lets you keep `dev`, `staging` and `prod` (or different projects) apart. If you say nothing, everything goes into `default`.

### Namespaces that already exist

| Namespace | What lives there |
|---|---|
| `default` | Your resources, if you say nothing else |
| `kube-system` | Kubernetes system pods (coredns, kube-proxy, metrics-server...) |
| `kube-public` | Publicly readable data (rarely used) |
| `kube-node-lease` | Node heartbeats |
| `ingress-nginx` | Created when you install the ingress controller |

### Creating and using one

```bash
kubectl create namespace dev
kubectl get namespaces

kubectl apply -f k8s/ -n dev          # apply into the dev namespace
kubectl get pods -n dev               # list pods in dev
kubectl get pods -A                   # pods in ALL namespaces

# Make dev the default namespace for your current context
kubectl config set-context --current --namespace=dev
```

As YAML:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

Or set it directly in a resource:

```yaml
metadata:
  name: express-deployment
  namespace: dev
```

### What is namespaced and what is cluster-wide

| Namespaced (separate per namespace) | Cluster-wide |
|---|---|
| Pod, Deployment, Service, ConfigMap, Secret, Ingress, PVC, HPA | Node, Namespace, PersistentVolume, StorageClass, IngressClass |

### Cross-namespace DNS

```
http://<service>.<namespace>.svc.cluster.local
for example: http://express-service.dev.svc.cluster.local
```

### Limiting a namespace's resources (optional)

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

### Remember
- A namespace is a virtual cluster for separating environments and projects.
- Inside a namespace, Services call each other by name. From another namespace, use `service.namespace`.

---

## 12. ConfigMap

> **In one line:** A **ConfigMap** stores **non-sensitive** config (port, URLs, `NODE_ENV`, feature flags) outside your Docker image. One image can then run in dev, staging and prod with different configs. For passwords, use a Secret (next chapter).

### Why you need it
If you hardcode `NODE_ENV=production` into the image, you need a different image for every environment. With a ConfigMap the image stays the same and only the config changes.

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

kubectl create configmap app-env --from-env-file=.env   # from a .env file (non-secret values only!)
```

### Three ways to use it in a Deployment

**1. One key at a time**

```yaml
env:
  - name: NODE_ENV
    valueFrom:
      configMapKeyRef:
        name: express-config
        key: NODE_ENV
```

**2. All keys at once (`envFrom`)**: the easiest

```yaml
envFrom:
  - configMapRef:
      name: express-config
  - secretRef:
      name: express-secret      # Secrets can be loaded in bulk the same way
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

### What happens when you update a ConfigMap

| How it was used | After you `apply` the changed ConfigMap |
|---|---|
| Environment variables | **NOT updated.** Restart the pods: `kubectl rollout restart deployment/express-deployment` |
| Mounted file | Updated automatically after a short delay (up to about 1 minute) |

### Inspect

```bash
kubectl get configmaps
kubectl describe configmap express-config
kubectl get configmap express-config -o yaml
```

### Remember
- ConfigMap = non-secret config. Secret = sensitive config.
- `envFrom` needs the least typing.
- After changing env-based config, restart the pods.

---

## 13. Secrets

> **In one line:** A **Secret** stores sensitive data such as database passwords, API keys and JWT secrets inside the cluster (base64 encoded), and gives them to pods as environment variables or files.

### Why a separate Secret?
Kubernetes treats Secrets with extra care (they are not printed in logs, and RBAC can restrict them). On local Docker Desktop, plain Secrets are fine. In production, use an external vault.

> ⚠️ **base64 is NOT encryption.** It is only encoding. Anyone with `kubectl get secret` access can decode it. Never commit a Secret YAML to git. Put it in `.gitignore`, or in production use *sealed-secrets* or *external-secrets*.

### Step 1 — Create base64 values

```bash
# Encode any string as base64
echo -n 'mypassword' | base64
# → bXlwYXNzd29yZA==

echo -n 'mongodb://localhost:27017/mydb' | base64
# → bW9uZ29kYjovL2xvY2FsaG9zdDoyNzAxNy9teWRi

# Decode to verify
echo 'bXlwYXNzd29yZA==' | base64 --decode
```

> Always use `echo -n` (no newline). Without `-n`, a hidden `\n` gets encoded and your password silently breaks.

### Step 2 — Write secret.yaml

```yaml
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: express-secret
type: Opaque                # Opaque = a generic key-value secret
data:
  MONGO_URI: bW9uZ29kYjovL2xvY2FsaG9zdDoyNzAxNy9teWRi  # base64 encoded
  JWT_SECRET: bXlzdXBlcnNlY3JldGtleQ==
  DB_PASSWORD: bXlwYXNzd29yZA==
```

> 💡 **Tip:** use **`stringData`** instead of `data` to write plain text directly (Kubernetes encodes it for you). It is easier locally, and the result in the cluster is the same.

```yaml
# k8s/secret.yaml: using stringData (plain text, easier locally)
apiVersion: v1
kind: Secret
metadata:
  name: express-secret
type: Opaque
stringData:                  # plain text; k8s base64-encodes it automatically
  MONGO_URI: "mongodb://localhost:27017/mydb"
  JWT_SECRET: "mysupersecretkey"
  DB_PASSWORD: "mypassword"
```

### Create from the CLI (no YAML file, nothing to commit by mistake)

```bash
kubectl create secret generic express-secret \
  --from-literal=JWT_SECRET=mysupersecretkey \
  --from-literal=DB_PASSWORD=mypassword

kubectl create secret generic app-env --from-env-file=.env.secret
```

### Step 3 — Apply it

```bash
kubectl apply -f k8s/secret.yaml

# Verify
kubectl get secrets
kubectl describe secret express-secret   # shows keys but NOT values

# Decode one value to check it
kubectl get secret express-secret -o jsonpath='{.data.MONGO_URI}' | base64 --decode
```

### Step 4 — Inject the Secret into your Deployment as env vars

```yaml
spec:
  containers:
    - name: express
      image: express-k8s:latest
      env:
        - name: MONGO_URI          # the env var name inside the container
          valueFrom:
            secretKeyRef:
              name: express-secret # the secret name (metadata.name)
              key: MONGO_URI         # the key inside the secret
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

### Shortcut: load every key at once

```yaml
envFrom:
  - secretRef:
      name: express-secret     # every key becomes an env var with the same name
```

### Mount a Secret as files (for certificates)

```yaml
volumeMounts:
  - name: secret-vol
    mountPath: /etc/secrets
    readOnly: true
volumes:
  - name: secret-vol
    secret:
      secretName: express-secret   # each key becomes a file in /etc/secrets
```

### Using it in Node.js / Express

```js
// Kubernetes injects secrets as environment variables
const mongoUri   = process.env.MONGO_URI;
const jwtSecret  = process.env.JWT_SECRET;
const dbPassword = process.env.DB_PASSWORD;

// The same code works locally with .env and in Kubernetes with Secrets
```

### Secret vs ConfigMap: which one to use

| Use case | ConfigMap | Secret |
|---|---|---|
| Database password | ❌ Never | ✅ Yes |
| API base URL | ✅ Yes | Not needed |
| JWT secret key | ❌ Never | ✅ Yes |
| `NODE_ENV = production` | ✅ Yes | Not needed |
| MongoDB connection string | ❌ if it has a password | ✅ Yes |
| Port number | ✅ Yes | Not needed |

### Secret types

| Type | Used for |
|---|---|
| `Opaque` | Generic key-value (the default) |
| `kubernetes.io/tls` | TLS certificate + key (for Ingress HTTPS) |
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

### Project structure (with the secret)

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
    └── secret.yaml       # ← add to .gitignore!
```

> ⚠️ **Always add `secret.yaml` to `.gitignore`.** Committing secrets to git, even as base64, is a serious security risk. Add `k8s/secret.yaml` to your `.gitignore`. Share secrets with teammates over a secure channel (1Password, Bitwarden, a private DM), never through git.

### Apply order matters

```bash
# 1. The Secret (and ConfigMap) must exist before the pods that use them start
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/configmap.yaml

# 2. Then apply the deployment (the pods can now find the secret)
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/ingress.yaml

# Verify the env vars reached the pod
kubectl exec -it <pod-name> -- sh
# Inside the pod:
echo $MONGO_URI
echo $JWT_SECRET
```

### Common mistakes
- A pod that references a missing Secret, ConfigMap or key gets stuck in **`CreateContainerConfigError`**. Create the secret and the pod recovers by itself.
- When you change a Secret, env vars are **not refreshed**. Run `kubectl rollout restart deployment/express-deployment`.

### Remember
- A Secret holds sensitive config. base64 is only encoding.
- Apply the Secret and ConfigMap **first**, the Deployment after.
- Never commit `secret.yaml` to git.

---

# PART 6 — RELIABILITY

## 14. Health Probes

> **In one line:** **Probes** are small health checks (HTTP, TCP or a command). They tell Kubernetes whether a container is really healthy, and when to restart it or send it traffic.

### Why you need it
A running container process does not mean a healthy app. The app might be hung, or it might not have connected to the database yet. Kubernetes cannot see this from outside, so you give it probes.

### The three probes

| Probe | Question it answers | If it fails |
|---|---|---|
| **startupProbe** | Has the app finished starting? | Keeps waiting; kills the container after too many failures. The other two probes are off until it passes. |
| **readinessProbe** | Can this pod take traffic right now? | The pod is **removed from the Service endpoints** (**no** restart) |
| **livenessProbe** | Is the app stuck or dead? | The container is **restarted** |

### Health endpoints in Express

```js
app.get("/health", (req, res) => res.status(200).json({ status: "ok" }));
```

In readiness, check real dependencies:

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
| `initialDelaySeconds` | How long to wait after the container starts before the first check |
| `periodSeconds` | How often to check |
| `timeoutSeconds` | How long to wait for a response (default 1s) |
| `failureThreshold` | How many failures in a row before Kubernetes acts |
| `successThreshold` | How many successes in a row to count as healthy again |

### Other probe types

```yaml
# TCP: just check that the port accepts connections
livenessProbe:
  tcpSocket:
    port: 3000

# Command: exit code 0 = healthy
livenessProbe:
  exec:
    command: ["cat", "/tmp/healthy"]
```

> ⚠️ **Do not check external dependencies in the liveness probe.** If MongoDB goes down and liveness fails, Kubernetes restarts **all** your pods for no reason. Put dependency checks in **readiness**, and keep liveness simple (is the process responding?).

> 💡 Readiness is what makes **rolling updates zero-downtime**. A new pod gets traffic only after its readiness probe passes.

### Remember
- Readiness = send traffic or not. Liveness = restart or not. Startup = give slow apps time.
- Keep liveness simple. Check the database in readiness.

---

## 15. Rolling Updates and Rollbacks

> **In one line:** When you change the pod template (image, env, resources), the Deployment creates a **new ReplicaSet** and moves pods from old to new, little by little. The update strategy controls how fast.

### Strategy settings

```yaml
spec:
  replicas: 4
  revisionHistoryLimit: 5          # how many old ReplicaSets to keep for rollback
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1                  # allow 1 extra pod above the desired count during an update
      maxUnavailable: 0            # never go below the desired count → zero downtime
```

| Field | Meaning |
|---|---|
| `maxSurge` | How many pods **above** `replicas` are allowed during an update (a number or %) |
| `maxUnavailable` | How many pods may be **down** during an update (a number or %) |
| `type: Recreate` | Kill all old pods first, then start new ones (causes downtime; used when two versions cannot run together) |

### What a rolling update looks like (replicas: 3, maxSurge 1, maxUnavailable 0)

```
Start:  [v1][v1][v1]
Step 1: [v1][v1][v1] + [v2 starting]
Step 2: [v1][v1][v2 ready] → kill one v1
Step 3: [v1][v2][v2] + [v2 starting] → ...
Done:   [v2][v2][v2]
```

### Trigger, watch and roll back

```bash
# Update the image (or edit the YAML and apply)
kubectl set image deployment/express-deployment express=express-k8s:v2

# Watch progress
kubectl rollout status deployment/express-deployment

# History
kubectl rollout history deployment/express-deployment
kubectl rollout history deployment/express-deployment --revision=2

# Rollback
kubectl rollout undo deployment/express-deployment                  # to the previous one
kubectl rollout undo deployment/express-deployment --to-revision=1  # to a specific one

# Pause / resume (combine several changes into one rollout)
kubectl rollout pause deployment/express-deployment
kubectl rollout resume deployment/express-deployment
```

### Use version tags instead of `:latest`

```bash
docker build -t express-k8s:v1 .
docker build -t express-k8s:v2 .
```

With `:latest`, the YAML **does not change**, so `kubectl apply` does nothing and you must run `rollout restart`. With unique tags, changing the tag in YAML triggers a rollout by itself, and rollback becomes meaningful.

### Graceful shutdown (so running requests do not break)

Kubernetes sends `SIGTERM`, waits `terminationGracePeriodSeconds` (default 30s), then sends `SIGKILL`.

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

### Remember
- Zero downtime = `maxUnavailable: 0` + a readiness probe.
- `rollout undo` goes back with one command.
- Use versioned tags, not `:latest`.

---

## 16. Volumes and Persistent Storage

> **In one line:** A container's filesystem disappears when the pod is deleted or restarted. For data you must keep (MongoDB, uploads), attach a **Volume**.

### Volume types

| Type | Lasts as long as | Use it for |
|---|---|---|
| `emptyDir` | The **pod** | Scratch space, sharing files between containers in a pod |
| `hostPath` | The **node's disk** | Local development only (mounting a folder from your machine). Avoid in production. |
| `configMap` / `secret` | Config as files | See the ConfigMap and Secret chapters |
| **PersistentVolumeClaim** | **Beyond the pod** | Databases and uploads: real persistence |

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

### How PV, PVC and StorageClass fit together

```
Pod  →  PersistentVolumeClaim (PVC)  →  PersistentVolume (PV)  →  the real disk
        "I need 1Gi"                    "here is 1Gi"
                          ↑
          A StorageClass creates the PV on demand (dynamic provisioning)
```

| Object | Who creates it | Meaning |
|---|---|---|
| **PersistentVolume (PV)** | An admin or the StorageClass (automatically) | A piece of storage in the cluster |
| **PersistentVolumeClaim (PVC)** | You | A request for storage: size + access mode |
| **StorageClass** | The cluster | Defines **how** storage is created. Docker Desktop has a default (`hostpath`) |

### PVC example

```yaml
# k8s/mongo-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pvc
spec:
  accessModes:
    - ReadWriteOnce          # one node can mount it read-write
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl get storageclass
kubectl get pvc
kubectl get pv
```

### Using a PVC in a pod

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
| `ReadWriteOnce` (RWO) | One node can mount it read-write |
| `ReadOnlyMany` (ROX) | Many nodes, read-only |
| `ReadWriteMany` (RWX) | Many nodes, read-write (needs special storage like NFS/EFS) |

> ⚠️ Deleting a PVC may delete the data too (it depends on the PV's reclaim policy; for dynamic storage the default is usually `Delete`).

> 💡 For production databases, a **managed service** (MongoDB Atlas, RDS, ElastiCache) is better. Running a database in Kubernetes is fine for local learning, but hard to operate in production.

### Remember
- A container's data goes away with the pod. To keep it, use a PVC.
- Pod → PVC → PV → disk. The StorageClass creates the PV for you.

---

# PART 7 — A REAL PROJECT

## 17. The full MERN stack on Kubernetes

> **In one line:** A real app has many parts. Each part gets its own **Deployment + Service**, and they find each other by **service name**. Only the **Ingress** is open to the outside.

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
  replicas: 1                      # a single mongod; do NOT scale this
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
  name: mongo                      # ← this name becomes the hostname
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
MONGO_URI = mongodb://mongo:27017/mydb       # "mongo" = the Service name
REDIS_URL = redis://redis:6379               # "redis" = the Service name
```

Put these in the ConfigMap / Secret, and **not** `localhost`.

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
# Frontend/dockerfile: multi-stage build
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build            # outputs dist/ (Vite) or build/ (CRA)

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
# SPA fallback so React Router routes still work after a refresh
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

> ⚠️ **A very common mistake:** React runs in the **user's browser**, not inside the cluster. It cannot call `http://backend` (that name only exists inside the cluster). Have the frontend call a relative path like `/api/...`, and let the Ingress route `/api` to the backend.

### Ingress: one entry point for both

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

nginx picks the **longest matching path**, so `/api/...` goes to the backend and everything else goes to the frontend.

### Deploy everything in the right order

```bash
docker build -t express-k8s:latest ./Backend
docker build -t frontend-k8s:latest ./Frontend

kubectl apply -f k8s/namespace.yaml        # if you use a namespace
kubectl apply -f k8s/configmap.yaml -f k8s/secret.yaml
kubectl apply -f k8s/mongo.yaml -f k8s/redis.yaml
kubectl apply -f k8s/backend.yaml -f k8s/frontend.yaml
kubectl apply -f k8s/ingress.yaml

kubectl get all
```

> 💡 Pods start in any order. If the backend starts before Mongo is ready, it may crash and restart a few times. That is normal. Make your app **retry the database connection**, and add a readiness probe.

### Remember
- Every part is a Deployment + Service. The Service name is the hostname (`mongo`, `redis`, `backend`).
- The frontend runs in the browser, so use a relative `/api` path plus Ingress.
- Keep Mongo at `replicas: 1` and give it a PVC.

---

# PART 8 — OPERATING YOUR CLUSTER

## 18. Everyday kubectl

> **In one line:** `kubectl <verb> <resource> <name> <flags>`. Remember this pattern, and most commands are just variations of it.

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
| `describe` | Show details and events |
| `apply` / `create` | Create or update from a file |
| `delete` | Remove a resource |
| `logs` | Show container output |
| `exec` | Run a command in a container |
| `edit` | Open a live resource in your editor |
| `scale` | Change the replica count |
| `rollout` | Manage Deployment rollouts |
| `port-forward` | Connect a local port to a pod or service |
| `top` | Show CPU and memory use (needs metrics-server) |
| `explain` | Show documentation for any field |

### Output formats

```bash
kubectl get pods -o wide                 # extra columns (IP, node)
kubectl get pods -o yaml                 # full YAML
kubectl get pod <name> -o json
kubectl get pods -o name                 # names only
kubectl get pods -w                      # watch live
kubectl get pods --sort-by=.metadata.creationTimestamp
kubectl get svc express-service -o jsonpath='{.spec.clusterIP}'
```

### Port-forward: reach anything without Ingress

```bash
kubectl port-forward pod/<pod-name> 8080:3000          # localhost:8080 → pod:3000
kubectl port-forward svc/express-service 8080:80       # localhost:8080 → service:80
kubectl port-forward svc/mongo 27017:27017             # connect MongoDB Compass to localhost:27017
```

Great for debugging a service directly, or opening the in-cluster Mongo with Compass. Press `Ctrl+C` to stop.

### Copy files in and out of a pod

```bash
kubectl cp <pod-name>:/app/logs/app.log ./app.log
kubectl cp ./config.json <pod-name>:/app/config.json
```

### A temporary debug pod

```bash
# A throwaway pod with curl/nslookup, deleted when you exit
kubectl run debug --rm -it --image=busybox:1.36 -- sh
# inside:
nslookup express-service
wget -qO- http://express-service/health

# If you need curl
kubectl run curl --rm -it --image=curlimages/curl -- sh
```

### Edit a live resource

```bash
kubectl edit deployment express-deployment
```

> Changes made this way are **not** saved in your YAML files. Update the file too, or the next `apply` will overwrite them.

### Shortcuts

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

### Remember
- The pattern is verb + resource + name.
- `-o wide`, `-o yaml` and `-w` are the most useful flags.
- `port-forward` and `kubectl run --rm -it` are your debugging friends.

---

## 19. Debugging and Troubleshooting

> **In one line:** There is one way to debug: go **from the bottom up** — Pods → Endpoints → Service → Ingress. Wherever it breaks is where the problem is.

### Step-by-step flowchart

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
│     ├─ fails              → wrong targetPort / app on the wrong port or on 127.0.0.1
│     └─ works ✅           → the Service is fine, the problem is Ingress → step 4
│
└─ 4. kubectl describe ingress + ingress controller logs
      ├─ 404                → host/path mismatch, wrong ingressClassName
      ├─ 503                → wrong backend service name/port
      └─ no ADDRESS         → controller not installed or not running
```

### Common errors and what to do

| Error | Cause | Debug command |
|---|---|---|
| **ImagePullBackOff / ErrImagePull** | The Docker image cannot be pulled. Usually the local image was not built, or the name/tag is wrong. | `kubectl describe pod <name>` → Events section |
| **CrashLoopBackOff** | The container keeps crashing. The app has a startup error. | `kubectl logs <pod-name> --previous` |
| **CreateContainerConfigError** | The referenced Secret, ConfigMap or key does not exist. | `kubectl describe pod <name>` → Events; create the missing secret |
| **OOMKilled** | The container went over its memory limit. | `kubectl describe pod <name>` → Last State: OOMKilled; raise `limits.memory` or fix the leak |
| **503 Service Unavailable** | The ingress controller is running but found no pods. Label mismatch, or the pods are not ready. | `kubectl get endpoints <service>` → if `<none>`, the labels do not match |
| **502 Bad Gateway** | Ingress reached the pod but the app crashed or refused, or the targetPort is wrong. | Check `targetPort`, pod logs and ingress controller logs |
| **404 Not Found** | nginx is running but has no rule for that host/path. | `kubectl describe ingress <name>` → check the rules |
| **Pending (pod)** | No node has enough CPU or memory for the pod. | `kubectl describe pod <name>` → Events |
| **Pending (PVC)** | No StorageClass, or the volume could not be created. | `kubectl describe pvc <name>` |
| **ECONNREFUSED** | A pod is calling another service using `localhost`. | Use `http://service-name`, not `http://localhost:PORT` |
| **Evicted** | The node ran low on resources. | `kubectl describe pod <name>`; set proper requests and limits |
| **Running but 0/1 READY** | The readiness probe is failing. | `kubectl describe pod <name>` → probe failure events |

### Essential debug commands

```bash
# See what is running
kubectl get pods
kubectl get services
kubectl get ingress
kubectl get all                              # everything at once
kubectl get events --sort-by=.lastTimestamp  # recent cluster events

# Look deeper
kubectl describe pod <pod-name>              # events + full config
kubectl describe ingress <ingress-name>      # routing rules + backend IPs
kubectl describe service <service-name>      # selector + endpoints

# Logs
kubectl logs <pod-name>                      # app output
kubectl logs <pod-name> -f                   # follow live
kubectl logs <pod-name> --previous           # logs from the crashed container
kubectl logs <pod-name> -c <container>       # multi-container pod
kubectl logs -l app=express --tail=50        # logs from all pods with a label
kubectl logs deploy/express-deployment       # logs from a deployment's pod

# Network debugging
kubectl get endpoints <service-name>         # pod IPs in the service pool
kubectl exec -it <pod-name> -- sh            # shell inside the pod
# Inside the pod: test another service
wget -qO- http://some-service/api/data
```

### Quick sanity checklist (labels and ports)

- Deployment `selector.matchLabels` == `template.metadata.labels`
- Service `selector` == Pod labels
- Service `targetPort` == the port the container really listens on
- Ingress backend `service.name` and `port.number` == the Service name and `port`
- The image exists locally: `docker images | grep express-k8s`

### Remember
- Order: pods → endpoints → port-forward → ingress.
- The **Events** section of `describe` tells you the most.
- If a pod crashes, run `logs --previous`.

---

## 20. Autoscaling (HPA)

> **In one line:** The **HPA** watches the CPU use of your pods. When it goes above a threshold, it adds pods. When traffic drops, it removes them. It needs **metrics-server**, which is **not** installed by default on Docker Desktop.

### How HPA thinks

```
Traffic grows → CPU per pod goes above 50%
    ↓
HPA adds a new pod (up to maxReplicas: 5)
    ↓
Requests are spread over more pods
    ↓
CPU per pod drops, because the load is shared
    ↓
Traffic falls → CPU is low
    ↓
HPA removes the extra pods (down to minReplicas: 1)
```

### Step 0 — Install Metrics Server (required)

> ⚠️ **HPA will not work without metrics-server.** `kubectl top pods` will say "Metrics API not available", and HPA will show `<unknown>/50%` instead of a CPU value.

**Option A: Official manifest + kubelet insecure TLS patch (recommended for Docker Desktop)**

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Docker Desktop uses a self-signed cert, so patch metrics-server to skip TLS verification:
kubectl patch deployment metrics-server -n kube-system \
  --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

# Wait until it is ready (~30 seconds)
kubectl rollout status deployment/metrics-server -n kube-system

# Verify: you should see CPU and Memory columns
kubectl top nodes
kubectl top pods
```

**Option B: With Helm (if you have Helm)**

```bash
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm repo update
helm upgrade --install metrics-server metrics-server/metrics-server \
  --namespace kube-system \
  --set args={--kubelet-insecure-tls}
```

> 💡 Metrics-server runs as a pod in the `kube-system` namespace on the master/control-plane node. Check with: `kubectl get pods -n kube-system | grep metrics` → `metrics-server-xxxx   1/1   Running`.

> `--kubelet-insecure-tls` is for **local development only**. Never use it on a real cluster.

### Create an HPA from the command line

```bash
kubectl autoscale deployment express-deployment \
  --min=1 \
  --max=5 \
  --cpu-percent=50
```

### Or as YAML (recommended)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: express-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: express-deployment    # must match the deployment name
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

> ⚠️ **`resources.requests` must be set** in your Deployment. HPA calculates utilization as *current CPU / requested CPU × 100*. With no requests there is no baseline, so HPA will not scale.

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

HPA works out the desired replicas for each metric and picks the **highest**.

### The formula

```
desiredReplicas = ceil( currentReplicas × currentUtilization / targetUtilization )

Example: 2 pods at 100% CPU, target 50%
         → ceil(2 × 100 / 50) = 4 pods
```

### Scale-down is deliberately slow

By default HPA waits about **5 minutes** of low usage before scaling down (the stabilization window), to avoid going up and down all the time. To tune it:

```yaml
spec:
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 60
```

### Generate load to test it

```bash
# Terminal 1: watch the HPA
kubectl get hpa -w

# Terminal 2: send a flood of requests to the service from inside the cluster
kubectl run load-gen --rm -it --image=busybox:1.36 -- \
  /bin/sh -c "while true; do wget -q -O- http://express-service/; done"

# Terminal 3: watch the pods appear
kubectl get pods -w
```

Stop the load generator (`Ctrl+C`) and watch the replicas shrink after the cooldown.

```bash
kubectl get hpa              # current CPU%, desired vs actual replicas
kubectl top pods             # live CPU and memory per pod
kubectl describe hpa express-hpa   # events + why it scaled
```

> If you use HPA, **remove `replicas:` from your Deployment YAML**. Otherwise every `kubectl apply` resets the count.

### Remember
- HPA needs three things: metrics-server, `requests`, and the HPA YAML.
- Scale-up is fast. Scale-down is slow.

---

# PART 9 — EXTRAS

## 21. Other Workload Types

A Deployment is for **stateless** apps. Kubernetes has other controllers for other jobs.

| Kind | Use it for | Key behavior |
|---|---|---|
| **Deployment** | Stateless apps (APIs, frontends) | Interchangeable pods, rolling updates |
| **StatefulSet** | Databases, Kafka: anything that needs a stable identity or storage | Pods are named `db-0`, `db-1`...; each gets its own PVC; ordered start and stop |
| **DaemonSet** | One pod **per node** (log collectors, monitoring agents) | A pod is added automatically when a node joins |
| **Job** | A one-time task (migration, seed script) | Retries until it succeeds, then stops |
| **CronJob** | Scheduled Jobs | Uses cron syntax |

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
      restartPolicy: Never         # a Job must use Never or OnFailure
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

## 22. Helm and Kustomize

> **In one line:** Copy-pasting YAML for dev, staging and prod gets messy fast. Helm and Kustomize solve this.

| Tool | The idea |
|---|---|
| **Helm** | A package manager. YAML **templates** + a `values.yaml`. You can install ready-made charts (nginx-ingress, mongodb, redis, prometheus...). |
| **Kustomize** | Built into kubectl. Keep one **base** YAML and apply small **overlays** (patches) for each environment. No templating. |

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

Installing nginx ingress with Helm (instead of the raw manifest):

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
kubectl kustomize k8s/overlays/prod      # preview the final YAML
kubectl apply -k k8s/overlays/prod       # apply it
```

---

## 23. Useful Tools

| Tool | What it does |
|---|---|
| **Docker Desktop Containers view** | See what is running at a glance |
| **k9s** | A terminal UI for the cluster: browse pods, logs and shells with keyboard shortcuts. `brew install k9s` |
| **Lens / OpenLens** | A desktop GUI for clusters |
| **kubectx / kubens** | Switch context or namespace fast: `kubectx docker-desktop`, `kubens dev` |
| **stern** | Tail logs from many pods at once: `stern express` |
| **kubectl neat** | Removes noisy fields from `-o yaml` output |
| **Kubernetes Dashboard** | The official web UI (separate install) |
| **VS Code Kubernetes extension** | YAML validation and a cluster explorer |
| **kubeconform / kubeval** | Validate YAML in CI before applying |
| **Skaffold / Tilt** | Rebuild and redeploy automatically when code changes |

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

## 24. Cleanup and Reset

```bash
# Delete resources you created from YAML
kubectl delete -f k8s/

# Delete by type and name
kubectl delete deployment express-deployment
kubectl delete svc express-service
kubectl delete hpa express-hpa

# Delete everything in the current namespace (careful!)
kubectl delete all --all

# Delete a whole namespace and everything in it
kubectl delete namespace dev

# Remove the ingress controller
kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.1/deploy/static/provider/cloud/deploy.yaml

# Free disk space by cleaning unused Docker images and containers
docker system prune -a
```

> `kubectl delete all --all` does **not** delete ConfigMaps, Secrets, PVCs or Ingress. Delete those separately.

### Fully reset the local cluster

Docker Desktop → Settings → Kubernetes → **Reset Kubernetes Cluster**. This wipes every resource and gives you a fresh single-node cluster. You will need to install ingress-nginx and metrics-server again.

### Pause Kubernetes to save RAM

Docker Desktop → Settings → Kubernetes → untick **Enable Kubernetes** → Apply. Enable it again later. Your YAML files are safe, but the cluster state is lost.

---

## 25. From Local to EKS

What changes when you move the same YAML from Docker Desktop to AWS EKS:

| Topic | Local (Docker Desktop) | EKS (AWS) |
|---|---|---|
| **Images** | Local Docker store, `imagePullPolicy: IfNotPresent` | Push to **ECR**, use the full image URL, `imagePullPolicy: Always` |
| **Image name** | `express-k8s:latest` | `<account>.dkr.ecr.<region>.amazonaws.com/express-k8s:v1` |
| **Nodes** | 1 node | Multiple nodes (managed node group / Fargate) |
| **Ingress controller** | nginx, LoadBalancer = `localhost` | nginx or the **AWS Load Balancer Controller (ALB)**; a real public DNS name |
| **Service type LoadBalancer** | Maps to localhost | Creates a real AWS load balancer (costs money) |
| **Storage** | hostpath StorageClass | EBS (`gp3`) for RWO, EFS for RWX |
| **Secrets** | Plain k8s Secrets | AWS Secrets Manager + External Secrets / CSI driver |
| **Domain / TLS** | Hosts file, self-signed | Route 53 + ACM / cert-manager |
| **Metrics-server** | Install by hand with `--kubelet-insecure-tls` | Normal install (no insecure flag) |
| **Context** | `docker-desktop` | Added by `aws eks update-kubeconfig` / eksctl |

```bash
# Switching contexts
kubectl config get-contexts
kubectl config use-context docker-desktop
kubectl config use-context <eks-context-name>
```

> Always run `kubectl config current-context` before `apply` or `delete`.

---

## 26. Best Practices Checklist

**Images and containers**
- [ ] Use specific image tags (`v1.2.3`), not just `:latest`
- [ ] Use small base images (`node:20-alpine`) and have a `.dockerignore`
- [ ] The app listens on `0.0.0.0` and handles `SIGTERM` gracefully
- [ ] Run as a non-root user where possible

**Deployments**
- [ ] `resources.requests` **and** `limits` on every container
- [ ] A `readinessProbe` (and a simple `livenessProbe`) configured
- [ ] `replicas >= 2` for anything that must stay available
- [ ] Consistent labels: `selector` == `template.labels` == Service `selector`
- [ ] Rolling update tuned (`maxUnavailable: 0` for zero downtime)

**Config and secrets**
- [ ] Non-secret config in a **ConfigMap**, sensitive config in a **Secret**
- [ ] Secret YAML files in `.gitignore`
- [ ] No connection strings or passwords hardcoded in the image or in YAML stored in git
- [ ] Restart pods after changing env-based ConfigMaps or Secrets

**Networking**
- [ ] Services talk by **service name**, never `localhost`
- [ ] One Ingress as the single entry point from outside
- [ ] `ingressClassName` set explicitly

**Operations**
- [ ] Check `kubectl config current-context` before destructive commands
- [ ] Use namespaces to separate environments and projects
- [ ] Keep YAML in git (one resource per file, in a `k8s/` folder)
- [ ] Use `kubectl apply -f` (declarative) instead of ad-hoc `kubectl edit`
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

* Kubernetes Local Setup Notes · Docker Desktop · MERN Stack · nginx Ingress*
