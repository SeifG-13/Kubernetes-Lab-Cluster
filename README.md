# Kubernetes Lab Cluster

A 3-node Kubernetes cluster built from scratch on a Windows 10 laptop using VirtualBox, Vagrant, and Ansible. Built for hands-on learning and interview preparation.

---

## Stack

| Tool | Version |
|------|---------|
| Kubernetes | v1.30.14 |
| containerd | v1.7.29 |
| Calico CNI | v3.28.0 |
| Ansible | 2.17.14 |
| Helm | 3.20.0 |
| Vagrant | 2.4.9 |
| VirtualBox | 7.2.2 |

---

## Cluster Architecture

| VM | Hostname | IP | Role | RAM | CPUs |
|----|----------|----|------|-----|------|
| 1 | master1 | 192.168.56.20 | Control plane | 2GB | 2 |
| 2 | worker1 | 192.168.56.30 | Worker | 2GB | 2 |
| 3 | worker2 | 192.168.56.31 | Worker | 2GB | 2 |

---

## Prerequisites

- Windows 10 with **Hyper-V disabled** (`bcdedit /set hypervisorlaunchtype off` + reboot)
- VirtualBox 7.x
- Vagrant 2.4.x

---

## Quick Start

```bash
# 1. Boot all VMs (installs Ansible on master1, distributes SSH keys)
vagrant up

# 2. SSH into master1
vagrant ssh master1

# 3. Build the cluster (runs all 6 playbooks)
export ANSIBLE_CONFIG=/vagrant/ansible/ansible.cfg
cd /vagrant/ansible
ansible-playbook site.yml

# 4. Verify
kubectl get nodes -o wide
```

---

## Project Structure

```
.
├── Vagrantfile                    # VM definitions (3 nodes)
├── ansible/
│   ├── ansible.cfg                # Ansible config
│   ├── inventory.ini              # Host groups: masters, workers
│   ├── site.yml                   # Master playbook (imports all)
│   ├── 01-common.yml              # Swap, kernel modules, sysctl
│   ├── 02-containerd.yml          # Container runtime (1.7.x)
│   ├── 03-kubeadm.yml             # kubeadm, kubelet, kubectl
│   ├── 04-init-master.yml         # kubeadm init on master1
│   ├── 05-calico.yml              # Calico CNI
│   └── 06-join-workers.yml        # Join worker nodes
└── manifests/
    ├── nginx-deployment.yaml      # Deployment, rolling update, rollback
    ├── configmap-demo.yaml        # ConfigMap + Secret injection
    ├── rbac-demo.yaml             # ServiceAccount, Role, RoleBinding
    ├── probes-demo.yaml           # Liveness, readiness, startup probes
    ├── qos-demo.yaml              # Guaranteed, Burstable, BestEffort
    ├── daemonset-demo.yaml        # DaemonSet with control-plane toleration
    ├── ingress-demo.yaml          # NGINX Ingress, path-based routing
    ├── storage-demo.yaml          # PersistentVolume + PVC
    ├── statefulset-demo.yaml      # StatefulSet + headless Service
    ├── statefulset-pvs.yaml       # hostPath PVs for StatefulSet
    ├── jobs-demo.yaml             # Job (parallel) + CronJob
    └── kube-prometheus-values.yaml # Helm values for Prometheus + Grafana
```

---

## Cluster Lifecycle

### Shutdown (graceful)
```bash
# 1. Drain workers (on master1)
kubectl drain worker1 --ignore-daemonsets --delete-emptydir-data
kubectl drain worker2 --ignore-daemonsets --delete-emptydir-data

# 2. Halt VMs (from Windows project folder)
vagrant halt worker1
vagrant halt worker2
vagrant halt master1

# 3. Verify
vagrant status   # all 3 should show: poweroff
```

### Resume
```bash
# Boot all VMs — cluster resumes, no need to re-run Ansible
vagrant up
vagrant ssh master1
kubectl get nodes
```

### Snapshots (recommended after first build)
```bash
vagrant snapshot save master1 lab-complete
vagrant snapshot save worker1 lab-complete
vagrant snapshot save worker2 lab-complete

# Restore
vagrant snapshot restore master1 lab-complete
```

---

## Accessing the Cluster from Windows

```powershell
# Install kubectl
winget install Kubernetes.kubectl

# Point to cluster
$env:KUBECONFIG = "C:\Users\SEIF\Desktop\k8s cluster\admin.conf"
kubectl get nodes
```

---

## Monitoring (Prometheus + Grafana)

```bash
# Install via Helm (run on master1)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kube-prom prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --values /vagrant/manifests/kube-prometheus-values.yaml

# Grafana UI
# http://192.168.56.30:30300  —  admin / admin123
# Add Prometheus datasource: http://kube-prom-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090
# Recommended dashboards: 1860 (Node Exporter Full), 15661 (K8s Global)
```

---

## Validation Exercises Completed

| # | Topic | Concepts |
|---|-------|---------|
| 10.1 | Deployment | Rolling update, rollback, ReplicaSet |
| 10.2 | Services | ClusterIP, NodePort, Endpoints |
| 10.3 | Ingress | NGINX controller, path-based routing |
| 10.4 | Storage | PV, PVC, Retain policy, hostPath |
| 10.5 | StatefulSet | Stable DNS, per-pod PVC, ordered startup |
| 10.6 | DaemonSet | One pod per node, tolerations |
| 10.7 | Jobs + CronJobs | completions, parallelism, concurrencyPolicy |
| 10.8 | ConfigMaps + Secrets | Env var + volume injection |
| 10.9 | RBAC | Role, ClusterRole, RoleBinding, can-i |
| 10.10 | Probes | Liveness, readiness, startup |
| 10.11 | QoS | Guaranteed, Burstable, BestEffort |
| 10.12 | Metrics Server | kubectl top nodes/pods |
| 10.13 | Prometheus + Grafana | Helm, PromQL, Node Exporter dashboards |

---

## Key Issues Resolved

| Issue | Fix |
|-------|-----|
| VM 40x slowdown (WSL2/Hyper-V conflict) | `bcdedit /set hypervisorlaunchtype off` + reboot |
| Kernel panic on boot | Increased RAM to 2GB per VM |
| ansible.cfg ignored (world-writable /vagrant) | `export ANSIBLE_CONFIG=/vagrant/ansible/ansible.cfg` |
| containerd 2.x incompatible with K8s 1.30 | Pinned to `containerd.io=1.7.*` |
| StatefulSet PVCs stuck Pending | Created hostPath PVs manually (no dynamic provisioner) |
