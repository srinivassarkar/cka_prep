## Introduction: Why Taints and Tolerations?  

In Kubernetes, **Taints and Tolerations** help **control which pods can be scheduled on which nodes**. They allow cluster administrators to **prevent certain workloads from running on specific nodes** while allowing exceptions when necessary.  

### **When Do We Need Taints & Tolerations?**  
Taints and tolerations are useful in several scenarios:  

| **Use Case** | **Explanation** |
|-------------|----------------|
| **Dedicated Nodes** | Taint nodes with a GPU to restrict them to **GPU-intensive workloads** only. |
| **Isolating Critical Workloads** | Ensure **critical workloads** run separately from **non-critical** ones. |
| **Preventing Scheduling on Control Plane Nodes** | Control plane nodes should only run **Kubernetes system workloads**. |
| **Maintenance Mode** | Taint a node under **maintenance** to stop new pods from being scheduled. |
| **Node Resource Constraints** | Prevent pods from being scheduled on nodes with **low memory or CPU**. |

### **Important:**  
Taints and tolerations **only apply during scheduling**. If a pod is already running on a node, adding a taint **will not remove the existing pod** (unless you use the **NoExecute** effect).  

---

## Understanding Taints  

A **Taint** is applied to a **node** to indicate that it **should not accept certain pods** unless they explicitly tolerate it.  

---

### **Effects of a Taint**  

| **Effect** | **Behavior** |
|------------|-------------|
| **NoSchedule** | New pods will **not be scheduled** unless they have a matching toleration. |
| **PreferNoSchedule** | Kubernetes **tries** to avoid scheduling pods but **doesn’t enforce it strictly**. |
| **NoExecute** | Existing pods **without a matching toleration will be evicted** from the node. |

---

## Understanding Tolerations  

A **Toleration** is applied to a **pod**, allowing it to **bypass a node’s taint** and be scheduled on it.  

---

# 0️⃣ Verify Cluster

```bash
kubectl get nodes -o wide
```

Expect:

* 1 control-plane
* 1–2 workers

Check taints:

```bash
kubectl describe node <control-plane-node> | grep -i taint
```

You’ll see:

```
node-role.kubernetes.io/control-plane:NoSchedule
```

✔ Control plane already protected.

---

# 1️⃣ Taint Worker Nodes

Pick worker names:

```bash
kubectl get nodes
```

Example:

* worker1
* worker2

Apply taints:

```bash
kubectl taint nodes worker1 storage=ssd:NoSchedule
kubectl taint nodes worker2 storage=hdd:NoSchedule
```

Verify:

```bash
kubectl describe node worker1 | grep -i taint
kubectl describe node worker2 | grep -i taint
```

---

# 2️⃣ Try Scheduling Without Toleration

```bash
kubectl run testpod --image=nginx
kubectl get pods
```

It will stay:

```
Pending
```

Check reason:

```bash
kubectl describe pod testpod
```

Look for:

```
0/2 nodes are available
```

✔ Scheduler blocked it.

Delete it:

```bash
kubectl delete pod testpod
```

---

# 3️⃣ Add Toleration (Exact Match)

Create file:

```bash
vi pod-ssd.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ssd-pod
spec:
  tolerations:
  - key: "storage"
    operator: "Equal"
    value: "ssd"
    effect: "NoSchedule"
  containers:
  - name: nginx
    image: nginx
```

Apply:

```bash
kubectl apply -f pod-ssd.yaml
kubectl get pods -o wide
```

✔ Should land on **worker1**

---

# 4️⃣ Exists Operator

Delete previous:

```bash
kubectl delete pod ssd-pod
```

Create:

```bash
vi pod-exists.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: exists-pod
spec:
  tolerations:
  - key: "storage"
    operator: "Exists"
    effect: "NoSchedule"
  containers:
  - name: nginx
    image: nginx
```

Apply:

```bash
kubectl apply -f pod-exists.yaml
kubectl get pods -o wide
```

✔ Can land on worker1 OR worker2

---

# 5️⃣ Test NoExecute (Eviction)

Add new taint:

```bash
kubectl taint nodes worker1 critical=true:NoExecute
```

Check running pod:

```bash
kubectl get pods -o wide
```

If pod doesn’t tolerate → it gets **evicted immediately**

Now test tolerationSeconds.

Create:

```bash
vi pod-noexecute.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: noexecute-pod
spec:
  tolerations:
  - key: "critical"
    operator: "Equal"
    value: "true"
    effect: "NoExecute"
    tolerationSeconds: 30
  containers:
  - name: nginx
    image: nginx
```

Apply:

```bash
kubectl apply -f pod-noexecute.yaml
kubectl get pods -w
```

✔ After 30 seconds → evicted.

---

# 6️⃣ PreferNoSchedule Behavior

Remove old taints:

```bash
kubectl taint nodes worker1 storage=ssd:NoSchedule-
kubectl taint nodes worker2 storage=hdd:NoSchedule-
```

Add soft taint:

```bash
kubectl taint nodes worker1 soft=true:PreferNoSchedule
```

Create simple pod:

```bash
kubectl run normalpod --image=nginx
kubectl get pods -o wide
```

✔ Scheduler prefers other nodes first
✔ If no option → will use worker1

---

# 7️⃣ Taints + Node Affinity (Production Pattern)

Real production = **Taint + NodeAffinity**

Label node:

```bash
kubectl label nodes worker1 env=prod
```

Create:

```bash
vi deploy.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prod-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: prod
  template:
    metadata:
      labels:
        app: prod
    spec:
      tolerations:
      - key: "soft"
        operator: "Equal"
        value: "true"
        effect: "PreferNoSchedule"
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: env
                operator: In
                values:
                - prod
      containers:
      - name: nginx
        image: nginx
```

Apply:

```bash
kubectl apply -f deploy.yaml
kubectl get pods -o wide
```

✔ Now scheduling is strict.

---

# 8️⃣ Clean Up Everything

```bash
kubectl delete deploy prod-app
kubectl delete pod exists-pod normalpod noexecute-pod
kubectl taint nodes worker1 soft=true:PreferNoSchedule-
kubectl taint nodes worker1 critical=true:NoExecute-
kubectl label nodes worker1 env-
```
---

# 🧠 Production Rule (Remember This)

Taint = repel
Toleration = allow
Affinity = force

If you rely only on tolerations → pod may still go to untainted nodes.
If you want strict segregation → **Taint + Required NodeAffinity**

---

### **Real-World Use Cases for This Segregation**  
This type of isolation can be based on:  
- **Projects:** Different teams using dedicated nodes.  
- **Departments:** Keeping workloads from Finance, HR, and IT separate.  
- **Environments:** Ensuring **production** workloads do not mix with **development** ones.  
- **Criticality:** Reserving high-performance nodes for **mission-critical applications**.  

By combining **taints, tolerations, node affinity, and anti-affinity**, we can **enforce stricter placement policies** and **ensure workloads run on appropriate nodes** based on project requirements and business logic.
We will discuss **node affinity and anti-affinity** in the next lecture.


---

## Key Takeaways  

- **Taints are applied to nodes**, **Tolerations are applied to pods**.  
- A **pod can only be scheduled** on a tainted node if it has a **matching toleration**.  
- **Three effects of taints:** `NoSchedule`, `PreferNoSchedule`, and `NoExecute`.  
- **Operator `Equal` (default)** requires an exact match, while **`Exists` ignores values**.  
- **Taints and tolerations work together** to control **pod placement and node access**.

---
