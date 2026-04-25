# Kubernetes Volumes | Ephemeral Storage | emptyDir & Downward API

## 🎯 Objective

Understand:

* Ephemeral storage in Kubernetes
* `emptyDir` (shared temp storage)
* Downward API (pod metadata injection)

---

## 📌 Core Concept

```
Ephemeral Storage = data lives only till Pod exists
Pod deleted → data gone
```

---

# 1️⃣ Ephemeral Storage

* Temporary storage tied to Pod lifecycle
* Deleted when Pod is deleted
* Used for:

  * Scratch space
  * Caching
  * Inter-container communication

---

# 2️⃣ emptyDir Volume

## 🧠 What it is

```
emptyDir = shared writable directory for containers in SAME pod
```

* Created when Pod starts
* Deleted when Pod is deleted
* Each Pod gets its own emptyDir (even in Deployment)

---

## ⚡ Why needed (Important)

Without volumes:

```
Each container → isolated writable layer
NO sharing
```

With emptyDir:

```
Shared writable space across containers
```

---

## 🔥 Use Cases

* Log sharing between containers
* Sidecar communication
* Temporary processing data

---

## 🧪 Demo: emptyDir

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-example
spec:
  containers:
  - name: busybox-container
    image: busybox
    command: ["/bin/sh", "-c", "sleep 3600"]
    volumeMounts:
    - mountPath: /data
      name: temp-storage

  - name: busybox-container-2
    image: busybox
    command: ["/bin/sh", "-c", "sleep 3600"]
    volumeMounts:
    - mountPath: /data
      name: temp-storage

  volumes:
  - name: temp-storage
    emptyDir: {}
```

---

## ✅ Verification

### Step 1 — Write file

```bash
kubectl exec emptydir-example -c busybox-container -- sh -c 'echo "What a file!" > /data/myfile.txt'
```

---

### Step 2 — Read from another container

```bash
kubectl exec emptydir-example -c busybox-container-2 -- cat /data/myfile.txt
```

**Output:**

```
What a file!
```

---

## 🧠 Key Takeaway

```
emptyDir = shared storage inside a Pod
```

---


# 3️⃣ Downward API

## 🧠 What it is

```
Downward API = Pod can read its own metadata
```

Example metadata:

* Pod name
* Namespace
* Labels
* Annotations

---

## ⚡ Why needed

* Dynamic config (no hardcoding)
* Self-aware containers
* Used heavily in sidecars

---

## 🧪 Demo 1: Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downwardapi-example
  labels:
    app: demo
spec:
  containers:
  - name: metadata-container
    image: busybox
    command: ["/bin/sh", "-c", "env && sleep 3600"]
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
```

---

## ✅ Verification

```bash
kubectl exec downwardapi-example -- env | grep POD_
```

**Output:**

```
POD_NAME=downwardapi-example
POD_NAMESPACE=default
```

---

## 🧠 Key Insight

```
Metadata injected as ENV variables
```

---

# 🧪 Demo 2: Files via Volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downwardapi-volume
  labels:
    app: good_app
    owner: hr
  annotations:
    version: "good_version"
spec:
  containers:
  - name: metadata-container
    image: busybox
    command: ["/bin/sh", "-c", "cat /etc/podinfo/* && sleep 3600"]
    volumeMounts:
    - name: downwardapi-volume
      mountPath: /etc/podinfo

  volumes:
  - name: downwardapi-volume
    downwardAPI:
      items:
      - path: "labels"
        fieldRef:
          fieldPath: metadata.labels
      - path: "annotations"
        fieldRef:
          fieldPath: metadata.annotations
```

---

## ✅ Verification

### Step 1 — Exec into Pod

```bash
kubectl exec -it downwardapi-volume -- sh
```

```bash
cd /etc/podinfo
ls -l
```

Expected:

```
labels -> ..data/labels
annotations -> ..data/annotations
```

---

### Step 2 — Read files

```bash
cat labels
```

```
app="good_app"
owner="hr"
```

```bash
cat annotations
```

```
version="good_version"
```

---

### Step 3 — Logs

```bash
kubectl logs downwardapi-volume
```

---

## 🧠 Key Insight

```
Metadata injected as FILES
```

---

# ⚠️ Important Concepts (Interview + CKA)

## emptyDir

```
✔ Pod-level storage
✔ Shared across containers
❌ Not persistent
```

---

## Downward API

```
✔ Metadata injection
✔ Env OR Files
✔ No API server calls needed
```

---

## Comparison

| Feature     | emptyDir     | Downward API    |
| ----------- | ------------ | --------------- |
| Purpose     | Data sharing | Metadata access |
| Scope       | Pod          | Pod metadata    |
| Persistence | No           | No              |
| Type        | Volume       | Env / Volume    |

---

# 🧠 Real DevOps Thinking

```
Need shared data between containers?
→ emptyDir

Need pod metadata inside container?
→ Downward API
```

---

# 🚀 What’s Next

```
ConfigMaps → config injection
Secrets → sensitive data
```

---

# 🧾 Final Memory Hook

```
emptyDir → share data
Downward API → know yourself
```
