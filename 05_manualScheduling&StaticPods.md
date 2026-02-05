# Kubernetes Pod Scheduling: Manual vs Static Pods

## First: Fix the Mental Model (Root Problem)

Kubernetes has **three separate actors** involved with Pods:

1. **API Server** – stores desired state (YAMLs)
2. **Scheduler** – decides *which node* a pod should run on
3. **Kubelet** – actually *runs* containers on a node

Most confusion comes from mixing these up.

### Normal Flow (Default Case)

```
kubectl apply
    ↓
API Server stores Pod (no node yet)
    ↓
Scheduler picks a node
    ↓
Scheduler writes nodeName
    ↓
Kubelet on that node runs the Pod
```

Keep this pipeline in your head. Now we'll break it *intentionally* in two different ways.

---

## Part 1: Manual Scheduling (Bypassing ONLY the Scheduler)

### What You Are Actually Doing

When you write:

```yaml
spec:
  nodeName: my-second-cluster-control-plane-66
```

You are saying:

> "Hey Kubernetes, **don't ask the scheduler**. I already decided the node."

### What Still Exists

| Component | Present? |
|-----------|----------|
| API Server | ✅ YES |
| Scheduler | ❌ BYPASSED |
| Kubelet | ✅ YES |

The new flow becomes:

```
kubectl apply
    ↓
API Server stores Pod (nodeName already set)
    ↓
Scheduler IGNORES it
    ↓
Kubelet on that node sees it and runs it
```

That's it. Nothing magical.

---

## Important Correction (Your Course Slightly Misled You)

> ❌ "Kubernetes deletes the Pod if nodeName is wrong"
>
> **Not accurate.**

### What Actually Happens

If `nodeName`:
* **does not exist**
* **kubelet is not running**
* **node is unreachable**

➡️ The Pod stays in **Pending forever**

It is **NOT garbage-collected by default**. Why? Because Kubernetes assumes *you intentionally hard-bound it*.

This is important for CKA.

---

## "But Control Plane Nodes Are Tainted – How Did It Still Run?"

This is the **single most important insight** 🔥

### Taints Are Enforced by the Scheduler, Not the Kubelet

Since you **bypassed the scheduler**, taints are irrelevant.

So:

```yaml
nodeName: control-plane
```

works even without tolerations.

📌 **Rule to remember:**

> **nodeName ignores taints, affinity, scheduler rules — everything**

---

## When Should You Use Manual Scheduling?

Think like a senior engineer:

✅ Debugging scheduler issues  
✅ Testing node-specific behavior  
❌ Production workloads (bad practice)

**You almost never use this in real systems.**

---

## Part 2: Static Pods (Bypassing API Server + Scheduler)

This is a **different beast**.

### Who Creates the Pod?

❌ Not kubectl  
❌ Not API Server  
❌ Not Scheduler  
✅ **Kubelet directly**

### How?

Kubelet watches a directory:

```bash
/etc/kubernetes/manifests/
```

Any YAML placed there → kubelet runs it.

---

## Static Pod Flow (This Is Key)

```
You put YAML on disk
    ↓
Kubelet detects file
    ↓
Kubelet runs container
    ↓
(Optional) Kubelet creates a mirror pod in API server
```

Notice:
* API Server is **optional**
* Scheduler is **never involved**

---

## Why Static Pods Exist (Real Reason)

Kubernetes has a **bootstrap problem**:

> "How do you run kube-apiserver *before* Kubernetes exists?"

**Answer: static pods**

That's why these run as static pods:
* kube-apiserver
* etcd
* kube-scheduler
* kube-controller-manager

They must exist **even if the cluster is broken**.

### The Real Reasons Static Pods Exist

1. **Bootstrapping the control plane** – This is the biggest reason. Core components like kube-apiserver, kube-controller-manager, kube-scheduler, and etcd are often run as static Pods. Why? Because the API server can't manage itself. Something has to start before the cluster is fully alive.

Static Pods let the control plane bring itself up from nothing.

**Static Pods exist so Kubernetes can run its most critical components without depending on Kubernetes itself.**

---

## Mirror Pods (Just Visibility, Nothing More)

### Mirror Pod Facts

| Feature | Mirror Pod |
|---------|-----------|
| Created by | Kubelet |
| Editable | ❌ No |
| Deletable | ❌ No |
| Purpose | Visibility only |

### Deleting It

```bash
kubectl delete pod xyz
```

➡️ kubelet recreates it instantly  
➡️ because **the file still exists**

---

## This Is Why `kubectl delete` "Doesn't Work"

Because you're deleting the **shadow**, not the source.

Real source = file on disk

```bash
rm /etc/kubernetes/manifests/static-pod.yaml
```

➡️ pod dies permanently

---

## Manual Scheduling vs Static Pods (The Real Difference)

Forget tables. Remember **who owns the pod**.

| Question | Manual Scheduling | Static Pod |
|----------|-------------------|-----------|
| Who creates pod? | API Server | Kubelet |
| Who decides node? | You | Kubelet (local) |
| Scheduler involved? | ❌ | ❌ |
| Survives API server down? | ❌ | ✅ |
| Used in production? | Rare | Core Kubernetes |

---

## Let's Map YOUR YAML Now

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-manual-3
spec:
  nodeName: my-second-cluster-control-plane-66
```

This means:

* API Server stores it ✅
* Scheduler is skipped ❌
* Kubelet on control plane runs it ✅
* Taints ignored ✅
* If that node dies → pod dies (no reschedule) ❌
