# 🧠 Kubernetes Control Plane Troubleshooting


> **One document. No hand-holding. No dumbing down.**
> This is your brain dump replacement — every scenario, every flag, every root cause.

---

## 0. First Principles

> *What never changes, no matter the cluster, no matter the failure.*

### The Mental Model

```
kubelet watches /etc/kubernetes/manifests/
        ↓
Spawns static pods (apiserver, etcd, controller-manager, scheduler)
        ↓
Publishes mirror pods to API server (read-only)
        ↓
You see them in kubectl — but files are the source of truth
```

**The Three Laws of Control Plane Troubleshooting:**

| Law | What it means |
|-----|---------------|
| **The manifest is god** | Fix the YAML in `/etc/kubernetes/manifests/` — kubelet reconciles automatically |
| **Socket before logs** | If a port isn't listening, there's no point reading logs yet |
| **etcd first, everything else second** | Apiserver can't function without etcd. Always rule it out |

### Port Map — Burn This Into Memory

| Component | Port | Protocol | Who connects to it |
|-----------|------|----------|--------------------|
| kube-apiserver | 6443 | HTTPS | Everything (kubectl, kubelet, controllers) |
| etcd client | 2379 | HTTPS | apiserver |
| etcd peer | 2380 | HTTPS | other etcd members |
| controller-manager | 10257 | HTTPS | metrics/health probes |
| scheduler | 10259 | HTTPS | metrics/health probes |
| kubelet | 10250 | HTTPS | apiserver |

### Static Pod Architecture

```
/etc/kubernetes/manifests/
├── kube-apiserver.yaml          ← change here → kubelet restarts pod
├── kube-controller-manager.yaml
├── kube-scheduler.yaml
└── etcd.yaml

Mirror pods (kubectl view only):
  kube-apiserver-<nodeName>
  kube-controller-manager-<nodeName>
  kube-scheduler-<nodeName>
  etcd-<nodeName>
```

**Why mirror pods?** Kubelet publishes them so `kubectl` can show status. But you can't edit them via kubectl — edit the file, and kubelet does the rest.

---

## 1. Reality Constraints

> *What Kubernetes actually does and doesn't do — the bits that bite you in the exam.*

### What kubeadm manages

- **Cert location:** `/etc/kubernetes/pki/`
- **Kubeconfigs:** `/etc/kubernetes/*.conf` (admin, controller-manager, scheduler)
- **Static pod manifests:** `/etc/kubernetes/manifests/`
- **etcd data dir:** `/var/lib/etcd` (default)
- **Cert validity:** 1 year for most certs; CA is 10 years

### What kubeadm does NOT do automatically

- Renew certs (you must run `kubeadm certs renew all`)
- Recover from bad manifests (you broke it, you fix it)
- Handle time sync (your OS must manage NTP)
- Clean up stale containerd state (use `crictl rm -f`)

### Critical constraints

| Constraint | Impact |
|------------|--------|
| etcd MUST be reachable before apiserver starts | Apiserver crashloops without etcd |
| Cert SANs must include all IPs/hostnames that connect | Even localhost mismatches cause TLS failures |
| `--secure-port=0` disables health/metrics endpoint | Liveness probe fails → kubelet kills pod → crashloop |
| `/var/lib/etcd` must be 0700 root:root | Wrong perms → etcd won't start |
| Clock skew >5s breaks leader election and TLS | NTP is not optional |

---

## 2. Decision Logic

> *When to use what. No ambiguity.*

### Primary Triage Flowchart

```
kubectl not working?
        │
        ├─► ss -lntp | grep 6443
        │     │
        │     ├─ NOT listening → crictl ps -a | grep apiserver
        │     │                        │
        │     │                        ├─ CrashLoop → tail /var/log/containers/kube-apiserver-*.log
        │     │                        │               → fix manifest, check etcd, check certs
        │     │                        └─ Not present → kubelet issue (systemctl status kubelet)
        │     │
        │     └─ Listening → kubeconfig issue? firewall? cert expired?
        │                     → kubectl config view --minify
        │                     → openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -enddate
        │
        ├─► Pod stuck Pending?
        │     │
        │     ├─ kubectl describe pod → Events section
        │     │     ├─ "no nodes available" → scheduler down OR node taints/resources
        │     │     └─ no events at all → scheduler definitely down
        │     │
        │     └─ kubectl -n kube-system get pods | grep scheduler
        │
        └─► Deployment not scaling?
              └─ kubectl -n kube-system get pods | grep controller-manager
```

### Tool Selection: Which Log Source to Use

| Situation | Use This |
|-----------|----------|
| Component is running | `kubectl -n kube-system logs -l component=<name>` |
| Component is crashing / not starting | `sudo tail -f /var/log/containers/<name>-*.log` |
| Need raw CRI view | `sudo crictl ps -a && sudo crictl logs <id>` |
| Port/socket inspection | `sudo ss -lntp` |
| Cert inspection | `openssl x509 -in <cert> -noout -text` |
| Quick cert expiry check | `sudo kubeadm certs check-expiration` |

### When to Use `ss` vs `crictl` vs `logs`

```
Symptom: connection refused → ss first (is anything listening?)
Symptom: crashloop           → crictl ps -a (container state) → then logs
Symptom: auth failures       → logs first (x509/kubeconfig errors)
Symptom: cert issues         → openssl + kubeadm certs check-expiration
```

---

## 3. Internal Working

> *How it actually happens under the hood — step by step.*

### API Server Startup Sequence

```
1. kubelet reads /etc/kubernetes/manifests/kube-apiserver.yaml
2. kubelet pulls image (if needed) via containerd
3. containerd creates container, mounts hostPath volumes (certs, etc.)
4. kube-apiserver binary starts, reads flags
5. Tries to connect to etcd on --etcd-servers (default: https://127.0.0.1:2379)
6. Opens port 6443 for HTTPS
7. Publishes mirror pod object to itself (chicken-and-egg: apiserver publishes its own mirror)
8. Health probe (--healthz-port or /readyz on 6443) returns OK
```

**Where it breaks:** Steps 3 (bad cert paths), 5 (etcd unreachable), 6 (port conflict), 8 (cert invalid)

### etcd Startup Sequence

```
1. kubelet reads /etc/kubernetes/manifests/etcd.yaml
2. etcd starts, opens --data-dir (default: /var/lib/etcd)
3. Opens --listen-client-urls (default: https://127.0.0.1:2379)
4. Opens --listen-peer-urls (default: https://0.0.0.0:2380)
5. TLS handshake for all connections (client certs + CA)
6. WAL replay + snapshot load
7. Leader election (raft protocol)
```

**Where it breaks:** Step 2 (wrong data-dir or missing mount), Step 3 (localhost removed from listen-client-urls), Step 5 (cert/SAN mismatch)

### Certificate Renewal Flow (kubeadm)

```
kubeadm certs renew all
    ↓
Reads CA from /etc/kubernetes/pki/ca.{crt,key}
    ↓
Issues new leaf certs (apiserver.crt, apiserver-kubelet-client.crt, etc.)
    ↓
Writes to /etc/kubernetes/pki/
    ↓
kubeadm init phase kubeconfig all
    ↓
Regenerates /etc/kubernetes/{admin,controller-manager,scheduler}.conf
    ↓
systemctl restart kubelet
    ↓
kubelet re-reads manifests → restarts static pods with new certs
```

**Gotcha:** If you renew while the clock is wrong (future), new certs have `NotBefore` in the future. You must reset the clock **then renew again**.

### Leader Election (Controller-Manager & Scheduler)

```
Multiple replicas possible → only ONE is active leader
Leader election via Lease objects in kube-system namespace
Clock skew > a few seconds → candidates disagree on lease expiry → flapping
Check: kubectl get leases -n kube-system | grep -E 'controller|scheduler'
```

---

## 4. Hands-On

> *Production-quality commands and YAML. Nothing simplified.*

### 4.1 `ss` Mastery

```bash
# All listening TCP sockets with process names (your go-to command)
sudo ss -lntp

# Filter for a specific port
sudo ss -lntp | grep 6443
sudo ss -lntp | grep -E ':2379|:2380'

# See ESTABLISHED connections (who's actually talking)
ss -tnp state established

# Filter established connections to/from a specific IP
ss -tnp state established | grep 172.31.44.224

# Summary of all socket types
ss -s
```

**Reading `ss -lntp` output:**

```
State   Recv-Q Send-Q  Local Address:Port   Peer Address:Port  Process
LISTEN  0      4096    *:6443               *:*                users:(("kube-apiserver",pid=7103,fd=3))
LISTEN  0      4096    127.0.0.1:2379       0.0.0.0:*          users:(("etcd",pid=1183,fd=7))
LISTEN  0      4096    172.31.35.156:2379   0.0.0.0:*          users:(("etcd",pid=1183,fd=8))
```

| Column | Meaning |
|--------|---------|
| Local Address:Port | Where **this machine** is listening |
| Peer Address:Port | Always `0.0.0.0:*` on LISTEN rows (no client yet) |
| Process | Which process owns the socket |

**Interpretation:**
- `*:6443` → apiserver listening on all interfaces (correct for multi-node)
- `127.0.0.1:2379` → etcd client port on loopback only (fine for stacked etcd)
- `172.31.x.x:2379` → etcd also listening on node IP (needed if apiserver uses node IP)

### 4.2 Static Pod Manifest Inspection

```bash
# Read top of apiserver manifest (flags, not full volume list)
sudo sed -n '1,120p' /etc/kubernetes/manifests/kube-apiserver.yaml

# Grep for specific flags
sudo grep -E 'advertise-address|etcd-servers|client-ca-file|enable-admission' \
  /etc/kubernetes/manifests/kube-apiserver.yaml

# etcd data dir and URLs
sudo grep -E 'data-dir|listen-client|advertise-client' \
  /etc/kubernetes/manifests/etcd.yaml

# Controller-manager critical flags
sudo grep -E 'kubeconfig|secure-port|cluster-signing' \
  /etc/kubernetes/manifests/kube-controller-manager.yaml

# Scheduler flags
sudo grep -E 'kubeconfig|secure-port' \
  /etc/kubernetes/manifests/kube-scheduler.yaml
```

### 4.3 Container Runtime (crictl)

```bash
# All containers including stopped/crashed
sudo crictl ps -a

# Logs for a specific container ID
sudo crictl logs <container-id>

# Force remove stuck container
sudo crictl rm -f <container-id>

# List all pod sandboxes
sudo crictl pods

# Stop and remove a stuck pod sandbox
sudo crictl stopp <pod-sandbox-id>
sudo crictl rmp <pod-sandbox-id>
```

### 4.4 etcd Health Commands

```bash
# Set env vars once (copy-paste ready for exam)
export ETCDCTL_API=3
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key

# Basic liveness check
etcdctl endpoint health

# Detailed status: DB size, leader, raft index
etcdctl endpoint status --write-out=table

# List all keys (dangerous in prod, fine in exam)
etcdctl get / --prefix --keys-only | head -30

# Defrag if DB is large (careful in prod)
etcdctl defrag
```

### 4.5 Certificate Operations

```bash
# Check all cert expiry dates at once
sudo kubeadm certs check-expiration

# Inspect a specific cert manually
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | grep -A2 Validity
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -enddate

# Check SANs on a cert
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | grep -A10 'Subject Alternative'

# Renew all certs
sudo kubeadm certs renew all

# Regenerate all kubeconfigs (after cert renewal)
sudo kubeadm init phase kubeconfig all

# Full cert renewal + cluster recovery flow
sudo kubeadm certs renew all
sudo kubeadm init phase kubeconfig all
sudo systemctl restart kubelet
mkdir -p ~/.kube
sudo cp -f /etc/kubernetes/admin.conf ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
kubectl get nodes
sudo kubeadm certs check-expiration
```

### 4.6 Simulating and Recovering Cert Expiry (Lab)

```bash
# === BREAK IT ===
# Stop time sync
sudo systemctl stop systemd-timesyncd 2>/dev/null || true
sudo systemctl stop chronyd 2>/dev/null || true

# Jump clock to future (forces x509 failures)
sudo date -s '2027-12-01 12:00:00'

# Verify the damage
sudo kubeadm certs check-expiration   # shows "expired"
journalctl -u kubelet -p err --since "15m" | grep -i x509

# === FIX IT ===
# Option A: Renew while in future (then re-renew after restoring time)
sudo kubeadm certs renew all
sudo kubeadm init phase kubeconfig all
sudo systemctl restart kubelet
mkdir -p ~/.kube && sudo cp -f /etc/kubernetes/admin.conf ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

# Restore time
sudo date -s 'NOW'   # or use actual current time string
sudo systemctl start systemd-timesyncd 2>/dev/null || true
sudo systemctl start chronyd 2>/dev/null || true

# IMPORTANT: Renew again with correct time (previous certs have NotBefore in 2027)
sudo kubeadm certs renew all
sudo kubeadm init phase kubeconfig all
sudo systemctl restart kubelet
mkdir -p ~/.kube && sudo cp -f /etc/kubernetes/admin.conf ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

# Verify
kubectl get nodes
sudo kubeadm certs check-expiration
```

### 4.7 Cleaning Stale containerd State (Post-Cert-Rotation)

```bash
# Error you'll see:
# "name ... is reserved for <old-container-id>"

# Fix for controller-manager
sudo crictl ps -a | grep -i kube-controller-manager
sudo crictl rm -f <CONTAINER_ID_1> <CONTAINER_ID_2>
# If sandbox also stuck:
sudo crictl pods | grep -i kube-controller-manager
sudo crictl stopp <POD_SANDBOX_ID>; sudo crictl rmp <POD_SANDBOX_ID>
sudo systemctl restart kubelet

# Same pattern for scheduler
sudo crictl ps -a | grep -i kube-scheduler
sudo crictl rm -f <CONTAINER_IDs>
sudo systemctl restart kubelet

# Verify both are healthy
kubectl -n kube-system get pods -l component=kube-controller-manager
kubectl -n kube-system get pods -l component=kube-scheduler
kubectl -n kube-system get lease | grep -E 'controller|scheduler'
```

---

## 5. Production Flow

> *Real-world patterns — how these pieces fit together.*

### Stacked etcd Topology (kubeadm default)

```
Control Plane Node
├── kube-apiserver        (port 6443, connects to 127.0.0.1:2379)
├── etcd                  (ports 2379/2380, data at /var/lib/etcd)
├── kube-controller-manager (port 10257, uses /etc/kubernetes/controller-manager.conf)
└── kube-scheduler        (port 10259, uses /etc/kubernetes/scheduler.conf)

All run as static pods, certs at /etc/kubernetes/pki/
```

### Cert Relationships (Who Signs What)

```
/etc/kubernetes/pki/ca.{crt,key}                    ← Cluster CA (10yr)
    ├── apiserver.crt                                ← API server TLS cert (1yr)
    ├── apiserver-kubelet-client.crt                 ← apiserver → kubelet auth
    ├── front-proxy-ca.crt                           ← Front proxy CA
    └── /etcd/ca.crt                                 ← etcd CA
            ├── etcd/server.crt                      ← etcd server TLS
            ├── etcd/peer.crt                        ← etcd peer TLS
            └── apiserver-etcd-client.crt            ← apiserver → etcd auth
```

### kubeconfig File Map

| File | Used by | References |
|------|---------|-----------|
| `/etc/kubernetes/admin.conf` | kubectl (admin) | apiserver client cert |
| `/etc/kubernetes/controller-manager.conf` | controller-manager | apiserver client cert |
| `/etc/kubernetes/scheduler.conf` | scheduler | apiserver client cert |
| `/etc/kubernetes/kubelet.conf` | kubelet | apiserver client cert |

All four are regenerated by `kubeadm init phase kubeconfig all`.

### Observability Pattern (Production Control Plane)

```bash
# Health check all 4 control plane components at once
kubectl -n kube-system get pods -l tier=control-plane
kubectl -n kube-system get lease

# Port confirmation
sudo ss -lntp | grep -E ':6443|:2379|:2380|:10257|:10259'

# Recent errors across all control plane components
sudo ls /var/log/containers/ | grep -E 'kube-apiserver|etcd|controller|scheduler'
sudo tail -n 30 /var/log/containers/kube-apiserver-*.log
```

---

## 6. Mistakes

> *What actually breaks in real systems. Root cause + fix.*

### Mistake 1: Wrong `--advertise-address` in apiserver

**Symptom:** `kubectl` times out from your workstation; `ss -lntp` shows 6443 listening on `127.0.0.1` only.

**Root cause:** `--advertise-address=127.0.0.1` in kube-apiserver.yaml → apiserver only binds to loopback.

**Fix:**
```bash
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
# Change: --advertise-address=127.0.0.1
# To:     --advertise-address=<control-plane-node-IP>
# Save → kubelet restarts pod automatically
```

---

### Mistake 2: etcd `--listen-client-urls` missing localhost

**Symptom:** etcd is running, but apiserver (on same node) can't connect to `127.0.0.1:2379`.

**Root cause:** Someone edited `--listen-client-urls` to only include the node IP, removing `https://127.0.0.1:2379`.

**Fix:**
```bash
sudo grep 'listen-client-urls' /etc/kubernetes/manifests/etcd.yaml
# If 127.0.0.1:2379 is missing, add it back:
# --listen-client-urls=https://127.0.0.1:2379,https://<node-ip>:2379
```

---

### Mistake 3: Cert SAN mismatch in etcd

**Symptom:** `x509: certificate is valid for 127.0.0.1, not 172.31.x.x` in apiserver logs.

**Root cause:** etcd cert was issued without the node IP in the SAN. Happens after IP changes or when external etcd uses a different IP than expected.

**Fix:**
```bash
# Check current SANs
openssl x509 -in /etc/kubernetes/pki/etcd/server.crt -noout -text | grep -A5 'Subject Alternative'

# Renew etcd certs with correct SANs
sudo kubeadm init phase certs etcd-server
sudo kubeadm init phase certs etcd-peer
sudo kubeadm init phase certs apiserver-etcd-client
sudo systemctl restart kubelet
```

---

### Mistake 4: `--secure-port=0` on controller-manager or scheduler

**Symptom:** Pod starts but immediately gets killed; liveness probe fails; `CrashLoopBackOff`.

**Root cause:** `--secure-port=0` disables the HTTPS health endpoint. Kubelet's liveness probe to `/healthz` or `/readyz` returns connection refused → pod gets killed → restart loop.

**Fix:**
```bash
sudo grep 'secure-port' /etc/kubernetes/manifests/kube-controller-manager.yaml
# Restore: --secure-port=10257 (controller-manager) or --secure-port=10259 (scheduler)
```

---

### Mistake 5: Wrong kubeconfig path in scheduler or controller-manager

**Symptom:** Scheduler/CM pod starts but can't authenticate to apiserver; logs show 401 Unauthorized or "no such file or directory".

**Root cause:** `--kubeconfig` flag pointing to wrong path.

**Fix:**
```bash
# Controller-manager: should be
--kubeconfig=/etc/kubernetes/controller-manager.conf

# Scheduler: should be
--kubeconfig=/etc/kubernetes/scheduler.conf

# Verify file exists
ls -la /etc/kubernetes/controller-manager.conf /etc/kubernetes/scheduler.conf
```

---

### Mistake 6: etcd wrong data-dir or missing volume mount

**Symptom:** etcd starts fresh every time (empty store), all cluster state gone after restart.

**Root cause:** `--data-dir` in etcd.yaml doesn't match the `hostPath` volume mount, OR the hostPath volume is missing entirely.

**Fix:**
```bash
# Check etcd.yaml for both the flag and the volume
sudo grep -A5 'data-dir\|hostPath' /etc/kubernetes/manifests/etcd.yaml

# Should see:
# --data-dir=/var/lib/etcd
# volumes:
#   hostPath:
#     path: /var/lib/etcd
#     type: DirectoryOrCreate

# Check permissions
ls -la /var/lib/etcd  # should be drwx------ root root (0700)
sudo chmod 700 /var/lib/etcd
```

---

### Mistake 7: Renewing certs while clock is wrong

**Symptom:** After cert renewal, components still fail with `x509: certificate is not yet valid` OR `NotBefore: 2027-12-01`.

**Root cause:** `kubeadm certs renew all` uses the system clock for `NotBefore`. Running it while the clock is in 2027 issues certs that aren't valid until 2027.

**Fix:**
```bash
# Reset clock, then renew again
sudo date -s 'NOW'                        # or set actual current datetime
sudo systemctl start systemd-timesyncd
sudo kubeadm certs renew all
sudo kubeadm init phase kubeconfig all
sudo systemctl restart kubelet
```

---

### Mistake 8: Stale containerd containers after rapid restarts

**Symptom:** `kubectl -n kube-system get pods` shows controller-manager or scheduler in `Error` or stuck state; `crictl ps -a` shows old container IDs still listed.

**Root cause:** Rapid restarts during cert rotation leave dead container records in containerd's state. Kubelet can't create a new container because the name is "reserved" by the old dead one.

**Fix:**
```bash
sudo crictl ps -a | grep kube-controller-manager
sudo crictl rm -f <dead-container-id>
sudo systemctl restart kubelet
```

---

## 7. Interview Answers

> *Full spoken sentences. Verbatim-ready. No bullet dumps.*

---

**Q: What are static pods and how do they differ from regular pods?**

"Static pods are pods managed directly by the kubelet on a specific node, rather than by the API server. The kubelet watches a directory — by default `/etc/kubernetes/manifests/` — and creates or recreates pods based on YAML files it finds there. If you delete a static pod via kubectl, it comes right back because the manifest file is still there. The API server doesn't schedule static pods; it just shows a read-only representation called a mirror pod. This is exactly how all four control plane components — the API server, etcd, the controller manager, and the scheduler — are managed on a kubeadm cluster."

---

**Q: The API server is down. Walk me through your diagnosis.**

"My first move is to check whether port 6443 is actually listening, using `ss -lntp | grep 6443`. If it's not listening, the process isn't running or is crash-looping. I'd then use `crictl ps -a | grep kube-apiserver` to see the container state. If it's crashing, I'd tail the container log at `/var/log/containers/kube-apiserver-*.log` to read the error. The most common causes are wrong cert or key paths in the manifest, a bad `--advertise-address`, or etcd being unreachable. I'd fix the manifest under `/etc/kubernetes/manifests/`, let kubelet reconcile, and verify the port comes up. If etcd is the issue, I'd address that first — the API server can't start without it."

---

**Q: How do you troubleshoot etcd on a kubeadm cluster?**

"I start with `crictl ps -a | grep etcd` to check if it's running. Then I tail `/var/log/containers/etcd-*.log` to see what it's complaining about — it'll tell me if it's a data directory issue, a TLS error, or a connectivity problem. I also run `ss -lntp | grep 2379` to confirm whether the client port is actually listening and on which address. If the API server can't reach it, there's often a mismatch between where etcd is listening and where the API server is trying to connect — for example, etcd only listening on `127.0.0.1` but the API server pointing to the node IP, or vice versa. For cert issues, I'll use `openssl x509` to check SANs. For a quick health check when etcd is running, I'll use `etcdctl endpoint health` with the right certs set."

---

**Q: Why would pods stay Pending even though nodes are Ready?**

"Pending without a Scheduled event almost always means the scheduler isn't running or can't place the pod. My first check is `kubectl -n kube-system get pods | grep scheduler`. If the scheduler is down, I look at its manifest and logs — common causes are a bad kubeconfig path or `--secure-port=0` which makes the liveness probe fail. If the scheduler is running, I'd look at the pod's events with `kubectl describe pod` — it'll tell me exactly why: insufficient resources, a taint the pod doesn't tolerate, a node affinity mismatch, or a PVC that hasn't bound. Each of those has a specific fix."

---

**Q: How do you renew certificates on a kubeadm cluster?**

"The exam-standard approach is `kubeadm certs renew all`, which re-issues all control plane leaf certificates using the existing CA. After that, I regenerate the kubeconfigs with `kubeadm init phase kubeconfig all`, then restart kubelet with `systemctl restart kubelet` so the static pods reload with the new certs. I also copy the fresh admin.conf to `~/.kube/config`. One critical gotcha: if you renew while the system clock is wrong — like if you jumped the clock to simulate expiry — the new certs will have a future `NotBefore` date. You have to reset the clock first, then renew again."

---

**Q: What does the controller manager do, and what breaks when it's down?**

"The controller manager runs all the built-in reconciliation controllers — ReplicaSet, Deployment, DaemonSet, Job, EndpointSlice, Node lifecycle, and many others. When it's down, Deployments won't scale: you'll have a desired count of 3 but stay at whatever replicas currently exist. New nodes can't get their bootstrap CSRs approved, so they can't join. Garbage collection of orphaned resources stops. The symptom is usually a Deployment with a desired count that never changes, and no 'Scaled up' events when you describe it. I'd check `kubectl -n kube-system get pods -l component=kube-controller-manager`, look at the logs, and fix the manifest — most commonly the kubeconfig path or the secure port setting."

---

**Q: What is a mirror pod?**

"A mirror pod is a read-only API object that the kubelet publishes to represent a static pod. Since static pods are managed by the kubelet directly from manifest files — not by the scheduler or API server — they wouldn't otherwise be visible in kubectl. The kubelet creates a mirror pod so you can see status, events, and logs via the API. The name is always `<podName>-<nodeName>`. You can read it and get logs, but you can't edit or delete it through kubectl — any change you make would just be reverted. To actually change or stop a static pod, you edit or remove the file in `/etc/kubernetes/manifests/`."

---

## 8. Debugging

> *Fast diagnosis paths. Follow the tree, find the root cause.*

### Path 1: `kubectl` Not Responding

```
Step 1: Is the API server listening?
  sudo ss -lntp | grep 6443
  
  ├─ YES (port open) ──────────────────────────────────────────────────┐
  │                                                                    │
  │  Step 2: Can you reach it?                                         │
  │    curl -k https://localhost:6443/readyz                           │
  │    ├─ OK → kubeconfig issue (check ~/.kube/config)                 │
  │    └─ FAIL → cert expired? firewall? (check apiserver logs)        │
  │                                                                    │
  └─ NO (nothing listening) ──────────────────────────────────────────►│
                                                                       │
  Step 2: Is the container running?                                    │
    sudo crictl ps -a | grep kube-apiserver                           │
    ├─ Running → something else on 6443 (lsof -i:6443)                │
    ├─ CrashLoop → Step 3                                              │
    └─ Not present → kubelet issue:                                    │
         systemctl status kubelet                                      │
         journalctl -u kubelet -n 50                                   │
                                                                       │
  Step 3: Read the crash logs                                          │
    sudo tail -n 50 /var/log/containers/kube-apiserver-*.log           │
                                                                       │
    ├─ "no such file or directory" (cert/key path) ─────────────────► │
    │    sudo grep 'cert\|key\|ca' /etc/kubernetes/manifests/kube-apiserver.yaml
    │    Fix the path → save → wait for kubelet to restart pod         │
    │                                                                  │
    ├─ "connection refused 127.0.0.1:2379" (etcd) ──────────────────► │
    │    Go to Path 2 (etcd diagnosis)                                 │
    │                                                                  │
    └─ "x509: certificate has expired" ────────────────────────────── │
         Go to Path 4 (cert expiry)                                    │
```

---

### Path 2: etcd Unhealthy

```
Step 1: Is etcd running?
  sudo crictl ps -a | grep etcd
  ├─ Running → go to Step 3
  └─ Not running / crashing → Step 2

Step 2: Read etcd logs
  sudo tail -n 100 /var/log/containers/etcd-*.log
  ├─ "opened backend db" at unexpected path → data-dir mismatch
  │    sudo grep 'data-dir' /etc/kubernetes/manifests/etcd.yaml
  │    Ensure --data-dir=/var/lib/etcd and hostPath matches
  ├─ "x509" / "no such file" → cert/key path wrong or SAN mismatch
  │    openssl x509 -in /etc/kubernetes/pki/etcd/server.crt -noout -text
  └─ "no space left on device" → disk full
       df -h /var/lib/etcd

Step 3: Is the right port listening?
  sudo ss -lntp | grep 2379
  ├─ 127.0.0.1:2379 → fine for stacked etcd (apiserver must use 127.0.0.1)
  ├─ <node-ip>:2379 only → update apiserver --etcd-servers or add 127.0.0.1 to listen-client-urls
  └─ Nothing → etcd crashed, back to Step 2

Step 4: etcd health check (when it's up)
  export ETCDCTL_API=3
  export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
  export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt
  export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key
  etcdctl endpoint health
  etcdctl endpoint status --write-out=table
```

---

### Path 3: Pods Stuck Pending

```
Step 1: Is it unscheduled?
  kubectl get pod <name> -o wide   # NODE column empty?
  kubectl describe pod <name> | grep -A20 Events:

Step 2: Read the scheduler event message
  "0/N nodes are available" → parse the reason:
  ├─ "Insufficient cpu/memory" → lower requests or free node capacity
  ├─ "node(s) had taint" → add toleration to pod spec
  ├─ "node(s) didn't match node selector/affinity" → fix labels or affinity
  ├─ "node(s) had volume node affinity conflict" → check PVC/SC zone
  └─ NO events at all → scheduler is down

Step 3: Is the scheduler running?
  kubectl -n kube-system get pods | grep scheduler
  sudo crictl ps -a | grep kube-scheduler
  sudo tail -n 50 /var/log/containers/kube-scheduler-*.log

Step 4: Common scheduler fixes
  Bad kubeconfig: grep 'kubeconfig' /etc/kubernetes/manifests/kube-scheduler.yaml
    → should be /etc/kubernetes/scheduler.conf
  secure-port=0: grep 'secure-port' /etc/kubernetes/manifests/kube-scheduler.yaml
    → restore --secure-port=10259
```

---

### Path 4: Cert Expiry

```
Step 1: Quick diagnosis
  sudo kubeadm certs check-expiration
  openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -enddate
  journalctl -u kubelet -p err --since "2h" | grep -i x509

Step 2: Fix (clock is correct)
  sudo kubeadm certs renew all
  sudo kubeadm init phase kubeconfig all
  sudo systemctl restart kubelet
  mkdir -p ~/.kube && sudo cp -f /etc/kubernetes/admin.conf ~/.kube/config
  sudo chown $(id -u):$(id -g) ~/.kube/config

Step 3: After renewal, if components still NotReady
  → Stale containerd containers?
  sudo crictl ps -a | grep -E 'controller|scheduler'
  sudo crictl rm -f <dead-ids>
  sudo systemctl restart kubelet

Step 4: Verify
  kubectl get nodes
  sudo kubeadm certs check-expiration
  kubectl -n kube-system get lease
```

---

## 9. Kill Switch

> *The absolute minimum to hold in memory. Read this 30 seconds before the exam.*

```
THE 6 RULES:
1. Can't reach kubectl → ss -lntp | grep 6443 (is it listening?)
2. Crash-looping       → crictl ps -a → tail /var/log/containers/<name>-*.log
3. Fix anything        → edit /etc/kubernetes/manifests/ → kubelet reconciles
4. etcd broken?        → check BEFORE blaming apiserver
5. Certs expired?      → kubeadm certs renew all → kubeadm init phase kubeconfig all → restart kubelet
6. Stale containers?   → crictl rm -f <id> → restart kubelet

THE PORT MAP:
6443=apiserver  2379=etcd-client  2380=etcd-peer  10257=CM  10259=scheduler

CERT RENEWAL FLOW (order matters):
kubeadm certs renew all → kubeadm init phase kubeconfig all → systemctl restart kubelet → cp admin.conf

CRITICAL GOTCHA:
Renew while clock is wrong → certs get wrong NotBefore → reset clock → renew AGAIN

"name is reserved for <ID>" error → crictl rm -f old containers → restart kubelet
```

---

## 10. Appendix

### Quick Reference: Component → Manifest → Kubeconfig → Port

| Component | Manifest | Kubeconfig | Health Port |
|-----------|----------|-----------|-------------|
| kube-apiserver | `/etc/kubernetes/manifests/kube-apiserver.yaml` | `/etc/kubernetes/admin.conf` | 6443 |
| etcd | `/etc/kubernetes/manifests/etcd.yaml` | N/A | 2379 (client) |
| kube-controller-manager | `/etc/kubernetes/manifests/kube-controller-manager.yaml` | `/etc/kubernetes/controller-manager.conf` | 10257 |
| kube-scheduler | `/etc/kubernetes/manifests/kube-scheduler.yaml` | `/etc/kubernetes/scheduler.conf` | 10259 |

---

### Quick Reference: `ss` Flags

| Flag | Meaning |
|------|---------|
| `-l` | Listening sockets only |
| `-t` | TCP only |
| `-n` | Numeric (no DNS) |
| `-p` | Show process |
| `-s` | Summary |
| No `-l` | See established connections |

```bash
sudo ss -lntp               # All listening (your default)
sudo ss -lntp | grep 6443   # Is apiserver listening?
ss -tnp state established   # Who is connected right now
```

---

### Quick Reference: `crictl` Cheatsheet

```bash
sudo crictl ps -a                    # All containers (including stopped)
sudo crictl logs <id>                # Container logs
sudo crictl rm -f <id>               # Force remove container
sudo crictl pods                     # List pod sandboxes
sudo crictl stopp <sandbox-id>       # Stop sandbox
sudo crictl rmp <sandbox-id>         # Remove sandbox
```

---

### Quick Reference: etcd Environment Variables

```bash
export ETCDCTL_API=3
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key
```

---

### Quick Reference: Common Break/Fix Patterns

| What's broken | Symptom | Where to look | Fix |
|---------------|---------|---------------|-----|
| apiserver not listening | kubectl timeout | `ss -lntp \| grep 6443` | Fix manifest flags |
| etcd unreachable | apiserver crashloop, `context deadline exceeded` | etcd logs, `ss \| grep 2379` | Fix listen-client-urls |
| cert expired | x509 errors everywhere | `kubeadm certs check-expiration` | `kubeadm certs renew all` |
| scheduler down | pods Pending (no events) | `crictl ps -a \| grep scheduler` | Fix kubeconfig/secure-port in manifest |
| CM down | deployments not scaling | `crictl ps -a \| grep controller` | Fix kubeconfig/secure-port in manifest |
| stale container | "name is reserved" | `crictl ps -a` | `crictl rm -f <id>` + restart kubelet |
| cert NotBefore in future | x509 not yet valid | `openssl -enddate` | Reset clock → renew again |

---

### Quick Reference: Cert Locations

```
/etc/kubernetes/pki/
├── ca.crt / ca.key                     ← Cluster CA
├── apiserver.crt / apiserver.key       ← API server TLS
├── apiserver-kubelet-client.crt/.key   ← apiserver→kubelet auth
├── apiserver-etcd-client.crt/.key      ← apiserver→etcd auth
├── front-proxy-ca.crt / .key
├── front-proxy-client.crt / .key
└── etcd/
    ├── ca.crt / ca.key                 ← etcd CA
    ├── server.crt / server.key         ← etcd server TLS
    └── peer.crt / peer.key             ← etcd peer TLS
```

---

### Scenario Summary Table

| Scenario | First Command | Root Cause Category | Fix Location |
|----------|---------------|--------------------|----|
| API server down | `ss -lntp \| grep 6443` | Manifest flags, certs, etcd | `/etc/kubernetes/manifests/kube-apiserver.yaml` |
| etcd cert/SAN issue | `crictl logs <etcd-id>` | x509 SAN mismatch | Reissue etcd certs with correct SANs |
| etcd data dir | `grep data-dir etcd.yaml` | Wrong path or no hostPath | Fix manifest volume + data-dir flag |
| Controller-manager recovery | `crictl ps -a \| grep controller` | kubeconfig/port/signing key | `/etc/kubernetes/manifests/kube-controller-manager.yaml` |
| Scheduler unreachable | `crictl ps -a \| grep scheduler` | kubeconfig/port | `/etc/kubernetes/manifests/kube-scheduler.yaml` |
| Expired certs | `kubeadm certs check-expiration` | NotAfter passed | `kubeadm certs renew all` + kubeconfig regen |

---

