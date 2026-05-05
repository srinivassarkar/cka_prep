# Lab 14 — NetworkPolicy

## What This Lab Is About

By default, every pod in a Kubernetes cluster can talk to every other pod — across namespaces, across teams, across environments. NetworkPolicy is how you lock that down. It is the firewall inside your cluster. Without it, a compromised pod in your dev namespace can reach your production database. With it misconfigured, legitimate traffic is silently dropped and your app breaks with no obvious error.

NetworkPolicy is the most skipped topic in K8s learning because it's invisible when it works and invisible when it breaks — you just get connection timeouts. Engineers who don't understand it spend hours debugging what looks like a DNS problem, a service misconfiguration, or an app bug — when it's actually a NetworkPolicy silently dropping packets.

This lab covers the 5 most critical NetworkPolicy scenarios in real clusters. The default deny baseline that blocks all traffic. Ingress rules that allow traffic from specific pods. Egress rules that control what pods can reach. Cross-namespace policies. And the systematic debug workflow for when traffic is silently dropped.

> NetworkPolicy failures look identical to every other networking failure — connection timeout, connection refused. There is no error that says "NetworkPolicy blocked this." The only way to know is to check systematically.

---

## Environment

- **Cluster:** KIND (Kubernetes IN Docker)
- **K8s Version:** v1.27.3
- **Node:** Single node (`kind-control-plane`)
- **Namespace for this lab:** `lab14`

```bash
kubectl create namespace lab14
kubectl config set-context --current --namespace=lab14
```

### Install a CNI That Supports NetworkPolicy

KIND uses `kindnet` by default which does **not** enforce NetworkPolicy. You need to install a CNI that does — Calico is the most common.

```bash
# Check if NetworkPolicy is enforced in your KIND cluster
# Create a test policy and verify it blocks traffic

# Option A — Reinstall KIND with Calico CNI (recommended for full NetworkPolicy support)
# Create a KIND cluster config that disables kindnet:
cat <<EOF > /tmp/kind-calico.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
networking:
  disableDefaultCNI: true    # Disable kindnet
  podSubnet: "192.168.0.0/16"
EOF

# kind delete cluster
# kind create cluster --config=/tmp/kind-calico.yaml

# Then install Calico:
# kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
# kubectl rollout status daemonset/calico-node -n kube-system --timeout=120s

# Option B — Use your existing KIND cluster
# NetworkPolicy YAML will apply but won't be enforced without a supporting CNI
# The scenarios show correct NetworkPolicy syntax and concepts
# Use Killercoda (which has Calico) to test enforcement behaviour

# Verify your CNI supports NetworkPolicy:
kubectl get pods -n kube-system | grep -E "calico|cilium|weave|flannel"
```

**Important:** All NetworkPolicy YAML in this lab is correct and production-ready. If your KIND cluster uses kindnet (default), policies apply but aren't enforced. Use Killercoda or a Calico-enabled cluster to observe actual traffic blocking.

---

## The 5 Scenarios

| # | Scenario | What You'll Learn |
|---|---|---|
| 01 | Default deny — block all traffic | NetworkPolicy baseline, how to apply without locking yourself out |
| 02 | Allow ingress from specific pods | `podSelector`, `namespaceSelector`, port rules |
| 03 | Egress control — restrict outbound | DNS egress, external access, egress deny patterns |
| 04 | Cross-namespace policies | `namespaceSelector`, namespace labels, multi-team isolation |
| 05 | Debug workflow — traffic silently dropped | Systematic elimination when NetworkPolicy may be the cause |

---

## Scenario 01 — Default Deny — Block All Traffic

### What You'll Break (Intentionally)

The first NetworkPolicy you apply to any namespace should be a default deny — block all ingress and egress by default. Then open only what's needed. This is the secure-by-default approach. Without it, all pods communicate freely and your NetworkPolicies are additive (only restricting some traffic while everything else flows).

### Understanding How NetworkPolicy Works

```
Without ANY NetworkPolicy in a namespace:
  → All ingress allowed (any pod can send traffic in)
  → All egress allowed (pods can reach anywhere)
  → Completely open

Once ANY NetworkPolicy selects a pod:
  → Only traffic explicitly allowed by a policy is permitted
  → Everything else is DENIED
  → NetworkPolicies are ADDITIVE — multiple policies combine with OR logic

Key insight:
  An empty podSelector ({}) selects ALL pods in the namespace
  A NetworkPolicy with no ingress rules = deny all ingress for selected pods
  A NetworkPolicy with no egress rules = deny all egress for selected pods
```

### Apply the Default Deny Baseline

```bash
# Deploy test workloads first
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: lab14
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
      tier: web
  template:
    metadata:
      labels:
        app: frontend
        tier: web
    spec:
      containers:
      - name: app
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "64Mi"
            cpu: "100m"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: lab14
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
      tier: api
  template:
    metadata:
      labels:
        app: backend
        tier: api
    spec:
      containers:
      - name: app
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "64Mi"
            cpu: "100m"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: lab14
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
      tier: db
  template:
    metadata:
      labels:
        app: database
        tier: db
    spec:
      containers:
      - name: db
        image: nginx:1.25   # Simulating a database
        ports:
        - containerPort: 5432
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "64Mi"
            cpu: "100m"
---
# Services for each tier
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
  namespace: lab14
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: lab14
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: database-svc
  namespace: lab14
spec:
  selector:
    app: database
  ports:
  - port: 5432
    targetPort: 5432
EOF

kubectl rollout status deployment/frontend deployment/backend deployment/database \
  -n lab14 --timeout=60s
```

### Verify Open Communication Before Policy

```bash
# Before any NetworkPolicy — all pods can reach each other
FRONTEND_POD=$(kubectl get pod -n lab14 -l app=frontend \
  -o jsonpath='{.items[0].metadata.name}')

# Frontend → Backend: should work (no policy yet)
kubectl exec ${FRONTEND_POD} -n lab14 -- \
  wget -q -O- --timeout=3 http://backend-svc.lab14.svc.cluster.local 2>&1 | head -2
# Welcome to nginx — open traffic

# Frontend → Database: should work (no policy yet) — this is the security gap
kubectl exec ${FRONTEND_POD} -n lab14 -- \
  wget -q -O- --timeout=3 http://database-svc.lab14.svc.cluster.local:5432 2>&1
# Connected — frontend can reach database directly — dangerous
```

### Apply Default Deny

```bash
# Default deny ALL ingress and egress for ALL pods in the namespace
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: lab14
spec:
  podSelector: {}             # Selects ALL pods in namespace
  policyTypes:
  - Ingress
  - Egress
  # No ingress or egress rules = deny all
EOF
```

### Observe the Effect

```bash
# Now all traffic is blocked
kubectl exec ${FRONTEND_POD} -n lab14 -- \
  wget -q -O- --timeout=3 http://backend-svc.lab14.svc.cluster.local 2>&1
# wget: download timed out
# Traffic is blocked by NetworkPolicy

# Even DNS is blocked (DNS uses egress on UDP port 53)
kubectl exec ${FRONTEND_POD} -n lab14 -- \
  wget -q -O- --timeout=3 http://backend-svc 2>&1
# wget: bad address 'backend-svc'
# DNS resolution fails — egress to CoreDNS is also blocked

# This is expected — default deny blocks everything including DNS
# Next scenarios will selectively open what's needed
```

### Prod Wisdom

**Apply default deny FIRST, then open what's needed — never the reverse.** If you apply specific allow policies first and default deny second, there's a window where traffic is open. The order matters. In prod, applying default deny to a namespace with running workloads will immediately break communication — plan the rollout of allow policies to happen in the same kubectl apply as the default deny, or use a tool like `kubectl apply -f policies/` on a directory containing both deny and allow policies together.

---

## Scenario 02 — Allow Ingress From Specific Pods

### What You'll Build

The correct ingress allow rules for a three-tier application. Frontend can receive traffic from anywhere (it's the public-facing tier). Backend can only receive traffic from frontend. Database can only receive traffic from backend. This is the minimal-access pattern.

### Apply Selective Ingress Rules

```bash
# Step 1 — Allow DNS egress for all pods (required for service discovery)
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: lab14
spec:
  podSelector: {}             # All pods
  policyTypes:
  - Egress
  egress:
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP            # Some DNS over TCP
EOF

# Step 2 — Frontend: allow ingress from anywhere (public-facing)
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-ingress
  namespace: lab14
spec:
  podSelector:
    matchLabels:
      app: frontend            # Applies to frontend pods only
  policyTypes:
  - Ingress
  ingress:
  - {}                         # Allow ingress from anywhere (no restriction)
EOF

# Step 3 — Backend: only allow ingress from frontend
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-ingress
  namespace: lab14
spec:
  podSelector:
    matchLabels:
      app: backend             # Applies to backend pods only
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend        # Only pods with app=frontend label can connect
    ports:
    - port: 80
      protocol: TCP
EOF

# Step 4 — Database: only allow ingress from backend
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-ingress
  namespace: lab14
spec:
  podSelector:
    matchLabels:
      app: database            # Applies to database pods only
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend         # Only pods with app=backend label can connect
    ports:
    - port: 5432
      protocol: TCP

EOF

# Step 5 — Allow frontend and backend egress to their respective targets
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-egress
  namespace: lab14
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: backend         # Frontend can only egress to backend
    ports:
    - port: 80
      protocol: TCP
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-egress
  namespace: lab14
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database        # Backend can only egress to database
    ports:
    - port: 5432
      protocol: TCP
EOF
```

### Verify the Access Matrix

```bash
FRONTEND_POD=$(kubectl get pod -n lab14 -l app=frontend \
  -o jsonpath='{.items[0].metadata.name}')
BACKEND_POD=$(kubectl get pod -n lab14 -l app=backend \
  -o jsonpath='{.items[0].metadata.name}')
DATABASE_POD=$(kubectl get pod -n lab14 -l app=database \
  -o jsonpath='{.items[0].metadata.name}')

echo "=== Test 1: Frontend → Backend (SHOULD WORK) ==="
kubectl exec ${FRONTEND_POD} -n lab14 -- \
  wget -q -O- --timeout=5 http://backend-svc 2>&1 | head -1
# Welcome to nginx — allowed

echo "=== Test 2: Frontend → Database (SHOULD FAIL) ==="
kubectl exec ${FRONTEND_POD} -n lab14 -- \
  wget -q -O- --timeout=3 http://database-svc:5432 2>&1
# wget: download timed out — blocked by policy

echo "=== Test 3: Backend → Database (SHOULD WORK) ==="
kubectl exec ${BACKEND_POD} -n lab14 -- \
  wget -q -O- --timeout=5 http://database-svc:5432 2>&1 | head -1
# Connected — allowed

echo "=== Test 4: Backend → Frontend (SHOULD FAIL) ==="
kubectl exec ${BACKEND_POD} -n lab14 -- \
  wget -q -O- --timeout=3 http://frontend-svc 2>&1
# wget: download timed out — backend can't initiate to frontend
```

### The podSelector AND vs OR Trap

```yaml
# This is a CRITICAL syntax distinction in NetworkPolicy

# Version A — AND logic (both conditions must match the SAME pod)
ingress:
- from:
  - podSelector:
      matchLabels:
        app: frontend
    namespaceSelector:             # Same list item = AND
      matchLabels:
        env: prod
# Allows: pods with app=frontend label AND in a namespace with env=prod label

# Version B — OR logic (either condition matches)
ingress:
- from:
  - podSelector:
      matchLabels:
        app: frontend              # Separate list item = OR
  - namespaceSelector:
      matchLabels:
        env: prod
# Allows: pods with app=frontend label OR any pod in a namespace with env=prod label

# The difference is a single hyphen (-) in YAML
# - podSelector AND namespaceSelector = same list item (no extra dash)
#   - namespaceSelector = new list item (new dash) = OR
# This is one of the most common NetworkPolicy mistakes
```

### Prod Wisdom

**DNS egress must always be explicitly allowed when using default deny.** This is the most common mistake after applying default deny — everything times out and engineers assume the policy is wrong, when actually it's just DNS being blocked. Always apply a DNS egress allow policy (UDP/TCP port 53 to CoreDNS) alongside default deny. And always verify your access matrix after applying policies — test every allowed path AND every blocked path. A policy that blocks what should be allowed is just as broken as one that allows what should be blocked.

---

## Scenario 03 — Egress Control — Restrict Outbound

### What You'll Build

Egress policies control what pods can reach outbound — external services, other namespaces, the internet. In secure environments, you want pods to be able to call specific external APIs but nothing else. This scenario covers the most important egress patterns.

### Allow Specific External Access

```bash
# Scenario: backend pods should be able to call an external payment API
# but nothing else on the internet

cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-external-egress
  namespace: lab14
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Egress
  egress:
  # Allow DNS (required for name resolution)
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP

  # Allow HTTPS to specific external IP range (payment provider)
  - to:
    - ipBlock:
        cidr: 203.0.113.0/24      # Payment provider IP range
        except:
        - 203.0.113.100/32         # Except this specific blocked IP
    ports:
    - port: 443
      protocol: TCP

  # Allow internal database access
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - port: 5432
      protocol: TCP
EOF
```

### Block All Internet Access Except Specific Ports

```bash
# Scenario: database pods should NEVER reach the internet
# Only allow intra-cluster traffic

cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-strict-egress
  namespace: lab14
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Egress
  egress:
  # Allow DNS only
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
  # Allow nothing else — no internet, no other services
  # Database should only receive connections, not initiate them
EOF
```

### Verify Egress Restrictions

```bash
DATABASE_POD=$(kubectl get pod -n lab14 -l app=database \
  -o jsonpath='{.items[0].metadata.name}')

echo "=== Database egress test: internet (SHOULD FAIL) ==="
kubectl exec ${DATABASE_POD} -n lab14 -- \
  wget -q -O- --timeout=3 http://google.com 2>&1
# wget: download timed out — blocked

echo "=== Database egress test: frontend (SHOULD FAIL) ==="
kubectl exec ${DATABASE_POD} -n lab14 -- \
  wget -q -O- --timeout=3 http://frontend-svc 2>&1
# wget: download timed out — blocked

echo "=== Database DNS still works ==="
kubectl exec ${DATABASE_POD} -n lab14 -- \
  nslookup kubernetes.default 2>&1 | grep Address
# Address: 10.96.0.1 — DNS works
```

### The ipBlock Selector

```yaml
# ipBlock selects traffic to/from specific IP ranges
# Use for: external services, cloud provider APIs, on-prem systems

egress:
- to:
  - ipBlock:
      cidr: 0.0.0.0/0           # All IPs (internet)
      except:
      - 10.0.0.0/8              # Except private RFC1918
      - 172.16.0.0/12           # Except private RFC1918
      - 192.168.0.0/16          # Except private RFC1918
  # This allows internet but blocks intra-cluster (pod CIDR is in private range)
  # Useful for: pods that need internet but shouldn't access internal services

ingress:
- from:
  - ipBlock:
      cidr: 10.0.0.0/8          # Allow from internal network only
  # Useful for: accepting traffic from VPN/on-prem networks
```

### Prod Wisdom

**Egress control is as important as ingress control — and more often ignored.** A compromised pod with unrestricted egress can exfiltrate data, call command-and-control servers, or pivot to internal services. In regulated environments (PCI-DSS, HIPAA, SOC2), egress control is a compliance requirement, not optional. The minimum egress policy for any pod that doesn't need internet access: allow DNS (port 53), allow specific internal services it needs, deny everything else. This is the network equivalent of `automountServiceAccountToken: false` — minimize the blast radius if the pod is compromised.

---

## Scenario 04 — Cross-Namespace Policies

### What You'll Build

Policies that span namespace boundaries. This is how you allow a monitoring system in the `monitoring` namespace to scrape metrics from your app in `lab14`, or allow an ingress controller in `ingress-nginx` to forward traffic to your services.

### Setup — Create Additional Namespace

```bash
kubectl create namespace lab14-monitoring
kubectl label namespace lab14-monitoring purpose=monitoring

# Label the lab14 namespace for policy selection
kubectl label namespace lab14 env=production
```

### Deploy a Monitoring Pod

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: prometheus
  namespace: lab14-monitoring
  labels:
    app: prometheus
spec:
  containers:
  - name: prometheus
    image: busybox:1.35
    command: ["sleep", "3600"]
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
EOF
```

### Apply Cross-Namespace Ingress Policy

```bash
# Allow prometheus in lab14-monitoring to scrape metrics from backend in lab14
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring-ingress
  namespace: lab14             # Policy lives in the TARGET namespace
spec:
  podSelector:
    matchLabels:
      app: backend             # Applies to backend pods in lab14
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:       # From the monitoring namespace
        matchLabels:
          purpose: monitoring
      podSelector:             # AND only from prometheus pods (AND logic — same item)
        matchLabels:
          app: prometheus
    ports:
    - port: 8080               # Metrics port (hypothetical)
      protocol: TCP
    - port: 80                 # For this lab, nginx serves on 80
      protocol: TCP
EOF
```

### Verify Cross-Namespace Access

```bash
# Prometheus (in monitoring namespace) → backend (in lab14)
kubectl exec prometheus -n lab14-monitoring -- \
  wget -q -O- --timeout=5 \
  http://backend-svc.lab14.svc.cluster.local 2>&1 | head -1
# Welcome to nginx — cross-namespace allowed

# Prometheus → database (no policy allows this)
kubectl exec prometheus -n lab14-monitoring -- \
  wget -q -O- --timeout=3 \
  http://database-svc.lab14.svc.cluster.local:5432 2>&1
# wget: download timed out — blocked, no policy allows monitoring → database
```

### Namespace Labels Are Required for namespaceSelector

```bash
# namespaceSelector matches by NAMESPACE LABELS not namespace name
# This is a common confusion

# WRONG assumption: this selects the "production" namespace by name
namespaceSelector:
  matchLabels:
    name: production           # This matches namespaces with label "name=production"
                               # NOT the namespace named "production"

# CORRECT: label the namespace first, then select by that label
kubectl label namespace lab14 env=production

# THEN use:
namespaceSelector:
  matchLabels:
    env: production            # Matches namespace with label env=production

# Kubernetes automatically adds a label to every namespace:
# kubernetes.io/metadata.name: <namespace-name>
# So you CAN select by name using this auto-label:
namespaceSelector:
  matchLabels:
    kubernetes.io/metadata.name: lab14    # Selects namespace named "lab14"

# Check what labels a namespace has
kubectl get namespace lab14 --show-labels
# NAME    STATUS   AGE   LABELS
# lab14   Active   30m   env=production,kubernetes.io/metadata.name=lab14
```

### The Ingress Controller Exception

```bash
# In clusters with an Ingress controller, you need to allow traffic
# from the ingress-nginx namespace to your pods
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-controller
  namespace: lab14
spec:
  podSelector:
    matchLabels:
      app: frontend            # Frontend receives external traffic via Ingress
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx   # Ingress controller namespace
    ports:
    - port: 80
      protocol: TCP
EOF
```

### Prod Wisdom

**Always label your namespaces when using NetworkPolicy — before you apply any policies.** The `namespaceSelector` is label-based, not name-based. If you apply a policy referencing a namespace label that doesn't exist yet, the policy silently matches nothing. In prod clusters, establish namespace labeling conventions on day one — `env`, `team`, `purpose` labels — and apply them consistently. Also always remember: NetworkPolicies live in the target namespace (where traffic is going TO), not the source namespace. A policy in `lab14` controls what comes INTO `lab14`, regardless of where it comes from.

---

## Scenario 05 — Debug Workflow — Traffic Silently Dropped

### What You'll Build

The systematic workflow for when traffic is being dropped and you suspect NetworkPolicy — but you're not sure. NetworkPolicy failures are silent: no log, no error message, just a timeout. This scenario gives you the elimination sequence to confirm NetworkPolicy is the cause and identify which policy is blocking.

### Apply a Broken State

```bash
# Add a policy with a label typo — silently drops legitimate traffic
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: broken-backend-policy
  namespace: lab14
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontendd       # TYPO: "frontendd" instead of "frontend"
                               # No pods match this label — effectively blocks all ingress
    ports:
    - port: 80
      protocol: TCP
EOF
```

```bash
FRONTEND_POD=$(kubectl get pod -n lab14 -l app=frontend \
  -o jsonpath='{.items[0].metadata.name}')

# Frontend → Backend: now broken due to typo
kubectl exec ${FRONTEND_POD} -n lab14 -- \
  wget -q -O- --timeout=3 http://backend-svc 2>&1
# wget: download timed out — silently blocked
```

### The NetworkPolicy Debug Workflow

```bash
# ============================================================
# STEP 1 — Eliminate other causes first
# ============================================================

# Is the backend pod running?
kubectl get pods -n lab14 -l app=backend
# Running — pod is fine

# Does the service have endpoints?
kubectl get endpoints backend-svc -n lab14
# 10.244.x.x:80 — endpoints exist, selector is correct

# Can we reach backend by port-forward (bypasses NetworkPolicy)?
kubectl port-forward svc/backend-svc 9090:80 -n lab14 &
curl -s http://localhost:9090 | head -2
# Welcome to nginx — pod is healthy and responding
kill %1

# Conclusion so far:
# Pod is running ✓
# Service has endpoints ✓
# Pod responds directly ✓
# But traffic from frontend is blocked → NetworkPolicy is the likely cause

# ============================================================
# STEP 2 — List all NetworkPolicies affecting the backend
# ============================================================

kubectl get networkpolicies -n lab14
# NAME                     POD-SELECTOR     AGE
# allow-dns-egress         <none>           10m
# allow-monitoring-ingress app=backend      5m
# backend-ingress          app=backend      10m
# backend-egress           app=backend      10m
# broken-backend-policy    app=backend      2m
# default-deny-all         <none>           10m

# Multiple policies select backend — all combine with OR logic
# One of them may be the problem

# ============================================================
# STEP 3 — Describe each policy that selects backend
# ============================================================

kubectl describe networkpolicy broken-backend-policy -n lab14
# Spec:
#   PodSelector:     app=backend
#   PolicyTypes:     Ingress
#   Ingress:
#     From:
#       PodSelector: app=frontendd    ← TYPO — "frontendd" not "frontend"
#     Ports:
#       80/TCP

kubectl describe networkpolicy backend-ingress -n lab14
# Spec:
#   PodSelector:     app=backend
#   PolicyTypes:     Ingress
#   Ingress:
#     From:
#       PodSelector: app=frontend     ← Correct label

# ============================================================
# STEP 4 — Check if the source pod has the right labels
# ============================================================

kubectl get pod -n lab14 -l app=frontend --show-labels
# NAME                  LABELS
# frontend-xxx-yyy      app=frontend,tier=web,...
# Label is "app=frontend" — matches backend-ingress policy
# Does NOT match "app=frontendd" — broken-backend-policy matches nothing

# ============================================================
# STEP 5 — Understand policy combination
# ============================================================

# Multiple policies on same pod combine with OR logic
# backend-ingress allows: from app=frontend
# broken-backend-policy allows: from app=frontendd (no such pods)

# Since broken-backend-policy overrides nothing (OR logic, not AND)
# backend-ingress SHOULD still allow frontend traffic
# But wait — we deleted the original backend-ingress in this scenario?

# Check what the actual current policies say:
kubectl get networkpolicies -n lab14 -o yaml | \
  grep -A 20 "podSelector:\|ingress:\|from:\|matchLabels:"

# If both policies exist: frontend can access backend (backend-ingress allows it)
# If ONLY broken-backend-policy exists: frontend is blocked (typo = no match)

# ============================================================
# STEP 6 — Test with a pod that matches the broken label
# ============================================================

# Create a pod with the typo label to confirm the policy works for that label
kubectl run test-frontendd \
  --image=busybox:1.35 \
  --rm -it \
  --restart=Never \
  -n lab14 \
  --labels="app=frontendd" \
  -- wget -q -O- --timeout=5 http://backend-svc 2>&1 | head -1
# Welcome to nginx — confirms broken policy WOULD work if label matched

# ============================================================
# STEP 7 — Fix: correct the typo in the policy
# ============================================================

cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: broken-backend-policy
  namespace: lab14
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend        # Fixed: correct label
    ports:
    - port: 80
      protocol: TCP
EOF

# Verify fix
kubectl exec ${FRONTEND_POD} -n lab14 -- \
  wget -q -O- --timeout=5 http://backend-svc 2>&1 | head -1
# Welcome to nginx — working again
```

### The Complete NetworkPolicy Debug Decision Tree

```
Traffic between pod A and pod B is blocked (timeout)?
│
├── STEP 1 — Eliminate non-NetworkPolicy causes
│   ├── kubectl get pod (pod running?)
│   ├── kubectl get endpoints (service has endpoints?)
│   └── kubectl port-forward pod → test (pod responds directly?)
│       ├── Port-forward FAILS → app/probe issue, not NetworkPolicy
│       └── Port-forward WORKS → NetworkPolicy or Service issue
│
├── STEP 2 — Are there NetworkPolicies in source/target namespace?
│   ├── kubectl get networkpolicies -n <source-ns>
│   └── kubectl get networkpolicies -n <target-ns>
│       └── No policies → NetworkPolicy not the cause
│           → Check Service selector, DNS, pod labels
│
├── STEP 3 — Which policies select the TARGET pod (ingress)?
│   kubectl get networkpolicies -n <target-ns> | grep <target-label>
│   kubectl describe each matching policy
│   └── Do any policies allow ingress from source pod?
│       ├── Yes → Check egress policies on SOURCE pod
│       └── No → Add/fix ingress policy in target namespace
│
├── STEP 4 — Which policies select the SOURCE pod (egress)?
│   kubectl get networkpolicies -n <source-ns>
│   kubectl describe each matching policy
│   └── Do any policies allow egress to target pod?
│       ├── Yes → Policy looks correct — check label typos
│       └── No → Add/fix egress policy in source namespace
│
├── STEP 5 — Verify labels match exactly
│   kubectl get pod -n <source-ns> --show-labels
│   kubectl get pod -n <target-ns> --show-labels
│   kubectl get namespace --show-labels
│   └── Compare against podSelector/namespaceSelector in policies
│       └── Any typo or missing label → fix the label or policy
│
└── STEP 6 — Test with a labeled debug pod
    kubectl run test --image=busybox --labels="app=source-label" ...
    → Confirms whether the policy works when labels match
```

### Quick NetworkPolicy Audit Commands

```bash
# See all NetworkPolicies in namespace
kubectl get networkpolicies -n <namespace>

# Describe all NetworkPolicies (full spec)
kubectl describe networkpolicies -n <namespace>

# Get NetworkPolicies as YAML
kubectl get networkpolicies -n <namespace> -o yaml

# Find policies that select a specific pod
kubectl get networkpolicies -n <namespace> -o json | \
  jq '.items[] | select(.spec.podSelector.matchLabels.app=="backend") | .metadata.name'

# Check namespace labels (required for namespaceSelector)
kubectl get namespaces --show-labels

# Check pod labels (required for podSelector)
kubectl get pods -n <namespace> --show-labels
```

---

## Key Commands Reference — Lab 14

```bash
# NetworkPolicy management
kubectl get networkpolicies -n <namespace>
kubectl get netpol -n <namespace>              # Short alias
kubectl describe networkpolicy <name> -n <namespace>
kubectl describe networkpolicies -n <namespace>  # All at once

# Test connectivity from inside a pod
kubectl exec <pod> -n <namespace> -- \
  wget -q -O- --timeout=5 http://<target-svc>
kubectl exec <pod> -n <namespace> -- \
  nc -zv <target-svc> <port>                  # TCP connect test
kubectl exec <pod> -n <namespace> -- \
  nslookup <service>                           # DNS test

# Port-forward for isolation testing (bypasses NetworkPolicy)
kubectl port-forward svc/<name> <local>:<port> -n <namespace>

# Check namespace labels (for namespaceSelector)
kubectl get namespace --show-labels
kubectl label namespace <name> key=value

# Check pod labels (for podSelector)
kubectl get pods -n <namespace> --show-labels

# Run a one-off test pod with specific labels
kubectl run netpol-test \
  --image=busybox:1.35 \
  --rm -it \
  --restart=Never \
  --labels="app=frontend" \
  -n <namespace> \
  -- wget -q -O- --timeout=5 http://<target>

# Delete a NetworkPolicy
kubectl delete networkpolicy <name> -n <namespace>

# Apply multiple policies at once
kubectl apply -f ./network-policies/
```

---

## Prod Wisdom — The Senior Engineer Mindset

Three things that define senior-level NetworkPolicy thinking:

**1. They apply default deny first, then open what's needed.** Starting with all-allow and then adding restrictions is backwards — there's always a window where traffic you didn't mean to allow gets through. The secure pattern is: apply `default-deny-all` and DNS egress as the baseline, then add specific allow policies for each traffic flow you need. This way, anything not explicitly allowed is denied from day one.

**2. They always test both directions — allowed AND denied.** After applying policies, they run the full access matrix: every pair of pods that should communicate, and every pair that shouldn't. A policy that blocks the wrong traffic is just as broken as one that allows it. Automated connectivity tests in CI/CD that verify the access matrix catch policy regressions before they reach prod.

**3. They know port-forward bypasses NetworkPolicy.** This is the key isolation technique for debugging. If a pod responds to port-forward but not to service-to-service traffic, the problem is NetworkPolicy (or Service config). If a pod doesn't respond to port-forward, the problem is the app itself. This single test immediately tells you whether to look at NetworkPolicy or the application.

The NetworkPolicy debug order:
```
port-forward to target pod → responds? (eliminates app issues)
kubectl get networkpolicies  → policies exist in this namespace?
kubectl describe each policy → which pods do they select?
kubectl get pods --show-labels → do source/target labels match policy selectors?
kubectl get namespace --show-labels → do namespace labels match namespaceSelector?
```

---

## Cleanup

```bash
kubectl delete namespace lab14
kubectl delete namespace lab14-monitoring
```

---

*Lab 14 complete. Move to Lab 15 — Taints, Tolerations & Node Affinity when ready.*