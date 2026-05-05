# Lab 12 — Namespace & Context Management

## What This Lab Is About

Namespaces are the primary isolation boundary in Kubernetes. Contexts are how you switch between clusters and namespaces without rewriting every command. Together they are the operational layer that separates dev from staging from prod — and when you get them wrong, the consequences range from deploying to the wrong environment to accidentally deleting production resources.

This lab covers the 5 most critical namespace and context failure modes in real clusters. Accidentally running commands in the wrong namespace. Contexts misconfigured to point at the wrong cluster. Resources that look missing because you're in the wrong namespace. Cross-namespace communication patterns and where they break. And the production workflow for safely managing multiple clusters and environments without the career-ending `kubectl delete` in the wrong place.

> Context switching errors are the single most common cause of accidental production changes. Not malice. Not bad code. Just the wrong namespace active when someone ran a command they were sure was safe.

---

## Environment

- **Cluster:** KIND (Kubernetes IN Docker)
- **K8s Version:** v1.27.3
- **Node:** Single node (`kind-control-plane`)
- **Namespaces for this lab:** `lab12-dev`, `lab12-staging`, `lab12-prod`

```bash
kubectl create namespace lab12-dev
kubectl create namespace lab12-staging
kubectl create namespace lab12-prod
```

---

## The 5 Scenarios

| # | Scenario | What You'll Learn |
|---|---|---|
| 01 | Wrong namespace — resource appears missing | `-n` flag, default namespace trap, `--all-namespaces` |
| 02 | Context misconfiguration — wrong cluster | kubeconfig, context structure, safe switching |
| 03 | Accidental cross-namespace delete | Namespace as blast radius limiter, RBAC scoping |
| 04 | Cross-namespace communication — where it breaks | DNS, NetworkPolicy, ServiceAccount scope |
| 05 | Production multi-cluster workflow | Safe context switching, kubens, prompt engineering |

---

## Scenario 01 — Wrong Namespace — Resource Appears Missing

### What You'll Break

You deploy a resource into one namespace but query it from another. The resource appears to not exist. You spend 10 minutes checking your YAML, re-applying, wondering why K8s isn't working — when the actual problem is you're looking in the wrong namespace. This is the single most common K8s operational mistake.

### Apply the Broken State

```bash
# Deploy resources into specific namespaces
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: lab12-dev        # Deployed into dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
      - name: api
        image: nginx:1.25
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "64Mi"
            cpu: "100m"
---
apiVersion: v1
kind: Service
metadata:
  name: api-svc
  namespace: lab12-dev        # Service in dev
spec:
  selector:
    app: api-server
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
  namespace: lab12-staging    # ConfigMap in staging
data:
  ENV: "staging"
  LOG_LEVEL: "debug"
EOF
```

### Simulate the Confusion

```bash
# Engineer thinks they're looking at dev but forgets to specify -n
kubectl get deployment api-server
# Error from server (NotFound): deployments.apps "api-server" not found
# Looks like it doesn't exist — but it does, just in lab12-dev

kubectl get svc api-svc
# Error from server (NotFound): services "api-svc" not found

# What namespace am I actually in right now?
kubectl config view --minify | grep namespace
# namespace: lab08   ← still set from a previous lab

# Engineer checks "all" but only checks default
kubectl get deployments
# No resources found in default namespace.
# Still wrong — not checking lab12-dev
```

### The Correct Investigation Pattern

```bash
# Step 1 — Always know which namespace you're in
kubectl config view --minify --output 'jsonpath={..namespace}'
echo ""
# OR
kubectl config current-context

# Step 2 — Look in the specific namespace
kubectl get deployment api-server -n lab12-dev
# NAME         READY   UP-TO-DATE   AVAILABLE   AGE
# api-server   1/1     1            1           2m

# Step 3 — When you don't know which namespace — search all
kubectl get deployments -A | grep api-server
# lab12-dev   api-server   1/1   1   1   2m

kubectl get svc -A | grep api
# lab12-dev   api-svc   ClusterIP   10.96.x.x   <none>   80/TCP   2m

kubectl get configmap -A | grep api-config
# lab12-staging   api-config   1   3m

# Step 4 — Set your context to the right namespace
kubectl config set-context --current --namespace=lab12-dev

# Now commands work without -n
kubectl get deployment api-server
# Found — correct
kubectl get svc api-svc
# Found — correct
```

### The Namespace Scope of Every Resource

```bash
# Not all resources are namespaced — know the difference
# Namespaced resources (scoped to a namespace):
kubectl api-resources --namespaced=true | head -20
# pods, services, deployments, configmaps, secrets, pvc, ingress...

# Cluster-scoped resources (no namespace — global):
kubectl api-resources --namespaced=false
# nodes, persistentvolumes, clusterroles, clusterrolebindings,
# storageclasses, namespaces themselves, ingressclasses...

# Check if a specific resource is namespaced
kubectl api-resources | grep deployment
# deployments   deploy   apps/v1   true   Deployment
# "true" = namespaced

kubectl api-resources | grep node
# nodes   node   v1   false   Node
# "false" = cluster-scoped
```

### The Most Important Namespace Commands

```bash
# See all resources across all namespaces
kubectl get all -A
kubectl get pods -A
kubectl get deployments -A

# See everything in a specific namespace
kubectl get all -n lab12-dev

# Set default namespace for current context
kubectl config set-context --current --namespace=lab12-dev

# Get current namespace
kubectl config view --minify | grep namespace

# List all namespaces and their status
kubectl get namespaces
# NAME              STATUS   AGE
# default           Active   6d
# kube-system       Active   6d
# kube-public       Active   6d
# kube-node-lease   Active   6d
# lab12-dev         Active   5m
# lab12-staging     Active   5m
# lab12-prod        Active   5m
```

### Prod Wisdom

**Always explicitly specify `-n <namespace>` in scripts and CI/CD pipelines** — never rely on the context's default namespace. A context's default namespace can be changed by any engineer at any time, making scripts that omit `-n` unpredictably dangerous. In interactive use, set the namespace explicitly with `kubectl config set-context --current --namespace=<ns>` and verify it before every significant operation. `kubectl get all -A` before any mass delete is non-negotiable.

---

## Scenario 02 — Context Misconfiguration — Wrong Cluster

### What You'll Learn

A context is a named combination of cluster + user + namespace. Misconfigured contexts — pointing at the wrong cluster endpoint, using stale credentials, or having ambiguous names — cause commands to silently hit the wrong cluster or fail with cryptic auth errors.

### Understand the kubeconfig Structure

```bash
# View the full kubeconfig
kubectl config view

# Output structure:
# clusters:      ← the K8s API server endpoints
# users:         ← credentials (certs, tokens)
# contexts:      ← named combinations of cluster + user + namespace
# current-context: ← which context is active right now
```

```bash
# See just the essential info
kubectl config view --minify
# clusters:
# - cluster:
#     server: https://127.0.0.1:34573  ← API server URL
#   name: kind-kind
# contexts:
# - context:
#     cluster: kind-kind
#     namespace: lab12-dev
#     user: kind-kind
#   name: kind-kind
# current-context: kind-kind
```

### Inspect Contexts

```bash
# List all contexts in kubeconfig
kubectl config get-contexts
# CURRENT   NAME        CLUSTER     AUTHINFO    NAMESPACE
# *         kind-kind   kind-kind   kind-kind   lab12-dev

# The * shows which context is currently active
# In a real multi-cluster setup you'd see:
# *   prod-us-east   prod-cluster   prod-admin    default
#     staging        stg-cluster    stg-user      default
#     dev-local      kind-kind      kind-kind     lab12-dev

# Get just the current context name
kubectl config current-context
# kind-kind

# Get the current context's cluster
kubectl config view --minify \
  -o jsonpath='{.clusters[0].cluster.server}'
# https://127.0.0.1:34573
```

### Create Multiple Contexts (Simulate Multi-Cluster)

```bash
# In a real environment you have multiple contexts for different clusters
# Simulate by creating contexts pointing at the same cluster but different namespaces

# Create a "prod" context
kubectl config set-context lab12-prod-context \
  --cluster=kind-kind \
  --user=kind-kind \
  --namespace=lab12-prod

# Create a "staging" context
kubectl config set-context lab12-staging-context \
  --cluster=kind-kind \
  --user=kind-kind \
  --namespace=lab12-staging

# Create a "dev" context
kubectl config set-context lab12-dev-context \
  --cluster=kind-kind \
  --user=kind-kind \
  --namespace=lab12-dev

# Verify all contexts exist
kubectl config get-contexts
# CURRENT   NAME                     CLUSTER     AUTHINFO    NAMESPACE
# *         kind-kind                kind-kind   kind-kind   lab12-dev
#           lab12-dev-context        kind-kind   kind-kind   lab12-dev
#           lab12-staging-context    kind-kind   kind-kind   lab12-staging
#           lab12-prod-context       kind-kind   kind-kind   lab12-prod
```

### Break It — Switch to Wrong Context

```bash
# Deploy something "important" in prod namespace
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: prod-database-config
  namespace: lab12-prod
data:
  DB_HOST: "prod-postgres.internal"
  DB_NAME: "production_db"
EOF

# Switch to prod context
kubectl config use-context lab12-prod-context

# Verify you're in prod
kubectl config current-context
# lab12-prod-context
kubectl config view --minify | grep namespace
# namespace: lab12-prod

# Now accidentally switch to staging thinking it's dev
kubectl config use-context lab12-staging-context

# Think you're in dev, run a delete
kubectl get configmaps
# NAME               DATA   AGE
# api-config         2      10m   ← this is STAGING config

# If you ran kubectl delete configmap api-config thinking you're in dev
# You'd actually delete staging config — silent misfire
```

### The Safe Context Switching Workflow

```bash
# ALWAYS verify after switching contexts
kubectl config use-context lab12-dev-context

# Verification triple-check:
echo "Context: $(kubectl config current-context)"
echo "Cluster: $(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')"
echo "Namespace: $(kubectl config view --minify -o jsonpath='{..namespace}')"
# Context:   lab12-dev-context
# Cluster:   https://127.0.0.1:34573
# Namespace: lab12-dev

# The one-liner verification command
kubectl config view --minify \
  -o jsonpath='Context: {.current-context}{"\n"}Cluster: {.clusters[0].cluster.server}{"\n"}Namespace: {..namespace}{"\n"}'
```

### Build a Safe Context Alias

```bash
# Add to ~/.bashrc or ~/.zshrc
# Shows cluster and namespace in your prompt
# kctx function — switch context and verify

# Quick verification alias
alias kctx-verify='echo "Context: $(kubectl config current-context) | Cluster: $(kubectl config view --minify -o jsonpath={.clusters[0].cluster.server}) | Namespace: $(kubectl config view --minify -o jsonpath={..namespace})"'

# Switch and immediately verify
kubectl config use-context lab12-prod-context && kctx-verify
# Context: lab12-prod-context | Cluster: https://127.0.0.1:34573 | Namespace: lab12-prod
```

### Dangerous Context Mistakes and Prevention

```bash
# Mistake 1 — Ambiguous context names
# BAD:  context named "prod" but cluster is staging
# GOOD: context named "prod-us-east-1-production"
#       Always include: environment + region + purpose

# Mistake 2 — Default namespace not set in context
# When no namespace is set, commands hit "default" namespace
kubectl config set-context lab12-prod-context \
  --cluster=kind-kind \
  --user=kind-kind \
  --namespace=lab12-prod    # Always set namespace in context

# Mistake 3 — Stale contexts pointing at decommissioned clusters
kubectl config get-contexts
# Clean up stale contexts
kubectl config delete-context <stale-context-name>

# Mistake 4 — Shared kubeconfig in CI/CD
# Each pipeline job should use its own isolated kubeconfig
# Never share a single kubeconfig file across parallel jobs
# KUBECONFIG env var can point to a different file
export KUBECONFIG=/tmp/ci-kubeconfig-${BUILD_ID}
```

### Prod Wisdom

**Name your contexts to be unmistakably clear about what they point at.** `prod` is a bad context name — it could be any prod cluster. `prod-us-east-eks-1` is a good context name — it tells you cloud provider, region, cluster type, and environment. In teams where multiple engineers share context names in runbooks and docs, ambiguous names are accidents waiting to happen. And always run the verification triple-check (current-context + cluster URL + namespace) after every context switch before running any write operation.

---

## Scenario 03 — Accidental Cross-Namespace Delete

### What You'll Break (and Prevent)

The most dangerous namespace mistake — running a destructive command in the wrong namespace. This scenario teaches how namespace isolation limits blast radius, how to verify before deleting, and the `--dry-run` pattern that should precede every mass delete.

### Set Up Resources to Protect

```bash
# Create important resources in prod namespace
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-service
  namespace: lab12-prod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: critical-service
  template:
    metadata:
      labels:
        app: critical-service
    spec:
      containers:
      - name: app
        image: nginx:1.25
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "64Mi"
            cpu: "100m"
---
apiVersion: v1
kind: Service
metadata:
  name: critical-svc
  namespace: lab12-prod
spec:
  selector:
    app: critical-service
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: prod-config
  namespace: lab12-prod
data:
  DB_HOST: "prod-db.internal"
  API_KEY: "prod-key-12345"
---
# Same named resources in dev (for confusion potential)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-service
  namespace: lab12-dev      # Same name as prod — easy to confuse
spec:
  replicas: 1
  selector:
    matchLabels:
      app: critical-service
  template:
    metadata:
      labels:
        app: critical-service
    spec:
      containers:
      - name: app
        image: nginx:1.25
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "64Mi"
            cpu: "100m"
EOF
```

### The --dry-run Pattern Before Any Delete

```bash
# NEVER delete without dry-run first
# This is the single most important habit in K8s operations

# Wrong — direct delete without verification
# kubectl delete deployment critical-service   ← which namespace?!

# Right — always dry-run first to see what WOULD be deleted
kubectl delete deployment critical-service \
  -n lab12-dev \
  --dry-run=client
# deployment.apps "critical-service" deleted (dry run)
# Now you see exactly what would be deleted — confirm namespace and resource

# Right — use -o yaml to see the full resource before deleting
kubectl get deployment critical-service -n lab12-dev -o yaml | \
  grep -E "name:|namespace:|replicas:"
# name: critical-service
# namespace: lab12-dev     ← confirm you're in dev, not prod
# replicas: 1

# Only after confirming:
kubectl delete deployment critical-service -n lab12-dev
```

### Mass Delete Protection Pattern

```bash
# Scenario: clean up all deployments in dev after a test
# The wrong way — could accidentally run in prod context:
# kubectl delete deployments --all

# The right way:
# Step 1 — Verify context
kubectl config current-context
kubectl config view --minify | grep namespace

# Step 2 — List what would be deleted
kubectl get deployments -n lab12-dev
# NAME               READY   UP-TO-DATE
# api-server         1/1     1
# critical-service   1/1     1
# (only 2 deployments in dev — looks right)

# Step 3 — Dry run the mass delete
kubectl delete deployments --all -n lab12-dev --dry-run=client
# deployment.apps "api-server" deleted (dry run)
# deployment.apps "critical-service" deleted (dry run)

# Step 4 — Execute with explicit namespace
kubectl delete deployments --all -n lab12-dev

# Step 5 — Verify prod is untouched
kubectl get deployments -n lab12-prod
# critical-service   3/3 — still running, untouched
```

### The Namespace Deletion Nuclear Option

```bash
# Deleting a namespace deletes EVERYTHING in it — instantly, irreversibly
# This is the most destructive single command in K8s

# WRONG: kubectl delete namespace lab12-prod  ← destroys all prod resources

# If you must delete a namespace:
# Step 1 — List everything in it first
kubectl get all -n lab12-prod
kubectl get configmaps,secrets,pvc,ingress -n lab12-prod

# Step 2 — Confirm the namespace name
echo "About to delete namespace: lab12-prod"
echo "Resources that will be destroyed:"
kubectl api-resources --namespaced=true -o name | \
  xargs -I{} kubectl get {} -n lab12-prod 2>/dev/null | grep -v "^$"

# Step 3 — If this is intentional test cleanup only:
kubectl delete namespace lab12-dev    # Only delete dev/test namespaces manually
# NEVER delete lab12-prod with a manual kubectl command in a real cluster
# Use GitOps / Terraform to manage prod namespace lifecycle
```

### ResourceQuota as a Guard

```bash
# Add ResourceQuota to prod namespace — limits what can be deployed
# Also acts as a signal: "this namespace matters, it has governance"
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: prod-quota
  namespace: lab12-prod
spec:
  hard:
    pods: "50"
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    services: "20"
    persistentvolumeclaims: "10"
EOF

# Any operation that exceeds quota fails — additional safety net
kubectl describe resourcequota prod-quota -n lab12-prod
```

### Prod Wisdom

**The blast radius of a mistake is limited to the namespace — if you're in the right namespace.** This is the entire reason namespaces exist. A delete in `lab12-dev` can never touch `lab12-prod`. Make this isolation work for you: add ResourceQuota to prod, add RBAC that limits who can write to prod, name your contexts unambiguously, and make `--dry-run=client` a muscle memory reflex before every delete. The engineers who never have prod incidents aren't the ones who never make mistakes — they're the ones whose mistakes can't reach prod.

---

## Scenario 04 — Cross-Namespace Communication — Where It Breaks

### What You'll Break

Services are namespaced. A pod in `lab12-dev` cannot reach a Service in `lab12-prod` using a short name — it needs the fully qualified DNS name. This is a communication boundary that surprises teams building microservices across namespaces.

### Apply the Setup

```bash
# Service in staging namespace
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  namespace: lab12-staging
spec:
  replicas: 1
  selector:
    matchLabels:
      app: auth-service
  template:
    metadata:
      labels:
        app: auth-service
    spec:
      containers:
      - name: auth
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
apiVersion: v1
kind: Service
metadata:
  name: auth-svc
  namespace: lab12-staging
spec:
  selector:
    app: auth-service
  ports:
  - port: 80
    targetPort: 80
EOF
```

### Break It — Short Name Across Namespaces

```bash
# Pod in dev trying to reach service in staging using short name
kubectl run cross-ns-debug \
  --image=busybox:1.35 \
  --rm -it \
  --restart=Never \
  -n lab12-dev \
  -- sh -c "
    echo '=== Test 1: Short name (will fail) ==='
    nslookup auth-svc 2>&1

    echo ''
    echo '=== Test 2: Wrong namespace (will fail) ==='
    nslookup auth-svc.lab12-dev.svc.cluster.local 2>&1

    echo ''
    echo '=== Test 3: Correct FQDN (will work) ==='
    nslookup auth-svc.lab12-staging.svc.cluster.local 2>&1

    echo ''
    echo '=== Test 4: HTTP via FQDN ==='
    wget -q -O- http://auth-svc.lab12-staging.svc.cluster.local 2>&1 | head -3
  "
```

Expected output:
```
=== Test 1: Short name (will fail) ===
nslookup: can't resolve 'auth-svc'
NXDOMAIN — short name only works within same namespace

=== Test 2: Wrong namespace (will fail) ===
nslookup: can't resolve 'auth-svc.lab12-dev.svc.cluster.local'
NXDOMAIN — service doesn't exist in lab12-dev

=== Test 3: Correct FQDN (will work) ===
Address: 10.96.x.x — resolves because full FQDN includes correct namespace

=== Test 4: HTTP via FQDN ===
Welcome to nginx! — cross-namespace communication working
```

### Cross-Namespace RBAC Scope

```bash
# ServiceAccount permissions are also namespace-scoped
# A Role in lab12-dev only grants permissions within lab12-dev
# To grant cross-namespace permissions you need ClusterRole + ClusterRoleBinding

# Example: SA in dev that can READ from staging namespace
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cross-ns-reader
  namespace: lab12-dev
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: staging-reader
  namespace: lab12-staging    # Role defined IN staging namespace
rules:
- apiGroups: [""]
  resources: ["configmaps", "services"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-sa-can-read-staging
  namespace: lab12-staging    # Binding in staging namespace
subjects:
- kind: ServiceAccount
  name: cross-ns-reader
  namespace: lab12-dev        # SA lives in dev but permission granted in staging
roleRef:
  kind: Role
  name: staging-reader
  apiGroup: rbac.authorization.k8s.io
EOF

# Verify cross-namespace permission
kubectl auth can-i list configmaps \
  --as=system:serviceaccount:lab12-dev:cross-ns-reader \
  -n lab12-staging
# yes — dev SA can list staging configmaps

kubectl auth can-i list configmaps \
  --as=system:serviceaccount:lab12-dev:cross-ns-reader \
  -n lab12-prod
# no — no permission in prod
```

### Cross-Namespace DNS Reference

```bash
# The FQDN pattern — memorise this
# <service-name>.<namespace>.svc.<cluster-domain>
# auth-svc.lab12-staging.svc.cluster.local

# Shortcuts that work:
# Within same namespace:
#   auth-svc

# Cross-namespace (preferred in app configs):
#   auth-svc.lab12-staging

# Fully qualified (always works everywhere):
#   auth-svc.lab12-staging.svc.cluster.local

# Verify DNS from inside a pod
kubectl run dns-check \
  --image=busybox:1.35 \
  --rm -it \
  --restart=Never \
  -n lab12-dev \
  -- sh -c "cat /etc/resolv.conf"
# nameserver 10.96.0.10
# search lab12-dev.svc.cluster.local svc.cluster.local cluster.local
# The search domains explain why short names only work in same namespace
```

### Prod Wisdom

**Cross-namespace communication is a deliberate architectural decision — not an accident.** When a service in `payments` needs to talk to a service in `auth`, that dependency should be documented, the DNS FQDN should be explicit in config, and the RBAC should be intentionally granted. If you're writing a short service name in an app's config and it's cross-namespace, the app will fail silently in production — it only worked in dev because both services happened to be in the same namespace. Always use FQDNs in application configuration. Always.

---

## Scenario 05 — Production Multi-Cluster Workflow

### What You'll Build

The complete production workflow for safely managing multiple clusters and namespaces. Shell prompt engineering, safe context switching habits, kubectl plugins, and the pre-operation checklist that prevents accidental prod changes.

### The Production Shell Prompt

```bash
# The most important safety tool is knowing WHERE you are at all times
# Add K8s context and namespace to your shell prompt

# For bash — add to ~/.bashrc:
parse_kube_ctx() {
  local ctx ns
  ctx=$(kubectl config current-context 2>/dev/null)
  ns=$(kubectl config view --minify -o jsonpath='{..namespace}' 2>/dev/null)
  ns=${ns:-default}
  if [[ -n "$ctx" ]]; then
    echo "[k8s: ${ctx}|${ns}]"
  fi
}
# Add to PS1:
# PS1='$(parse_kube_ctx) \u@\h:\w\$ '

# For zsh with oh-my-zsh — use the kube plugin
# plugins=(... kube-ps1)
# PROMPT='$(kube_ps1)'$PROMPT

# What this looks like:
# [k8s: prod-us-east|production] user@hostname:~$
# [k8s: lab12-dev-context|lab12-dev] user@hostname:~$
```

### Install kubens and kubectx (Strongly Recommended)

```bash
# kubectx — fast context switcher
# kubens  — fast namespace switcher
# Both by Ahmet Alp Balkan — the most useful kubectl plugins

# Install via krew (kubectl plugin manager)
kubectl krew install ctx
kubectl krew install ns

# Usage:
# kubectx              → list all contexts
# kubectx prod         → switch to prod context
# kubectx -            → switch back to previous context (toggle)
# kubens               → list all namespaces
# kubens lab12-dev     → switch to lab12-dev namespace
# kubens -             → switch to previous namespace

# Manual install (if krew not available):
# curl -L https://raw.githubusercontent.com/ahmetb/kubectx/master/kubectx -o /usr/local/bin/kubectx
# curl -L https://raw.githubusercontent.com/ahmetb/kubectx/master/kubens -o /usr/local/bin/kubens
# chmod +x /usr/local/bin/kubectx /usr/local/bin/kubens
```

### Safe Context Switching Function

```bash
# Add to ~/.bashrc or ~/.zshrc
# Safe kubectl context switch with mandatory verification
kswitch() {
  local target_context=$1
  if [[ -z "$target_context" ]]; then
    echo "Usage: kswitch <context-name>"
    kubectl config get-contexts
    return 1
  fi

  # Show what we're switching FROM
  echo "Switching FROM: $(kubectl config current-context)"
  echo "           TO:  ${target_context}"

  # Perform the switch
  kubectl config use-context "$target_context"

  # Always show verification after switch
  echo ""
  echo "=== VERIFY ==="
  echo "Context:   $(kubectl config current-context)"
  echo "Cluster:   $(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')"
  echo "Namespace: $(kubectl config view --minify -o jsonpath='{..namespace}')"
  echo "=============="

  # Extra warning for prod contexts
  if echo "$target_context" | grep -qi "prod\|production"; then
    echo ""
    echo "⚠️  WARNING: You are now in a PRODUCTION context."
    echo "⚠️  Double-check every command before running it."
  fi
}
```

### The Pre-Operation Checklist

```bash
# Run this checklist before EVERY significant kubectl write operation

pre_op_check() {
  echo "=== PRE-OPERATION CHECKLIST ==="
  echo ""
  echo "1. Current context:"
  echo "   $(kubectl config current-context)"
  echo ""
  echo "2. Cluster endpoint:"
  echo "   $(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')"
  echo ""
  echo "3. Default namespace:"
  echo "   $(kubectl config view --minify -o jsonpath='{..namespace}')"
  echo ""
  echo "4. Resources in target namespace:"
  kubectl get all -n "${1:-$(kubectl config view --minify -o jsonpath='{..namespace}')}"
  echo ""
  echo "Are you sure you want to proceed? (yes/no)"
}

# Run before any destructive operation:
# pre_op_check lab12-prod
```

### The --namespace Discipline

```bash
# In all automation, scripts, and CI/CD:
# ALWAYS use explicit -n flag — never rely on context default namespace

# BAD — relies on context namespace (changes when someone does kubens)
kubectl apply -f deployment.yaml

# GOOD — explicit namespace, immune to context namespace changes
kubectl apply -f deployment.yaml -n lab12-prod

# BAD — delete without namespace
kubectl delete deployment critical-service

# GOOD — explicit namespace + dry-run first
kubectl delete deployment critical-service -n lab12-dev --dry-run=client
kubectl delete deployment critical-service -n lab12-dev

# In Helm (also specify namespace explicitly)
# helm upgrade my-app ./chart --namespace lab12-prod
```

### Managing Multiple Kubeconfig Files

```bash
# In real multi-cluster setups, each cluster has its own kubeconfig file
# Never manually merge them — use KUBECONFIG env var to combine

# Method 1 — Point at multiple files (they merge automatically)
export KUBECONFIG=~/.kube/config:~/.kube/prod-config:~/.kube/staging-config

# Method 2 — Merge into one file safely
KUBECONFIG=~/.kube/config:~/.kube/new-cluster-config \
  kubectl config view --flatten > ~/.kube/merged-config
# Review the merged file before using it
cp ~/.kube/merged-config ~/.kube/config

# Method 3 — Per-session isolated kubeconfig (safest for prod operations)
# Create a temporary kubeconfig with only prod access
cp ~/.kube/prod-only-config /tmp/prod-session-kubeconfig
export KUBECONFIG=/tmp/prod-session-kubeconfig
# Now only prod context is available — can't accidentally use dev kubeconfig
# Unset when done:
unset KUBECONFIG
```

### Namespace Naming Conventions

```bash
# Good namespace naming conventions prevent confusion:

# Environment-based (most common):
# dev, staging, prod
# Or more explicit: app-dev, app-staging, app-prod

# Team-based (for platform teams):
# team-payments, team-auth, team-infra

# Combined (best for large orgs):
# payments-dev, payments-staging, payments-prod
# auth-dev, auth-staging, auth-prod

# What NOT to do:
# test    ← ambiguous — whose test?
# temp    ← never gets deleted
# new     ← new what? new when?
# default ← never put real workloads in default namespace

# Enforce naming with admission controllers in real prod clusters
# Or just establish the convention and document it in your runbook

# Label namespaces for easy filtering
kubectl label namespace lab12-prod environment=production team=platform
kubectl label namespace lab12-staging environment=staging team=platform
kubectl label namespace lab12-dev environment=development team=platform

# Now filter namespaces by label
kubectl get namespaces -l environment=production
# NAME          STATUS   AGE
# lab12-prod    Active   30m
```

### The Complete Context Safety Reference

```bash
# See everything — current state snapshot
kubectl config view

# Current context only
kubectl config current-context

# All contexts list
kubectl config get-contexts

# Switch context
kubectl config use-context <context-name>

# Set namespace in current context
kubectl config set-context --current --namespace=<namespace>

# Create a new context
kubectl config set-context <name> \
  --cluster=<cluster> \
  --user=<user> \
  --namespace=<namespace>

# Delete a context (doesn't delete the cluster — just the local config entry)
kubectl config delete-context <name>

# Rename a context
kubectl config rename-context <old-name> <new-name>

# Verify cluster you're talking to
kubectl cluster-info

# Unset current context (forces explicit -n on every command)
kubectl config unset current-context
```

---

## Key Commands Reference — Lab 12

```bash
# Namespace management
kubectl get namespaces
kubectl create namespace <name>
kubectl delete namespace <name>    # DANGEROUS — deletes everything inside
kubectl label namespace <name> key=value
kubectl get all -n <namespace>
kubectl get all -A                 # All namespaces

# Determine where you are
kubectl config current-context
kubectl config view --minify | grep namespace
kubectl config view --minify -o jsonpath='{..namespace}'

# Context operations
kubectl config get-contexts
kubectl config use-context <name>
kubectl config set-context --current --namespace=<namespace>
kubectl config set-context <name> --cluster=<c> --user=<u> --namespace=<ns>
kubectl config delete-context <name>
kubectl config rename-context <old> <new>

# Safe delete workflow
kubectl get <resource> -n <namespace>           # See what's there
kubectl delete <resource> <name> -n <ns> --dry-run=client  # Preview
kubectl delete <resource> <name> -n <namespace>  # Execute

# Resource scope check
kubectl api-resources --namespaced=true   # Namespace-scoped resources
kubectl api-resources --namespaced=false  # Cluster-scoped resources

# Cross-namespace DNS
# <service>.<namespace>.svc.cluster.local
kubectl run dns-test --image=busybox:1.35 --rm -it --restart=Never \
  -n <namespace> -- nslookup <svc>.<target-ns>.svc.cluster.local
```

---

## Prod Wisdom — The Senior Engineer Mindset

Three things that define senior-level namespace and context discipline:

**1. They make their current context visible at all times.** The K8s context and namespace in the shell prompt is not a nice-to-have — it is a safety requirement. Every engineer who has accidentally deleted production resources was in the wrong context and didn't know it. If you can see `[prod-us-east|production]` in your prompt, you hesitate before running `kubectl delete`. If you can't see it, you don't hesitate — and that's when accidents happen.

**2. They use explicit `-n` in every script, every time.** Scripts that rely on the context's default namespace are fragile. Any engineer who changes their context before running the script hits the wrong target. Explicit `-n <namespace>` on every `kubectl` command in automation is the only safe pattern.

**3. They dry-run before every delete, and list before every mass operation.** `--dry-run=client` is free. It costs one extra command. In exchange, it shows you exactly what would be deleted before it happens. `kubectl get all -n <namespace>` before a mass delete shows you the blast radius. These two habits together have prevented more prod incidents than any amount of RBAC or admission control.

The namespace/context safety order:
```
kubectl config current-context     → Am I in the right cluster?
kubectl config view --minify | grep namespace  → Right namespace?
kubectl get all -n <target-ns>     → Right resources visible?
kubectl <operation> --dry-run=client  → Preview the change
kubectl <operation> -n <explicit-ns>  → Execute with explicit namespace
```

---

## Cleanup

```bash
# Clean up lab contexts
kubectl config delete-context lab12-dev-context 2>/dev/null || true
kubectl config delete-context lab12-staging-context 2>/dev/null || true
kubectl config delete-context lab12-prod-context 2>/dev/null || true

# Reset to kind-kind context
kubectl config use-context kind-kind

# Delete namespaces
kubectl delete namespace lab12-dev lab12-staging lab12-prod
```

---

*Lab 12 complete. Move to Lab 13 — HPA when ready.*