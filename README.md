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
- IP plan: see [docs/IPAM.md](docs/IPAM.md). Flat `192.168.48.0/22` LAN, no MAC reservations.
- Flux (repo system-manifests): base/ + components/ + clusters/{mgmt,dev,siem,wrk01}.
  Mgmt rolls out to the other clusters via Kustomization spec.kubeConfig.secretRef.
  Secrets with SOPS + age.
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
