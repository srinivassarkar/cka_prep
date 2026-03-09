# Kubernetes Pod Lifecycle & Debugging Lab
---

# Lab 0 — Cluster Check

Before starting.

```bash
kubectl get nodes
```

Expected:

```
NAME       STATUS   ROLES
node01     Ready    control-plane
```

---

# Lab 1 — Observe Pod Lifecycle (Fast Job)

### Goal

See a **short-lived pod lifecycle**.

---

### Run Pod

```bash
kubectl run ubuntu-ls \
  --image=ubuntu \
  --restart=Never \
  -- ls
```

---

### Watch Lifecycle

In another terminal:

```bash
kubectl get pods -w
```

Expected flow:

```
Pending
ContainerCreating
Completed
```

Notice:

```
Running phase might not appear
```

because the container finished too quickly.

---

### Investigate Pod

```bash
kubectl describe pod ubuntu-ls
```

Look for:

```
Status: Succeeded
```

But in `kubectl get pods` you see:

```
Completed
```

🧠 Important:

```
Succeeded = internal phase
Completed = kubectl display
```

---

# Lab 2 — Observe Full Lifecycle

Now run a longer container.

---

### Run Pod

```bash
kubectl run ubuntu-sleep \
  --image=ubuntu \
  --restart=Never \
  -- sleep 10
```

---

### Watch lifecycle

```bash
kubectl get pods -w
```

You will see:

```
Pending
ContainerCreating
Running
Completed
```

Now you observed the **full lifecycle**.

---

# Lab 3 — Restart Policy Investigation

Restart policy determines **what happens after container exit**.

---

## Create Pod

File: `restart-never.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restart-never-demo
spec:
  restartPolicy: Never
  containers:
  - name: fail-container
    image: busybox
    command: ["sh","-c","echo crashing; exit 1"]
```

Apply:

```bash
kubectl apply -f restart-never.yaml
```

Check:

```bash
kubectl get pods
```

Result:

```
STATUS: Error
```

Container **does NOT restart**.

---

# Lab 4 — Restart Policy Always

Now change restart policy.

---

File: `restart-always.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: restart-always-demo
spec:
  restartPolicy: Always
  containers:
  - name: fail-container
    image: busybox
    command: ["sh","-c","echo crashing; exit 1"]
```

Apply:

```bash
kubectl apply -f restart-always.yaml
```

Check:

```bash
kubectl get pods
```

You will see:

```
CrashLoopBackOff
```

Why?

Because container keeps restarting.

---

### Check restart count

```bash
kubectl get pod restart-always-demo
```

Example:

```
RESTARTS: 5
```

---

### Check logs

```bash
kubectl logs restart-always-demo
```

or previous container:

```bash
kubectl logs restart-always-demo --previous
```

---

🧠 Important

CrashLoopBackOff means:

```
container crashes
kubelet restarts it
repeat
```

Backoff delay increases.

---

# Lab 5 — Simulate ImagePullBackOff

Now simulate **wrong image**.

---

File: `bad-image.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: bad-image-demo
spec:
  containers:
  - name: app
    image: nginx:fake-tag
```

Apply:

```bash
kubectl apply -f bad-image.yaml
```

Check:

```bash
kubectl get pods
```

Result:

```
ErrImagePull
ImagePullBackOff
```

---

### Investigate

```bash
kubectl describe pod bad-image-demo
```

Look at:

```
Events:
Failed to pull image
```

This is **how real debugging happens**.

---

# Lab 6 — Pod Termination Behavior

Now test **SIGTERM → SIGKILL flow**.

---

Create pod.

File: `termination-demo.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: termination-demo
spec:
  terminationGracePeriodSeconds: 30
  containers:
  - name: app
    image: nginx
```

Apply:

```bash
kubectl apply -f termination-demo.yaml
```

---

### Delete Pod

```bash
kubectl delete pod termination-demo
```

Kubernetes flow:

```
SIGTERM
(wait 30 seconds)
SIGKILL
```

---

### Force Delete

```bash
kubectl delete pod termination-demo --force=true --grace-period=0
```

Now:

```
Immediate kill
```

No graceful shutdown.

---

# Lab 7 — Image Pull Policy Investigation

Create:

File: `image-policy.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: image-policy-demo
spec:
  containers:
  - name: app
    image: nginx:latest
    imagePullPolicy: Always
```

Apply:

```bash
kubectl apply -f image-policy.yaml
```

Every restart:

```
image pulled again
```

---

# Important Real-World Debug Workflow

When a pod fails, always follow this order:

### Step 1 — Check pods

```bash
kubectl get pods
```

---

### Step 2 — Describe pod

```bash
kubectl describe pod POD_NAME
```

Look at:

```
Events
```

---

### Step 3 — Check logs

```bash
kubectl logs POD_NAME
```

or

```bash
kubectl logs POD_NAME --previous
```

---

### Step 4 — Exec inside

```bash
kubectl exec -it POD_NAME -- sh
```

---

# Real DevOps Mental Model

Pod lifecycle:

```
Pending
  ↓
ContainerCreating
  ↓
Running
  ↓
Succeeded / Failed
```

---

# Common Real Errors

| Error            | Meaning                      |
| ---------------- | ---------------------------- |
| CrashLoopBackOff | container crashes repeatedly |
| ImagePullBackOff | image cannot be pulled       |
| ErrImagePull     | immediate image failure      |
| OOMKilled        | container ran out of memory  |
| Pending          | scheduler can't place pod    |

---

# CKA Exam Memory Cheatsheet

```
Always → long running apps
OnFailure → jobs
Never → debug pods
```

Image pull:

```
latest → Always
version tag → IfNotPresent
```

---
Below is a **clean Advanced Section** you can append to the same GitHub markdown doc.
It keeps the **same lab-notebook style** and focuses on **real DevOps debugging incidents**, which is perfect for CKA prep.

---

# Advanced Section — Real-World Kubernetes Debugging Scenarios

These scenarios simulate **actual production incidents** that DevOps engineers investigate.

You will practice:

* diagnosing scheduling issues
* debugging container crashes
* troubleshooting image registry problems

Each lab follows the **standard debugging workflow**.

```
1. kubectl get pods
2. kubectl describe pod
3. kubectl logs
4. investigate events
```

---

# Scenario 1 — Pod Stuck in Pending for 30 Minutes

### Goal

Simulate a **scheduler failure** where a pod cannot be scheduled.

Common real causes:

* insufficient CPU/memory
* node selector mismatch
* taints/tolerations
* node unavailable

---

## Create Pod with Impossible NodeSelector

File: `pending-pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pending-demo
spec:
  nodeSelector:
    disktype: ssd
  containers:
  - name: nginx
    image: nginx
```

Apply:

```bash
kubectl apply -f pending-pod.yaml
```

---

## Observe Pod Status

```bash
kubectl get pods
```

Result:

```
pending-demo   0/1   Pending
```

The pod never schedules.

---

## Investigate the Issue

Run:

```bash
kubectl describe pod pending-demo
```

Look at **Events**.

Example:

```
0/1 nodes are available: 1 node(s) didn't match node selector.
```

This means:

```
scheduler cannot find a node with label disktype=ssd
```

---

## Fix the Problem

Check node labels:

```bash
kubectl get nodes --show-labels
```

Add label:

```bash
kubectl label node node01 disktype=ssd
```

Now the pod schedules.

Check:

```bash
kubectl get pods -w
```

Expected:

```
Pending → ContainerCreating → Running
```

---

## Key Lesson

Pods remain **Pending** when the **scheduler cannot place them on any node**.

Common real-world causes:

| Cause                  | Example                    |
| ---------------------- | -------------------------- |
| CPU shortage           | resource requests too high |
| Node selector mismatch | wrong labels               |
| Taints                 | pod missing toleration     |
| Node offline           | node not Ready             |

---

# Scenario 2 — Pod CrashLoopBackOff Due to Bad Command

This is **one of the most common production incidents**.

The container starts but **immediately crashes**.

---

## Create Broken Pod

File: `bad-command.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: crashloop-demo
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh","-c","exit 1"]
```

Apply:

```bash
kubectl apply -f bad-command.yaml
```

---

## Check Pod Status

```bash
kubectl get pods
```

Result:

```
crashloop-demo   0/1   CrashLoopBackOff
```

The container repeatedly crashes.

---

## Investigate Restart Count

```bash
kubectl get pod crashloop-demo
```

Example:

```
RESTARTS: 5
```

Kubernetes keeps restarting it.

---

## Check Container Logs

```bash
kubectl logs crashloop-demo
```

If container restarts too fast:

```bash
kubectl logs crashloop-demo --previous
```

---

## Inspect Pod Details

```bash
kubectl describe pod crashloop-demo
```

Look for:

```
Last State: Terminated
Exit Code: 1
```

Meaning:

```
container exited with failure
```

---

## Fix the Issue

Change command to something valid.

Example:

```yaml
command: ["sh","-c","while true; do echo running; sleep 10; done"]
```

Apply updated YAML.

Pod will run normally.

---

## Key Lesson

CrashLoopBackOff occurs when:

```
container starts
container crashes
kubelet restarts it
repeat
```

Backoff delay increases after every restart.

Common real-world causes:

| Cause                | Example          |
| -------------------- | ---------------- |
| bad startup command  | incorrect script |
| missing env variable | config issue     |
| app crash            | runtime bug      |
| missing dependency   | DB not reachable |

---

# Scenario 3 — ImagePullBackOff Due to Private Registry

This simulates **authentication failure** when pulling container images.

Common in enterprise clusters.

---

## Create Pod Using Private Image

File: `private-image.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-registry-demo
spec:
  containers:
  - name: app
    image: private-registry/myapp:v1
```

Apply:

```bash
kubectl apply -f private-image.yaml
```

---

## Check Pod Status

```bash
kubectl get pods
```

Result:

```
ErrImagePull
ImagePullBackOff
```

---

## Investigate the Error

Run:

```bash
kubectl describe pod private-registry-demo
```

Look at **Events**.

Example:

```
Failed to pull image
unauthorized: authentication required
```

Meaning:

```
cluster cannot access private registry
```

---

## Fix Using ImagePullSecret

First create secret.

```bash
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.example.com \
  --docker-username=user \
  --docker-password=password
```

Then update pod.

```yaml
spec:
  imagePullSecrets:
  - name: regcred
```

Reapply YAML.

Pod now pulls image successfully.

---

## Key Lesson

ImagePullBackOff means:

```
kubelet failed to download container image
```

Common causes:

| Cause            | Example              |
| ---------------- | -------------------- |
| wrong image name | typo                 |
| wrong tag        | version not exists   |
| private registry | auth required        |
| network issues   | registry unreachable |

---

# Real DevOps Debugging Workflow

Always follow this order:

### Step 1 — Check pods

```bash
kubectl get pods
```

---

### Step 2 — Describe pod

```bash
kubectl describe pod POD_NAME
```

Check:

```
Events
```

---

### Step 3 — Check logs

```bash
kubectl logs POD_NAME
```

Or previous container:

```bash
kubectl logs POD_NAME --previous
```

---

### Step 4 — Exec inside container

```bash
kubectl exec -it POD_NAME -- sh
```

---

# Kubernetes Failure Pattern Cheatsheet

| Error            | Meaning                     |
| ---------------- | --------------------------- |
| Pending          | scheduler cannot place pod  |
| CrashLoopBackOff | container keeps crashing    |
| ErrImagePull     | image pull failed           |
| ImagePullBackOff | repeated pull failures      |
| OOMKilled        | container ran out of memory |

---

# DevOps Mental Model for Pod Failures

```
Pending
 ↓
ContainerCreating
 ↓
Running
 ↓
Succeeded / Failed
```

Failures can occur at **any stage**.

---

# Final Takeaway

These three scenarios represent **some of the most common Kubernetes incidents**:

1️⃣ Scheduling failure → **Pending**
2️⃣ Application crash → **CrashLoopBackOff**
3️⃣ Registry failure → **ImagePullBackOff**


---
