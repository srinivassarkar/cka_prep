# Kubernetes Multi-Container Pods — Investigative Lab (Init + Sidecar)

*(Hands-on Observational Learning for CKA)*

---

# 0️⃣ Lab Goal

Understand how **multi-container pods behave internally**.

We will investigate:

| Pattern                  | What we observe                  |
| ------------------------ | -------------------------------- |
| Init Container           | Pod startup blocking behavior    |
| Multiple Init Containers | Sequential execution             |
| Sidecar                  | Runtime cooperation              |
| Shared Network           | localhost communication          |
| Failure Behavior         | What happens when containers die |

---

# 1️⃣ Environment Check

Before starting.

```bash
kubectl get nodes
```

Expected:

```
NAME        STATUS   ROLES           AGE
node01      Ready    control-plane
```

---

# 2️⃣ Lab 1 — Init Container (Dependency Gate)

### Goal

Prove that:

```
Init container MUST finish before main container starts
```

---

# File: `init-demo.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
  - name: check-api
    image: curlimages/curl:latest
    command:
    - sh
    - -c
    - |
      echo "Checking API availability..."
      sleep 20
      until curl -s https://kubernetes.io > /dev/null; do
        echo "Waiting for API..."
        sleep 5
      done
      echo "API reachable!"

  containers:
  - name: main-app
    image: nginx:latest
```

---

# Apply

```bash
kubectl apply -f init-demo.yaml
```

---

# Observe Pod Lifecycle

Immediately check:

```bash
kubectl get pods
```

You should see:

```
NAME        READY   STATUS     RESTARTS
init-demo   0/1     Init:0/1
```

**Observation**

```
Pod status shows Init phase
Main container not started yet
```

---

# Investigate Init Container

```bash
kubectl describe pod init-demo
```

Look at:

```
Init Containers:
  check-api
```

and events.

---

# Check Init Logs

```bash
kubectl logs init-demo -c check-api
```

Expected:

```
Checking API availability...
Waiting for API...
API reachable!
```

---

# After Completion

Run again:

```bash
kubectl get pods
```

Expected:

```
NAME        READY   STATUS    RESTARTS
init-demo   1/1     Running
```

Main container **starts only now**.

---

# Key Observation

Init containers act like:

```
startup gates
```

Main container cannot start until **ALL init containers succeed**.

---

# 3️⃣ Lab 2 — Multiple Init Containers (Sequential Execution)

### Goal

Prove:

```
Init containers run ONE BY ONE
not in parallel
```

---

# File: `init-two-containers.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo-2
  labels:
    app: main-app
spec:

  initContainers:

  - name: check-api
    image: curlimages/curl:latest
    command:
    - sh
    - -c
    - |
      echo "Checking external API..."
      sleep 15
      until curl -s https://kubernetes.io > /dev/null; do
        echo "Waiting for API..."
        sleep 5
      done
      echo "External API reachable"

  - name: check-service
    image: curlimages/curl:latest
    command:
    - sh
    - -c
    - |
      echo "Checking service DNS..."
      until nslookup main-app-svc.default.svc.cluster.local; do
        echo "Waiting for service..."
        sleep 5
      done
      echo "Service reachable"

  containers:
  - name: main-app
    image: nginx
```

---

# Apply Service First

File: `svc.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: main-app-svc
spec:
  selector:
    app: main-app
  ports:
  - port: 80
    targetPort: 80
```

Apply:

```bash
kubectl apply -f svc.yaml
kubectl apply -f init-two-containers.yaml
```

---

# Observe Execution Order

Watch:

```bash
kubectl get pods -w
```

You will see something like:

```
Init:0/2
Init:1/2
Running
```

Meaning:

```
Init 1 finished
Init 2 started
```

---

# Check Logs Individually

First init:

```
kubectl logs init-demo-2 -c check-api
```

Second init:

```
kubectl logs init-demo-2 -c check-service
```

---

# Important Learning

Init containers behave like:

```
step1 → step2 → step3 → main container
```

They are **sequential**.

---

# 4️⃣ Lab 3 — Sidecar Container

### Goal

Observe **two containers running simultaneously in the same pod**.

---

# File: `sidecar-demo.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-logging-demo
spec:

  containers:

  - name: main-app
    image: nginx
    ports:
    - containerPort: 80

  - name: health-logger
    image: curlimages/curl
    command:
    - sh
    - -c
    - |
      while true; do
        curl -s http://localhost:80 > /dev/null && \
        echo "Main app healthy" || echo "Main app unhealthy"
        sleep 5
      done
```

---

# Apply

```bash
kubectl apply -f sidecar-demo.yaml
```

---

# Verify Containers

```bash
kubectl get pod sidecar-logging-demo -o jsonpath='{.spec.containers[*].name}'
```

Output:

```
main-app health-logger
```

---

# Check Logs

```bash
kubectl logs sidecar-logging-demo -c health-logger
```

Expected:

```
Main app healthy
Main app healthy
Main app healthy
```

---

# Why This Works

Both containers share:

```
same network namespace
```

So:

```
localhost:80
```

works.

---

# Verify Network Namespace

Enter sidecar container.

```bash
kubectl exec -it sidecar-logging-demo -c health-logger -- sh
```

Run:

```
curl localhost
```

You will see:

```
nginx HTML page
```

Proof:

```
sidecar -> main container communication works
```

---

# 5️⃣ Failure Simulation (Very Important)

Kill main container.

```bash
kubectl exec -it sidecar-logging-demo -c main-app -- sh
```

Inside container:

```
kill 1
```

---

# Observe

Check logs again.

```
kubectl logs sidecar-logging-demo -c health-logger
```

Now:

```
Main app unhealthy
Main app unhealthy
```

---

# Why?

Sidecar keeps running even if main container dies.

---

# Important Exam Insight

In **Sidecar Pattern**:

```
main container failure
≠
sidecar container stop
```

Containers are **independent**.

---

# Architecture Visualization

```
        Pod Network
        (same IP)

     ┌─────────────────┐
     │      POD        │
     │                 │
     │  main-app       │
     │  nginx:80       │
     │        ▲        │
     │        │        │
     │ localhost:80    │
     │        │        │
     │  sidecar        │
     │ health logger   │
     │                 │
     └─────────────────┘
```

---

# CKA Exam Memory Trick

Remember:

```
Init → setup phase
Sidecar → helper runtime
Ambassador → proxy to external world
Adapter → data translator
```

---

# Ultra-Simple Memory Model

```
Init = before start
Sidecar = helper
Ambassador = proxy
Adapter = translator
```

---

# Real Production Examples

| Pattern    | Example                      |
| ---------- | ---------------------------- |
| Sidecar    | Fluentd logging              |
| Sidecar    | Envoy proxy                  |
| Init       | DB migrations                |
| Init       | Secret fetch                 |
| Ambassador | API gateway                  |
| Adapter    | Prometheus metrics formatter |

---

# Advanced Practice (Highly Recommended)

Try modifying:

### Experiment 1

Add shared volume.

```
emptyDir
```

Main container writes logs → sidecar reads them.

---

### Experiment 2

Break init container.

Change:

```
curl kubernetes.io
```

to

```
curl badsite
```

Observe:

```
pod stuck in Init
```

---

# CKA Exam Trap

Remember:

```
initContainers:
```

is **separate from**

```
containers:
```

Many students accidentally put init containers inside containers list.

---

# Final Mental Model

```
Pod
 ├── initContainers (run once)
 └── containers (run forever)
        ├── main container
        ├── sidecar
        ├── ambassador
        └── adapter
```

---

# Common patterns: 

# 1️⃣ Sidecar with Shared Volume (VERY COMMON)

Right now your sidecar **checks health via network**.

But the **most common sidecar pattern in production is log shipping**.

Example:

```
main container → writes logs
sidecar → reads logs → sends to logging system
```

You should practice **one example like this**.

Example YAML:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-log-demo
spec:
  volumes:
  - name: shared-logs
    emptyDir: {}

  containers:

  - name: main-app
    image: busybox
    command: ["sh", "-c", "while true; do echo 'app log entry' >> /var/log/app.log; sleep 5; done"]
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log

  - name: log-reader
    image: busybox
    command: ["sh", "-c", "tail -f /var/log/app.log"]
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log
```

Observe:

```
kubectl logs sidecar-log-demo -c log-reader
```

You’ll see:

```
app log entry
app log entry
```

Now you understand **volume sharing in multi-container pods**.

---

# 2️⃣ Init Container Preparing Data

Another **classic pattern**:

```
init container → prepares config/data
main container → uses it
```

Example:

```
init → download config
main → run app
```

Example YAML:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-volume-demo
spec:
  volumes:
  - name: shared-data
    emptyDir: {}

  initContainers:
  - name: init-download
    image: busybox
    command: ["sh","-c","echo 'config file' > /data/config.txt"]
    volumeMounts:
    - name: shared-data
      mountPath: /data

  containers:
  - name: main-app
    image: busybox
    command: ["sh","-c","cat /data/config.txt && sleep 3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /data
```

Check:

```
kubectl logs init-volume-demo
```

You’ll see:

```
config file
```

Now you understand **init container preparing environment**.

---

# 3️⃣ Know How to Debug Multi-Container Pods

This is **important for CKA tasks**.

Useful commands:

### List containers

```
kubectl get pod POD -o jsonpath='{.spec.containers[*].name}'
```

---

### Logs from specific container

```
kubectl logs POD -c CONTAINER
```

Example:

```
kubectl logs sidecar-logging-demo -c health-logger
```

---

### Exec into specific container

```
kubectl exec -it POD -c CONTAINER -- sh
```

Example:

```
kubectl exec -it sidecar-logging-demo -c main-app -- sh
```

---

### Check init container status

```
kubectl describe pod POD
```

Look for:

```
Init Containers:
```

---

# 🧠 The Only Things CKA Actually Tests

Memorize this mental model:

```
initContainers:
   run first
   run sequentially
   must succeed

containers:
   run together
   share network
   share volumes
```

---

# 🧠 Ultra Important CKA Memory Rule

```
Init → setup
Sidecar → helper
Ambassador → proxy
Adapter → data transformer
```
---
