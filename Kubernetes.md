# Kubernetes on Local Machine — Complete Guide 

**Docker Desktop · MERN Stack · nginx Ingress**


---

## Ye guide kaise padhni hai

Is guide ka flow ek hi kahani follow karta hai:

```
Problem kya hai?  →  K8s ka mental model  →  Setup  →  Pod  →  Deployment  →  Service
   →  (Checkpoint 1: app chalao)  →  Ingress  →  (Checkpoint 2: browser se kholo)
   →  Config + Secrets  →  Probes + Rolling update + Storage
   →  Poora MERN stack  →  Debug + Scaling  →  Extras
```

Har chapter ek hi template follow karta hai, taaki tumhe pata rahe kahan kya milega:

| Section | Matlab |
|---|---|
| **Ek line me** | Chapter ka nichod, 10 second me |
| **Kyun chahiye** | Is cheez ke bina kya problem aati hai |
| **Kaise kaam karta hai** | Concept + diagram |
| **Hands-on** | Seedha chalane wala YAML / commands |
| **Galtiyan** | Jahan log aksar atakte hain |
| **Yaad rakho** | Chapter ka summary |

**Running example:** poori guide me ek hi app use hoti hai — ek **Express backend** jiska Docker image `express-k8s:latest` hai, port **3000** par chalta hai. Aage chal ke isme MongoDB, Redis aur React frontend jod denge.

---

## Table of Contents

**Part 1 — Samajh**
- [1. Kubernetes kya hai aur kyun](#1-kubernetes-kya-hai-aur-kyun)
- [2. Architecture: andar kya chal raha hai](#2-architecture-andar-kya-chal-raha-hai)

**Part 2 — Setup**
- [3. Docker Desktop me Kubernetes enable karo](#3-docker-desktop-me-kubernetes-enable-karo)
- [4. YAML likhna seekho](#4-yaml-likhna-seekho)

**Part 3 — Core Building Blocks**
- [5. Pod](#5-pod)
- [6. Deployment (aur uske andar ReplicaSet)](#6-deployment-aur-uske-andar-replicaset)
- [7. Labels aur Selectors](#7-labels-aur-selectors)
- [8. Service](#8-service)
- [Checkpoint 1: App deploy karke chalao](#checkpoint-1-app-deploy-karke-chalao)

**Part 4 — Bahar se Traffic**
- [9. Ingress Controller](#9-ingress-controller)
- [10. Ingress](#10-ingress)
- [Checkpoint 2: Browser se kholo](#checkpoint-2-browser-se-kholo)

**Part 5 — Configuration**
- [11. Namespaces](#11-namespaces)
- [12. ConfigMap](#12-configmap)
- [13. Secrets](#13-secrets)

**Part 6 — Reliability**
- [14. Health Probes](#14-health-probes)
- [15. Rolling Updates aur Rollback](#15-rolling-updates-aur-rollback)
- [16. Volumes aur Persistent Storage](#16-volumes-aur-persistent-storage)

**Part 7 — Real Project**
- [17. Poora MERN stack Kubernetes par](#17-poora-mern-stack-kubernetes-par)

**Part 8 — Operate karna**
- [18. kubectl daily use](#18-kubectl-daily-use)
- [19. Debugging aur Troubleshooting](#19-debugging-aur-troubleshooting)
- [20. Autoscaling (HPA)](#20-autoscaling-hpa)

**Part 9 — Extras**
- [21. Dusre Workload types](#21-dusre-workload-types)
- [22. Helm aur Kustomize](#22-helm-aur-kustomize)
- [23. Useful Tools](#23-useful-tools)
- [24. Cleanup aur Reset](#24-cleanup-aur-reset)
- [25. Local se EKS tak](#25-local-se-eks-tak)
- [26. Best Practices Checklist](#26-best-practices-checklist)
- [Quick Reference Card](#quick-reference-card)

---

# PART 1 — SAMAJH

## 1. Kubernetes kya hai aur kyun

> **Ek line me:** Kubernetes ek **container manager** hai. Tum bolte ho "mujhe meri app ki 3 copies chahiye", aur wo hamesha 3 copies chalti rakhta hai — crash ho to restart, load badhe to scale, naya version aaye to bina downtime ke update.

### Kyun chahiye

Maan lo tumne Express app ka Docker container bana liya. Local par `docker run` se sab theek hai. Ab production socho:

- Container raat 3 baje crash ho gaya → **kaun restart karega?**
- Traffic 10x ho gaya → **kaun naye containers chalayega?**
- Naya version deploy karna hai → **users ko downtime kyun mile?**
- 5 services hain (frontend, backend, auth, mongo, redis) → **ek dusre ko kaise dhundhenge?** IP har restart par badal jata hai.

Ye sab kaam manually karna painful hai. Kubernetes (short: **K8s**) ye sab automatically karta hai.

### Kubernetes ke bina vs saath

| Situation | Bina Kubernetes | Kubernetes ke saath |
|---|---|---|
| Container crash | App down, manual restart | Automatically restart |
| Traffic spike | App slow ya crash | HPA aur containers add karta hai |
| Multiple services | Manual ports, complex networking | Internal DNS — services naam se baat karti hain |
| Naya version deploy | Downtime | Rolling update — zero downtime |
| Multiple copies | Har container manually manage | `replicas: 3` likho, baaki K8s sambhalta hai |

### Is guide me aane wale key terms (pehle se jaan lo)

| Term | Aasan matlab |
|---|---|
| **Pod** | Sabse chhoti unit. Ek (ya zyada) container ka wrapper. |
| **Deployment** | Pods ka manager — chalu rakhta hai, update aur rollback karta hai. |
| **ReplicaSet** | "N pods hamesha zinda rahein" ki guarantee deta hai. Deployment ke andar hota hai. |
| **Service** | Pods tak pahunchne ka **stable address**. |
| **Ingress** | HTTP routing rules — kaunsa domain/path kis service par jaye. |
| **Ingress Controller** | Wo nginx pod jo Ingress rules padh kar asli traffic route karta hai. |
| **Namespace** | Cluster ke andar virtual cluster. Environments alag karne ke liye. |
| **ConfigMap** | Non-secret config (env variables). |
| **Secret** | Sensitive config (password, token). |
| **Volume / PVC** | Storage jo pod ke baad bhi bacha rahe (database ke liye). |
| **HPA** | Load ke hisaab se pods automatically badhata/ghatata hai. |
| **Node** | Machine (VM/physical) jis par pods chalte hain. Docker Desktop me 1 node. |
| **Cluster** | Control plane + saare nodes. |

### Sabse zaroori idea: Declarative model

Tum K8s ko **order nahi dete** ("container start karo"). Tum **desired state batate ho** ("mujhe 3 copies chahiye"). K8s lagatar dekhta rehta hai ki **actual state = desired state** hai ya nahi. Fark mile to khud theek kar deta hai. Is loop ko **reconciliation** kehte hain.

```
Tum YAML likhte ho (desired state: "3 pods chahiye")
        ↓
API Server use etcd me store karta hai
        ↓
Controllers dekhte rehte hain: actual ≠ desired ?
        ↓
Fark mila → pod create / delete / restart
```

> **Analogy:** Restaurant ka manager. Tum bolte ho "hamesha 3 chef kitchen me rahne chahiye." Ek chef chhutti le le to manager turant naya bulata hai. Tumhe har baar bolna nahi padta.

### Yaad rakho
- K8s = containers ka automatic manager.
- Tum **kya chahiye** likhte ho (YAML), K8s **kaise karna hai** khud dekhta hai.
- Self-healing, scaling, rolling updates, service discovery — ye 4 sabse bade fayde.

---

## 2. Architecture: andar kya chal raha hai

> **Ek line me:** Cluster = **Control Plane** (dimaag, decide karta hai) + **Worker Nodes** (haath-pair, jahan pods chalte hain). Docker Desktop me dono tumhari ek machine par, ek node `docker-desktop` me hote hain.

### Cluster ka poora picture

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
│  └──────────────────────────────┘     │       └─ Pod 3 (HPA ne banaya)    │ │
│                                       └───────────────────────────────────┘ │
│  Service · Ingress · HPA · ConfigMap / Secret                                │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Control Plane ke 4 hisse

| Component | Kaam | Analogy |
|---|---|---|
| **API Server** | Har `kubectl` command ka entry point | Reception desk |
| **Scheduler** | Naya pod kis node par chalega, ye decide karta hai | Seat allotment wala |
| **Controller Manager** | Control loops chalata hai (ReplicaSet, Deployment...) | Floor manager jo gap check karta rehta hai |
| **etcd** | Poore cluster ki state ka key-value database | Register / record book |

### Worker Node ke 3 hisse

| Component | Kaam |
|---|---|
| **kubelet** | Node ka agent. API server se kaam leta hai, pods chalu rakhta hai |
| **kube-proxy** | Services ke liye networking rules sambhalta hai |
| **Container runtime** | Asli me containers chalata hai (containerd/Docker) |

### Jab tum `kubectl apply -f deployment.yaml` chalate ho to kya hota hai

1. `kubectl` YAML ko **API Server** ko bhejta hai.
2. API Server validate karke **etcd** me store karta hai.
3. **Deployment controller** dekhta hai naya Deployment aaya → **ReplicaSet** banata hai.
4. **ReplicaSet controller** dekhta hai "2 pods chahiye, 0 hain" → **Pod objects** banata hai.
5. **Scheduler** har pod ko ek node assign karta hai.
6. Us node ka **kubelet** image pull karke container start karta hai.
7. Pod `Running` → `Ready` hota hai → Service us par traffic bhejna shuru karti hai.

### Traffic ka safar (request ki journey)

```
Browser
   │  http://express.local ya http://localhost
   ▼
Ingress Controller (nginx pod)     ← asli traffic yahan aata hai
   │  Ingress rules padhta hai
   ▼
Ingress rules (tumhari ingress.yaml)  ← "kis path ko kahan bhejna hai"
   ▼
Service (ClusterIP)                ← label se pods dhundhta hai, load balance karta hai
   ▼
Pod (Express container)            ← request handle karke response deta hai
```

Is journey ko yaad rakho — ye guide ke Part 3 aur 4 me ek-ek step karke banayenge.

### Yaad rakho
- Control Plane **decide** karta hai, Worker Node **chalata** hai.
- `kubectl` hamesha API Server se baat karta hai, kisi pod se seedha nahi.
- Docker Desktop ka cluster single-node hai — seekhne ke liye perfect, production jaisa multi-node nahi.

---

# PART 2 — SETUP

## 3. Docker Desktop me Kubernetes enable karo

> **Ek line me:** Settings me ek checkbox, 2-3 minute wait, aur tumhare laptop par poora Kubernetes cluster ready.

### Step 1 — Enable karo

Docker Desktop → **Settings (gear icon)** → **Kubernetes** → **Enable Kubernetes** tick karo → **Apply & Restart**.

2–3 minute wait karo. Neeche status bar me **green Kubernetes icon** aa jaye to ready.

> 💡 RAM kam ho to Docker Desktop ko kam se kam **4 GB** do (Settings → Resources). Isse kam me K8s + pods + metrics-server slow lagega.

### Step 2 — Verify karo kubectl chal raha hai

```bash
kubectl version
kubectl get nodes

# Expected output:
NAME             STATUS   ROLES
docker-desktop   Ready    control-plane
```

### Step 3 — Context check karo

```bash
kubectl config current-context
# Output hona chahiye: docker-desktop

kubectl config get-contexts
# kubectl jitne clusters ko jaanta hai unki list
```

`~/.kube/config` file me saare clusters ki connection info rehti hai. Baad me EKS cluster banaoge to wahan nayi entry add hogi aur context automatically EKS par switch ho jayega. Wapas aane ke liye:

```bash
kubectl config use-context docker-desktop
```

> ⚠️ **Galti jo mehengi padti hai:** `apply` ya `delete` chalane se pehle hamesha `kubectl config current-context` dekho. Local ke bajay production cluster par command chal gayi to bada nuksan hota hai.

### Local images kaise kaam karti hain

Docker Desktop ka Kubernetes **wahi Docker image store** use karta hai jo tumhara `docker` CLI karta hai. Matlab `docker build -t express-k8s:latest .` ke baad image K8s ko turant dikhti hai — registry me push karne ki zaroorat nahi. Isi liye local me `imagePullPolicy: IfNotPresent` kaam karta hai.

> ⚠️ Agar tag `:latest` hai aur `imagePullPolicy` set nahi kiya, to K8s default `Always` maan leta hai aur Docker Hub se pull karne ki koshish karta hai → **`ImagePullBackOff`**. Local images ke liye hamesha `IfNotPresent` set karo (ya `v1` jaisa versioned tag use karo).

### Yaad rakho
- `kubectl get nodes` me `docker-desktop  Ready` dikhna chahiye.
- Context check karna adat bana lo.
- Local image ke liye `imagePullPolicy: IfNotPresent`.

---

## 4. YAML likhna seekho

> **Ek line me:** K8s ka har resource ek YAML file hai jisme 4 cheezein zaroor hoti hain: `apiVersion`, `kind`, `metadata`, `spec`.

### Kyun pehle YAML?
Aage ke saare chapters me tum YAML hi likhoge. Isliye pehle structure samajh lo, phir har resource me bas `spec` badlega.

### Chaar mandatory fields

```yaml
apiVersion: ...    # kaunsa API group/version
kind: ...          # kis type ka resource
metadata:          # name, labels, namespace, annotations
  name: ...
spec:              # desired state (shape kind ke hisaab se badalti hai)
  ...
```

### apiVersion kaise pata kare

Alag resources alag **API groups** me rehte hain:

| Resource | apiVersion | Kyun |
|---|---|---|
| Pod, Service, ConfigMap, Secret, Namespace, PVC | `v1` | Core group — sabse purane, basic resources. Group prefix nahi lagta. |
| Deployment, ReplicaSet, StatefulSet, DaemonSet | `apps/v1` | Apps group — pods manage karne wale high-level resources. |
| Ingress, NetworkPolicy | `networking.k8s.io/v1` | Networking group. |
| HorizontalPodAutoscaler | `autoscaling/v2` | Autoscaling group. |
| Job, CronJob | `batch/v1` | Batch group — ek baar ya scheduled kaam. |

> 💡 Bhool jao to: `kubectl explain deployment` ya `kubectl api-resources`.

### Ek file me multiple resources

`---` se alag karo:

```yaml
apiVersion: apps/v1
kind: Deployment
# ... deployment config

---                    # separator — yahan naya resource shuru

apiVersion: v1
kind: Service
# ... service config
```

> 💡 Project bada ho to files **alag** rakho (ek resource, ek file) aur poora folder ek saath apply karo: `kubectl apply -f k8s/`.

### Project structure (hum ye banayenge)

```
your-project/
├── Backend/
│   ├── server.js
│   ├── package.json
│   └── dockerfile
└── k8s/                  # saari Kubernetes YAML files yahan
    ├── deployment.yaml   # app kaise chalegi
    ├── service.yaml      # app tak kaise pahunchenge
    └── ingress.yaml      # bahar ka traffic kaise andar aayega
```

### YAML ko generate karna (type karne se bacho)

```bash
# dry-run sirf YAML print karta hai, kuch create nahi karta
kubectl create deployment express --image=express-k8s:latest --dry-run=client -o yaml > deployment.yaml
kubectl create service clusterip express-service --tcp=80:3000 --dry-run=client -o yaml > service.yaml

# Kisi bhi resource ke saare fields dekhne ke liye
kubectl explain deployment.spec.template.spec.containers
```

### Imperative vs Declarative

| Style | Example | Kab use kare |
|---|---|---|
| **Imperative** | `kubectl create deployment ...`, `kubectl scale ...` | Quick experiments, seekhne ke liye |
| **Declarative** | `kubectl apply -f deployment.yaml` | Real projects — git me YAML hi source of truth |

### `apply` vs `create` vs `replace`

| Command | Kya karta hai |
|---|---|
| `kubectl create -f` | Banata hai; already hai to **error** |
| `kubectl apply -f` | Banata ya update karta hai; dobara chalana safe. **Ye use karo.** |
| `kubectl replace -f` | Poora replace; resource na ho to error |

### Galtiyan
- YAML me **tab nahi**, sirf **spaces** (2 spaces). Tab se error aata hai.
- Strings jo number/bool jaisi dikhti hain, quote me likho: `"128Mi"`, `"true"`.
- Apply se pehle check: `kubectl apply -f file.yaml --dry-run=server`.

### Yaad rakho
- Har YAML = `apiVersion + kind + metadata + spec`.
- `apply` hi default command hai.
- Doubt ho to `kubectl explain`.

---

# PART 3 — CORE BUILDING BLOCKS

## 5. Pod

> **Ek line me:** **Pod** = ek ya zyada containers ka wrapper. Ye K8s ki sabse chhoti unit hai. Tum pods ko seedha kam hi banate ho — Deployment banata hai.

### Kyun chahiye
K8s container ko seedha nahi chalata, pod ke andar chalata hai. Pod ke andar ke saare containers **ek hi IP aur ek hi network (`localhost`)** share karte hain, aur storage bhi share kar sakte hain. Zyada-tar pods me bas ek container hota hai: tumhari app.

### Pod ke andar kya hota hai

| Cheez | Matlab |
|---|---|
| **Container(s)** | Ek ya zyada Docker containers. Pod ke andar `localhost` share karte hain. |
| **IP Address** | Har pod ka apna IP. **Har restart par badal jata hai** — isi liye Service use karte hain, pod IP nahi. |
| **Labels** | `app: express` jaise tags. Service/Deployment inhe use karke pod dhundhte hain. |
| **Resources** | CPU/memory ki limits aur requests. |

### Pod ka lifecycle

```
Pending            → pod schedule hua, image pull ho rahi hai
   ↓
Running            → container start ho gaya
   ↓
Ready              → readiness probe pass, ab traffic aayega
   ↓
CrashLoopBackOff   → container baar-baar crash ho raha hai, K8s retry kar raha hai
   ↓
ImagePullBackOff   → Docker image pull nahi ho pa rahi
```

Baaki states: `Succeeded` (job khatam), `Failed`, `Terminating` (delete ho raha hai), `Evicted` (node ke resources khatam), `OOMKilled` (container ne memory limit cross ki).

### Hands-on: ek test pod

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

Delete karne ke baad pod **wapas nahi aayega** — kyunki kisi ne use manage nahi kiya. Yahi pod ki sabse badi kamzori hai, aur Deployment isi ko theek karta hai (agla chapter).

> 💡 Production me pods kabhi seedha mat banao. Standalone pod crash ho jaye to koi restart nahi karega.

### Multi-container pod (sidecar pattern)

Ek pod ke containers network + volumes share karte hain. Common pattern: **main app + sidecar** (log shipper, proxy, config reloader).

```
Pod
├── container: express      (main app, port 3000)
└── container: log-shipper  (shared volume se logs padhke bahar bhejta hai)
```

### Pod ko inspect karna

```bash
kubectl get pods                       # saare pods
kubectl get pods -o wide               # pod IP aur node ke saath
kubectl describe pod <pod-name>        # poori details + events
kubectl logs <pod-name>                # app ka console output
kubectl exec -it <pod-name> -- sh      # pod ke andar shell
```

### Yaad rakho
- Pod = containers ka wrapper, apna IP, restart par IP badalta hai.
- Standalone pod mar jaye to wapas nahi aata → **Deployment use karo**.
- Debug ke liye `describe`, `logs`, `exec`.

---

## 6. Deployment (aur uske andar ReplicaSet)

> **Ek line me:** **Deployment** pods ka manager hai. Tum batate ho kaunsa image, kitni copies, kitne resources — wo hamesha utne pods zinda rakhta hai, update aur rollback bhi sambhalta hai.

### Kyun chahiye
Pichhle chapter me dekha: akela pod mar jaye to khatam. Deployment wo manager hai jo bolta hai: "2 pods hamesha chalte rehne chahiye." Ek crash hua → naya bana deta hai.

### Deployment ki zimmedariyan

| Kaam | Matlab |
|---|---|
| **Replica management** | Hamesha exact utne pods chalu rakhta hai. Ek crash hua to turant naya. |
| **Rolling update** | Naya image push karo to pods ek-ek karke replace hote hain. Purane tab tak traffic dete hain jab tak naye ready na ho — zero downtime. |
| **Rollback** | Naye version me bug ho to ek command se purane working version par wapas. |
| **Scaling** | Manually replicas badlo ya HPA se automatic. |

### Poori deployment.yaml (line-by-line comments ke saath)

```yaml
apiVersion: apps/v1          # Deployment 'apps' group me rehta hai
kind: Deployment
metadata:
  name: express-deployment
spec:
  replicas: 2              # hamesha 2 pods chalte rahein
  selector:
    matchLabels:
      app: express          # (1) Deployment un pods ko manage karega jinpar ye label ho
  template:              # har pod ka blueprint
    metadata:
      labels:
        app: express        # (2) har pod par ye label chipka do
        owner: ankur       # extra label — Deployment ise ignore karta hai
    spec:
      containers:
        - name: express
          image: express-k8s:latest
          imagePullPolicy: IfNotPresent  # local image ho to wahi use karo
          ports:
            - containerPort: 3000         # tumhari app is port par sunti hai
          resources:
            limits:
              memory: "128Mi"            # pod maximum itni memory le sakta hai
              cpu: "500m"               # maximum CPU (500m = aadha core)
            requests:
              memory: "64Mi"             # is pod ke liye reserved memory
              cpu: "250m"               # reserved CPU (HPA ke liye zaroori)
```

### Important fields ka matlab

| Field | Value | Kya karta hai |
|---|---|---|
| `apiVersion` | `apps/v1` | Deployment `apps` group me hai, core group (`v1`) me nahi — wo Pod/Service ke liye hai. |
| `replicas` | `2` | Hamesha 2 running pods. Ek crash → naya turant. |
| `selector.matchLabels` | `app: express` | Deployment un pods ko manage karta hai jinpe **saare** ye labels hon. `template.metadata.labels` se match hona zaroori. |
| `template.metadata.labels` | `app: express` | Is Deployment ke har pod par lagne wale labels. Service inhi se pods dhundhti hai. |
| `imagePullPolicy` | `IfNotPresent` | Local image ho to wahi. EKS par `Always` karo taaki ECR se pull ho. |
| `resources.requests` | `cpu: 250m` | Is pod ke liye reserved CPU/memory. **HPA ko requests chahiye** — utilization % isi se nikalta hai. |
| `resources.limits` | `cpu: 500m` | Pod maximum itna use kar sakta hai. Ek pod dusron ko bhookha na rakhe. |

> ⚠️ `spec.selector` **immutable** hai (banne ke baad badal nahi sakte). Badalna ho to Deployment delete karke dobara banao.

### CPU aur Memory units

| CPU | Matlab | Memory | Matlab |
|---|---|---|---|
| `1000m` | 1 poora core | `64Mi` | 64 mebibyte (≈ 67 MB) |
| `500m` | Aadha core | `128Mi` | 128 mebibyte |
| `250m` | Chauthai core | `1Gi` | 1 gibibyte |

### Requests vs Limits: limit cross hone par kya hota hai

| Resource | **Limit** cross | Natija |
|---|---|---|
| **CPU** | Throttle (slow) | Pod chalta rehta hai, bas slow ho jata hai |
| **Memory** | Container kill | Status `OOMKilled`, phir restart |

**Requests** ko Scheduler use karta hai ye decide karne me ki pod kahan fit hoga. **Limits** runtime par enforce hoti hain.

### Andar ka ReplicaSet — Deployment kaise kaam karta hai

Deployment khud pods nahi banata. Wo ek **ReplicaSet** banata hai, aur ReplicaSet pods banata hai.

```
Deployment  →  ReplicaSet manage karta hai
    ↓
ReplicaSet  →  "replicas: 2" matlab 2 pods hamesha zinda
    ↓
Pod 1 (running)   Pod 2 (running)
    ↓  Pod 1 crash hua
ReplicaSet dekhta hai: 1 chal raha hai, chahiye 2
    ↓
ReplicaSet automatically Pod 3 banata hai
```

**Fir Deployment ki zaroorat kyun, sirf ReplicaSet kyun nahi?**

| Agar use karo | Milta hai |
|---|---|
| **Deployment** | ReplicaSet management + rolling updates + rollback history. Practice me hamesha ye. |
| **ReplicaSet seedha** | Sirf pod count maintain. Rolling update nahi, rollback nahi. Recommended nahi. |

`kubectl get replicasets` me tumhe auto-generated naam dikhega, jaise `express-deployment-7d9f8b6c4`. Ye hash pod template ka fingerprint hai. Image update karne par **naya ReplicaSet** banta hai aur purana scale down hokar 0 par rehta hai (rollback ke liye bacha ke rakha jata hai):

```bash
kubectl get replicasets

# Rollout ke baad 2 ReplicaSets dikhenge:
# express-deployment-7d9f8b6c4   2   2   2   (naya — active)
# express-deployment-5c8a3d1b2   0   0   0   (purana — rollback ke liye rakha)
```

ReplicaSet pods ko **label selector** se gintaa hai. Agar tum same labels wala pod haath se bana do, to ReplicaSet use bhi ginega aur ho sakta hai apna ek pod delete kar de. Isliye har Deployment ke labels unique rakho.

Reference ke liye ReplicaSet ki YAML (practice me Deployment use karo):

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

### Hands-on: self-healing dekho

```bash
kubectl get pods                         # pod ke naam note karo
kubectl delete pod <ek-pod-ka-naam>      # ek ko maar do
kubectl get pods -w                      # dekho: naya pod turant aa jata hai
```

### Deployment ke useful commands

```bash
kubectl get deployments
kubectl rollout status deployment/express-deployment
kubectl rollout undo deployment/express-deployment    # rollback
kubectl scale deployment express-deployment --replicas=5
kubectl rollout restart deployment/express-deployment # image dobara pull
```

### Yaad rakho
- Deployment → ReplicaSet → Pods (teen layer).
- `selector.matchLabels` aur `template.labels` match hone chahiye.
- `requests` HPA ke liye zaroori, `limits` pod ko control me rakhti hai.
- Memory limit cross = OOMKilled, CPU limit cross = bas slow.

---

## 7. Labels aur Selectors

> **Ek line me:** **Labels** pods par lage key-value tags hain, **Selectors** unhe filter karne wale. Deployment aur Service dono isi se pods dhundhte hain — koi pod ka naam ya IP hardcode nahi hota.

### Kyun chahiye
Pods mar kar naye banenge, naam aur IP badlenge. To "ye wala pod" kehna bekaar hai. Isliye K8s kehta hai: "jis pod par `app=express` label ho, wahi mera hai."

### Teen-tarfa connection

```yaml
# deployment.yaml
selector:
  matchLabels:
    app: express      # (1) Deployment is label wale pods ko manage karta hai
template:
  metadata:
    labels:
      app: express    # (2) Pod par ye label lagta hai

---
# service.yaml
selector:
  app: express        # (3) Service is label wale pods ko traffic bhejti hai
```

(1) aur (3) ko (2) se match karna zaroori hai. Yahi pura jaadu hai.

### Selector matching rules

| Pod ke Labels | Selector: `app=express` | Result |
|---|---|---|
| `app: express` | Match | ✅ Selected |
| `app: express, env: prod` | Match (extra label ignore) | ✅ Selected |
| `app: backend` | Match nahi | ❌ Ignored |
| `app: express, env: staging` | `app=express` match hota hai | ✅ Selected |

### Labels se staging aur production alag karna

```yaml
# staging-deployment.yaml
selector:
  matchLabels:
    app: express
    env: staging     # sirf staging pods manage karega

# production-deployment.yaml
selector:
  matchLabels:
    app: express
    env: production  # sirf production pods manage karega
```

Dono ek hi cluster me rehte hain par ek dusre me dakhal nahi dete.

### CLI se labels ke saath kaam

```bash
kubectl get pods --show-labels
kubectl get pods -l app=express                 # label se filter
kubectl get pods -l 'env in (staging,prod)'     # set-based selector
kubectl label pod <pod-name> env=debug          # label add
kubectl label pod <pod-name> env-               # label "env" hatao
```

### Labels vs Annotations

| | Labels | Annotations |
|---|---|---|
| Kaam | Objects **pehchanna aur select** karna | Extra **metadata** lagana |
| Selectors use karte hain? | Haan | Nahi |
| Example | `app: express` | `nginx.ingress.kubernetes.io/proxy-body-size: "10m"` |

### Galtiyan
- Label me typo (`app: expres`) → Service ko koi pod nahi milta → `kubectl get endpoints` me `<none>`.
- Do alag Deployments ke same labels → ek dusre ke pods ginne lagte hain.

### Yaad rakho
- Label = tag, Selector = filter.
- Deployment (own pods) aur Service (route traffic) dono selector se kaam karte hain.

---

## 8. Service

> **Key idea:** Pods temporary hain, unka IP badalta rehta hai. **Service** ek **stable address** deti hai jo kabhi nahi badalta, aur peeche ke pods me traffic baant deti hai.

### Kyun chahiye
Maan lo frontend backend ko call karna chahta hai. Backend pod ka IP `10.1.0.5` tha, wo crash hua, naya pod `10.1.0.9` par aaya. Frontend ka kya? Service ye problem solve karti hai: wo ek fixed naam/IP deti hai, aur andar se **label selector** se live pods dhundhti rehti hai. Pod add/remove ho to list turant update.

### Basic service.yaml

```yaml
apiVersion: v1              # Service core resource hai
kind: Service
metadata:
  name: express-service
spec:
  selector:
    app: express            # app=express label wale SAARE pods dhundho
  ports:
    - protocol: TCP
      port: 80              # Service cluster ke andar is port par sunti hai
      targetPort: 3000      # container ke andar tumhari app ka port
  type: ClusterIP          # sirf cluster ke andar accessible
```

### port vs targetPort

| Field | Value | Matlab |
|---|---|---|
| `port` | `80` | Service ka port. Dusre pods `express-service:80` call karte hain. |
| `targetPort` | `3000` | Container ke andar asli port. Deployment ke `containerPort` se match hona chahiye. |

### Teen Service types

| Type | Kahan se accessible | Kab use kare |
|---|---|---|
| **ClusterIP** | Sirf cluster ke andar | Pod-to-pod. Default. Ingress ke saath use hota hai. |
| **NodePort** | Bahar se port number ke through | Local testing. `localhost:30001`. Ingress nahi chahiye. |
| **LoadBalancer** | Cloud load balancer se | Production (AWS/GCP). Asli LB banta hai, paise lagte hain. |

(Ek aur type hai **ExternalName**: Service ke naam ko kisi external DNS naam se map karta hai — jaise managed DB.)

### NodePort (local testing ke liye)

```yaml
type: NodePort
ports:
  - port: 80
    targetPort: 3000
    nodePort: 30001    # browser me localhost:30001
```

NodePort range: **30000–32767**.

### Service load balancing kaise karti hai

```
Request aati hai (selector: app=express)
    ↓
Endpoints list dekhti hai (live pod IPs)
    ↓
Request 1 → Pod 1 (192.168.1.10)
Request 2 → Pod 2 (192.168.1.11)
Request 3 → Pod 3 (192.168.1.12)
Request 4 → Pod 1 (round robin)
```

```bash
kubectl get services
kubectl get endpoints express-service   # pool me kaunse pod IPs hain
```

### Service DNS: pods naam se baat karte hain

Har Service ko CoreDNS ke through internal DNS naam milta hai:

```
<service-name>.<namespace>.svc.cluster.local
```

| Kahan se call kar rahe ho | Kaise |
|---|---|
| Same namespace | `http://express-service` |
| Dusra namespace | `http://express-service.other-ns` |
| Poora naam | `http://express-service.default.svc.cluster.local` |

```js
// Kisi dusre pod ke andar (jaise frontend se backend call)
const res = await fetch("http://express-service/api/users"); // port 80 → targetPort 3000
```

> ⚠️ Pod ke andar `localhost` ka matlab **sirf wahi pod** hai. Dusri service ko call karna ho to **service ka naam** use karo, kabhi `localhost` nahi (isi se `ECONNREFUSED` aata hai).

### Yaad rakho
- Service = stable address + load balancer, pods ko label se dhundhti hai.
- `targetPort` = container ka port, `port` = service ka port.
- Cluster ke andar: service name se baat. Bahar se: NodePort / Ingress.

---

## Checkpoint 1: App deploy karke chalao

Ab tak Pod, Deployment, Labels, Service seekhe. Ab chalake dekhte hain.

### Step 1 — Docker image banao

```bash
cd Backend/
docker build -t express-k8s:latest .
```

Backend ka minimal `dockerfile`:

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

> App ko `0.0.0.0` par listen karna chahiye (ya bas `app.listen(3000)`), sirf `127.0.0.1` par nahi, warna Service us tak pahunch nahi paati.

### Step 2 — YAML apply karo

```bash
kubectl apply -f k8s/    # k8s folder ki saari files apply

# Ya ek-ek karke:
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

### Step 3 — Verify karo

```bash
kubectl get pods           # STATUS: Running, READY: 1/1
kubectl get services       # service sahi ports ke saath dikhe
kubectl get endpoints express-service   # pod IPs dikhne chahiye
```

### Step 4 — Abhi (Ingress ke bina) app kholo

Ingress agle part me aayega. Abhi do aasan tarike:

```bash
# Tarika A: port-forward (kisi bhi service par kaam karta hai)
kubectl port-forward svc/express-service 8080:80
# Ab browser: http://localhost:8080
```

Ya Service me `type: NodePort` + `nodePort: 30001` karke `http://localhost:30001`.

Agar ye chal gaya to Part 3 pura hua: Pod → Deployment → Service sab kaam kar raha hai.

### Code change ke baad update

```bash
# 1. Code badla → image dobara banao
docker build -t express-k8s:latest .

# 2. Deployment restart karo taaki naya image uthaye
kubectl rollout restart deployment/express-deployment
```

### Delete karna ho to

```bash
kubectl delete -f k8s/                              # folder ki sab cheezein delete
kubectl delete deployment express-deployment        # ek resource delete
```

---

# PART 4 — BAHAR SE TRAFFIC

## 9. Ingress Controller

> **Ek line me:** **Ingress Controller** ek **nginx pod** hai jo cluster ke andar chalta hai, Ingress rules padhta hai, aur asli HTTP traffic route karta hai. Iske bina Ingress YAML sirf ek bekaar config file hai.

### Kyun pehle Controller, Ingress baad me?
Ingress sirf **rules** hain ("is path ko us service par bhejo"). Rules ko koi **lagu** bhi karna padta hai. Wahi kaam Controller karta hai. Isliye controller ek baar install karna padta hai.

> **Analogy:** Ingress = reception ka "kaun kis floor jayega" wala chart. Ingress Controller = wo receptionist jo chart padh kar logon ko bhejta hai. Receptionist na ho to chart deewar par lata raha.

### Ingress vs Ingress Controller

| | Ingress Resource | Ingress Controller |
|---|---|---|
| Kya hai | Routing rules wali YAML file | Cluster me chalta nginx pod |
| Kaun banata hai | Tum — `kubectl apply -f ingress.yaml` | Ek baar install — `kubectl apply -f nginx-url` |
| Traffic route karta hai? | Nahi, sirf rules define karta hai | Haan, rules padh ke asli traffic route karta hai |
| Namespace | `default` (tumhari app ka) | `ingress-nginx` (apna alag) |

### Install karo

```bash
kubectl apply -f \
  https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.1/deploy/static/provider/cloud/deploy.yaml
```

### Verify karo

```bash
kubectl get pods -n ingress-nginx

# Expected:
NAME                                        READY   STATUS
ingress-nginx-controller-xxxxxxxxx-xxxxx    1/1     Running

kubectl get svc -n ingress-nginx
# ingress-nginx-controller   LoadBalancer   ...   EXTERNAL-IP: localhost
```

Docker Desktop par controller ki `LoadBalancer` Service ko `EXTERNAL-IP: localhost` milta hai. Isi liye `http://localhost` (port 80/443) tumhari app tak pahunch jata hai.

### Controller ke logs (404/502 debug ke liye bahut kaam ke)

```bash
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller
```

### Yaad rakho
- Controller ek baar install hota hai, alag namespace (`ingress-nginx`) me.
- Ingress YAML ke bina controller kuch route nahi karega, aur controller ke bina Ingress YAML bekaar hai.

---

## 10. Ingress

> **Ek line me:** **Ingress** = HTTP routing rules ki YAML: "`express.local/` ko `express-service` par bhejo", "`/auth` ko `auth-service` par bhejo". Ek hi entry point, kai services.

### Kyun chahiye
Har service ke liye alag NodePort ya LoadBalancer banana mehenga aur ganda hai. Ingress se **ek hi entry point** (port 80/443) se host aur path ke hisaab se multiple services tak pahunch sakte ho.

### Poori ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1   # Ingress networking group me hai
kind: Ingress
metadata:
  name: express-ingress
spec:
  ingressClassName: nginx         # kaunsa controller is rule ko sambhalega
  rules:
    - host: express.local          # kaunsa domain match kare
      http:
        paths:
          - path: /
            pathType: Prefix         # / aur uske baad ka sab match
            backend:
              service:
                name: express-service
                port:
                  number: 80
```

### Important fields

| Field | Value | Kya karta hai |
|---|---|---|
| `ingressClassName` | `nginx` | Batata hai kaunsa controller ye rule uthaye. nginx aur traefik dono ho to har ek sirf apne rules uthata hai. |
| `host` | `express.local` | Sirf is Host header wala traffic match hota hai. Local me hosts file me add karna padta hai. |
| `pathType: Prefix` | `Prefix` | Path aur uske aage sab match. `/api` → `/api/users`, `/api/data` bhi. |
| `pathType: Exact` | `Exact` | Sirf wahi exact path. `/api` → `/api/users` match **nahi** hota. |

### Local me sabse aasan: host hata do

```yaml
rules:
  - http:                     # host: field nahi — saara traffic match
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: express-service
              port:
                number: 80
```

`http://localhost` se seedha khulega, hosts file edit nahi karni padegi.

### Host ke saath: `express.local` hosts file me add karo

**Mac / Linux**

```bash
sudo nano /etc/hosts

# Neeche ye line add karo:
127.0.0.1  express.local
```

**Windows** (Notepad **Administrator** me kholo)

```
File: C:\Windows\System32\drivers\etc\hosts
Line: 127.0.0.1  express.local
```

Ab browser me `http://express.local` nginx Ingress Controller par aayega.

### Path-based routing: ek domain, kai services

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

nginx hamesha **sabse lamba matching path** chunta hai, isliye `/auth/...` auth-service par jayega aur baaki sab main-service par.

### Useful nginx annotations

```yaml
metadata:
  name: express-ingress
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"        # bade uploads allow (default 1m)
    nginx.ingress.kubernetes.io/proxy-read-timeout: "120"     # slow APIs ke liye (seconds)
    nginx.ingress.kubernetes.io/rewrite-target: /$2           # path prefix hataana (neeche dekho)
    nginx.ingress.kubernetes.io/enable-cors: "true"           # ingress level par CORS
```

### Path prefix strip karna (rewrite)

`localhost/api/users` ko backend tak `/users` banakar bhejna ho:

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

nginx ingress WebSocket upgrade by default support karta hai. Lambe connections ke liye timeouts badhao:

```yaml
annotations:
  nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
  nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
```

Agar Socket.io ke saath **ek se zyada backend replicas** hain, to sticky sessions chahiye (ya Redis adapter):

```yaml
annotations:
  nginx.ingress.kubernetes.io/affinity: "cookie"
```

### HTTPS locally (optional)

```bash
# self-signed certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=express.local"

# TLS secret banao
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

### Galtiyan
- `ingressClassName` bhool gaye → koi controller rule nahi uthata, `ADDRESS` khali.
- Backend `service.name` ya `port.number` galat → **503**.
- Host/path match nahi → **404**.

### Yaad rakho
- Ingress = rules. Controller = unhe chalane wala.
- Local me: host hata do (`localhost`) ya hosts file me `express.local` add karo.
- Longest path match jeetta hai.

---

## Checkpoint 2: Browser se kholo

```bash
kubectl apply -f k8s/ingress.yaml
kubectl get ingress           # ADDRESS dikhna chahiye (localhost)
```

| Setup | URL |
|---|---|
| Ingress me host nahi | `http://localhost` |
| `host: express.local` + hosts file | `http://express.local` |
| NodePort service | `http://localhost:30001` |

Poora safar ab chal raha hai: **Browser → Ingress Controller → Ingress rule → Service → Pod**. Agar nahi chala to [Chapter 19](#19-debugging-aur-troubleshooting) ka flowchart follow karo.

---

# PART 5 — CONFIGURATION

## 11. Namespaces

> **Ek line me:** **Namespace** cluster ke andar ka virtual cluster hai. Resources ko group karta hai taaki naam na takrayein aur `dev`, `staging`, `prod` ya alag projects alag rakh sako. Kuch na bolo to sab `default` me jata hai.

### Pehle se maujood namespaces

| Namespace | Kya rehta hai |
|---|---|
| `default` | Tumhare resources, agar kuch aur na bolo |
| `kube-system` | K8s ke system pods (coredns, kube-proxy, metrics-server...) |
| `kube-public` | Publicly readable data (kam use hota hai) |
| `kube-node-lease` | Nodes ke heartbeat |
| `ingress-nginx` | Ingress controller install karne par banta hai |

### Banana aur use karna

```bash
kubectl create namespace dev
kubectl get namespaces

kubectl apply -f k8s/ -n dev          # dev namespace me apply
kubectl get pods -n dev               # dev ke pods
kubectl get pods -A                   # SAARE namespaces ke pods

# Current context ka default namespace dev kar do
kubectl config set-context --current --namespace=dev
```

YAML me:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

Ya resource me seedha:

```yaml
metadata:
  name: express-deployment
  namespace: dev
```

### Kaun namespaced hai, kaun cluster-wide

| Namespaced (har namespace me alag) | Cluster-wide |
|---|---|
| Pod, Deployment, Service, ConfigMap, Secret, Ingress, PVC, HPA | Node, Namespace, PersistentVolume, StorageClass, IngressClass |

### Cross-namespace DNS

```
http://<service>.<namespace>.svc.cluster.local
jaise: http://express-service.dev.svc.cluster.local
```

### Namespace ki resource limit (optional)

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

> ⚠️ Namespace delete karne par uske **andar ka sab kuch delete** ho jata hai. `kubectl delete namespace dev` destructive hai.

### Yaad rakho
- Namespace = virtual cluster, environments aur projects alag karne ke liye.
- Namespace ke andar Services naam se baat karti hain; bahar se `service.namespace` se.

---

## 12. ConfigMap

> **Ek line me:** **ConfigMap** me **non-sensitive** config (port, URL, `NODE_ENV`, feature flags) Docker image se bahar rakhte hain. Ek hi image dev/staging/prod me alag config ke saath chal sakti hai. Password ke liye Secret (agla chapter).

### Kyun chahiye
Agar `NODE_ENV=production` image me hardcode kar diya to har environment ke liye alag image banani padegi. ConfigMap se image same rehti hai, config alag.

### ConfigMap banao

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

Ya CLI se:

```bash
kubectl create configmap express-config \
  --from-literal=NODE_ENV=production \
  --from-literal=PORT=3000

kubectl create configmap app-env --from-env-file=.env   # .env file se (sirf non-secret wale!)
```

### Deployment me use karne ke 3 tarike

**1. Ek-ek key**

```yaml
env:
  - name: NODE_ENV
    valueFrom:
      configMapKeyRef:
        name: express-config
        key: NODE_ENV
```

**2. Saari keys ek saath (`envFrom`)** — sabse aasan

```yaml
envFrom:
  - configMapRef:
      name: express-config
  - secretRef:
      name: express-secret      # Secrets bhi isi tarah bulk me load hote hain
```

**3. Files ki tarah mount**

```yaml
volumeMounts:
  - name: config-vol
    mountPath: /app/config
volumes:
  - name: config-vol
    configMap:
      name: express-config      # har key /app/config me ek file ban jati hai
```

### ConfigMap update karne par kya hota hai

| Kaise use kiya tha | Changed ConfigMap ko `apply` karne ke baad |
|---|---|
| Env variables | **Update NAHI hota** — pods restart karo: `kubectl rollout restart deployment/express-deployment` |
| Mounted file | Thodi der me khud update (~1 min tak) |

### Inspect

```bash
kubectl get configmaps
kubectl describe configmap express-config
kubectl get configmap express-config -o yaml
```

### Yaad rakho
- ConfigMap = non-secret config, Secret = sensitive.
- `envFrom` sabse kam typing.
- Env-based config badalne ke baad pods restart karna padta hai.

---

## 13. Secrets

> **Ek line me:** **Secret** DB password, API keys, JWT secrets jaisi sensitive cheezein cluster me (base64 encoded) rakhta hai, aur pods ko env variables ya files ke roop me deta hai.

### Kyun alag se Secret?
Secrets ko K8s special treat karta hai (logs me nahi aate, RBAC se restrict ho sakte hain). Local Docker Desktop me plain Secrets theek hain; production me external vaults use karte hain.

> ⚠️ **base64 encryption NAHI hai.** Ye sirf encoding hai. Jiske paas `kubectl get secret` ka access hai wo decode kar lega. Secret YAML kabhi git me commit mat karo. `.gitignore` me daalo, ya production me *sealed-secrets* / *external-secrets* use karo.

### Step 1 — base64 values banao

```bash
# Koi bhi string base64 me
echo -n 'mypassword' | base64
# → bXlwYXNzd29yZA==

echo -n 'mongodb://localhost:27017/mydb' | base64
# → bW9uZ29kYjovL2xvY2FsaG9zdDoyNzAxNy9teWRi

# Verify karne ke liye decode
echo 'bXlwYXNzd29yZA==' | base64 --decode
```

> Hamesha `echo -n` (bina newline). `-n` ke bina chhupa hua `\n` encode ho jata hai aur password chupchaap kharab ho jata hai.

### Step 2 — secret.yaml likho

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

> 💡 **Tip:** `data` ki jagah **`stringData`** use karo to plain text seedha likh sakte ho (K8s khud encode karta hai). Local me aasan hai, cluster me result same.

```yaml
# k8s/secret.yaml — stringData ke saath (plaintext, local ke liye aasan)
apiVersion: v1
kind: Secret
metadata:
  name: express-secret
type: Opaque
stringData:                  # plain text — k8s khud base64 kar deta hai
  MONGO_URI: "mongodb://localhost:27017/mydb"
  JWT_SECRET: "mysupersecretkey"
  DB_PASSWORD: "mypassword"
```

### CLI se banao (koi YAML file nahi, git me commit ka darr nahi)

```bash
kubectl create secret generic express-secret \
  --from-literal=JWT_SECRET=mysupersecretkey \
  --from-literal=DB_PASSWORD=mypassword

kubectl create secret generic app-env --from-env-file=.env.secret
```

### Step 3 — Apply karo

```bash
kubectl apply -f k8s/secret.yaml

# Verify
kubectl get secrets
kubectl describe secret express-secret   # keys dikhata hai, values NAHI

# Ek value decode karke dekho
kubectl get secret express-secret -o jsonpath='{.data.MONGO_URI}' | base64 --decode
```

### Step 4 — Deployment me env vars ki tarah inject karo

```yaml
spec:
  containers:
    - name: express
      image: express-k8s:latest
      env:
        - name: MONGO_URI          # container ke andar env var ka naam
          valueFrom:
            secretKeyRef:
              name: express-secret # secret ka naam (metadata.name)
              key: MONGO_URI         # secret ke andar ki key
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

### Shortcut: saari keys ek saath

```yaml
envFrom:
  - secretRef:
      name: express-secret     # saari keys same naam ke env vars ban jati hain
```

### Secret ko files ki tarah mount karo (certificates ke liye)

```yaml
volumeMounts:
  - name: secret-vol
    mountPath: /etc/secrets
    readOnly: true
volumes:
  - name: secret-vol
    secret:
      secretName: express-secret   # har key → /etc/secrets me ek file
```

### Node.js / Express me access

```js
// Kubernetes secrets ko environment variables ki tarah inject karta hai
const mongoUri   = process.env.MONGO_URI;
const jwtSecret  = process.env.JWT_SECRET;
const dbPassword = process.env.DB_PASSWORD;

// Wahi code local me .env ke saath aur Kubernetes me Secrets ke saath chalta hai
```

### Secret vs ConfigMap: kab kaunsa

| Use case | ConfigMap | Secret |
|---|---|---|
| Database password | ❌ Kabhi nahi | ✅ Haan |
| API base URL | ✅ Haan | Zaroorat nahi |
| JWT secret key | ❌ Kabhi nahi | ✅ Haan |
| `NODE_ENV = production` | ✅ Haan | Zaroorat nahi |
| MongoDB connection string | ❌ agar password hai | ✅ Haan |
| Port number | ✅ Haan | Zaroorat nahi |

### Secret types

| Type | Kis kaam ka |
|---|---|
| `Opaque` | Generic key-value (default) |
| `kubernetes.io/tls` | TLS cert + key (Ingress HTTPS ke liye) |
| `kubernetes.io/dockerconfigjson` | Private registry (ECR, Docker Hub) se pull ke credentials |

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

### Project structure (secret ke saath)

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
    └── secret.yaml       # ← .gitignore me daalo!
```

> ⚠️ **`secret.yaml` hamesha `.gitignore` me.** Base64 me bhi git me commit karna bada security risk hai. `.gitignore` me `k8s/secret.yaml` likho. Teammates ko secrets secure channel (1Password, Bitwarden, private DM) se do, git se kabhi nahi.

### Apply order matters

```bash
# 1. Secret (aur ConfigMap) pehle — jin pods me reference hai unke start hone se pehle
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/configmap.yaml

# 2. Phir deployment (ab pods ko secret mil jayega)
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/ingress.yaml

# Verify: env vars pod me aaye ya nahi
kubectl exec -it <pod-name> -- sh
# Pod ke andar:
echo $MONGO_URI
echo $JWT_SECRET
```

### Galtiyan
- Pod me missing Secret/ConfigMap/key ka reference → pod **`CreateContainerConfigError`** me atakta hai. Secret bana do, pod khud recover ho jata hai.
- Secret change karne par env vars **refresh nahi hote** → `kubectl rollout restart deployment/express-deployment`.

### Yaad rakho
- Secret = sensitive config, base64 sirf encoding hai.
- Secret/ConfigMap **pehle** apply karo, Deployment baad me.
- `secret.yaml` kabhi git me nahi.

---

# PART 6 — RELIABILITY

## 14. Health Probes

> **Ek line me:** **Probes** chhote health checks hain (HTTP / TCP / command) jinse K8s jaanta hai ki container sach me theek hai ya nahi, aur kab restart karna hai ya traffic dena hai.

### Kyun chahiye
Container ka process chal raha hai ka matlab ye nahi ki app theek hai. Ho sakta hai app hang ho gayi, ya abhi DB se connect hi nahi hui. K8s ko ye bahar se dikhta nahi — isliye probes.

### Teen probes

| Probe | Sawaal | Fail hone par |
|---|---|---|
| **startupProbe** | App start ho chuki hai? | Intezaar karta hai; bahut fail ho to kill. Pass hone tak baaki do band. |
| **readinessProbe** | Abhi pod traffic le sakta hai? | Pod Service endpoints se **hata diya jata hai** (restart **nahi**) |
| **livenessProbe** | App atak/mar gayi? | Container **restart** hota hai |

### Express me health endpoints

```js
app.get("/health", (req, res) => res.status(200).json({ status: "ok" }));
```

Readiness me asli dependencies check karo:

```js
app.get("/ready", async (req, res) => {
  const dbOk = mongoose.connection.readyState === 1;   // 1 = connected
  res.status(dbOk ? 200 : 503).json({ db: dbOk });
});
```

### deployment.yaml me probes

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
      failureThreshold: 30       # 30 × 2s = start hone ke liye 60s tak
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

| Field | Matlab |
|---|---|
| `initialDelaySeconds` | Container start ke baad pehla check kab |
| `periodSeconds` | Kitni der me ek baar check |
| `timeoutSeconds` | Response ka kitna intezaar (default 1s) |
| `failureThreshold` | Kitne lagataar fail ke baad action |
| `successThreshold` | Wapas healthy maanne ke liye kitne lagataar success |

### Dusre probe types

```yaml
# TCP — bas port connection accept karta hai ya nahi
livenessProbe:
  tcpSocket:
    port: 3000

# Command — exit code 0 = healthy
livenessProbe:
  exec:
    command: ["cat", "/tmp/healthy"]
```

> ⚠️ **Liveness probe me external dependencies check mat karo.** MongoDB down hua aur liveness fail hui to K8s tumhare **saare pods restart** kar dega, bina wajah. Dependency check **readiness** me, liveness simple rakho (process respond kar raha hai?).

> 💡 Readiness hi **rolling update ko zero-downtime** banata hai — naya pod tabhi traffic pata hai jab uski readiness pass ho.

### Yaad rakho
- Readiness = traffic dena ya nahi. Liveness = restart karna ya nahi. Startup = slow start ko time dena.
- Liveness simple, readiness me DB check.

---

## 15. Rolling Updates aur Rollback

> **Ek line me:** Pod template badalte hi (image, env, resources) Deployment **naya ReplicaSet** banata hai aur pods ko dheere-dheere purane se naye me shift karta hai. Kitni tezi se — ye strategy se control hota hai.

### Strategy settings

```yaml
spec:
  replicas: 4
  revisionHistoryLimit: 5          # rollback ke liye kitne purane ReplicaSets rakhne hain
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1                  # update ke dauran desired se 1 extra pod allowed
      maxUnavailable: 0            # kabhi desired count se neeche nahi → zero downtime
```

| Field | Matlab |
|---|---|
| `maxSurge` | Update ke dauran `replicas` se **upar** kitne extra pods allowed (number ya %) |
| `maxUnavailable` | Update ke dauran kitne pods **down** ho sakte hain (number ya %) |
| `type: Recreate` | Pehle saare purane pods maro, phir naye banao (downtime hota hai — jab do versions saath nahi chal sakte) |

### Rolling update kaisa dikhta hai (replicas: 3, maxSurge 1, maxUnavailable 0)

```
Pehle:  [v1][v1][v1]
Step 1: [v1][v1][v1] + [v2 start ho raha]
Step 2: [v1][v1][v2 ready] → ek v1 maro
Step 3: [v1][v2][v2] + [v2 start ho raha] → ...
Final:  [v2][v2][v2]
```

### Trigger, watch, rollback

```bash
# Image update (ya YAML edit karke apply)
kubectl set image deployment/express-deployment express=express-k8s:v2

# Progress dekho
kubectl rollout status deployment/express-deployment

# History
kubectl rollout history deployment/express-deployment
kubectl rollout history deployment/express-deployment --revision=2

# Rollback
kubectl rollout undo deployment/express-deployment                  # pichhle par
kubectl rollout undo deployment/express-deployment --to-revision=1  # kisi specific par

# Pause / resume (kai changes ek hi rollout me)
kubectl rollout pause deployment/express-deployment
kubectl rollout resume deployment/express-deployment
```

### `:latest` ki jagah version tags use karo

```bash
docker build -t express-k8s:v1 .
docker build -t express-k8s:v2 .
```

`:latest` ke saath YAML me **koi change dikhta hi nahi**, isliye `kubectl apply` kuch nahi karta — tumhe `rollout restart` chalana padta hai. Unique tags me YAML ka tag badalte hi rollout khud trigger hota hai aur rollback ka matlab bhi banta hai.

### Graceful shutdown (chalti requests na tootein)

K8s pehle `SIGTERM` bhejta hai, `terminationGracePeriodSeconds` (default 30s) tak rukta hai, phir `SIGKILL`.

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

### Yaad rakho
- Zero downtime = `maxUnavailable: 0` + readiness probe.
- `rollout undo` se ek command me wapas.
- Versioned tags use karo, `:latest` nahi.

---

## 16. Volumes aur Persistent Storage

> **Ek line me:** Container ka filesystem pod delete/restart par gayab ho jata hai. Jo data bachana hai (MongoDB, uploads) uske liye **Volume** lagao.

### Volume types

| Type | Kab tak rehta hai | Use case |
|---|---|---|
| `emptyDir` | Jab tak **pod** hai | Scratch space, pod ke containers ke beech files share |
| `hostPath` | Node ki **disk** par | Sirf local dev (apni machine ka folder mount). Production me avoid. |
| `configMap` / `secret` | Config files ki tarah | ConfigMap / Secret chapters dekho |
| **PersistentVolumeClaim** | Pod ke **baad bhi** | Database, uploads — asli persistence |

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

### PV, PVC, StorageClass kaise judte hain

```
Pod  →  PersistentVolumeClaim (PVC)  →  PersistentVolume (PV)  →  asli disk
        "mujhe 1Gi chahiye"             "ye lo 1Gi"
                          ↑
          StorageClass ye PV demand par khud bana deti hai (dynamic provisioning)
```

| Object | Kaun banata hai | Matlab |
|---|---|---|
| **PersistentVolume (PV)** | Admin ya StorageClass (automatic) | Cluster me storage ka ek tukda |
| **PersistentVolumeClaim (PVC)** | Tum | Storage ki maang: size + access mode |
| **StorageClass** | Cluster | Storage **kaise** banega ye define karti hai. Docker Desktop me default (`hostpath`) hoti hai |

### PVC example

```yaml
# k8s/mongo-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pvc
spec:
  accessModes:
    - ReadWriteOnce          # ek node read-write me mount kar sakta hai
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl get storageclass
kubectl get pvc
kubectl get pv
```

### Pod me PVC use karna

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

| Mode | Matlab |
|---|---|
| `ReadWriteOnce` (RWO) | Ek node read-write mount kar sakta hai |
| `ReadOnlyMany` (ROX) | Kai nodes, sirf read |
| `ReadWriteMany` (RWX) | Kai nodes read-write (NFS/EFS jaisi special storage chahiye) |

> ⚠️ PVC delete karne se data bhi ja sakta hai (PV ki reclaim policy par depend; dynamic ka default aam taur par `Delete`).

> 💡 Production databases ke liye **managed service** (MongoDB Atlas, RDS, ElastiCache) behtar hai. K8s me DB chalana local seekhne ke liye theek hai, production me operationally mushkil.

### Yaad rakho
- Container ka data pod ke saath jata hai; bachana ho to PVC.
- Pod → PVC → PV → disk. StorageClass PV khud banati hai.

---

# PART 7 — REAL PROJECT

## 17. Poora MERN stack Kubernetes par

> **Ek line me:** Asli app me kai hisse hote hain. Har hissa apna **Deployment + Service** hota hai, aur sab ek dusre ko **service naam** se dhundhte hain. Bahar sirf **Ingress** khula hota hai.

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

### MongoDB (PVC + Deployment + Service ek file me)

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
  replicas: 1                      # ek hi mongod — isko scale MAT karo
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
  name: mongo                      # ← yahi hostname ban jata hai
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

### Cluster ke andar connection strings

```
MONGO_URI = mongodb://mongo:27017/mydb       # "mongo" = Service ka naam
REDIS_URL = redis://redis:6379               # "redis" = Service ka naam
```

Inhe ConfigMap / Secret me rakho — `localhost` **nahi**.

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

### Frontend (React build, nginx se serve)

```dockerfile
# Frontend/dockerfile — multi-stage build
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build            # dist/ (Vite) ya build/ (CRA) banata hai

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
# SPA fallback taaki React Router ke routes refresh par bhi chalein
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

> ⚠️ **Bahut common galti:** React **user ke browser** me chalta hai, cluster ke andar nahi. Wo `http://backend` call nahi kar sakta (ye naam sirf cluster ke andar exist karta hai). Frontend se relative path `/api/...` call karo aur Ingress `/api` ko backend par route karega.

### Ingress: dono ke liye ek entry point

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

nginx **sabse lamba matching path** chunta hai, isliye `/api/...` backend par jata hai aur baaki sab frontend par.

### Sahi order me deploy

```bash
docker build -t express-k8s:latest ./Backend
docker build -t frontend-k8s:latest ./Frontend

kubectl apply -f k8s/namespace.yaml        # agar namespace use kar rahe ho
kubectl apply -f k8s/configmap.yaml -f k8s/secret.yaml
kubectl apply -f k8s/mongo.yaml -f k8s/redis.yaml
kubectl apply -f k8s/backend.yaml -f k8s/frontend.yaml
kubectl apply -f k8s/ingress.yaml

kubectl get all
```

> 💡 Pods kisi bhi order me start hote hain. Backend Mongo se pehle start ho gaya to crash hokar kuch baar restart ho sakta hai — ye normal hai. App me **DB connection retry** rakho aur readiness probe lagao.

### Yaad rakho
- Har hissa = Deployment + Service. Naam hi hostname hai (`mongo`, `redis`, `backend`).
- Frontend browser me chalta hai → `/api` relative path + Ingress.
- Mongo `replicas: 1` hi rakho aur PVC lagao.

---

# PART 8 — OPERATE KARNA

## 18. kubectl daily use

> **Ek line me:** `kubectl <verb> <resource> <name> <flags>` — itna format yaad rakho, baaki commands is pattern ke variations hain.

### Command ki anatomy

```
kubectl  <verb>   <resource>   <name>        <flags>
kubectl  get      pods                        -n dev -o wide
kubectl  delete   deployment   express-deployment
```

### Common verbs

| Verb | Kya karta hai |
|---|---|
| `get` | Resources ki list |
| `describe` | Detail + events |
| `apply` / `create` | File se banao ya update |
| `delete` | Resource hatao |
| `logs` | Container output |
| `exec` | Container me command chalao |
| `edit` | Live resource ko editor me kholo |
| `scale` | Replicas badlo |
| `rollout` | Deployment rollouts manage |
| `port-forward` | Local port ko pod/service se jodo |
| `top` | CPU/memory usage (metrics-server chahiye) |
| `explain` | Kisi bhi field ka documentation |

### Output formats

```bash
kubectl get pods -o wide                 # extra columns (IP, node)
kubectl get pods -o yaml                 # poori YAML
kubectl get pod <name> -o json
kubectl get pods -o name                 # sirf naam
kubectl get pods -w                      # live watch
kubectl get pods --sort-by=.metadata.creationTimestamp
kubectl get svc express-service -o jsonpath='{.spec.clusterIP}'
```

### Port-forward: Ingress ke bina kuch bhi kholo

```bash
kubectl port-forward pod/<pod-name> 8080:3000          # localhost:8080 → pod:3000
kubectl port-forward svc/express-service 8080:80       # localhost:8080 → service:80
kubectl port-forward svc/mongo 27017:27017             # MongoDB Compass ko localhost:27017 se jodo
```

Service ko seedha debug karne ya cluster ke Mongo ko Compass se kholne ke liye bahut kaam ka. Band karne ke liye `Ctrl+C`.

### Pod se files copy karna

```bash
kubectl cp <pod-name>:/app/logs/app.log ./app.log
kubectl cp ./config.json <pod-name>:/app/config.json
```

### Temporary debug pod

```bash
# curl/nslookup wala throwaway pod — exit karte hi delete
kubectl run debug --rm -it --image=busybox:1.36 -- sh
# andar:
nslookup express-service
wget -qO- http://express-service/health

# curl chahiye to
kubectl run curl --rm -it --image=curlimages/curl -- sh
```

### Live resource edit

```bash
kubectl edit deployment express-deployment
```

> Is tarah ke changes tumhari YAML files me **nahi** jaate — YAML bhi update karo, nahi to agla `apply` unhe overwrite kar dega.

### Shortcuts

```bash
# Chhote resource naam
kubectl get po          # pods
kubectl get svc         # services
kubectl get deploy      # deployments
kubectl get rs          # replicasets
kubectl get ing         # ingress
kubectl get cm          # configmaps
kubectl get ns          # namespaces
kubectl get hpa         # horizontal pod autoscalers
kubectl get pvc         # persistent volume claims

# kubectl ke liye alias k (~/.zshrc ya ~/.bashrc me)
alias k=kubectl
```

### Yaad rakho
- Verb + resource + name pattern.
- `-o wide`, `-o yaml`, `-w` sabse zyada kaam aate hain.
- `port-forward` aur `kubectl run --rm -it` debugging ke dost.

---

## 19. Debugging aur Troubleshooting

> **Ek line me:** Debugging ka ek hi rasta hai: **upar se neeche** chalo — Pods → Endpoints → Service → Ingress. Jahan atke wahin problem hai.

### Systematic flowchart

```
App reachable nahi?
│
├─ 1. kubectl get pods
│     ├─ Pending            → describe pod (resources? PVC? image?)
│     ├─ ImagePullBackOff   → image naam/tag? local me bana hai? pullPolicy?
│     ├─ CrashLoopBackOff   → kubectl logs --previous (app error?)
│     ├─ Running 0/1        → readiness probe fail → describe pod
│     └─ Running 1/1 ✅     → step 2
│
├─ 2. kubectl get endpoints <service>
│     ├─ <none>             → Service selector ≠ Pod labels
│     └─ IPs dikh rahe ✅   → step 3
│
├─ 3. kubectl port-forward svc/<service> 8080:80 → curl localhost:8080
│     ├─ fail               → targetPort galat / app galat port ya 127.0.0.1 par sun rahi
│     └─ chal gaya ✅       → Service theek hai, problem Ingress me → step 4
│
└─ 4. kubectl describe ingress + ingress controller logs
      ├─ 404                → host/path mismatch, galat ingressClassName
      ├─ 503                → backend service naam/port galat
      └─ ADDRESS nahi       → controller install nahi / chal nahi raha
```

### Common errors aur kya karein

| Error | Wajah | Debug command |
|---|---|---|
| **ImagePullBackOff / ErrImagePull** | Docker image pull nahi ho rahi. Aksar local image bani nahi ya naam/tag galat. | `kubectl describe pod <name>` → Events section |
| **CrashLoopBackOff** | Container baar-baar crash. App me startup error. | `kubectl logs <pod-name> --previous` |
| **CreateContainerConfigError** | Reference kiya Secret/ConfigMap/key exist nahi karta. | `kubectl describe pod <name>` → Events; missing secret banao |
| **OOMKilled** | Container ne memory limit cross ki. | `kubectl describe pod <name>` → Last State: OOMKilled; `limits.memory` badhao ya leak theek karo |
| **503 Service Unavailable** | Ingress controller chal raha hai par pods nahi mile. Label mismatch ya pods ready nahi. | `kubectl get endpoints <service>` → `<none>` to labels match nahi |
| **502 Bad Gateway** | Ingress pod tak pahuncha par app crash/refuse, ya targetPort galat. | `targetPort`, pod logs, ingress controller logs dekho |
| **404 Not Found** | nginx chal raha hai par us host/path ka rule nahi. | `kubectl describe ingress <name>` → rules dekho |
| **Pending (pod)** | Kisi node par pod ke liye CPU/memory nahi. | `kubectl describe pod <name>` → Events |
| **Pending (PVC)** | StorageClass nahi / volume provision nahi ho paya. | `kubectl describe pvc <name>` |
| **ECONNREFUSED** | Pod dusri service ko `localhost` se call kar raha hai. | `http://service-name` use karo, `http://localhost:PORT` nahi |
| **Evicted** | Node ke resources kam pad gaye. | `kubectl describe pod <name>`; sahi requests/limits set karo |
| **Running par 0/1 READY** | Readiness probe fail ho rahi. | `kubectl describe pod <name>` → probe failure events |

### Essential debug commands

```bash
# Kya chal raha hai
kubectl get pods
kubectl get services
kubectl get ingress
kubectl get all                              # sab ek saath
kubectl get events --sort-by=.lastTimestamp  # haal ke cluster events

# Gehrai se dekho
kubectl describe pod <pod-name>              # events + poora config
kubectl describe ingress <ingress-name>      # routing rules + backend IPs
kubectl describe service <service-name>      # selector + endpoints

# Logs
kubectl logs <pod-name>                      # app output
kubectl logs <pod-name> -f                   # live follow
kubectl logs <pod-name> --previous           # crash hue container ke logs
kubectl logs <pod-name> -c <container>       # multi-container pod
kubectl logs -l app=express --tail=50        # label wale saare pods ke logs
kubectl logs deploy/express-deployment       # deployment ke pod ke logs

# Network debug
kubectl get endpoints <service-name>         # service pool ke pod IPs
kubectl exec -it <pod-name> -- sh            # pod ke andar shell
# Pod ke andar: dusri service test
wget -qO- http://some-service/api/data
```

### Quick sanity checklist (label aur port)

- Deployment `selector.matchLabels` == `template.metadata.labels`
- Service `selector` == Pod labels
- Service `targetPort` == container ka asli listening port
- Ingress backend `service.name` aur `port.number` == Service ka naam aur `port`
- Image local me hai: `docker images | grep express-k8s`

### Yaad rakho
- Order: pods → endpoints → port-forward → ingress.
- `describe` ka **Events** section sabse zyada batata hai.
- Crash ho to `logs --previous`.

---

## 20. Autoscaling (HPA)

> **Ek line me:** **HPA** pods ki CPU usage dekhta rehta hai. Threshold cross ho to pods badhata hai, traffic kam ho to ghatata hai. Iske liye **metrics-server** chahiye, jo Docker Desktop me by default install **nahi** hota.

### HPA kaise soch ta hai

```
Traffic badha → pod ki CPU 50% se upar
    ↓
HPA naya pod add karta hai (maxReplicas: 5 tak)
    ↓
Requests zyada pods me baat gayi
    ↓
Har pod ki CPU kam — load baant gaya
    ↓
Traffic ghata → CPU kam
    ↓
HPA extra pods hata deta hai (minReplicas: 1 tak)
```

### Step 0 — Metrics Server install karo (zaroori)

> ⚠️ **Metrics-server ke bina HPA kaam nahi karega.** `kubectl top pods` "Metrics API not available" dega aur HPA CPU ke jagah `<unknown>/50%` dikhayega.

**Option A: Official manifest + kubelet insecure TLS patch (Docker Desktop ke liye recommended)**

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Docker Desktop self-signed cert use karta hai — TLS verify skip karne ka patch:
kubectl patch deployment metrics-server -n kube-system \
  --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

# Ready hone ka wait (~30 seconds)
kubectl rollout status deployment/metrics-server -n kube-system

# Verify — CPU aur Memory columns dikhne chahiye
kubectl top nodes
kubectl top pods
```

**Option B: Helm se (agar Helm hai)**

```bash
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm repo update
helm upgrade --install metrics-server metrics-server/metrics-server \
  --namespace kube-system \
  --set args={--kubelet-insecure-tls}
```

> 💡 Metrics-server `kube-system` namespace me master/control-plane node par pod ki tarah chalta hai. Check: `kubectl get pods -n kube-system | grep metrics` → `metrics-server-xxxx   1/1   Running`.

> `--kubelet-insecure-tls` sirf **local development** ke liye hai. Asli cluster par kabhi nahi.

### HPA banao (command se)

```bash
kubectl autoscale deployment express-deployment \
  --min=1 \
  --max=5 \
  --cpu-percent=50
```

### Ya YAML se (recommended)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: express-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: express-deployment    # deployment ke naam se match
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50   # CPU > 50% par scale
```

> ⚠️ Deployment me **`resources.requests` set hona zaroori hai.** HPA utilization = *current CPU / requested CPU × 100* se nikalta hai. Requests ke bina baseline nahi, to HPA scale nahi karega.

### Memory ko second metric banao (optional)

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

HPA har metric ke liye desired replicas nikalta hai aur **sabse zyada** wala chunta hai.

### Formula

```
desiredReplicas = ceil( currentReplicas × currentUtilization / targetUtilization )

Example: 2 pods, CPU 100%, target 50%
         → ceil(2 × 100 / 50) = 4 pods
```

### Scale-down jaan-boojhkar dheere hota hai

By default HPA ~**5 minute** kam usage dekhne ke baad hi scale down karta hai (stabilization window), taaki baar-baar upar-neeche na ho. Tune karna ho:

```yaml
spec:
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 60
```

### Load generate karke test karo

```bash
# Terminal 1 — HPA watch
kubectl get hpa -w

# Terminal 2 — cluster ke andar se service par requests ki bauchhar
kubectl run load-gen --rm -it --image=busybox:1.36 -- \
  /bin/sh -c "while true; do wget -q -O- http://express-service/; done"

# Terminal 3 — pods aate hue dekho
kubectl get pods -w
```

Load generator band karo (`Ctrl+C`) aur cooldown ke baad replicas ghatte dekho.

```bash
kubectl get hpa              # current CPU%, desired vs actual replicas
kubectl top pods             # har pod ki live CPU aur memory
kubectl describe hpa express-hpa   # events + kyun scale hua
```

> HPA use kar rahe ho to Deployment YAML se **`replicas:` hata do** (warna har `kubectl apply` count reset kar dega).

### Yaad rakho
- HPA = metrics-server + `requests` + HPA YAML, teeno chahiye.
- Scale up tez, scale down dheere.

---

# PART 9 — EXTRAS

## 21. Dusre Workload types

Deployment **stateless** apps ke liye hai. Baaki kaamon ke liye alag controllers hain.

| Kind | Kis kaam ke liye | Khaas baat |
|---|---|---|
| **Deployment** | Stateless apps (API, frontend) | Interchangeable pods, rolling updates |
| **StatefulSet** | Database, Kafka — jahan stable identity/storage chahiye | Pods ke naam `db-0`, `db-1`...; har ek ka apna PVC; ordered start/stop |
| **DaemonSet** | Har **node par ek pod** (log collector, monitoring agent) | Naya node judte hi pod auto-add |
| **Job** | Ek baar chalne wala kaam (migration, seed script) | Success tak retry, phir ruk jata hai |
| **CronJob** | Scheduled Jobs | Cron syntax |

### Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-seed
spec:
  backoffLimit: 3                  # 3 baar tak retry
  template:
    spec:
      restartPolicy: Never         # Job me Never ya OnFailure hi chalta hai
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
  schedule: "0 2 * * *"            # roz raat 02:00 baje
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
kubectl create job manual-run --from=cronjob/nightly-cleanup   # CronJob abhi trigger karo
```

### StatefulSet (chhota example)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongo
spec:
  serviceName: mongo-headless      # headless Service chahiye (clusterIP: None)
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
  volumeClaimTemplates:            # har replica ko apna PVC automatically
    - metadata:
        name: data
      spec:
        accessModes: [ReadWriteOnce]
        resources:
          requests:
            storage: 1Gi
```

---

## 22. Helm aur Kustomize

> **Ek line me:** Dev/staging/prod ke liye YAML copy-paste karna jaldi hi gandagi ban jata hai. Helm aur Kustomize isi ko sambhalte hain.

| Tool | Idea |
|---|---|
| **Helm** | Package manager. YAML **templates** + `values.yaml`. Ready-made charts (nginx-ingress, mongodb, redis, prometheus...) install kar sakte ho. |
| **Kustomize** | kubectl me built-in. Ek **base** YAML rakho aur har environment ke liye chhote **overlays** (patches) lagao. Templating nahi. |

### Helm basics

```bash
# Helm install (Mac)
brew install helm

helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo redis

helm install my-redis bitnami/redis                      # chart install
helm install my-redis bitnami/redis -f my-values.yaml    # custom values ke saath
helm install my-redis bitnami/redis --set auth.enabled=false

helm list                                                # installed releases
helm upgrade my-redis bitnami/redis -f my-values.yaml
helm rollback my-redis 1
helm uninstall my-redis
```

nginx ingress Helm se install karna (raw manifest ki jagah):

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
kubectl kustomize k8s/overlays/prod      # final YAML ka preview
kubectl apply -k k8s/overlays/prod       # apply
```

---

## 23. Useful Tools

| Tool | Kya karta hai |
|---|---|
| **Docker Desktop Containers view** | Kya chal raha hai ek nazar me |
| **k9s** | Terminal UI — keyboard se pods, logs, shell. `brew install k9s` |
| **Lens / OpenLens** | Cluster ke liye desktop GUI |
| **kubectx / kubens** | Context / namespace jaldi badlo: `kubectx docker-desktop`, `kubens dev` |
| **stern** | Kai pods ke logs ek saath tail: `stern express` |
| **kubectl neat** | `-o yaml` output se faltu fields hatata hai |
| **Kubernetes Dashboard** | Official web UI (alag install) |
| **VS Code Kubernetes extension** | YAML validation, cluster explorer |
| **kubeconform / kubeval** | CI me apply se pehle YAML validate |
| **Skaffold / Tilt** | Code change par auto rebuild + redeploy (inner-loop dev) |

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

## 24. Cleanup aur Reset

```bash
# YAML se banaye resources delete
kubectl delete -f k8s/

# Type/naam se delete
kubectl delete deployment express-deployment
kubectl delete svc express-service
kubectl delete hpa express-hpa

# Current namespace ka sab kuch delete (savdhan!)
kubectl delete all --all

# Poora namespace aur uska sab kuch
kubectl delete namespace dev

# Ingress controller hatao
kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.1/deploy/static/provider/cloud/deploy.yaml

# Disk bachane ke liye unused Docker images/containers saaf
docker system prune -a
```

> `kubectl delete all --all` ConfigMaps, Secrets, PVCs ya Ingress ko **delete nahi karta** — unhe alag se delete karo.

### Local cluster ka full reset

Docker Desktop → Settings → Kubernetes → **Reset Kubernetes Cluster**. Har resource saaf hokar fresh single-node cluster milta hai (ingress-nginx aur metrics-server dobara install karne padenge).

### RAM bachane ke liye Kubernetes pause

Docker Desktop → Settings → Kubernetes → **Enable Kubernetes** untick → Apply. Baad me dobara enable karo (YAML files safe, par cluster state chali jati hai).

---

## 25. Local se EKS tak

Wahi YAML Docker Desktop se AWS EKS par le jao to kya badalta hai:

| Topic | Local (Docker Desktop) | EKS (AWS) |
|---|---|---|
| **Images** | Local Docker store, `imagePullPolicy: IfNotPresent` | **ECR** me push, poora image URL, `imagePullPolicy: Always` |
| **Image naam** | `express-k8s:latest` | `<account>.dkr.ecr.<region>.amazonaws.com/express-k8s:v1` |
| **Nodes** | 1 node | Multiple nodes (managed node group / Fargate) |
| **Ingress controller** | nginx, LoadBalancer = `localhost` | nginx ya **AWS Load Balancer Controller (ALB)**; asli public DNS |
| **Service type LoadBalancer** | localhost se map | Asli AWS ELB banta hai (paise lagte hain) |
| **Storage** | hostpath StorageClass | RWO ke liye EBS (`gp3`), RWX ke liye EFS |
| **Secrets** | Plain k8s Secrets | AWS Secrets Manager + External Secrets / CSI driver |
| **Domain / TLS** | `/etc/hosts`, self-signed | Route 53 + ACM / cert-manager |
| **Metrics-server** | Manually, `--kubelet-insecure-tls` ke saath | Normal install (insecure flag nahi) |
| **Context** | `docker-desktop` | `aws eks update-kubeconfig` / eksctl se add hota hai |

```bash
# Context switch
kubectl config get-contexts
kubectl config use-context docker-desktop
kubectl config use-context <eks-context-name>
```

> `apply` / `delete` se pehle hamesha `kubectl config current-context`.

---

## 26. Best Practices Checklist

**Images aur containers**
- [ ] Specific image tags (`v1.2.3`), sirf `:latest` nahi
- [ ] Chhote base images (`node:20-alpine`), `.dockerignore` maujood
- [ ] App `0.0.0.0` par sunti hai aur `SIGTERM` gracefully handle karti hai
- [ ] Jahan ho sake non-root user

**Deployments**
- [ ] Har container par `resources.requests` **aur** `limits`
- [ ] `readinessProbe` (aur simple `livenessProbe`) set
- [ ] Availability chahiye to `replicas >= 2`
- [ ] Labels consistent: `selector` == `template.labels` == Service `selector`
- [ ] Rolling update tune (zero downtime ke liye `maxUnavailable: 0`)

**Config aur secrets**
- [ ] Non-secret config **ConfigMap** me, sensitive **Secret** me
- [ ] Secret YAML `.gitignore` me
- [ ] Connection strings/passwords image ya git wali YAML me hardcode nahi
- [ ] Env-based ConfigMap/Secret badalne ke baad pods restart

**Networking**
- [ ] Services **service names** se baat karein, kabhi `localhost` nahi
- [ ] Bahar ke liye ek Ingress, single entry point
- [ ] `ingressClassName` explicitly set

**Operations**
- [ ] Destructive commands se pehle `kubectl config current-context`
- [ ] Environments / projects ke liye namespaces
- [ ] YAML git me (ek resource ek file, `k8s/` folder me)
- [ ] `kubectl apply -f` (declarative), ad-hoc `kubectl edit` nahi
- [ ] Debug order: pods → endpoints → port-forward → ingress

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

*  Kubernetes Local Setup Notes · Docker Desktop · MERN Stack · nginx Ingress*
