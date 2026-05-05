# Lab 07 — Ingress

## What This Lab Is About

Ingress is how HTTP and HTTPS traffic from outside the cluster reaches your services. It sits in front of everything — it's the front door. A misconfigured Ingress means your users can't reach your app, even when every pod, service, and deployment behind it is perfectly healthy. The app is running. The service has endpoints. But the front door is broken and you get 404, 502, or a connection timeout.

This lab covers the 4 most critical Ingress failure modes in real clusters. An Ingress rule that routes to a non-existent service backend. Path routing that silently drops requests because of a missing path type or prefix mismatch. TLS misconfiguration that breaks HTTPS. And the full systematic debug workflow when Ingress returns 502 or 404 for no obvious reason.

> Ingress failures are deceptive — the problem is rarely where you think it is. It could be the Ingress rule, the controller, the backend service, or the TLS secret. The only way to find it is a systematic top-down elimination.

---

## Environment

- **Cluster:** KIND (Kubernetes IN Docker)
- **K8s Version:** v1.27.3
- **Node:** Single node (`kind-control-plane`)
- **Namespace for this lab:** `lab07`

### Install NGINX Ingress Controller for KIND

Ingress requires a controller to function. For KIND, the NGINX Ingress Controller is the standard choice. Install it before starting the scenarios.

```bash
# Install NGINX Ingress Controller with KIND-specific config
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Wait for the controller to be ready
kubectl rollout status deployment ingress-nginx-controller \
  -n ingress-nginx --timeout=90s

# Verify
kubectl get pods -n ingress-nginx
# NAME                                        READY   STATUS      RESTARTS   AGE
# ingress-nginx-controller-xxx-yyy            1/1     Running     0          60s

# Verify the IngressClass exists
kubectl get ingressclass
# NAME    CONTROLLER             PARAMETERS   AGE
# nginx   k8s.io/ingress-nginx   <none>       60s
```

```bash
# Create the lab namespace
kubectl create namespace lab07
kubectl config set-context --current --namespace=lab07
```

---

## The 4 Scenarios

| # | Scenario | What You'll Learn |
|---|---|---|
| 01 | Ingress routes to wrong backend service | Service name mismatch, 503 from controller |
| 02 | Path routing silently drops requests | `pathType`, prefix vs exact, trailing slash trap |
| 03 | TLS misconfiguration breaks HTTPS | Wrong secret name, missing secret, cert mismatch |
| 04 | 502 Bad Gateway — full debug workflow | Systematic top-down elimination for any Ingress failure |

---

## Scenario 01 — Ingress Routes to Wrong Backend Service

### What You'll Break

An Ingress rule pointing to a backend service that either doesn't exist or has the wrong name. The Ingress controller can't find the service, returns 503, and the user sees a "Service Temporarily Unavailable" from nginx — not from your app. This is one of the most common Ingress mistakes.

### Apply the Broken State

```bash
# Deploy a healthy backend app
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: lab07
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: app
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: web-app-svc            # Service is named "web-app-svc"
  namespace: lab07
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
EOF

# Create Ingress pointing to WRONG service name
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-ingress
  namespace: lab07
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: web-app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app-service   # WRONG: actual service is "web-app-svc"
            port:
              number: 80
EOF
```

### Symptoms You Will Observe

```bash
# Add host entry to /etc/hosts for local testing
echo "127.0.0.1 web-app.local" | sudo tee -a /etc/hosts

# Test the Ingress (KIND exposes Ingress on localhost port 80)
curl -s http://web-app.local
# <html>
# <head><title>503 Service Temporarily Unavailable</title></head>
# ...
# nginx/1.x.x
```

503 from nginx — not from your app. The controller received the request but couldn't route it.

```bash
kubectl get ingress web-app-ingress -n lab07
# NAME               CLASS   HOSTS          ADDRESS     PORTS   AGE
# web-app-ingress    nginx   web-app.local  localhost   80      30s
```

Ingress has an address — controller is running. But backend is broken.

### Investigate

```bash
# Step 1 — Describe the Ingress
kubectl describe ingress web-app-ingress -n lab07
# Rules:
#   Host          Path  Backends
#   web-app.local /     web-app-service:80 (<error: endpoints "web-app-service" not found>)
# The error is right there in describe output

# Step 2 — Check what services actually exist
kubectl get svc -n lab07
# NAME          TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
# web-app-svc   ClusterIP   10.96.x.x     <none>        80/TCP    2m
# Service is "web-app-svc" not "web-app-service"

# Step 3 — Check Ingress controller logs for the error
kubectl logs -n ingress-nginx \
  $(kubectl get pod -n ingress-nginx -l app.kubernetes.io/component=controller \
  -o jsonpath='{.items[0].metadata.name}') --tail=20
# "error obtaining service endpoints" service="lab07/web-app-service"
```

The `describe ingress` output is your fastest path — it shows the backend error inline.

### Fix

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-ingress
  namespace: lab07
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: web-app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app-svc      # Fixed: correct service name
            port:
              number: 80
EOF

# Verify describe shows no errors now
kubectl describe ingress web-app-ingress -n lab07
# Backends: web-app-svc:80 (10.244.x.x:80,10.244.x.y:80)
# IPs listed means backend is healthy

# Test
curl -s http://web-app.local | grep -i "welcome"
# <h1>Welcome to nginx!</h1> — working
```

### The Ingress Backend Verification Checklist

```bash
# Always run these three after creating/updating an Ingress
# 1 — Check Ingress describe for backend errors
kubectl describe ingress <name> -n <namespace>

# 2 — Verify the backend service exists and has the right port
kubectl get svc <backend-service-name> -n <namespace>

# 3 — Verify backend service has endpoints (pods are running and selected)
kubectl get endpoints <backend-service-name> -n <namespace>
```

### Prod Wisdom

`kubectl describe ingress` is the most underused Ingress debug command. It shows the resolved backend IPs inline — if you see `<error: endpoints not found>` next to a backend, that's your entire answer. The service name is wrong, the port is wrong, or the service doesn't exist. Compare it against `kubectl get svc` and the fix is immediate. Always `describe` the Ingress before assuming the problem is in your application.

---

## Scenario 02 — Path Routing Silently Drops Requests

### What You'll Break

An Ingress with multiple path rules where one path never matches because of `pathType` misconfiguration, wrong prefix, or the trailing slash trap. Requests to `/api/users` return 404 from nginx — not from your app — because the path rule doesn't match. The rule is there. The service is healthy. But the path matching is wrong.

### Apply the Broken State

```bash
# Deploy two backend apps
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: lab07
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: app
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
  namespace: lab07
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-backend
  namespace: lab07
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api-backend
  template:
    metadata:
      labels:
        app: api-backend
    spec:
      containers:
      - name: app
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: api-svc
  namespace: lab07
spec:
  selector:
    app: api-backend
  ports:
  - port: 80
    targetPort: 80
EOF

# Create Ingress with path routing issues
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-ingress
  namespace: lab07
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix        # Matches everything — frontend
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
      - path: /api
        pathType: Exact         # PROBLEM: Exact only matches "/api" not "/api/users"
        backend:
          service:
            name: api-svc
            port:
              number: 80
EOF
```

```bash
echo "127.0.0.1 myapp.local" | sudo tee -a /etc/hosts

# Test frontend path — works
curl -s http://myapp.local/ | grep -i "welcome"
# Works — nginx welcome page

# Test exact API path — works for /api exactly
curl -s -o /dev/null -w "%{http_code}" http://myapp.local/api
# 200

# Test a real API subpath — FAILS — goes to frontend instead of api-svc
curl -s -o /dev/null -w "%{http_code}" http://myapp.local/api/users
# 200 but from frontend-svc — NOT from api-svc — silent mismatch
# The Prefix "/" catches everything that doesn't exactly match "/api"
```

The request silently routes to the wrong backend. No 404 — just wrong responses. This is the most dangerous type of path routing bug.

### Investigate

```bash
# Step 1 — Describe Ingress to see rules
kubectl describe ingress path-ingress -n lab07
# Rules:
#   myapp.local
#     /      Prefix   frontend-svc:80
#     /api   Exact    api-svc:80

# Step 2 — Understand pathType matching rules
# Exact:  ONLY matches the exact path string — /api matches /api, NOT /api/users
# Prefix: Matches the path and all sub-paths — /api matches /api, /api/users, /api/v2/...
# ImplementationSpecific: Controller-defined behaviour

# Step 3 — Test path routing explicitly
curl -v http://myapp.local/api/users 2>&1 | grep "< HTTP\|Server:"
# Check response headers to see which backend responded
```

### Fix — Use Prefix pathType for the API

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-ingress
  namespace: lab07
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /api(/|$)(.*)      # Regex: matches /api and all subpaths
        pathType: ImplementationSpecific
        backend:
          service:
            name: api-svc
            port:
              number: 80
      - path: /
        pathType: Prefix          # Catch-all — frontend last
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
EOF

# Test all paths
curl -s -o /dev/null -w "%{http_code}" http://myapp.local/
# 200 — frontend

curl -s -o /dev/null -w "%{http_code}" http://myapp.local/api
# 200 — api-svc

curl -s -o /dev/null -w "%{http_code}" http://myapp.local/api/users
# 200 — api-svc — now correctly routed
```

### Path Routing Rules — Know All Three Types

```
pathType: Exact
  Matches:     /api
  No match:    /api/, /api/users, /api/v2

pathType: Prefix
  Matches:     /api, /api/, /api/users, /api/v2/anything
  No match:    /apiv2 (must be path segment boundary)

pathType: ImplementationSpecific
  Matching:    Defined by the Ingress controller (nginx uses regex)
  Use when:    You need regex matching or complex rewrite rules

Path ordering rule:
  Longest path wins — /api/v2 is checked before /api before /
  Exact before Prefix at the same path length
  Always put catch-all "/" last
```

### The Trailing Slash Trap

```bash
# This is a common silent failure in prod
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: slash-trap
  namespace: lab07
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /dashboard/        # Note trailing slash
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
EOF

# /dashboard (no trailing slash) — does NOT match /dashboard/
curl -s -o /dev/null -w "%{http_code}" http://myapp.local/dashboard
# 404 — doesn't match because path is /dashboard/ with trailing slash

# /dashboard/ — matches
curl -s -o /dev/null -w "%{http_code}" http://myapp.local/dashboard/
# 200
```

Fix: Remove trailing slash from path rules unless you specifically need it:

```yaml
path: /dashboard    # Without trailing slash matches /dashboard and /dashboard/*
pathType: Prefix
```

### Prod Wisdom

Path routing bugs are silent in prod — requests don't 404, they go to the wrong backend and return wrong responses. Users report "wrong data" or "wrong page" instead of an error, making it much harder to diagnose. Always test every path explicitly after creating multi-path Ingress rules. And always put the most specific paths first and the catch-all `/` last — the NGINX controller evaluates rules top to bottom by specificity, and a misplaced catch-all swallows everything.

---

## Scenario 03 — TLS Misconfiguration Breaks HTTPS

### What You'll Break

An Ingress with TLS enabled but either the wrong secret name, a secret that doesn't exist, or a certificate that doesn't match the host. The result is HTTPS failures — either a TLS handshake error, a browser certificate warning, or a fallback to HTTP that your app doesn't expect.

### Create a Self-Signed Certificate for Testing

```bash
# Generate a self-signed cert for testing (requires openssl)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=secure-app.local/O=lab07" \
  -addext "subjectAltName=DNS:secure-app.local"

# Store as a TLS Secret in K8s
kubectl create secret tls secure-app-tls \
  --cert=tls.crt \
  --key=tls.key \
  -n lab07

# Verify the secret
kubectl get secret secure-app-tls -n lab07
# NAME             TYPE                DATA   AGE
# secure-app-tls   kubernetes.io/tls   2      5s
```

### Apply the Broken State — Wrong Secret Name

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  namespace: lab07
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - secure-app.local
    secretName: secure-app-tls-v2   # WRONG: secret is named "secure-app-tls"
  rules:
  - host: secure-app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app-svc
            port:
              number: 80
EOF

echo "127.0.0.1 secure-app.local" | sudo tee -a /etc/hosts

# Test HTTPS
curl -k https://secure-app.local
# curl: (35) OpenSSL SSL_connect: SSL_ERROR_SYSCALL
# OR you get the NGINX fake certificate (ingress-nginx default cert)
# NOT your app's certificate

# Check what certificate is being served
curl -vk https://secure-app.local 2>&1 | grep "subject:\|issuer:\|CN="
# CN=Kubernetes Ingress Controller Fake Certificate
# This is the default fallback cert — your real cert was not found
```

### Investigate

```bash
# Step 1 — Describe Ingress — check TLS section
kubectl describe ingress tls-ingress -n lab07
# TLS:
#   secure-app-tls-v2 terminates secure-app.local
# No error shown here — describe doesn't validate secret existence

# Step 2 — Check if the secret actually exists
kubectl get secret secure-app-tls-v2 -n lab07
# Error from server (NotFound): secrets "secure-app-tls-v2" not found

kubectl get secret -n lab07
# NAME             TYPE                DATA   AGE
# secure-app-tls   kubernetes.io/tls   2      5m
# Correct secret name is "secure-app-tls" not "secure-app-tls-v2"

# Step 3 — Check Ingress controller logs for TLS errors
kubectl logs -n ingress-nginx \
  $(kubectl get pod -n ingress-nginx \
  -l app.kubernetes.io/component=controller \
  -o jsonpath='{.items[0].metadata.name}') --tail=30 | grep -i "tls\|secret\|cert"
# Error getting SSL certificate "lab07/secure-app-tls-v2": local SSL certificate
# lab07/secure-app-tls-v2 was not found. Using default certificate

# Step 4 — Verify the secret type is correct
kubectl get secret secure-app-tls -n lab07 -o jsonpath='{.type}'
# kubernetes.io/tls   ← correct type
```

### Fix — Correct Secret Name

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  namespace: lab07
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - secure-app.local
    secretName: secure-app-tls    # Fixed: correct secret name
  rules:
  - host: secure-app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app-svc
            port:
              number: 80
EOF

# Verify your cert is now being served (not the fake one)
curl -vk https://secure-app.local 2>&1 | grep "CN="
# CN=secure-app.local   ← your cert

# Full HTTPS test
curl -sk https://secure-app.local | grep -i "welcome"
# Welcome to nginx — working over HTTPS
```

### Break It — Host Mismatch in TLS

```bash
# Secret cert is for secure-app.local
# Ingress TLS hosts lists a different domain
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-mismatch
  namespace: lab07
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - different-app.local        # MISMATCH: cert is for secure-app.local
    secretName: secure-app-tls
  rules:
  - host: different-app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app-svc
            port:
              number: 80
EOF

echo "127.0.0.1 different-app.local" | sudo tee -a /etc/hosts

# Test — gets fake cert because host doesn't match cert CN
curl -vk https://different-app.local 2>&1 | grep "CN="
# CN=Kubernetes Ingress Controller Fake Certificate
# Served fake cert because host didn't match certificate SAN
```

### TLS Secret Requirements

```bash
# A TLS secret must have exactly these two keys:
kubectl get secret secure-app-tls -n lab07 -o json | \
  jq '.data | keys'
# ["tls.crt", "tls.key"]

# Inspect the certificate details
kubectl get secret secure-app-tls -n lab07 \
  -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | openssl x509 -noout -text | grep -A 5 "Subject Alternative\|Subject:"
# Subject: CN=secure-app.local
# DNS:secure-app.local
# The cert CN and SAN must match the Ingress host
```

### Force HTTPS Redirect

```bash
# In prod you always want HTTP → HTTPS redirect
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  namespace: lab07
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"      # Redirect HTTP to HTTPS
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true" # Even behind load balancers
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - secure-app.local
    secretName: secure-app-tls
  rules:
  - host: secure-app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app-svc
            port:
              number: 80
EOF

# HTTP request now redirects to HTTPS
curl -v http://secure-app.local 2>&1 | grep "< HTTP\|Location:"
# HTTP/1.1 308 Permanent Redirect
# Location: https://secure-app.local
```

### Prod Wisdom

TLS issues in Ingress are almost always one of three things — wrong secret name, wrong namespace (secret must be in the same namespace as the Ingress), or host mismatch between the Ingress rule and the certificate SAN. The tell is the NGINX fake certificate — when you see `CN=Kubernetes Ingress Controller Fake Certificate` in the TLS handshake, your real cert was not found or not matched. Check the controller logs immediately — they will say exactly which secret it couldn't find.

---

## Scenario 04 — 502 Bad Gateway — Full Debug Workflow

### What You'll Break

A 502 Bad Gateway from the Ingress controller. This means the controller received the request, found the backend, tried to forward it — and the backend rejected or failed to respond. Unlike 503 (backend not found) or 404 (path not matched), 502 means the controller IS talking to the pod but the pod is returning an error or timing out. This is the most nuanced Ingress failure.

### Apply the Broken State

```bash
# Deploy a backend that crashes on startup (simulates a bad deployment)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-backend
  namespace: lab07
spec:
  replicas: 2
  selector:
    matchLabels:
      app: broken-backend
  template:
    metadata:
      labels:
        app: broken-backend
    spec:
      containers:
      - name: app
        image: nginx:1.25
        ports:
        - containerPort: 80
        # Readiness probe pointing to wrong path — pods never become Ready
        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 2
          periodSeconds: 3
          failureThreshold: 2
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: broken-backend-svc
  namespace: lab07
spec:
  selector:
    app: broken-backend
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: broken-ingress
  namespace: lab07
spec:
  ingressClassName: nginx
  rules:
  - host: broken-app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: broken-backend-svc
            port:
              number: 80
EOF

echo "127.0.0.1 broken-app.local" | sudo tee -a /etc/hosts
```

```bash
curl -s -o /dev/null -w "%{http_code}" http://broken-app.local
# 502
```

### The Systematic 502 Debug Workflow

**Run every step in order. Every time. Build the habit.**

```bash
# STEP 1 — Confirm the error and type
curl -v http://broken-app.local 2>&1 | grep "< HTTP"
# HTTP/1.1 502 Bad Gateway
# 502 = controller reached backend, backend failed to respond

# STEP 2 — Describe the Ingress
kubectl describe ingress broken-ingress -n lab07
# Check:
# - Does the backend show real IPs or <error: endpoints not found>?
# - Is the host correct?
# - Is the path correct?
# Backends: broken-backend-svc:80 (10.244.x.x:80,10.244.y.y:80)
# IPs are there — controller CAN reach the backend

# STEP 3 — Check backend pods
kubectl get pods -n lab07 -l app=broken-backend
# NAME                          READY   STATUS    RESTARTS   AGE
# broken-backend-xxx-aaa        0/1     Running   0          2m
# broken-backend-xxx-bbb        0/1     Running   0          2m
# READY is 0/1 — pods are NOT ready

# STEP 4 — Check Endpoints (are pods in Service rotation?)
kubectl get endpoints broken-backend-svc -n lab07
# NAME                  ENDPOINTS   AGE
# broken-backend-svc    <none>      2m
# No endpoints — pods are Running but not Ready, so not in Service

# STEP 5 — Find out why pods aren't Ready
kubectl describe pod $(kubectl get pod -n lab07 -l app=broken-backend \
  -o jsonpath='{.items[0].metadata.name}') -n lab07 | grep -A 10 "Events\|Readiness"
# Warning  Unhealthy  kubelet  Readiness probe failed:
# HTTP probe failed with statuscode: 404

# STEP 6 — Check Ingress controller logs
kubectl logs -n ingress-nginx \
  $(kubectl get pod -n ingress-nginx \
  -l app.kubernetes.io/component=controller \
  -o jsonpath='{.items[0].metadata.name}') --tail=30
# connect() failed (111: Connection refused) while connecting to upstream
# No live upstreams while connecting to upstream

# STEP 7 — Port-forward directly to a pod to isolate (bypasses Ingress + Service)
POD=$(kubectl get pod -n lab07 -l app=broken-backend \
  -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward pod/${POD} 9090:80 -n lab07 &
curl -s http://localhost:9090
# Returns nginx HTML — pod is alive and responding on port 80
kill %1

# STEP 8 — Port-forward to Service (bypasses Ingress only)
kubectl port-forward svc/broken-backend-svc 9091:80 -n lab07 &
curl -s http://localhost:9091
# curl: (7) Failed to connect — service has no endpoints (pods not Ready)
kill %1
```

**Diagnosis:** Pod is alive and responding on port 80. But readiness probe fails (wrong path `/ready`). So pods never enter Service Endpoints. Ingress → Service → no endpoints → 502.

### Fix

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-backend
  namespace: lab07
spec:
  replicas: 2
  selector:
    matchLabels:
      app: broken-backend
  template:
    metadata:
      labels:
        app: broken-backend
    spec:
      containers:
      - name: app
        image: nginx:1.25
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /              # Fixed: correct path nginx actually serves
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
EOF

# Wait for pods to become Ready
kubectl rollout status deployment/broken-backend -n lab07

# Verify endpoints are populated now
kubectl get endpoints broken-backend-svc -n lab07
# ENDPOINTS: 10.244.x.x:80,10.244.y.y:80

# Test
curl -s -o /dev/null -w "%{http_code}" http://broken-app.local
# 200 — working
```

### The Complete Ingress Debug Decision Tree

```
Ingress not working?
│
├── What HTTP status are you getting?
│   ├── Connection refused / timeout
│   │   └── Is the Ingress controller running?
│   │       kubectl get pods -n ingress-nginx
│   │
│   ├── 404
│   │   └── Path not matching any rule
│   │       → kubectl describe ingress — check path rules and host
│   │       → Is the correct Host header being sent?
│   │       → pathType correct? (Exact vs Prefix)
│   │
│   ├── 503 Service Temporarily Unavailable
│   │   └── Backend service not found
│   │       → kubectl describe ingress — look for "endpoints not found"
│   │       → kubectl get svc — does backend service exist?
│   │       → Service name/port in Ingress matches actual Service?
│   │
│   └── 502 Bad Gateway
│       └── Controller reached backend but backend failed
│           → kubectl get endpoints <backend-svc> — any endpoints?
│           │   ├── No endpoints → pods not Ready → check readiness probe
│           │   └── Has endpoints → pod responding?
│           │       → kubectl port-forward pod/<name> → test directly
│           └── kubectl logs ingress-nginx controller — what error?
│
├── TLS issues?
│   ├── Seeing "Fake Certificate" → wrong secret name or host mismatch
│   │   → kubectl get secret — does TLS secret exist in same namespace?
│   │   → cert CN/SAN matches Ingress host?
│   └── kubectl logs ingress-nginx controller | grep -i "tls\|cert\|secret"
│
└── Always check Ingress controller logs last
    kubectl logs -n ingress-nginx <controller-pod> --tail=50
```

---

## Key Commands Reference — Lab 07

```bash
# Ingress inspection
kubectl get ingress -n <namespace>
kubectl describe ingress <name> -n <namespace>

# Ingress controller logs (most useful for 502/503 debugging)
kubectl logs -n ingress-nginx \
  $(kubectl get pod -n ingress-nginx \
  -l app.kubernetes.io/component=controller \
  -o jsonpath='{.items[0].metadata.name}') --tail=50

# Check IngressClass
kubectl get ingressclass

# Test HTTP paths
curl -v http://<host>/<path>
curl -sk https://<host>/<path>

# Check TLS certificate being served
curl -vk https://<host> 2>&1 | grep "CN=\|subject:\|issuer:"

# Verify TLS secret
kubectl get secret <name> -n <namespace>
kubectl get secret <name> -n <namespace> -o jsonpath='{.type}'
kubectl get secret <name> -n <namespace> \
  -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text

# Port-forward for layer isolation
kubectl port-forward svc/<backend-svc> <local>:<port> -n <namespace>
kubectl port-forward pod/<backend-pod> <local>:<port> -n <namespace>

# Check backend endpoints
kubectl get endpoints <backend-svc> -n <namespace>

# Full status check
kubectl get ingress,svc,endpoints,pods -n <namespace>
```

---

## Prod Wisdom — The Senior Engineer Mindset

Three things that define senior-level Ingress debugging:

**1. They know the status code tells them exactly where the failure is.** 404 = path matching. 503 = service not found. 502 = service found but backend not responding. Each code maps to a specific layer. They go straight to that layer instead of checking everything randomly.

**2. They use port-forward to isolate the layer.** Ingress → Service → Pod is a three-layer stack. Port-forward to the pod bypasses Ingress and Service. Port-forward to the Service bypasses Ingress only. Narrow it to one layer, fix that layer, verify. This turns a 20-minute debug session into a 3-minute one.

**3. They always check the Ingress controller logs.** `kubectl describe ingress` shows the Ingress resource state but not the controller's runtime errors. The controller logs show the actual error — wrong secret, no upstream, SSL handshake failed — with precise detail. Make it a habit: Ingress not working → controller logs within the first 60 seconds.

The Ingress debug order:
```
HTTP status code → describe ingress → backend endpoints
→ pod port-forward → service port-forward → controller logs
```

---

## Cleanup

```bash
# Remove /etc/hosts entries added during lab
sudo sed -i '/web-app.local\|myapp.local\|secure-app.local\|broken-app.local\|different-app.local/d' /etc/hosts

# Delete namespace
kubectl delete namespace lab07

# Optionally remove ingress-nginx (keep for Lab 08 if needed)
# kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

---

*Lab 07 complete. Move to Lab 08 — Logging & Debugging Workflow when ready.*