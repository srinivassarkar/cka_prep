# 🚀 Kubernetes Autoscaling Deep Dive

## KIND → Metrics Server → HPA → VPA → Cluster Autoscaler (Production-Level Understanding)

---

# 🧱 Architecture Diagram

```
                        +-------------------+
                        |     User Traffic  |
                        +---------+---------+
                                  |
                                  v
                        +-------------------+
                        |   nginx-hpa Pod   |
                        +---------+---------+
                                  |
             -----------------------------------------------
             |                   |                         |
             v                   v                         v
+------------------+   +------------------+   +-------------------------+
| HPA              |   | Cluster          |   | VPA                     |
| Scales Replicas  |   | Autoscaler       |   | (Recommender Mode)      |
| based on CPU %   |   | Scales Nodes     |   | Recommends CPU/Memory   |
+------------------+   +------------------+   +-------------------------+
```

---

# 📦 1️⃣ Environment Setup

## System Info

```bash
uname -a
```

```bash
docker info | grep -i cgroup
```

Result:

* Cgroup Version: 1
* Driver: cgroupfs

> **Important:** CPU limits use Linux CFS (Completely Fair Scheduler) quotas — this directly affects throttling behavior at the container level.

---

# 🐳 2️⃣ Create KIND Multi-Node Cluster

## kind-config.yaml

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: scaling-lab
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

Create cluster:

```bash
kind create cluster --config kind-config.yaml
```

Verify:

```bash
kubectl get nodes
```

---

# 📊 3️⃣ Install Metrics Server (Required for HPA)

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Patch for KIND (insecure TLS):

```bash
kubectl edit deployment metrics-server -n kube-system
```

Add under `args`:

```yaml
- --kubelet-insecure-tls
```

Verify:

```bash
kubectl top nodes
kubectl top pods
```

> ✅ If this works → HPA can function.

---

# 🌐 4️⃣ Deploy NGINX Application

## nginx-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-hpa
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-hpa
  template:
    metadata:
      labels:
        app: nginx-hpa
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            cpu: 100m
            memory: 200Mi
          limits:
            cpu: 500m
            memory: 500Mi
        ports:
        - containerPort: 80
```

Apply:

```bash
kubectl apply -f nginx-deployment.yaml
```

---

# 📈 5️⃣ Create Horizontal Pod Autoscaler (HPA)

## hpa.yaml

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-hpa
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

Apply:

```bash
kubectl apply -f hpa.yaml
```

Verify:

```bash
kubectl get hpa
```

---

# 🧠 Internal Working of HPA

## The Control Loop

Every **15 seconds**, the HPA controller:

1. Checks the Metrics API
2. Calculates average CPU utilization across all pods
3. Computes desired replicas using the formula below

## Scaling Formula

```
desiredReplicas = currentReplicas × (currentMetricValue / targetMetricValue)
```

### Example

```
currentReplicas   = 2
currentUtilization = 100%
targetUtilization  = 50%

desiredReplicas = 2 × (100 / 50) = 4 replicas
```

## How CPU % Is Calculated

```
CPU % = current_usage / request
```

Example:

* request = 100m
* usage = 50m
* CPU% = 50%

If above 50% → scale up.

---

# ⚙️ 6️⃣ Load Testing

```bash
kubectl run -i --tty load-generator --rm --image=busybox -- /bin/sh
```

Inside the shell:

```sh
while true; do wget -q -O- http://nginx-hpa; done
```

Observe HPA reacting:

```bash
kubectl get hpa
kubectl get pods
```

---

# 🔥 HPA: Interview Gold

## Why did HPA NOT scale? — Checklist

| Check | What to Look For |
|-------|-----------------|
| Is `metrics-server` running? | `kubectl get pods -n kube-system \| grep metrics` |
| Are CPU **requests** defined? | HPA needs requests to calculate CPU% |
| Is load **sustained** long enough? | HPA has a stabilization window |
| Is `maxReplicas` already reached? | HPA won't go beyond it |

## HPA Limitations

* Works best for **stateless** apps
* Does **not** scale reliably on memory
* CPU scaling is **reactive**, not predictive
* Scaling **down** has a cooldown delay (default: 5 minutes)

---

# ☁️ PART 2 — CLUSTER AUTOSCALER (Conceptual)

> **HPA scales pods. Cluster Autoscaler scales nodes.**

## The Full Elastic Flow

```
Load ↑
  → CPU usage ↑
  → HPA increases replicas
  → Some pods become Pending (no node capacity)
  → Cluster Autoscaler detects unschedulable pods
  → Adds new node to the cluster
  → Pending pods get scheduled
  → Load distributed

Load ↓
  → HPA scales down pods
  → Nodes become idle
  → Cluster Autoscaler removes idle nodes
```

**That is real cloud-native elasticity.**

## Production Insight

In real clusters, HPA and Cluster Autoscaler work together:

* HPA increases replicas
* Nodes fill up → pods go **Pending**
* Cluster Autoscaler provisions a new node
* Scheduler places pods on the new node

Without Cluster Autoscaler, pods stay Pending indefinitely.

---

# 📐 7️⃣ Install Vertical Pod Autoscaler (VPA)

## Install CRDs

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/vertical-pod-autoscaler/deploy/vpa-v1-crd-gen.yaml
```

## Install RBAC

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/vertical-pod-autoscaler/deploy/vpa-rbac.yaml
```

## Deploy Components

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/vertical-pod-autoscaler/deploy/recommender-deployment.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/vertical-pod-autoscaler/deploy/updater-deployment.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/vertical-pod-autoscaler/deploy/admission-controller-deployment.yaml
```

Verify:

```bash
kubectl get pods -n kube-system | grep vpa
```

> ⚠️ Admission controller may hang in KIND due to TLS/webhook. Not required for recommendation mode.

---

# 🧠 How VPA Works — First Principles

VPA does **NOT** change replicas.

VPA changes:

```
CPU & memory requests
```

It can **restart pods** to apply new resource values.

## VPA Components (Deep Understanding)

| Component | Role |
|-----------|------|
| **Recommender** | Watches historical usage, calculates ideal requests |
| **Updater** | Evicts pods whose resources are out of date |
| **Admission Controller** | Injects new resource requests during pod creation |

```
Recommender → calculates ideal resources
Updater     → evicts pods that need updating
Admission   → injects updated requests into new pods
```

---

# 📊 8️⃣ Create VPA (Recommendation Mode)

## vpa.yaml

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: nginx-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-hpa
  updatePolicy:
    updateMode: "Off"
```

Apply:

```bash
kubectl apply -f vpa.yaml
```

Check recommendation:

```bash
kubectl describe vpa nginx-vpa
```

---

# 📊 VPA Recommendation Output Explained

```
Target:
  Cpu: 25m
  Memory: 250Mi
Upper Bound:
  Cpu: 1069m
Lower Bound:
  Cpu: 10m
Uncapped Target:
  Cpu: 25m
```

## What Those Fields Mean

| Field | Meaning |
|-------|---------|
| **Lower Bound** | Safe minimum — don't go below this |
| **Target** | Ideal request to set |
| **Upper Bound** | Safe maximum — protects from spikes |
| **Uncapped Target** | Raw calculation before policy limits applied |

### Interpretation

* Current request (100m) is **over-provisioned**
* Real usage is ~25m
* Upper bound (1069m) protects from burst spikes

---

# 🔥 updateMode Explained

```
Off     → Recommend only (read VPA output, apply manually)
Initial → Set resources only during pod creation (no eviction)
Auto    → Evict + recreate pod with new resource values (disruptive)
```

> ⚠️ `Auto` mode = **pod restart**. Use with caution in production.

---

# 🧠 Deep Production Insights

## Requests vs Limits

| Field | Purpose |
|-------|---------|
| **request** | Scheduling guarantee — node must have this available |
| **limit** | Hard cap — triggers Linux CFS throttling when exceeded |

## What Causes CPU Throttling?

If you set:

```
request = 25m
limit   = 25m
```

Linux sets `cpu.cfs_quota_us` and `cpu.cfs_period_us` to enforce 25m exactly.
The container gets **throttled immediately** when it exceeds 25m — even for microsecond bursts.

## Safe Production Pattern

```yaml
resources:
  requests:
    cpu: 100m
  limits:
    cpu: 500m
```

Why this works:

* Guaranteed baseline scheduling (100m)
* Burst headroom allowed (up to 500m)
* HPA reacts before throttling hurts users

---

# ⚠️ HPA + VPA Conflict — The Oscillation Problem

HPA uses:

```
CPU% = usage / request
```

**The problem:**

If VPA **reduces** the request (e.g., from 100m → 25m):

```
Same CPU usage / smaller request = higher CPU%
→ HPA sees CPU% spike
→ HPA scales up replicas unnecessarily
→ VPA recalculates on more pods
→ Oscillation loop
```

This is the most dangerous misconfiguration in production autoscaling.

---

# 🔥 HPA vs VPA — Brain Table

| Feature | HPA | VPA |
|---------|-----|-----|
| Scales | Replicas | CPU / Memory requests |
| Restarts pods? | No | Yes (in Auto/Initial mode) |
| Uses metrics-server? | Yes | No (uses its own metrics pipeline) |
| Good for | Stateless apps | Right-sizing workloads |
| Risk | Thrashing if requests change | Pod disruption on update |

---

# 🏭 Production Scaling Strategy

| Workload | Strategy |
|----------|----------|
| **Frontend** | HPA — CPU-based scaling |
| **Backend APIs** | HPA — CPU or custom metrics (RPS, latency) |
| **Databases** | Vertical scaling (manual) or managed service |
| **Batch/ML jobs** | VPA — right-size resources per job |
| **Cluster** | Cluster Autoscaler — node provisioning |

**Real-world pattern:**

* Use VPA in `Off` mode to get recommendations
* Apply recommendations to requests manually
* Use HPA on custom metrics (RPS, queue depth) — more predictive than CPU

---

# 💣 Interview Traps (Know These)

| Trap | Root Cause | Fix |
|------|-----------|-----|
| HPA not scaling | CPU requests not defined | Add `resources.requests.cpu` |
| HPA not reacting | `metrics-server` missing | Install and patch metrics-server |
| VPA causing restarts | `updateMode: Auto` | Use `Off` or `Initial` in production |
| HPA + VPA CPU conflict | VPA reduces request → HPA sees spike | Never run HPA (CPU) + VPA (Auto) together |
| Pods pending forever | Cluster Autoscaler not installed | Install Cluster Autoscaler |

---

# 🏁 Final Architecture View

```
                        +--------------------+
                        |    Load Traffic    |
                        +---------+----------+
                                  |
                                  v
                          +---------------+
                          |   nginx Pod   |
                          +-------+-------+
                                  |
          --------------------------------------------------------
          |                       |                              |
          v                       v                              v
+------------------+   +--------------------+   +-----------------------+
| HPA              |   | Cluster Autoscaler |   | VPA (Recommender)     |
| Replica Scaling  |   | Node Provisioning  |   | Resource Suggestion   |
+------------------+   +--------------------+   +-----------------------+
          |                       ^
          | Pods Pending          |
          +-----------------------+
           Triggers node scale-up
```

---

# 🔥 Elite-Level Understanding — Full Elastic System Flow

```
Load ↑
  → CPU usage ↑
  → HPA scales pods
  → Nodes full → Cluster Autoscaler scales nodes
  → Load distributed across new pods

Load ↓
  → HPA scales down pods
  → Idle nodes removed by Cluster Autoscaler
```

VPA runs alongside in `Off` mode:

```
VPA Recommender watches usage patterns
  → Generates right-sizing recommendations
  → Engineer reviews and adjusts requests manually
  → Next deploy uses optimal resource values
```

---

# 🎓 Final Learning Outcome

After this lab you now understand:

* How CPU requests influence scaling math in HPA
* How limits trigger Linux CFS throttling
* Why `limit = request` is dangerous (throttling on any burst)
* Why HPA + VPA must be designed carefully (oscillation risk)
* How Cluster Autoscaler extends pod scaling to node scaling
* How VPA components (Recommender, Updater, Admission) work together
* How production-safe autoscaling is layered: VPA right-sizes → HPA reacts → Cluster Autoscaler provisions
* How KIND differs from cloud clusters (no cloud node pools, TLS quirks)

---

# 📌 Conclusion

Autoscaling is not about:

```
kubectl autoscale
```

It is about:

* Linux CPU scheduler and CGroup quotas
* Scheduling guarantees (requests) vs hard caps (limits)
* Scaling reaction time and stabilization windows
* Traffic burst modeling
* Right-sizing to avoid waste and throttling
* Coordinating HPA + VPA + Cluster Autoscaler without conflicts
* Production stability through layered, well-understood elasticity
