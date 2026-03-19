# 🧠 ConfigMap & Secret : CORE MENTAL MODEL (Never Forget This)

```
ConfigMap = non-secret config  →  env vars / files / args
Secret     = sensitive config  →  env vars / files / args (base64, not encrypted!)
```

👉 Both decouple config from code. Same injection patterns. Different trust level.

```
App needs config?
  → sensitive?   YES → Secret
  → sensitive?   NO  → ConfigMap
```

---

# 🔥 1. CONFIGMAP (The What & Why)

### ❌ Without ConfigMap (hardcoded — DON'T DO THIS IN PROD)

```yaml
env:
  - name: APP
    value: frontend
  - name: ENVIRONMENT
    value: production
```

👉 Problem: change config = redeploy. Not scalable across environments.

---

### ✅ With ConfigMap (real world)

Config lives outside the pod spec. Change it without touching the Deployment.

---

# ⚡ 2. THREE WAYS TO INJECT CONFIG (Know All Three)

```
1. env vars      → fast, static after pod start
2. volume mount  → files, dynamic updates possible
3. CLI args      → rare, don't focus on this
```

---

# ⚡ 3. envFrom — BULK INJECT (CKA Speed Trick)

Instead of mapping keys one by one, inject ALL keys at once:

```yaml
envFrom:
- configMapRef:
    name: frontend-cm          # ALL keys from ConfigMap → env vars

- secretRef:
    name: frontend-secret      # ALL keys from Secret → env vars
```

👉 **CKA use:** Saves time when task says "inject all config into the pod."

👉 **Prod caveat:** Less explicit — all 20 keys of a ConfigMap land in the container. Know the tradeoff vs per-key `valueFrom`.

---

# 🔥 4. SECRET (Critical Distinction)

### 🧠 Encoding ≠ Encryption (INTERVIEW TRAP)

| | Encoding (base64) | Encryption (AES/TLS) |
|---|---|---|
| Reversible | Yes, instantly | Yes, only with key |
| Secure | ❌ NO | ✅ YES |
| Kubernetes default | ✅ Secrets use this | ❌ Not by default |

👉 **Interview line:**
> "Kubernetes Secrets are base64-encoded, NOT encrypted. etcd stores them as plain text unless encryption at rest is explicitly enabled."

---

# ⚡ 5. SECRET TYPES (Interview Differentiator)

```
Opaque                              → generic, user-defined (default)
kubernetes.io/tls                   → TLS cert + key (ingress, mTLS)
kubernetes.io/dockerconfigjson      → private registry image pull
kubernetes.io/service-account-token → SA tokens (auto-created by k8s)
```

```bash
# TLS secret — very common in prod for ingress HTTPS
kubectl create secret tls my-tls-secret \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key

# Image pull secret — private registry auth
kubectl create secret docker-registry my-registry-secret \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass \
  --docker-email=user@example.com
```

👉 **Interview line:**
> "Opaque is for arbitrary data. In prod I use `kubernetes.io/tls` for ingress TLS termination and `docker-registry` type for private image pulls."

---

# ⚡ 6. IMMUTABLE ConfigMaps & Secrets (CKA Tested)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
immutable: true      # 👈 cannot update data after this — must delete + recreate
data:
  db_host: mysql-service
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
immutable: true
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQ=
```

```bash
# Check if immutable
kubectl describe configmap app-config | grep Immutable
```

👉 **Why it matters in prod:** Kubernetes stops watching immutable ConfigMaps for changes → reduces API server load at scale. Use for config that must never drift (DB hostnames, fixed feature flags).

👉 **Interview line:**
> "Immutable ConfigMaps prevent accidental config drift and reduce kube-apiserver load at scale — important in clusters with thousands of pods."

---

# 🚨 REAL OUTAGE / PROD RULES

### ❗ Env var from ConfigMap/Secret → NOT live updated

```
Pod running → update ConfigMap → env var inside pod UNCHANGED
```

👉 Must restart pod to pick up change.

---

### ✅ Volume mount (no subPath) → LIVE updated

```
Pod running → update ConfigMap → file inside pod UPDATES automatically (~60s)
```

👉 App must re-read the file (hot reload). Kubernetes uses symlinks internally.

---

### ❗ Volume mount WITH subPath → NOT live updated

```
subPath = file is COPIED not symlinked → no dynamic updates
```

---

### 🧠 Update Behavior Cheat Sheet

```
env var (configMapKeyRef / secretKeyRef)  → static, set at pod start
envFrom (configMapRef / secretRef)        → static, set at pod start
volume mount, no subPath                  → dynamic ✅ (~60s delay)
volume mount, with subPath                → static ❌ (copied, not symlinked)
immutable: true                           → static forever ❌ (by design)
```

---

# ⚡ 7. VOLUME MOUNT GOTCHA (Very Common Bug)

### ❌ Wrong — trying to mount directory OVER a file path

```yaml
mountPath: /usr/share/nginx/html/index.html  # FAILS — can't mount dir over file
```

### ✅ Correct — mount the DIRECTORY

```yaml
mountPath: /usr/share/nginx/html   # mounts ALL configmap keys as files here
```

### ✅ Correct — mount SINGLE file with subPath

```yaml
mountPath: /usr/share/nginx/html/index.html
subPath: index.html   # file-over-file = OK. But no live updates.
```

---

### Without `items:` → ALL keys mount as files

```
/usr/share/nginx/html/APP
/usr/share/nginx/html/ENVIRONMENT
/usr/share/nginx/html/index.html
```

### With `items:` → ONLY listed keys mount

```yaml
items:
  - key: index.html
    path: index.html
```

---

# ⚡ 8. RBAC ON SECRETS (Interview Must-Know)

```bash
# Who can read secrets in a namespace? (audit question in prod)
kubectl get rolebindings,clusterrolebindings -A | grep secret

# Check what a specific SA can do
kubectl auth can-i get secrets --as=system:serviceaccount:production:app-sa -n production
```

```yaml
# Role — allow only get/list on secrets in one namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
```

```yaml
# Bind the Role to a specific ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: secret-reader-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: production
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
# Imperative shortcut (faster in CKA)
kubectl create role secret-reader \
  --verb=get,list --resource=secrets -n production

kubectl create rolebinding secret-reader-binding \
  --role=secret-reader \
  --serviceaccount=production:app-sa -n production
```

👉 **Interview line:**
> "Secrets are only as secure as your RBAC. By default, any pod in the namespace can read all secrets. In prod, we scope secret access per ServiceAccount using Roles — not ClusterRoles — to follow least privilege."

---

# 🎯 CKA + INTERVIEW HIT QUESTIONS

### Q: ConfigMap vs Secret?
👉 ConfigMap = non-sensitive. Secret = sensitive + base64 encoded.

### Q: Are Secrets secure by default?
👉 NO. base64 only. Need encryption at rest + RBAC to be actually secure.

### Q: When does a pod pick up ConfigMap changes?
👉 Env vars: never (until restart). Volume mount without subPath: dynamically (~60s).

### Q: subPath behavior?
👉 File copied, not symlinked. No live updates even if ConfigMap changes.

### Q: Pod fails to start after ConfigMap volume mount?
👉 Check if mountPath is a file path — must be a directory. Use subPath for file-level precision.

### Q: What is an immutable ConfigMap?
👉 `immutable: true` — data cannot be changed. Must delete + recreate. Reduces API server load at scale.

### Q: Secret types you know?
👉 Opaque (generic), tls (ingress), dockerconfigjson (private registry), service-account-token (auto SA).

### Q: How to restrict which pods can read a Secret?
👉 RBAC — create Role with get/list on secrets, bind to a specific ServiceAccount. Don't use ClusterRole for this.

### Q: envFrom vs env.valueFrom?
👉 `envFrom` = all keys at once (faster, less explicit). `valueFrom` = per key (explicit, safer in prod).

### Q: How to create a ConfigMap from a file?
```bash
kubectl create configmap nginx-config --from-file=nginx.conf
```

### Q: How to create a Secret without base64 encoding manually?
```bash
kubectl create secret generic db-creds \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=secret
# kubectl handles encoding automatically
```

---

# 🧠 DEBUGGING CHEAT SHEET (OUTAGE MODE)

```bash
# Check ConfigMap exists and has the right keys
kubectl get cm <name> -o yaml

# Check Secret (values will be base64)
kubectl get secret <name> -o yaml

# Decode a secret value inline
kubectl get secret frontend-secret -o jsonpath='{.data.DB_USER}' | base64 --decode

# Verify env vars inside pod
kubectl exec -it <pod> -- printenv | grep -E 'APP|DB_USER'

# Verify volume mounted file
kubectl exec -it <pod> -- cat /etc/secrets/DB_USER
kubectl exec -it <pod> -- cat /usr/share/nginx/html/index.html

# Pod not starting? Check events
kubectl describe pod <pod>

# Check if ConfigMap is immutable
kubectl describe configmap <name> | grep Immutable

# Check RBAC — can this SA read secrets?
kubectl auth can-i get secrets \
  --as=system:serviceaccount:<namespace>:<sa-name> -n <namespace>
```

---

# 🔥 LABS (DO THIS — NO SKIP)

---

## 🧪 LAB 1: ConfigMap as Environment Variables (per-key)

### Step 1: Create the ConfigMap

```yaml
# frontend-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-cm
data:
  APP: "frontend"
  ENVIRONMENT: "production"
```

```bash
kubectl apply -f frontend-cm.yaml
# ⚠️ Always apply ConfigMap BEFORE the Deployment
```

---

### Step 2: Deployment — per-key injection

```yaml
# frontend-deploy-env.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend-container
        image: nginx
        env:
        - name: APP
          valueFrom:
            configMapKeyRef:
              name: frontend-cm
              key: APP
        - name: ENVIRONMENT
          valueFrom:
            configMapKeyRef:
              name: frontend-cm
              key: ENVIRONMENT
```

```bash
kubectl apply -f frontend-deploy-env.yaml
```

---

### Step 3: Deployment — bulk injection with envFrom (CKA speed)

```yaml
# frontend-deploy-envfrom.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend-container
        image: nginx
        envFrom:
        - configMapRef:
            name: frontend-cm     # ALL keys injected as env vars at once
```

```bash
kubectl apply -f frontend-deploy-envfrom.yaml
```

---

### Step 4: Verify

```bash
kubectl get pods -l app=frontend
kubectl exec -it <pod-name> -- printenv | grep -E 'APP|ENVIRONMENT'
# APP=frontend
# ENVIRONMENT=production
```

---

### Step 5: Test static behavior (prod awareness)

```bash
kubectl edit cm frontend-cm        # change ENVIRONMENT to staging
kubectl exec -it <pod-name> -- printenv | grep ENVIRONMENT
# Still: ENVIRONMENT=production ❌ — env vars are static

kubectl rollout restart deployment frontend-deploy
kubectl exec -it <new-pod-name> -- printenv | grep ENVIRONMENT
# Now: ENVIRONMENT=staging ✅
```

---

## 🧪 LAB 2: ConfigMap from File (--from-file)

```bash
# Create a real nginx config file
cat > nginx.conf << 'EOF'
server {
  listen 80;
  server_name localhost;
  location / {
    root /usr/share/nginx/html;
    index index.html;
  }
}
EOF

# Create ConfigMap from file — key = filename, value = file contents
kubectl create configmap nginx-config --from-file=nginx.conf

# Verify
kubectl get cm nginx-config -o yaml
```

```bash
# From a .env style file (KEY=VALUE per line)
cat > app.env << 'EOF'
APP=frontend
ENVIRONMENT=production
DB_HOST=mysql-service
EOF

kubectl create configmap app-config --from-env-file=app.env

# Verify
kubectl get cm app-config -o yaml
```

---

## 🧪 LAB 3: ConfigMap as Volume Mount (with HTML file)

### Step 1: ConfigMap with embedded file

```yaml
# frontend-cm-with-html.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-cm
data:
  APP: "frontend"
  ENVIRONMENT: "production"
  index.html: |
    <!DOCTYPE html>
    <html>
    <head><title>Welcome</title></head>
    <body>
      <h1>This page is served by nginx. Welcome to Cloud With VarJosh!!</h1>
    </body>
    </html>
```

```bash
kubectl apply -f frontend-cm-with-html.yaml
```

---

### Step 2: Deployment — directory mount (dynamic updates)

```yaml
# frontend-deploy-vol.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deploy
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
      - name: frontend-container
        image: nginx
        envFrom:
        - configMapRef:
            name: frontend-cm
        volumeMounts:
        - name: html-volume
          mountPath: /usr/share/nginx/html   # directory — only index.html mounts here
          readOnly: true
      volumes:
      - name: html-volume
        configMap:
          name: frontend-cm
          items:
          - key: index.html     # only this key, not APP or ENVIRONMENT
            path: index.html
```

```bash
kubectl apply -f frontend-deploy-vol.yaml
```

---

### Step 3: Alternative — subPath (single file, no live updates)

```yaml
        volumeMounts:
        - name: html-volume
          mountPath: /usr/share/nginx/html/index.html   # exact file
          subPath: index.html                           # file-over-file OK
      volumes:
      - name: html-volume
        configMap:
          name: frontend-cm
```

> ⚠️ subPath = no live updates. Directory mount = live updates (~60s).

---

### Step 4: NodePort Service

```yaml
# frontend-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 31000
```

```bash
kubectl apply -f frontend-svc.yaml
kubectl exec -it <pod-name> -- cat /usr/share/nginx/html/index.html
curl http://localhost:31000      # KIND
curl http://<Node-IP>:31000      # others
```

---

### Step 5: Test live update (prod awareness)

```bash
kubectl edit cm frontend-cm     # change H1 text in index.html
# Wait ~60 seconds
kubectl exec -it <pod-name> -- cat /usr/share/nginx/html/index.html
# File updated automatically ✅ (directory mount, no subPath)
```

---

## 🧪 LAB 4: Immutable ConfigMap

```yaml
# immutable-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
immutable: true
data:
  db_host: "mysql-service"
  db_port: "3306"
```

```bash
kubectl apply -f immutable-cm.yaml

# Try to update it — will FAIL
kubectl edit cm app-config
# Error: ConfigMap is immutable and cannot be updated

# Must delete and recreate to change
kubectl delete cm app-config
kubectl apply -f immutable-cm.yaml

# Verify
kubectl describe cm app-config | grep Immutable
# Immutable: true
```

---

## 🧪 LAB 5: Secrets as Environment Variables

### Step 1: Encode values

```bash
echo -n 'frontenduser' | base64   # ZnJvbnRlbmR1c2Vy
echo -n 'frontendpass' | base64   # ZnJvbnRlbmRwYXNz

# Decode to verify
echo 'ZnJvbnRlbmR1c2Vy' | base64 --decode   # frontenduser
```

---

### Step 2: Create Secret (declarative)

```yaml
# frontend-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: frontend-secret
type: Opaque
data:
  DB_USER: ZnJvbnRlbmR1c2Vy       # base64 of 'frontenduser'
  DB_PASSWORD: ZnJvbnRlbmRwYXNz   # base64 of 'frontendpass'
```

```bash
kubectl apply -f frontend-secret.yaml

# Or imperatively (kubectl handles encoding automatically — no manual base64)
kubectl create secret generic frontend-secret \
  --from-literal=DB_USER=frontenduser \
  --from-literal=DB_PASSWORD=frontendpass
```

---

### Step 3: Deployment — per-key Secret injection

```yaml
# frontend-deploy-secret-env.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deploy
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
        - name: frontend-container
          image: nginx
          env:
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: frontend-secret
                  key: DB_USER
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: frontend-secret
                  key: DB_PASSWORD
```

```bash
kubectl apply -f frontend-deploy-secret-env.yaml

# Verify — Kubernetes auto-decodes base64, pod sees plain text
kubectl exec -it <pod-name> -- printenv | grep DB
# DB_USER=frontenduser
# DB_PASSWORD=frontendpass
```

---

### Step 4: Bulk inject Secret with envFrom

```yaml
        envFrom:
        - secretRef:
            name: frontend-secret    # ALL secret keys → env vars at once
```

---

### Step 5: Decode a secret without exec (useful in outage)

```bash
kubectl get secret frontend-secret -o jsonpath='{.data.DB_USER}' | base64 --decode
# frontenduser
```

---

## 🧪 LAB 6: Secrets as Volume Mount (files)

```yaml
# frontend-deploy-secret-vol.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deploy
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
      volumes:
        - name: secret-volume
          secret:
            secretName: frontend-secret
            # No 'items' = ALL keys mount as individual files
      containers:
        - name: frontend-container
          image: nginx
          env:
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: frontend-secret
                  key: DB_USER
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: frontend-secret
                  key: DB_PASSWORD
          volumeMounts:
          - name: secret-volume
            mountPath: /etc/secrets     # each key = one file here
            readOnly: true
```

```bash
kubectl apply -f frontend-deploy-secret-vol.yaml

# Verify files
kubectl exec -it <pod-name> -- ls /etc/secrets
# DB_USER   DB_PASSWORD

kubectl exec -it <pod-name> -- cat /etc/secrets/DB_USER
# frontenduser ✅ (auto-decoded, plain text in file)
```

---

## 🧪 LAB 7: TLS Secret (prod pattern)

```bash
# Generate self-signed cert for testing
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=myapp.example.com/O=myapp"

# Create TLS secret
kubectl create secret tls my-tls-secret \
  --cert=tls.crt \
  --key=tls.key

# Verify
kubectl get secret my-tls-secret -o yaml
# type: kubernetes.io/tls
# data:
#   tls.crt: <base64>
#   tls.key: <base64>
```

---

## 🧪 LAB 8: RBAC on Secrets

```bash
# Create ServiceAccount
kubectl create serviceaccount app-sa -n production

# Create Role — only get/list secrets in production namespace
kubectl create role secret-reader \
  --verb=get,list \
  --resource=secrets \
  -n production

# Bind Role to ServiceAccount
kubectl create rolebinding secret-reader-binding \
  --role=secret-reader \
  --serviceaccount=production:app-sa \
  -n production

# Verify — can this SA read secrets?
kubectl auth can-i get secrets \
  --as=system:serviceaccount:production:app-sa -n production
# yes ✅

# Verify — can it delete secrets? (should be NO)
kubectl auth can-i delete secrets \
  --as=system:serviceaccount:production:app-sa -n production
# no ✅
```

---

## 🧪 BONUS: Backend deploy with Secret as volume

```yaml
# backend-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      volumes:
        - name: secret-volume
          secret:
            secretName: backend-secret
      containers:
        - name: backend-container
          image: hashicorp/http-echo
          args:
            - "-text=Hello from Backend"
          volumeMounts:
          - name: secret-volume
            mountPath: /etc/secrets
            readOnly: true
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 9090
      targetPort: 5678
  selector:
    app: backend
```

---

# 🧠 FINAL MEMORY LOCK

```
ConfigMap  = config, not sensitive
Secret     = sensitive, base64 ONLY (not encrypted by default!)

env var / envFrom      → static, needs pod restart to update
volume, no subPath     → dynamic ✅ (~60s)
volume, with subPath   → static ❌ (copied not symlinked)
immutable: true        → static forever, reduces API server load

Secret types:
  Opaque              → generic key-value
  tls                 → ingress HTTPS
  dockerconfigjson    → private registry
  service-account-token → SA (auto-managed)

RBAC:
  Use Role (not ClusterRole) for namespace-scoped secret access
  Bind to ServiceAccount, not to users directly
  Always verify with: kubectl auth can-i

Always apply ConfigMap/Secret BEFORE Deployment
Never store Secrets in Git
Enable etcd encryption at rest in prod
Use RBAC to restrict who can read Secrets
Use immutable: true for config that must never drift
```
