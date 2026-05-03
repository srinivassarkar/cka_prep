# ⚡ Kubernetes Pod Priority, PriorityClass & Preemption


> *"Priority tells the scheduler who goes first. Preemption tells it who gets kicked out. Knowing the difference — and exactly when each fires — is what separates a CKA pass from a CKA fail."*

---

## 0. First Principles

The mental model that never changes:

```
Cluster resources are finite.
When two pods compete for the same slot, something has to win.
Priority decides the winner.
Preemption clears the way.
```

**Two distinct mechanisms — often confused:**

```
PRIORITY     → Scheduling order when resources are available
               "Schedule the 1000 before the 10"

PREEMPTION   → Eviction trigger when resources are NOT available
               "Kill the 10 to make room for the 1000"
```

**Three actors:**

```
PriorityClass   → cluster-scoped policy object (the rulebook)
      │
      ▼
  Pod spec       → references priorityClassName (the player)
      │
      ▼
  Scheduler      → reads priority, makes decisions (the referee)
```

**What never changes:**
- Higher numeric value = higher priority. Always.
- Preemption is **scheduler-driven**, not kubelet-driven. Running pods are never proactively evicted due to priority alone.
- QoS (Guaranteed/Burstable/BestEffort) is completely separate from Priority. They operate at different layers.

---

## 1. Reality Constraints

What Kubernetes actually does and doesn't do:

| Claim | Reality |
|---|---|
| "High-priority pods always run" | ❌ They're preferred, but still subject to taints, affinity, resource limits |
| "Preemption fires during CPU pressure" | ❌ Preemption only fires when a **new pod fails to schedule** |
| "kubelet evicts low-priority pods during OOM" | ❌ kubelet uses QoS class, not PriorityClass, for eviction |
| "PriorityClass is namespace-scoped" | ❌ It's **cluster-scoped** — one object, usable everywhere |
| "You can set priority > 1 billion" | ❌ User range: `-2,147,483,648` to `1,000,000,000`. System reserves `2,000,000,000+` |
| "Multiple globalDefault: true is fine" | ❌ API server rejects any second `globalDefault: true` |
| "preemptionPolicy: Never = low scheduling priority" | ❌ It still schedules before lower-priority pods — just won't evict them |
| "Preemption is immediate and ungraceful" | ❌ Eviction is **graceful** — respects `terminationGracePeriodSeconds` |
| "No priorityClassName = priority 0" | ⚠️ Only if no `globalDefault` class exists. If one does, it inherits that. |

**Priority value ranges:**

```
┌─────────────────────────────────────────────────────────┐
│  -2,147,483,648        User-defined range        1,000,000,000 │
├─────────────────────────────────────────────────────────┤
│                   (gap — not used)                      │
├─────────────────────────────────────────────────────────┤
│  2,000,000,000   system-cluster-critical                │
│  2,000,001,000   system-node-critical                   │
└─────────────────────────────────────────────────────────┘
```

---

## 2. Decision Logic

### Should I use Priority? Which method?

```
Do I need to control which pods get resources first?
├── NO → Don't bother. Default priority 0 for all.
└── YES → Do I need different tiers?
    ├── YES → Create named PriorityClasses (high/medium/low)
    └── NO → One class with a single value is fine

Should the high-priority pod evict others if it can't schedule?
├── YES (default) → preemptionPolicy: PreemptLowerPriority (default, omit field)
└── NO  → preemptionPolicy: Never
    When? Stateful apps, logging agents, non-disruptive batch jobs

Should there be a cluster-wide default for pods that don't specify?
├── YES → Mark one class: globalDefault: true
└── NO → Pods with no priorityClassName get priority 0
```

### preemptionPolicy: PreemptLowerPriority vs Never

| Behavior | PreemptLowerPriority (default) | Never |
|---|---|---|
| Scheduled before lower-priority pods | ✅ | ✅ |
| Evicts lower-priority pods to get scheduled | ✅ | ❌ |
| Stays Pending if no room | ❌ | ✅ (waits) |
| Use case | Critical APIs, payment systems | Logging agents, batch, stateful apps |

### Priority vs QoS — which fires when?

| Situation | Mechanism | Trigger |
|---|---|---|
| New pod can't be scheduled (resources full) | **Preemption** (scheduler) | New pod arrival |
| Node is under memory pressure | **QoS-based eviction** (kubelet) | Node-level OOM |
| CPU throttling on a running pod | **CFS throttling** (kernel) | Runtime |
| Pod priority order at scheduling time | **Priority** (scheduler) | Every scheduling cycle |

These are independent systems. A `BestEffort` QoS pod with `high-priority` PriorityClass can be OOM-killed by the kubelet while a `Guaranteed` QoS pod with `low-priority` stays alive.

---

## 3. Internal Working

### How priority affects scheduling — step by step

```
1. Pod created → API Server writes to etcd
         │
2. Scheduler picks up unscheduled pod
         │
3. Scheduler reads pod.spec.priorityClassName
         → resolves to numeric value from PriorityClass object
         → if no priorityClassName: use globalDefault value, or 0
         │
4. Scheduler adds pod to the scheduling queue (priority queue, not FIFO)
         │
5. Higher-priority pods are dequeued and attempted first
         │
6. Scheduler runs Filtering: find nodes that can fit the pod
         │
         ├── Nodes found → Score → Bind → Done ✅
         │
         └── No nodes found → Preemption check
                  │
7. Scheduler identifies victim pods (lower priority than the pending pod)
         │
8. Scheduler simulates eviction: "if I remove pod X, can the new pod fit?"
         │
9. Victims selected → API Server sends DELETE (graceful termination)
         │
10. Victims terminate (respecting terminationGracePeriodSeconds)
         │
11. Resources freed → Pending pod gets scheduled ✅
```

### What happens to preempted pods?

```
Preempted pod → graceful DELETE → terminates
      │
      └── If owned by Deployment/ReplicaSet → controller creates replacement
                │
                └── Replacement goes into scheduling queue
                          │
                          └── May stay Pending if resources are still insufficient
```

Preempted pods don't disappear from the cluster design — their controllers will try to reschedule them. They just won't get CPU until resources free up.

### The scheduling queue internals

The scheduler uses a **priority queue** (not FIFO). Pod ordering is:
1. Priority value (highest first)
2. Within same priority: arrival time (older first)

This means: two pods with the same priority class are scheduled in arrival order. A new pod with higher priority jumps the entire queue.

---

## 4. Hands-On

### Step 1: Inspect cluster capacity and existing PriorityClasses

```bash
# Node capacity
kubectl describe node <node-name> | grep -A10 "Allocatable"

# Existing priority classes (system ones are always present)
kubectl get pc
# NAME                      VALUE         GLOBAL-DEFAULT
# system-cluster-critical   2000000000    false
# system-node-critical      2000001000    false

# Who uses system-node-critical?
kubectl describe pod -n kube-system etcd-<node>
kubectl describe pod -n kube-system kube-apiserver-<node>
# Priority:             2000001000
# Priority Class Name:  system-node-critical
```

### Step 2: Create PriorityClasses

```yaml
# 01-priorityclass.yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000
globalDefault: false
description: "Critical production workloads that must preempt lower-priority pods if needed"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: medium-priority
value: 100
globalDefault: false
description: "Important batch or background workloads that can tolerate delay"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 10
globalDefault: true
description: "Default for development and non-critical test workloads"
```

```bash
kubectl apply -f 01-priorityclass.yaml
kubectl get pc
# NAME              VALUE   GLOBAL-DEFAULT
# high-priority     1000    false
# low-priority      10      true       ← any pod without priorityClassName gets 10
# medium-priority   100     false
```

### Step 3: Non-disruptive high-priority class

```yaml
# Use when: high priority but must not evict running pods
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-non-disruptive
value: 900
globalDefault: false
preemptionPolicy: Never
description: "High priority scheduling preference, but no preemption"
```

### Step 4: Deploy low-priority workload

```yaml
# 02-low-priority-nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lp-nginx
spec:
  replicas: 5
  selector:
    matchLabels:
      app: lp-nginx
  template:
    metadata:
      labels:
        app: lp-nginx
    spec:
      # No priorityClassName → inherits globalDefault (low-priority = 10)
      containers:
        - name: nginx
          image: nginx:latest
          resources:
            requests:
              memory: "64Mi"
              cpu: "1000m"    # 1 vCPU per pod → 5 vCPUs total
```

```bash
kubectl apply -f 02-low-priority-nginx.yaml

# Verify priority assignment
kubectl describe pod <lp-nginx-pod> | grep -i priority
# Priority:             10
# Priority Class Name:  low-priority
```

### Step 5: Deploy high-priority workload and observe preemption

```yaml
# 03-high-priority-nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hp-nginx
spec:
  replicas: 5
  selector:
    matchLabels:
      app: hp-nginx
  template:
    metadata:
      labels:
        app: hp-nginx
    spec:
      priorityClassName: high-priority    # value: 1000
      containers:
        - name: nginx
          image: nginx:latest
          resources:
            requests:
              memory: "64Mi"
              cpu: "3000m"    # 3 vCPUs per pod → 15 vCPUs total requested
```

```bash
# Watch in real time BEFORE applying
kubectl get pods -w

# Apply and observe preemption
kubectl apply -f 03-high-priority-nginx.yaml

# After: verify which pods survived
kubectl describe node <node> | grep -A5 "Non-terminated Pods"
```

**What you'll see on an 11-vCPU single node:**

```
Before: ~5.950 vCPUs used (control plane ~950m + 5×lp-nginx 5000m)
After:  hp-nginx pods (3000m each) evict lp-nginx pods to fit

hp-nginx pods: Running (priority 1000 wins)
lp-nginx pods: some Pending or Terminating (evicted, controller tries to reschedule)
```

### Step 6: Verify via describe

```bash
kubectl describe pod <hp-nginx-pod> | grep -i priority
# Priority:             1000
# Priority Class Name:  high-priority

kubectl get events --sort-by='.lastTimestamp' | grep -i preempt
# Preempted: default/lp-nginx-xxx
```

---

## 5. Production Flow

### Real-world priority tier architecture

```
┌──────────────────────────────────────────────────────────┐
│                    Cluster                               │
│                                                          │
│  [system-node-critical: 2000001000]                     │
│   etcd, kube-apiserver, kubelet agents                  │
│                                                          │
│  [system-cluster-critical: 2000000000]                  │
│   CoreDNS, kube-proxy                                   │
│                                                          │
│  ════════════════ USER WORKLOAD RANGE ════════════════  │
│                                                          │
│  [critical-prod: 1000]  preemptionPolicy: PreemptLower  │
│   payment-api, auth-service, checkout                   │
│                                                          │
│  [standard-prod: 500]   preemptionPolicy: PreemptLower  │
│   product-api, user-service, search                     │
│                                                          │
│  [batch-jobs: 100]      preemptionPolicy: Never         │
│   nightly-reports, data-sync, ML training               │
│                                                          │
│  [dev-default: 10]      globalDefault: true             │
│   dev environments, test pods, staging                  │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### Design patterns

**Pattern 1: Protect stateful/logging agents**
```yaml
# Logging agents, monitoring sidecars — don't let them evict others
preemptionPolicy: Never
value: 500
# Still prioritized during normal scheduling
# But won't cause disruption during resource crunch
```

**Pattern 2: Tiered CI/CD pipelines**
```
prod-deploy pods: critical-prod (1000)
integration-test: batch-jobs (100)
unit-test:        dev-default (10)
# CI tests get bumped when prod deploys compete for nodes
```

**Pattern 3: Cluster autoscaler integration**
```
Pending pod (high-priority) → triggers Cluster Autoscaler
Autoscaler adds node → pod schedules without preempting
Best of both worlds: scale out before evicting
```

### PriorityClass + PodDisruptionBudget (PDB) — the safety net

Preemption respects PDBs. If evicting a pod would violate a PDB, the scheduler finds a different victim. Combine both for production safety:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payment-api-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: payment-api
# Even during preemption, at least 2 payment-api pods stay alive
```

---

## 6. Mistakes

### Mistake 1: Expecting preemption during node CPU pressure

**Symptom:** Node is CPU-throttling, low-priority pods are running, high-priority pods are also running — user expects low-priority pods to be evicted  
**Root cause:** Preemption is **scheduler-driven**, not runtime-driven. It only fires when a **new pod fails to schedule**. Running pods are never proactively evicted due to priority.  
**Fix:** Use QoS classes (Guaranteed/Burstable/BestEffort) for runtime eviction behavior. Or: trigger a rescheduling cycle by deleting and redeploying workloads.

---

### Mistake 2: Two PriorityClasses with `globalDefault: true`

**Symptom:** `kubectl apply` fails with validation error  
**Root cause:** Kubernetes enforces exactly one `globalDefault: true` per cluster  
**Fix:**
```bash
# Find the existing globalDefault
kubectl get pc -o yaml | grep -A5 "globalDefault: true"
# Patch the old one before creating the new one
kubectl patch pc <old-class> -p '{"globalDefault": false}'
```

---

### Mistake 3: Setting `preemptionPolicy: Never` but expecting the pod to always run

**Symptom:** High-priority pod is Pending indefinitely — thought "high priority" means it always runs  
**Root cause:** `preemptionPolicy: Never` prevents eviction of others. If the cluster is full, the pod waits.  
**Fix:** Either remove `preemptionPolicy: Never`, or ensure cluster has capacity (autoscaler, node pools). Be deliberate about which workloads get `Never`.

---

### Mistake 4: PriorityClass value above 1 billion

**Symptom:**
```
json: cannot unmarshal number 1000000001 into Go struct field PriorityClass.value of type int32
```
**Root cause:** User range is strictly capped at `1,000,000,000`. System classes live above `2,000,000,000`.  
**Fix:** Stay within `[-2147483648, 1000000000]` for all user-defined classes.

---

### Mistake 5: Assuming evicted pods are gone for good

**Symptom:** lp-nginx pods are evicted, operator panics thinking the deployment is broken  
**Root cause:** Preemption gracefully deletes pods. Deployment controller immediately creates replacements — they're just Pending waiting for resources.  
**Fix:** Check `kubectl get pods` — Pending status means rescheduling is in progress. This is expected and correct behavior. Scale down the high-priority deployment if you need the low-priority ones back.

---

### Mistake 6: Using same label selector in both Deployments

**Symptom:** One Deployment hijacks pods from the other; ReplicaSets fight over pod ownership  
**Root cause:** Both `lp-nginx` and `hp-nginx` using `app: nginx` as selector — Kubernetes matches ownership by labels  
**Fix:** Always use unique labels per Deployment:
```yaml
# lp-nginx deployment
labels:
  app: lp-nginx
# hp-nginx deployment
labels:
  app: hp-nginx
```

---

## 7. Interview Answers

**Q: What is a PriorityClass in Kubernetes and why does it exist?**

> "A PriorityClass is a cluster-scoped resource that maps a human-readable name to a numeric priority value. It exists to decouple priority configuration from individual workload manifests — instead of every team hardcoding a number in their pod spec, you define tiers like `high-priority`, `batch-jobs`, and `dev-default` once at the cluster level, and workloads just reference the name. This also gives operators a single control plane for reviewing and adjusting scheduling policy without touching individual deployments."

---

**Q: What is preemption in Kubernetes, and when does it fire?**

> "Preemption is the scheduler's mechanism for evicting lower-priority pods to make room for a higher-priority pod that cannot be scheduled. It fires specifically when a high-priority pod is pending and the scheduler has exhausted all options — no node has sufficient free resources to accommodate it. At that point, the scheduler simulates which lower-priority pods it could evict, selects the minimum set needed, and sends graceful delete requests for those pods. Importantly, preemption is entirely scheduler-driven — it does not fire due to runtime resource pressure like CPU spikes or memory usage on running pods. That's a completely separate mechanism handled by the kubelet using QoS classes."

---

**Q: What's the difference between Pod Priority and QoS class?**

> "They operate at different layers of the system and serve different purposes. Pod Priority, defined via PriorityClass, is a scheduling construct — it tells the scheduler which pods to prefer when placing workloads on nodes and which pods to evict if a higher-priority workload can't be scheduled. QoS class — Guaranteed, Burstable, or BestEffort — is a runtime construct automatically derived from a pod's CPU and memory requests and limits. It governs what the kubelet does when a node is under memory pressure and needs to evict running pods. A pod can have high scheduling priority but BestEffort QoS, which means it gets preferred at scheduling time but is first in line for kubelet eviction if the node runs out of memory."

---

**Q: What does `preemptionPolicy: Never` do, and when would you use it?**

> "Setting `preemptionPolicy: Never` on a PriorityClass means that pods using that class will be preferred over lower-priority pods in the scheduling queue when resources are available, but will never evict running pods to get scheduled. If the cluster is full, those pods stay Pending rather than disrupting existing workloads. I'd use it for things like logging agents, monitoring sidecars, or stateful batch jobs where disruption is worse than a scheduling delay. It's a way of saying: this workload is important, but not important enough to evict something else."

---

**Q: What happens to pods that are preempted?**

> "Preemption deletes pods gracefully — they go through their normal termination flow, respecting `terminationGracePeriodSeconds`. If the preempted pods are managed by a Deployment or ReplicaSet, the controller immediately creates replacement pods, which enter the scheduling queue as Pending. They'll reschedule if and when resources free up. So preemption doesn't destroy the workload design — it temporarily pauses lower-priority pods to let the critical workload run. If you need those lower-priority pods back, reducing the replica count of the high-priority deployment frees up resources."

---

**Q: Can preemption violate a PodDisruptionBudget?**

> "No. The scheduler respects PodDisruptionBudgets during preemption. If evicting a pod would bring a deployment below its `minAvailable` threshold, the scheduler will not choose that pod as a victim. It looks for other candidates. If no valid victim exists that satisfies all PDB constraints, the high-priority pod stays Pending rather than violating the budget. This makes PDBs a powerful safety net in priority-heavy clusters."

---

## 8. Debugging

### Fast diagnosis path for priority/preemption issues

```
Pod is Pending — is it a priority/preemption issue?
         │
kubectl describe pod <pod> | grep -i "reason\|priority\|preempt\|insufficient"
         │
         ├── "Insufficient cpu/memory" → scheduling failure
         │         │
         │         ├── Does it have a priorityClassName?
         │         │       kubectl get pod <pod> -o yaml | grep priorityClassName
         │         │       └── NO → set one; pod may be stuck behind lower-priority pods
         │         │
         │         ├── preemptionPolicy: Never on its class?
         │         │       kubectl get pc <classname> -o yaml | grep preemptionPolicy
         │         │       └── YES + cluster full → pod will stay Pending by design
         │         │
         │         └── Is preemption happening at all?
         │                 kubectl get events | grep -i preempt
         │                 └── No events → scheduler didn't find valid victims
         │                               → check PDB constraints
         │
         ├── "No nodes available" → all nodes filtered (taint/affinity)
         │         └── Different issue — check tolerations/affinity
         │
         └── Nothing obvious → check scheduler logs
                 kubectl -n kube-system logs <scheduler-pod> | grep <pod-name>
```

### Command toolkit

```bash
# 1. What priority does this pod actually have?
kubectl describe pod <pod> | grep -i "priority"
# Priority:             1000
# Priority Class Name:  high-priority

# 2. List all priority classes and their values
kubectl get pc -o custom-columns="NAME:.metadata.name,VALUE:.value,DEFAULT:.globalDefault,PREEMPTION:.preemptionPolicy"

# 3. Who is using a given priority class?
kubectl get pods -A -o jsonpath='{range .items[?(@.spec.priorityClassName=="high-priority")]}{.metadata.namespace}/{.metadata.name}{"\n"}{end}'

# 4. What events fired during scheduling / preemption?
kubectl get events --sort-by='.lastTimestamp' | grep -iE "preempt|evict|scheduled|failed"

# 5. Node resource allocation (see what's eating capacity)
kubectl describe node <node> | grep -A30 "Allocated resources"

# 6. Scheduler logs (if you have access)
kubectl -n kube-system logs -l component=kube-scheduler --tail=100 | grep -i "preempt\|evict\|priority"

# 7. Show all pending pods with their priority
kubectl get pods -A --field-selector=status.phase=Pending \
  -o custom-columns="NS:.metadata.namespace,NAME:.metadata.name,PRIORITY:.spec.priority,CLASS:.spec.priorityClassName"

# 8. Simulate: what would happen if I delete pod X?
kubectl get pod <pod> -o yaml | grep -i "requests"
# Then manually calculate: current node usage - pod requests = freed space
```

---

## 9. Kill Switch

**10-second recall — absolute minimum:**

```
PriorityClass → cluster-scoped → numeric value → higher = preferred

Priority   = scheduling ORDER (fires every cycle)
Preemption = eviction when a new pod CAN'T schedule (only then)

preemptionPolicy:
  PreemptLowerPriority (default) → evicts lower pods to fit
  Never                          → waits instead of evicting

User values: max 1,000,000,000
System:      2,000,000,000+ (don't touch)

globalDefault: true → only ONE per cluster
No class + no globalDefault → priority 0

ImagePullBackOff on high-priority pod? Priority won't help. Fix the image.
Pod Pending with preemptionPolicy: Never? It's waiting. That's correct.
```

---

## 10. Appendix

### PriorityClass field reference

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: <name>            # cluster-scoped, no namespace
value: <int32>            # range: -2147483648 to 1000000000
globalDefault: false      # true = applies to pods without priorityClassName (one per cluster)
preemptionPolicy: PreemptLowerPriority  # or: Never
description: "Human-readable description"
```

### Pod spec: using a PriorityClass

```yaml
spec:
  priorityClassName: high-priority   # references PriorityClass by name
  containers:
    - name: app
      resources:
        requests:
          cpu: "500m"       # Without requests, priority is irrelevant — scheduler can't measure fit
          memory: "128Mi"
```

### Quick-create commands

```bash
# View all priority classes
kubectl get pc
kubectl get priorityclasses   # long form

# Describe a specific class
kubectl get pc high-priority -o yaml

# Find all pods using a specific class
kubectl get pods -A -o jsonpath='{range .items[?(@.spec.priorityClassName=="high-priority")]}{.metadata.namespace}/{.metadata.name}{"\n"}{end}'

# Check what priority a running pod has
kubectl get pod <pod> -o jsonpath='{.spec.priority}'

# Delete a priority class (pods using it keep their numeric priority until rescheduled)
kubectl delete pc low-priority
```

### Priority class tier cheatsheet

| Name | Value | preemptionPolicy | globalDefault | Use case |
|---|---|---|---|---|
| `system-node-critical` | 2000001000 | PreemptLowerPriority | false | etcd, kube-apiserver (built-in) |
| `system-cluster-critical` | 2000000000 | PreemptLowerPriority | false | CoreDNS, kube-proxy (built-in) |
| `critical-prod` | 1000 | PreemptLowerPriority | false | Payment APIs, auth services |
| `standard-prod` | 500 | PreemptLowerPriority | false | Standard microservices |
| `batch-jobs` | 100 | Never | false | Nightly jobs, ML training |
| `dev-default` | 10 | Never | true | Dev/test workloads |

### Priority vs QoS — side-by-side

| Property | Pod Priority | QoS Class |
|---|---|---|
| Defined by | `PriorityClass` + `priorityClassName` | CPU/memory `requests` and `limits` |
| Who uses it | Scheduler | Kubelet |
| Affects | Scheduling order, preemption victims | Node-level eviction order |
| Set explicitly? | Yes (`priorityClassName`) | No (derived automatically) |
| Runtime eviction? | ❌ | ✅ |
| Scheduling priority? | ✅ | ❌ |

### Events to watch during preemption

```bash
kubectl get events --sort-by='.lastTimestamp' | grep -iE "preempt|evict|kill|backoff"

# Expected events during preemption:
# Preempting:    Preempted pod lp-nginx-xxx on node node1
# Killing:       Stopping container nginx
# Scheduled:     Successfully assigned hp-nginx-xxx to node1
```

---

