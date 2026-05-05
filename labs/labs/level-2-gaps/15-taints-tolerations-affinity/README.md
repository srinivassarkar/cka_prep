# Lab 15 — Taints, Tolerations & Node Affinity

## What This Lab Is About

Taints, tolerations, and node affinity are the three mechanisms that control which pods land on which nodes. In a small cluster they seem unnecessary. In a real prod cluster with GPU nodes, spot instances, high-memory nodes, and dedicated infra nodes — they are what keeps your ML workloads off your web servers, your batch jobs off your prod database nodes, and your monitoring agents running everywhere.

Get them wrong and you get pods stuck in `Pending` with cryptic scheduler messages, or pods landing on the wrong nodes entirely — consuming expensive GPU capacity for a basic nginx pod, or evicting production workloads from dedicated nodes they were supposed to have to themselves.

This lab covers the 5 most critical scheduling control scenarios. Taints that block all pods from landing on a node. Tolerations that allow specific pods through. Node affinity that pulls pods toward nodes with specific characteristics. The combined pattern used for dedicated node pools. And the systematic debug workflow for any `Pending` pod caused by scheduling constraints.

> The scheduler is not random. Every pod placement decision is deterministic — based on requests, taints, tolerations, and affinity rules. When a pod is `Pending`, the scheduler knows exactly why. Your job is to read what it's telling you.

---

## Environment

- **Cluster:** KIND (Kubernetes IN Docker)
- **K8s Version:** v1.27.3
- **Node:** Single node (`kind-control-plane`)
- **Namespace for this lab:** `lab15`

```bash
kubectl create namespace lab15
kubectl config set-context --current --namespace=lab15
```

### Important Note for KIND

KIND runs a single node. In real clusters, taints and affinity are used to target specific nodes in a pool. In KIND, all concepts apply identically — the only difference is that when the single node is tainted, ALL pods without the toleration become unschedulable. The scenarios below account for this. After each taint scenario, the taint is removed before proceeding so the cluster stays healthy.

---

## The 5 Scenarios

| # | Scenario | What You'll Learn |
|---|---|---|
| 01 | Node taint — no toleration | NoSchedule, NoExecute, PreferNoSchedule effects |
| 02 | Toleration — allowing pods through | Matching taints, effect specificity, wildcard tolerations |
| 03 | Node affinity — required and preferred | Hard vs soft scheduling constraints, label matching |
| 04 | Combined pattern — dedicated node pools | Taint + toleration + affinity together |
| 05 | Debug workflow — Pending from scheduling | Reading scheduler messages, fixing constraints |

---

## The Mental Model — Before Starting

```
Three mechanisms, three different owners:

TAINT (on the node) — NODE says "I don't want certain pods"
  kubectl taint node <name> key=value:effect

TOLERATION (on the pod) — POD says "I can handle that node's taint"
  spec.tolerations: [{key: "...", operator: "...", value: "...", effect: "..."}]

NODE AFFINITY (on the pod) — POD says "I want to run on nodes with these labels"
  spec.affinity.nodeAffinity: ...

Combined use case:
  Taint a GPU node → only GPU pods tolerate it → GPU pods also have affinity
  for the GPU node → result: GPU pods land on GPU nodes, nothing else does

Taints without tolerations = exclusion (node pushes pods away)
Affinity without taints   = preference (pod is pulled toward nodes)
Both together             = dedicated scheduling (exclusive node pools)
```

---

## Scenario 01 — Node Taint — No Toleration

### What You'll Break

A pod that cannot schedule because the node has a taint the pod doesn't tolerate. Three taint effects exist — each behaves differently. Understanding all three and their error messages is the foundation of scheduling debug.

### The Three Taint Effects

```
NoSchedule
  → New pods WITHOUT matching toleration will NOT be scheduled on this node
  → Existing pods already running: UNAFFECTED
  → Most common taint for dedicated nodes

PreferNoSchedule
  → Scheduler tries to AVOID placing pods without toleration here
  → If no other node is available: scheduler places the pod here anyway
  → Soft preference — never causes Pending

NoExecute
  → New pods WITHOUT matching toleration will NOT be scheduled
  → Existing pods WITHOUT matching toleration are EVICTED immediately
  → With tolerationSeconds: evicted after that many seconds
  → Most aggressive effect — used for node maintenance and failure
```

### Apply NoSchedule Taint

```bash
# Add a taint simulating a dedicated GPU node
kubectl taint node kind-control-plane \
  dedicated=gpu:NoSchedule

# Verify taint is applied
kubectl describe node kind-control-plane | grep -A 3 "Taints:"
# Taints: dedicated=gpu:NoSchedule

# Deploy a pod WITHOUT toleration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-toleration-pod
  namespace: lab15
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
EOF

kubectl get pod no-toleration-pod -n lab15
# NAME                 READY   STATUS    RESTARTS   AGE
# no-toleration-pod    0/1     Pending   0          15s
```

### Read the Scheduler Message

```bash
kubectl describe pod no-toleration-pod -n lab15 | grep -A 8 "Events:"
# Warning  FailedScheduling  default-scheduler
# 0/1 nodes are available: 1 node(s) had untolerated taint
# {dedicated: gpu}. preemption: 0/1 nodes are available:
# 1 Preemption is not helpful for scheduling.

# Decode the message:
# "1 node(s) had untolerated taint {dedicated: gpu}"
# → 1 node exists, it has taint "dedicated=gpu:NoSchedule"
# → Pod has no matching toleration → cannot schedule
```

### Observe NoSchedule vs Existing Pods

```bash
# Existing pods running BEFORE the taint are NOT evicted by NoSchedule
kubectl get pods -n kube-system
# CoreDNS, kube-proxy etc: still Running
# NoSchedule only affects NEW scheduling decisions
# Compare with NoExecute which evicts existing pods too

# Remove the taint before next scenario
kubectl taint node kind-control-plane dedicated=gpu:NoSchedule-
# The trailing "-" removes the taint

kubectl get pod no-toleration-pod -n lab15
# Now schedules — taint removed
kubectl delete pod no-toleration-pod -n lab15
```

### Apply NoExecute Taint

```bash
# Deploy a pod first, then apply NoExecute taint
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: will-be-evicted
  namespace: lab15
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
EOF

kubectl wait --for=condition=ready pod/will-be-evicted -n lab15 --timeout=30s
kubectl get pod will-be-evicted -n lab15
# Running

# Apply NoExecute taint — existing pod will be evicted
kubectl taint node kind-control-plane \
  maintenance=true:NoExecute

# Watch the pod get evicted
kubectl get pod will-be-evicted -n lab15 -w
# Running → Terminating → gone
# Pod is evicted immediately because it has no toleration for this taint

# Remove taint (IMPORTANT — this taint evicts ALL pods including system pods in KIND)
kubectl taint node kind-control-plane maintenance=true:NoExecute-

# Verify cluster is recovering
kubectl get pods -n kube-system -w
# System pods restart — wait for them to stabilize
```

### Apply NoExecute With tolerationSeconds

```bash
# pods can tolerate a taint for a limited time before being evicted
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: tolerates-briefly
  namespace: lab15
spec:
  tolerations:
  - key: "maintenance"
    operator: "Equal"
    value: "true"
    effect: "NoExecute"
    tolerationSeconds: 60      # Pod tolerates this taint for 60 seconds
                                # After 60s: evicted even though it has a toleration
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
EOF

kubectl wait --for=condition=ready pod/tolerates-briefly -n lab15 --timeout=30s

# Apply NoExecute taint
kubectl taint node kind-control-plane maintenance=true:NoExecute

# Pod stays running for 60 seconds then gets evicted
kubectl get pod tolerates-briefly -n lab15 -w
# Running → (60 seconds later) → Terminating

# Remove taint immediately for cluster stability
kubectl taint node kind-control-plane maintenance=true:NoExecute-
```

### Taint Reference — All Valid Formats

```bash
# Add a taint
kubectl taint node <node-name> key=value:effect
kubectl taint node <node-name> key:effect         # No value — key-only taint

# Remove a taint (trailing dash)
kubectl taint node <node-name> key=value:effect-
kubectl taint node <node-name> key:effect-
kubectl taint node <node-name> key-               # Remove all taints with this key

# View taints on a node
kubectl describe node <node-name> | grep -A 3 "Taints:"
kubectl get node <node-name> -o jsonpath='{.spec.taints}' | jq '.'

# View all node taints in cluster
kubectl get nodes -o json | \
  jq '.items[] | {name: .metadata.name, taints: .spec.taints}'
```

### Prod Wisdom

**`NoSchedule` is the right default taint for dedicated nodes — not `NoExecute`.** `NoExecute` is aggressive — it evicts existing pods immediately and is designed for node failure scenarios, not node dedication. When you want a node reserved for specific workloads, taint it with `NoSchedule` — new pods without the toleration won't land there, and whatever is currently running is not disrupted. Reserve `NoExecute` for actual node failures and controlled maintenance drain operations.

---

## Scenario 02 — Toleration — Allowing Pods Through

### What You'll Build

The correct toleration configuration to allow specific pods through node taints. Including exact match, exists operator, wildcard tolerations, and the critical difference between tolerating a specific taint vs tolerating all taints.

### Setup — Apply a Taint

```bash
kubectl taint node kind-control-plane \
  workload-type=gpu:NoSchedule
```

### Exact Match Toleration

```bash
# Toleration that exactly matches the taint
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: exact-toleration-pod
  namespace: lab15
spec:
  tolerations:
  - key: "workload-type"         # Must match taint key exactly
    operator: "Equal"            # Use Equal when providing a value
    value: "gpu"                 # Must match taint value exactly
    effect: "NoSchedule"         # Must match taint effect exactly
  containers:
  - name: app
    image: nvidia/cuda:11.0-base # Simulating a GPU workload
    command: ["sleep", "3600"]
    resources:
      requests:
        memory: "64Mi"
        cpu: "100m"
      limits:
        memory: "128Mi"
        cpu: "200m"
EOF

kubectl get pod exact-toleration-pod -n lab15
# Running — toleration matches taint exactly
```

### Exists Operator — Key-Only Toleration

```bash
# Tolerates any taint with the key "workload-type" regardless of value
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: exists-toleration-pod
  namespace: lab15
spec:
  tolerations:
  - key: "workload-type"
    operator: "Exists"           # Matches ANY value for this key
    effect: "NoSchedule"         # Still specific to NoSchedule effect
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
EOF

kubectl get pod exists-toleration-pod -n lab15
# Running — Exists matches "workload-type=gpu" regardless of value
```

### Wildcard — Tolerate ALL Taints

```bash
# A pod that tolerates every taint on every node
# USE WITH CAUTION — this schedules on ANY node including tainted ones
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: wildcard-toleration-pod
  namespace: lab15
spec:
  tolerations:
  - operator: "Exists"           # No key specified = match ANY key
                                  # No effect specified = match ANY effect
                                  # This pod tolerates EVERYTHING
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
EOF

kubectl get pod wildcard-toleration-pod -n lab15
# Running — wildcard toleration bypasses all node taints
```

### When Toleration Effect Matters

```bash
# Toleration without effect specified = matches ALL effects for that key
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: any-effect-pod
  namespace: lab15
spec:
  tolerations:
  - key: "workload-type"
    operator: "Exists"
    # No effect: = tolerates NoSchedule, PreferNoSchedule, AND NoExecute
    # for the "workload-type" key
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
EOF

kubectl get pod any-effect-pod -n lab15
# Running
```

### The Toleration Matching Rules

```
Toleration matches a Taint when ALL of the following are true:

1. key matches OR toleration has no key (wildcard)
2. operator is Equal AND value matches
   OR operator is Exists (value ignored)
3. effect matches OR toleration has no effect (matches all effects)

Examples:
  Taint: dedicated=gpu:NoSchedule

  Toleration A: key=dedicated, op=Equal, value=gpu, effect=NoSchedule
    → MATCHES (exact match)

  Toleration B: key=dedicated, op=Exists, effect=NoSchedule
    → MATCHES (Exists ignores value)

  Toleration C: key=dedicated, op=Equal, value=gpu (no effect)
    → MATCHES (no effect = all effects)

  Toleration D: key=dedicated, op=Equal, value=cpu, effect=NoSchedule
    → NO MATCH (value mismatch: gpu ≠ cpu)

  Toleration E: key=workload, op=Equal, value=gpu, effect=NoSchedule
    → NO MATCH (key mismatch: dedicated ≠ workload)

  Toleration F: op=Exists (no key, no effect)
    → MATCHES (wildcard — matches everything)
```

```bash
# Clean up taint before next scenario
kubectl taint node kind-control-plane workload-type=gpu:NoSchedule-

# Clean up pods
kubectl delete pods --all -n lab15
```

### Prod Wisdom

**Wildcard tolerations (`operator: Exists` with no key) should never appear in regular application pods.** They are appropriate only for DaemonSets that must run on every node (monitoring agents, log collectors, network plugins) — because DaemonSets intentionally need to run even on tainted nodes. If you see a wildcard toleration on a regular Deployment, it means someone worked around a scheduling problem by bypassing all taints — which defeats the purpose of having dedicated nodes. Fix the specific toleration instead.

---

## Scenario 03 — Node Affinity — Required and Preferred

### What You'll Build

Node affinity rules on pods that pull them toward nodes with specific labels. Two modes — `requiredDuringSchedulingIgnoredDuringExecution` (hard constraint — pod stays Pending if no match) and `preferredDuringSchedulingIgnoredDuringExecution` (soft preference — falls back to any node if no match).

### Label the Node First

```bash
# In a real cluster you have nodes with different labels
# Kind only has one node — we label it to simulate different node types
kubectl label node kind-control-plane \
  node-type=compute \
  region=us-east-1 \
  disk=ssd \
  gpu=true

# Verify labels
kubectl get node kind-control-plane --show-labels
```

### Required Affinity — Hard Constraint

```bash
# Pod MUST land on a node with disk=ssd label
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: required-affinity-pod
  namespace: lab15
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disk
            operator: In
            values:
            - ssd
            - nvme                # OR — either ssd or nvme disk
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
EOF

kubectl get pod required-affinity-pod -n lab15
# Running — node has disk=ssd label, affinity satisfied
```

### Break It — Required Affinity With No Match

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-match-affinity-pod
  namespace: lab15
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-type
            operator: In
            values:
            - high-memory          # No node has node-type=high-memory
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
EOF

kubectl get pod no-match-affinity-pod -n lab15
# STATUS: Pending

kubectl describe pod no-match-affinity-pod -n lab15 | grep -A 5 "Events:"
# Warning  FailedScheduling  default-scheduler
# 0/1 nodes are available: 1 node(s) didn't match Pod's node affinity/selector.
```

### Preferred Affinity — Soft Constraint

```bash
# Pod PREFERS nodes with gpu=true but will schedule anywhere if none match
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: preferred-affinity-pod
  namespace: lab15
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80               # Higher weight = stronger preference (1-100)
        preference:
          matchExpressions:
          - key: gpu
            operator: In
            values:
            - "true"
      - weight: 20               # Lower weight = weaker preference
        preference:
          matchExpressions:
          - key: region
            operator: In
            values:
            - us-east-1
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
EOF

kubectl get pod preferred-affinity-pod -n lab15
# Running — schedules even if no gpu node exists because it's preferred not required

# Fix the required affinity pod from above
kubectl delete pod no-match-affinity-pod -n lab15

# Option A: Add the required label to the node
kubectl label node kind-control-plane node-type=high-memory

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-match-affinity-pod
  namespace: lab15
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-type
            operator: In
            values:
            - high-memory
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
EOF

kubectl get pod no-match-affinity-pod -n lab15
# Running now — node has the required label
```

### All Affinity Operators

```yaml
# In matchExpressions:
operator: In              # Node label value is in the list
  values: [ssd, nvme]

operator: NotIn           # Node label value is NOT in the list
  values: [hdd]           # Avoid spinning disk nodes

operator: Exists          # Node has this label key (any value)
  # no values field

operator: DoesNotExist    # Node does NOT have this label key
  # no values field

operator: Gt              # Node label value > specified value (numeric strings)
  values: ["4"]           # Nodes with more than 4 (e.g., num-gpus > 4)

operator: Lt              # Node label value < specified value
  values: ["8"]

# Multiple matchExpressions in same nodeSelectorTerm = AND logic
# Multiple nodeSelectorTerms = OR logic
nodeSelectorTerms:
- matchExpressions:         # Term 1: disk=ssd AND region=us-east-1
  - key: disk
    operator: In
    values: [ssd]
  - key: region
    operator: In
    values: [us-east-1]
- matchExpressions:         # OR Term 2: disk=nvme (any region)
  - key: disk
    operator: In
    values: [nvme]
```

### IgnoredDuringExecution — What It Means

```
requiredDuringScheduling IgnoredDuringExecution
  → "Required" = pod won't schedule if no match (hard)
  → "IgnoredDuringExecution" = if node labels CHANGE after scheduling,
    the pod is NOT evicted (it stays running)
  → The affinity rule only applies at scheduling time

preferredDuringScheduling IgnoredDuringExecution
  → "Preferred" = scheduler tries to match but falls back (soft)
  → "IgnoredDuringExecution" = same — labels changing after schedule = no eviction

There is no "RequiredDuringExecution" in stable K8s yet
(would evict pods if node labels change to no longer match)
```

### Prod Wisdom

**Use `required` affinity only when the constraint is truly mandatory** — ML training that literally cannot run without GPU hardware, for example. For everything else, use `preferred` affinity. A `required` affinity that has no matching nodes causes `Pending` pods — which is a prod outage if the label is missing or mislabeled. `Preferred` affinity with weight fails gracefully by scheduling anywhere. Start with preferred, move to required only when you have absolute confidence in your node labeling pipeline.

---

## Scenario 04 — Combined Pattern — Dedicated Node Pools

### What You'll Build

The production pattern for dedicated node pools — combining taints, tolerations, and node affinity to ensure specific workloads land on specific nodes AND no other workloads share those nodes.

### Why You Need Both Taint AND Affinity

```
Taint alone (without affinity):
  → Other pods stay off the tainted node ✓
  → But your GPU pods might still land on non-GPU nodes ✗
  → Taint keeps others out but doesn't pull your pods in

Affinity alone (without taint):
  → Your GPU pods are pulled toward GPU nodes ✓
  → But other pods can also land on GPU nodes ✗
  → Affinity attracts your pods but doesn't keep others out

Taint + Toleration + Affinity (the complete pattern):
  → Taint keeps other pods off GPU nodes ✓
  → Toleration lets GPU pods onto GPU nodes ✓
  → Affinity ensures GPU pods actually land on GPU nodes ✓
  → Result: GPU nodes reserved exclusively for GPU workloads ✓
```

### Apply the Complete Pattern

```bash
# Step 1 — Label and taint the node (in real cluster: done when node joins)
kubectl label node kind-control-plane \
  node-pool=gpu-pool \
  accelerator=nvidia-tesla-v100

kubectl taint node kind-control-plane \
  node-pool=gpu-pool:NoSchedule

# Step 2 — Deploy a GPU workload with correct toleration + affinity
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-training-job
  namespace: lab15
spec:
  replicas: 2
  selector:
    matchLabels:
      app: gpu-training
  template:
    metadata:
      labels:
        app: gpu-training
    spec:
      # Toleration — allows this pod through the node taint
      tolerations:
      - key: "node-pool"
        operator: "Equal"
        value: "gpu-pool"
        effect: "NoSchedule"

      # Affinity — ensures this pod PREFERS (or requires) the GPU node
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-pool
                operator: In
                values:
                - gpu-pool        # Must land on gpu-pool nodes

      containers:
      - name: training
        image: busybox:1.35
        command: ["sh", "-c"]
        args:
        - |
          echo "GPU training job running on $(hostname)"
          echo "Node: $(cat /etc/hostname)"
          sleep 3600
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
EOF

kubectl rollout status deployment/gpu-training-job -n lab15

# Step 3 — Try to deploy a regular pod WITHOUT toleration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: regular-workload
  namespace: lab15
spec:
  # No toleration — cannot land on gpu-pool node
  # No affinity — scheduler would try any node
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
EOF

kubectl get pod regular-workload -n lab15
# STATUS: Pending — taint blocks it from the only node (KIND single node)
# In a multi-node cluster: regular pod lands on non-GPU nodes

kubectl describe pod regular-workload -n lab15 | grep -A 5 "Events:"
# 0/1 nodes are available: 1 node(s) had untolerated taint {node-pool: gpu-pool}

# GPU training job is running, regular workload cannot reach the GPU node
kubectl get pods -n lab15
# gpu-training-job-xxx   1/1   Running  ← on gpu-pool node
# regular-workload        0/1   Pending  ← blocked from gpu-pool node
```

### Real-World Node Pool Patterns

```yaml
# Pattern 1 — Spot/Preemptible node pool
# Node labels:  cloud.google.com/gke-spot=true
# Node taint:   cloud.google.com/gke-spot=true:NoSchedule

# Pod toleration (only batch/fault-tolerant workloads):
tolerations:
- key: cloud.google.com/gke-spot
  operator: Equal
  value: "true"
  effect: NoSchedule

# Pod affinity (prefer spot nodes for cost savings):
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      preference:
        matchExpressions:
        - key: cloud.google.com/gke-spot
          operator: In
          values: ["true"]

---
# Pattern 2 — Dedicated high-memory nodes for databases
# Node labels:  node-type=high-memory, memory=large
# Node taint:   dedicated=database:NoSchedule

tolerations:
- key: dedicated
  operator: Equal
  value: database
  effect: NoSchedule

affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: node-type
          operator: In
          values: [high-memory]

---
# Pattern 3 — Infra/system node (control-plane style)
# DaemonSets need to run on ALL nodes including tainted ones
# DaemonSet pods get default tolerations for master/infra taints

tolerations:
- operator: Exists  # Wildcard — tolerate all taints
# Appropriate ONLY for DaemonSets (monitoring, logging, CNI)
```

```bash
# Clean up taint after scenario
kubectl taint node kind-control-plane node-pool=gpu-pool:NoSchedule-
kubectl delete pod regular-workload -n lab15
```

### Prod Wisdom

**The taint+toleration+affinity triple is the only complete solution for node pool isolation.** Taint alone keeps pods off — but doesn't guarantee your workloads land there. Affinity alone pulls your workloads there — but doesn't keep others off. You need both. In cloud environments (EKS, GKE, AKS), managed node pools automatically apply taints and labels when you create a pool. Your job is to add the correct toleration and affinity to pods that belong there. Missing the affinity means your GPU pods might land on regular nodes — wasting expensive GPU capacity and potentially starving the GPU node.

---

## Scenario 05 — Debug Workflow — Pending From Scheduling Constraints

### What You'll Build

The complete systematic debug workflow for any pod stuck in `Pending` because of taints, tolerations, or affinity. The scheduler tells you exactly what went wrong — you just need to know how to read its message.

### Apply Multiple Broken Scenarios

```bash
# Set up a complex environment with multiple issues

# Label and taint the node
kubectl label node kind-control-plane \
  environment=production \
  tier=frontend

kubectl taint node kind-control-plane \
  environment=production:NoSchedule

# Pod 1 — wrong taint value in toleration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: wrong-taint-value
  namespace: lab15
spec:
  tolerations:
  - key: "environment"
    operator: "Equal"
    value: "staging"            # WRONG: taint is "production", not "staging"
    effect: "NoSchedule"
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
EOF

# Pod 2 — wrong affinity label key
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: wrong-affinity-label
  namespace: lab15
spec:
  tolerations:
  - key: "environment"
    operator: "Equal"
    value: "production"
    effect: "NoSchedule"
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: tier
            operator: In
            values:
            - backend            # WRONG: node has tier=frontend, not backend
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
EOF

# Pod 3 — missing toleration entirely
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: missing-toleration
  namespace: lab15
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: environment
            operator: In
            values:
            - production
  # Has affinity for the node BUT no toleration for the taint
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
EOF
```

### The Debug Workflow

```bash
# ALL THREE PODS ARE PENDING — systematic debug for each

# ============================================================
# DEBUG POD 1: wrong-taint-value
# ============================================================

kubectl describe pod wrong-taint-value -n lab15 | grep -A 8 "Events:"
# 0/1 nodes are available: 1 node(s) had untolerated taint {environment: production}
# Reason: taint not tolerated

# Check what taint the node has
kubectl describe node kind-control-plane | grep -A 5 "Taints:"
# Taints: environment=production:NoSchedule

# Check what toleration the pod has
kubectl get pod wrong-taint-value -n lab15 \
  -o jsonpath='{.spec.tolerations}' | jq '.'
# value: "staging"  ← value mismatch: taint has "production", toleration has "staging"

# Fix: correct the toleration value
kubectl delete pod wrong-taint-value -n lab15
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: wrong-taint-value
  namespace: lab15
spec:
  tolerations:
  - key: "environment"
    operator: "Equal"
    value: "production"          # Fixed: matches taint value
    effect: "NoSchedule"
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
EOF
kubectl get pod wrong-taint-value -n lab15
# Running

# ============================================================
# DEBUG POD 2: wrong-affinity-label
# ============================================================

kubectl describe pod wrong-affinity-label -n lab15 | grep -A 8 "Events:"
# 0/1 nodes are available: 1 node(s) didn't match Pod's node affinity/selector
# Reason: affinity mismatch (taint is tolerated but affinity fails)

# Check node labels
kubectl get node kind-control-plane --show-labels
# tier=frontend  ← node has tier=frontend

# Check pod affinity
kubectl get pod wrong-affinity-label -n lab15 \
  -o jsonpath='{.spec.affinity}' | jq '.'
# values: ["backend"]  ← pod wants tier=backend, node has tier=frontend

# Fix: correct the affinity value
kubectl delete pod wrong-affinity-label -n lab15
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: wrong-affinity-label
  namespace: lab15
spec:
  tolerations:
  - key: "environment"
    operator: "Equal"
    value: "production"
    effect: "NoSchedule"
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: tier
            operator: In
            values:
            - frontend           # Fixed: matches node label tier=frontend
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
EOF
kubectl get pod wrong-affinity-label -n lab15
# Running

# ============================================================
# DEBUG POD 3: missing-toleration
# ============================================================

kubectl describe pod missing-toleration -n lab15 | grep -A 8 "Events:"
# 0/1 nodes are available: 1 node(s) had untolerated taint {environment: production}
# Same message as Pod 1 — taint not tolerated
# But this pod has correct affinity — the problem is the missing toleration

# Check the pod spec
kubectl get pod missing-toleration -n lab15 \
  -o jsonpath='{.spec.tolerations}' | jq '.'
# [] or null — no tolerations defined

# Fix: add the required toleration
kubectl delete pod missing-toleration -n lab15
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: missing-toleration
  namespace: lab15
spec:
  tolerations:
  - key: "environment"           # Added: required toleration
    operator: "Equal"
    value: "production"
    effect: "NoSchedule"
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: environment
            operator: In
            values:
            - production
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
EOF
kubectl get pod missing-toleration -n lab15
# Running
```

### The Complete Scheduler Message Decoder

```
Message format:
"0/N nodes are available: X node(s) <reason>"

Where N = total nodes, X = nodes with this specific rejection reason
Multiple reasons can appear for the same nodes.

REASON                                    ROOT CAUSE              FIX
──────────────────────────────────────────────────────────────────────────────────
"had untolerated taint {key: value}"     Taint not tolerated     Add/fix toleration
"didn't match Pod's node affinity"       Affinity no match       Fix affinity or label node
"Insufficient cpu/memory"               Resource requests        Reduce requests or add nodes
"didn't match Pod's node selector"      nodeSelector mismatch   Fix nodeSelector or label node
"had taint {node.k8s.io/not-ready}"     Node NotReady           Fix the node
"had taint {node.k8s.io/unreachable}"   Node unreachable        Network/node issue
"didn't have free ports for the pod"    Host port conflict       Change hostPort
"Preemption is not helpful"             Can't preempt           Adjust priority/resources
"PodFitsHostPorts"                      Port conflict           Remove hostPort

Reading multi-reason messages:
"0/3 nodes are available: 1 Insufficient memory, 2 node(s) had untolerated taint"
→ 3 total nodes: 1 rejected for memory, 2 rejected for taint
→ Fix: add toleration AND reduce memory request
```

### The Complete Scheduling Debug Sequence

```bash
# When a pod is Pending — run in this order every time

# STEP 1 — Read the scheduler message (everything you need is here)
kubectl describe pod <name> -n <namespace> | grep -A 10 "Events:"

# STEP 2 — If taint-related: check node taints
kubectl describe node <node-name> | grep -A 5 "Taints:"
kubectl get nodes -o json | jq '.items[] | {name: .metadata.name, taints: .spec.taints}'

# STEP 3 — If taint-related: check pod tolerations
kubectl get pod <name> -n <namespace> \
  -o jsonpath='{.spec.tolerations}' | jq '.'

# STEP 4 — If affinity-related: check node labels
kubectl get node --show-labels
kubectl get node <node-name> -o jsonpath='{.metadata.labels}' | jq '.'

# STEP 5 — If affinity-related: check pod affinity rules
kubectl get pod <name> -n <namespace> \
  -o jsonpath='{.spec.affinity}' | jq '.'

# STEP 6 — If resource-related: check node capacity
kubectl describe node <node-name> | grep -A 10 "Allocated resources:"

# STEP 7 — Check if the pod itself has scheduling constraints
kubectl get pod <name> -n <namespace> -o yaml | \
  grep -A 50 "tolerations\|affinity\|nodeSelector\|nodeName"

# STEP 8 — Simulate scheduling decision (useful for complex policies)
# Check which nodes a pod WOULD schedule on
kubectl get nodes --show-labels | grep <required-label>
```

---

## Key Commands Reference — Lab 15

```bash
# Taint operations
kubectl taint node <node> key=value:effect          # Add taint
kubectl taint node <node> key=value:effect-         # Remove specific taint
kubectl taint node <node> key-                      # Remove all taints with key
kubectl describe node <node> | grep -A 3 "Taints:"  # View taints
kubectl get node <node> -o jsonpath='{.spec.taints}' | jq '.'

# Node label operations
kubectl label node <node> key=value                 # Add label
kubectl label node <node> key-                      # Remove label
kubectl get nodes --show-labels                     # View all node labels
kubectl get node <node> -o jsonpath='{.metadata.labels}' | jq '.'

# Pod scheduling inspection
kubectl get pod <name> -n <ns> -o jsonpath='{.spec.tolerations}' | jq '.'
kubectl get pod <name> -n <ns> -o jsonpath='{.spec.affinity}' | jq '.'
kubectl get pod <name> -n <ns> -o jsonpath='{.spec.nodeSelector}'
kubectl get pod <name> -n <ns> -o jsonpath='{.spec.nodeName}'

# Scheduler messages
kubectl describe pod <name> -n <ns> | grep -A 10 "Events:"

# Find which node a pod landed on
kubectl get pod <name> -n <ns> -o wide

# Find all pods on a specific node
kubectl get pods -A --field-selector spec.nodeName=<node-name>

# Check if a node would accept a pod (dry-run approach)
# Attempt to create the pod with --dry-run=server
kubectl apply -f pod.yaml --dry-run=server -n <namespace>
```

---

## The Scheduling Decision Summary

```
When you submit a pod, the scheduler evaluates every node:

Phase 1 — FILTERING (eliminates nodes that CAN'T run the pod)
  ✓ Node has enough CPU (requests fit)
  ✓ Node has enough memory (requests fit)
  ✓ Pod tolerates all node taints
  ✓ Pod's nodeSelector labels match node
  ✓ Pod's required nodeAffinity matches node
  ✓ Node has required ports free (hostPort)
  ✓ Node is Ready and not cordoned

Phase 2 — SCORING (ranks remaining nodes by preference)
  + Preferred nodeAffinity weight
  + Least-loaded node (spread workloads)
  + Pod affinity/anti-affinity
  + Data locality

If Phase 1 eliminates ALL nodes → pod stays Pending
If Phase 1 passes → pod scheduled on highest-score node from Phase 2
```

---

## Prod Wisdom — The Senior Engineer Mindset

Three things that define senior-level scheduling control:

**1. They always read the full scheduler message — every word.** `0/3 nodes: 2 Insufficient memory, 1 untolerated taint` tells you exactly what to fix and on how many nodes. Most engineers stop at the first reason. Senior engineers read all reasons in the message — a pod might have two problems simultaneously, and fixing only one still leaves it Pending.

**2. They use required affinity only when truly required, preferred otherwise.** `requiredDuringScheduling` with no matching nodes = Pending = outage. If your node labels are managed by an automated pipeline that could fail or mislabel, `preferred` affinity degrades gracefully. `required` affinity fails hard. Know which failure mode is acceptable for your workload before choosing.

**3. They test scheduling constraints before deploying to prod.** Apply the pod with `--dry-run=server` and check the Events. Label a staging node with production labels and verify the pod lands there. Run `kubectl describe node` to confirm taints are applied correctly. The 2-minute pre-deploy verification prevents the 20-minute prod debug session.

The scheduling constraint debug order:
```
kubectl describe pod → Events → read full scheduler message
kubectl describe node → Taints section (taint problem?)
kubectl get node --show-labels → Labels section (affinity problem?)
kubectl get pod -o jsonpath tolerations → toleration values correct?
kubectl get pod -o jsonpath affinity → affinity operators and values correct?
```

---

## Cleanup

```bash
# Remove all labels and taints added during this lab
kubectl taint node kind-control-plane \
  dedicated=gpu:NoSchedule- \
  workload-type=gpu:NoSchedule- \
  maintenance=true:NoExecute- \
  environment=production:NoSchedule- \
  node-pool=gpu-pool:NoSchedule- \
  2>/dev/null || true

kubectl label node kind-control-plane \
  node-type- \
  region- \
  disk- \
  gpu- \
  accelerator- \
  node-pool- \
  tier- \
  environment- \
  2>/dev/null || true

# Delete namespace
kubectl delete namespace lab15
```

---

Level 2 Complete.
