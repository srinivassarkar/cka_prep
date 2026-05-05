# Lab 10 — PVC & Storage

## What This Lab Is About

Persistent storage is where Kubernetes gets stateful. Pods are ephemeral by design — when a pod dies, its filesystem dies with it. PersistentVolumes (PV) and PersistentVolumeClaims (PVC) are how you give pods storage that survives restarts, rescheduling, and upgrades. Get this wrong and your database loses data, your stateful app corrupts its state, or your pod sits in `Pending` forever waiting for storage that will never bind.

This lab covers the 5 most critical storage failure modes in real clusters. A PVC stuck in `Pending` because no PV matches its requirements. A pod that can't mount its volume because of access mode conflicts. A StatefulSet where storage doesn't follow the pod when it reschedules. StorageClass misconfiguration that silently fails dynamic provisioning. And the correct production pattern for stateful workloads that need durable, reliable storage.

> In prod, storage failures are the most dangerous class of Kubernetes failures. A networking issue drops traffic. A storage issue loses data. These are not the same severity.

---

## Environment

- **Cluster:** KIND (Kubernetes IN Docker)
- **K8s Version:** v1.27.3
- **Node:** Single node (`kind-control-plane`)
- **Namespace for this lab:** `lab10`

```bash
kubectl create namespace lab10
kubectl config set-context --current --namespace=lab10
```

### Check Available StorageClasses in KIND

```bash
kubectl get storageclass
# NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE
# standard (default)   rancher.io/local-path   Delete          WaitForFirstConsumer
```

KIND ships with a default StorageClass (`standard`) backed by `local-path-provisioner`. This handles dynamic provisioning for all labs here.

---

## The 5 Scenarios

| # | Scenario | What You'll Learn |
|---|---|---|
| 01 | PVC stuck in `Pending` — no matching PV | Storage class, access modes, capacity matching |
| 02 | Volume mount fails — access mode conflict | ReadWriteOnce vs ReadWriteMany, multi-pod attach |
| 03 | Pod loses data on restart — wrong volume type | emptyDir vs PVC, ephemeral vs persistent |
| 04 | StatefulSet storage — PVC not following the pod | How StatefulSets manage PVCs differently |
| 05 | Production pattern — storage lifecycle and reclaim | ReclaimPolicy, retain vs delete, data safety |

---

## Scenario 01 — PVC Stuck in Pending — No Matching PV

### What You'll Break

A PVC that requests storage with requirements no available PV or StorageClass can satisfy. The PVC stays in `Pending` indefinitely. The pod that needs it also stays in `Pending`. No errors are thrown — just silence and a stalled workload.

### Break It — Wrong StorageClass Name

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: broken-pvc-storageclass
  namespace: lab10
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: premium-ssd    # WRONG: this StorageClass doesn't exist in KIND
  resources:
    requests:
      storage: 1Gi
EOF

kubectl get pvc broken-pvc-storageclass -n lab10
# NAME                      STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# broken-pvc-storageclass   Pending                                      premium-ssd    30s
# STATUS: Pending — StorageClass doesn't exist, no provisioner to handle it
```

```bash
# Deploy a pod that depends on this PVC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pvc-pending-pod
  namespace: lab10
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c", "echo 'writing data' > /data/test.txt && sleep 3600"]
    volumeMounts:
    - name: data-vol
      mountPath: /data
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
  volumes:
  - name: data-vol
    persistentVolumeClaim:
      claimName: broken-pvc-storageclass
EOF

kubectl get pod pvc-pending-pod -n lab10
# NAME             READY   STATUS    RESTARTS   AGE
# pvc-pending-pod  0/1     Pending   0          30s
# Pod is Pending because PVC is Pending
```

### Investigate

```bash
# Step 1 — Check PVC status
kubectl get pvc -n lab10
# STATUS: Pending for broken-pvc-storageclass

# Step 2 — Describe PVC for the reason
kubectl describe pvc broken-pvc-storageclass -n lab10
# Events:
#   Warning  ProvisioningFailed  persistentvolume-controller
#   storageclass.storage.k8s.io "premium-ssd" not found

# Step 3 — What StorageClasses actually exist?
kubectl get storageclass
# NAME                 PROVISIONER             RECLAIMPOLICY
# standard (default)   rancher.io/local-path   Delete

# Step 4 — Describe the pod to confirm it's waiting for PVC
kubectl describe pod pvc-pending-pod -n lab10 | grep -A 5 "Events"
# Warning  FailedScheduling  default-scheduler
# 0/1 nodes are available: pod has unbound immediate PersistentVolumeClaims

# Step 5 — What PVs exist (if any)?
kubectl get pv
# No resources found — no PVs manually created, dynamic provisioning failed
```

### Fix

```bash
# Delete the broken PVC and pod
kubectl delete pod pvc-pending-pod -n lab10
kubectl delete pvc broken-pvc-storageclass -n lab10

# Create PVC with the correct StorageClass
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: correct-pvc
  namespace: lab10
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: standard     # Fixed: matches available StorageClass in KIND
  resources:
    requests:
      storage: 1Gi
EOF

# NOTE: In KIND with WaitForFirstConsumer, PVC stays Pending until a pod uses it
kubectl get pvc correct-pvc -n lab10
# STATUS: Pending (WaitForFirstConsumer — binds when pod schedules)

# Deploy the pod — this triggers binding
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: correct-pvc-pod
  namespace: lab10
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c"]
    args:
    - |
      echo "Writing to persistent storage..."
      echo "hello-persistent-world" > /data/test.txt
      cat /data/test.txt
      sleep 3600
    volumeMounts:
    - name: data-vol
      mountPath: /data
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
  volumes:
  - name: data-vol
    persistentVolumeClaim:
      claimName: correct-pvc
EOF

# PVC binds once pod is scheduled
kubectl get pvc correct-pvc -n lab10
# STATUS: Bound  ← PVC is now bound to a PV

kubectl get pv
# NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   STORAGECLASS
# pvc-xxx-yyy  1Gi        RWO            Delete           Bound    standard

kubectl logs correct-pvc-pod -n lab10
# Writing to persistent storage...
# hello-persistent-world
```

### Break It — Requesting Too Much Storage

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: too-large-pvc
  namespace: lab10
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 100Ti    # Requesting 100 Terabytes — node doesn't have this
EOF

kubectl get pvc too-large-pvc -n lab10
# STATUS: Pending — provisioner can't allocate 100Ti on a KIND node

kubectl describe pvc too-large-pvc -n lab10 | grep -A 3 "Events"
# Warning  ProvisioningFailed  rancher.io/local-path
# failed to provision volume: ...not enough space
```

```bash
# Fix — request a reasonable amount
kubectl delete pvc too-large-pvc -n lab10

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: reasonable-pvc
  namespace: lab10
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 500Mi    # Reasonable for KIND
EOF
```

### The PVC Binding Requirements — All Must Match

```
For a PVC to bind to a PV (static provisioning):
  ✓ StorageClass must match
  ✓ Access mode must be compatible
  ✓ PV capacity must be >= PVC request
  ✓ PV must be in Available state (not already Bound)
  ✓ Label selectors must match (if specified)

For dynamic provisioning (StorageClass with provisioner):
  ✓ StorageClass must exist
  ✓ Provisioner must be running and healthy
  ✓ Node must have enough underlying storage
  ✓ Access mode must be supported by the provisioner
```

### Prod Wisdom

**Always verify the StorageClass name before deploying PVCs.** A typo in `storageClassName` silently leaves the PVC in `Pending` forever — no error, no alert, just a stalled pod. In prod clusters, run `kubectl get storageclass` before writing any PVC spec and copy the name exactly. For critical workloads, `kubectl describe pvc` immediately after creation is a 5-second check that confirms binding is progressing.

---

## Scenario 02 — Volume Mount Fails — Access Mode Conflict

### What You'll Break

Multiple pods trying to mount the same PVC with `ReadWriteOnce` access mode. `ReadWriteOnce` means the volume can only be mounted as read-write by **one node** at a time. When a second pod on a different node (or the same node in some cases) tries to mount it, the mount fails. This is the most common storage failure in multi-replica stateful workloads.

### Understanding Access Modes

```
ReadWriteOnce (RWO)
  → One node can mount read-write at a time
  → Multiple pods on the SAME node can share it
  → Second node trying to mount = failure
  → Use for: single-instance databases, single-replica stateful apps

ReadOnlyMany (ROX)
  → Many nodes can mount read-only simultaneously
  → No node can write
  → Use for: shared config, static assets, reference data

ReadWriteMany (RWX)
  → Many nodes can mount read-write simultaneously
  → Requires special storage backend (NFS, CephFS, EFS)
  → NOT supported by local-path or most cloud block storage
  → Use for: shared writable storage across many pods

ReadWriteOncePod (RWOP) — K8s 1.22+
  → Only ONE POD in the entire cluster can mount read-write
  → Stricter than RWO (which allows same-node multi-pod)
  → Use for: single-writer guarantee across the cluster
```

### Break It — Multi-Replica Deployment with RWO PVC

```bash
# Create a RWO PVC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rwo-pvc
  namespace: lab10
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 500Mi
EOF

# Deploy with 3 replicas all trying to use the same RWO PVC
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rwo-multi-replica
  namespace: lab10
spec:
  replicas: 3               # 3 pods all wanting RWO volume
  selector:
    matchLabels:
      app: rwo-multi
  template:
    metadata:
      labels:
        app: rwo-multi
    spec:
      containers:
      - name: app
        image: busybox:1.35
        command: ["sh", "-c", "sleep 3600"]
        volumeMounts:
        - name: shared-data
          mountPath: /data
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "64Mi"
            cpu: "100m"
      volumes:
      - name: shared-data
        persistentVolumeClaim:
          claimName: rwo-pvc
EOF

kubectl get pods -n lab10 -l app=rwo-multi
# In KIND (single node), all pods land on the same node
# so RWO may actually work here — but in multi-node clusters:
# Pod 1: Running (first to mount on its node)
# Pod 2: ContainerCreating or stuck (different node, RWO conflict)
# Pod 3: ContainerCreating or stuck
```

```bash
# Describe a stuck pod to see the access mode error
kubectl describe pod $(kubectl get pod -n lab10 -l app=rwo-multi \
  -o jsonpath='{.items[1].metadata.name}') -n lab10 | grep -A 5 "Events"
# Warning  FailedAttachVolume  attachdetach-controller
# Multi-Attach error for volume: Volume is already exclusively attached
# to one node and can't be attached to another
```

### The Correct Pattern — One PVC Per Pod Via StatefulSet

```bash
# Clean up
kubectl delete deployment rwo-multi-replica -n lab10
kubectl delete pvc rwo-pvc -n lab10

# StatefulSets give each pod its own PVC automatically
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: stateful-app
  namespace: lab10
spec:
  serviceName: stateful-app
  replicas: 3
  selector:
    matchLabels:
      app: stateful-app
  template:
    metadata:
      labels:
        app: stateful-app
    spec:
      containers:
      - name: app
        image: busybox:1.35
        command: ["sh", "-c"]
        args:
        - |
          # Each pod writes to its OWN storage
          echo "Pod ${HOSTNAME} writing to /data/pod.txt"
          echo "I am ${HOSTNAME}" > /data/pod.txt
          cat /data/pod.txt
          sleep 3600
        volumeMounts:
        - name: pod-data
          mountPath: /data
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "64Mi"
            cpu: "100m"
  volumeClaimTemplates:              # Each pod gets its own PVC
  - metadata:
      name: pod-data
    spec:
      accessModes: ["ReadWriteOnce"] # RWO is fine because each pod has its own
      storageClassName: standard
      resources:
        requests:
          storage: 256Mi
EOF

kubectl get pods -n lab10 -l app=stateful-app
# stateful-app-0   1/1   Running
# stateful-app-1   1/1   Running
# stateful-app-2   1/1   Running

kubectl get pvc -n lab10
# pod-data-stateful-app-0   Bound   256Mi
# pod-data-stateful-app-1   Bound   256Mi
# pod-data-stateful-app-2   Bound   256Mi
# Each pod has its own PVC — no RWO conflict
```

### Prod Wisdom

**Never attach a single RWO PVC to a multi-replica Deployment.** RWO guarantees exclusive node-level access — a second node mounting it will fail. For multi-replica workloads that need shared storage, use `ReadWriteMany` with a compatible backend (NFS, CephFS, AWS EFS). For multi-replica workloads that need independent storage, use a StatefulSet with `volumeClaimTemplates`. For single-replica stateful apps (Redis, single-instance Postgres), a Deployment with a single RWO PVC is acceptable.

---

## Scenario 03 — Pod Loses Data on Restart — Wrong Volume Type

### What You'll Break

A pod using `emptyDir` thinking it's persistent. `emptyDir` lives as long as the pod lives on a node — it survives container restarts within the same pod but is **wiped when the pod is deleted or rescheduled**. Developers who don't understand this write data to `emptyDir` and lose it every time the pod restarts for any reason.

### Apply the Broken State

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: data-losing-pod
  namespace: lab10
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c"]
    args:
    - |
      echo "Writing important data..."
      echo "user_count=1000" > /data/state.txt
      echo "session_id=abc123" >> /data/state.txt
      echo "Data written:"
      cat /data/state.txt
      sleep 3600
    volumeMounts:
    - name: data-vol
      mountPath: /data
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
  volumes:
  - name: data-vol
    emptyDir: {}            # Ephemeral — wiped when pod is deleted
EOF

kubectl wait --for=condition=ready pod/data-losing-pod -n lab10 --timeout=30s

kubectl logs data-losing-pod -n lab10
# Writing important data...
# user_count=1000
# session_id=abc123

# Verify data exists
kubectl exec data-losing-pod -n lab10 -- cat /data/state.txt
# user_count=1000
# session_id=abc123
```

### Observe the Data Loss

```bash
# Delete and recreate the pod — simulates a reschedule or OOMKill
kubectl delete pod data-losing-pod -n lab10

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: data-losing-pod
  namespace: lab10
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c"]
    args:
    - |
      echo "Checking for existing data..."
      if [ -f /data/state.txt ]; then
        echo "Data found:"
        cat /data/state.txt
      else
        echo "DATA LOST — /data/state.txt does not exist"
      fi
      sleep 3600
    volumeMounts:
    - name: data-vol
      mountPath: /data
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
  volumes:
  - name: data-vol
    emptyDir: {}
EOF

kubectl wait --for=condition=ready pod/data-losing-pod -n lab10 --timeout=30s
kubectl logs data-losing-pod -n lab10
# Checking for existing data...
# DATA LOST — /data/state.txt does not exist
# emptyDir was wiped when the pod was deleted
```

### emptyDir Does Survive Container Restarts

```bash
# emptyDir survives container crashes within the SAME pod lifecycle
# Only pod deletion/rescheduling wipes it
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-restart-test
  namespace: lab10
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c"]
    args:
    - |
      if [ -f /data/count.txt ]; then
        COUNT=$(cat /data/count.txt)
        echo "Restart #${COUNT} — data survived container restart"
        echo $((COUNT + 1)) > /data/count.txt
      else
        echo "First start — writing count"
        echo "1" > /data/count.txt
      fi
      sleep 5
      exit 1   # Simulate crash — container restarts but pod stays alive
    volumeMounts:
    - name: temp-vol
      mountPath: /data
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
  volumes:
  - name: temp-vol
    emptyDir: {}

EOF

# Watch restarts — data survives container crash but not pod deletion
kubectl get pod emptydir-restart-test -n lab10 -w
# RESTARTS climbs but data survives between container restarts
sleep 20
kubectl logs emptydir-restart-test -n lab10 --previous
# Restart #1 — data survived container restart
```

### Fix — Use a PVC for Persistent Data

```bash
kubectl delete pod data-losing-pod -n lab10
kubectl delete pod emptydir-restart-test -n lab10

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: persistent-state-pvc
  namespace: lab10
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 256Mi
---
apiVersion: v1
kind: Pod
metadata:
  name: data-surviving-pod
  namespace: lab10
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c"]
    args:
    - |
      if [ -f /data/state.txt ]; then
        echo "Existing data found — survived pod restart:"
        cat /data/state.txt
      else
        echo "First run — writing data"
        echo "user_count=1000" > /data/state.txt
        echo "session_id=abc123" >> /data/state.txt
        cat /data/state.txt
      fi
      sleep 3600
    volumeMounts:
    - name: persistent-data
      mountPath: /data
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
  volumes:
  - name: persistent-data
    persistentVolumeClaim:
      claimName: persistent-state-pvc
EOF

kubectl wait --for=condition=ready pod/data-surviving-pod -n lab10 --timeout=60s
kubectl logs data-surviving-pod -n lab10
# First run — writing data
# user_count=1000

# Delete and recreate — data should survive
kubectl delete pod data-surviving-pod -n lab10

# Recreate same pod spec
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: data-surviving-pod
  namespace: lab10
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c"]
    args:
    - |
      if [ -f /data/state.txt ]; then
        echo "Data survived pod deletion:"
        cat /data/state.txt
      else
        echo "DATA LOST"
      fi
      sleep 3600
    volumeMounts:
    - name: persistent-data
      mountPath: /data
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
  volumes:
  - name: persistent-data
    persistentVolumeClaim:
      claimName: persistent-state-pvc
EOF

kubectl wait --for=condition=ready pod/data-surviving-pod -n lab10 --timeout=60s
kubectl logs data-surviving-pod -n lab10
# Data survived pod deletion:
# user_count=1000
# session_id=abc123  ← persisted across pod deletion and recreation
```

### Volume Type Reference

```
emptyDir
  Lifecycle:  Lives as long as the pod on a node
  Survives:   Container restarts within the same pod
  Wiped on:   Pod deletion, pod rescheduling to another node
  Use for:    Temp files, caches, scratch space, shared between
              containers in the same pod

hostPath
  Lifecycle:  Lives on the node filesystem
  Survives:   Pod deletion (data stays on the node)
  Wiped on:   Node replacement, pod moves to different node
  Use for:    Node-level monitoring agents, NOT for app data

PersistentVolumeClaim (PVC)
  Lifecycle:  Independent of pod lifecycle
  Survives:   Pod deletion, pod rescheduling, node replacement
              (cloud volumes follow the pod to new nodes)
  Wiped on:   PVC deletion (if ReclaimPolicy: Delete)
  Use for:    Any data that must survive pod restarts
              Databases, queues, user uploads, state
```

### Prod Wisdom

**`emptyDir` is a cache, not a database.** In prod, anything written to `emptyDir` that matters — session state, user data, application state — is silently lost the moment the pod is evicted or rescheduled. The tell is a stateful app that "randomly" loses data after a node pressure event. Always ask: "if this pod were killed right now and restarted on a different node, would data written here survive?" If the answer is no, it should be a PVC.

---

## Scenario 04 — StatefulSet Storage — PVC Not Following the Pod

### What You'll Break

A StatefulSet where the PVC lifecycle is misunderstood. When a StatefulSet pod is deleted, the PVC is NOT deleted — it stays bound, waiting. When the pod is recreated (by the StatefulSet controller), it reattaches to the same PVC. This is correct behaviour — but it trips up engineers who expect the PVC to follow or who accidentally delete PVCs during maintenance.

### Apply the Setup

```bash
# Ensure StatefulSet from Scenario 02 is running
kubectl get statefulset stateful-app -n lab10
# If not present, recreate it from Scenario 02

# Write unique data to each pod's storage
for i in 0 1 2; do
  kubectl exec stateful-app-${i} -n lab10 -- \
    sh -c "echo 'pod-${i}-unique-data' > /data/identity.txt"
done

# Verify each pod has unique data
for i in 0 1 2; do
  echo "=== stateful-app-${i} ==="
  kubectl exec stateful-app-${i} -n lab10 -- cat /data/identity.txt
done
# pod-0-unique-data
# pod-1-unique-data
# pod-2-unique-data
```

### Delete a Pod — Observe PVC Stays

```bash
# Delete pod-1 — StatefulSet will recreate it
kubectl delete pod stateful-app-1 -n lab10

# Watch pod-1 come back
kubectl get pods -n lab10 -l app=stateful-app -w
# stateful-app-1 goes: Terminating → Pending → ContainerCreating → Running

# Check PVCs — they were NOT deleted
kubectl get pvc -n lab10
# pod-data-stateful-app-0   Bound
# pod-data-stateful-app-1   Bound   ← still here, unchanged
# pod-data-stateful-app-2   Bound

# Verify pod-1 reattached to its OWN PVC with data intact
kubectl exec stateful-app-1 -n lab10 -- cat /data/identity.txt
# pod-1-unique-data  ← data survived pod deletion and recreation
```

### The Dangerous Part — Deleting the StatefulSet

```bash
# When you delete a StatefulSet, pods are deleted but PVCs are NOT
kubectl delete statefulset stateful-app -n lab10

kubectl get pods -n lab10
# No pods — StatefulSet and pods gone

kubectl get pvc -n lab10
# pod-data-stateful-app-0   Bound   ← PVCs survive StatefulSet deletion
# pod-data-stateful-app-1   Bound
# pod-data-stateful-app-2   Bound

# Recreate the StatefulSet
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: stateful-app
  namespace: lab10
spec:
  serviceName: stateful-app
  replicas: 3
  selector:
    matchLabels:
      app: stateful-app
  template:
    metadata:
      labels:
        app: stateful-app
    spec:
      containers:
      - name: app
        image: busybox:1.35
        command: ["sh", "-c"]
        args:
        - |
          if [ -f /data/identity.txt ]; then
            echo "Data recovered: $(cat /data/identity.txt)"
          else
            echo "No existing data found"
          fi
          sleep 3600
        volumeMounts:
        - name: pod-data
          mountPath: /data
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "64Mi"
            cpu: "100m"
  volumeClaimTemplates:
  - metadata:
      name: pod-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: standard
      resources:
        requests:
          storage: 256Mi
EOF

kubectl rollout status statefulset/stateful-app -n lab10

# Each pod reattaches to its original PVC — data is intact
for i in 0 1 2; do
  echo "=== stateful-app-${i} ==="
  kubectl logs stateful-app-${i} -n lab10 | head -1
done
# Data recovered: pod-0-unique-data
# Data recovered: pod-1-unique-data
# Data recovered: pod-2-unique-data
```

### StatefulSet vs Deployment Storage Comparison

```
                    Deployment              StatefulSet
──────────────────────────────────────────────────────────────
Pod names           Random suffix           Stable: app-0, app-1
PVC per pod         No (shared PVC)         Yes (volumeClaimTemplates)
Pod identity        Interchangeable         Sticky (pod-0 always gets pvc-0)
Scale down          Any pod deleted         Highest ordinal deleted first (app-2, then app-1)
Scale up            New pod, any PVC        app-N gets pvc-N (new or existing)
PVC on delete       N/A                     PVC retained (not deleted with pod)
DNS                 Shared service          Individual: pod-0.svc.ns.svc.cluster.local
Use for             Stateless apps          Databases, queues, ordered stateful apps
```

### Prod Wisdom

**StatefulSet PVCs are your data safety net — never delete them without a backup.** When you scale down a StatefulSet or delete it, PVCs are intentionally left behind. This is protective. The risk is the opposite — engineers running cleanup scripts that delete all PVCs in a namespace after removing a StatefulSet, not realising they're deleting the database storage. Always check `kubectl get pvc` before any namespace cleanup. PVCs that look orphaned after a StatefulSet deletion are intentionally retained.

---

## Scenario 05 — Production Pattern — Storage Lifecycle and Reclaim

### What You'll Build

The correct production approach to storage lifecycle. Understanding `ReclaimPolicy` and what happens to data when PVCs are deleted. Capacity planning. And the storage verification checklist for stateful workloads.

### ReclaimPolicy — What Happens When PVC Is Deleted

```bash
# Check the ReclaimPolicy of available StorageClasses
kubectl get storageclass -o custom-columns=\
"NAME:.metadata.name,PROVISIONER:.provisioner,RECLAIM:.reclaimPolicy,BINDING:.volumeBindingMode"
# NAME       PROVISIONER              RECLAIM   BINDING
# standard   rancher.io/local-path   Delete    WaitForFirstConsumer
```

```
ReclaimPolicy: Delete (common default)
  When PVC is deleted → PV is deleted → underlying storage is deleted
  DATA IS GONE — no recovery
  Use for: ephemeral environments, test clusters, non-critical data

ReclaimPolicy: Retain
  When PVC is deleted → PV status changes to "Released" → storage kept
  DATA IS SAFE — PV must be manually reclaimed or reused
  PV cannot be automatically rebound until manually processed
  Use for: production databases, any data that must survive PVC deletion

ReclaimPolicy: Recycle (deprecated — don't use)
  Scrubs the volume and makes it available again
  Not supported by most modern provisioners
```

### Observe Delete Policy

```bash
# Create a PVC with the default Delete policy
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: delete-policy-pvc
  namespace: lab10
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: standard   # standard uses Delete policy
  resources:
    requests:
      storage: 256Mi
EOF

# Trigger binding with a pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: delete-policy-pod
  namespace: lab10
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c", "echo 'critical data' > /data/db.sql && sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: delete-policy-pvc
EOF

kubectl wait --for=condition=ready pod/delete-policy-pod -n lab10 --timeout=60s

# Get the PV name that was dynamically provisioned
PV_NAME=$(kubectl get pvc delete-policy-pvc -n lab10 \
  -o jsonpath='{.spec.volumeName}')
echo "PV Name: ${PV_NAME}"

kubectl get pv ${PV_NAME}
# STATUS: Bound   RECLAIM POLICY: Delete

# Delete the pod first, then the PVC
kubectl delete pod delete-policy-pod -n lab10
kubectl delete pvc delete-policy-pvc -n lab10

# Check what happened to the PV
kubectl get pv ${PV_NAME} 2>/dev/null || echo "PV was automatically deleted"
# PV was automatically deleted — and so was all data on it
```

### Retain Policy — Protecting Production Data

```bash
# Create a StorageClass with Retain policy for production data
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard-retain
provisioner: rancher.io/local-path   # Same provisioner as standard
reclaimPolicy: Retain                # Data survives PVC deletion
volumeBindingMode: WaitForFirstConsumer
EOF

# Create a PVC using the Retain StorageClass
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: retain-pvc
  namespace: lab10
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: standard-retain
  resources:
    requests:
      storage: 256Mi
EOF

# Bind it with a pod and write data
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: retain-pod
  namespace: lab10
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c"]
    args:
    - |
      echo "writing production data"
      echo "SELECT * FROM orders WHERE status='pending'" > /data/backup.sql
      cat /data/backup.sql
      sleep 3600
    volumeMounts:
    - name: data
      mountPath: /data
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: retain-pvc
EOF

kubectl wait --for=condition=ready pod/retain-pod -n lab10 --timeout=60s

RETAIN_PV=$(kubectl get pvc retain-pvc -n lab10 \
  -o jsonpath='{.spec.volumeName}')

# Delete pod and PVC
kubectl delete pod retain-pod -n lab10
kubectl delete pvc retain-pvc -n lab10

# PV is NOT deleted — status changes to Released
kubectl get pv ${RETAIN_PV}
# STATUS: Released   RECLAIM POLICY: Retain
# Data is safe — PV still exists

# To reuse this PV: remove the claimRef so it becomes Available again
kubectl patch pv ${RETAIN_PV} \
  --type=json \
  -p='[{"op":"remove","path":"/spec/claimRef"}]'

kubectl get pv ${RETAIN_PV}
# STATUS: Available — can now be claimed by a new PVC
```

### The Production Storage Checklist

```
Before deploying any stateful workload:
  □ Identify the StorageClass and its ReclaimPolicy
  □ Confirm the provisioner is healthy and can satisfy the request
  □ Confirm access mode matches usage pattern (RWO for single replica)
  □ Use StatefulSet + volumeClaimTemplates for multi-replica stateful apps
  □ Never use a single RWO PVC in a multi-replica Deployment
  □ Set ReclaimPolicy: Retain for production databases
  □ Label PVCs for easy identification: kubectl label pvc <name> env=prod tier=database
  □ Set up regular backup jobs (CronJob + velero or custom)
  □ Test data recovery before you need it in prod

Storage debug flow:
  □ kubectl get pvc -n <ns>               → STATUS: Pending or Bound?
  □ kubectl describe pvc <name> -n <ns>   → Events: provisioning error?
  □ kubectl get pv                        → PV exists? Status?
  □ kubectl get storageclass              → StorageClass exists?
  □ kubectl describe pod <name> -n <ns>   → FailedMount? Volume attach error?
```

---

## Key Commands Reference — Lab 10

```bash
# PVC and PV status
kubectl get pvc -n <namespace>
kubectl get pv
kubectl describe pvc <name> -n <namespace>
kubectl describe pv <name>

# StorageClass inspection
kubectl get storageclass
kubectl describe storageclass <name>

# StatefulSet storage
kubectl get statefulset -n <namespace>
kubectl get pvc -n <namespace> -l <label-selector>

# Check what PV a PVC is bound to
kubectl get pvc <name> -n <namespace> \
  -o jsonpath='{.spec.volumeName}'

# Check what PVC a PV is bound to
kubectl get pv <name> \
  -o jsonpath='{.spec.claimRef.name}'

# Check volume mounts in a pod
kubectl describe pod <name> -n <namespace> | grep -A 10 "Volumes:"
kubectl exec <pod> -n <namespace> -- df -h
kubectl exec <pod> -n <namespace> -- mount | grep <mountpath>

# Check if data exists in a mounted volume
kubectl exec <pod> -n <namespace> -- ls -la /data/
kubectl exec <pod> -n <namespace> -- cat /data/<file>

# Patch PV to make it Available again (Retain policy)
kubectl patch pv <pv-name> \
  --type=json \
  -p='[{"op":"remove","path":"/spec/claimRef"}]'

# Watch PVC binding
kubectl get pvc -n <namespace> -w
```

---

## Prod Wisdom — The Senior Engineer Mindset

Three things that separate senior engineers in storage management:

**1. They check ReclaimPolicy before touching PVCs in prod.** `kubectl get storageclass` shows the reclaim policy. `Delete` means data is gone when the PVC is deleted — no undo, no recovery. `Retain` means the PV lives on. In prod, databases should always use `Retain`. Knowing this before running `kubectl delete pvc` is the difference between a maintenance window and a data loss incident.

**2. They always use StatefulSets for stateful workloads.** Deployments are for stateless apps. StatefulSets give each pod a stable identity, a stable DNS name, and its own PVC that follows it through restarts and rescheduling. Using a Deployment with a shared RWO PVC for a database is a guarantee of future pain — either access mode conflicts in multi-node setups or shared-write corruption.

**3. They verify storage before go-live, not during an incident.** Write data, delete the pod, recreate the pod, verify data survived. This 2-minute test before every stateful deployment launch catches emptyDir vs PVC mistakes, wrong StorageClass, and ReclaimPolicy surprises. Run it in staging, run it in prod's first deploy, run it every time.

The storage debug order:
```
kubectl get pvc          → Pending or Bound?
kubectl describe pvc     → provisioning error? wrong StorageClass?
kubectl get pv           → PV exists? Released or Bound?
kubectl get storageclass → provisioner running? ReclaimPolicy?
kubectl describe pod     → FailedMount? attach/detach error?
kubectl exec -- df -h    → is the volume actually mounted and writable?
```

---

## Cleanup

```bash
kubectl delete namespace lab10

# PVs may remain if using Retain policy — clean up manually
kubectl get pv | grep Released | awk '{print $1}' | xargs kubectl delete pv 2>/dev/null || true
```

---

*Lab 10 complete. Move to Lab 11 — Node Troubleshooting when ready.*