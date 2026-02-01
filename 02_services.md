## SERVICE TYPE COMPARISON (EXAM TABLE)

| Type          | Internal | External | IP Type        | Typical Use                     | Notes |
|---------------|----------|----------|----------------|----------------------------------|-------|
| ClusterIP     | ✅       | ❌       | Virtual        | Microservices                    | Default service type |
| NodePort      | ✅       | ✅       | Node IP        | Dev/Test                         | Exposes on `<NodeIP>:30000–32767` |
| LoadBalancer  | ✅       | ✅       | Public IP      | Prod (simple)                    | Needs cloud provider |
| ExternalName  | ❌       | ✅       | DNS alias      | External dependencies            | No selector, no proxying |
| Headless      | ✅       | ❌       | None (`None`)  | Stateful apps (e.g. DBs)         | Direct Pod DNS |
| Ingress*      | ✅       | ✅       | Public IP/DNS  | HTTP/HTTPS routing               | **Not a Service**, uses rules |


We’ll build this flow:

```
Client → NodePort Service → Frontend Pod (nginx)
                     ↓
               ClusterIP Service → Backend Pod (http-echo)
```

---

# 🔹 HANDS-ON: Kubernetes Services

## 0️⃣ Verify clean cluster

```bash
kubectl get nodes
kubectl get all
```

You should see **nothing running** except kube-system.

---

## 1️⃣ Create BACKEND Deployment (http-echo)

We **must pass args**, otherwise CrashLoop.

### Create YAML first (safe + exam-aligned)

```bash
kubectl create deployment backend \
  --image=hashicorp/http-echo \
  --dry-run=client -o yaml > backend-deploy.yaml
```

Edit it:

```bash
vi backend-deploy.yaml
```

Modify container section:

```yaml
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: http-echo
        image: hashicorp/http-echo
        args:
          - "-text=hello from backend"
          - "-listen=:8080"
```

Apply:

```bash
kubectl apply -f backend-deploy.yaml
```

Verify:

```bash
kubectl get pods -l app=backend
kubectl describe pod -l app=backend
```

✅ Pod should be **Running**, not CrashLoop.

---

## 2️⃣ Expose BACKEND via ClusterIP Service

```bash
kubectl expose deployment backend \
  --name=backend-svc \
  --port=8080 \
  --target-port=8080 \
  --type=ClusterIP
```

Verify:

```bash
kubectl get svc backend-svc
kubectl describe svc backend-svc
```

Key things to check:

* Selector: `app=backend`
* Endpoints populated ✅

---

## 3️⃣ Test BACKEND from inside cluster

Run temporary pod:

```bash
kubectl run testpod --image=busybox -it --rm -- sh
```

Inside pod:

```sh
wget -qO- http://backend-svc:8080
```

Expected output:

```
hello from backend
```

Exit:

```sh
exit
```

🔥 Backend service confirmed.

---

## 4️⃣ Create FRONTEND Deployment (nginx)

```bash
kubectl create deployment frontend \
  --image=nginx
```

Verify:

```bash
kubectl get pods -l app=frontend
```

---

## 5️⃣ Expose FRONTEND via NodePort

```bash
kubectl expose deployment frontend \
  --name=frontend-svc \
  --port=80 \
  --target-port=80 \
  --type=NodePort
```

Check NodePort:

```bash
kubectl get svc frontend-svc
```

Example output:

```
80:31xxx/TCP
```

---

## 6️⃣ Access FRONTEND from outside

Get node IP:

```bash
kubectl get nodes -o wide
```

Access:

```text
http://<NODE-IP>:<NODE-PORT>
```

You should see **nginx default page**.

---

## 7️⃣ (Optional) Make frontend talk to backend

Exec into frontend pod:

```bash
kubectl exec -it deploy/frontend -- bash
```

Inside pod:

```bash
apt update && apt install -y curl
curl http://backend-svc:8080
```

Expected:

```
hello from backend
```

Exit:

```bash
exit
```

---

## 8️⃣ Inspect Endpoints (VERY EXAMMY)

```bash
kubectl get endpointslices
kubectl describe endpointslices backend-svc
```

If endpoints are empty → **service broken**.

---

## 9️⃣ Cleanup (discipline)

```bash
kubectl delete deploy backend frontend
kubectl delete svc backend-svc frontend-svc
```

---

# 🔒 LOCK THESE IN

* `Deployment` → Pods
* `Service` → selects Pods via **labels**
* `ClusterIP` → internal only
* `NodePort` → external via `NodeIP:Port`
* No args for `http-echo` → **CrashLoop**
* No endpoints → **selector mismatch**

# ONE-LINE MEMORY HOOKS

* `Service` = stable IP + DNS
* `ClusterIP` = internal
* `NodePort` = NodeIP:Port
* `LoadBalancer` = cloud LB
* `ExternalName` = DNS alias only
* All services use selectors
* No selector = no traffic
---
