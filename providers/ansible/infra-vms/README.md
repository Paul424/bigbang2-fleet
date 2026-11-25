# infra-vms

Ansible setup to manage KVM VMs on Proxmox as clones of a cloud-init
template. Meant to host DNS and step-ca. For addresses, see [docs/IPAM.md](../../../docs/IPAM.md).

```
inventory/
  hosts.yml                  # proxmox nodes + infra_vms
  group_vars/all/main.yml    # API, network, cloud-init defaults
  group_vars/all/vault.yml   # (create yourself) encrypted secrets
  group_vars/infra_vms.yml   # default VM sizing
  host_vars/<vm>.yml         # vm_id, vm_node, ansible_host (+ overrides)
roles/
  pve_repos/                 # disables enterprise repos, enables no-subscription repos
  pve_template/              # builds the template with qm on the node (SSH)
  proxmox_vm/                # clone + config + resize + start through the API
  step_ca/                   # installs step-ca, creates root + intermediate CA
  powerdns/                  # PowerDNS auth + recursor, zones, TSIG keys
  talos_image/               # Talos nocloud ISO from the Image Factory to the nodes
  talos_cluster/             # Talos machine configs, apply, bootstrap, kubeconfig
  cilium/                    # CNI: Cilium (eBPF, kube-proxy replacement, L2 LB pool)
playbooks/
  pve-template.yml
  vms.yml
  vms-destroy.yml
  step-ca.yml
  dns.yml
  talos-vms.yml
  talos-cluster.yml
files/talos/<cluster>/
  secrets.yaml               # Talos cluster secrets, ansible-vault encrypted (commit it)
  talosconfig, kubeconfig    # admin credentials (gitignored)
files/pki/
  root_ca.crt                # fetched root certificate (public, commit it)
  root_ca_key                # fetched encrypted root key (gitignored!)
```

Run all commands from the **repository root**. The `ansible.cfg` there points
to the inventory, roles and collections in this directory.

## Prerequisites

1. **Tooling**
   ```sh
   conda create -n ansible python=3.15
   conda activate ansible
   pip install -r ./providers/ansible/infra-vms/requirements.txt
   ansible-galaxy collection install -r ./providers/ansible/infra-vms/requirements.yml
   ```

2. **API token on Proxmox**
   Run on any Proxmox node (shell) as root:
   ```sh
   pveum user add ansible@pve
   pveum role add Provisioner -privs "VM.Allocate VM.Clone VM.Config.CDROM VM.Config.CPU VM.Config.Cloudinit VM.Config.Disk VM.Config.HWType VM.Config.Memory VM.Config.Network VM.Config.Options VM.Audit VM.PowerMgmt Datastore.AllocateSpace Datastore.Audit SDN.Use Sys.Audit"
   pveum aclmod / -user ansible@pve -role Provisioner
   pveum user token add ansible@pve infra --privsep 0
   ```

   Store the token in an Ansible vault:
   ```sh
   cd providers/ansible/infra-vms/inventory/group_vars/all
   cp vault.yml.example vault.yml && $EDITOR vault.yml
   ansible-vault encrypt vault.yml
   ```

   Ansible needs the vault password on every run. Pass `--ask-vault-pass`
   (`-J`), or set `vault_password_file` in `ansible.cfg`.

3. **SSH as root to the Proxmox nodes**
   ```sh
   ssh-keygen -t ed25519 -a 100 -C "pbox@P17"
   ssh-copy-id root@192.168.50.170
   ```

4. **Edit the inventory**

    1. IPs of the nodes
    2. `pve_template_vmid` per node
    3. `lab_domain`
    4. gateway
    5. bridge/VLAN
    6. per VM: `vm_id`/`vm_node`/`ansible_host`

## Build

```sh
# 1. Build the template on every node (idempotent; skipped if the VMID already exists)
ansible-playbook providers/ansible/infra-vms/playbooks/pve-template.yml -J

# 2. Create / update VMs
ansible-playbook providers/ansible/infra-vms/playbooks/vms.yml -J
ansible-playbook providers/ansible/infra-vms/playbooks/vms.yml --limit dns1 -J

# Remove
ansible-playbook providers/ansible/infra-vms/playbooks/vms-destroy.yml --limit dns2 -e confirm=yes -J
```

To add a VM, put the host in `inventory/hosts.yml` under a group in
`infra_vms`, and add `inventory/host_vars/<name>.yml`:

```yaml
vm_id: 130
vm_node: p2
ansible_host: 192.168.48.30
vm_cores: 4
vm_memory: 4096
vm_disk_size: 40G
```

## step-ca

```sh
ansible-playbook providers/ansible/infra-vms/playbooks/step-ca.yml -J
```

Add three passwords to the vault first (`ansible-vault edit .../vault.yml`):
`vault_step_ca_root_password`, `vault_step_ca_intermediate_password` and
`vault_step_ca_provisioner_password` (see `vault.yml.example`).

On the first run the role:

1. installs `step-cli` and `step-ca` from the Smallstep apt repo;
2. runs `step ca init` (standalone, name `Bigbangfleet Lab`, SANs
   `ca.bigbangfleet.lab`, `step-ca.bigbangfleet.lab` and the VM IP, listening on
   `:443`), with a JWK provisioner `admin@bigbangfleet.lab` and an ACME
   provisioner `acme`;
3. re-encrypts the intermediate key with its own password, so the root
   password never stays on the VM;
4. sets leaf certificate lifetimes (default 24h, max 90 days) and installs a
   hardened systemd unit;
5. fetches `root_ca.crt` and the encrypted `root_ca_key` to `files/pki/`,
   then **removes the root key from the VM** (`step_ca_remove_root_key`).

Later runs leave the existing PKI alone; they only update the config and unit.

To replace the intermediate, run the playbook with
`-e step_ca_rotate_intermediate=true`. See [docs/CA.md](../../../docs/CA.md).

> **Back up `files/pki/root_ca_key` and the root password**, for example in a
> password manager. Both are needed to sign a new intermediate. The file is
> gitignored and exists only on the machine that ran the playbook.

Endpoints:
- ACME directory: `https://ca.bigbangfleet.lab/acme/acme/directory`
- Client bootstrap: `step ca bootstrap --ca-url https://192.168.48.20 --fingerprint <fp>`
  (the playbook prints the fingerprint)

## DNS

```sh
ansible-playbook providers/ansible/infra-vms/playbooks/dns.yml
```

Installs PowerDNS Authoritative (`:5300`) and Recursor (`:53`) on dns1/dns2,
with dns1 as primary and dns2 as secondary. Creates the zones from
`inventory/group_vars/dns.yml` and a TSIG key per cluster subzone. Add
`vault_pdns_api_key` and `vault_dns_tsig_keys` to the vault first. See
[docs/DNS.md](../../../docs/DNS.md).

## Talos VMs

```sh
ansible-playbook providers/ansible/infra-vms/playbooks/talos-vms.yml --limit 'proxmox,mgmt'
```

1. Registers a schematic with the Talos Image Factory (extension
   `siderolabs/qemu-guest-agent`) and downloads the **nocloud** ISO to
   `local:iso` on every Proxmox node.
2. Creates an empty VM per node (no clone): SeaBIOS, q35, `x86-64-v3`, VirtIO
   SCSI, disk + cloud-init drive + ISO, boot order `scsi0;ide2`.
3. Talos reads the static IP, gateway and nameservers from the cloud-init
   drive (`ipconfig0`), so the node is reachable on its final IP in maintenance
   mode. The playbook waits for the Talos API on `:50000`.

Sizing and the Talos version are in `inventory/group_vars/cluster_vms/main.yml`.
The nodes are in `inventory/hosts.yml` under `cluster_vms`. The installer
image to use in the Talos machine config is printed at the end of the ISO play.

## Talos cluster

```sh
ansible-playbook providers/ansible/infra-vms/playbooks/talos-cluster.yml -e talos_cluster=mgmt
export TALOSCONFIG=$PWD/providers/ansible/infra-vms/files/talos/mgmt/talosconfig
export KUBECONFIG=$PWD/providers/ansible/infra-vms/files/talos/mgmt/kubeconfig
```

Runs entirely on the controller and needs `talosctl` (same minor version as
`talos_version`), `helm`, and `vault_password_file` in `ansible.cfg`.
`api.<cluster>.bigbangfleet.lab` must resolve on the controller, because the
kubeconfig points to it.

1. **Secrets**: on the first run, `talosctl gen secrets` creates
   `files/talos/<cluster>/secrets.yaml`, encrypted with ansible-vault. Every
   later run regenerates the same configs from it.
2. **Configs**: `talosctl gen config --with-secrets` plus patches, in a
   temporary directory that is always removed. Talos 1.14 generates a
   multi-document config, so the patches target those documents:
   - all nodes (`common.yaml.j2`): `machine.certSANs`, `UnattendedInstallConfig`
     (nocloud installer image from the schematic, disk `/dev/sda`),
     `KubeNodeConfig.nodeIP.validSubnets`, `ResolverConfig` (dns1/dns2), a
     `LinkAliasConfig` `lan0` for the NIC (MAC prefix `bc:24:11`), and the
     step-ca root as `TrustedRootsConfig`;
   - control planes (`controlplane.yaml.j2`): `Layer2VIPConfig` on `lan0`,
     `KubeAPIServerConfig.certExtraSANs`, Pod Security (`talos_psa`), OIDC
     (`talos_oidc`), the control-plane taint removed if
     `talos_schedule_on_controlplanes`, and with `talos_cni: none` no
     `KubeFlannelCNIConfig` and `KubeProxyConfig.enabled: false`;
   - optional: `talos_extra_patches` (all nodes) and `talos_node_patch` (per node).

   Hostname, IP, gateway and DNS search domain come from the Proxmox cloud-init
   drive. The generated `HostnameConfig` (`auto: stable`) has the lowest
   priority, so the cloud-init hostname wins.
3. **Apply**, one node at a time. A node in maintenance mode gets
   `apply-config --insecure`. A configured node gets a `--dry-run` first, and
   is only updated (`--mode=auto`, may reboot) if the config changed. After
   each update the playbook waits for `talosctl health` before the next node.
4. **Bootstrap** etcd on the first control plane (skipped if etcd already
   exists), wait for `talosctl health`, then write `talosconfig` and
   `kubeconfig` to `files/talos/<cluster>/`.

The clusters use `talos_cni: none` (no flannel, no kube-proxy). In that case
the `talos_cluster` role only waits for the API server, and the `cilium` role
then installs the CNI:

- Helm chart `cilium` in `kube-system` (version `cilium_version`), with the
  Talos settings (KubePrism `localhost:7445`, cgroup, capabilities),
  `kubeProxyReplacement`, BPF masquerading and Hubble;
- `policyCIDRMatchMode: [nodes]`, so `ipBlock` rules in NetworkPolicies
  (Big Bang `controlPlaneCidr`) match node IPs;
- Istio compatibility (`socketLB.hostNamespaceOnly`, `cni.exclusive: false`);
- a `CiliumLoadBalancerIPPool` from `cilium_lb_pool` plus a
  `CiliumL2AnnouncementPolicy`, so LoadBalancer services get an IP from
  the cluster's IPAM pool without MetalLB;
- it finishes when all nodes are Ready and `talosctl health` passes.

Flux can take over the Helm release later (same name and namespace).
Run only the CNI part with `--tags cilium`.

Cluster settings are in `inventory/group_vars/<cluster>.yml`
(`talos_cluster_name`, `talos_vip`, `talos_node_subnet`,
`talos_schedule_on_controlplanes`). The node role is `talos_role`
(`controlplane` by default, set `worker` per host).

## Design choices

- **One template per node**: `local-lvm` isn't shared, so you can't clone to
  another node. Each node gets its own template VMID (`pve_template_vmid`), and
  VMs are cloned from the template on their own `vm_node`. With shared storage
  (Ceph/NFS), one template would be enough.
- **No-subscription repos**: `pve_repos` disables the enterprise repos, which
  return 401 without a subscription, and adds the no-subscription repos. Set
  `pve_repos_no_subscription: false` to leave the repos alone.
- **Debian 13 genericcloud** with `qemu-guest-agent` baked in through
  `virt-customize`; SeaBIOS + q35, VirtIO SCSI, default display (serial port kept for `qm terminal`).
- **Static IPs through cloud-init** (`ipconfig0`). During bootstrap,
  `lab_nameservers` points to public resolvers. Switch it to the dns1/dns2 IPs
  once they're deployed.
- Hardware changes (cores/memory) are applied through `update: true`. Some
  of them need a VM restart.
