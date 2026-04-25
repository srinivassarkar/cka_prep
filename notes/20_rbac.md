# 🚀 Kubernetes RBAC (Deep Dive + Hands-on + Interview Ready)

---

## 🧠 0. First Principles

RBAC answers:

```
WHO → can do WHAT → on WHICH resource → WHERE
```

| Piece    | Meaning                       |
| -------- | ----------------------------- |
| Who      | User / Group / ServiceAccount |
| What     | verbs (get, list, create…)    |
| Resource | pods, deployments, nodes      |
| Where    | namespace OR cluster          |

---

# 🔐 1. Real-World Flow

```
User → Authentication → Authorization (RBAC) → API Server
```

👉 Kubernetes does **NOT create users**

✔ Uses:

* OIDC (Okta, Google)
* IAM (AWS/GCP/Azure)
* Certificates (labs)
* ServiceAccounts (pods)

---

# ⚡ 2. Core Mental Model

```
Role            → namespace permissions
ClusterRole     → cluster OR reusable permissions

RoleBinding     → attach in namespace
ClusterRoleBinding → attach cluster-wide
```

---

# 🔥 3. Decision Tree

### Cluster resource?

→ `ClusterRole + ClusterRoleBinding`

### Namespaced resource?

| Need                | Use                              |
| ------------------- | -------------------------------- |
| One namespace       | Role + RoleBinding               |
| Multiple namespaces | ClusterRole + RoleBinding        |
| All namespaces      | ClusterRole + ClusterRoleBinding |

---

# 🧪 4. Hands-on Lab

---

## 🧱 Setup

```bash
kubectl create ns dev
kubectl create ns prod

kubectl config set-context --current --namespace=dev
```

---

# 🧪 Lab 1 — Role + RoleBinding

---

## 📄 File: `1-role-rb.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: workload-editor
rules:
# Rule 1: Pods (core API group "")
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create", "list"]

# Rule 2: Deployments (apps API group)
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["create", "list"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: bind-workload-editor
  namespace: dev
subjects:
- kind: User
  name: seema
  apiGroup: rbac.authorization.k8s.io

- kind: Group
  name: junior-admins
  apiGroup: rbac.authorization.k8s.io

roleRef:
  kind: Role
  name: workload-editor
  apiGroup: rbac.authorization.k8s.io
```

---

## 🧠 Explanation

### Role

* Scoped to `dev`
* Gives:

  * create/list pods
  * create/list deployments

👉 `apiGroups: [""]` = core resources (pods)
👉 `apps` = deployments

---

### RoleBinding

* Attaches role to:

  * user `seema`
  * group `junior-admins`

👉 Works ONLY in `dev`

---

## 🧪 Test

```bash
kubectl auth can-i create pods --as=seema
kubectl auth can-i delete pods --as=seema
```

---

# 🧪 Lab 2 — ClusterRole + ClusterRoleBinding

---

## 📄 File: `2-cr-crb.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: bind-node-reader
subjects:
- kind: User
  name: seema
  apiGroup: rbac.authorization.k8s.io

roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## 🧠 Explanation

### ClusterRole

* Cluster-level permission
* Allows:

  * get/list/watch nodes

👉 Nodes are NOT namespaced → must use ClusterRole

---

### ClusterRoleBinding

* Gives access across entire cluster

👉 Seema can now read nodes from anywhere

---

## 🧪 Test

```bash
kubectl auth can-i get nodes --as=seema
```

---

# 🧪 Lab 3 — ClusterRole + RoleBinding

---

## 📄 File: `3-cr-rb.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: elevated-workload-editor
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create", "delete", "update"]

- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["create", "delete", "update", "patch"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: use-clusterrole-in-prod
  namespace: prod
roleRef:
  kind: ClusterRole
  name: elevated-workload-editor
  apiGroup: rbac.authorization.k8s.io

subjects:
- kind: User
  name: seema
  apiGroup: rbac.authorization.k8s.io
```

---

## 🧠 Explanation

### ClusterRole

* Defines reusable permissions:

  * full control on pods
  * full control on deployments

---

### RoleBinding

* Applied ONLY in `prod`

👉 This is the key:

```
ClusterRole = reusable
RoleBinding = scope control
```

---

## 🧪 Test

```bash
kubectl auth can-i create pods -n prod --as=seema
kubectl auth can-i create pods -n dev --as=seema
```

---

# 💣 5. Critical Insight

### ❌ Wrong Thinking

```
ClusterRole = cluster-wide
```

### ✅ Correct Thinking

```
Binding decides scope
```

---

# 🔐 6. Why NOT Always ClusterRoleBinding?

Because:

```
ClusterRoleBinding = ALL namespaces
```

Includes:

* kube-system ❌
* monitoring ❌
* future namespaces ❌

---

### ✅ Safer Design

```
ClusterRole + RoleBinding
```

---

# 🧠 7. Real Admin Workflow

```
1. Identify requirement
2. Identify identity (user/group)
3. Define ClusterRole
4. Bind via RoleBinding (scoped)
5. Validate with kubectl auth can-i
```

---

# 🔍 8. Debug Commands

```bash
kubectl auth can-i <verb> <resource> --as=<user> -n <ns>
kubectl get rolebindings -A
kubectl get clusterrolebindings
```

---

# 💬 9. Interview Killer Answers

---

### 👉 How RBAC works?

> Kubernetes uses external authentication (OIDC/IAM). RBAC handles authorization using Roles/ClusterRoles and bindings.

---

### 👉 Role vs ClusterRole?

> Roles are namespace-scoped. ClusterRoles are reusable and can be applied cluster-wide or namespace-scoped depending on binding.

---

### 👉 Why not ClusterRoleBinding?

> It grants access across all namespaces, violating least privilege.

---

# 💣 10. Traps

* Kubernetes stores users ❌
* ClusterRole always global ❌
* Use CRB everywhere ❌

---

# 🧠 11. Golden Rule

```
Role → defines permissions
Binding → defines scope
```

---

# ⚡ 12. 10-sec Revision

```
Role = namespace  
ClusterRole = reusable  

RoleBinding = scoped  
ClusterRoleBinding = global  

ClusterRole + RoleBinding = safe reuse
```

---

# 🔚 Final Insight

RBAC =

```
Identity + Permission + Scope + Security
```
