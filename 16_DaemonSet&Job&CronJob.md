# DaemonSet, Job & CronJob — CKA + Real-World Mastery (PRO)

---

# 🔥 1. Core Intuition (FIRST PRINCIPLES)

## 🧠 Controllers Mental Model

* **Deployment** → long-running apps
* **DaemonSet** → 1 pod per node
* **Job** → run once → complete
* **CronJob** → schedule Jobs

👉 Think:

* Deployment = app
* DaemonSet = infra agent
* Job = script
* CronJob = scheduler

---

# ⚡ 2. DaemonSet (Node-Level Brain)

## 💡 What it really does

* 1 pod per node
* New node → pod auto-created
* Node removed → pod deleted
* Pod deleted → recreated

---

## 🧪 Real Use Cases (INTERVIEW GOLD)

* Logging → FluentBit / Filebeat
* Monitoring → Node Exporter
* Networking → CNI (Calico, Cilium)
* Security → Falco

👉 Rule: **Runs everywhere = DaemonSet**

---

## ⚠️ Control Plane Trick

```bash
node-role.kubernetes.io/control-plane:NoSchedule
```

```yaml
tolerations:
- operator: Exists
```

👉 Means: run on **ALL nodes (ignore taints)**

---

## 🚀 Advanced Scheduling (MISSING PIECE 🔥)

### Run only on specific nodes

```yaml
nodeSelector:
  disktype: ssd
```

### Better: Node Affinity

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: gpu
          operator: Exists
```

👉 Interview Q:

> “Run DS only on GPU nodes”

---

## 🧾 DaemonSet Manifest

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
  namespace: logging-ns
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      tolerations:
      - operator: Exists
      containers:
      - name: log-collector
        image: busybox
        command: ["/bin/sh","-c","while true; do echo logs; sleep 30; done"]
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

---

## ▶️ Commands

```bash
kubectl create ns logging-ns
kubectl config set-context --current --namespace=logging-ns

kubectl apply -f ds.yaml

kubectl get ds
kubectl get pods -o wide
kubectl describe ds log-collector
```

---

## 🔍 What to Observe

* Pods = number of nodes
* Each pod on different node
* Control-plane included

---

## 🧨 Troubleshooting

| Issue                | Reason         | Fix         |
| -------------------- | -------------- | ----------- |
| Pod missing          | Node NotReady  | check nodes |
| Not on control-plane | no toleration  | add it      |
| CrashLoopBackOff     | bad command    | logs        |
| Volume fail          | hostPath issue | verify path |

---

# ⚡ 3. Job (Run → Finish → Exit)

## 💡 What it really does

* Runs task until success
* Retries on failure
* Stops after completion

---

## 🚀 Advanced Fields (IMPORTANT 🔥)

```yaml
activeDeadlineSeconds: 100
backoffLimit: 4
ttlSecondsAfterFinished: 60
```

👉 Controls:

* max runtime
* retry count
* cleanup

---

## 🧾 Job Manifest

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
spec:
  completions: 2
  parallelism: 2
  backoffLimit: 4
  ttlSecondsAfterFinished: 60
  template:
    spec:
      containers:
      - name: hello
        image: busybox
        command: ["bin/sh","-c","echo hello && sleep 10"]
      restartPolicy: Never
```

---

## ▶️ Commands

```bash
kubectl apply -f job.yaml

kubectl get jobs
kubectl get pods
kubectl describe job hello-job
```

---

## 🔍 Observe

* Pods → Completed
* Parallel execution
* Job finishes

---

## ⚠️ Interview Trap

👉 kubelet vs Job controller

* kubelet → restarts container
* Job → creates new pod

---

## 🧨 Troubleshooting

| Issue          | Reason            | Fix    |
| -------------- | ----------------- | ------ |
| infinite retry | failing container | logs   |
| not finishing  | wrong command     | fix    |
| too many pods  | high parallelism  | reduce |

---

# ⏰ 4. CronJob (Scheduler Brain)

## 💡 What it really does

* CronJob → creates Job → creates Pods

---

## 🚀 Advanced Fields (CRITICAL 🔥)

```yaml
concurrencyPolicy: Forbid
successfulJobsHistoryLimit: 3
failedJobsHistoryLimit: 1
```

### Concurrency Policy

* Allow → default
* Forbid → skip if running
* Replace → kill old, run new

👉 Interview Q:

> “Previous job still running?”

---

## 🧾 CronJob Manifest

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cronjob
spec:
  schedule: "*/1 * * * *"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      completions: 2
      parallelism: 2
      backoffLimit: 4
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            command: ["bin/sh","-c","echo hello && sleep 10"]
          restartPolicy: Never
```

---

## ▶️ Commands

```bash
kubectl apply -f cronjob.yaml

kubectl get cronjob
kubectl get jobs
kubectl get pods
```

---

## 🔍 Observe

* New Job every minute
* Jobs create Pods

---

## 🧨 Troubleshooting

| Issue          | Reason     | Fix      |
| -------------- | ---------- | -------- |
| not triggering | bad cron   | validate |
| time mismatch  | UTC        | adjust   |
| too many jobs  | no cleanup | TTL      |

---

# 🧠 5. Cron Cheat Sheet

```
* * * * *
| | | | |
| | | | └ day
| | | └ month
| | └ day of month
| └ hour
└ minute
```

---

# ⚔️ 6. Final Comparison

| Resource   | Use         |
| ---------- | ----------- |
| Deployment | apps        |
| DaemonSet  | node agents |
| Job        | one-time    |
| CronJob    | scheduled   |

---

# 💣 7. KillerCoda Practice

* Deploy DS → add node → verify
* Delete pod → recreated
* Fail Job → observe retry
* CronJob → observe schedule

---

# 🧠 8. Interview Questions

### Q1

Why DaemonSet?
→ runs on every node

### Q2

Why restartPolicy: Never?
→ Job handles retry

### Q3

Job vs CronJob?
→ one-time vs scheduled

### Q4 (HARD)

Who retries failed pod?
→ Job controller

---

# 🚨 9. Production & Advanced Scenarios (GAME CHANGER)

## 🔍 Debugging Commands (MUST KNOW)

```bash
kubectl logs <pod>
kubectl describe pod <pod>
kubectl get events
kubectl get pods --watch
```

---

## 🧨 Scenario 1: CronJob not running

Check:

* schedule syntax
* controller logs
* events

---

## 🧨 Scenario 2: Job stuck

Check:

* pod logs
* backoffLimit
* image issue

---

## 🧨 Scenario 3: DS not on node

Check:

* taints
* tolerations
* node labels

---

## ⚙️ Resource Awareness

* New node → DS pod created
* But → scheduler checks CPU/memory

---

## 🏭 Real Production Patterns

| Use Case        | Resource  |
| --------------- | --------- |
| log shipping    | DaemonSet |
| metrics         | DaemonSet |
| DB backup       | Job       |
| nightly cleanup | CronJob   |

---

## 🧩 YAML Depth (MUST PRACTICE)

Add:

* env vars
* secrets
* configMaps
* PVC

---

# 🧩 Final Insight

* DaemonSet = infra layer
* Job = execution layer
* CronJob = automation layer

👉 Together = **production automation system**

---

