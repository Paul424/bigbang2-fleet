# Bigbang2 - Fleet Management

## Objectives

Homelab on Proxmox (HP machines) with 4 Talos clusters and central policy management.

## Clusters (one subzone per cluster under `bigbangfleet.lab`)
- mgmt  – Keycloak (IdP), Kyverno policies, Flux hub (fleet management)
- dev   – GitLab, Harbor (migrated from minikube; Harbor also mirrors registry1.dso.mil)
- siem  – Wazuh or OpenSearch; receives Talos and kube-apiserver audit logs
- wrk01 – Big Bang stack (runs on Flux itself; align its own Kyverno policies)

## Decisions
- Clusters are NOT created with Cluster API.
- Two layers: Ansible outside k8s, Flux inside k8s.
- Ansible (repo homelab-infra): community.proxmox for VMs (clone of a cloud-init template),
  roles base, powerdns, step-ca, talos-cluster. Secrets in ansible-vault.
- Infra VMs outside the clusters: dns1/dns2 (PowerDNS auth + recursor, on different
  Proxmox nodes) and step-ca (ACME, root distributed via trust-manager and Talos TrustedRootsConfig).
- DNS: domain `bigbangfleet.lab`, a subzone per cluster (e.g. `api.mgmt.bigbangfleet.lab`),
  external-dns (rfc2136) with a TSIG key per zone.
  Talos nameservers -> recursors. No /etc/hosts or hostAliases.
- Storage: Rook-Ceph on the mgmt cluster (raw disks from the Proxmox `data` VG); other clusters
  consume it as an external cluster via ceph-csi, with a pool and cephx user per cluster.
  See [docs/STORAGE.md](docs/STORAGE.md).
- IP plan: see [docs/IPAM.md](docs/IPAM.md). Flat `192.168.48.0/22` LAN, no MAC reservations.
- Flux (repo system-manifests): base/ + components/ + clusters/{mgmt,dev,siem,wrk01}.
  Mgmt rolls out to the other clusters via Kustomization spec.kubeConfig.secretRef.
  Secrets with SOPS + GPG (one key per cluster, private key in ansible-vault,
  secret `env/sops-gpg` in the cluster).
- system-manifests on GitHub, not only in our own GitLab, so it doesn't depend on the dev cluster.

## Order
1. Proxmox template + infra VMs (DNS, step-ca)
2. mgmt cluster + Flux bootstrap + Keycloak
3. dev (GitLab/Harbor) -> 4. siem -> 5. wrk01 (Big Bang)

## Open
- Number of HP nodes, RAM/cores -> VM distribution

## Prerequisites

1. Python; through miniconda /w pip
2. Ansible; load using the requirements.txt
3. [docker](https://docs.docker.com/engine/install/ubuntu/)
4. [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
5. [kubelogin plugin](https://github.com/int128/kubelogin/releases)
6. [flux](https://fluxcd.io/flux/installation/)
7. [sops](https://getsops.io/docs/installation/)
8. [talosctl](https://www.talos.dev/latest/talos-guides/install/talosctl/) (same minor version as the clusters)
9. [helm](https://helm.sh/docs/intro/install/)

## Run

Run everything from the repository root. The vault password comes from
`vault_password_file` in `ansible.cfg`. Without it, add `-J` to every
`ansible-playbook` command.

### 0. Tooling and secrets (once)

```sh
conda activate ansible
pip install -r providers/ansible/infra-vms/requirements.txt
ansible-galaxy collection install -r providers/ansible/infra-vms/requirements.yml

# Proxmox API token, step-ca, PowerDNS and TSIG secrets (see vault.yml.example)
ansible-vault edit providers/ansible/infra-vms/inventory/group_vars/all/vault.yml
```

### 1. Proxmox: repos + cloud-init template on every node

```sh
ansible-playbook providers/ansible/infra-vms/playbooks/pve-template.yml
```

### 2. Infra VMs: step-ca and DNS

```sh
ansible-playbook providers/ansible/infra-vms/playbooks/vms.yml --limit step-ca
ansible-playbook providers/ansible/infra-vms/playbooks/step-ca.yml

ansible-playbook providers/ansible/infra-vms/playbooks/vms.yml --limit dns
ansible-playbook providers/ansible/infra-vms/playbooks/dns.yml
```

Back up `providers/ansible/infra-vms/files/pki/root_ca_key` and the root
password. The router needs a conditional forward for `bigbangfleet.lab` to
dns1/dns2 (see [docs/DNS.md](docs/DNS.md)). Check it with:

```sh
dig +short api.mgmt.bigbangfleet.lab    # -> 192.168.49.1
```

### 3. mgmt cluster: VMs, Talos, Cilium, GitOps

```sh
# Talos ISO to the Proxmox nodes + VMs (maintenance mode, IP via cloud-init)
ansible-playbook providers/ansible/infra-vms/playbooks/talos-vms.yml --limit 'proxmox,mgmt'

# Machine configs, apply, bootstrap etcd, kubeconfig, Cilium
ansible-playbook providers/ansible/infra-vms/playbooks/talos-cluster.yml -e talos_cluster=mgmt

# SOPS key + namespace env + secret env/sops-gpg, then Flux (environments/mgmt/gitops)
ansible-playbook providers/ansible/infra-vms/playbooks/bootstrap-cluster.yml -e talos_cluster=mgmt

export TALOSCONFIG=$PWD/providers/ansible/infra-vms/files/talos/mgmt/talosconfig
export KUBECONFIG=$PWD/providers/ansible/infra-vms/files/talos/mgmt/kubeconfig
talosctl health
kubectl get nodes -o wide
```

### Rebuilding a single VM

```sh
ansible-playbook providers/ansible/infra-vms/playbooks/vms-destroy.yml --limit step-ca -e confirm=yes
ansible-playbook providers/ansible/infra-vms/playbooks/vms.yml --limit step-ca
```

### Useful partial runs

```sh
# Only the SOPS key / only Flux
ansible-playbook providers/ansible/infra-vms/playbooks/bootstrap-cluster.yml -e talos_cluster=mgmt --tags sops
ansible-playbook providers/ansible/infra-vms/playbooks/bootstrap-cluster.yml -e talos_cluster=mgmt --tags gitops
# Only the CNI
ansible-playbook providers/ansible/infra-vms/playbooks/talos-cluster.yml -e talos_cluster=mgmt --tags cilium
# New step-ca intermediate (see docs/CA.md)
ansible-playbook providers/ansible/infra-vms/playbooks/step-ca.yml -e step_ca_rotate_intermediate=true
```
