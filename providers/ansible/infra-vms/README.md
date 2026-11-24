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
playbooks/
  pve-template.yml
  vms.yml
  vms-destroy.yml
  step-ca.yml
  dns.yml
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
