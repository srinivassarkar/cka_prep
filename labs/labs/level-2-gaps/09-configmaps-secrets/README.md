# Lab 09 — ConfigMaps & Secrets

## What This Lab Is About

ConfigMaps and Secrets are how configuration and sensitive data get into your pods. They look simple — create a key-value store, mount it into a pod. But the failure modes are subtle and the consequences are serious. A pod that starts with stale config because the mount didn't refresh. A secret that's base64 encoded but not encrypted. An app that silently uses default values because the ConfigMap mount failed without throwing an error.

This lab covers the 5 most critical ConfigMap and Secret failure modes in real clusters. A pod that doesn't pick up ConfigMap changes without a restart. Mounting failures when the key doesn't exist. The difference between `envFrom` and volume mounts and when each breaks. Secret data that's malformed because of incorrect base64. And the correct production pattern for managing sensitive config without leaking it.

> ConfigMaps and Secrets are not just storage — they are the contract between your platform and your application. Break the contract silently and your app runs with wrong config. Break it loudly and your pod never starts.

---

## Environment

- **Cluster:** KIND (Kubernetes IN Docker)
- **K8s Version:** v1.27.3
- **Node:** Single node (`kind-control-plane`)
- **Namespace for this lab:** `lab09`

```bash
kubectl create namespace lab09
kubectl config set-context --current --namespace=lab09
```

---

## The 5 Scenarios

| # | Scenario | What You'll Learn |
|---|---|---|
| 01 | ConfigMap update not reflected in pod | Env var vs volume mount refresh behaviour |
| 02 | Missing key causes pod failure | `optional` flag, partial mounts, key projection |
| 03 | Wrong injection method breaks the app | `envFrom` vs `env.valueFrom` vs volume mount |
| 04 | Malformed Secret — bad base64 | How Secrets are stored, decoded, and broken |
| 05 | Production pattern — immutable configs and secret rotation | Safe config management in real clusters |

---

## Scenario 01 — ConfigMap Update Not Reflected in Pod

### What You'll Break

One of the most common prod surprises — you update a ConfigMap, check the pod logs, and the old value is still there. The app is running stale config. This happens because environment variables sourced from ConfigMaps are **baked in at pod start time** — they never update without a pod restart. Volume-mounted ConfigMaps DO update automatically, but with a delay. Knowing which is which determines whether your config change takes effect.

### Apply the Setup

```bash
# Create a ConfigMap with initial config
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: lab09
data:
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "10"
  FEATURE_FLAG: "false"
  config.yaml: |
    log_level: info
    max_connections: 10
    feature_flag: false
EOF

# Deploy a pod using ConfigMap in TWO ways simultaneously
# Method A: env vars (won't auto-update)
# Method B: volume mount (will auto-update with delay)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: config-app
  namespace: lab09
spec:
  replicas: 1
  selector:
    matchLabels:
      app: config-app
  template:
    metadata:
      labels:
        app: config-app
    spec:
      containers:
      - name: app
        image: busybox:1.35
        command: ["sh", "-c"]
        args:
        - |
          while true; do
            echo "=== $(date) ==="
            echo "[ENV VAR]    LOG_LEVEL = ${LOG_LEVEL}"
            echo "[ENV VAR]    FEATURE_FLAG = ${FEATURE_FLAG}"
            echo "[VOLUME]     config.yaml content:"
            cat /etc/config/config.yaml
            echo "---"
            sleep 15
          done
        # Method A — env vars from ConfigMap
        envFrom:
        - configMapRef:
            name: app-config
        # Method B — volume mount from ConfigMap
        volumeMounts:
        - name: config-volume
          mountPath: /etc/config
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "64Mi"
            cpu: "100m"
      volumes:
      - name: config-volume
        configMap:
          name: app-config
EOF
```

```bash
kubectl rollout status deployment/config-app -n lab09

# See current values
kubectl logs $(kubectl get pod -n lab09 -l app=config-app \
  -o jsonpath='{.items[0].metadata.name}') -n lab09 --tail=10
# [ENV VAR]    LOG_LEVEL = info
# [ENV VAR]    FEATURE_FLAG = false
# [VOLUME]     log_level: info
#              feature_flag: false
```

### Break It — Update ConfigMap, Observe the Gap

```bash
# Update the ConfigMap — change LOG_LEVEL and FEATURE_FLAG
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: lab09
data:
  LOG_LEVEL: "debug"            # Changed
  MAX_CONNECTIONS: "50"         # Changed
  FEATURE_FLAG: "true"          # Changed
  config.yaml: |
    log_level: debug
    max_connections: 50
    feature_flag: true
EOF

# Wait 30-60 seconds then check logs
sleep 60
kubectl logs $(kubectl get pod -n lab09 -l app=config-app \
  -o jsonpath='{.items[0].metadata.name}') -n lab09 --tail=15
```

You will observe:
```
[ENV VAR]    LOG_LEVEL = info          ← STALE: still shows old value
[ENV VAR]    FEATURE_FLAG = false      ← STALE: env vars never update
[VOLUME]     log_level: debug          ← UPDATED: volume reflects new value
             feature_flag: true        ← UPDATED: volume auto-refreshed
```

### Investigate

```bash
# Confirm the ConfigMap itself is updated
kubectl get configmap app-config -n lab09 -o yaml
# data shows the new values

# Check the volume mount — it updates automatically (kubelet sync period ~1min)
kubectl exec $(kubectl get pod -n lab09 -l app=config-app \
  -o jsonpath='{.items[0].metadata.name}') -n lab09 -- \
  cat /etc/config/config.yaml
# log_level: debug  ← volume is updated

# Check env vars — they are frozen at pod start
kubectl exec $(kubectl get pod -n lab09 -l app=config-app \
  -o jsonpath='{.items[0].metadata.name}') -n lab09 -- \
  env | grep LOG_LEVEL
# LOG_LEVEL=info  ← env var still shows old value

# The only way to update env vars from ConfigMap is to restart the pod
kubectl rollout restart deployment/config-app -n lab09
kubectl rollout status deployment/config-app -n lab09

# Now check env vars again
sleep 10
kubectl logs $(kubectl get pod -n lab09 -l app=config-app \
  -o jsonpath='{.items[0].metadata.name}') -n lab09 --tail=8
# [ENV VAR]    LOG_LEVEL = debug       ← now updated after pod restart
# [ENV VAR]    FEATURE_FLAG = true     ← updated
```

### The Refresh Behaviour Table

```
Injection Method          Updates After ConfigMap Change?
──────────────────────────────────────────────────────────
envFrom (ConfigMap)       NO  — frozen at pod start
env.valueFrom (ConfigMap) NO  — frozen at pod start
Volume mount (ConfigMap)  YES — auto-updates within ~1 minute
                                (kubelet sync period)

envFrom (Secret)          NO  — frozen at pod start
env.valueFrom (Secret)    NO  — frozen at pod start
Volume mount (Secret)     YES — auto-updates within ~1 minute

NOTE: Even with volume mounts, the APPLICATION must re-read
the file to pick up the change. Apps that read config only
at startup won't benefit from auto-update — they need a
SIGHUP or restart regardless.
```

### Fix — Trigger a Rollout on ConfigMap Change

```bash
# The correct prod pattern: annotate deployment with configmap version
# This forces a rollout when the ConfigMap changes

# Get the ConfigMap resourceVersion (changes every update)
CM_VERSION=$(kubectl get configmap app-config -n lab09 \
  -o jsonpath='{.metadata.resourceVersion}')

# Annotate the deployment to trigger rollout
kubectl patch deployment config-app -n lab09 \
  -p "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"configmap-version\":\"${CM_VERSION}\"}}}}}"

# This triggers a rolling restart — new pods pick up latest env vars
kubectl rollout status deployment/config-app -n lab09
```

### Prod Wisdom

**Never assume a ConfigMap update immediately reflects in your app.** If you use `envFrom` or `env.valueFrom`, the pod must restart. If you use volume mounts, the file updates automatically but your app must re-read it. In prod, the safe pattern for config changes is: update ConfigMap → trigger `kubectl rollout restart` → verify new pods have new config. Anything less risks running stale config silently.

---

## Scenario 02 — Missing Key Causes Pod Failure

### What You'll Break

A pod referencing a specific key from a ConfigMap or Secret that doesn't exist. Two different failure modes — one that prevents the pod from starting (`CreateContainerConfigError`) and one where the pod starts but the mounted file is simply missing. Understanding which mode you're in determines how you find and fix it.

### Break It — Missing Key in env.valueFrom

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: partial-config
  namespace: lab09
data:
  EXISTING_KEY: "this key exists"
  # MISSING_KEY does not exist in this ConfigMap
---
apiVersion: v1
kind: Pod
metadata:
  name: missing-key-pod
  namespace: lab09
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c", "echo ${EXISTING_KEY} && echo ${MISSING_KEY} && sleep 3600"]
    env:
    - name: EXISTING_KEY
      valueFrom:
        configMapKeyRef:
          name: partial-config
          key: EXISTING_KEY
    - name: MISSING_KEY
      valueFrom:
        configMapKeyRef:
          name: partial-config
          key: MISSING_KEY       # This key doesn't exist
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
EOF
```

```bash
kubectl get pod missing-key-pod -n lab09
# NAME              READY   STATUS                       RESTARTS   AGE
# missing-key-pod   0/1     CreateContainerConfigError   0          10s

kubectl describe pod missing-key-pod -n lab09 | grep -A 5 "Events"
# Warning  Failed  kubelet  Error: couldn't find key MISSING_KEY
# in ConfigMap lab09/partial-config
```

### Fix Option A — Add the Missing Key

```bash
kubectl patch configmap partial-config -n lab09 \
  --patch '{"data":{"MISSING_KEY":"now it exists"}}'

kubectl delete pod missing-key-pod -n lab09

# Recreate — now the key exists
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: missing-key-pod
  namespace: lab09
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c", "echo ${EXISTING_KEY} && echo ${MISSING_KEY} && sleep 3600"]
    env:
    - name: EXISTING_KEY
      valueFrom:
        configMapKeyRef:
          name: partial-config
          key: EXISTING_KEY
    - name: MISSING_KEY
      valueFrom:
        configMapKeyRef:
          name: partial-config
          key: MISSING_KEY
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
EOF

kubectl get pod missing-key-pod -n lab09
# Running
```

### Fix Option B — Mark the Key as Optional

```bash
kubectl delete pod missing-key-pod -n lab09

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: missing-key-optional
  namespace: lab09
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c"]
    args:
    - |
      echo "EXISTING_KEY = ${EXISTING_KEY}"
      echo "OPTIONAL_KEY = ${OPTIONAL_KEY:-not set}"
      sleep 3600
    env:
    - name: EXISTING_KEY
      valueFrom:
        configMapKeyRef:
          name: partial-config
          key: EXISTING_KEY
    - name: OPTIONAL_KEY
      valueFrom:
        configMapKeyRef:
          name: partial-config
          key: NONEXISTENT_KEY
          optional: true          # Pod starts even if key is missing
                                  # Env var is simply not set
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
EOF

kubectl get pod missing-key-optional -n lab09
# Running — pod starts even without the key

kubectl logs missing-key-optional -n lab09
# EXISTING_KEY = this key exists
# OPTIONAL_KEY = not set
```

### Volume Mount — Specific Key Projection

```bash
# Mount only specific keys from a ConfigMap as files
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: multi-key-config
  namespace: lab09
data:
  app.conf: "timeout=30\nretries=3"
  db.conf: "host=postgres\nport=5432"
  debug.conf: "verbose=true"     # We don't want to mount this one
---
apiVersion: v1
kind: Pod
metadata:
  name: selective-mount-pod
  namespace: lab09
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c"]
    args:
    - |
      echo "=== /etc/config contents ==="
      ls -la /etc/config/
      echo "=== app.conf ==="
      cat /etc/config/app.conf
      echo "=== db.conf ==="
      cat /etc/config/db.conf
      sleep 3600
    volumeMounts:
    - name: config-vol
      mountPath: /etc/config
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
  volumes:
  - name: config-vol
    configMap:
      name: multi-key-config
      items:                      # Mount only specific keys
      - key: app.conf
        path: app.conf            # → /etc/config/app.conf
      - key: db.conf
        path: db.conf             # → /etc/config/db.conf
        # debug.conf intentionally excluded
EOF

kubectl logs selective-mount-pod -n lab09
# /etc/config/ shows: app.conf db.conf  (no debug.conf)
```

### Prod Wisdom

Use `optional: true` for keys that genuinely may not exist across all environments — for example, a debug key that only exists in staging. Use required (default) keys for anything the app cannot start without. The distinction forces you to be explicit about which config is mandatory — and prevents silent failures where an app starts without its required config and uses hardcoded defaults instead. Always prefer explicit over silent.

---

## Scenario 03 — Wrong Injection Method Breaks the App

### What You'll Break

Three injection methods exist — `envFrom`, `env.valueFrom`, and volume mounts. Each has different behaviour for large values, special characters, multiline content, and binary data. Using the wrong method for the content type breaks the app in non-obvious ways.

### Break It — Special Characters in Env Vars

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: lab09
type: Opaque
stringData:
  # Password with special characters that break shell interpolation
  DB_PASSWORD: "p@ss\$w0rd!with#special&chars"
  DB_URL: "postgresql://user:p@ss\$w0rd!@postgres:5432/mydb"
---
apiVersion: v1
kind: Pod
metadata:
  name: special-char-pod
  namespace: lab09
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c"]
    args:
    - |
      # This is UNSAFE — shell interpolation mangles special chars
      echo "Password is: $DB_PASSWORD"
      echo "URL is: $DB_URL"
      sleep 3600
    envFrom:
    - secretRef:
        name: db-secret
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
EOF

kubectl logs special-char-pod -n lab09
# Password is: p@ss$w0rd!with#special&chars
# URL is: postgresql://user:p@ss$w0rd!@postgres:5432/mydb
# Actually works fine via envFrom — K8s handles the value correctly
# But if your app reads it via shell script (not direct env access), it can break
```

### Break It — Multiline Content as Env Var

```bash
kubectl delete pod special-char-pod -n lab09

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: cert-config
  namespace: lab09
data:
  # TLS certificate — multiline content
  TLS_CERT: |
    -----BEGIN CERTIFICATE-----
    MIICvDCCAaQCCQDhpLvXu4xqFDANBgkqhkiG9w0BAQsFADAbMQswCQ==
    -----END CERTIFICATE-----
---
apiVersion: v1
kind: Pod
metadata:
  name: cert-env-pod
  namespace: lab09
spec:
  containers:
  - name: app
    image: busybox:1.35
    command: ["sh", "-c"]
    args:
    - |
      echo "=== Via env var (broken for multiline) ==="
      echo "${TLS_CERT}" | wc -l
      # Shell strips newlines — cert becomes one line — invalid PEM

      echo "=== Via volume file (correct for multiline) ==="
      cat /etc/tls/tls.crt | wc -l
      # File preserves newlines — cert is valid PEM
      sleep 3600
    env:
    - name: TLS_CERT
      valueFrom:
        configMapKeyRef:
          name: cert-config
          key: TLS_CERT
    volumeMounts:
    - name: cert-vol
      mountPath: /etc/tls
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
  volumes:
  - name: cert-vol
    configMap:
      name: cert-config
      items:
      - key: TLS_CERT
        path: tls.crt
EOF

kubectl logs cert-env-pod -n lab09
# Via env var: 1 line  ← newlines collapsed — cert is broken
# Via volume:  4 lines ← newlines preserved — cert is valid
```

### The Injection Method Decision Table

```
Content Type              Best Method        Why
──────────────────────────────────────────────────────────────
Simple key=value config   envFrom            Easiest, clean env vars
Single specific value     env.valueFrom      Precise, explicit
Multiline (certs, keys)   Volume mount       Preserves newlines and structure
Binary data               Volume mount       Only method that handles binary
Large config files        Volume mount       No env var size limits issue
Frequently changing       Volume mount       Auto-updates without pod restart
Sensitive + audit trail   env.valueFrom      Explicit reference, easier to audit
App reads from file       Volume mount       App directly reads config file path

NEVER use env var for:
  - TLS certificates and private keys
  - JSON/YAML config files
  - Anything > 1MB (env has size limits)
  - Binary content
```

### The envFrom vs env.valueFrom Distinction

```bash
# envFrom — injects ALL keys from ConfigMap/Secret as env vars
# Use when: you want everything from the ConfigMap in the environment
envFrom:
- configMapRef:
    name: app-config     # ALL keys become env vars

# env.valueFrom — injects ONE specific key
# Use when: you need a specific key, or want to rename it
env:
- name: MY_DB_HOST       # Custom env var name
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: DB_HOST       # Specific key from ConfigMap
```

### Prod Wisdom

Use **volume mounts for anything that is file-shaped** — certificates, private keys, config files, JSON, YAML. Use **env vars for anything that is value-shaped** — simple strings, ports, feature flags, log levels. This distinction is not just about correctness — it's about security. An env var is visible to every process in the container and shows up in `docker inspect` output. A volume-mounted secret file can have tight file permissions (`0400`) limiting access to specific users.

---

## Scenario 04 — Malformed Secret — Bad Base64

### What You'll Break

Secrets store values as base64-encoded strings. When you create a Secret via raw YAML with the `data` field (not `stringData`), you must encode values correctly. Wrong base64, padding errors, or accidentally encoding an already-encoded value causes the Secret to be stored with corrupt data — and your app gets garbage.

### Break It — Incorrect Base64 in Secret YAML

```bash
# WRONG — manually providing bad base64
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: malformed-secret
  namespace: lab09
type: Opaque
data:
  API_KEY: "this-is-not-base64!!!"    # NOT valid base64
EOF
# Error from server: Secret in version "v1" cannot be handled:
# illegal base64 data at input byte 7
# K8s rejects this at apply time — good
```

```bash
# WRONG — double-encoded (encoding an already base64 value)
ALREADY_ENCODED=$(echo -n "mypassword" | base64)
echo $ALREADY_ENCODED
# bXlwYXNzd29yZA==

# Accidentally encoding it again
DOUBLE_ENCODED=$(echo -n $ALREADY_ENCODED | base64)
echo $DOUBLE_ENCODED
# Ylhsd2NHeHZkMjl5WkE9PQ==

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: double-encoded
  namespace: lab09
type: Opaque
data:
  PASSWORD: "${DOUBLE_ENCODED}"   # Double encoded — garbage value
EOF

# Pod gets garbage when it reads this secret
kubectl get secret double-encoded -n lab09 \
  -o jsonpath='{.data.PASSWORD}' | base64 -d
# bXlwYXNzd29yZA==   ← decodes to the base64 string, not the original password
# Correct value should be "mypassword"
```

### The Safe Way — Always Use stringData

```bash
# stringData — K8s handles the base64 encoding for you
# No manual encoding, no encoding errors
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: correct-secret
  namespace: lab09
type: Opaque
stringData:
  API_KEY: "my-actual-api-key-plaintext"
  DB_PASSWORD: "p@ssw0rd!with$pecial#chars"
  MULTILINE_KEY: |
    line one
    line two
    line three
EOF

# Verify it stored correctly
kubectl get secret correct-secret -n lab09 \
  -o jsonpath='{.data.API_KEY}' | base64 -d
# my-actual-api-key-plaintext  ← correct

kubectl get secret correct-secret -n lab09 \
  -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
# p@ssw0rd!with$pecial#chars  ← correct, special chars preserved
```

### Inspecting and Debugging Secrets

```bash
# Check what keys exist in a Secret (don't expose values)
kubectl get secret correct-secret -n lab09 -o json | jq '.data | keys'
# ["API_KEY", "DB_PASSWORD", "MULTILINE_KEY"]

# Decode a specific key value
kubectl get secret correct-secret -n lab09 \
  -o jsonpath='{.data.API_KEY}' | base64 -d

# Check Secret type
kubectl get secret correct-secret -n lab09 \
  -o jsonpath='{.type}'
# Opaque

# List all secrets in namespace (don't show values)
kubectl get secrets -n lab09

# Describe shows keys but not values
kubectl describe secret correct-secret -n lab09
# Data
# ====
# API_KEY:       26 bytes
# DB_PASSWORD:   26 bytes
# MULTILINE_KEY: 30 bytes
# (sizes shown, values hidden)
```

### Secret Types — Know What Exists

```
Type                              Use Case
──────────────────────────────────────────────────────────
Opaque                            Generic — any key/value
kubernetes.io/tls                 TLS cert + private key (tls.crt, tls.key)
kubernetes.io/dockerconfigjson    Registry pull credentials
kubernetes.io/service-account-token  SA token (auto-created by K8s)
kubernetes.io/ssh-auth            SSH private key
kubernetes.io/basic-auth          Username + password
bootstrap.kubernetes.io/token     Bootstrap tokens
```

```bash
# Create a TLS secret correctly (from files)
# kubectl create secret tls my-tls \
#   --cert=path/to/cert.pem \
#   --key=path/to/key.pem \
#   -n lab09

# Create a docker registry pull secret
# kubectl create secret docker-registry my-registry \
#   --docker-server=registry.example.com \
#   --docker-username=user \
#   --docker-password=pass \
#   -n lab09
```

### Prod Wisdom

**Always use `stringData` when creating Secrets via YAML.** The `data` field requires valid base64 and is error-prone. `stringData` accepts plaintext and K8s handles encoding internally. In the stored Secret object, both fields are stored as `data` (base64) — `stringData` is just a write-only convenience. And always use `kubectl create secret` or sealed-secrets/external-secrets for prod — never commit plaintext Secret YAML to git, not even with `stringData`.

---

## Scenario 05 — Production Pattern — Immutable Configs and Secret Rotation

### What You'll Build

The correct production approach to ConfigMap and Secret management. Immutable resources that prevent accidental changes. A safe rotation pattern for secrets. And the `subPath` volume mount that avoids overwriting the entire directory.

### Immutable ConfigMaps and Secrets

```bash
# Immutable ConfigMap — cannot be changed after creation
# Must delete and recreate to update
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v1
  namespace: lab09
immutable: true                   # Once created, data cannot be modified
data:
  APP_VERSION: "1.0.0"
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
EOF

# Try to update it
kubectl patch configmap app-config-v1 -n lab09 \
  --patch '{"data":{"LOG_LEVEL":"debug"}}'
# Error: configmap "app-config-v1" is immutable

# Correct pattern: create a new version
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v2            # New version with new name
  namespace: lab09
immutable: true
data:
  APP_VERSION: "1.0.1"
  LOG_LEVEL: "debug"             # Updated value
  MAX_CONNECTIONS: "150"
EOF

# Update the deployment to use new ConfigMap version
kubectl set env deployment/config-app \
  --from=configmap/app-config-v2 \
  -n lab09
# This triggers a rolling restart with new config
```

### The subPath Mount — Don't Overwrite the Directory

```bash
# Problem: mounting a ConfigMap at /etc/nginx/nginx.conf
# without subPath REPLACES the entire /etc/nginx/ directory
# with just the ConfigMap files — breaking everything else in /etc/nginx/

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-custom-config
  namespace: lab09
data:
  nginx.conf: |
    events {}
    http {
      server {
        listen 80;
        location /healthz {
          return 200 'OK';
          add_header Content-Type text/plain;
        }
        location / {
          return 200 'Hello from custom nginx';
          add_header Content-Type text/plain;
        }
      }
    }
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-custom
  namespace: lab09
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    volumeMounts:
    - name: nginx-config
      mountPath: /etc/nginx/nginx.conf   # Mount ONLY this file
      subPath: nginx.conf                # subPath: inject single file
                                          # without replacing the directory
    resources:
      requests:
        memory: "64Mi"
        cpu: "100m"
      limits:
        memory: "128Mi"
        cpu: "200m"
  volumes:
  - name: nginx-config
    configMap:
      name: nginx-custom-config
EOF

kubectl wait --for=condition=ready pod/nginx-custom -n lab09 --timeout=30s

# Verify custom config is working
kubectl exec nginx-custom -n lab09 -- nginx -t
# nginx: configuration file /etc/nginx/nginx.conf test is successful

kubectl port-forward pod/nginx-custom 8080:80 -n lab09 &
curl http://localhost:8080/healthz
# OK
curl http://localhost:8080/
# Hello from custom nginx
kill %1
```

### Secret Rotation Pattern

```bash
# Step 1 — Current secret version in use
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials-v1
  namespace: lab09
type: Opaque
stringData:
  DB_PASSWORD: "old-password-version-1"
  DB_USER: "appuser"
EOF

# Deployment uses v1
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rotation-app
  namespace: lab09
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rotation-app
  template:
    metadata:
      labels:
        app: rotation-app
    spec:
      containers:
      - name: app
        image: busybox:1.35
        command: ["sh", "-c"]
        args:
        - |
          echo "DB_USER: ${DB_USER}"
          echo "DB_PASSWORD: ${DB_PASSWORD}"
          sleep 3600
        env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials-v1
              key: DB_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials-v1
              key: DB_PASSWORD
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "64Mi"
            cpu: "100m"
EOF

kubectl rollout status deployment/rotation-app -n lab09

kubectl logs $(kubectl get pod -n lab09 -l app=rotation-app \
  -o jsonpath='{.items[0].metadata.name}') -n lab09
# DB_USER: appuser
# DB_PASSWORD: old-password-version-1

# Step 2 — Rotate: create new secret version
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials-v2
  namespace: lab09
type: Opaque
stringData:
  DB_PASSWORD: "new-password-version-2"
  DB_USER: "appuser"
EOF

# Step 3 — Update deployment to use new secret (triggers rolling restart)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rotation-app
  namespace: lab09
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rotation-app
  template:
    metadata:
      labels:
        app: rotation-app
    spec:
      containers:
      - name: app
        image: busybox:1.35
        command: ["sh", "-c"]
        args:
        - |
          echo "DB_USER: ${DB_USER}"
          echo "DB_PASSWORD: ${DB_PASSWORD}"
          sleep 3600
        env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials-v2   # Updated to v2
              key: DB_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials-v2   # Updated to v2
              key: DB_PASSWORD
        resources:
          requests:
            memory: "32Mi"
            cpu: "50m"
          limits:
            memory: "64Mi"
            cpu: "100m"
EOF

kubectl rollout status deployment/rotation-app -n lab09

kubectl logs $(kubectl get pod -n lab09 -l app=rotation-app \
  -o jsonpath='{.items[0].metadata.name}') -n lab09
# DB_USER: appuser
# DB_PASSWORD: new-password-version-2  ← rotated successfully

# Step 4 — Clean up old secret after confirming new one works
kubectl delete secret db-credentials-v1 -n lab09
```

### The Production ConfigMap/Secret Checklist

```
For every ConfigMap:
  □ Use immutable: true for configs that shouldn't change after deploy
  □ Version the name (app-config-v1, v2) if using immutable pattern
  □ Use volume mount for file-shaped content (yaml, conf, certs)
  □ Use envFrom for simple key=value app settings
  □ Use subPath when mounting a single file into an existing directory
  □ Trigger rollout restart after updating a mutable ConfigMap

For every Secret:
  □ Always use stringData (not data) when writing YAML by hand
  □ Never commit Secret YAML to git — use sealed-secrets or external-secrets
  □ Create new version (v2) for rotation, update deployment, delete v1
  □ Use kubernetes.io/tls type for TLS certs, not Opaque
  □ Disable automountServiceAccountToken on pods that don't need it
  □ Use resourceNames in RBAC to limit which secrets each SA can read
```

---

## Key Commands Reference — Lab 09

```bash
# Create ConfigMap from literals
kubectl create configmap <name> \
  --from-literal=KEY=value \
  --from-literal=KEY2=value2 \
  -n <namespace>

# Create ConfigMap from file
kubectl create configmap <name> \
  --from-file=config.yaml \
  -n <namespace>

# Create Secret (always use kubectl create, not raw YAML with data field)
kubectl create secret generic <name> \
  --from-literal=KEY=value \
  -n <namespace>

# Create TLS Secret
kubectl create secret tls <name> \
  --cert=tls.crt \
  --key=tls.key \
  -n <namespace>

# Inspect ConfigMap
kubectl get configmap <name> -n <namespace> -o yaml
kubectl describe configmap <name> -n <namespace>

# Inspect Secret (values are base64 — use decode)
kubectl get secret <name> -n <namespace> -o yaml
kubectl get secret <name> -n <namespace> \
  -o jsonpath='{.data.<key>}' | base64 -d

# Verify a pod's env vars
kubectl exec <pod> -n <namespace> -- env | grep <KEY>

# Verify a volume mounted file
kubectl exec <pod> -n <namespace> -- cat /etc/config/<filename>

# Trigger rollout after config change
kubectl rollout restart deployment/<name> -n <namespace>

# Patch a configmap
kubectl patch configmap <name> -n <namespace> \
  --patch '{"data":{"KEY":"newvalue"}}'
```

---

## Prod Wisdom — The Senior Engineer Mindset

Three things that define senior-level config management:

**1. They know that env vars freeze at pod start.** Every time a ConfigMap changes, they ask: "does this need a rollout restart or does volume mount auto-update suffice?" For stateless apps with short-lived configs, volume auto-update is fine. For anything where the app reads config only at startup, a rollout restart is mandatory.

**2. They use `stringData` and never touch base64 manually.** Manual base64 encoding is a source of prod incidents — wrong padding, double encoding, newlines stripped. `stringData` eliminates the entire class of encoding errors. Reserve the `data` field for reading output only.

**3. They version their secrets for safe rotation.** Secret rotation with `kubectl edit secret` to change values in place is dangerous — there's a window where old and new pods might be using different credentials simultaneously. The versioned secret pattern (v1 → v2) is explicit, auditable, and rollback-safe.

The ConfigMap/Secret debug order:
```
kubectl describe pod   → CreateContainerConfigError? Missing key?
kubectl get configmap/secret  → Does the resource exist in the right namespace?
kubectl get secret -o jsonpath | base64 -d  → Is the value correct?
kubectl exec -- env | grep KEY  → Did the env var actually get set?
kubectl exec -- cat /etc/config/file  → Did the volume mount work?
```

---

## Cleanup

```bash
kubectl delete namespace lab09
```

---

*Lab 09 complete. Move to Lab 10 — PVC & Storage when ready.*