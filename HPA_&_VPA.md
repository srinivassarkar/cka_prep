# PART 1 — HORIZONTAL POD AUTOSCALER (HPA)

---

# 🧠 First Principles

HPA changes:

```
replicas count
```

It does NOT change:

* CPU limits
* Memory limits
* Node size

It scales **number of pods**.

---

# 🔥 LAB 1 — Setup HPA Environment

### Step 1 — Ensure metrics-server works

```bash
kubectl top nodes
```

If fails → fix metrics-server.

HPA depends on:

```
metrics.k8s.io
```

---

### Step 2 — Deploy nginx-hpa.yaml

```bash
kubectl apply -f nginx-hpa.yaml
```

Check:

```bash
kubectl get deploy
```

Should show:

```
1 replica
```

---

# 🔥 LAB 2 — Create HPA

Create:

```bash
kubectl autoscale deployment nginx-deploy \
  --cpu-percent=50 \
  --min=1 \
  --max=5
```

OR YAML way:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deploy
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

Apply.

Check:

```bash
kubectl get hpa
```

---

# 🧠 What 50% Means

It means:

```
Current CPU usage / requested CPU
```

Important:

HPA uses:

```
CPU usage ÷ CPU request
```

NOT CPU limit.

---

# 🔥 VERY IMPORTANT

If requests.cpu not defined:

HPA will NOT work.

It calculates percentage based on requests.

---

# 🔥 LAB 3 — Generate Load

Exec into pod:

```bash
kubectl exec -it <pod-name> -- sh
```

Install curl or use busybox pod to generate load:

```bash
while true; do wget -q -O- http://nginx-svc; done
```

Watch:

```bash
kubectl get hpa -w
```

You’ll see:

```
Replicas increase
```

Then:

```bash
kubectl get pods
```

Scaling out happens.

---

# 🧠 Internal Working of HPA

Every 15 seconds:

1. HPA controller checks metrics API
2. Calculates average CPU utilization
3. Computes desired replicas:

Formula (important):

```
desiredReplicas = currentReplicas × (currentMetric / targetMetric)
```

Example:

* currentReplicas = 2
* current utilization = 100%
* target = 50%

So:

```
2 × (100/50) = 4 replicas
```

---

# 🔥 Interview Gold

If asked:

> Why did HPA not scale?

You check:

* Is metrics-server running?
* Are CPU requests defined?
* Is load sustained long enough?
* Is maxReplicas reached?

---

# 🔥 HPA Limitations

* Works best for stateless apps
* Does not scale based on memory reliably
* CPU scaling reactive, not predictive
* Scaling down has cooldown delay

---

# 🔥 Production Insight

In real clusters:

* HPA + Cluster Autoscaler
* If pods scale but nodes full → pending pods
* Cluster Autoscaler adds nodes

---

# 🔥 PART 2 — CLUSTER AUTOSCALER (Conceptual)

HPA scales pods.
Cluster Autoscaler scales nodes.

Flow:

1. HPA increases replicas
2. Some pods become Pending
3. Cluster Autoscaler detects unschedulable pods
4. Adds new node
5. Pods get scheduled

That’s real elasticity.

---

# 🔥 PART 3 — VERTICAL POD AUTOSCALER (VPA)

Now your YAML.

---

# 🧠 First Principles

VPA changes:

```
CPU & memory requests
```

NOT replicas.

It can restart pods.

---

# 🔥 LAB 4 — Deploy VPA Setup

Apply:

```bash
kubectl apply -f nginx-vpa.yaml
kubectl apply -f vpa.yaml
```

Check:

```bash
kubectl get vpa
```

Wait some time.

Then:

```bash
kubectl describe vpa nginx-vpa
```

You’ll see:

```
Recommendation:
  Target:
  Lower Bound:
  Upper Bound:
```

---

# 🧠 What Those Fields Mean

Lower Bound → safe minimum
Target → ideal request
Upper Bound → safe max

Uncapped Target → raw calculation

---

# 🔥 updateMode Explained

```
Off     → recommend only
Initial → set only during pod creation
Auto    → evict + recreate pod with new resources
```

Auto = disruptive (pod restart)

---

# 🔥 IMPORTANT: VPA + HPA Conflict

They conflict if both control CPU.

Why?

HPA uses CPU request to calculate %.
VPA changes CPU request.

This causes unstable scaling.

Production rule:

* Use HPA for CPU scaling
* Use VPA for memory tuning
* Or use VPA in "Off" mode for recommendations

---

# 🔥 LAB 5 — Observe Evictions

```bash
kubectl get events
```

You’ll see:

```
VPA evicting pod
```

Pod gets recreated.

---

# 🧠 Why Restart Required?

Because:

Resource requests are immutable.
Pod spec change = recreate.

---

# 🔥 VPA Components (Deep Understanding)

VPA consists of:

1. Recommender
2. Updater
3. Admission Controller

Recommender → calculates
Updater → evicts pods
Admission → injects new requests

---

# 🔥 HPA vs VPA Quick Brain Table

| Feature              | HPA            | VPA                               |
| -------------------- | -------------- | --------------------------------- |
| Scales               | Replicas       | CPU/Memory                        |
| Restarts pod?        | No             | Yes                               |
| Uses metrics-server? | Yes            | No (uses custom metrics pipeline) |
| Good for             | Stateless apps | Right-sizing                      |

---

# 🔥 Scaling Strategy in Production

Real-world pattern:

Frontend:

* HPA
* CPU-based scaling

Backend:

* HPA

Databases:

* Vertical scaling (manual)
* Or managed service

Cluster:

* Cluster Autoscaler

---

# 💣 Interview Traps

1. HPA not working → missing CPU requests
2. HPA scaling not happening → metrics-server missing
3. VPA causing restarts → updateMode Auto
4. VPA + HPA CPU conflict
5. Node not scaling → Cluster Autoscaler missing

---

# 🔥 Elite-Level Understanding

Elastic system flow:

Load ↑
→ CPU usage ↑
→ HPA scales pods
→ Nodes full
→ Cluster Autoscaler scales nodes
→ Load distributed

Load ↓
→ HPA scales down
→ Idle nodes removed

That’s cloud-native elasticity.

---
