# **Deployment Hands-On (Killercoda | Latest kubeadm)**

## **0. Sanity Check (Always First)**

```bash
kubectl get nodes
kubectl get pods -A
```

Expected:

* 2 nodes (controlplane + worker)
* controlplane **schedulable** (taint removed already)

---

## **1. Create Deployment (Declarative – Exam Safe)**

### Create manifest

```bash
vi nginx-deploy.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  annotations:
    kubernetes.io/change-cause: "Initial nginx deployment"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.19
```

Apply:

```bash
kubectl apply -f nginx-deploy.yaml
```

---

## **2. Verify Deployment State**

```bash
kubectl get deployments
kubectl get rs
kubectl get pods -o wide
```

**What you should see**

* 1 Deployment
* 1 ReplicaSet
* 3 Pods spread across nodes (scheduler dependent)

---

## **3. Inspect Deep (CKA Habit)**

```bash
kubectl describe deployment nginx-deployment
```

Focus areas:

* Replicas: 3 desired / 3 available
* Strategy: RollingUpdate
* Annotation: change-cause present

---

## **4. Rollout History (Critical)**

```bash
kubectl rollout history deployment nginx-deployment
```

Expected:

```
REVISION  CHANGE-CAUSE
1         Initial nginx deployment
```

---

## **5. Update Image (Rolling Update)**

Imperative (allowed in exam):

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.20
```

Annotate (important):

```bash
kubectl annotate deployment nginx-deployment \
kubernetes.io/change-cause="Upgrade nginx 1.19 → 1.20"
```

Watch rollout live:

```bash
kubectl rollout status deployment nginx-deployment
```

---

## **6. Verify New ReplicaSet**

```bash
kubectl get rs
```

You should see:

* Old RS → replicas **0**
* New RS → replicas **3**

History:

```bash
kubectl rollout history deployment nginx-deployment
```

---

## **7. Upgrade Again (Create More Revisions)**

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.21
kubectl annotate deployment nginx-deployment \
kubernetes.io/change-cause="Upgrade nginx 1.20 → 1.21"
```

Verify:

```bash
kubectl rollout history deployment nginx-deployment
```

---

## **8. Rollback (THIS IS EXAM MONEY)**

Rollback to previous:

```bash
kubectl rollout undo deployment nginx-deployment
```

Rollback to specific revision:

```bash
kubectl rollout undo deployment nginx-deployment --to-revision=1
```

Verify:

```bash
kubectl describe deployment nginx-deployment
kubectl rollout history deployment nginx-deployment
```

---

## **9. Scale Deployment**

```bash
kubectl scale deployment nginx-deployment --replicas=5
kubectl get pods
```

Deployment handles scaling — **never scale RS directly**.

---

## **10. Pause & Resume (Advanced Control)**

Pause:

```bash
kubectl rollout pause deployment nginx-deployment
```

Make multiple changes safely:

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.22
kubectl scale deployment nginx-deployment --replicas=4
```

Resume rollout:

```bash
kubectl rollout resume deployment nginx-deployment
```

---

## **11. Edit Live (Emergency Fix Mode)**

```bash
kubectl edit deployment nginx-deployment
```

* Useful when YAML file is missing
* Exam-valid but risky → be precise

---

## **12. Export YAML (VERY IMPORTANT)**

```bash
kubectl get deployment nginx-deployment -o yaml
```

Used to:

* Recreate resources
* Understand defaults
* Debug exam questions

---

## **13. Cleanup (Always Clean Playground)**

```bash
kubectl delete deployment nginx-deployment
```

---

# **Mental Model Lock-In**

* Deployment **creates & owns ReplicaSets**
* ReplicaSet **creates & owns Pods**
* Every image change = **new ReplicaSet**
* Rollback = **RS switch**
* Annotation = **human memory**

---
