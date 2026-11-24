# Bigbang2 - Fleet Management

## Objectives

Homelab op Proxmox (HP-machines) met 4 Talos-clusters en centraal policy-beheer.

## Clusters (subzone per cluster onder lab.<domein>)
- mgmt  – Keycloak (IdP), Kyverno-policies, Flux hub (fleet management)
- dev   – GitLab, Harbor (migratie vanaf minikube; Harbor ook mirror voor registry1.dso.mil)
- siem  – Wazuh of OpenSearch; ontvangt Talos- en kube-apiserver-audit logs
- wrk01 – Big Bang stack (draait zelf op Flux, eigen Kyverno-policies afstemmen)

## Beslissingen
- Clusters worden NIET via Cluster API aangemaakt.
- Twee lagen: Ansible buiten k8s, Flux binnen k8s.
- Ansible (repo homelab-infra): community.proxmox voor VM's (clone van cloud-init template),
  rollen base, powerdns, step-ca, talos-cluster. Secrets in ansible-vault.
- Infra-VM's buiten de clusters: dns1/dns2 (PowerDNS auth + recursor, op verschillende
  Proxmox-nodes) en step-ca (ACME, root via trust-manager en Talos TrustedRootsConfig).
- DNS: eigen subzone per cluster, external-dns (rfc2136) met TSIG-key per zone.
  Talos nameservers -> recursors. Geen /etc/hosts of hostAliases.
- Flux (repo system-manifests): base/ + components/ + clusters/{mgmt,dev,siem,wrk01}.
  Mgmt rolt uit naar de andere clusters via Kustomization spec.kubeConfig.secretRef.
  Secrets met SOPS + age.
- system-manifests op GitHub (niet alleen in de eigen GitLab, om niet af te hangen van het dev-cluster).

## Volgorde
1. Proxmox-template + infra-VM's (DNS, step-ca)
2. mgmt-cluster + Flux bootstrap + Keycloak
3. dev (GitLab/Harbor) -> 4. siem -> 5. wrk01 (Big Bang)

## Open
- Aantal HP-nodes, RAM/cores -> VM-verdeling en IP-plan
- Domeinnaam en IP-ranges (LB-pools per cluster)

## Prerequisites

1. Python; through miniconda /w pip
2. Ansible; load using the requirements.txt
3. [docker](https://docs.docker.com/engine/install/ubuntu/)
4. [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
5. [kubelogin plugin](https://github.com/int128/kubelogin/releases)
6. [flux](https://fluxcd.io/flux/installation/)
7. [sops](https://getsops.io/docs/installation/)



