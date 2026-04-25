# Kubernetes Core & Extensions 

## 🧠 1. Big Idea (Most Important Line)

```
Kubernetes = Core (orchestration) + Extensions (runtime, network, storage)
```

👉 Core manages cluster

👉 Plugins actually make things work

---

## 🏛️ 2. Kubernetes Core (Know this cold)

### Control Plane (Brain)

```
API Server → entry point (kubectl talks here)
Scheduler → assigns Pods to nodes
Controller Manager → maintains desired state
etcd → cluster state (DB)
Cloud Controller → cloud integration
```

---

### Node Components (Workers)

```
kubelet → runs containers (talks to CRI)
kube-proxy → networking rules (Services)
```

---

## 🔌 3. Extensions = 3 Types

### 1. Plugins (IMPORTANT — CKA focus)

```
CRI → Runtime (runs containers)
CNI → Network (connects pods)
CSI → Storage (persistent data)
```

👉 **Golden line:**

```
Runtime runs it
Network connects it
Storage persists it
```

---

### 2. Add-ons

```
Ingress → external traffic
Prometheus → monitoring
Grafana → dashboards
Istio → service mesh
```

---

### 3. Third-party tools

```
Helm → package manager
ArgoCD → GitOps
```

---

## 🔗 4. CRI, CNI, CSI (Deep but simple)

### 🏃 CRI — Container Runtime

* Runs containers
* Pulls images
* Start/stop containers
* Used by kubelet

Examples:

```
containerd (default)
CRI-O
gVisor (secure)
Kata (VM isolation)
```

---

### 🌐 CNI — Networking

* Assigns Pod IP
* Pod-to-pod communication
* Network policies

Examples:

```
Calico (policies)
Cilium (eBPF, high perf)
Flannel (simple)
```

---

### 💾 CSI — Storage

* Create volumes
* Attach/mount to Pods
* Snapshots, resize

Examples:

```
AWS EBS
Azure Disk
GCE PD
Ceph / Longhorn (on-prem)
```

---

## 🔁 5. Call Flow (VERY IMPORTANT)

```
kubelet
  ↓
CRI (run container)
  ↓
CNI (attach network)

kubelet
  ↓
CSI (attach storage)
```

👉 Don’t mix this up.

---

## ⚠️ 6. Docker Note (Common Question)

```
Docker runtime removed (dockershim gone)
```

BUT:

```
Docker images still work (OCI standard)
containerd is used internally
```

---

## 🧩 7. Why Plugin Architecture? (Interview GOLD)

Give exactly these 4:

```
1. Interoperability → works with any tool
2. Vendor Neutrality → no lock-in
3. Innovation → plugins evolve independently
4. Scalability & Maintainability → decoupled system
```

---

## 🧠 8. Mental Model (Keep this)

```
Control Plane = brain
Nodes = workers

CRI → "run container"
CNI → "give network"
CSI → "give storage"
```

---

## 🧪 9. Real DevOps Thinking

When something breaks, think:

```
Pod not running?
→ CRI issue

Pod no network?
→ CNI issue

Volume not mounting?
→ CSI issue
```

---

## 🎯 Final Memory Hook

```
Core = manages
CRI = runs
CNI = connects
CSI = stores
```
