# K8s Cluster Build Log

**Hardware:** 16GB RAM | i7-10th gen | Windows 10 Home
**Tools:** VirtualBox 7.2.2 | Vagrant 2.4.9 | Ansible 2.17.14 | Helm 3.20.0
**Cluster:** 3-VM Kubernetes on 192.168.56.0/24
**Date:** 2026-03-05

---

## Cluster Architecture

| VM | Hostname | IP | Role | RAM | CPUs |
|----|----------|----|------|-----|------|
| 1 | master1 | 192.168.56.20 | Control plane | 2GB | 2 |
| 2 | worker1 | 192.168.56.30 | Worker | 2GB | 2 |
| 3 | worker2 | 192.168.56.31 | Worker | 2GB | 2 |

**Kubernetes:** v1.30.14
**containerd:** v1.7.29
**CNI:** Calico v3.28.0 (pod CIDR: 192.168.0.0/16)
**Ansible:** 2.17.14 (runs from master1)

---

## Phase Status

| Phase | Description | Status |
|-------|-------------|--------|
| 0 | Prerequisites + Vagrantfile | ✅ DONE |
| 1 | VM Provisioning (vagrant up) | ✅ DONE |
| 2 | Common prereqs (swap/modules/sysctl) | ✅ DONE |
| 3 | containerd 1.7.29 | ✅ DONE |
| 4 | kubeadm/kubelet/kubectl | ✅ DONE |
| 5 | kubeadm init (master1) | ✅ DONE |
| 6 | Calico CNI | ✅ DONE |
| 7 | Join workers | ✅ DONE |
| 10.1 | Deployment + rolling update + rollback | ✅ DONE |
| 10.2 | Services (ClusterIP, NodePort) | ✅ DONE |
| 10.3 | Ingress (NGINX controller) | ✅ DONE |
| 10.4 | Storage (hostPath PV/PVC) | ✅ DONE |
| 10.5 | StatefulSet (headless svc, per-pod PVC) | ✅ DONE |
| 10.6 | DaemonSet (with control-plane toleration) | ✅ DONE |
| 10.7 | Jobs + CronJobs | ✅ DONE |
| 10.8 | ConfigMaps + Secrets | ✅ DONE |
| 10.9 | RBAC + Namespaces | ✅ DONE |
| 10.10 | Probes (liveness, readiness, startup) | ✅ DONE |
| 10.11 | Resource requests/limits + QoS classes | ✅ DONE |
| 10.12 | Metrics Server + kubectl top | ✅ DONE |
| 10.13 | Prometheus + Grafana (Helm) | ✅ DONE |

---

## Verified Cluster State

```
NAME      STATUS   ROLES           VERSION    INTERNAL-IP     CONTAINER-RUNTIME
master1   Ready    control-plane   v1.30.14   192.168.56.20   containerd://1.7.29
worker1   Ready    worker          v1.30.14   192.168.56.30   containerd://1.7.29
worker2   Ready    worker          v1.30.14   192.168.56.31   containerd://1.7.29
```

---

## Issues Resolved During Build

| Issue | Cause | Fix |
|-------|-------|-----|
| Kernel panic on boot | 384MB RAM — Ubuntu 22.04 min is ~512MB | Set all VMs to 2GB |
| VM 40x slowdown | WSL2 forces VirtualBox into NEM mode | bcdedit /set hypervisorlaunchtype off + reboot |
| ansible.cfg ignored | /vagrant is world-writable | `export ANSIBLE_CONFIG=/vagrant/ansible/ansible.cfg` |
| kubeadm `--version` flag | kubeadm uses `version` subcommand, not `--version` | Fixed in playbook |
| CRI ImageService error | containerd 2.x changed plugin API | Pinned to containerd 1.7.x |
| StatefulSet PVCs Pending | No dynamic provisioner on bare-metal | Created 3 hostPath PVs manually |
| Ingress 404 from Windows | curl missing Host header | Added `-H "Host: demo.local"` |
| Grafana no datasource | Grafana 11 sidecar incompatibility | Added Prometheus datasource manually via UI |

---

## Quick Reference

```bash
# SSH into cluster
vagrant ssh master1

# Run Ansible (always set this first)
export ANSIBLE_CONFIG=/vagrant/ansible/ansible.cfg
cd /vagrant/ansible

# Cluster health
kubectl get nodes -o wide
kubectl get pods -A
kubectl cluster-info

# Access cluster from Windows
$env:KUBECONFIG = "C:\Users\SEIF\Desktop\k8s cluster\admin.conf"
kubectl get nodes

# Snapshots
vagrant snapshot save master1 cluster-ready
vagrant snapshot save worker1 cluster-ready
vagrant snapshot save worker2 cluster-ready

# Prometheus + Grafana (Helm)
helm install kube-prom prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --values /vagrant/manifests/kube-prometheus-values.yaml
# Grafana: http://192.168.56.30:30300  admin/admin123
```

---

## Validation Exercises Completed

| Exercise | Key Concept Proven |
|----------|--------------------|
| Deployment | Rolling update (maxSurge/maxUnavailable), rollback via ReplicaSet |
| Services | ClusterIP (internal VIP), NodePort (external), Endpoints object |
| Ingress | NGINX controller, path-based routing, Host header requirement |
| Storage | PV/PVC lifecycle, Retain policy, data survives pod deletion |
| StatefulSet | Stable pod names, headless DNS, per-pod PVC, ordered startup/shutdown |
| DaemonSet | One pod per node, toleration to include control-plane |
| Jobs | completions + parallelism, restartPolicy: Never vs OnFailure |
| CronJob | Scheduled Jobs, concurrencyPolicy: Forbid, history limits |
| ConfigMap | Env var injection + volume mount (file) injection |
| Secrets | base64 encoding (not encryption), same injection methods as ConfigMap |
| RBAC | Role (namespace-scoped), ClusterRole (cluster-wide), RoleBinding, can-i |
| Probes | Liveness restarts unhealthy pods, readiness gates traffic, startup protects slow init |
| QoS | Guaranteed (requests=limits), Burstable (partial), BestEffort (none) |
| Metrics Server | Scrapes kubelet, powers kubectl top nodes/pods |
| Prometheus+Grafana | Helm deploy, Node Exporter Full dashboard, PromQL datasource |
