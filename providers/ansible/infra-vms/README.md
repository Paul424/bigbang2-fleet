# infra-vms

Ansible setup to manage KVM-VM's on Proxmox as clone of a cloud-init
template. Meant to host DNS and Step-ca.

```
inventory/
  hosts.yml                  # proxmox-nodes + infra_vms
  group_vars/all/main.yml    # API, netwerk, cloud-init defaults
  group_vars/all/vault.yml   # (zelf aanmaken) versleutelde secrets
  group_vars/infra_vms.yml   # default VM-sizing
  host_vars/<vm>.yml         # vm_id, vm_node, ansible_host (+ overrides)
roles/
  pve_template/              # bouwt template via qm op de node (SSH)
  proxmox_vm/                # clone + config + resize + start via API
playbooks/
  pve-template.yml
  vms.yml
  vms-destroy.yml
```

All commands are run from the **repository root**, where `ansible.cfg` points
to the inventory, roles and collections in this directory.

## Prerequisites

1. **Tooling**
   ```sh
   conda create -n ansible python=3.15
   conda activate ansible
   pip install -r ./providers/ansible/infra-vms/requirements.txt
   ansible-galaxy collection install -r ./providers/ansible/infra-vms/requirements.yml 
   ```

2. **API-token op Proxmox**
   Run on ny proxmox node (shell) as root
   ```sh
   pveum user add ansible@pve
   pveum role add Provisioner -privs "VM.Allocate VM.Clone VM.Config.CDROM VM.Config.CPU VM.Config.Cloudinit VM.Config.Disk VM.Config.HWType VM.Config.Memory VM.Config.Network VM.Config.Options VM.Audit VM.PowerMgmt Datastore.AllocateSpace Datastore.Audit SDN.Use Sys.Audit"
   pveum aclmod / -user ansible@pve -role Provisioner
   pveum user token add ansible@pve infra --privsep 0
   ```
   
   Store the token in an Ansible vault
   ```sh
   cd providers/ansible/infra-vms/inventory/group_vars/all
   cp vault.yml.example vault.yml && $EDITOR vault.yml
   ansible-vault encrypt vault.yml
   ```

3. **SSH as root to Proxmox-nodes**
   ```sh
   ssh-keygen -t ed25519 -a 100 -C "pbox@P17"
   ssh-copy-id root@192.168.50.170
   ```

4. **Edit inventory**

    1. IP's o/t nodes
    2. `pve_template_vmid` per node
    3. `lab_domain`
    4. gateway
    5. bridge/VLAN
    6. per VM `vm_id`/`vm_node`/`ansible_host`.

## Build

```sh
# 1. Template bouwen op elke node (idempotent; slaat over als VMID al bestaat)
ansible-playbook providers/ansible/infra-vms/playbooks/pve-template.yml

# 2. VM's aanmaken / bijwerken
ansible-playbook providers/ansible/infra-vms/playbooks/vms.yml --ask-vault-pass
ansible-playbook providers/ansible/infra-vms/playbooks/vms.yml --limit dns1 --ask-vault-pass

# Verwijderen
ansible-playbook providers/ansible/infra-vms/playbooks/vms-destroy.yml --limit dns2 -e confirm=yes --ask-vault-pass
```

Nieuwe VM toevoegen: host in `inventory/hosts.yml` onder een groep in
`infra_vms`, plus `inventory/host_vars/<naam>.yml`:

```yaml
vm_id: 130
vm_node: pve2
ansible_host: 192.168.1.70
vm_cores: 4
vm_memory: 4096
vm_disk_size: 40G
```

## Ontwerpkeuzes

- **Template per node**: `local-lvm` is niet gedeeld, dus clonen naar een
  andere node kan niet. Elke node krijgt een eigen template-VMID
  (`pve_template_vmid`) en VM's worden gecloned van het template op hun
  eigen `vm_node`. Met gedeelde storage (Ceph/NFS) kan één template volstaan.
- **Debian 13 genericcloud** met `qemu-guest-agent` erin gebakken via
  `virt-customize`; UEFI (OVMF) + q35, serial console.
- **Statische IP's via cloud-init** (`ipconfig0`). Tijdens bootstrap wijzen
  `lab_nameservers` naar publieke resolvers; zet ze na het uitrollen van
  dns1/dns2 om naar hun IP's.
- Wijzigingen aan hardware (cores/memory) worden via `update: true`
  toegepast; sommige vereisen een herstart van de VM.
