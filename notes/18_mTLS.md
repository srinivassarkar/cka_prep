# 🧠 CORE IDEA 

👉 Kubernetes TLS is NOT about paths
👉 It’s about answering 3 questions every time:

```
1. Who is CLIENT?
2. Who is SERVER?
3. Who TRUSTS whom (CA)?
```

If you can answer this → you can debug ANY TLS issue in K8s.

---

# 🔥 PART 1 — PRIVATE CA (WHY IT EXISTS)

### Reality:

* K8s uses **PRIVATE CA internally**
* Not public (no Let's Encrypt inside cluster)

### Why?

```
Internal trust → controlled
External trust → public CA
```

### Pro-level insight:

* Most clusters use:

  * 1 CA (simple)
  * OR separate CA for etcd (secure)

👉 WHY separate etcd CA?

```
etcd = brain of cluster
If compromised → full cluster gone

So isolate trust boundary
```

---

# 🧠 PART 2 — CLIENT vs SERVER (MOST IMPORTANT)

### GOLD RULE:

👉 **Who sends request = CLIENT**
👉 **Who responds = SERVER**

---

### Reality mapping:

| Component          | Role            |
| ------------------ | --------------- |
| kubectl            | Client          |
| Scheduler          | Client          |
| Controller Manager | Client          |
| API Server         | Client + Server |
| Kubelet            | Client + Server |
| etcd               | ALWAYS Server   |

---

### 🔥 Trick (INTERVIEW GOLD):

👉 API Server changes role based on flow

Example:

```
kubectl → API server
API server = SERVER

API server → kubelet
API server = CLIENT
```

---

# 🧠 PART 3 — HOW TLS ACTUALLY WORKS (REAL MODEL)

Forget theory. Think like this:

```
CLIENT → verifies SERVER cert
SERVER → verifies CLIENT cert (mTLS)
```

Where is everything stored?

```
CLIENT → kubeconfig file
SERVER → manifest file
```

---

# ⚡ MASTER RULE (BIGGEST TAKEAWAY)

👉 **Don’t memorize paths**
👉 **Read config files**

```
Server config → manifests
Client config → kubeconfig
```

---

# 🔥 PART 4 — 3 CRITICAL FLOWS (THIS IS THE GAME)

---

## 🔹 FLOW 1: Scheduler → API Server (PURE mTLS)

### Flow:

```
Scheduler (client) → API Server (server)
```

---

### SERVER SIDE (API SERVER)

From manifest:

```
--tls-cert-file=apiserver.crt
```

👉 This is what API server shows

---

### CLIENT SIDE (Scheduler)

From kubeconfig:

```
certificate-authority-data → trust CA
client-certificate-data → identity
client-key-data → private key
```

---

### FINAL FLOW:

```
Scheduler verifies API server (via CA)
API server verifies Scheduler (via CA)

= TRUE mTLS
```

---

## 🔹 FLOW 2: API Server → etcd (STRICT mTLS)

### Flow:

```
API Server (client) → etcd (server)
```

---

### SERVER (etcd)

```
--cert-file=server.crt
--trusted-ca-file=etcd-ca.crt
```

---

### CLIENT (API Server)

```
--etcd-certfile
--etcd-keyfile
--etcd-cafile
```

---

### FINAL:

```
API server verifies etcd
etcd verifies API server

= STRICT mTLS (VERY IMPORTANT)
```

👉 This is the **most secure connection in cluster**

---

## 🔹 FLOW 3: API Server → Kubelet (TRICKY ⚠️)

### Flow:

```
API Server (client) → Kubelet (server)
```

---

### REALITY (VERY IMPORTANT):

👉 NOT FULL mTLS

---

### What happens:

```
Kubelet verifies API server ✔️
API server DOES NOT strictly verify kubelet ❌
```

---

### Why?

Because:

```
Kubelet already trusted via:
- bootstrap token
- CSR approval
```

👉 So Kubernetes uses:

```
TLS + RBAC (not strict cert validation)
```

---

### 🔥 INTERVIEW LINE:

👉 “This is asymmetric TLS, not full mTLS”

---

# 🧠 PART 5 — KUBELET MAGIC (BOOTSTRAP)

---

### How kubelet gets cert?

```
1. Uses bootstrap token
2. Sends CSR
3. API server signs it
4. Gets kubelet.crt
```

---

# ⚡ CSR + USER ACCESS (REAL WORLD)

---

# 🧠 BIG IDEA:

```
Authentication = Certificate
Authorization = RBAC
```

---

# 🔥 COMPLETE FLOW (REMEMBER THIS)

---

## STEP 1 — User creates key

```
seema.key
```

---

## STEP 2 — Create CSR

```
CN=seema  ← THIS = USERNAME
```

---

## STEP 3 — Admin creates CSR object

```
kind: CertificateSigningRequest
```

---

## STEP 4 — Admin approves

```
kubectl certificate approve seema
```

---

## STEP 5 — Get certificate

```
seema.crt
```

---

## STEP 6 — Configure kubeconfig

Now user can talk to API server

---

## STEP 7 — RBAC (VERY IMPORTANT)

```
Auth → certificate
Access → RBAC
```

---

## FINAL FLOW:

```
User → proves identity via cert
API server → checks RBAC
```

---

# 🔥 REAL WORLD MAPPING

| Layer    | Purpose         |
| -------- | --------------- |
| TLS Cert | Who are you     |
| RBAC     | What can you do |

---

# ⚠️ MOST COMMON CONFUSION (NOW CLEAR)

### ❌ Wrong thinking:

“TLS gives access”

### ✅ Correct:

```
TLS = authentication only
RBAC = authorization
```

---

# 🧪 LABS (THIS WILL MAKE YOU 10x BETTER)

---

# 🔥 LAB 1 — TRACE ANY TLS FLOW (MUST DO)

### Goal:

Understand like a pro (not memorization)

---

### Step:

```
cd /etc/kubernetes/manifests
cat kube-apiserver.yaml
```

Find:

```
--client-ca-file
--etcd-*
--kubelet-client-*
```

---

### Then:

```
cat /etc/kubernetes/scheduler.conf
```

---

### Then decode cert:

```
openssl x509 -in <cert> -text -noout
```

---

### Observe:

* CN (identity)
* Issuer (CA)

---

👉 RESULT:

You’ll literally SEE trust chain

---

# 🔥 LAB 2 — DEBUG mTLS YOURSELF

Break something:

```
mv /etc/kubernetes/pki/ca.crt /tmp/
```

Then:

```
kubectl get pods
```

---

### Observe:

* TLS errors
* handshake failure

---

👉 This builds **real troubleshooting skill**

---

# 🔥 LAB 3 — CSR FLOW (MOST IMPORTANT)

---

### Create user:

```
openssl genrsa -out dev.key 2048
openssl req -new -key dev.key -out dev.csr -subj "/CN=dev"
```

---

### Create CSR object

Apply YAML

---

### Approve:

```
kubectl certificate approve dev
```

---

### Extract cert:

```
kubectl get csr dev -o jsonpath='{.status.certificate}' | base64 -d > dev.crt
```

---

### Test:

```
kubectl auth can-i get pods --as=dev
```

---

👉 THIS is real DevOps work

---

# 🔥 LAB 4 — KUBELET DEEP DIVE (ADVANCED)

---

```
ps -ef | grep kubelet
```

Find config:

```
/var/lib/kubelet/config.yaml
```

---

Then:

```
cd /var/lib/kubelet/pki
ls
```

---

Inspect:

```
openssl x509 -in kubelet.crt -text -noout
```

---

👉 Observe:

* CN=system:node:<node>
* Issuer CA

---
EXTRAS
---

## 🔥 1. Break your cluster (MANDATORY)

Do:

```bash
mv /etc/kubernetes/pki/apiserver.crt /tmp/
```

Then:

```bash
kubectl get pods
```

👉 Now FIX it

---

## 🔥 2. Expire a cert (simulate real world)

```bash
openssl x509 -in apiserver.crt -noout -dates
```

Understand:

* expiry
* renewal
* impact

---

## 🔥 3. Debug kubelet failure

Stop kubelet:

```bash
systemctl stop kubelet
```

Start again → check logs:

```bash
journalctl -u kubelet -f
```

👉 Watch TLS errors live


---
# 🧠 FINAL MENTAL MODEL (NEVER FORGET)

```
1. Identify CLIENT
2. Identify SERVER
3. Find config (manifest / kubeconfig)
4. Find:
   - cert
   - key
   - CA
5. Verify trust chain
```

---
