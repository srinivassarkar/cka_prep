# Kubernetes Namespaces

### **What is a Namespace?**

* Logical partition in a cluster for **isolation, organization, and resource control**.
* Enables:

  * Security & access isolation
  * Avoiding naming conflicts
  * Resource quotas and limits
  * Environment segregation (dev/test/prod)
  * Multi-tenancy management

---

### **Default Namespaces**

| Namespace                   | Purpose                                             |
| --------------------------- | --------------------------------------------------- |
| `default`                   | Default for resources without a namespace           |
| `kube-system`               | Kubernetes system components (kube-dns, kube-proxy) |
| `kube-public`               | Publicly readable info                              |
| `kube-node-lease`           | Node heartbeat info                                 |
| `local-path-storage` (KIND) | Local persistent storage provisioner                |

---

## **Step 0: Create Namespace**

`app1-ns.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app1-ns
```

Apply:

```bash
kubectl apply -f app1-ns.yaml
```

Set context to `app1-ns` so all future commands default to this namespace:

```bash
kubectl config set-context --current --namespace=app1-ns
kubectl config view --minify | grep namespace:  # verify
```

✅ Now, we **don’t need -n app1-ns** in every command.

---

## **Step 1: Backend Deployment + ClusterIP Service**

`backend-deploy.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deploy
  namespace: app1-ns
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
      containers:
        - name: backend-container
          image: hashicorp/http-echo
          args:
            - "-text=Hello from Backend"
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: app1-ns
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - protocol: TCP
      port: 9090     # service port
      targetPort: 5678  # container port
```

Apply:

```bash
kubectl apply -f backend-deploy.yaml
kubectl get all  # should show backend pods + service in app1-ns
```

✅ Backend is **internal-only**, accessible via `backend-svc:9090` **inside app1-ns**.

---

## **Step 2: Frontend Deployment + NodePort Service**

`frontend-deploy.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deploy
  namespace: app1-ns
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
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
  namespace: app1-ns
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
    - protocol: TCP
      port: 80        # ClusterIP port
      targetPort: 80  # container port
      nodePort: 31000 # external access (30000-32767)
```

Apply:

```bash
kubectl apply -f frontend-deploy.yaml
kubectl get all  # verify frontend pods + NodePort service
```

✅ Frontend is **externally accessible** via `<NodeIP>:31000`.

---

## **Step 3: Verify Namespace Isolation**

### **3.1 Internal Backend Access (same namespace)**

```bash
kubectl run test-pod -it --rm --restart=Never --image=busybox /bin/sh
# inside pod
curl backend-svc:9090  # ✅ works
```

### **3.2 Cross-namespace backend access**

```bash
kubectl run test-pod-default -n default -it --rm --restart=Never --image=busybox /bin/sh
curl backend-svc:9090               # ❌ fails
curl backend-svc.app1-ns:9090       # ✅ works (FQDN format)
```

---

## **Step 4: Context Switching / Default Namespace**

Switch back to `default` namespace:

```bash
kubectl config set-context --current --namespace=default
kubectl run test-default -it --rm --restart=Never --image=busybox /bin/sh
```

Switch back to `app1-ns` anytime:

```bash
kubectl config set-context --current --namespace=app1-ns
```

---

## **Step 5: Analysis / Key Points**

| Feature                  | Observation                                                                |
| ------------------------ | -------------------------------------------------------------------------- |
| Namespace isolation      | `backend-svc` is **only reachable inside `app1-ns`** by default            |
| Cross-namespace access   | Requires **FQDN**: `<service>.<namespace>:<port>`                          |
| Service type differences | ClusterIP → internal-only (backend), NodePort → external access (frontend) |
| Selectors & labels       | **Deployment labels must match service selectors** for traffic routing     |
| Default namespace        | Saves `-n <ns>` flags in commands                                          |
| Resource segregation     | Future quotas or network policies can be applied per namespace             |
| NodePort range           | Must be **30000-32767**                                                    |

---


### **Namespace Best Practices**

* Segregate environments (dev/test/prod)
* Use for multi-tenancy
* Apply resource quotas
* Clear naming: `<team>-<app>-<env>-ns`
* Avoid manual deletion without caution
* Apply network policies for isolation

---
### **Key Points to Remember**

1. **Selectors & Labels:** Services route traffic using `selector.matchLabels`. Always match deployment labels.
2. **ClusterIP vs NodePort:**

   * ClusterIP → internal-only (backend)
   * NodePort → external access (frontend)
3. **Namespace FQDN:** `<service>.<namespace>:<port>` for cross-namespace access
4. **YAML Separators (`---`)** needed when combining multiple objects in one file.
5. **Default namespace context** simplifies commands (`-n` optional).

---
