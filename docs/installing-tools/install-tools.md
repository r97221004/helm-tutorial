# Installing Tools

> Helm needs a cluster to install into and a `helm` binary to do it. Both come from the companion Ansible tutorial.

---

> ⚠️ **Installation lives in the companion Ansible tutorial.** This page only lists which playbooks to run, in which order, and how to confirm the result.

The `helm` CLI itself is just a binary, but most of what it does (`install`, `upgrade`, `rollback`, `list`) needs a Kubernetes cluster to talk to. So this course needs a single-node cluster, plus Helm and k9s, all installed with two playbooks.

### What you need

- **A control machine** with Ansible and a clone of [r97221004/ansible-tutorial](https://github.com/r97221004/ansible-tutorial) — see its [Install Ansible](https://github.com/r97221004/ansible-tutorial/blob/master/docs/getting-started/install-ansible.md).
- **A target Linux VM** (Ubuntu 22.04) reachable over SSH, with passwordless sudo — see [Switch to SSH Connection](https://github.com/r97221004/ansible-tutorial/blob/master/docs/fundamentals/ssh-connection.md). A comfortable size is **2 vCPU, 4 GB RAM, 20 GB disk**.
- **The VM in the inventory** — edit `ansible/inventory/azure.ini`. Despite the name, it works for any VM, cloud or local. What matters is that the host sits in the `[control]` group, which every playbook in this guide targets:

  ```ini
  [control]
  azure-vm ansible_host=<YOUR_VM_IP> ansible_user=<YOUR_USER> ansible_connection=ssh
  ```

Run the Ansible commands on this page from the root of the `ansible-tutorial` repo, since the inventory and playbook paths are relative to it. Check that Ansible can reach the VM:

```bash
ansible control -i ansible/inventory/azure.ini -m ping
```

### 1. Kubernetes — the kubeadm role

```bash
ansible-playbook -i ansible/inventory/azure.ini ansible/playbooks/install_kubeadm.yml
```

This builds a vanilla, single-node Kubernetes cluster (containerd, kubelet/kubeadm/kubectl, flannel) and copies the admin kubeconfig to `~/.kube/config`, the file Helm reads, as the [architecture page](../helm-intro/architecture.md) showed. Every step is explained in **[ansible-tutorial → kubeadm Role](https://github.com/r97221004/ansible-tutorial/blob/master/docs/roles/kubeadm.md)**, and what each component *is* in **[k8s-tutorial → Set Up a Cluster (kubeadm)](https://github.com/r97221004/k8s-tutorial/blob/master/docs/getting-started/setup-kubeadm.md)**.

> **Fast lane — k3s.** Swap `install_kubeadm.yml` for `install_k3s.yml`; everything else stays the same. k3s comes up faster but hides more; kubeadm stays the reference cluster for this guide. Comparison in [Choosing k3s vs kubeadm](https://github.com/r97221004/ansible-tutorial/blob/master/docs/roles/k3s-vs-kubeadm.md).

### 2. Helm and k9s — the kube_tools role

```bash
ansible-playbook -i ansible/inventory/azure.ini ansible/playbooks/install_kube_tools.yml
```

- **Helm** — required. The role installs the latest release, which is Helm 4. To pin a version, set `helm_version` in the inventory to a tag from [Helm's releases](https://github.com/helm/helm/releases), e.g. `v4.x.y`.
- **k9s** — optional but recommended: you'll use it to *watch* each release's objects appear, change on upgrade, and disappear on uninstall.

Details in **[ansible-tutorial → kube_tools Role](https://github.com/r97221004/ansible-tutorial/blob/master/docs/roles/kube-tools.md)**.

> **Prefer Helm on your laptop?** Run the same playbook with `-i ansible/inventory/localhost.ini` to install Helm and k9s on your own machine instead, then copy the VM's `~/.kube/config` to yours. Your laptop must be able to reach the API server on port 6443, at an address its certificate accepts, so this takes extra network setup; this guide runs everything on the VM to avoid it.

### Verify your setup

SSH into the VM and confirm:

```bash
kubectl get nodes                 # one node, STATUS Ready
kubectl get pods -n kube-system   # all Running (see below)
helm version --short              # v4.x.x
helm list -A                      # an empty table, and no error
k9s version                       # k9s is installed
```

What `kube-system` shows depends on the cluster: with kubeadm, the control plane Pods, CoreDNS and kube-proxy (flannel runs in its own `kube-flannel` namespace); with k3s, CoreDNS, Traefik, local-path-provisioner and metrics-server, since k3s runs its control plane and flannel inside the k3s process itself.

`helm list -A` is the one that proves Helm can reach the cluster. An empty table is success; `Error: Kubernetes cluster unreachable` means Helm can't use `~/.kube/config`, so fix `kubectl` first.

If any of these fail, see the Ansible tutorial's [Troubleshooting](https://github.com/r97221004/ansible-tutorial/blob/master/docs/operating/troubleshooting.md). Every chart you install later depends on this cluster being healthy.

---

[← Helm Architecture](../helm-intro/architecture.md) · [↑ Contents](../../README.md) · [Helm Fundamentals →](../helm-fundamentals/overview.md)
