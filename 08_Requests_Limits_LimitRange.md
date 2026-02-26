# Requests, Limits, LimitRange

We’ll simulate 6 real exam scenarios:

1. Scheduler behavior (Requests)
2. CPU throttling (Limits)
3. Memory OOM kill
4. Node allocation math
5. LimitRange enforcement
6. Edge cases examiners love

---

# ⚔️ LAB 1 — Scheduling Based on Requests

### 🎯 Goal

Understand: **Scheduler ONLY looks at requests. NOT limits.**

---

## Step 1: Check node capacity

```bash
kubectl describe node <node-name>
```

Look at:

```
Capacity:
Allocatable:
Allocated resources:
```

🧠 Study Note:

* Scheduler uses **Allocatable**
* NOT Capacity
* NOT Limits

---

## Step 2: Create heavy request pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: big-request
spec:
  containers:
  - name: c1
    image: nginx
    resources:
      requests:
        cpu: "10"
        memory: "10Gi"
```

Apply:

```bash
kubectl apply -f big.yaml
```

Check:

```bash
kubectl get pods
```

You’ll see:

```
Pending
```

Now describe:

```bash
kubectl describe pod big-request
```

You’ll see:

```
Insufficient cpu
Insufficient memory
```

---

### 🔥 KEY EXAM POINT

If Pod is **Pending**
→ ALWAYS check:

```
kubectl describe pod <name>
```

Look for:

* Insufficient CPU
* Insufficient memory
* Taints
* NodeSelector mismatch

---

# ⚔️ LAB 2 — Requests vs Limits Difference

Create this:

```yaml
resources:
  requests:
    cpu: "100m"
  limits:
    cpu: "2000m"
```

Node has 2 CPUs.

Scheduler only checks:

```
100m
```

Even though limit is 2 CPUs.

🧠 Study Note:

> Scheduler DOES NOT care about limits.

---

# ⚔️ LAB 3 — CPU Throttling (Real Understanding)

Use your cpu.yaml.

Apply:

```bash
kubectl apply -f cpu.yaml
```

Watch:

```bash
kubectl top pod cpu-demo
```

You’ll see CPU close to 900m max.

Now exec inside:

```bash
kubectl exec -it cpu-demo -- sh
```

Run:

```bash
top
```

You’ll see CPU stuck near limit.

---

### 🧠 Study Points

* CPU = Compressible resource
* CPU exceeding limit = throttled
* Pod does NOT die
* Performance degrades

---

### 🔥 CKA Trap Question

If CPU limit not defined:
→ Pod can use full node CPU.

Correct.

---

# ⚔️ LAB 4 — Memory OOM Kill (Important)

Modify memory.yaml:

```yaml
args: ["--vm", "1", "--vm-bytes", "250M", "--vm-hang", "1"]
```

Apply.

Watch:

```bash
kubectl get pods -w
```

You’ll see:

```
CrashLoopBackOff
```

Now:

```bash
kubectl describe pod memory-demo
```

Look for:

```
OOMKilled
```

---

### 🧠 WHY Killed Even If Node Has Free Memory?

Because:

* Memory limit enforced by cgroups
* Not by scheduler
* Hard boundary

---

### 🔥 Exam Brain Rule

Memory = Incompressible
CPU = Compressible

If memory exceeds limit → container dies.

---

# ⚔️ LAB 5 — Node Allocation Math (CKA Favorite)

Assume:

Node:

* 4 CPU
* 8Gi memory

Pod:

```
requests:
  cpu: 1
  memory: 2Gi
```

Max pods scheduler can place?

Answer:

* CPU limit: 4 pods
* Memory limit: 4 pods

So 4 pods.

Even if:

* Limits = 3 CPU
* Scheduler only checks request

---

### 🧠 KEY CONCEPT

Scheduling density = based on **requests only**

Limits do NOT affect scheduling.

---

# ⚔️ LAB 6 — LimitRange Enforcement

Apply your limitrange.yaml

```bash
kubectl apply -f limitrange.yaml
```

Now create pod WITHOUT resources:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-resources
spec:
  containers:
  - name: nginx
    image: nginx
```

Apply.

Now check:

```bash
kubectl get pod no-resources -o yaml
```

You will see:
Requests and limits automatically injected.

---

### 🧠 Important

LimitRange is:

* Namespace scoped
* Applies only at creation
* Not retroactive

---

# ⚔️ LAB 7 — Min/Max Violation

Create pod requesting:

```yaml
requests:
  cpu: "100m"
```

But min in LimitRange = 500m

Apply.

Result:

Pod will FAIL.

Error:

```
must be greater than or equal to 500m
```

---

# ⚔️ LAB 8 — What If Limit < Request?

Try this:

```yaml
resources:
  requests:
    cpu: "1"
  limits:
    cpu: "500m"
```

Apply.

Result:

Validation error.

Rule:

```
Limit must be >= Request
```

---

# ⚔️ LAB 9 — No Limits Scenario (Danger Zone)

If:

* No limits defined
* No LimitRange

Pod can:

* Consume full node memory
* Kill neighbors

This is the noisy neighbor problem.

---

# ⚔️ LAB 10 — Check Allocated Resources Per Node

```bash
kubectl describe node <node>
```

Look at:

```
Allocated resources:
```

This is scheduler accounting.

---

# ⚔️ LAB 11 — QoS Classes (VERY IMPORTANT)

Run:

```bash
kubectl get pod <name> -o yaml | grep qosClass
```

You’ll see one of:

1. Guaranteed
2. Burstable
3. BestEffort

---

### 🧠 How QoS is Decided

### 1️⃣ Guaranteed

Requests == Limits for CPU AND memory.

### 2️⃣ Burstable

Requests < Limits

### 3️⃣ BestEffort

No requests or limits defined.

---

### 🔥 OOM Kill Order

When node memory pressure:

1. BestEffort dies first
2. Burstable next
3. Guaranteed last

CKA loves this.

---

# ⚔️ LAB 12 — Remove Metrics Server (Break It)

Delete metrics-server:

```bash
kubectl delete -f components.yaml
```

Now:

```bash
kubectl top nodes
```

Fails.

Because:

* metrics.k8s.io API missing.

Check:

```bash
kubectl get apiservice
```

Look for:

```
v1beta1.metrics.k8s.io
```

---

# ⚔️ LAB 13 — Check API Aggregation

```bash
kubectl get apiservice v1beta1.metrics.k8s.io -o yaml
```

If status False:
→ metrics-server not healthy.

---

# 🔥 MASTER CHECKLIST (MEMORIZE THIS)

✔ Scheduler uses requests
✔ Limits enforced by kernel
✔ CPU → throttled
✔ Memory → OOM kill
✔ LimitRange → namespace scoped
✔ Requests decide density
✔ QoS affects OOM order
✔ Limit must be >= Request
✔ metrics-server required for kubectl top
✔ Node Allocatable used for scheduling

---

# 🧠 Real-World DevOps Insight

In production:

* Never leave memory unlimited
* CPU limit sometimes avoided (to prevent throttling)
* Always set requests
* Monitor with Prometheus, not just metrics-server

---
Perfect.
Now we move to **cluster control mode**.

Requests/Limits = per Pod control
**ResourceQuota = per Namespace control**

No theory fluff. Pure killer hands-on + exam brain.

---

# 🔥 What is ResourceQuota?

It restricts **total resource consumption inside a namespace**.

It can limit:

* Total CPU
* Total Memory
* Total Pods
* Total Services
* Total ConfigMaps
* Total PersistentVolumeClaims
* Total Requests
* Total Limits

It is namespace-scoped.

---

# ⚔️ LAB 1 — Basic CPU + Memory Quota

## Step 1 — Create quota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: default
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 2Gi
    limits.cpu: "4"
    limits.memory: 4Gi
```

Apply:

```bash
kubectl apply -f rq.yaml
```

Check:

```bash
kubectl describe quota
```

You’ll see:

```
Used
Hard
```

---

# 🧠 Study Point

Quota tracks:

* Used (current consumption)
* Hard (maximum allowed)

Scheduler + API server enforce this at creation time.

---

# ⚔️ LAB 2 — Try to Break It

Create pod:

```yaml
resources:
  requests:
    cpu: "1"
    memory: "1Gi"
  limits:
    cpu: "2"
    memory: "2Gi"
```

Create 2 such pods.

That consumes:

Requests:

* 2 CPU
* 2Gi memory

Limits:

* 4 CPU
* 4Gi memory

Now try creating third pod.

It will FAIL.

Error:

```
exceeded quota
```

---

# 🔥 EXAM RULE

ResourceQuota blocks resource creation at API level.

Pod will not even be created.

Not Pending.
Not Scheduled.
Creation denied.

---

# ⚔️ LAB 3 — Enforce Pod Count Limit

```yaml
spec:
  hard:
    pods: "2"
```

Apply.

Now try creating 3rd pod.

Fails immediately.

---

# 🧠 Important

Quota can restrict:

```
pods
services
configmaps
persistentvolumeclaims
secrets
replicationcontrollers
deployments
```

CKA sometimes tests:
Limit total PVC count.

---

# ⚔️ LAB 4 — Quota Without Requests (Important Trap)

If a namespace has ResourceQuota for:

```
requests.cpu
```

And you try creating pod WITHOUT resource requests:

It will FAIL.

Error:

```
must specify requests.cpu
```

🔥 This is common exam trap.

---

# 🧠 Why?

Because quota tracking works on requests/limits.
If not defined → cannot calculate → reject.

---

# ⚔️ LAB 5 — ResourceQuota + LimitRange Together

This is real-world combo.

Flow:

1. LimitRange injects default requests
2. ResourceQuota evaluates total usage

Order:
Admission Controller chain:

LimitRanger → ResourceQuota

So:

If pod has no requests:
LimitRange adds defaults
Then Quota evaluates

---

# 🔥 CRITICAL CONCEPT

ResourceQuota does NOT schedule.
It only blocks creation.

Scheduler still uses requests.

---

# ⚔️ LAB 6 — Storage Quota (PVC)

Example:

```yaml
spec:
  hard:
    persistentvolumeclaims: "3"
    requests.storage: 10Gi
```

Now:

* Only 3 PVCs allowed
* Total storage requested must not exceed 10Gi

Try exceeding → denied.

---

# ⚔️ LAB 7 — Check Quota Status Properly

```bash
kubectl describe quota compute-quota
```

Look at:

```
Resource         Used     Hard
--------         ----     ----
requests.cpu     1        2
limits.cpu       2        4
```

Used updates dynamically.

---

# ⚔️ LAB 8 — Delete Pod Effect

Delete a pod:

```bash
kubectl delete pod <name>
```

Quota Used decreases immediately.

---

# 🔥 Scheduler + Quota Mental Model

Pod creation flow:

1. API Server receives request
2. Admission controllers run:

   * LimitRange
   * ResourceQuota
3. If passes → Pod stored in etcd
4. Scheduler picks node
5. Kubelet runs pod

Quota never interacts with scheduler.

---

# ⚔️ LAB 9 — Object Count Quota

Restrict ConfigMaps:

```yaml
spec:
  hard:
    configmaps: "2"
```

Try creating 3rd → denied.

CKA loves this.

---

# 🔥 Memory Brain Chart

| Feature       | Scope     | Enforced By | Blocks Creation? | Affects Scheduling? |
| ------------- | --------- | ----------- | ---------------- | ------------------- |
| Requests      | Pod       | Scheduler   | No               | Yes                 |
| Limits        | Pod       | Kernel      | No               | No                  |
| LimitRange    | Namespace | Admission   | No               | Indirect            |
| ResourceQuota | Namespace | Admission   | YES              | No                  |

Memorize this.

---

# ⚔️ LAB 10 — Remove Quota

```bash
kubectl delete quota compute-quota
```

Pods can now scale freely.

---

# 🧠 Advanced Insight (DevOps Reality)

Production setup:

* LimitRange → force defaults
* ResourceQuota → prevent abuse
* Monitoring → Prometheus
* Alerts → based on request saturation

Cluster multi-tenant strategy:
Each team gets namespace + quota.

---

# 🔥 CKA-Level Edge Cases

1. Quota on limits but not requests → only limits counted
2. If quota exists for requests → pod MUST specify requests
3. Quota updates do NOT affect existing pods
4. Deleting quota does NOT delete pods
5. ReplicaSet scaling fails if quota exceeded

---

# ⚔️ Killer Drill (Do This Fast)

1. Create namespace `team-a`
2. Add ResourceQuota:

   * pods: 3
   * requests.cpu: 2
3. Create 4 nginx pods
4. Observe failure
5. Delete one pod
6. Create again

Time yourself.

Target: < 6 minutes.

---
