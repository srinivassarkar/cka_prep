# Kubernetes Health Probes — Complete Revision Guide

Think of probes as **Kubernetes asking your container questions**.

```
Are you alive?  → Liveness Probe
Are you ready?  → Readiness Probe
Did you start properly? → Startup Probe
```

All probes are executed by the **Kubelet on the node**.

---

# 1️⃣ Readiness Probe (RP)

### Purpose

Checks whether the application is **ready to receive traffic**.

If it fails:

```
Pod stays running
BUT
removed from Service endpoints
```

So **traffic stops going to the pod**.

Container **is NOT restarted**.

---

### Real Example

A web app that needs:

```
database connection
cache connection
config loading
```

During that time it is **not ready**.

Readiness probe prevents traffic until it is ready.

---

### Example

```yaml
readinessProbe:
  httpGet:
    path: /readyz
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

Flow:

```
Container starts
↓
Wait 5 seconds
↓
Kubelet probes /readyz
↓
200–399 → Ready
other → NotReady
```

---

### Important Behavior

If readiness probe fails:

```
Pod status → Running
Pod condition → NotReady
Service stops sending traffic
```

---

# 2️⃣ Liveness Probe (LP)

### Purpose

Detect **dead or stuck applications**.

Example problems:

```
deadlock
infinite loop
memory leak
app freeze
```

If the probe fails:

```
Kubelet restarts the container
```

---

### Example

```yaml
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 3
  periodSeconds: 5
```

Meaning:

```
run: cat /tmp/healthy
```

If file missing → command fails → restart container.

---

### Behavior

```
FailureThreshold reached
↓
Kubelet kills container
↓
Container restarted
```

---

# 3️⃣ Startup Probe (SP)

Used for **slow-starting applications**.

Problem:

```
Liveness probe may kill container too early
```

Example:

```
Java apps
Legacy monoliths
Large initialization apps
```

Startup probe **delays liveness + readiness**.

---

### Example

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

Meaning:

```
tries 30 times
every 10 seconds
```

Total startup time allowed:

```
30 × 10 = 300 seconds
```

After success:

```
Startup probe stops
Readiness + Liveness start
```

---

# 4️⃣ Probe Mechanisms

Kubernetes supports **3 probe types**.

---

## HTTP GET Probe

Most common.

```yaml
httpGet:
  path: /healthz
  port: 8080
```

Success:

```
HTTP 200–399
```

Failure:

```
400+
timeout
```

---

## TCP Probe

Checks if port is open.

```yaml
tcpSocket:
  port: 3306
```

Used for:

```
databases
tcp services
```

---

## Exec Probe

Runs command inside container.

```yaml
exec:
  command:
  - cat
  - /tmp/healthy
```

Success:

```
exit code 0
```

Failure:

```
non-zero exit
```

---

# 5️⃣ Probe Timing Parameters

These control probe behavior.

| Parameter           | Meaning                 |
| ------------------- | ----------------------- |
| initialDelaySeconds | wait before first probe |
| periodSeconds       | probe frequency         |
| timeoutSeconds      | probe response timeout  |
| successThreshold    | successes required      |
| failureThreshold    | failures before action  |

---

### Example

```yaml
initialDelaySeconds: 5
periodSeconds: 10
failureThreshold: 3
```

Meaning:

```
wait 5 sec
probe every 10 sec
after 3 failures → action
```

---

# 6️⃣ Behavior in Multi-Container Pods

Important rule:

```
Pod status = worst container state
```

Example:

```
container1 → Running
container2 → CrashLoopBackOff
```

Pod status:

```
CrashLoopBackOff
```

---

### Probe behavior per container

| Probe     | Behavior                     |
| --------- | ---------------------------- |
| Startup   | container-specific           |
| Readiness | affects entire pod traffic   |
| Liveness  | restarts only that container |

---

### Example scenario

Pod with 2 containers:

```
main-app
sidecar
```

If readiness fails for **sidecar**:

```
pod becomes NotReady
```

Service stops traffic.

---

# 7️⃣ Why Use Both RP and LP

If only **Liveness** is used:

```
temporary issue
→ container restarts unnecessarily
```

If only **Readiness** is used:

```
dead container
→ never restarted
```

Best practice:

```
Startup Probe (optional)
Readiness Probe
Liveness Probe
```

---

# 8️⃣ Probe Comparison (Memory Shortcut)

| Probe     | Question               | Action              |
| --------- | ---------------------- | ------------------- |
| Startup   | Did the app start?     | restart             |
| Readiness | Can you serve traffic? | remove from service |
| Liveness  | Are you alive?         | restart             |

---

### Quick mental model

```
Startup → initialization phase

Readiness → traffic control

Liveness → crash recovery
```

---

# 9️⃣ What You Should Observe in Killercoda

When running the labs, watch these commands.

### Watch pod status

```
kubectl get pods -w
```

---

### Check readiness column

```
kubectl get pods
```

Example:

```
NAME                   READY   STATUS
readiness-probe-demo   0/1     Running
```

Means:

```
pod running
but NOT ready
```

---

### Check restart count

```
kubectl get pod liveness-probe-demo
```

Example:

```
RESTARTS: 4
```

Meaning liveness probe triggered restart.

---

### Inspect events

```
kubectl describe pod POD_NAME
```

Look for:

```
Liveness probe failed
Readiness probe failed
```

---

# 10️⃣ What Each Demo Teaches

### Readiness Demo

You delete:

```
/tmp/ready
```

Result:

```
pod → NotReady
traffic stops
```

Container still running.

---

### Liveness Demo

After 10 seconds:

```
/healthz returns 500
```

Result:

```
container restarted
```

---

### Startup Demo

Startup probe waits until:

```
port 9444 open
```

Then:

```
readiness + liveness start
```

---

# What CKA Usually Tests

Very common tasks:

```
Add liveness probe
Add readiness probe
Fix broken probe
Identify failing probe
```

Typical exam YAML edit:

```
kubectl edit pod mypod
```

Add:

```
livenessProbe
readinessProbe
```

---
