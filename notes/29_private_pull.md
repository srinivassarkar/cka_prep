# 🔐 Kubernetes Private Registry Authentication
### 

> *"Kubernetes doesn't pull images. It delegates. Understanding who delegates to whom — and how credentials travel — is the whole game."*

---

## 0. First Principles

The mental model that never changes:

```
YOU → kubectl apply → API Server → kubelet (on node) → CRI (containerd/CRI-O) → Registry
                                        ↑
                         Credentials must be HERE at pull time
```

**Three truths to tattoo on your brain:**

1. **Kubernetes doesn't pull images** — the Container Runtime Interface (CRI) does, on the node where the pod lands.
2. **Credentials must reach the kubelet** — either via the pod spec (`imagePullSecrets`) or via the pod's `ServiceAccount` (which the kubelet inspects).
3. **Secrets are namespace-scoped** — a secret in `default` is invisible in `app1-ns`. Always create secrets in the target namespace.

---

## 1. Reality Constraints

What Kubernetes actually does and doesn't do:

| Claim | Reality |
|---|---|
| "Kubernetes pulls images" | ❌ CRI (containerd, CRI-O) pulls images on the node |
| "One secret works everywhere" | ❌ Secrets are namespace-scoped — recreate per namespace |
| "default SA auto-gets pull secrets" | ❌ You must patch the default SA explicitly |
| "Custom SA inherits from default SA" | ❌ Custom SAs start clean — no pull secrets by default |
| "imagePullSecrets are mounted in the pod" | ❌ They are **only** used at pull time, never inside the container |
| "latest tag is fine in prod" | ❌ Tags are mutable; use semantic versioning (`v1.2.3`) |
| "docker-registry secret only works with Docker Hub" | ❌ Works with any registry using Docker v2 / token auth |

**Image identifier format:**
```
<registry>/<namespace>/<repository>:<tag>
```

```
docker.io/cloudwithvarjosh/app1:1.0.0
    │           │              │    │
    registry    namespace      repo tag
```

**Kubernetes defaulting behavior:**
```yaml
image: nginx
# Interpreted as: docker.io/library/nginx:latest
```

**Vendor exceptions that break the standard format:**

| Vendor | Example |
|---|---|
| Amazon ECR | `123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:v1` |
| Google Artifact Registry | `us-central1-docker.pkg.dev/project/repo/image:v1` |
| Harbor (nested) | `harbor.mycompany.com/dev/backend/app:v1` |

---

## 2. Decision Logic

### When to use which method

```
Do you need private image pull credentials?
├── NO → Public image, no action needed
└── YES → Where do you need them?
    ├── One specific Pod → imagePullSecrets in pod spec (Demo 1)
    ├── All pods in a namespace (same registry) → Patch default SA (Demo 2)
    └── Isolated workloads with separate identities → Custom ServiceAccount (Demo 3)
```

### Method comparison table

| Method | Scope | Effort | Best For |
|---|---|---|---|
| `imagePullSecrets` in pod spec | Per Pod | Per manifest | Quick testing, one-off pods |
| Patch default ServiceAccount | Namespace-wide | Once per namespace | Shared namespace, same registry |
| Custom ServiceAccount + pull secret | Per workload | Per SA | Prod isolation, RBAC boundaries |
| Workload Identity (IRSA, Workload Identity) | Cluster/Cloud | IAM setup | EKS/GKE/AKS — no static secrets |

### Secret type mapping

| Secret Type | Used For |
|---|---|
| `kubernetes.io/dockerconfigjson` | Registry credentials (imagePullSecrets) |
| `kubernetes.io/service-account-token` | API server authentication (in-cluster) |
| `Opaque` | Generic secrets (env vars, files) |

---

## 3. Internal Working

### How a private image gets pulled — step by step

```
1. User runs: kubectl apply -f pod.yaml
        │
2. API Server validates the manifest and writes pod spec to etcd
        │
3. Scheduler assigns the pod to a node
        │
4. Kubelet on that node watches for new pod assignments
        │
5. Kubelet reads pod.spec.imagePullSecrets
   OR reads pod.spec.serviceAccountName → fetches that SA → reads SA.imagePullSecrets
        │
6. Kubelet passes image name + credentials to CRI (containerd/CRI-O)
        │
7. CRI authenticates to the registry using the credentials
        │
8. CRI pulls the image layers to the node's local image store
        │
9. CRI creates and starts the container
        │
10. Pod transitions: Pending → ContainerCreating → Running
```

### What's inside a docker-registry secret

```json
{
  "auths": {
    "https://index.docker.io/v1/": {
      "username": "cloudwithvarjosh",
      "password": "YOUR_PASSWORD",
      "email": "you@example.com",
      "auth": "<base64(username:password)>"
    }
  }
}
```

This is stored base64-encoded under `.dockerconfigjson` in the Secret's `data` field. It mirrors `~/.docker/config.json` on your local machine.

### ServiceAccount token vs imagePullSecrets — they're different things

```
ServiceAccount
├── .token          → JWT for in-cluster API auth (mounted at /var/run/secrets/kubernetes.io/serviceaccount/)
└── .imagePullSecrets → Registry credentials (passed to CRI at pull time, never mounted)
```

If you want to prevent the SA token from being mounted (hardened pods):
```yaml
spec:
  automountServiceAccountToken: false
```

---

## 4. Hands-On

### Demo 0: Build and push a private image

```bash
# Build with semantic version tag (not latest!)
docker build -t cloudwithvarjosh/app1:1.0.0 .

# Login if on Linux
docker login

# Push to Docker Hub
docker push cloudwithvarjosh/app1:1.0.0
```

Then go to Docker Hub → Settings → Visibility → **Private**.

**The Dockerfile:**
```dockerfile
FROM python:3.9-slim
WORKDIR /app
ADD app.py app.py
RUN pip install flask
EXPOSE 5000
CMD ["python", "app.py"]
```

**The app:**
```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def home():
    return "Hello, Docker!"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

---

### Demo 1: Pod-level imagePullSecrets

**Step 1: Create the docker-registry secret**
```bash
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=cloudwithvarjosh \
  --docker-password=YOUR_PASSWORD \
  --docker-email=you@example.com \
  -n default
```

**Step 2: Reference it in the pod spec**
```yaml
# 01-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app1-pod1
spec:
  containers:
    - name: python-cont
      image: docker.io/cloudwithvarjosh/app1:1.0.0
  imagePullSecrets:
    - name: dockerhub-secret
```

```bash
kubectl apply -f 01-pod.yaml
kubectl get pod app1-pod1
```

**Step 3: Inspect and decode the secret (handle with care)**
```bash
# View the raw secret
kubectl get secret dockerhub-secret -o yaml

# Decode the .dockerconfigjson value
kubectl get secret dockerhub-secret \
  -o jsonpath='{.data.\.dockerconfigjson}' | base64 --decode | jq .
```

---

### Demo 2: Namespace-wide via patching default ServiceAccount

```bash
# Create namespace
kubectl create ns app1-ns

# Create the pull secret IN THE TARGET NAMESPACE
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=cloudwithvarjosh \
  --docker-password=YOUR_PASSWORD \
  --docker-email=you@example.com \
  -n app1-ns

# Patch the default SA (Option A: CLI)
kubectl patch serviceaccount default \
  -n app1-ns \
  -p '{"imagePullSecrets": [{"name": "dockerhub-secret"}]}'

# Patch the default SA (Option B: edit)
kubectl edit serviceaccount default -n app1-ns
# Add under spec:
# imagePullSecrets:
# - name: dockerhub-secret

# Verify
kubectl get serviceaccount default -n app1-ns -o yaml
```

**Pod manifest — no imagePullSecrets needed:**
```yaml
# 02-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app1-pod2
  namespace: app1-ns
spec:
  containers:
    - name: python-cont
      image: docker.io/cloudwithvarjosh/app1:1.0.0
  # No imagePullSecrets! Inherited from default SA.
```

```bash
kubectl apply -f 02-pod.yaml
kubectl get pod -n app1-ns
```

---

### Demo 3: Custom ServiceAccount with pull secrets

```bash
# Create custom SA
kubectl create serviceaccount app1-sa -n app1-ns

# Attach pull secret to the custom SA
kubectl patch serviceaccount app1-sa \
  -n app1-ns \
  -p '{"imagePullSecrets": [{"name": "dockerhub-secret"}]}'

# Verify
kubectl get sa app1-sa -n app1-ns -o yaml
```

**Pod manifest with custom SA:**
```yaml
# 03-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app1-pod3
  namespace: app1-ns
spec:
  serviceAccountName: app1-sa   # ← explicitly reference custom SA
  containers:
    - name: python-cont
      image: docker.io/cloudwithvarjosh/app1:1.0.0
```

```bash
kubectl apply -f 03-pod.yaml
kubectl get pod app1-pod3 -n app1-ns
```

---

### ECR / GCR / Azure equivalents

```bash
# Amazon ECR (static token — rotates every 12h, prefer IRSA in prod)
aws ecr get-login-password --region us-east-1 | \
  kubectl create secret docker-registry ecr-secret \
  --docker-server=123456789012.dkr.ecr.us-east-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region us-east-1) \
  -n my-namespace

# GitHub Container Registry
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=YOUR_GITHUB_USERNAME \
  --docker-password=YOUR_GITHUB_PAT \
  -n my-namespace

# Google Artifact Registry (using service account key)
kubectl create secret docker-registry gar-secret \
  --docker-server=us-central1-docker.pkg.dev \
  --docker-username=_json_key \
  --docker-password="$(cat sa-key.json)" \
  -n my-namespace
```

---

## 5. Production Flow

### Real-world architecture

```
┌─────────────────────────────────────────────────────┐
│                   Production Namespace               │
│                                                      │
│  ┌──────────┐   imagePullSecrets   ┌──────────────┐ │
│  │ backend  │ ←─────────────────── │  backend-sa  │ │
│  │   pod    │                      │  (custom SA) │ │
│  └──────────┘                      └──────┬───────┘ │
│                                           │         │
│  ┌──────────┐   imagePullSecrets          │         │
│  │ frontend │ ←───────────────────────────┘         │
│  │   pod    │  (same SA, different pods)            │
│  └──────────┘                                       │
│                                                      │
│  Secret: ecr-secret (in this namespace only)        │
└─────────────────────────────────────────────────────┘
```

### Design patterns

**Pattern 1: Per-team namespace isolation**
```
team-a-ns:   ecr-secret + team-a-sa  → team-a images
team-b-ns:   ecr-secret + team-b-sa  → team-b images
# Secrets duplicated per namespace — by design
```

**Pattern 2: Cloud workload identity (preferred for EKS/GKE/AKS)**
```
EKS:  IRSA (IAM Roles for Service Accounts) → no static secrets
GKE:  Workload Identity → no static secrets
AKS:  Pod Identity / Federated Credentials → no static secrets
```
No secrets = no rotation = no leakage risk.

**Pattern 3: External secret management**
```
HashiCorp Vault / AWS Secrets Manager / Sealed Secrets
      → External Secrets Operator → k8s Secret (auto-refreshed)
```

### CI/CD integration checklist

- [ ] Secret created in every target namespace (dev, staging, prod)
- [ ] SA patched or pod spec updated before deployment
- [ ] Secrets rotated on cadence (or using workload identity)
- [ ] RBAC: restrict `get`/`list` on secrets to only what's needed
- [ ] Never print secret values in pipeline logs

---

## 6. Mistakes

### Mistake 1: Secret in wrong namespace

**Symptom:** `ImagePullBackOff` even though the secret exists  
**Root cause:** Secret created in `default`, pod in `app1-ns` — secrets don't cross namespaces  
**Fix:**
```bash
kubectl get secret dockerhub-secret -n app1-ns   # Should exist here
kubectl create secret docker-registry dockerhub-secret ... -n app1-ns
```

---

### Mistake 2: Forgot to patch the SA after creating the secret

**Symptom:** Pod without explicit `imagePullSecrets` fails even though SA exists  
**Root cause:** SA exists but has no `imagePullSecrets` field attached  
**Fix:**
```bash
kubectl get sa default -n app1-ns -o yaml | grep imagePullSecrets
# If nothing → patch it
kubectl patch sa default -n app1-ns -p '{"imagePullSecrets": [{"name": "dockerhub-secret"}]}'
```

---

### Mistake 3: Custom SA but no `serviceAccountName` in pod spec

**Symptom:** Pod uses `default` SA instead of custom SA, falls back to `ImagePullBackOff`  
**Root cause:** `serviceAccountName` field omitted — Kubernetes defaults to `default` SA  
**Fix:**
```yaml
spec:
  serviceAccountName: app1-sa   # ← this is required, not optional
```

---

### Mistake 4: Stale ECR token

**Symptom:** Worked yesterday, fails today with `401 Unauthorized`  
**Root cause:** ECR login tokens expire after 12 hours  
**Fix (short-term):** Regenerate the secret  
**Fix (real):** Use IRSA — no token to rotate

---

### Mistake 5: Image tagged `latest` causing unexpected behavior

**Symptom:** Deploy "succeeds" but runs old code  
**Root cause:** `latest` tag was overwritten; node cached the old image  
**Fix:**
```yaml
image: docker.io/myorg/app:v1.2.3   # Always pin to immutable tag
```
```bash
# Force re-pull (not recommended in prod, use versioning instead)
kubectl delete pod <pod> && kubectl apply -f pod.yaml
```

---

### Mistake 6: Using `WidthType.PERCENTAGE` in secret decode scripts 😄

Kidding — but seriously: **never decode and log secrets in CI pipelines.** Set `DOCKER_CONFIG` env var or use workload identity instead.

---

## 7. Interview Answers

**Q: What happens when Kubernetes tries to pull a private image without credentials?**

> "When a pod references a private image without providing pull credentials, the kubelet passes the image reference to the container runtime — typically containerd — which attempts an unauthenticated pull. The registry returns a 401 Unauthorized response. The kubelet retries with exponential backoff, and the pod surfaces this as `ErrImagePull` initially, transitioning to `ImagePullBackOff` as retries accumulate. The event log from `kubectl describe pod` will show the exact registry error message, often `pull access denied` or `insufficient_scope: authorization failed`."

---

**Q: Who actually pulls container images in Kubernetes, and how do credentials get to them?**

> "Kubernetes itself doesn't pull images — that responsibility belongs to the container runtime, which is typically containerd or CRI-O, running on the worker node. The kubelet acts as the intermediary. When it schedules a container, it reads the pod's `imagePullSecrets` directly from the pod spec, or it resolves the pod's ServiceAccount and reads `imagePullSecrets` from there. It then passes those credentials to the CRI, which uses them to authenticate to the registry. The credentials never get mounted into the container — they're only consumed at pull time."

---

**Q: What are the three ways to provide pull credentials, and when would you use each?**

> "First, you can add `imagePullSecrets` directly in the pod spec — this is the most explicit and granular approach, but it requires updating every pod manifest. Second, you can attach the secret to the namespace's default ServiceAccount using `kubectl patch` — this is cleaner for a namespace where all workloads use the same registry, because every pod that uses the default SA inherits the credentials automatically. Third, you can create a custom ServiceAccount, attach the secret to it, and reference that SA in specific pods using `serviceAccountName` — this is the production pattern because it enforces workload isolation and maps cleanly to RBAC policies. In cloud environments, a fourth option is workload identity — IRSA on EKS, Workload Identity on GKE — which eliminates static secrets entirely."

---

**Q: Do custom ServiceAccounts inherit imagePullSecrets from the default SA?**

> "No, they don't. Each ServiceAccount in Kubernetes is independent. Custom ServiceAccounts start completely clean — no imagePullSecrets, no automatically inherited credentials. You must explicitly patch or edit each custom SA to attach pull secrets. This is actually by design: it forces you to be deliberate about which workloads have access to which registries."

---

**Q: Are secrets shared across namespaces?**

> "No. Secrets in Kubernetes are namespace-scoped resources. A secret created in the `default` namespace is completely invisible to pods in `app1-ns`. If you need the same registry credentials in multiple namespaces, you must create the secret in each one. This is a common gotcha — people create the secret once and wonder why pods in other namespaces still fail with `ImagePullBackOff`. Tools like the External Secrets Operator can help automate this synchronization across namespaces."

---

**Q: How would you handle private registry auth at scale in a production EKS cluster?**

> "At scale, static secrets are a liability — they expire, they drift, and someone eventually leaks them. The production answer for EKS is IRSA, IAM Roles for Service Accounts. You annotate a Kubernetes ServiceAccount with an IAM role ARN, and pods using that SA automatically receive short-lived AWS credentials via the token projection mechanism. The kubelet uses those credentials to call `ecr:GetAuthorizationToken` and pull images without a single static secret in etcd. For non-cloud or multi-cloud environments, I'd integrate an External Secrets Operator with Vault or AWS Secrets Manager to dynamically inject and rotate secrets."

---

## 8. Debugging

### Fast diagnosis path for `ImagePullBackOff`

```
kubectl describe pod <pod-name> -n <namespace>
         │
         └── Look at Events section
                    │
         ┌──────────┴────────────────────────────────────────┐
         │                                                    │
  "pull access denied"                              "not found" / 404
  "insufficient_scope"                                    │
  "401 Unauthorized"                             Image doesn't exist
         │                                       or tag is wrong
         ▼                                       → Fix the image path
  Credential problem
         │
         ├── Does the secret exist in the pod's namespace?
         │       kubectl get secret -n <namespace>
         │       └── NO → Create it here
         │
         ├── Is the secret attached to the SA or pod spec?
         │       kubectl get sa <sa-name> -n <ns> -o yaml | grep imagePullSecrets
         │       kubectl get pod <pod> -n <ns> -o yaml | grep imagePullSecrets
         │       └── NO → Patch the SA or update pod spec
         │
         ├── Are the credentials in the secret correct?
         │       kubectl get secret <secret> -n <ns> \
         │         -o jsonpath='{.data.\.dockerconfigjson}' | base64 --decode | jq .
         │       └── Expired / wrong → Recreate the secret
         │
         └── Is the pod using the right SA?
                 kubectl get pod <pod> -o yaml | grep serviceAccountName
                 └── Wrong SA → Update pod spec
```

### Command reference for live debugging

```bash
# 1. What's the pod status + events?
kubectl describe pod <pod> -n <ns>

# 2. What SA is the pod using?
kubectl get pod <pod> -n <ns> -o jsonpath='{.spec.serviceAccountName}'

# 3. What imagePullSecrets does that SA have?
kubectl get sa <sa-name> -n <ns> -o yaml

# 4. Does the secret exist in this namespace?
kubectl get secret -n <ns> | grep docker

# 5. Are the credentials valid? (decode and inspect)
kubectl get secret <secret> -n <ns> \
  -o jsonpath='{.data.\.dockerconfigjson}' | base64 --decode | jq .

# 6. Test the credentials manually (from your laptop)
docker login https://index.docker.io/v1/ -u <username> -p <password>
docker pull docker.io/<namespace>/<repo>:<tag>

# 7. Check kubelet logs on the node (if you have node access)
journalctl -u kubelet | grep -i "image\|pull\|auth" | tail -50
```

---

## 9. Kill Switch

**10-second recall — absolute minimum:**

```
Private image → needs imagePullSecrets
                │
                ├── Pod spec: imagePullSecrets: [{name: <secret>}]
                ├── SA: patch SA → imagePullSecrets, then ref SA in pod
                └── Secret is namespace-scoped → recreate per namespace

Secret type: kubernetes.io/dockerconfigjson
Create:      kubectl create secret docker-registry <name> \
               --docker-server= --docker-username= --docker-password=

ImagePullBackOff → describe pod → check events → secret in right NS? SA patched? Creds valid?
```

---

## 10. Appendix

### Quick reference: secret creation

```bash
# Docker Hub
kubectl create secret docker-registry <name> \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<user> \
  --docker-password=<pass> \
  --docker-email=<email> \
  -n <namespace>

# Amazon ECR
kubectl create secret docker-registry ecr-secret \
  --docker-server=<account>.dkr.ecr.<region>.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region <region>) \
  -n <namespace>

# GitHub Container Registry
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=<github-user> \
  --docker-password=<pat> \
  -n <namespace>
```

### Quick reference: SA patching

```bash
# Patch via CLI (fastest)
kubectl patch sa <sa-name> -n <ns> \
  -p '{"imagePullSecrets": [{"name": "<secret-name>"}]}'

# Multiple secrets
kubectl patch sa <sa-name> -n <ns> \
  -p '{"imagePullSecrets": [{"name": "secret1"}, {"name": "secret2"}]}'

# Verify
kubectl get sa <sa-name> -n <ns> -o yaml | grep -A5 imagePullSecrets
```

### Quick reference: registry formats

| Registry | Server URL | Username |
|---|---|---|
| Docker Hub | `https://index.docker.io/v1/` | your username |
| Amazon ECR | `<account>.dkr.ecr.<region>.amazonaws.com` | `AWS` |
| Google Artifact Registry | `<region>-docker.pkg.dev` | `_json_key` |
| GitHub Container Registry | `ghcr.io` | your username |
| Azure Container Registry | `<name>.azurecr.io` | service principal |
| Harbor | `harbor.<your-domain>` | your username |

### Pod spec: imagePullSecrets cheatsheet

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
  namespace: app1-ns
spec:
  serviceAccountName: app1-sa          # Optional: use custom SA
  automountServiceAccountToken: false  # Optional: harden if no API access needed
  containers:
    - name: app
      image: docker.io/myorg/app:v1.2.3
  imagePullSecrets:                    # Explicit per-pod credentials
    - name: dockerhub-secret
    - name: ecr-secret                 # Multiple registries supported
```

### Secret decode one-liner

```bash
kubectl get secret <secret> -n <ns> \
  -o jsonpath='{.data.\.dockerconfigjson}' \
  | base64 --decode | jq .auths
```

### Namespace-wide setup: full sequence

```bash
kubectl create ns <ns>
kubectl create secret docker-registry <secret> \
  --docker-server=<server> --docker-username=<user> \
  --docker-password=<pass> -n <ns>
kubectl patch sa default -n <ns> \
  -p '{"imagePullSecrets": [{"name": "<secret>"}]}'
kubectl get sa default -n <ns> -o yaml   # Verify
```

---
