# 🧠 PV and PVC : CORE MENTAL MODEL (Never Forget This)

```
Pod → PVC → PV → StorageClass → CSI Driver → Actual Disk
```

👉 Think like infra flow, not YAML.

* **Pod = wants storage**
* **PVC = request**
* **PV = actual disk**
* **StorageClass = automation**
* **CSI = plugin talking to cloud/on-prem**

---

# 🔥 1. STORAGE TYPES (Interview + Prod Critical)

### ❌ emptyDir

* Pod dies → data gone
  👉 use for temp/cache

### ⚠️ hostPath

* Node-level storage
* Same node → shared
* Different node → 💀 data lost

👉 **Prod Rule:**

> If pod reschedules → data gone → incident

### ✅ PV + PVC (REAL WORLD)

* Decouples compute from storage
* Data survives pod restart

👉 **Golden rule:**

> Pod is replaceable, data is not

---

# ⚡ 2. PV vs PVC (VERY IMPORTANT)

| Concept | Who creates     | Meaning        |
| ------- | --------------- | -------------- |
| PV      | Admin / Dynamic | Actual storage |
| PVC     | Developer       | Request        |

---

### 🧠 Binding Logic (INTERVIEW GOLD)

PVC will bind to PV ONLY if:

* storage ≥ requested
* accessMode matches
* storageClass matches
* PV is **Available**

👉 If not → `Pending`

---

# 🚨 REAL OUTAGE SCENARIOS

### ❗ PVC stuck in Pending

Check:

```bash
kubectl describe pvc
```

👉 Common reasons:

* No PV available
* StorageClass mismatch
* accessModes mismatch

---

### ❗ Pod stuck in Pending (VERY COMMON)

```bash
kubectl describe pod
```

Look for:

```
waiting for first consumer
```

👉 Meaning:

* StorageClass = `WaitForFirstConsumer`
* Pod not scheduled yet

---

### ❗ Data lost after restart

👉 Root cause:

* used `emptyDir` or `hostPath`

---

# ⚡ 3. ACCESS MODES (Shortcut Trick)

```
RWO  → one node
RWX  → many nodes
ROX  → read-only many
RWOP → one pod only
```

---

### 🧠 Real-world mapping:

* DB → RWO
* Shared logs → RWX
* Config → ROX

---

# ⚡ 4. RECLAIM POLICY (Production Critical)

| Policy  | Behavior     |
| ------- | ------------ |
| Delete  | disk deleted |
| Retain  | disk stays   |
| Recycle | ❌ deprecated |

---

### 🧠 Interview line:

> "Retain is used for data safety, Delete for automation"

---

# ⚡ 5. STORAGE CLASS (THE REAL MAGIC)

👉 Without StorageClass:

* Admin creates PV manually

👉 With StorageClass:

* PV auto-created

---

### 🧠 Key line (remember this forever):

> **PVC + StorageClass = auto PV creation**

---

### ⚡ WaitForFirstConsumer (VERY IMPORTANT)

Problem:

* Volume in AZ-1
* Pod in AZ-2 → ❌ fail

Solution:

```
WaitForFirstConsumer
```

👉 Volume created AFTER pod scheduling

---

# 🧠 6. CSI (MODERN WAY)

Old:

```
in-tree plugins ❌
```

New:

```
CSI drivers ✅
```

👉 Example:

* AWS → `ebs.csi.aws.com`

---

# 🔥 LAB (DO THIS — NO SKIP)

## 🧪 LAB 1: Break hostPath

### Step 1: Create the Pod

```yaml
# hostpath-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-example
spec:
  containers:
  - name: busybox-container
    image: busybox
    command: ["sh", "-c", "cat /data && sleep 3600"]
    volumeMounts:
    - mountPath: /data
      name: host-volume
  volumes:
  - name: host-volume
    hostPath:
      path: /tmp/hostfile
      type: FileOrCreate
```

```bash
kubectl apply -f hostpath-pod.yaml
```

### Step 2: Write data

```bash
kubectl exec -it hostpath-example -- sh
echo "hello" > /data
```

### Step 3: Verify sharing across pods on same node

First, check which node the pod landed on:

```bash
kubectl get pods -o wide
```

Then schedule a second pod on the **same node** (replace `my-second-cluster-worker2` with your actual node name):

```yaml
# hostpath-verify.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-verify
spec:
  nodeName: my-second-cluster-worker2   # 👈 must match original pod's node
  containers:
  - name: busybox-container
    image: busybox
    command: ["sh", "-c", "cat /data && sleep 3600"]
    volumeMounts:
    - mountPath: /data
      name: host-volume
  volumes:
  - name: host-volume
    hostPath:
      path: /tmp/hostfile
      type: FileOrCreate
```

```bash
kubectl apply -f hostpath-verify.yaml
kubectl exec hostpath-verify -- cat /data
# Expected: hello ✅
```

### Step 4: Delete pod + recreate on different node

👉 Observe:
❌ Data gone (if different node)

---

## 🧪 LAB 2: PV + PVC Binding

### Step 1: Delete default StorageClass (KIND/Minikube only — avoids interference)

```bash
kubectl get sc standard -o yaml > sc.yaml   # backup first!
kubectl delete sc standard
```

### Step 2: Create the PV

```yaml
# pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data
```

```bash
kubectl apply -f pv.yaml
kubectl get pv
kubectl describe pv example-pv
```

### Step 3: Create the PVC

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: example-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

```bash
kubectl apply -f pvc.yaml
kubectl get pvc
```

👉 Expected:

```
STATUS: Bound
```

### Step 4: Verify binding

```bash
kubectl describe pv example-pv
```

👉 Look for:

```
Claim: default/example-pvc
```

### Step 5: Create the Pod

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
spec:
  containers:
  - name: nginx-container
    image: nginx
    volumeMounts:
    - mountPath: /usr/share/nginx/html
      name: persistent-storage
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: example-pvc
```

```bash
kubectl apply -f pod.yaml
kubectl describe pod example-pod
```

---

## 🧪 LAB 3: Simulate Failure

Change PVC to request `ReadWriteMany` while PV only supports `ReadWriteOnce`:

```yaml
# pvc-broken.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: example-pvc
spec:
  accessModes:
    - ReadWriteMany    # 👈 mismatch! PV only has RWO
  resources:
    requests:
      storage: 2Gi
```

👉 Result:

```
STATUS: Pending
```

💡 WHY: PV only supports RWO

---

## 🧪 LAB 4: Dynamic Provisioning (StorageClass)

### Step 1: Restore the default StorageClass

```bash
kubectl apply -f sc.yaml
```

The restored StorageClass looks like this:

```yaml
# sc.yaml (reference)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

### Step 2: Delete old PV + PVC

```bash
kubectl delete pvc example-pvc
kubectl delete pv example-pv
```

### Step 3: Apply PVC ONLY (no PV created manually)

```yaml
# pvc-dynamic.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: example-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  # No storageClassName → uses cluster default (standard)
```

```bash
kubectl apply -f pvc-dynamic.yaml
kubectl get pvc
# STATUS: Pending  (WaitForFirstConsumer — normal!)
```

### Step 4: Create Pod to trigger provisioning

```yaml
# pod-dynamic.yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
spec:
  containers:
  - name: nginx-container
    image: nginx
    volumeMounts:
    - mountPath: /usr/share/nginx/html
      name: persistent-storage
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: example-pvc
```

```bash
kubectl apply -f pod-dynamic.yaml
kubectl get pvc   # now Bound ✅
kubectl get pv    # auto-created PV ✅
```

---

# 🧠 DEBUGGING CHEAT SHEET (OUTAGE MODE)

```bash
kubectl get pvc
kubectl describe pvc

kubectl get pv

kubectl get sc

kubectl describe pod
```

---

### 🔥 Fast Thinking Flow:

```
Pod Pending?
 → check PVC

PVC Pending?
 → check PV / SC

PV not binding?
 → check:
   size
   accessMode
   storageClass
```

---

# 🎯 CKA + INTERVIEW HIT QUESTIONS

### Q: Why PVC Pending?

👉 mismatch OR no PV OR no SC

---

### Q: Difference PV vs PVC?

👉 PV = supply, PVC = demand

---

### Q: Why WaitForFirstConsumer?

👉 AZ alignment

---

### Q: hostPath problem?

👉 not portable, node tied

---

### Q: Retain vs Delete?

👉 data safety vs automation

---

# 🧠 FINAL MEMORY LOCK

```
Stateless → Deployment
Stateful → PVC

Wrong storage = data loss
Wrong SC = pod stuck
Wrong accessMode = no binding
```
