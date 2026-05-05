# Setup Manual

## KIND (Local)

**Requirements:** Docker, kubectl, kind

```bash
# Install kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x kind && mv kind /usr/local/bin/

# Create cluster
kind create cluster --name k8s-labs

# Verify
kubectl cluster-info
kubectl get nodes
# kind-control-plane   Ready   1m
```

Set default namespace per lab:
```bash
kubectl config set-context --current --namespace=lab01
```

---

## Killercoda

**URL:** https://killercoda.com/playgrounds/scenario/kubernetes

- Click **Start** — cluster ready in 60 seconds
- 2 nodes: `controlplane` + `node01`
- Calico CNI — NetworkPolicy enforced (Lab 14 works here)
- Session expires in 60 minutes — save your YAMLs

---

## Which to Use

| Lab | KIND | Killercoda |
|-----|------|------------|
| 01–13, 15 | ✓ | ✓ |
| 14 (NetworkPolicy) | ✗ kindnet | ✓ Calico |

---

## Reset Between Labs

```bash
# KIND
kubectl delete namespace lab0X
kubectl create namespace lab0X

# Killercoda — just refresh the browser
```