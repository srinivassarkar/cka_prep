---

# 🚀 KUBERNETES SCHEDULING MASTER LAB

(NodeSelector → Required Affinity → Preferred → Operators → OR Logic → Debugging)

---

# 🔥 PHASE 0 — Reset Everything

```bash
kubectl delete deploy --all
kubectl delete pod --all
```

Check nodes:

```bash
kubectl get nodes -o wide
kubectl get nodes --show-labels
```

🧠 Study:

* Default labels exist (`kubernetes.io/*`)
* Never rely on dynamic labels in production design.

---

# 🔥 PHASE 1 — nodeSelector (Strict & Simple)

## 1️⃣ Label Nodes

```bash
kubectl label nodes worker1 storage=ssd
kubectl label nodes worker2 storage=hdd
```

Verify:

```bash
kubectl get nodes --show-labels | grep storage
```

---

## 2️⃣ Create nodeSelector Deployment

```bash
vi ns.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ns-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ns
  template:
    metadata:
      labels:
        app: ns
    spec:
      nodeSelector:
        storage: ssd
      containers:
      - name: nginx
        image: nginx
```

Apply:

```bash
kubectl apply -f ns.yaml
kubectl get pods -o wide
```

✔ All pods on worker1

---

## 3️⃣ AND Condition

Add:

```bash
kubectl label nodes worker1 env=prod
```

Modify:

```yaml
nodeSelector:
  storage: ssd
  env: prod
```

✔ Must match BOTH.

---

## 4️⃣ Break It

Remove label:

```bash
kubectl label nodes worker1 env-
```

Pods → Pending.

Check:

```bash
kubectl describe pod <podname>
```

You’ll see:

```
0/2 nodes available
```

---

## 🧠 nodeSelector Core Facts

* Hard rule
* Exact match only
* AND logic only
* No OR
* No weights
* Not retroactive
* Scheduler only checks during scheduling

---

# 🔥 PHASE 2 — Required Node Affinity

Delete:

```bash
kubectl delete deploy ns-deploy
```

Relabel:

```bash
kubectl label nodes worker1 storage=ssd
```

---

## 5️⃣ Basic Required Affinity

```bash
vi na.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: na-deploy
spec:
  replicas: 5
  selector:
    matchLabels:
      app: na
  template:
    metadata:
      labels:
        app: na
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: storage
                operator: In
                values:
                - ssd
      containers:
      - name: nginx
        image: nginx
```

✔ Same behavior as nodeSelector
But more expressive.

---

## 🧠 Required Affinity Deep Rules

Inside one `nodeSelectorTerms`:
→ matchExpressions = AND

Multiple `nodeSelectorTerms`:
→ OR

Not retroactive (IgnoredDuringExecution).

---

# 🔥 PHASE 3 — True OR Logic (Real Upgrade)

Label second node:

```bash
kubectl label nodes worker2 storage=hdd
```

Modify values:

```yaml
values:
- ssd
- hdd
```

Pods now distribute across both nodes.

---

## 🔥 REAL OR (Multiple Terms)

```yaml
nodeSelectorTerms:
- matchExpressions:
  - key: zone
    operator: In
    values:
    - east
- matchExpressions:
  - key: env
    operator: In
    values:
    - dev
```

Meaning:

```
(zone=east) OR (env=dev)
```

This is commonly misunderstood in interviews.

---

# 🔥 PHASE 4 — Gt / Lt Operator (Integer Comparison)

Clean:

```bash
kubectl delete deploy --all
```

Add numeric labels:

```bash
kubectl label nodes worker1 cpu=8
kubectl label nodes worker2 cpu=2
```

---

## Gt Example

```yaml
- key: cpu
  operator: Gt
  values:
  - "4"
```

✔ Only worker1

---

## Lt Example

```yaml
- key: cpu
  operator: Lt
  values:
  - "4"
```

✔ Only worker2

---

## 🧠 Gt/Lt Rules

* Works only in affinity
* Label must be numeric string
* If non-numeric → rule fails → Pending

Exam trap area.

---

# 🔥 PHASE 5 — NotIn / DoesNotExist (Anti-Placement)

Label:

```bash
kubectl label nodes worker1 env=prod
kubectl label nodes worker2 env=dev
```

Block dev:

```yaml
- key: env
  operator: NotIn
  values:
  - dev
```

✔ Avoids worker2

---

DoesNotExist:

```yaml
- key: dedicated
  operator: DoesNotExist
```

✔ Avoid nodes having that key.

---

## 🧠 Operator Memory Table

| Operator     | Meaning                |
| ------------ | ---------------------- |
| In           | Must match values      |
| NotIn        | Must NOT match values  |
| Exists       | Key must exist         |
| DoesNotExist | Key must NOT exist     |
| Gt           | Greater than (numeric) |
| Lt           | Less than (numeric)    |

---

# 🔥 PHASE 6 — Preferred Affinity (Soft + Weight)

Delete old:

```bash
kubectl delete deploy --all
```

Create:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: storage
          operator: In
          values:
          - ssd
          - hdd
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 10
      preference:
        matchExpressions:
        - key: storage
          operator: In
          values:
          - ssd
    - weight: 5
      preference:
        matchExpressions:
        - key: storage
          operator: In
          values:
          - hdd
```

✔ More pods on SSD.

---

## 🧠 Scheduler Internal Mechanics

Scheduler Phases:

1️⃣ Filter phase
→ Remove nodes failing required rules

2️⃣ Score phase
→ Add weight for each preferred match

3️⃣ Sum weights

4️⃣ Highest score wins

⚠ But scoring also includes:

* CPU
* Memory
* Taints
* Other plugins

Affinity is one plugin in scoring stack.

---

# 🔥 PHASE 7 — Debugging Like a Senior

If pod stuck:

```bash
kubectl describe pod <pod>
```

Look for:

```
0/2 nodes available
```

More detailed:

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

You’ll see exact reason:

* Node didn't match affinity
* Insufficient CPU
* Node had taint

This is real production debugging skill.

---

# 🚨 CRITICAL DIFFERENCE TABLE

| Feature     | nodeSelector | requiredAffinity | preferredAffinity |
| ----------- | ------------ | ---------------- | ----------------- |
| Hard rule   | ✅            | ✅                | ❌                 |
| OR logic    | ❌            | ✅                | ✅                 |
| AND logic   | ✅            | ✅                | ✅                 |
| Weight      | ❌            | ❌                | ✅                 |
| Gt/Lt       | ❌            | ✅                | ✅                 |
| Retroactive | ❌            | ❌                | ❌                 |

Only **NoExecute taint** causes eviction.

---

# 🧠 PRODUCTION DESIGN PATTERNS

Dedicated hardware
→ Taint + Required Affinity

Soft preference
→ Preferred only

Environment isolation
→ Required affinity + namespace strategy

Avoid dev nodes
→ NotIn

Numeric scaling (large nodes only)
→ Gt operator

---

# 🧠 MASTER MENTAL MODEL

Taints → repel
nodeSelector → strict match
requiredAffinity → strict + logic
preferredAffinity → scoring preference

Scheduler = Filter → Score → Bind

Retroactive?
Only NoExecute taint.

---
