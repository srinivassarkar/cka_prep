# ⚡ Kubernetes Data Plane Troubleshooting 

> **Stack assumed throughout:** `kubeadm` + `containerd` + `Calico`
> **Your triage order:** Node/kubelet → CNI/IPs → Services/kube-proxy → DNS → Storage → App

---

## 0. First Principles — The Mental Model That Never Changes

> *Before touching anything, ask: "Which layer is broken?"*

The Kubernetes data plane is a **layered stack**. Each layer depends on all layers below it. This is not optional knowledge — it is the lens through which every failure must be read.

```
┌─────────────────────────────────────────────┐
│          Application (probes, limits)        │  ← Layer 6: App
├─────────────────────────────────────────────┤
│          Storage (PVC / CSI)                 │  ← Layer 5: Storage
├─────────────────────────────────────────────┤
│          DNS (CoreDNS / resolv.conf)         │  ← Layer 4: Name Resolution
├─────────────────────────────────────────────┤
│          Services (kube-proxy / iptables)    │  ← Layer 3: Traffic Routing
├─────────────────────────────────────────────┤
│          Pod Networking (CNI / veth / IPAM)  │  ← Layer 2: Packet Delivery
├─────────────────────────────────────────────┤
│          Node Agent (kubelet + CRI)          │  ← Layer 1: Node Life
└─────────────────────────────────────────────┘
```

**Axiom 1 — Symptoms lie, causes don't.**
A DNS failure can look like an app crash. A kube-proxy bug can masquerade as a bad selector. Always trace the failure to the lowest broken layer.

**Axiom 2 — Read Events before editing anything.**
`kubectl describe pod <p>` tells you the cause 80% of the time. Don't touch YAML until you've read it.

**Axiom 3 — Smallest fix, immediate verification.**
One change at a time. Verify with a throwaway busybox pod. Move on.

**Axiom 4 — Kubernetes doesn't lie about endpoints.**
Empty `kubectl get endpoints <svc>` = the problem is the selector or pod readiness, *not* kube-proxy.

**Axiom 5 — The data plane only works if the control plane is healthy.**
`kubectl get componentstatuses` (or `kubectl get pods -n kube-system`) first. If etcd/API/scheduler/controller-manager are broken, fix them before anything else.

---

## 1. Reality Constraints — What Kubernetes Actually Does and Doesn't Do

| Component | What it does | What it does NOT do |
|---|---|---|
| **kubelet** | Registers node, starts/stops Pods via CRI, runs probes, reports status | Fix its own broken certs; heal from swap being ON |
| **containerd / CRI-O** | Pulls images, creates containers/sandboxes, exposes CRI socket | Fix mismatched socket paths kubelet is told to use |
| **CNI plugin** | Assigns Pod IPs, wires veth pairs, programs routes, enforces NetworkPolicy | Assign IPs if its DaemonSet is down or `/etc/cni/net.d` is empty |
| **kube-proxy** | Programs iptables/IPVS rules for Service VIPs → Pod endpoints | Create endpoints; it only reads them |
| **CoreDNS** | Resolves `<svc>.<ns>.svc.cluster.local` for Pods | Resolve if it's scaled to 0, or if kube-proxy can't route to the `kube-dns` Service |
| **CSI driver** | Provisions and mounts volumes via controller + node plugins | Mount volumes if the node plugin pod is down |

**Kubernetes does NOT:**
- Automatically heal a node whose kubelet cert has expired
- Auto-rotate TLS certs beyond the initial bootstrap (unless `rotateCertificates: true`)
- Fix a NetworkPolicy that blocks DNS traffic to CoreDNS
- Restart a stopped `containerd` systemd service
- Provide a default StorageClass unless you install one

---

## 2. Decision Logic — When to Use What

### 🧭 Master Triage Flowchart

```
kubectl get nodes
       │
       ├─ Node NotReady? ──────────────────────────────────────────────► [Scenario 1] Node/kubelet
       │
       └─ All nodes Ready?
              │
              └─ kubectl get pods -A
                     │
                     ├─ ContainerCreating (no IP)? ───────────────────► [Scenario 2] CNI
                     │
                     ├─ ImagePullBackOff? ────────────────────────────► [Scenario 5] Registry/Auth
                     │
                     ├─ CrashLoopBackOff / OOMKilled? ────────────────► [Scenario 6] App/Probes
                     │
                     ├─ Pending (PVC issue)? ─────────────────────────► [Scenario 7] CSI/PVC
                     │
                     └─ All Running?
                            │
                            └─ Service unreachable?
                                   │
                                   ├─ ClusterIP fails? ───────────────► [Scenario 3] kube-proxy
                                   ├─ NodePort fails? ────────────────► [Scenario 8] NodePort
                                   └─ DNS fails? ─────────────────────► [Scenario 4] CoreDNS
```

### Quick Cause → Check → Fix Map

| Symptom | First Check | Root Cause | Fix |
|---|---|---|---|
| Node NotReady | `systemctl status kubelet` | kubelet stopped / swap / bad cert | `systemctl start kubelet` or `swapoff -a` |
| ContainerCreating stall | `ls /etc/cni/net.d/` | CNI config missing or DS down | Re-apply CNI DaemonSet |
| ClusterIP fails | `kubectl get endpoints <svc>` | Empty endpoints (selector mismatch) | Fix Service selector |
| DNS fails | `nslookup kubernetes.default` in pod | CoreDNS pods down or resolv.conf wrong | Restart CoreDNS / fix kubelet `--cluster-dns` |
| ImagePullBackOff | `describe pod` Events | Wrong tag or missing imagePullSecret | Fix image ref or create secret |
| CrashLoopBackOff | `logs --previous` | Bad command / OOMKilled / bad probe | Fix args/limits/probe timing |
| PVC Pending | `describe pvc` | No default StorageClass | Annotate SC as default |
| NodePort dead | `ss -lnt` on node | `externalTrafficPolicy: Local` + no local pod | Move pod to that node or switch to `Cluster` |

---

## 3. Internal Working — How It Actually Happens

### 3.1 Pod Scheduling → Running (the full path)

```
API Server receives Pod spec
       │
       ▼
Scheduler assigns Pod to Node (writes nodeName)
       │
       ▼
kubelet's watch loop picks up the Pod
       │
       ├─► Calls CRI (containerd): "create sandbox"
       │         └─► containerd calls CNI plugin
       │                   └─► CNI allocates IP, creates veth pair, programs routes
       │
       ├─► CRI pulls image (via containerd's image service)
       │
       ├─► CRI creates container in sandbox
       │
       ├─► kubelet starts probe lifecycle (liveness / readiness / startup)
       │
       └─► kubelet updates Pod status → API Server → etcd
```

**Where each scenario breaks in this path:**

| Failure | Where it breaks |
|---|---|
| Node NotReady | Before kubelet even picks up the Pod |
| CNI not initialized | Step: "containerd calls CNI plugin" |
| ImagePullBackOff | Step: "CRI pulls image" |
| CrashLoopBackOff | After container starts — process exits |
| Probe failures | kubelet probe lifecycle |

### 3.2 Service Request Flow (ClusterIP)

```
Pod A sends request to ClusterIP:port
       │
       ▼
iptables (programmed by kube-proxy) intercepts the packet
       │
       ▼
DNAT rule rewrites destination → PodIP:targetPort (ECMP across healthy endpoints)
       │
       ▼
Packet arrives at Pod B
```

**Where kube-proxy gets its data:**
```
API Server → EndpointSlices → kube-proxy watches → writes iptables/IPVS rules
```
If `kube-proxy` is down: rules go stale. New pods added to the Service won't get traffic.

### 3.3 DNS Resolution Flow

```
Pod's /etc/resolv.conf: nameserver 10.96.0.10 (kube-dns ClusterIP)
       │
       ▼
nslookup kubernetes.default.svc.cluster.local
       │
       ▼
UDP packet → iptables DNAT → CoreDNS Pod IP:53
       │
       ▼
CoreDNS queries kube-apiserver for Service → returns ClusterIP
       │
       ▼
Response → Pod
```

**Why DNS can fail even when CoreDNS pods are Running:**
- kube-proxy rules for `kube-dns` Service aren't programmed → UDP never reaches CoreDNS pods
- Wrong `nameserver` in `/etc/resolv.conf` (kubelet `--cluster-dns` misconfigured)
- NetworkPolicy blocks UDP/53 to CoreDNS pods

---

## 4. Hands-On — Break It and Fix It (Production-Quality Labs)

> 🧪 **Lab structure:** Exact break command → symptom → diagnosis path → fix → verify
> Run these in order — each one ingrains a muscle memory pattern.

---

### 🔴 Scenario 1 — Node NotReady: New pods avoid the node

**Break it:**
```bash
# SSH to the worker node
sudo systemctl stop kubelet
```

**What you'll see:**
```bash
kubectl get nodes
# NAME       STATUS     ROLES    AGE   VERSION
# worker-1   NotReady   <none>   3d    v1.29.0

kubectl get pods -o wide
# New pods show Pending — scheduler skips NotReady nodes
```

**Diagnose it:**
```bash
# Step 1: Cluster view
kubectl describe node worker-1 | grep -A10 "Conditions:"
# Condition: Ready = False
# Reason: KubeletNotReady or NodeStatusUnknown

# Step 2: On the node
sudo systemctl status kubelet
# ● kubelet.service - kubelet: The Kubernetes Node Agent
#    Active: inactive (dead)   ← the smoking gun

sudo journalctl -u kubelet -b -o cat | tail -30
# (no recent entries — it was stopped)
```

**Fix it:**
```bash
sudo systemctl start kubelet
sudo systemctl enable kubelet   # persist across reboots
```

**Verify:**
```bash
kubectl get nodes -w
# worker-1   Ready   <none>   3d   v1.29.0   ← comes back within ~30s

# Test scheduling:
kubectl run probe --image=busybox:1.36 --restart=Never -- sleep 60
kubectl get pod probe -o wide  # should land on worker-1
kubectl delete pod probe
```

---

### 🔴 Scenario 2 — Node NotReady: Pod exec/logs stall

**Break it:**
```bash
# Introduce a bad kubelet config (wrong API server port)
sudo cp /etc/kubernetes/kubelet.conf /etc/kubernetes/kubelet.conf.bak
sudo sed -i 's/6443/9999/' /etc/kubernetes/kubelet.conf
sudo systemctl restart kubelet
```

**What you'll see:**
```bash
kubectl exec -it <pod> -- sh
# Error: ... failed to upgrade connection: dial tcp <node>:<port>: connect: connection refused

kubectl logs <pod>
# same stall / error
```

**Diagnose it:**
```bash
sudo journalctl -u kubelet -b -o cat | grep -i "6443\|connect\|refused" | tail -20
# failed to run Kubelet: cannot connect to https://<apiserver>:9999
# dial tcp ... :9999: connect: connection refused
```

**Fix it:**
```bash
sudo cp /etc/kubernetes/kubelet.conf.bak /etc/kubernetes/kubelet.conf
sudo systemctl restart kubelet
```

**Verify:**
```bash
kubectl get nodes
# worker-1   Ready   ← back to normal

kubectl exec -it <any-pod-on-node> -- echo "exec works"
```

---

### 🔴 Scenario 3 — Node NotReady: kubelet shows connection errors (cert issue)

**Break it:**
```bash
# Corrupt the CA cert path in kubelet config
sudo cp /etc/kubernetes/kubelet.conf /etc/kubernetes/kubelet.conf.bak
sudo sed -i 's|certificate-authority-data:.*|certificate-authority: /etc/kubernetes/nonexistent-ca.crt|' \
  /etc/kubernetes/kubelet.conf
sudo systemctl restart kubelet
```

**What you'll see:**
```bash
kubectl get nodes
# worker-1   NotReady

sudo journalctl -u kubelet -b | grep -i "x509\|cert\|ca\|no such"
# open /etc/kubernetes/nonexistent-ca.crt: no such file or directory
# x509: certificate signed by unknown authority
```

**Fix it:**
```bash
sudo cp /etc/kubernetes/kubelet.conf.bak /etc/kubernetes/kubelet.conf
sudo systemctl restart kubelet
```

**Verify:**
```bash
kubectl get nodes
# worker-1   Ready
```

> 💡 **CKA tip:** In the exam you might see a kubelet.conf with wrong `server:` IP/port or broken cert paths. Always check `journalctl -u kubelet` first — it prints the exact error string.

---

### 🔴 Scenario 4 — Pods stuck ContainerCreating (no IP)

**Break it:**
```bash
# On the worker node: move CNI config away
sudo mv /etc/cni/net.d /etc/cni/net.d.bak
sudo systemctl restart kubelet

# Create a test pod
kubectl run no-ip --image=nginx --restart=Never
```

**What you'll see:**
```bash
kubectl get pod no-ip
# NAME    STATUS              RESTARTS   AGE
# no-ip   ContainerCreating   0          2m   ← stuck

kubectl describe pod no-ip | tail -20
# Events:
#   Warning  FailedCreatePodSandBox  ...  Failed to create pod sandbox:
#            rpc error: ... failed to setup network for sandbox:
#            CNI plugin not initialized
```

**Diagnose it:**
```bash
# Check CNI config
ls -la /etc/cni/net.d/
# ls: cannot access '/etc/cni/net.d': No such file or directory  ← smoking gun

# Check CNI DaemonSet
kubectl -n kube-system get ds | grep -i calico
# calico-node   3   3   3   3   3   <none>   50m
# (DS looks fine — but nodes can't read config)

sudo journalctl -u kubelet -b -o cat | grep -i "cni\|sandbox\|plugin" | tail -10
# Failed to create pod sandbox: ... CNI plugin not initialized
```

**Fix it:**
```bash
sudo mv /etc/cni/net.d.bak /etc/cni/net.d
sudo systemctl restart kubelet
```

**Verify:**
```bash
kubectl delete pod no-ip
kubectl run no-ip --image=nginx --restart=Never
kubectl get pod no-ip -o wide
# NAME    STATUS    READY   IP           NODE
# no-ip   Running   1/1     10.244.1.x   worker-1   ← gets IP, goes Running

kubectl delete pod no-ip
```

---

### 🔴 Scenario 5 — ClusterIP request fails (curl http://svc:port)

**Break it:**
```bash
# Create a deployment and service with a MISMATCHED selector
kubectl create deployment webserver --image=nginx --replicas=2
kubectl expose deployment webserver --port=80 --target-port=80 --name=websvc

# Now break the selector
kubectl patch svc websvc -p '{"spec":{"selector":{"app":"typo-wrong-label"}}}'
```

**What you'll see:**
```bash
kubectl run curl-test --image=busybox:1.36 --restart=Never -- sh -c \
  'wget -qO- http://websvc:80 || echo FAIL'
kubectl logs curl-test
# FAIL   (or: wget: bad address 'websvc')
```

**Diagnose it:**
```bash
kubectl get svc websvc -o wide
# NAME     TYPE        CLUSTER-IP    PORT(S)   SELECTOR
# websvc   ClusterIP   10.96.x.x     80/TCP    app=typo-wrong-label   ← bad selector

kubectl get endpoints websvc
# NAME     ENDPOINTS   AGE
# websvc   <none>      2m   ← empty! nothing matches the selector

kubectl get pods -l app=webserver
# Shows pods with label app=webserver — not app=typo-wrong-label

kubectl get netpol   # check for blocking policies
# No resources found.
```

**Fix it:**
```bash
kubectl patch svc websvc -p '{"spec":{"selector":{"app":"webserver"}}}'
```

**Verify:**
```bash
kubectl get endpoints websvc
# NAME     ENDPOINTS                         AGE
# websvc   10.244.0.x:80,10.244.1.x:80      2m   ← populated!

kubectl run curl-test2 --image=busybox:1.36 --restart=Never -- sh -c \
  'wget -qO- http://websvc:80 | head -5'
kubectl logs curl-test2
# <!DOCTYPE html>  ← nginx is responding

kubectl delete pod curl-test curl-test2
kubectl delete svc websvc
kubectl delete deploy webserver
```

---

### 🔴 Scenario 6 — DNS lookup fails (nslookup kubernetes.default)

**Break it:**
```bash
# Scale CoreDNS to 0
kubectl scale deployment coredns -n kube-system --replicas=0
```

**What you'll see:**
```bash
kubectl run dns-test --image=busybox:1.36 --restart=Never -- sh -c \
  'nslookup kubernetes.default; echo exit=$?'
kubectl logs dns-test
# Server:    10.96.0.10
# Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
# nslookup: can't resolve 'kubernetes.default'
# exit=1
```

**Diagnose it:**
```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns
# NAME                       READY   STATUS    RESTARTS
# (none — scaled to 0)

kubectl -n kube-system get deployment coredns
# NAME      READY   UP-TO-DATE   AVAILABLE
# coredns   0/0     0            0   ← manually scaled down

# Check resolv.conf is pointing to right IP
kubectl run resolv-check --image=busybox:1.36 --restart=Never -- cat /etc/resolv.conf
kubectl logs resolv-check
# nameserver 10.96.0.10   ← correct, but nobody's home

# Check kube-dns Service (still exists, points to no pods)
kubectl -n kube-system get svc kube-dns
# NAME       TYPE        CLUSTER-IP   PORT(S)          
# kube-dns   ClusterIP   10.96.0.10   53/UDP,53/TCP

kubectl delete pod resolv-check
```

**Fix it:**
```bash
kubectl scale deployment coredns -n kube-system --replicas=2
```

**Verify:**
```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns -w
# Wait for Running...

kubectl run dns-test2 --image=busybox:1.36 --restart=Never -- sh -c \
  'nslookup kubernetes.default.svc.cluster.local'
kubectl logs dns-test2
# Server:    10.96.0.10
# Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
# Name:      kubernetes.default.svc.cluster.local
# Address 1: 10.96.0.1 kubernetes.default.svc.cluster.local   ← resolved!

kubectl delete pod dns-test dns-test2
```

---

## 5. Production Flow — Real-World Architecture and Design Patterns

### 5.1 The 90-Second Cluster Triage (Exam Mode)

```bash
# T+0: Cluster health (10s)
kubectl get nodes -o wide
kubectl get pods -A --field-selector=status.phase!=Running | head -30

# T+10: Data plane components (15s)
kubectl -n kube-system get pods | egrep 'calico|flannel|cilium|kube-proxy|coredns'

# T+25: Broad event scan (20s)
kubectl get events -A --sort-by='.lastTimestamp' | tail -30

# T+45: Node-level (on the suspect node) (20s)
sudo systemctl status kubelet containerd
sudo journalctl -u kubelet -b -o cat | tail -40

# T+65: Quick connectivity test (25s)
kubectl run probe --image=busybox:1.36 --restart=Never -- sh -c \
  'nslookup kubernetes.default; wget -qO- http://kubernetes:443 || echo done'
kubectl logs probe
kubectl delete pod probe
```

### 5.2 Data Plane Component Dependency Map

```
containerd ──────────────────────────────────────────────────────────┐
   └─ CRI socket (/run/containerd/containerd.sock)                    │
                                                                       ▼
kubelet (reads kubelet.conf) ──────────────────────────────► Pod lifecycle
   └─ CNI plugin call on sandbox create                               │
          └─ /etc/cni/net.d/ ──────────────────────────► Pod gets IP │
                                                                       │
kube-proxy (watches EndpointSlices) ──────────────────► iptables rules│
   └─ programs VIP → PodIP DNAT rules                                 │
                                                                       │
CoreDNS (uses kube-proxy rules to be reachable) ─────► name → IP     │
   └─ depends on kube-dns Service being routable                      │
                                                                       │
CSI driver (controller + node plugin) ──────────────► volume mount   │
   └─ node plugin must run on the same node as the Pod                │
                                                                       ▼
                                          Pod Running + Connected + Named + Stored
```

### 5.3 CNI Troubleshooting Architecture

```bash
# Check CNI config and plugin binary both present
ls -la /etc/cni/net.d/      # config
ls -la /opt/cni/bin/        # plugin binary (e.g., calico, bridge, host-local)

# CNI DaemonSet must have a pod on EVERY node
kubectl -n kube-system get ds calico-node -o wide
# DESIRED = READY must match total node count

# If a node is missing a CNI pod, check node taints:
kubectl describe node <node> | grep Taint
# The CNI DaemonSet tolerates most taints — if not, add the toleration
```

### 5.4 kube-proxy Modes and Implications

| Mode | How it works | Implication when broken |
|---|---|---|
| `iptables` (default) | NAT rules in kernel | Rules persist until kube-proxy rewrites — stale connections work briefly |
| `ipvs` | LVS hash table | Faster at scale; needs `ipvs` kernel modules loaded |

```bash
# Check kube-proxy mode
kubectl -n kube-system get cm kube-proxy -o yaml | grep mode

# Verify iptables rules exist for a Service
sudo iptables -t nat -L KUBE-SERVICES | grep <cluster-ip>
```

---

## 6. Mistakes — What Actually Breaks in Real Systems

### 6.1 Swap Left ON After Node Reboot

**Root cause:** `/etc/fstab` still has the swap entry. After the next reboot, swap comes back, kubelet refuses to start.

**Symptom:** Node stays `NotReady` after maintenance window. `journalctl -u kubelet` shows `running with swap on is not supported`.

**Fix:**
```bash
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab   # permanent
sudo systemctl start kubelet
```

---

### 6.2 kubelet Certificate Expired (X.509)

**Root cause:** Cluster was set up without `rotateCertificates: true` in kubelet config. The 1-year cert expired.

**Symptom:** `x509: certificate has expired or is not yet valid` in kubelet logs. All nodes go NotReady simultaneously.

**Fix:**
```bash
# Approve pending CSR (if auto-rotation was attempted)
kubectl get csr
kubectl certificate approve <csr-name>

# Or manually renew with kubeadm
sudo kubeadm certs renew all
sudo systemctl restart kubelet
```

---

### 6.3 CNI DaemonSet Accidentally Scaled or Deleted

**Root cause:** Someone ran `kubectl delete ds calico-node -n kube-system` or applied a bad manifest.

**Symptom:** All new Pods stuck `ContainerCreating`. Existing pods continue running (they already have IPs).

**Fix:**
```bash
# Re-apply the CNI manifest (use the exact version you deployed)
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
# Wait for DaemonSet to be fully Ready
kubectl -n kube-system rollout status ds/calico-node
```

---

### 6.4 Service `targetPort` Mismatch (App Moved to Different Port)

**Root cause:** App was updated to listen on port `8080` instead of `80`. Service still says `targetPort: 80`.

**Symptom:** Endpoints are populated (correct label), but `curl <ClusterIP>:80` returns connection refused.

**Fix:**
```bash
kubectl patch svc <svc> -p '{"spec":{"ports":[{"port":80,"targetPort":8080,"protocol":"TCP"}]}}'

# Verify: curl into a pod directly first
kubectl exec -it <pod> -- wget -qO- localhost:8080
```

---

### 6.5 `externalTrafficPolicy: Local` With No Local Endpoint

**Root cause:** NodePort Service set to `Local` (for source IP preservation) but no backend Pod scheduled on the node being hit.

**Symptom:** NodePort unreachable from external client hitting Node A, but works hitting Node B (where a Pod exists).

**Fix:**
```bash
# Option 1: Schedule a pod on the target node
kubectl patch deploy <d> --patch '{"spec":{"template":{"spec":{"nodeSelector":{"kubernetes.io/hostname":"<node>"}}}}}'

# Option 2: Switch to Cluster policy (loses source IP)
kubectl patch svc <svc> -p '{"spec":{"externalTrafficPolicy":"Cluster"}}'
```

---

### 6.6 CoreDNS Crashing Due to ConfigMap Error

**Root cause:** Someone edited the CoreDNS ConfigMap and introduced a Corefile syntax error.

**Symptom:** CoreDNS pods in `CrashLoopBackOff`. `kubectl logs` shows `plugin/loop: Loop detected`.

**Fix:**
```bash
kubectl -n kube-system edit cm coredns
# Fix the Corefile syntax error

kubectl -n kube-system rollout restart deploy/coredns
kubectl -n kube-system rollout status deploy/coredns
```

---

### 6.7 PVC Stuck Pending — No Default StorageClass

**Root cause:** PVC doesn't specify `storageClassName`, assumes a default exists. No SC is annotated as default.

**Symptom:** `kubectl get pvc` shows `Pending`. `kubectl describe pvc` says `no persistent volumes available` or `no default storage class`.

**Fix:**
```bash
kubectl get sc
# Find your SC name
kubectl annotate sc <sc-name> storageclass.kubernetes.io/is-default-class=true --overwrite
# PVC auto-binds within seconds
```

---

## 7. Interview Answers — Verbatim-Ready Responses

**Q: Walk me through how you'd troubleshoot a node showing NotReady.**

> "My first move is `kubectl describe node <node>` to read the Conditions block — that tells me if it's memory pressure, disk pressure, PID pressure, or the kubelet itself that's the problem. Then I SSH to the node and run `systemctl status kubelet` to see if the service is even running. If it's stopped, I check `journalctl -u kubelet -b` for the root cause — the three most common are: swap was left enabled after a reboot, the kubelet certificate expired, or the API server address in kubelet.conf is wrong. I fix the underlying cause, restart kubelet, and watch `kubectl get nodes` until it returns Ready. The whole flow takes about 90 seconds to diagnose and usually 30 seconds to fix."

---

**Q: A pod is stuck in ContainerCreating. How do you debug it?**

> "I immediately run `kubectl describe pod <name>` and look at the Events section at the bottom. ContainerCreating usually means CNI failure — the most common event is 'CNI plugin not initialized' or 'failed to set up sandbox.' I check if `/etc/cni/net.d/` exists and has a valid config on the node, and whether the CNI DaemonSet — Calico, Flannel, Cilium — has a healthy pod on that specific node. If the CNI config is missing, I re-apply the CNI add-on manifest and restart kubelet. Then I delete and recreate the stuck pod and verify it gets an IP."

---

**Q: `curl http://my-service:80` from inside a pod fails. What do you check?**

> "I go in this order. First: `kubectl get endpoints my-service`. If it shows `<none>`, the selector is broken — I compare the Service selector with the pod labels using `kubectl get pods -l <selector>`. If endpoints are populated, the problem is deeper. I check if kube-proxy's DaemonSet is healthy with `kubectl -n kube-system get ds kube-proxy`. I also check for NetworkPolicies that might be blocking the traffic. If kube-proxy looks fine and there's no NetworkPolicy, I verify the app is actually listening on the targetPort by exec-ing into the pod and running `wget localhost:<port>`. Ninety percent of the time it's the selector or the targetPort."

---

**Q: DNS resolution fails inside pods. What's your diagnostic path?**

> "First, I run `kubectl -n kube-system get pods -l k8s-app=kube-dns` to see if CoreDNS is running. If it's down or in CrashLoopBackOff, I check `kubectl logs` for the CoreDNS pod. If CoreDNS is running, I spin up a busybox pod and run `nslookup kubernetes.default.svc.cluster.local 10.96.0.10` — using the kube-dns ClusterIP directly — to check if CoreDNS can respond at all. Then I check `cat /etc/resolv.conf` inside the pod to confirm it's pointing to the right nameserver. A wrong nameserver means kubelet's `--cluster-dns` flag is misconfigured. I also check if a NetworkPolicy is blocking UDP/53 traffic to CoreDNS pods."

---

**Q: Explain the relationship between kube-proxy and Services.**

> "kube-proxy is the component that makes Services actually work at the network level. When a Service is created, the API server stores it in etcd. kube-proxy watches the API server — specifically EndpointSlices — and translates each Service's ClusterIP and port mapping into iptables or IPVS rules in the kernel. When a pod sends a packet to a ClusterIP, the kernel intercepts it and rewrites the destination to one of the backend pod IPs using DNAT. kube-proxy doesn't proxy the traffic directly in userspace — it just programs the rules. If kube-proxy is unhealthy, old rules stay until they're cleaned up, but new pods won't receive traffic and deleted pods might still attract connections briefly."

---

**Q: What's the difference between `externalTrafficPolicy: Cluster` and `Local`?**

> "With `Cluster`, when a packet hits any node's NodePort, kube-proxy may DNAT it to a pod on a different node — traffic can hop across nodes. This means the source IP seen by the pod is the node's IP, not the client's. With `Local`, kube-proxy only forwards to pods on the same node — if no pod is local, the connection is dropped. This preserves the source IP, which matters for rate limiting or audit logging, but it means you need to ensure backends are evenly distributed or you'll have uneven load and dead NodePorts on nodes without backends."

---

**Q: A PVC stays Pending. What are the three things you check?**

> "One: `kubectl describe pvc <name>` — read exactly why it's not binding. Two: `kubectl get sc` — verify a StorageClass exists and one is annotated as default if the PVC doesn't specify `storageClassName`. Three: `kubectl -n kube-system get pods | grep csi` — verify the CSI driver's controller and node plugin pods are Running. The node plugin must be scheduled on the same node as the consuming pod, so I check that specifically. Most of the time it's the missing default StorageClass."

---

## 8. Debugging — Fast Diagnosis Paths

### 8.1 Node NotReady — Decision Tree

```
kubectl describe node <node> | grep -A15 "Conditions:"
       │
       ├─ DiskPressure=True ──────► df -h on node; free up space
       ├─ MemoryPressure=True ────► free -h; evict or limit workloads
       ├─ PIDPressure=True ───────► ps aux | wc -l; find runaway procs
       └─ Ready=False (no pressure)
              │
              ▼
       sudo systemctl status kubelet
              │
              ├─ inactive (dead) ──► journalctl -u kubelet -b | tail -30
              │                          ├─ "swap" → swapoff -a
              │                          ├─ "x509" → cert issue → check kubelet.conf
              │                          ├─ ":6443 connection refused" → API unreachable
              │                          └─ "CRI" → check containerd
              │
              └─ active (running) ─► node is registering but failing healthcheck
                                         └─ nc -vz <apiserver> 6443
                                              └─ fails → network/firewall issue
```

### 8.2 ClusterIP Not Working — Decision Tree

```
curl http://<svc>:<port> fails from inside a pod
       │
       ▼
kubectl get endpoints <svc>
       │
       ├─ <none> ──────────────────────────────────────────────────────────────►
       │   kubectl describe svc <svc>   # read selector
       │   kubectl get pods -l <selector> # do pods exist with that label?
       │       ├─ no pods match → fix selector or pod labels
       │       └─ pods exist but not Ready → check readiness probes
       │
       └─ IPs shown ─────────────────────────────────────────────────────────►
              │
              ▼
          kubectl exec <pod> -- wget -qO- localhost:<targetPort>
              │
              ├─ fails → app not listening on that port → fix targetPort or app
              └─ works → kube-proxy or NetworkPolicy issue
                             │
                             ├─ kubectl get netpol → inspect rules
                             └─ kubectl -n kube-system get ds kube-proxy
                                    └─ not Ready → rollout restart ds/kube-proxy
```

### 8.3 DNS Failure — Decision Tree

```
nslookup kubernetes.default fails in pod
       │
       ├─ Step 1: Check CoreDNS pods
       │   kubectl -n kube-system get pods -l k8s-app=kube-dns
       │       ├─ Not Running → kubectl logs <coredns-pod>
       │       │                    ├─ Loop detected → fix Corefile ConfigMap
       │       │                    └─ Other errors → describe pod for clues
       │       └─ Running → continue
       │
       ├─ Step 2: Test direct DNS query
       │   kubectl exec <test-pod> -- nslookup kubernetes.default 10.96.0.10
       │       ├─ works → resolv.conf nameserver is wrong → fix kubelet --cluster-dns
       │       └─ fails → CoreDNS is not reachable → check kube-proxy rules for kube-dns
       │
       └─ Step 3: NetworkPolicy
           kubectl get netpol -n kube-system
           kubectl get netpol   # in pod's namespace
           # Look for rules blocking egress to port 53 or ingress to CoreDNS pods
```

### 8.4 ContainerCreating Stuck — Decision Tree

```
kubectl describe pod <p> | grep -A20 "Events:"
       │
       ├─ "CNI plugin not initialized"
       │   └─ ls /etc/cni/net.d/   # on the node
       │       ├─ empty/missing → restore CNI config + restart kubelet
       │       └─ config exists → CNI DaemonSet pod missing on this node
       │              └─ kubectl -n kube-system get pods -o wide | grep <node>
       │                 kubectl -n kube-system get ds <cni-ds>
       │                 → rollout restart ds/<cni-ds>
       │
       ├─ "failed to pull image" → ImagePullBackOff path (see 8.5)
       │
       └─ "context deadline exceeded" (CRI timeout)
           └─ sudo systemctl status containerd
               └─ restart containerd + restart kubelet
```

### 8.5 ImagePullBackOff — Quick Path

```bash
kubectl describe pod <p> | grep -A5 "Events:"
# "Failed to pull image ... manifest unknown" → wrong tag → fix image ref
# "unauthorized" / "403" → missing imagePullSecret → create docker-registry secret
# "network timeout" → containerd can't reach registry → check DNS + firewall

# Test from node:
crictl pull <image:tag>
```

---

## 9. Kill Switch — 10-Second Recall

> *The absolute minimum to hold in memory. Internalize these.*

```
LAYER ORDER:  kubelet → CRI → CNI → kube-proxy → CoreDNS → CSI → App

NODE NOTREADY:    systemctl status kubelet → journalctl -u kubelet -b
STUCK POD:        kubectl describe pod → Events tell you everything
EMPTY ENDPOINTS:  selector mismatch → fix labels
DNS BROKEN:       CoreDNS scaled? resolv.conf nameserver right?
PVC PENDING:      default StorageClass annotated?

GOLDEN RULE:  Read Events first. Smallest fix. Verify with busybox pod.
```

| Shortcut | Command |
|---|---|
| Node status | `kubectl get nodes -o wide` |
| Pod events | `kubectl describe pod <p> | tail -20` |
| Endpoints | `kubectl get endpoints <svc>` |
| kubelet logs | `sudo journalctl -u kubelet -b -o cat | tail -50` |
| CoreDNS pods | `kubectl -n kube-system get pods -l k8s-app=kube-dns` |
| CNI DaemonSet | `kubectl -n kube-system get ds` |
| kube-proxy | `kubectl -n kube-system rollout restart ds/kube-proxy` |
| DNS test pod | `kubectl run t --image=busybox:1.36 --restart=Never -- sh -c 'nslookup kubernetes.default'` |

---

## 10. Appendix — Quick Reference Card

### 10.1 Systemd Service Commands

```bash
sudo systemctl status kubelet        # is it running?
sudo systemctl start kubelet         # start it
sudo systemctl restart kubelet       # restart (after config change)
sudo systemctl enable kubelet        # persist across reboots
sudo systemctl status containerd     # CRI runtime
sudo systemctl restart containerd    # restart CRI
```

### 10.2 kubelet Log Grep Patterns

```bash
sudo journalctl -u kubelet -b -o cat | grep -i "swap"        # swap ON
sudo journalctl -u kubelet -b -o cat | grep -i "x509"        # cert issue
sudo journalctl -u kubelet -b -o cat | grep -i "6443"        # API connect
sudo journalctl -u kubelet -b -o cat | grep -i "cni"         # CNI failure
sudo journalctl -u kubelet -b -o cat | grep -i "cri\|rpc"    # CRI issue
sudo journalctl -u kubelet -b -o cat | grep -i "oom\|evict"  # OOM/eviction
```

### 10.3 Crictl Commands (CRI Debugging)

```bash
crictl info                          # runtime info, cgroup driver
crictl ps -a                         # all containers (including exited)
crictl images                        # cached images on node
crictl pull <image:tag>              # test image pull
crictl logs <container-id>           # container logs
crictl inspect <container-id>        # full container spec
```

### 10.4 Network Debugging Commands

```bash
# Service and endpoints
kubectl get svc,endpoints -o wide -n <ns>
kubectl describe svc <svc>

# kube-proxy mode
kubectl -n kube-system get cm kube-proxy -o yaml | grep mode

# iptables rules for a service
sudo iptables -t nat -L KUBE-SERVICES -n | grep <cluster-ip>
sudo iptables -t nat -L KUBE-SEP-<hash> -n    # endpoint rules

# Pod-to-pod connectivity test
kubectl exec -it <pod-a> -- wget -qO- http://<pod-b-ip>:<port>

# DNS test (specify nameserver explicitly)
kubectl exec -it <pod> -- nslookup kubernetes.default.svc.cluster.local 10.96.0.10
```

### 10.5 CNI Reference

```bash
ls /etc/cni/net.d/                   # CNI config files
ls /opt/cni/bin/                     # CNI plugin binaries
kubectl -n kube-system get ds        # CNI DaemonSet status
kubectl -n kube-system get pods -o wide | grep <cni-name>   # per-node status
```

### 10.6 PVC / Storage Reference

```bash
kubectl get pvc,pv,sc                # all storage resources
kubectl describe pvc <pvc>           # why not Bound?
kubectl get sc                       # StorageClasses
kubectl annotate sc <sc> storageclass.kubernetes.io/is-default-class=true --overwrite

# CSI driver pods
kubectl -n kube-system get pods | grep -i csi
```

### 10.7 Throwaway Debug Pods

```bash
# DNS test
kubectl run dns-probe --image=busybox:1.36 --restart=Never -- sh -c \
  'nslookup kubernetes.default; cat /etc/resolv.conf'

# HTTP test
kubectl run http-probe --image=busybox:1.36 --restart=Never -- sh -c \
  'wget -qO- http://<svc>:<port> || echo FAIL'

# Persistent debug (sleep)
kubectl run debug-pod --image=busybox:1.36 --restart=Never -- sleep 3600
kubectl exec -it debug-pod -- sh

# Cleanup all probes
kubectl delete pod dns-probe http-probe debug-pod --ignore-not-found
```

### 10.8 Common Exit Codes

| Exit Code | Meaning | Usual Cause |
|---|---|---|
| 0 | Success | Normal exit |
| 1 | Generic error | App crash, script error |
| 137 | OOMKilled (SIGKILL) | Memory limit exceeded |
| 139 | Segfault | App bug / bad memory access |
| 143 | SIGTERM (graceful stop) | Normal pod shutdown |
| 1 (CrashLoop) | Repeated exits | Bad command, missing config |

### 10.9 Triage Order — Always This Sequence

```
1. kubectl get nodes              → Is the node alive?
2. kubectl get pods -A            → What's broken?
3. kubectl describe pod <p>       → Events tell the story
4. systemctl status kubelet       → Node agent healthy?
5. ls /etc/cni/net.d/             → CNI config present?
6. kubectl get endpoints <svc>    → Service has backends?
7. kubectl -n kube-system get pods → CoreDNS / kube-proxy healthy?
8. kubectl get pvc                → Storage bound?
9. kubectl logs <pod> --previous  → What did the app say before dying?
```

---

