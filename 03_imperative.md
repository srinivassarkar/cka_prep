# **Step 0: Prep Environment**

Check cluster nodes:

```bash
kubectl get nodes -o wide
```

* Confirm both nodes are Ready.
* Control plane node has taint removed → workloads can be scheduled here.

---

# **Step 1: Backend Deployment + ClusterIP Service**

### 1️⃣ Generate Deployment YAML

```bash
kubectl create deployment backend-deploy \
  --image=hashicorp/http-echo \
  --replicas=3 \
  --port=5678 \
  --dry-run=client -o yaml > backend-deploy.yaml
```

### 2️⃣ Generate ClusterIP Service YAML

```bash
kubectl expose deployment backend-deploy \
  --type=ClusterIP \
  --port=9090 \
  --target-port=5678 \
  --name=backend-svc \
  --dry-run=client -o yaml >> backend-deploy.yaml
```

### 3️⃣ Edit `backend-deploy.yaml` manually

1. Add YAML separator between Deployment & Service:

```yaml
---
```

2. Add **args** to http-echo container so it actually responds:

```yaml
spec:
  containers:
    - name: backend-container
      image: hashicorp/http-echo
      args:
        - "-text=Hello from Backend"
        - "-listen=:5678"
```

3. Add **labels & selector** (critical! Service uses selector to route traffic):

```yaml
metadata:
  labels:
    app: backend
spec:
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
```

✅ After this, the Deployment is fully functional and the ClusterIP service can route to pods correctly.

---

# **Step 2: Frontend Deployment + NodePort Service**

### 1️⃣ Generate Deployment YAML

```bash
kubectl create deployment frontend-deploy \
  --image=nginx \
  --replicas=3 \
  --port=80 \
  --dry-run=client -o yaml > frontend-deploy.yaml
```

### 2️⃣ Generate NodePort Service YAML

```bash
kubectl expose deployment frontend-deploy \
  --type=NodePort \
  --port=80 \
  --target-port=80 \
  --name=frontend-svc \
  --dry-run=client -o yaml >> frontend-deploy.yaml
```

### 3️⃣ Edit `frontend-deploy.yaml` manually

1. Add YAML separator between Deployment & Service:

```yaml
---
```

2. Add **labels & selector**:

```yaml
metadata:
  labels:
    app: frontend
spec:
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
```

3. Add **nodePort manually** (imperative cannot do this):

```yaml
spec:
  ports:
    - port: 80
      targetPort: 80
      nodePort: 31000
```

✅ Frontend Deployment + NodePort Service ready.

---

# **Step 3: Apply YAMLs**

```bash
kubectl apply -f backend-deploy.yaml
kubectl apply -f frontend-deploy.yaml
```

---

# **Step 4: Verify Everything**

```bash
kubectl get all --show-labels
kubectl describe svc backend-svc
kubectl describe svc frontend-svc
kubectl get pods -o wide
```

* **backend-svc:** ClusterIP → internal only
* **frontend-svc:** NodePort → external access on host node IP + port 31000

---

# **Step 5: Test Connectivity**

1. **Inside cluster:** Test frontend → backend

```bash
kubectl exec -it <frontend-pod-name> -- curl http://backend-svc:9090
```

Expected: `Hello from Backend`

2. **External access (NodePort):**

```bash
curl http://<NodeIP>:31000
```

Expected: nginx default page

---

## 🔒 KEY TAKEAWAYS (MEMORIZE)

* Imperative = **fast + flexible**
* Use `--dry-run=client -o yaml` to generate YAML
* `http-echo` **needs args** or crashes
* `nodePort` ❌ cannot be set imperatively
* Deployment → labels must be added via YAML
* Pods → labels can be set imperatively
* `kubectl run --rm -it` = **debug superpower**

---


