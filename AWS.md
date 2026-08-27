# ☁️ AWS ECS Deployment — Complete Guide

> Dono PDFs (AWS Core Concepts + AWS ECS Deployment — Sheryians Notes) combine karke ek final guide bana di hai. Jo topics dono mein cover nahi the (CloudWatch details, Fargate vs EC2, full step-by-step deployment flow, networking deep-dive, autoscaling, CI/CD) wo bhi add kiye hain. End mein interview questions hain.

---

## 📚 Table of Contents

1. [What is AWS?](#1-what-is-aws)
2. [VPC — Virtual Private Cloud](#2-vpc--virtual-private-cloud)
3. [Subnets (Public vs Private)](#3-subnets-public-vs-private)
4. [Security Group](#4-security-group)
5. [Target Group](#5-target-group)
6. [Application Load Balancer (ALB)](#6-application-load-balancer-alb)
7. [IAM — Users, Roles & Policies](#7-iam--users-roles--policies)
7.5. [Practical Steps — Root Se Bachna, IAM User + Credentials Banana](#75-practical-steps--root-se-bachna-iam-user--credentials-banana)
8. [ECR — Elastic Container Registry](#8-ecr--elastic-container-registry)
9. [ECS — Elastic Container Service](#9-ecs--elastic-container-service)
10. [ECS on EC2 vs Fargate](#10-ecs-on-ec2-vs-fargate)
11. [Task Role vs Task Execution Role](#11-task-role-vs-task-execution-role)
12. [CloudWatch — Logs & Monitoring](#12-cloudwatch--logs--monitoring)
13. [Auto Scaling (ECS Service + EC2)](#13-auto-scaling-ecs-service--ec2)
14. [Why Not Use Root User Credentials](#14-why-not-use-root-user-credentials)
15. [Full Deployment — Step by Step](#15-full-deployment--step-by-step)
16. [CI/CD Integration (GitHub Actions → ECR → ECS)](#16-cicd-integration-github-actions--ecr--ecs)
17. [Concept Diagrams](#17-concept-diagrams)
18. [Common Mistakes & Debugging](#18-common-mistakes--debugging)
19. [Interview Questions (AWS + ECS)](#19-interview-questions-aws--ecs)
20. [Final Takeaway](#20-final-takeaway)

---

## 1. What is AWS?

**AWS (Amazon Web Services)** ek cloud computing platform hai jo on-demand compute, storage, networking, database, aur container/orchestration services deta hai — bina tujhe khud physical servers manage kiye.

**Key points:**
- Pay-as-you-go model — jitna use karo utna hi paisa
- Global infrastructure — Regions (e.g. `ap-south-1` Mumbai) aur har Region ke andar multiple Availability Zones (AZs) — fault tolerance ke liye
- Is guide mein hum specifically dekhenge: **VPC → Security → Load Balancing → IAM → ECR (image storage) → ECS (container orchestration)** — ye poora stack milke ek Dockerized backend/frontend ko production mein deploy karta hai

---

## 2. VPC — Virtual Private Cloud

**Definition:** VPC ek logically isolated network hai AWS ke andar jaha tu apne resources (EC2, databases, ECS services) launch karta hai.

**Key points:**
- Tu khud IP address range define karta hai (CIDR block, e.g. `10.0.0.0/16`)
- Networking pe full control — IP, routing, security
- Basically ek private data center jaisa hai AWS ke andar

**Diagram:**
```
AWS Cloud
   │
   └── VPC (10.0.0.0/16)
          │
          ├── Subnet A
          └── Subnet B
```

---

## 3. Subnets (Public vs Private)

**Definition:** Subnet ek chhota network hai VPC ke andar.

**Types:**
- **Public Subnet:** Internet Gateway ke through internet access hai
- **Private Subnet:** Direct internet access nahi hai

**Key points:**
- Har subnet ek hi Availability Zone se belong karta hai
- Resources ko organize aur secure karne ke liye use hota hai

**Diagram:**
```
VPC
 │
 ├── Public Subnet (10.0.1.0/24)
 │       └── EC2 (with internet)
 │
 └── Private Subnet (10.0.2.0/24)
         └── Database (no internet)
```

**Common pattern (production):** ALB public subnet mein rehta hai (internet-facing), ECS tasks aur database private subnet mein (internet se directly unreachable — sirf ALB/NAT Gateway ke through traffic aata hai).

---

## 4. Security Group

**Definition:** Security Group ek virtual firewall hai jo AWS resources pe attach hota hai.

**Key points:**
- Inbound aur outbound traffic control karta hai
- Instance level pe kaam karta hai (per-resource, not per-subnet)
- **Stateful:** Agar inbound allow hai, return traffic automatically allow ho jaata hai (outbound rule likhne ki zaroorat nahi)

**Example rules:**
```
Inbound Rules:
  Port 80  (HTTP)  → 0.0.0.0/0        (allow from anywhere)
  Port 443 (HTTPS) → 0.0.0.0/0
  Port 5000 (backend) → sirf ALB Security Group se allow (not public)

Outbound Rules:
  All traffic → 0.0.0.0/0  (default, usually left open)
```

**ECS-specific SG pattern:**
- **ALB Security Group:** allow port 80/443 from anywhere (`0.0.0.0/0`)
- **ECS Service Security Group:** allow traffic only from ALB's Security Group (not the whole internet) — chaining SGs is a common security best practice

---

## 5. Target Group

**Definition:** Target Group registered targets (EC2 instances, containers, IP addresses) ko route karne ke liye use hota hai.

**Key points:**
- Load Balancers isko use karte hain traffic route karne ke liye
- Targets pe **health checks** perform karta hai
- Sirf **healthy** targets ko hi traffic route karta hai — unhealthy target automatically traffic se hat jaata hai

**Typical config:**
```
Target Type:     IP address (required for ECS Fargate)
Protocol:        HTTP
Port:            80 (container port jispe app sun raha hai)
Health check path: /health  (ya / agar dedicated endpoint nahi hai)
```
ECS khud automatically containers ko is Target Group mein register/deregister karta hai jab task start/stop hota hai — manual register karne ki zaroorat nahi.

---

## 6. Application Load Balancer (ALB)

**Definition:** ALB incoming HTTP/HTTPS traffic ko multiple targets mein distribute karta hai.

**Key points:**
- Layer 7 (Application Layer) pe kaam karta hai — matlab ye URL path, host header dekh ke route kar sakta hai
- **Path-based routing** support karta hai (e.g. `/api/*` → backend service, `/` → frontend service)
- **Host-based routing** bhi support karta hai (e.g. `api.myapp.com` vs `myapp.com`)
- Availability aur scalability improve karta hai — ek EC2/container down ho jaaye toh baaki traffic handle karte rehte hain

**Diagram:**
```
Client Request
      │
      ▼
     ALB
    /    \
   ▼      ▼
  ECS    ECS
 Task1   Task2
```

**ALB vs NLB (extra context not in original notes):**

| | ALB | NLB (Network Load Balancer) |
|---|---|---|
| Layer | 7 (Application) | 4 (Transport) |
| Routing | Path/host-based | IP/port-based only |
| Use case | Web apps, APIs, microservices | High-performance, low-latency (gaming, TCP-heavy) |
| ECS ke saath common? | ✅ Most common choice | Less common, used for special cases |

---

## 7. IAM — Users, Roles & Policies

### IAM User
**Definition:** Ek individual identity jiske paas AWS access ke liye permissions hain.
- Username + credentials (password/access keys) hoti hain
- Insaan (developer/admin) use karta hai

### IAM Role
**Definition:** Ek identity jo services ya users assume kar sakte hain.
- Koi permanent credentials nahi hote
- Temporary access milta hai (auto-rotating security tokens)
- AWS services use karte hain (EC2, ECS, Lambda)

### IAM Policy (missing from original notes — added)
**Definition:** Ek JSON document jo define karta hai ki kya allowed hai aur kya nahi — Users aur Roles dono ko policies attach hoti hain.
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
```
**Key principle — Least Privilege:** Kabhi bhi `"Action": "*"` (sab kuch allow) mat do — sirf wahi permissions do jo actual kaam ke liye chahiye.

---

## 7.5 Practical Steps — Root Se Bachna, IAM User + Credentials Banana

*(Ye sabse zaroori practical step hai jo original notes mein bilkul nahi tha — sirf "root use mat karo" bola tha, "toh phir kya karo" nahi bataya. Yaha exact console steps hain.)*

### Step 1 — Account bana lene ke baad SIRF ye kaam root se karo, phir root ko touch mat karo
Jab tu AWS account banata hai (email + password se), wo **root user** hota hai. Root se bas ye 2 kaam karo aur uske baad daily use ke liye kabhi login mat karo root se:
1. **MFA enable karo root pe** — IAM console → Security credentials → Assign MFA device (Google Authenticator app se QR scan karke)
2. **Ek IAM user bana lo apne liye** (neeche steps)

### Step 2 — IAM User banana (root ki jagah ye use karega)
1. AWS Console → search bar mein **"IAM"** type karo → IAM dashboard khulega
2. Left sidebar mein **Users** → **Create user** button click karo
3. **User name** do (e.g. `anand-admin` ya `deploy-user`)
4. **"Provide user access to the AWS Management Console"** checkbox — agar console se login bhi karna hai toh check karo, password set karo
5. **Next** click karo — ab permissions dene ka step aayega

### Step 3 — Permissions dena (ye asli important step hai)
Yaha 3 options milte hain:

| Option | Kab use karo |
|---|---|
| **Add user to group** | Best practice — pehle ek Group banao (e.g. `Developers`), usme policies attach karo, phir user ko group mein daal do |
| **Copy permissions from existing user** | Agar koi already-configured user hai jaisa access chahiye |
| **Attach policies directly** | Quick setup, chhote projects ke liye theek hai |

**Beginner/learning ke liye simplest tareeka — directly policies attach karo:**
1. **"Attach policies directly"** select karo
2. Search box mein type karo aur ye policies dhoondh ke check karo (jitni zaroorat ho utni hi, sab mat de do):
   - `AmazonEC2ContainerRegistryFullAccess` (ECR ke liye)
   - `AmazonECS_FullAccess` (ECS ke liye)
   - `AmazonVPCFullAccess` (VPC/Subnet/Security Group ke liye)
   - `ElasticLoadBalancingFullAccess` (ALB/Target Group ke liye)
   - `CloudWatchLogsFullAccess` (logs dekhne ke liye)
   - `IAMFullAccess` (agar tujhe khud Task Roles bhi banane hain) — **caution: ye powerful hai, sirf trusted setup mein do**
3. **Next** → **Create user**

> **Production mein better tareeka:** Har service ke liye alag chhoti custom policy banao (sirf jo actions chahiye), full-access managed policies avoid karo. Learning/personal project ke liye upar wala tareeka chalta hai.

### Step 4 — Programmatic Access (Access Key) generate karna
Console login alag hai, **CLI/Docker se AWS ko commands bhejne** ke liye alag "Access Key" chahiye:
1. Newly created user pe click karo (IAM → Users → apna user)
2. **Security credentials** tab
3. **Access keys** section → **Create access key**
4. Use case select karo → **"Command Line Interface (CLI)"** → confirm karo checkbox
5. **Create access key**
6. Yaha do cheezein milengi — **ye ek hi baar dikhengi, dobara nahi milengi:**
   ```
   Access Key ID:     AKIAxxxxxxxxxxxxxxxx
   Secret Access Key: wJalrxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   ```
   **Turant `.csv` download kar lo ya kisi safe jagah save kar lo** — Secret Access Key sirf ek baar dikhta hai.

### Step 5 — AWS CLI install + configure karna (apne local machine/terminal pe)
```bash
# AWS CLI install (agar nahi hai)
# Mac: brew install awscli
# Windows: installer download karo aws.amazon.com/cli se
# Linux: sudo apt install awscli   (ya official installer use karo)

# Verify install
aws --version

# Ab apni credentials configure karo
aws configure
```
Ye 4 cheezein poochega:
```
AWS Access Key ID [None]:     <paste your Access Key ID>
AWS Secret Access Key [None]: <paste your Secret Access Key>
Default region name [None]:   ap-south-1        (Mumbai, ya jo tera nearest region ho)
Default output format [None]: json
```
Bas — ab tera terminal is IAM user ki identity se AWS se baat kar sakta hai (root credentials kahi bhi use nahi hui).

### Step 6 — Verify karo ki setup sahi hai
```bash
aws sts get-caller-identity
```
Output kuch aisa aayega:
```json
{
  "UserId": "AIDAxxxxxxxxxxxxxx",
  "Account": "123456789012",
  "Arn": "arn:aws:iam::123456789012:user/anand-admin"
}
```
Agar ye "root" ki jagah tera IAM user ka ARN dikha raha hai — matlab setup sahi hai, ab tu safely ECR/ECS commands chala sakta hai.

---

## 8. ECR — Elastic Container Registry

**Definition:** ECR ek managed Docker container registry hai (Docker Hub ka AWS-native alternative).

**Key points:**
- Container images store karta hai
- ECS ke sath integrated hai
- Secure aur scalable

### Step-by-step: Apni local image ko AWS tak kaise pahunchayein

*(Ye "docker push" jaisa hi hai jo Docker Hub ke liye karte the — bas destination AWS ka ECR hai. Har step ka matlab bhi neeche hai.)*

**Step 1 — Apna Account ID pata karo**
```bash
aws sts get-caller-identity --query "Account" --output text
```
Output ek 12-digit number dega (e.g. `123456789012`) — ye tera AWS Account ID hai, aage sab commands mein use hoga.

**Step 2 — ECR mein ek Repository banao** (ye ek naam-based folder hai jaha tera image store hoga)
```bash
aws ecr create-repository --repository-name my-app --region ap-south-1
```
Repository ka naam usually project/app ke naam se milta hai. Ye console se bhi ho sakta hai: ECR → **Create repository** → naam do → Create.

**Step 3 — Docker ko ECR ke sath authenticate karo**
```bash
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.ap-south-1.amazonaws.com
```
**Ye kya kar raha hai:** `aws ecr get-login-password` ek temporary token generate karta hai (12 ghante valid), jo pipe (`|`) ke through seedha `docker login` ko diya jaata hai. Isse tera Docker CLI ab ECR registry ke sath authenticated ho jaata hai — yaha wahi IAM user credentials use ho rahi hain jo tune Section 7.5 mein `aws configure` se set ki thi.

**Step 4 — Apni local image ko build karo** (agar pehle se nahi bani)
```bash
docker build -t my-app .
```

**Step 5 — Image ko ECR ke registry naam se "tag" karo**
```bash
docker tag my-app:latest 123456789012.dkr.ecr.ap-south-1.amazonaws.com/my-app:latest
```
**Ye kyun zaroori hai:** Docker ko batana padta hai ki ye image kaha jaani hai. Local naam (`my-app:latest`) sirf tere machine ke liye hai — ECR ka full registry path alag banana padta hai (`<account-id>.dkr.ecr.<region>.amazonaws.com/<repo-name>:<tag>`), tabhi `docker push` ko pata chalega kaha bhejna hai.

**Step 6 — Push karo**
```bash
docker push 123456789012.dkr.ecr.ap-south-1.amazonaws.com/my-app:latest
```
Ye tera image ko layer-by-layer AWS ECR tak upload kar dega — bilkul waisa hi jaisa Docker Hub pe `docker push` karte waqt hota hai.

**Step 7 — Verify karo image chali gayi**
```bash
aws ecr describe-images --repository-name my-app --region ap-south-1
```
Ya console mein: ECR → apna repository open karo → image tag aur push date dikhega.

**Quick reference (sab commands ek jagah):**
```bash
# 1. Account ID nikaalo
aws sts get-caller-identity --query "Account" --output text

# 2. Repository banao
aws ecr create-repository --repository-name my-app --region <region>

# 3. Docker ko ECR se authenticate karo
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com

# 4. Image build karo
docker build -t my-app .

# 5. Image tag karo ECR path ke sath
docker tag my-app:latest <account-id>.dkr.ecr.<region>.amazonaws.com/my-app:latest

# 6. Push karo
docker push <account-id>.dkr.ecr.<region>.amazonaws.com/my-app:latest
```

> **Note:** Ye authentication token (Step 3) sirf **12 ghante** valid rehta hai — agla din push karna ho toh Step 3 dobara chalana padega.

---

## 9. ECS — Elastic Container Service

**Definition:** ECS ek container orchestration service hai Docker containers run aur manage karne ke liye.

**Key points:**
- EC2 ya Fargate pe containers run karta hai
- Scaling, deployment, management handle karta hai
- ECR ke sath kaam karta hai images ke liye

**Diagram:**
```
ECR (Image Storage)
      │
      ▼
 ECS Cluster
      │
      └── Task (Container)
```

### Key ECS terminology (missing from original notes — added):

| Term | Meaning |
|---|---|
| **Cluster** | Logical grouping of resources (EC2 instances or Fargate capacity) where tasks run |
| **Task Definition** | Blueprint (JSON/YAML) — kaunsi image, kitna CPU/memory, ports, env vars, roles |
| **Task** | Ek running instance of a Task Definition — jaisa Docker mein "container" hota hai |
| **Service** | Ek "manager" jo desired number of Tasks maintain karta hai — crash ho jaaye toh auto-replace karta hai, ALB se bhi integrate hota hai |

**Sample Task Definition (simplified):**
```json
{
  "family": "my-app-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "backend",
      "image": "<account-id>.dkr.ecr.<region>.amazonaws.com/my-app:latest",
      "portMappings": [{ "containerPort": 5000, "protocol": "tcp" }],
      "environment": [{ "name": "MONGO_URI", "value": "your_atlas_uri" }],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-app",
          "awslogs-region": "ap-south-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

---

## 10. ECS on EC2 vs Fargate

*(This comparison is not covered in the original notes — added since it's a core ECS decision.)*

| | ECS on EC2 | ECS on Fargate |
|---|---|---|
| Server management | Tu khud EC2 instances manage karta hai (patching, scaling) | Serverless — AWS khud manage karta hai underlying servers |
| Control | Zyada control (custom AMIs, instance types) | Kam control, but zero ops overhead |
| Billing | Per EC2 instance (running 24/7 = paisa lagta hai even if idle) | Per task, per second — sirf jitna use utna paisa |
| Best for | Predictable/heavy workloads, cost optimization at scale | Variable workloads, quick setup, small teams, MVPs |
| Setup complexity | Zyada (cluster capacity manage karna padta hai) | Kam — bas Task Definition banao aur run karo |

**Beginner/small project ke liye Fargate hi recommended hai** — koi server manage nahi karna padta.

---

## 11. Task Role vs Task Execution Role

*(Ye ek most-confused topic hai interviews mein — dono notes mein tha, yaha side-by-side clarify kiya)*

### Task Role
**Definition:** Container ke andar chal rahi **application** ko permissions deta hai.
- Tera app agar S3 se file read kare, ya DynamoDB access kare — uske liye Task Role chahiye
- "Ye role batata hai ki tera code AWS ke andar kya kar sakta hai"

### Task Execution Role
**Definition:** ECS agent khud (infrastructure) ko permissions deta hai, tere app ko nahi.
- ECR se image pull karna
- CloudWatch mein logs bhejna
- Secrets Manager se secrets fetch karna (agar use kiya ho)
- "Ye role batata hai ki ECS khud kya operations perform kar sakta hai taaki tera container start ho sake"

**Simple rule of thumb:**
```
Task Execution Role  → "container ko START karne" ke liye chahiye (image pull, logs setup)
Task Role            → container ke ANDAR chal rahe app ko AWS access dene ke liye chahiye
```

---

## 12. CloudWatch — Logs & Monitoring

*(Original ECS PDF mein sirf heading thi, detail add kar raha hoon)*

**Definition:** CloudWatch AWS ka monitoring aur logging service hai — ECS tasks ke logs, metrics, alarms sab yahi manage hota hai.

**Key points:**
- Container ke `stdout`/`stderr` automatically CloudWatch Logs mein jaate hain (agar `awslogs` driver Task Definition mein configure ho)
- **Log Groups** — har service/app ka apna log group (e.g. `/ecs/my-app`)
- **Metrics** — CPU utilization, memory utilization, request count (ALB se) — auto track hote hain
- **Alarms** — threshold cross hone pe notification/action trigger karte hain (e.g. CPU > 80% → scale out)

**Debugging flow:**
```bash
# Container crash ho gaya? Pehle CloudWatch Logs check karo
aws logs tail /ecs/my-app --follow
```

---

## 13. Auto Scaling (ECS Service + EC2)

*(Missing from original notes — added, important production concept)*

**ECS Service Auto Scaling** — Task count ko automatically badhana/ghatana based on load:
```
Target Tracking Policy:
  Metric: ECS Service Average CPU Utilization
  Target: 60%
  Min tasks: 2
  Max tasks: 10
```
Agar CPU 60% se zyada jaata hai → naye tasks add ho jaate hain. Load kam hone pe wapas scale down.

**Why min 2 tasks (not 1)?** High availability ke liye — agar ek task crash/deploy ho raha ho, doosra abhi bhi traffic serve karta rahe.

---

## 14. Why Not Use Root User Credentials

**Definition:** Root user account owner hai — saari AWS services pe full access hota hai.

**Reasons to avoid:**
- Full access — koi restriction nahi
- High security risk agar compromise ho jaaye
- Permissions limit nahi kar sakte

**Best practice:**
- Root user sirf account setup ke liye use karo
- IAM users banao limited permissions ke sath
- Root account pe MFA (Multi-Factor Authentication) enable karo

---

## 15. Full Deployment — Step by Step

*(Screenshot mein heading thi "Full Deployment - Step by Step" — poora detailed flow yaha likh raha hoon)*

```
Step 1: Create ECR Repository
   → aws ecr create-repository --repository-name my-app

Step 2: Build & Push Docker Image
   → docker build -t my-app .
   → docker tag my-app:latest <account-id>.dkr.ecr.<region>.amazonaws.com/my-app:latest
   → docker push <account-id>.dkr.ecr.<region>.amazonaws.com/my-app:latest

Step 3: Create VPC + Subnets (or use default VPC for quick setup)
   → Public subnet (for ALB) + Private subnet (for ECS tasks, optional)

Step 4: Create Security Groups
   → ALB SG: allow 80/443 from anywhere
   → ECS Task SG: allow traffic only from ALB SG

Step 5: Create Target Group
   → Target type: IP, Protocol: HTTP, Port: container's port
   → Configure health check path

Step 6: Create Application Load Balancer
   → Attach to public subnets
   → Listener on port 80 (or 443 with SSL cert) → forward to Target Group

Step 7: Create IAM Roles
   → Task Execution Role (pull image, push logs)
   → Task Role (app's AWS permissions, if any)

Step 8: Create ECS Cluster
   → Choose Fargate (serverless) or EC2 launch type

Step 9: Create Task Definition
   → Image URI (from ECR), CPU/memory, port mappings, env vars, roles, log config

Step 10: Create ECS Service
   → Link Task Definition + Cluster + Target Group + Security Group + Subnets
   → Set desired task count (e.g. 2)

Step 11: Verify Deployment
   → Check ECS console — tasks should be "RUNNING"
   → Check Target Group — targets should be "healthy"
   → Hit the ALB's DNS name in browser — app should load

Step 12: (Optional) Attach custom domain + SSL
   → Route 53 for DNS, ACM (AWS Certificate Manager) for free SSL cert
   → Point domain to ALB, add HTTPS listener
```

---

## 16. CI/CD Integration (GitHub Actions → ECR → ECS)

*(Not in original notes — added since this is what "real" deployment looks like beyond manual steps)*

```
git push (main branch)
     │
     ▼
GitHub Actions workflow triggers
     │
     ├── docker build -t my-app .
     ├── docker push to ECR (new tag, e.g. git SHA)
     │
     ▼
Update ECS Task Definition (new image URI)
     │
     ▼
aws ecs update-service --force-new-deployment
     │
     ▼
ECS does a ROLLING DEPLOYMENT
  (starts new tasks with new image, waits for healthy,
   then drains + stops old tasks — zero downtime)
```

**Key idea:** ECS Service ka default deployment type **rolling update** hota hai — ye zero-downtime deploys deta hai kyunki naye tasks pehle healthy hone tak purane tasks chalte rehte hain.

---

## 17. Concept Diagrams

**A. Full Request Flow (Summary Flow from original notes, expanded)**
```
User Request
     │
     ▼
    ALB  (public subnet, Security Group allows 80/443)
     │
     ▼
Target Group  (health-checks targets)
     │
     ▼
ECS Task (Container)  (private subnet, SG allows only ALB traffic)
     │
     ├── uses Task Role ────────► Access other AWS services (S3, DynamoDB)
     │
     └── started by ECS Agent using Task Execution Role
              │
              ▼
        Pull Image from ECR
              │
              └──► Logs pushed to CloudWatch
```

**B. Full Infrastructure Layout**
```
                        AWS Cloud
                            │
                           VPC (10.0.0.0/16)
                  ┌─────────┴─────────┐
                  │                   │
         Public Subnet          Private Subnet
                  │                   │
                 ALB              ECS Tasks
                  │                   │
                  └─────Target Group──┘
                            │
                      (health checks)
```

**C. Image → Deployment Pipeline**
```
Local Docker Build → ECR (image storage) → ECS Task Definition (blueprint)
       → ECS Service (maintains desired count) → ALB (routes traffic)
       → CloudWatch (logs everything)
```

---

## 18. Common Mistakes & Debugging

| Mistake | Fix |
|---|---|
| Task stuck in "PENDING" forever | Check subnet has route to internet (NAT Gateway/IGW) for Fargate to pull image from ECR |
| Target Group shows "unhealthy" | Health check path wrong, or container port mismatch with Task Definition port mapping |
| Task keeps starting and stopping (crash loop) | Check CloudWatch Logs — usually app crashes due to missing env var or wrong port binding |
| "AccessDenied" pulling image from ECR | Task Execution Role missing `AmazonECSTaskExecutionRolePolicy` |
| App unreachable via ALB | Security Group not allowing ALB → ECS task traffic, or Target Group pointing to wrong port |
| 502 Bad Gateway from ALB | App not listening on the port declared in Task Definition/Target Group |

---

## 19. Interview Questions (AWS + ECS)

**Networking Basics**
1. What is a VPC, and why would you create a custom one instead of using the default?
2. Difference between a public and private subnet — what actually makes a subnet "public"?
3. Why does every subnet belong to exactly one Availability Zone?

**Security**
4. Difference between a Security Group and a Network ACL?
5. What does "Security Groups are stateful" mean in practice?
6. Why would you make the ECS task's Security Group only accept traffic from the ALB's Security Group instead of `0.0.0.0/0`?

**Load Balancing**
7. Why is ALB (Layer 7) preferred over NLB (Layer 4) for most web app deployments?
8. How does a Target Group know whether to send traffic to a container?
9. What's path-based routing, and when would you use it (e.g. one ALB for both frontend and backend)?

**IAM**
10. Difference between an IAM User and an IAM Role — why do AWS services need Roles instead of Users?
11. What is the principle of least privilege, and how does it apply to Task Roles?
12. Why should you never use root credentials for day-to-day operations?
12a. Walk through the steps of setting up a new IAM user with CLI access — what's actually happening at each step (user creation, policy attachment, access key generation)?
12b. Why do Access Keys (for CLI/programmatic access) only show the Secret Access Key once? What would you do if you lost it?
12c. What does `aws sts get-caller-identity` do, and why is it useful right after configuring the AWS CLI?

**ECS Core**
13. Explain the relationship between Task Definition, Task, and Service in ECS.
14. Difference between **Task Role** and **Task Execution Role** — give a concrete example of each.
15. What happens when an ECS Task crashes — who restarts it, and how?
16. Difference between ECS on EC2 and ECS on Fargate — when would you choose each?

**ECR**
17. What is ECR, and how is it different from Docker Hub?
18. How does ECS authenticate to pull a private image from ECR?

**Deployment / Operations**
19. Walk through what happens end-to-end when you push a new image and redeploy an ECS service.
20. What is a rolling deployment, and how does it achieve zero downtime?
21. How would you debug a Task stuck in "PENDING" state?
22. How would you debug a Target Group showing all targets as "unhealthy"?
23. How do you view logs for a crashing ECS container?

**Scaling & Monitoring**
24. How does ECS Service Auto Scaling decide when to add/remove tasks?
25. Why would you keep a minimum of 2 tasks running instead of 1, even for a small app?
26. What kind of metrics would you monitor in CloudWatch for an ECS-based app?

**Scenario-based**
27. You deploy a new image version and the app goes down for all users — what likely went wrong in your deployment setup?
28. Design a basic production architecture for a MERN app on AWS ECS — what components would you use and how do they connect?
29. Your ECS task can't pull the image from ECR — walk through how you'd diagnose it (IAM? networking? image tag?).

---

## 20. Final Takeaway

AWS ECS deployment ka mental model:

> **VPC = tera private network. Security Groups = firewall rules. ALB + Target Group = traffic router jo health-check karke sirf healthy containers ko traffic bhejta hai. ECR = image storage. ECS = orchestrator jo images ko containers ke roop mein run/scale/replace karta hai, IAM Roles ke through permissions leta hai, aur sab kuch CloudWatch mein log hota hai.**

Poora flow ek line mein:
```
Code → Docker image → ECR → ECS Task Definition → ECS Service (Fargate/EC2)
     → ALB routes public traffic → Target Group health-checks → healthy container serves request
     → logs go to CloudWatch, auto-scaling reacts to load
```
