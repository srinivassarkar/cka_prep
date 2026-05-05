# Lab 13 — HPA (Horizontal Pod Autoscaler)

## What This Lab Is About

HPA automatically scales the number of pod replicas based on observed metrics — CPU, memory, or custom metrics. In prod, HPA is what stands between your app and a traffic spike that would otherwise take it down. Get it right and your app scales gracefully under load and scales back down to save cost. Get it wrong and you get thrashing (constant scale up/down), pods that never scale because metrics-server is missing, or an HPA that fights your manual scaling.

This lab covers the 5 most critical HPA failure modes in real clusters. An HPA stuck at `<unknown>` because metrics-server isn't installed. An HPA that doesn't scale because resource requests are missing on the pods. Thrashing caused by aggressive scale-down settings. The HPA and manual scaling conflict from Lab 02 — revisited in depth. And the correct production HPA configuration with stabilization windows and proper thresholds.

> HPA is not a fire-and-forget feature. It is a dynamic control loop that reacts to your cluster's real-time state. Misconfigure it and it either never fires or fires constantly — both are production problems.

---

## Environment

- **Cluster:** KIND (Kubernetes IN Docker)
- **K8s Version:** v1.27.3
- **Node:** Single node (`kind-control-plane`)
- **Namespace for this lab:** `lab13`

```bash
kubectl create namespace lab13
kubectl config set-context --current --namespace=lab13
```

### Install Metrics-Server (Required for HPA)

HPA requires metrics-server to read CPU and memory from pods. Without it, all metrics show as `<unknown>`.

```bash
# Install metrics-server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Patch for KIND — metrics-server needs insecure TLS in KIND
kubectl patch deployment metrics-server \
  -n kube-system \
  --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

# Wait for metrics-server to be ready
kubectl rollout status deployment metrics-server -n kube-system --timeout=90s

# Verify metrics are flowing
kubectl top nodes
kubectl top pods -A
# If these return data: metrics-server is working
```

---

## The 5 Scenarios

| # | Scenario | What You'll Learn |
|---|---|---|
| 01 | HPA stuck at `<unknown>` | Missing metrics-server, missing resource requests |
| 02 | HPA never scales — requests not set | Why requests are mandatory for HPA to work |
| 03 | HPA thrashing — unstable replica count | Scale-down cooldown, stabilization windows |
| 04 | HPA vs manual scaling conflict | Who owns the replica count, how to resolve |
| 05 | Production HPA pattern | Correct thresholds, stabilization, min/max replicas |

---

## Scenario 01 — HPA Stuck at `<unknown>`

### What You'll Break

An HPA where the target metric shows `<unknown>`. The HPA was created successfully, the deployment is running, but the autoscaler has no data to act on — so it never scales. This happens when metrics-server is missing, unhealthy, or when the pods have no resource requests defined.

### Apply the Broken State

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: unknown-metrics-app
  namespace: lab13
spec:
  replicas: 1
  selector:
    matchLabels:
      app: unknown-metrics-app
  template:
    metadata:
      labels:
        app: unknown-metrics-app
    spec:
      containers:
      - name: app
        image: nginx:1.25
        # NO resources defined — HPA cannot calculate utilization %
        # Utilization % = actual usage / request
        # If request is 0: division impossible = <unknown>
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: unknown-hpa
  namespace: lab13
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: unknown-metrics-app
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
EOF
```

### Symptoms You Will Observe

```bash
kubectl get hpa unknown-hpa -n lab13
# NAME           REFERENCE                        TARGETS         MINPODS   MAXPODS   REPLICAS
# unknown-hpa    Deployment/unknown-metrics-app   <unknown>/50%   1         5         1
# TARGETS shows <unknown>/50% — HPA has no metric data to act on
```

### Investigate

```bash
# Step 1 — Check HPA status in detail
kubectl describe hpa unknown-hpa -n lab13

# Look for these sections:
# Conditions:
#   AbleToScale     True    ReadyForNewScale
#   ScalingActive   False   FailedGetResourceMetric
#     unable to get metrics for resource cpu: unable to fetch metrics from
#     resource metrics API: the server could not find the requested resource

# Step 2 — Is metrics-server running?
kubectl get pods -n kube-system | grep metrics-server
# If not found: metrics-server not installed
# If CrashLoopBackOff: metrics-server has an issue

# Step 3 — Can we get metrics at all?
kubectl top pods -n lab13
# error: Metrics API not available
# OR:
# NAME                              CPU(cores)   MEMORY(bytes)
# unknown-metrics-app-xxx           0m           0Mi
# If top works but HPA shows unknown: resource requests are missing

# Step 4 — Check if pod has resource requests
kubectl get pod -n lab13 -l app=unknown-metrics-app \
  -o jsonpath='{.items[0].spec.containers[0].resources}'
# {} ← empty — no resources defined
# This is why HPA shows <unknown>
# HPA calculates: utilization = actual_cpu / requested_cpu * 100
# If requested_cpu = 0: this calculation is undefined = <unknown>

# Step 5 — Check the metrics API directly
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/namespaces/lab13/pods" | jq '.'
# If this fails: metrics-server issue
# If this returns data but HPA is <unknown>: resource requests missing
```

### Fix

```bash
kubectl delete hpa unknown-hpa -n lab13
kubectl delete deployment unknown-metrics-app -n lab13

# Redeploy with resource requests defined
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-app
  namespace: lab13
spec:
  replicas: 1
  selector:
    matchLabels:
      app: metrics-app
  template:
    metadata:
      labels:
        app: metrics-app
    spec:
      containers:
      - name: app
        image: nginx:1.25
        resources:
          requests:
            cpu: "100m"         # Required for HPA CPU utilization
            memory: "64Mi"      # Required for HPA memory utilization
          limits:
            cpu: "200m"
            memory: "128Mi"
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: metrics-hpa
  namespace: lab13
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: metrics-app
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
EOF

# Wait for metrics to populate (can take 60-90 seconds)
sleep 90

kubectl get hpa metrics-hpa -n lab13
# NAME          REFERENCE              TARGETS   MINPODS   MAXPODS   REPLICAS
# metrics-hpa   Deployment/metrics-app  2%/50%    1         5         1
# Shows real percentage now — HPA is active and watching
```

### The `<unknown>` Decision Tree

```
HPA shows <unknown>?
│
├── kubectl top pods -n <namespace>
│   ├── Error: Metrics API not available
│   │   └── metrics-server not installed or not working
│   │       → kubectl get pods -n kube-system | grep metrics
│   │       → kubectl logs -n kube-system deployment/metrics-server
│   │
│   └── Returns data (metrics-server works)
│       └── Resource requests missing on pods
│           → kubectl get pod -o jsonpath='{..resources}'
│           → Add requests.cpu and requests.memory to pod spec
│
└── kubectl describe hpa
    └── Conditions section → exact error message
```

### Prod Wisdom

**`<unknown>` in HPA targets always has one of two causes** — metrics-server is broken, or resource requests are missing. These are the only two. Check `kubectl top pods` first — if it works, requests are the problem. If it fails, metrics-server is the problem. Never deploy an HPA without first confirming `kubectl top pods` returns real data and that all pods in the target deployment have explicit `requests.cpu` set.

---

## Scenario 02 — HPA Never Scales — Requests Not Set

### What You'll Break

A subtler version of Scenario 01. Metrics-server is working, pods show CPU usage in `kubectl top`, but the HPA still doesn't scale. The reason: the pods have limits defined but no requests. HPA calculates utilization as `actual / requested`. Without a request, the calculation fails silently — or uses 0 as the denominator, which breaks the percentage calculation entirely.

### Apply the Broken State

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: no-request-app
  namespace: lab13
spec:
  replicas: 1
  selector:
    matchLabels:
      app: no-request-app
  template:
    metadata:
      labels:
        app: no-request-app
    spec:
      containers:
      - name: app
        image: polinux/stress:latest
        command: ["stress"]
        args: ["--cpu", "1", "--timeout", "600"]
        resources:
          # Limits defined but NO requests
          limits:
            cpu: "500m"
            memory: "128Mi"
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: no-request-hpa
  namespace: lab13
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: no-request-app
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 30
EOF
```

```bash
# Wait for pod to start
kubectl rollout status deployment/no-request-app -n lab13

# Pod is using CPU
kubectl top pods -n lab13 -l app=no-request-app
# NAME                          CPU(cores)   MEMORY(bytes)
# no-request-app-xxx-yyy        490m         1Mi
# Lots of CPU — should definitely be scaling

# But HPA shows unknown or wrong value
kubectl get hpa no-request-hpa -n lab13
# TARGETS: <unknown>/30%  OR  0%/30%  — never scales despite high CPU
```

### Investigate

```bash
# Step 1 — Check HPA conditions
kubectl describe hpa no-request-hpa -n lab13 | grep -A 20 "Conditions:"
# ScalingActive: False
# FailedGetResourceMetric: missing request for cpu

# Step 2 — Inspect pod resources
kubectl get pod -n lab13 -l app=no-request-app \
  -o jsonpath='{.items[0].spec.containers[0].resources}' | jq '.'
# {
#   "limits": {
#     "cpu": "500m",
#     "memory": "128Mi"
#   }
# }
# No "requests" field — that's the problem

# Step 3 — Understand the math
# HPA formula: utilization % = (sum of pod CPU usage) / (sum of pod CPU requests) * 100
# If requests = 0 (not set): sum = 0, formula breaks
# K8s sets implicit request = limit when only limit is set in some versions
# But HPA still can't reliably calculate — behaviour is undefined
```

### Fix

```bash
kubectl delete hpa no-request-hpa -n lab13
kubectl delete deployment no-request-app -n lab13

cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: no-request-app
  namespace: lab13
spec:
  replicas: 1
  selector:
    matchLabels:
      app: no-request-app
  template:
    metadata:
      labels:
        app: no-request-app
    spec:
      containers:
      - name: app
        image: polinux/stress:latest
        command: ["stress"]
        args: ["--cpu", "1", "--timeout", "600"]
        resources:
          requests:
            cpu: "100m"         # Added: HPA uses this as the denominator
            memory: "64Mi"
          limits:
            cpu: "500m"
            memory: "128Mi"
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: no-request-hpa
  namespace: lab13
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: no-request-app
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 30
EOF

kubectl rollout status deployment/no-request-app -n lab13

# Wait 60-90 seconds for metrics to populate and HPA to react
sleep 90

kubectl get hpa no-request-hpa -n lab13
# TARGETS: 490%/30%   REPLICAS: 5
# Actual 490m / request 100m * 100 = 490% utilization
# Way above 30% threshold — HPA scales to maxReplicas immediately

kubectl get pods -n lab13 -l app=no-request-app
# 5 pods running — HPA scaled up to maximum
```

### The HPA Math — Know This

```
CPU Utilization % Calculation:
  utilization = (sum of all pod actual cpu) / (sum of all pod cpu requests) × 100

Example:
  3 pods, each using 80m CPU, each requesting 100m CPU
  utilization = (3 × 80m) / (3 × 100m) × 100
             = 240m / 300m × 100
             = 80%

  If target is 50% → scale up
  Desired replicas = ceil(current × (current% / target%))
                   = ceil(3 × (80 / 50))
                   = ceil(4.8)
                   = 5 replicas

Memory Utilization % Calculation:
  Same formula — uses requests.memory as denominator
  Memory-based HPA is less common because memory doesn't
  compress like CPU — OOMKill happens before utilization-based
  scaling has time to help. Use memory HPA carefully.

AverageValue (alternative to Utilization):
  Instead of %, use absolute values per pod
  target:
    type: AverageValue
    averageValue: 500m    # Scale when avg pod CPU > 500m
  Does NOT require resource requests to be set
  Better choice when request tuning is difficult
```

### Prod Wisdom

**HPA and resource requests are inseparable.** You cannot use percentage-based CPU or memory HPA without requests set on every container in the target pods. This is not a recommendation — it is a hard requirement of how the utilization formula works. If setting accurate requests is difficult for your app, use `AverageValue` instead of `Utilization` as the target type — it operates on absolute metric values and doesn't require requests. But the better answer is: profile your app, set accurate requests, and use utilization-based HPA.

---

## Scenario 03 — HPA Thrashing — Unstable Replica Count

### What You'll Break

An HPA that constantly scales up and down — thrashing. Every 15 seconds the HPA recalculates, and if the threshold is too tight or the scale-down is too aggressive, replicas yo-yo constantly. This wastes resources, causes deployment churn, and in the worst case, causes brief unavailability during rapid scale-down.

### Apply the Broken State — Aggressive HPA

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: thrashing-app
  namespace: lab13
spec:
  replicas: 1
  selector:
    matchLabels:
      app: thrashing-app
  template:
    metadata:
      labels:
        app: thrashing-app
    spec:
      containers:
      - name: app
        image: nginx:1.25
        resources:
          requests:
            cpu: "100m"
            memory: "64Mi"
          limits:
            cpu: "200m"
            memory: "128Mi"
---
# Overly aggressive HPA — no stabilization
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: thrashing-hpa
  namespace: lab13
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: thrashing-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 20    # Very low threshold — scales up easily
  # No behavior section = default behavior:
  # Scale up: immediate, no delay
  # Scale down: 5 min stabilization window (default since K8s 1.18)
  # But with threshold at 20%: constant fluctuation between 1 and many replicas
EOF
```

```bash
kubectl rollout status deployment/thrashing-app -n lab13

# Watch HPA decisions over time
kubectl get hpa thrashing-hpa -n lab13 -w
# REPLICAS column will fluctuate as HPA reacts to slight CPU changes
# Even nginx idle uses some CPU — flips between 1 and 2 replicas constantly
```

### Observe Thrashing Events

```bash
# Check HPA events — rapid scale up/down is visible here
kubectl describe hpa thrashing-hpa -n lab13 | grep -A 30 "Events:"
# Normal  SuccessfulRescale  New size: 2; reason: cpu resource utilization ...above target
# Normal  SuccessfulRescale  New size: 1; reason: All metrics below target
# Normal  SuccessfulRescale  New size: 2; reason: cpu resource utilization ...above target
# Constantly flipping — this is thrashing
```

### Fix — Add Stabilization Windows

```bash
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: thrashing-hpa
  namespace: lab13
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: thrashing-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60    # More realistic threshold
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60    # Wait 60s of sustained high load before scaling up
      policies:
      - type: Percent
        value: 100                      # Can double replicas per scaling event
        periodSeconds: 60               # But only once per 60 seconds
    scaleDown:
      stabilizationWindowSeconds: 300   # Wait 5 min of sustained low load before scaling down
      policies:
      - type: Pods
        value: 1                        # Remove at most 1 pod per scaling event
        periodSeconds: 120              # And only every 2 minutes
EOF

# HPA now:
# - Won't scale up until CPU stays above 60% for 60s (not just one spike)
# - Won't scale down until CPU stays below 60% for 5 min
# - Scales down gently: 1 pod every 2 minutes instead of all at once
```

### Understanding the Behavior Block

```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 60
    # HPA looks back 60 seconds and uses the LOWEST replica count
    # during that window as the baseline. Prevents rapid oscillation.
    # Default: 0 (no stabilization for scale-up)

    policies:
    - type: Percent          # Scale by percentage of current replicas
      value: 100             # Max 100% increase per period (double)
      periodSeconds: 60      # Per 60 second window

    - type: Pods             # OR scale by absolute pod count
      value: 4               # Max 4 pods per period
      periodSeconds: 60

    selectPolicy: Max        # Use whichever policy allows MORE scaling
                             # selectPolicy: Min = more conservative
                             # selectPolicy: Disabled = prevent scale-up entirely

  scaleDown:
    stabilizationWindowSeconds: 300
    # HPA looks back 300 seconds and uses the HIGHEST replica count
    # during that window. Prevents scale-down during brief quiet periods.
    # Default: 300 (5 minutes — good default for prod)

    policies:
    - type: Pods
      value: 1               # Remove at most 1 pod at a time
      periodSeconds: 120     # Every 2 minutes

    selectPolicy: Min        # Use whichever policy allows LESS scaling
                             # More conservative = safer for stateful apps
```

### Common Thrashing Causes and Fixes

```
CAUSE: Threshold too low (20% CPU on a mostly-idle app)
FIX:   Raise threshold to match typical loaded utilization (50-70%)

CAUSE: No scale-down stabilization window
FIX:   behavior.scaleDown.stabilizationWindowSeconds: 300

CAUSE: Scale-down removes too many pods at once
FIX:   behavior.scaleDown.policies: [{type: Pods, value: 1}]

CAUSE: Metrics fluctuate because of noisy neighbours
FIX:   Increase periodSeconds in policies to smooth out spikes

CAUSE: initialDelaySeconds on probes causes startup CPU spike
FIX:   behavior.scaleUp.stabilizationWindowSeconds: 120
       (wait for startup transient to pass before acting on the spike)
```

### Prod Wisdom

**The default HPA behavior (no `behavior` block) is appropriate for many apps but not all.** Scale-down has a built-in 5-minute stabilization window in K8s 1.18+ — that's good. But scale-up has zero stabilization by default — a single 15-second spike can trigger scale-up. For apps with bursty but short-lived CPU spikes (garbage collection, cron jobs, cache warming), a 60-120 second scale-up stabilization window prevents unnecessary scaling events. Add the `behavior` block to every production HPA — it is the difference between an autoscaler that helps and one that creates noise.

---

## Scenario 04 — HPA vs Manual Scaling Conflict

### What You'll Break (Revisited in Depth)

When an HPA manages a Deployment, it owns the `replicas` field. Any manual `kubectl scale` command is overridden by the HPA within seconds. This was touched in Lab 02 — here we go deeper into why it happens, how to detect it, and the correct way to intervene when HPA is misbehaving.

### Apply the Setup

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: managed-app
  namespace: lab13
spec:
  replicas: 2
  selector:
    matchLabels:
      app: managed-app
  template:
    metadata:
      labels:
        app: managed-app
    spec:
      containers:
      - name: app
        image: nginx:1.25
        resources:
          requests:
            cpu: "100m"
            memory: "64Mi"
          limits:
            cpu: "200m"
            memory: "128Mi"
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: managed-hpa
  namespace: lab13
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: managed-app
  minReplicas: 2
  maxReplicas: 8
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
EOF

kubectl rollout status deployment/managed-app -n lab13
```

### The Conflict in Action

```bash
# Watch HPA's replica management
kubectl get hpa managed-hpa -n lab13
# REPLICAS: 2 (minReplicas)

# Engineer tries to manually scale to 5
kubectl scale deployment managed-app --replicas=5 -n lab13
kubectl get deployment managed-app -n lab13
# READY: 5/5 — briefly at 5

# HPA reconciliation loop runs every 15 seconds
sleep 20

kubectl get deployment managed-app -n lab13
# READY: 2/2 — HPA brought it back to minReplicas (low CPU = scale down)
# Your manual scale was silently reversed

# Check HPA events to see it overriding you
kubectl describe hpa managed-hpa -n lab13 | grep -A 15 "Events:"
# Normal  SuccessfulRescale  New size: 5; reason: Current number of replicas
#         below Spec.MinReplicas
# Wait — this doesn't make sense
# Actually the HPA saw 5 replicas and since CPU is low, scaled back to min
```

### Why This Happens — The Ownership Model

```bash
# HPA sets the replicas field on the Deployment
# Think of it like kubectl scale --replicas=N running every 15 seconds

# You can see the HPA's last scale time and decision
kubectl get hpa managed-hpa -n lab13 -o json | \
  jq '{
    currentReplicas: .status.currentReplicas,
    desiredReplicas: .status.desiredReplicas,
    lastScaleTime: .status.lastScaleTime,
    conditions: .status.conditions
  }'

# The HPA owns the spec.replicas field
# When HPA is present, the Deployment's spec.replicas is effectively read-only
# Any value you set is overwritten by the HPA controller
```

### The Correct Way to Intervene

```bash
# WRONG: kubectl scale deployment managed-app --replicas=6
# This is overridden by HPA in seconds

# RIGHT — Option 1: Adjust HPA minReplicas (persistent change)
kubectl patch hpa managed-hpa -n lab13 \
  --patch '{"spec":{"minReplicas":4}}'

kubectl get hpa managed-hpa -n lab13
# MINPODS: 4  REPLICAS: 4
# HPA now maintains at least 4 — your floor is respected

# RIGHT — Option 2: Temporarily disable HPA for emergency manual control
kubectl patch hpa managed-hpa -n lab13 \
  --patch '{"spec":{"minReplicas":6,"maxReplicas":6}}'
# min == max: HPA holds at exactly 6 replicas
# Effectively manual scaling while keeping HPA structure intact

# RIGHT — Option 3: Delete HPA entirely for full manual control
kubectl delete hpa managed-hpa -n lab13
kubectl scale deployment managed-app --replicas=3 -n lab13
# Now kubectl scale works — no HPA to override it
# Recreate HPA when ready to resume autoscaling
```

### Detecting HPA Ownership

```bash
# How to know if a Deployment is HPA-managed
# Check if an HPA references the deployment
kubectl get hpa -n lab13 \
  -o jsonpath='{.items[?(@.spec.scaleTargetRef.name=="managed-app")].metadata.name}'
# managed-hpa — confirmed HPA is managing this deployment

# Or check all HPAs in namespace
kubectl get hpa -n lab13
# If REFERENCE column includes your deployment: it's HPA-managed

# Check HPA status on the deployment itself
kubectl get deployment managed-app -n lab13 -o yaml | \
  grep -A 3 "annotations"
# annotations may include HPA-set fields
```

### Prod Wisdom

**When HPA is present, `kubectl scale` is a temporary override that gets reversed.** This is by design — the HPA is the source of truth for replica count. In prod, the correct way to change scaling behaviour is to update the HPA (`minReplicas`, `maxReplicas`, or the target utilization). If you need to emergency-pin to a specific replica count, set `minReplicas == maxReplicas` on the HPA — this gives you manual control without breaking the HPA ownership chain, so you can resume autoscaling by just updating the values back.

---

## Scenario 05 — Production HPA Pattern

### What You'll Build

The complete, production-grade HPA configuration. Realistic thresholds, proper stabilization windows, multi-metric scaling, and the verification workflow to confirm HPA is working correctly.

### The Complete Production HPA Setup

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prod-app
  namespace: lab13
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: prod-app
  template:
    metadata:
      labels:
        app: prod-app
    spec:
      containers:
      - name: app
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"         # Accurate baseline — what app uses at rest
            memory: "64Mi"
          limits:
            cpu: "500m"         # Peak — what app can burst to
            memory: "256Mi"
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 15
          failureThreshold: 3
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: prod-hpa
  namespace: lab13
  annotations:
    description: "HPA for prod-app. Scales on CPU (primary) and memory (secondary)."
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: prod-app

  minReplicas: 2              # Always >= 2 for HA in prod
  maxReplicas: 20             # Cap based on node capacity

  metrics:
  # Primary metric: CPU utilization
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60    # Scale up when avg CPU > 60%
                                   # Scale down when avg CPU < 60%
                                   # Chosen to leave headroom before limits

  # Secondary metric: Memory utilization
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70    # Memory-based scaling as safety net

  behavior:
    scaleUp:
      # Don't react to brief spikes — require sustained load
      stabilizationWindowSeconds: 60
      policies:
      # Allow aggressive scale-up to handle real load spikes
      - type: Percent
        value: 100              # Can double replicas per window
        periodSeconds: 60
      - type: Pods
        value: 4                # Or add 4 pods, whichever is more
        periodSeconds: 60
      selectPolicy: Max         # Use whichever allows MORE scale-up

    scaleDown:
      # Be conservative scaling down — avoid removing pods too fast
      stabilizationWindowSeconds: 300   # 5 min of sustained low load
      policies:
      - type: Pods
        value: 1                # Remove max 1 pod at a time
        periodSeconds: 120      # Every 2 minutes
      - type: Percent
        value: 10               # Or 10% of replicas, whichever is LESS
        periodSeconds: 120
      selectPolicy: Min         # Use whichever allows LESS scale-down (conservative)
EOF
```

### Verify the Production HPA Is Working

```bash
# Step 1 — Confirm HPA sees real metrics (not <unknown>)
kubectl get hpa prod-hpa -n lab13
# TARGETS: 2%/60%, 5%/70%  REPLICAS: 2
# Both metrics showing real values — good

# Step 2 — Confirm HPA conditions are healthy
kubectl describe hpa prod-hpa -n lab13 | grep -A 15 "Conditions:"
# AbleToScale:     True   ReadyForNewScale
# ScalingActive:   True   ValidMetricFound
# ScalingLimited:  False  DesiredWithinRange
# All three conditions green = HPA is healthy and active

# ScalingLimited: True means HPA WANTS to scale but is blocked by min/max bounds
# This is normal and expected when at minReplicas with low load

# Step 3 — Verify min/max bounds are set correctly
kubectl get hpa prod-hpa -n lab13 \
  -o jsonpath='{.spec.minReplicas},{.spec.maxReplicas}'
# 2,20

# Step 4 — Check last scale event
kubectl describe hpa prod-hpa -n lab13 | grep -A 5 "Events:"
# Should show successful scale events or no events (stable)
```

### Simulate Load to Trigger Scaling

```bash
# Generate CPU load to trigger HPA scale-up
# Run a load generator for 3 minutes
kubectl run load-generator \
  --image=busybox:1.35 \
  --rm -it \
  --restart=Never \
  -n lab13 \
  -- sh -c "
    while true; do
      wget -q -O- http://prod-app.lab13.svc.cluster.local > /dev/null 2>&1
    done
  " &

# In another terminal, watch HPA react
kubectl get hpa prod-hpa -n lab13 -w
# TARGETS will climb above 60% as load increases
# REPLICAS will scale up (after 60s stabilization window)

# Clean up load generator
kubectl delete pod load-generator -n lab13 2>/dev/null || true
```

### The Production HPA Checklist

```
Before deploying HPA in prod:
  □ metrics-server is installed and kubectl top pods returns real data
  □ All containers in target deployment have requests.cpu set
  □ All containers have requests.memory if using memory-based HPA
  □ minReplicas >= 2 for any HA service
  □ maxReplicas set based on cluster capacity and cost budget
  □ Target utilization has headroom below limits (60% of limit is good baseline)
  □ behavior block defined with scaleDown stabilizationWindowSeconds >= 300
  □ behavior.scaleDown.policies limits pod removal rate
  □ HPA conditions show AbleToScale: True and ScalingActive: True
  □ Test with synthetic load before prod launch

Common threshold guidelines:
  CPU: 60-70% utilization target — headroom for spikes
  Memory: 70-80% utilization target — K8s can't add memory mid-request like CPU
  Custom metric: depends entirely on the metric semantics

Never:
  □ Set minReplicas: 0 for production services (no pods = no traffic handling)
  □ Use HPA without resource requests on pods
  □ Set target utilization above 80% (no headroom for load spikes)
  □ Use kubectl scale on HPA-managed deployments
  □ Deploy HPA without checking metrics-server health first
```

---

## Key Commands Reference — Lab 13

```bash
# HPA status
kubectl get hpa -n <namespace>
kubectl describe hpa <name> -n <namespace>

# Watch HPA decisions in real time
kubectl get hpa <name> -n <namespace> -w

# HPA conditions (healthy check)
kubectl describe hpa <name> -n <namespace> | grep -A 15 "Conditions:"

# Check what metrics HPA is reading
kubectl get --raw \
  "/apis/metrics.k8s.io/v1beta1/namespaces/<namespace>/pods" | jq '.'

# Metrics server health
kubectl get pods -n kube-system | grep metrics-server
kubectl top pods -n <namespace>
kubectl top nodes

# Adjust HPA bounds (correct way to intervene)
kubectl patch hpa <name> -n <namespace> \
  --patch '{"spec":{"minReplicas":<N>}}'

kubectl patch hpa <name> -n <namespace> \
  --patch '{"spec":{"maxReplicas":<N>}}'

# Emergency pin to exact replica count
kubectl patch hpa <name> -n <namespace> \
  --patch '{"spec":{"minReplicas":<N>,"maxReplicas":<N>}}'

# Scale HPA-managed deployment (wrong way — gets overridden)
# kubectl scale deployment <name> --replicas=N  ← don't do this

# Check if deployment is HPA-managed
kubectl get hpa -n <namespace> -o json | \
  jq '.items[] | select(.spec.scaleTargetRef.name=="<deployment-name>") | .metadata.name'

# HPA events
kubectl describe hpa <name> -n <namespace> | grep -A 20 "Events:"
```

---

## Prod Wisdom — The Senior Engineer Mindset

Three things that define senior-level HPA thinking:

**1. They treat `<unknown>` as a two-cause problem with a clear diagnostic path.** Metrics-server broken or requests missing — those are the only two causes. `kubectl top pods` tells you which one in 5 seconds. They don't guess, they don't restart pods — they check metrics-server health, then check pod resource requests. In that order. Every time.

**2. They always define the `behavior` block.** Default HPA behavior is scale-up instantly, scale-down after 5 minutes. For most prod apps, scale-up should be stabilized (60-120 seconds) to avoid reacting to transient spikes, and scale-down should be rate-limited (1 pod at a time, every 2 minutes) to prevent sudden capacity loss. The `behavior` block is not optional in prod — it is the difference between an autoscaler that is calm and predictable versus one that thrashes.

**3. They never use `kubectl scale` on an HPA-managed deployment.** When HPA is present, updating `minReplicas` or `maxReplicas` on the HPA is the only valid way to change replica count. If you need emergency manual control, pin the HPA with `minReplicas == maxReplicas`. If HPA itself is the problem, delete it, scale manually, fix the HPA config, and re-apply. Treating HPA like it doesn't exist leads to chasing your own tail as the HPA silently reverts every change.

The HPA debug order:
```
kubectl get hpa                      → TARGETS showing real value or <unknown>?
kubectl top pods                     → metrics-server working?
kubectl describe hpa → Conditions    → ScalingActive: True?
kubectl get pod -o jsonpath resources → requests.cpu set?
kubectl describe hpa → Events        → what scaling decisions were made?
```

---

## Cleanup

```bash
kubectl delete namespace lab13
```

---

*Lab 13 complete. Move to Lab 14 — NetworkPolicy when ready.*