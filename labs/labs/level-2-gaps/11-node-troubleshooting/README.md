# Lab 11 — Node Troubleshooting

## What This Lab Is About

Nodes are the machines your pods run on. When a node has a problem, it's not one pod that's affected — it's every pod on that node. A node going `NotReady` is a cluster-wide event. Pods get evicted, rescheduled, or stuck in `Terminating`. Depending on the failure mode, the impact can range from degraded performance to a total outage for every workload on that node.

This lab covers the 5 most critical node failure scenarios in real clusters. A node in `NotReady` state and how to diagnose why. Cordoning and draining a node safely before maintenance. What happens to pods when a node disappears and how eviction timers work. Node resource pressure causing pod evictions. And the systematic node debug workflow that senior engineers run every time a node alert fires.

> When a pod fails, one app is affected. When a node fails, every app on it is affected. Node troubleshooting is cluster-level triage — the blast radius is bigger and the urgency is higher.

---

## Environment

- **Cluster:** KIND (Kubernetes IN Docker)
- **K8s Version:** v1.27.3
- **Node:** Single node (`kind-control-plane`)
- **Namespace for this lab:** `lab11`

```bash
kubectl create namespace lab11
kubectl config set-context --current --namespace=lab11
```

### Important Note for KIND

KIND runs a single-node cluster. Several scenarios in this lab simulate multi-node behaviours — node pressure, node failure, cordoning — that in real clusters affect specific nodes. In KIND, since there is only one node, some operations (like drain) require `--ignore-daemonsets` and will evict all pods. The commands and concepts are identical to real multi-node clusters — the blast radius is just the whole cluster in KIND.

---

## The 5 Scenarios

| # | Scenario | What You'll Learn |
|---|---|---|
| 01 | Node in `NotReady` — diagnosis workflow | Conditions, kubelet, container runtime |
| 02 | Cordon and Drain — safe maintenance | Node isolation, pod eviction, PodDisruptionBudgets |
| 03 | Node disappears — pod eviction timers | `node.kubernetes.io/unreachable`, tolerations, eviction |
| 04 | Node resource pressure — pod evictions | MemoryPressure, DiskPressure, OOM eviction order |
| 05 | Full node debug workflow | Systematic top-down node investigation |

---

## Scenario 01 — Node in `NotReady` — Diagnosis Workflow

### What You'll Break (Simulate)

A node that goes `NotReady`. In a real cluster this happens when the kubelet stops reporting to the API server — network partition, kubelet crash, OOM on the node, disk full, container runtime crash. We simulate the diagnosis workflow since we can't safely crash KIND's only node.

### Observe a Healthy Node First

```bash
kubectl get nodes
# NAME                 STATUS   ROLES           AGE   VERSION
# kind-control-plane   Ready    control-plane   6d    v1.27.3

kubectl describe node kind-control-plane
# The describe output has several critical sections — learn them all
```

### Reading Node Conditions — The Health Signal

```bash
kubectl get node kind-control-plane \
  -o jsonpath='{.status.conditions}' | jq '.'
```

Or more readably:

```bash
kubectl describe node kind-control-plane | grep -A 20 "Conditions:"
```

You will see:

```
Conditions:
  Type                 Status   Reason                       Message
  ──────────────────────────────────────────────────────────────────
  MemoryPressure       False    KubeletHasSufficientMemory   kubelet has sufficient memory
  DiskPressure         False    KubeletHasSufficientDisk     kubelet has sufficient disk space
  PIDPressure          False    KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True     KubeletReady                 kubelet is posting ready status
```

```
Condition          Status: True means           Status: False means
──────────────────────────────────────────────────────────────────
MemoryPressure     Node is LOW on memory        Node memory is fine
DiskPressure       Node is LOW on disk          Node disk is fine
PIDPressure        Node is LOW on PIDs          PID count is fine
Ready              Node is healthy              Node is NotReady ← alert
```

When `Ready` is `False` or `Unknown`, the node is not healthy.

### What NotReady Looks Like and Why

```bash
# In a real cluster, a NotReady node shows as:
# NAME          STATUS     ROLES    AGE   VERSION
# worker-node   NotReady   <none>   2d    v1.27.3

# The most common causes and their signals:

# CAUSE 1: kubelet stopped
# Signal: Ready condition goes Unknown (API server lost contact)
# Last heartbeat timestamp stops updating
kubectl get node <node-name> \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].lastHeartbeatTime}'
# Timestamp frozen = kubelet is dead or unreachable

# CAUSE 2: Container runtime (containerd/docker) crashed
# Signal: kubelet is running but can't manage containers
# Pods stuck in ContainerCreating forever
# kubelet logs show: "container runtime is not running"

# CAUSE 3: Disk full
# Signal: DiskPressure = True
# Evictions start, new pods won't schedule

# CAUSE 4: OOM on the node (kernel OOM killer)
# Signal: Node goes NotReady suddenly
# Node system logs show OOM kill messages

# CAUSE 5: Network partition
# Signal: Node shows Unknown from API server perspective
# But node itself may be healthy — just unreachable
```

### The NotReady Diagnosis Steps

```bash
# From the control plane (what we can do in KIND or any cluster):

# Step 1 — Which nodes are NotReady?
kubectl get nodes
kubectl get nodes --field-selector status.conditions[?(@.type=="Ready")].status=Unknown

# Step 2 — What are the node conditions?
kubectl describe node <node-name> | grep -A 10 "Conditions:"

# Step 3 — When did it last heartbeat?
kubectl get node <node-name> \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].lastHeartbeatTime}'

# Step 4 — What pods are on this node and what state are they in?
kubectl get pods -A --field-selector spec.nodeName=<node-name>

# Step 5 — Events on the node
kubectl describe node <node-name> | grep -A 20 "Events:"

# Step 6 — Node capacity and allocations
kubectl describe node <node-name> | grep -A 15 "Allocated resources:"

# On the node itself (SSH required in real clusters):
# sudo systemctl status kubelet
# sudo journalctl -u kubelet -n 100 --no-pager
# sudo systemctl status containerd
# df -h                          # Disk usage
# free -m                        # Memory
# top                            # CPU/memory live
# dmesg | grep -i "oom\|kill"    # OOM events
```

### Simulate NotReady Effects on Pods

```bash
# Deploy a workload to observe what happens to pods when node goes NotReady
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-test-app
  namespace: lab11
spec:
  replicas: 3
  selector:
    matchLabels:
      app: node-test-app
  template:
    metadata:
      labels:
        app: node-test-app
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

kubectl rollout status deployment/node-test-app -n lab11

# When a node goes NotReady in a real cluster:
# 1. Pods on that node get taint: node.kubernetes.io/unreachable:NoExecute
# 2. After tolerationSeconds (default 300s = 5 min), pods are evicted
# 3. Deployment controller reschedules evicted pods to healthy nodes
# 4. Pods on the dead node show: Terminating (stuck until node comes back)
#    OR: Unknown status (if node truly unreachable)

# Manually taint the node to simulate NotReady effects (careful in KIND)
# kubectl taint node kind-control-plane node.kubernetes.io/unreachable:NoExecute
# Don't run this in KIND — it evicts all pods including system pods
```

### Understanding System Taints Added on NotReady

```
When a node goes NotReady, K8s automatically adds these taints:

node.kubernetes.io/not-ready:NoSchedule
  → New pods won't be scheduled here

node.kubernetes.io/unreachable:NoSchedule
  → Same — during network partition

node.kubernetes.io/unreachable:NoExecute
  → Existing pods evicted after tolerationSeconds
  → Default toleration: 300 seconds (5 minutes)
  → After 5 min: pods evicted and rescheduled elsewhere

Critical system pods (kube-proxy, CoreDNS) have longer or infinite tolerations
so they don't get evicted during brief node hiccups.
```

### Prod Wisdom

When a node alert fires, the first question is always: **is the kubelet alive?** If the kubelet is running and healthy, the node's container runtime or networking is the issue. If the kubelet is dead, the node itself has a problem. SSH to the node, run `systemctl status kubelet` — that one command tells you which direction to investigate. Never spend 10 minutes in `kubectl describe` when `ssh` and `journalctl` give you the kubelet's exact error in 30 seconds.

---

## Scenario 02 — Cordon and Drain — Safe Maintenance

### What You'll Learn

Cordoning marks a node as unschedulable — no new pods land on it, but existing pods keep running. Draining evicts all pods from the node, making it safe for maintenance, upgrades, or decommission. Getting the drain wrong causes running workloads to crash without graceful shutdown.

### Set Up Workloads First

```bash
# Deploy a multi-replica app (survives drain gracefully)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: graceful-app
  namespace: lab11
spec:
  replicas: 2
  selector:
    matchLabels:
      app: graceful-app
  template:
    metadata:
      labels:
        app: graceful-app
    spec:
      # Graceful shutdown — 30s to finish in-flight requests
      terminationGracePeriodSeconds: 30
      containers:
      - name: app
        image: nginx:1.25
        lifecycle:
          preStop:
            exec:
              command: ["sh", "-c", "sleep 5"]   # Drain connection pool, finish requests
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "64Mi"
            cpu: "100m"
EOF

kubectl rollout status deployment/graceful-app -n lab11
kubectl get pods -n lab11 -o wide
# Confirm pods are on kind-control-plane
```

### Cordon — Stop New Scheduling

```bash
# Cordon the node — no new pods will schedule here
kubectl cordon kind-control-plane

kubectl get node kind-control-plane
# NAME                 STATUS                     ROLES
# kind-control-plane   Ready,SchedulingDisabled   control-plane
# SchedulingDisabled = node is cordoned

# Existing pods keep running — cordon only affects NEW scheduling
kubectl get pods -n lab11
# Still running — cordon doesn't evict existing pods

# Try to scale up — new pods will be Pending (nowhere to schedule)
kubectl scale deployment graceful-app --replicas=4 -n lab11
kubectl get pods -n lab11
# 2 Running (existing), 2 Pending (can't schedule — node cordoned)
```

### Uncordon — Re-enable Scheduling

```bash
kubectl uncordon kind-control-plane

kubectl get node kind-control-plane
# STATUS: Ready  ← SchedulingDisabled is gone

# Pending pods now schedule
kubectl get pods -n lab11
# All 4 Running
```

### Drain — Evict All Pods (Maintenance Mode)

```bash
# Scale back to 2 first
kubectl scale deployment graceful-app --replicas=2 -n lab11

# Drain — evicts all pods, cordons the node
# In KIND we need these flags:
#   --ignore-daemonsets: DaemonSet pods can't be evicted (they respawn)
#   --delete-emptydir-data: allows eviction of pods with emptyDir volumes
kubectl drain kind-control-plane \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=30 \         # Give pods 30s to terminate gracefully
  --timeout=120s              # Abort if drain takes longer than 2 min
```

Output you will see:
```
node/kind-control-plane cordoned
evicting pod lab11/graceful-app-xxx-aaa
evicting pod lab11/graceful-app-xxx-bbb
evicting pod kube-system/coredns-xxx-yyy
pod/graceful-app-xxx-aaa evicted
pod/graceful-app-xxx-bbb evicted
node/kind-control-plane drained
```

```bash
# Node is now cordoned AND all evictable pods are gone
kubectl get node kind-control-plane
# STATUS: Ready,SchedulingDisabled

kubectl get pods -n lab11
# Pods are Pending or rescheduled (no other node in KIND — they'll be Pending)

# In a real multi-node cluster:
# Pods would reschedule to other healthy nodes immediately
```

### PodDisruptionBudget — Protect Against Aggressive Drains

```bash
# PDB prevents drain from evicting too many pods at once
# Ensures minimum availability during node maintenance
cat <<EOF | kubectl apply -f -
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: graceful-app-pdb
  namespace: lab11
spec:
  minAvailable: 1             # At least 1 pod must remain running at all times
  selector:
    matchLabels:
      app: graceful-app
EOF

# Uncordon first to get pods running again
kubectl uncordon kind-control-plane
kubectl rollout status deployment/graceful-app -n lab11

# Now try to drain — PDB will slow it down
# Drain evicts pods one at a time, waits for replacement before evicting next
kubectl drain kind-control-plane \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=10 \
  --timeout=120s
# With PDB: drain evicts pod 1, waits for pod 2 to be healthy, then proceeds
# Without PDB: drain evicts all pods simultaneously (potential outage window)
```

### The Drain Workflow for Real Maintenance

```bash
# Standard pre-maintenance checklist:

# 1 — Confirm current node state
kubectl get nodes
kubectl describe node <node-name> | grep -A 5 "Allocated resources"

# 2 — Check PodDisruptionBudgets in affected namespaces
kubectl get pdb -A

# 3 — Cordon first (stop new scheduling, observe impact)
kubectl cordon <node-name>

# 4 — Check what pods will be evicted
kubectl get pods -A --field-selector spec.nodeName=<node-name>

# 5 — Drain with appropriate flags
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=30 \
  --timeout=300s

# 6 — Perform maintenance

# 7 — Uncordon when maintenance is complete
kubectl uncordon <node-name>

# 8 — Verify pods reschedule successfully
kubectl get pods -A -w
```

### Prod Wisdom

**Always cordon before you drain, and always check PDBs before draining.** A drain without checking PDBs can blow past availability guarantees and cause a service outage during what should be a safe maintenance window. The sequence is: check PDBs → cordon → verify impact → drain with grace period → wait for rescheduling → uncordon. `--grace-period` is not optional in prod — without it, pods are killed immediately with SIGKILL, bypassing graceful shutdown hooks and potentially corrupting in-flight operations.

---

## Scenario 03 — Node Disappears — Pod Eviction Timers

### What You'll Learn

When a node suddenly becomes unreachable, Kubernetes doesn't immediately evict the pods on it. It waits — first marking them `Unknown`, then evicting after a timeout. Understanding these timers prevents panic during an incident and helps you tune pod placement for faster recovery.

### The Eviction Timeline

```
T+0s    Node stops responding to API server
T+0s    Node condition transitions to Unknown

T+40s   Node.status.conditions[Ready].status = Unknown
        (controlled by --node-monitor-grace-period on controller-manager, default 40s)

T+40s   K8s adds taints to the node:
          node.kubernetes.io/unreachable:NoSchedule
          node.kubernetes.io/unreachable:NoExecute (with tolerationSeconds)

T+300s  Default toleration expires (tolerationSeconds: 300 = 5 minutes)
        Pods on unreachable node are evicted (marked for deletion)
        Deployment/StatefulSet controllers create replacements on healthy nodes

T+300s+ Evicted pods show as: Terminating
        They stay Terminating until the node comes back (or is force-deleted)
        because the kubelet on the dead node never sends the final delete confirmation
```

### Observe the Default Tolerations

```bash
# Every pod gets these tolerations automatically from K8s
kubectl get pod $(kubectl get pod -n lab11 -l app=graceful-app \
  -o jsonpath='{.items[0].metadata.name}') -n lab11 \
  -o jsonpath='{.spec.tolerations}' | jq '.'
```

You will see:
```json
[
  {
    "effect": "NoExecute",
    "key": "node.kubernetes.io/not-ready",
    "operator": "Exists",
    "tolerationSeconds": 300
  },
  {
    "effect": "NoExecute",
    "key": "node.kubernetes.io/unreachable",
    "operator": "Exists",
    "tolerationSeconds": 300
  }
]
```

These are injected automatically. `tolerationSeconds: 300` means 5 minutes before eviction.

### Tune Eviction Timers for Faster Recovery

```bash
# For latency-sensitive workloads: reduce tolerationSeconds
# Pod will be evicted and rescheduled faster when a node goes down
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fast-failover-app
  namespace: lab11
spec:
  replicas: 2
  selector:
    matchLabels:
      app: fast-failover
  template:
    metadata:
      labels:
        app: fast-failover
    spec:
      tolerations:
      - key: "node.kubernetes.io/unreachable"
        operator: "Exists"
        effect: "NoExecute"
        tolerationSeconds: 30    # Evict after 30s instead of 300s
      - key: "node.kubernetes.io/not-ready"
        operator: "Exists"
        effect: "NoExecute"
        tolerationSeconds: 30    # Faster failover for critical services
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

# For batch/stateful workloads: increase tolerationSeconds
# Allow more time for transient network issues to resolve before evicting
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tolerant-app
  namespace: lab11
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tolerant-app
  template:
    metadata:
      labels:
        app: tolerant-app
    spec:
      tolerations:
      - key: "node.kubernetes.io/unreachable"
        operator: "Exists"
        effect: "NoExecute"
        tolerationSeconds: 600   # Wait 10 min — stateful, rescheduling is expensive
      - key: "node.kubernetes.io/not-ready"
        operator: "Exists"
        effect: "NoExecute"
        tolerationSeconds: 600
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

### Force-Delete Stuck Terminating Pods

```bash
# When a node is truly gone (hardware failure, deleted VM), pods stay
# in Terminating forever because the kubelet never confirms deletion
# After confirming the node is truly gone:

kubectl get pods -n lab11
# NAME                    READY   STATUS        RESTARTS   AGE
# some-pod-xxx-yyy        0/1     Terminating   0          10m  ← stuck

# Force delete — skips graceful termination
kubectl delete pod some-pod-xxx-yyy -n lab11 --grace-period=0 --force

# WARNING: only force-delete when you are certain the node is gone
# Force-deleting a pod on a live node means two copies may run simultaneously
# This is especially dangerous for stateful apps (split-brain, data corruption)
```

### Prod Wisdom

**The 5-minute eviction timer exists to protect against false positives.** A brief network blip shouldn't cause mass pod eviction and rescheduling. But for truly critical services, 5 minutes of non-availability while waiting for eviction is unacceptable. Tune `tolerationSeconds` to match the workload's SLA — reduce it for stateless services that can reschedule safely, keep it high for stateful services where rescheduling has a cost. And never force-delete a pod on a node you're not certain is gone — the risk of running duplicate stateful pods is data corruption.

---

## Scenario 04 — Node Resource Pressure — Pod Evictions

### What You'll Learn

When a node runs low on memory or disk, the kubelet starts evicting pods to reclaim resources. The eviction order follows QoS classes (from Lab 05). Understanding what triggers eviction, which pods are evicted first, and how to read eviction signals lets you react before the cascade becomes a full node failure.

### Eviction Thresholds

```bash
# Default kubelet eviction thresholds (these trigger eviction):
# memory.available < 100Mi   → pods start being evicted
# nodefs.available < 10%     → disk evictions
# nodefs.inodesFree < 5%     → inode evictions
# imagefs.available < 15%    → image filesystem evictions

# Check current node memory and disk
kubectl describe node kind-control-plane | grep -A 8 "Conditions:"
# MemoryPressure: True/False
# DiskPressure:   True/False
```

### Observe Node Pressure

```bash
# Check actual node resource usage
kubectl top node kind-control-plane
# NAME                 CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# kind-control-plane   250m         12%    1200Mi          60%

# Detailed allocation view
kubectl describe node kind-control-plane | grep -A 12 "Allocated resources:"
# Resource           Requests      Limits
# cpu                500m (25%)    1 (50%)
# memory             512Mi (25%)   1Gi (50%)
# ephemeral-storage  0 (0%)        0 (0%)
```

### What Happens Under Memory Pressure

```bash
# Deploy workloads across all QoS classes (review from Lab 05)
cat <<EOF | kubectl apply -f -
# BestEffort — no resources set — evicted FIRST
apiVersion: v1
kind: Pod
metadata:
  name: besteffort-node-pod
  namespace: lab11
spec:
  containers:
  - name: app
    image: nginx:1.25
    # No resources = BestEffort
---
# Burstable — limits > requests — evicted SECOND
apiVersion: v1
kind: Pod
metadata:
  name: burstable-node-pod
  namespace: lab11
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "128Mi"
        cpu: "200m"
---
# Guaranteed — limits == requests — evicted LAST
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-node-pod
  namespace: lab11
spec:
  containers:
  - name: app
    image: nginx:1.25
    resources:
      requests:
        memory: "64Mi"
        cpu: "100m"
      limits:
        memory: "64Mi"    # Same as request = Guaranteed
        cpu: "100m"
EOF

# Confirm QoS classes
kubectl get pods -n lab11 \
  -o custom-columns="NAME:.metadata.name,QOS:.status.qosClass" \
  | grep -E "node-pod"
# besteffort-node-pod   BestEffort
# burstable-node-pod    Burstable
# guaranteed-node-pod   Guaranteed
```

### Detecting and Diagnosing Evictions

```bash
# Check for evicted pods (they show as Failed with reason Evicted)
kubectl get pods -n lab11 --field-selector=status.phase=Failed

# Describe an evicted pod for the reason
kubectl describe pod <evicted-pod-name> -n lab11
# Status: Failed
# Reason: Evicted
# Message: The node was low on resource: memory.
#          Threshold quantity: 100Mi, available: 80Mi.
#          Container app was using 45Mi, request is 0.

# See eviction events on the node
kubectl describe node kind-control-plane | grep -A 5 "Events:"
# Warning  Evicted  kubelet  Evicting pod lab11/besteffort-pod

# Namespace-wide event sweep for evictions
kubectl get events -n lab11 \
  --field-selector reason=Evicted \
  --sort-by='.lastTimestamp'

# Check all namespaces for evictions
kubectl get pods -A --field-selector=status.phase=Failed | grep Evicted
```

### Node Pressure Conditions

```bash
# Check all node conditions programmatically
kubectl get node kind-control-plane -o json | \
  jq '.status.conditions[] | {type: .type, status: .status, reason: .reason, message: .message}'

# Output when under pressure:
# {
#   "type": "MemoryPressure",
#   "status": "True",
#   "reason": "KubeletHasInsufficientMemory",
#   "message": "kubelet has insufficient memory available"
# }

# When MemoryPressure = True:
# - Node gets taint: node.kubernetes.io/memory-pressure:NoSchedule
# - New pods won't schedule here automatically
# - BestEffort pods start getting evicted immediately
```

### Prod Wisdom

**Node evictions are a symptom, not the root cause.** When you see pods being evicted, the real question is: why is the node under pressure? Check `kubectl top node` and `kubectl top pods -A --sort-by=memory` to find which pod is consuming the most. Almost always it's a pod without memory limits (BestEffort) consuming everything it can. The fix is usually adding resource limits — not restarting the evicted pods. Evicted pods come back. Without fixing the root cause, they get evicted again.

---

## Scenario 05 — Full Node Debug Workflow

### What You'll Build

This scenario puts it all together — the complete systematic workflow a senior engineer runs when a node alert fires. No matter what the symptom, this sequence finds the root cause.

### The Complete Node Investigation Sequence

```bash
# ============================================================
# PHASE 1 — TRIAGE: What is the node state?
# ============================================================

# Step 1 — Node overview
kubectl get nodes -o wide
# NAME    STATUS   ROLES   AGE   VERSION   INTERNAL-IP   OS-IMAGE
# Shows: Ready/NotReady, IP, K8s version

# Step 2 — Node conditions (the health dashboard)
kubectl describe node kind-control-plane | grep -A 15 "Conditions:"
# MemoryPressure / DiskPressure / PIDPressure / Ready
# Any True on pressure conditions = resource problem
# Ready = False or Unknown = node itself is unhealthy

# Step 3 — Is the node cordoned?
kubectl get node kind-control-plane | grep SchedulingDisabled
# If yes: intentional maintenance, or accidental cordon

# ============================================================
# PHASE 2 — RESOURCE USAGE: What is consuming the node?
# ============================================================

# Step 4 — Node-level resource usage
kubectl top node kind-control-plane
# CPU%, Memory% — if > 85% either: resource problem

# Step 5 — Allocated vs actual
kubectl describe node kind-control-plane | grep -A 12 "Allocated resources:"
# Compare Requests/Limits vs actual Capacity
# Requests >> Capacity = overcommitted node

# Step 6 — Which pods are using the most on this node?
kubectl top pods -A --sort-by=memory \
  --field-selector spec.nodeName=kind-control-plane
# Find the heaviest consumers

# ============================================================
# PHASE 3 — PODS: What is running on this node?
# ============================================================

# Step 7 — All pods on this node
kubectl get pods -A \
  --field-selector spec.nodeName=kind-control-plane \
  -o wide

# Step 8 — Are any pods evicted or failed?
kubectl get pods -A \
  --field-selector spec.nodeName=kind-control-plane,status.phase=Failed

# Step 9 — Pods that are Not Ready
kubectl get pods -A \
  --field-selector spec.nodeName=kind-control-plane \
  | grep -v "Running\|Completed"

# ============================================================
# PHASE 4 — EVENTS: What has been happening?
# ============================================================

# Step 10 — Node-level events
kubectl describe node kind-control-plane | tail -30
# Events section shows recent warnings, evictions, pressure events

# Step 11 — All warnings in the cluster
kubectl get events -A \
  --field-selector type=Warning \
  --sort-by='.lastTimestamp' | tail -20

# Step 12 — Events for a specific pod that's having issues
kubectl get events -n <namespace> \
  --field-selector involvedObject.name=<pod-name> \
  --sort-by='.lastTimestamp'

# ============================================================
# PHASE 5 — SYSTEM: Node system health (SSH required in real clusters)
# ============================================================

# In KIND, exec into the node container
docker exec -it kind-control-plane bash

# Once inside:
# Check kubelet status
systemctl status kubelet

# Kubelet logs (last 50 lines)
journalctl -u kubelet -n 50 --no-pager

# Container runtime status
systemctl status containerd

# Disk usage
df -h
# Look for any filesystem at > 85%

# Memory
free -m
# Available column — if < 200MB: memory pressure imminent

# Load average
uptime
# 1-min load > number of CPUs = CPU pressure

# OOM events from kernel
dmesg | grep -i "oom\|out of memory\|kill" | tail -20

# Process consuming most memory
ps aux --sort=-%mem | head -20

# Exit the node container
exit
```

### The Node Debug Decision Tree

```
Node alert fires
│
├── kubectl get nodes
│   ├── STATUS: NotReady or Unknown
│   │   ├── Recent heartbeat? (lastHeartbeatTime)
│   │   │   ├── Recent: kubelet alive, runtime/network issue
│   │   │   │   → SSH → systemctl status containerd
│   │   │   │   → SSH → ip addr / ping (network check)
│   │   │   └── Stale (minutes/hours ago): kubelet dead or node unreachable
│   │   │       → SSH → systemctl status kubelet
│   │   │       → SSH → journalctl -u kubelet -n 100
│   │   │
│   │   └── Can't SSH? → Node is unreachable (hardware/network failure)
│   │       → Check cloud console (AWS EC2, GCP VM, etc.)
│   │       → Decide: wait for recovery or force-delete pods
│   │
│   └── STATUS: Ready (but something is wrong)
│       │
│       ├── MemoryPressure: True
│       │   → kubectl top pods -A --sort-by=memory
│       │   → Find pod without limits consuming everything
│       │   → Add limits or evict the offending pod
│       │
│       ├── DiskPressure: True
│       │   → SSH → df -h (find full filesystem)
│       │   → docker system prune (clean unused images)
│       │   → Clean log files or extend disk
│       │
│       └── Pods evicted / NotReady but node Ready
│           → QoS class issue — check limits on evicted pods
│           → kubectl describe pod <evicted> | grep "Reason\|Message"
│
└── kubectl get nodes (all ready, but pods failing)
    → Problem is not the node — go to pod/service/ingress debugging
```

### Quick Node Health Summary Command

```bash
# One command to get the full node picture
kubectl describe node kind-control-plane | \
  grep -E "Conditions:|MemoryPressure|DiskPressure|PIDPressure|Ready|Allocated|Requests|Limits|Events" | \
  head -40
```

### Creating a Node Debug Pod

```bash
# Sometimes you need to debug from inside the cluster on a specific node
# Run a privileged debug pod on a specific node
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: node-debug
  namespace: lab11
spec:
  nodeName: kind-control-plane    # Force to the specific node
  hostNetwork: true               # Use node's network namespace
  hostPID: true                   # See all processes on the node
  containers:
  - name: debug
    image: busybox:1.35
    command: ["sleep", "3600"]
    securityContext:
      privileged: true
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
EOF

kubectl wait --for=condition=ready pod/node-debug -n lab11 --timeout=30s

# From inside this pod you can see the node's full process list
kubectl exec -it node-debug -n lab11 -- ps aux | head -20

# See node's network interfaces
kubectl exec -it node-debug -n lab11 -- ip addr

# Check node disk from inside
kubectl exec -it node-debug -n lab11 -- df -h

# Clean up
kubectl delete pod node-debug -n lab11
```

---

## Key Commands Reference — Lab 11

```bash
# Node overview
kubectl get nodes
kubectl get nodes -o wide
kubectl get nodes --show-labels

# Node health
kubectl describe node <name>
kubectl describe node <name> | grep -A 15 "Conditions:"
kubectl describe node <name> | grep -A 12 "Allocated resources:"

# Node conditions as JSON
kubectl get node <name> -o jsonpath='{.status.conditions}' | jq '.'

# Node events
kubectl describe node <name> | tail -30

# Pods on a specific node
kubectl get pods -A --field-selector spec.nodeName=<node-name>
kubectl get pods -A --field-selector spec.nodeName=<node-name>,status.phase=Failed

# Resource usage (requires metrics-server)
kubectl top nodes
kubectl top pods -A --sort-by=memory

# Cordon / Uncordon
kubectl cordon <node-name>
kubectl uncordon <node-name>

# Drain
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=30 \
  --timeout=300s

# Force-delete stuck Terminating pod (only when node is confirmed gone)
kubectl delete pod <name> -n <namespace> --grace-period=0 --force

# PodDisruptionBudget
kubectl get pdb -A
kubectl describe pdb <name> -n <namespace>

# Node taints
kubectl describe node <name> | grep Taints
kubectl taint node <name> key=value:effect
kubectl taint node <name> key=value:effect-    # Remove taint

# Last heartbeat time
kubectl get node <name> \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].lastHeartbeatTime}'
```

---

## Prod Wisdom — The Senior Engineer Mindset

Three things that define senior-level node troubleshooting:

**1. They distinguish between node-level and pod-level failures immediately.** When a node goes `NotReady`, every pod on it becomes a victim — but the problem is the node, not the pods. Fixing pod configurations won't help. The first question is always: "is this a node problem or a pod problem?" `kubectl get nodes` answers this in 2 seconds and determines the entire direction of the investigation.

**2. They always cordon before they drain, and they check PDBs before cordoning.** Draining without cordoning first means new pods can schedule to the node during the drain operation, creating a moving target. Draining without checking PDBs means you might violate availability guarantees and cause an outage during maintenance. The sequence is non-negotiable: check PDBs → cordon → drain.

**3. They know eviction timers and tune them deliberately.** The default 5-minute eviction timer is not right for every workload. Stateless web services should fail over in 30 seconds. Stateful databases might tolerate 10 minutes to avoid an expensive reschedule for a transient blip. Tuning `tolerationSeconds` per workload based on its SLA and reschedule cost is senior-level platform thinking.

The node debug order — run it the same every time:
```
kubectl get nodes          → Ready or NotReady?
kubectl describe node      → Conditions, Allocated resources, Events
kubectl top node           → CPU/Memory usage
kubectl get pods -A --field-selector spec.nodeName=<node>  → What's running?
SSH → systemctl status kubelet → Is kubelet alive?
SSH → journalctl -u kubelet    → What is kubelet saying?
SSH → df -h / free -m          → Disk and memory at OS level
```

---

## Cleanup

```bash
# Ensure node is uncordoned before cleanup
kubectl uncordon kind-control-plane 2>/dev/null || true

kubectl delete namespace lab11
```

---

*Lab 11 complete. Move to Lab 12 — Namespace & Context Management when ready.*