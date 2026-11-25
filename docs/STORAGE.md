# Storage – Ceph on the mgmt cluster

## Decision

The **mgmt cluster is also the storage cluster**. It runs Rook-Ceph on raw
disks from the `data` volume group of each Proxmox host. All other clusters
(dev, siem, wrk01) consume that Ceph cluster as an **external cluster**
through ceph-csi. There is one Ceph cluster for the whole lab, not one per
cluster.

## Layout

| Proxmox host | mgmt node | OSD disk (Talos)        | Status   |
|--------------|-----------|-------------------------|----------|
| p1           | mgmt-cp1  | `scsi1` → `/dev/sdb`, 400G | planned |
| p2           | mgmt-cp2  | `scsi1` → `/dev/sdb`, 400G | planned |
| p3           | mgmt-cp3  | `scsi1` → `/dev/sdb`, 400G | planned |
| p4–p6        | mgmt-w1…3 | `scsi1` → `/dev/sdb`       | future  |

- The disks are defined as `vm_extra_disks` in
  `providers/ansible/infra-vms/inventory/group_vars/mgmt.yml`. The Proxmox
  storage ID of the data VG is `data` (check with `pvesm status`).
- The disks have `backup=0`, so vzdump never backs up Ceph data.
- Talos installs on `/dev/sda` (`scsi0`) and must never touch `/dev/sdb`. In
  Rook, use `deviceFilter: ^sdb` instead of `useAllDevices`.

### Capacity

Pools use 3× replication, and Ceph should stay below ~80% full.

| Nodes | Raw     | Usable (size 3) | Comfortable (80%) |
|-------|---------|-----------------|-------------------|
| 3     | 1.2 TB  | 400 GB          | ~320 GB           |
| 6     | 2.4 TB  | 800 GB          | ~640 GB           |

With 3 hosts, one host can be down (`min_size 2`) and I/O continues, but
data isn't fully redundant until the host is back. With 6 hosts, Ceph can
recover onto the remaining hosts.

## Requirements and constraints

- **Host networking for Ceph** (`network.provider: host` in the
  CephCluster). Mons and OSDs must be reachable from the *other* clusters on
  the node IPs (`192.168.49.10–.12`). On the pod network they would only be
  reachable inside mgmt.
- **Stable mon IPs**: mons on host networking use the node IPs. Don't change
  node IPs without moving mons first.
- **Memory**: an OSD defaults to `osd_memory_target` 4 GiB. Add a mon
  (~1–2 GiB), a mgr (~1 GiB) and the Rook operator on top of the control plane,
  Keycloak, Flux and Kyverno, and 8 GB per node is too tight. The mgmt nodes
  therefore get **12 GB** (`vm_memory` in `group_vars/mgmt.yml`). Other
  clusters stay at 8 GB. Keep `osd_memory_target` at ~3 GiB.
- **Talos**: the `rbd` kernel module is available by default. The machine
  config needs no extra extensions for RBD. CephFS on consumer clusters uses
  the kernel client, which Talos also includes.
- **Upgrades**: Talos upgrades and reboots of mgmt nodes affect storage for
  *all* clusters. Upgrade one node at a time and wait for `HEALTH_OK` in
  between (set `noout` during planned maintenance).

## Per-cluster isolation

For each consumer cluster `<c>` in `dev`, `siem`, `wrk01` (and `mgmt` itself):

| Resource (in mgmt)          | Name           | Purpose                                  |
|-----------------------------|----------------|------------------------------------------|
| `CephBlockPool`             | `k8s-<c>`      | RBD volumes (RWO), with a quota          |
| `CephFilesystemSubVolumeGroup` | `k8s-<c>`   | CephFS volumes (RWX), optional           |
| `CephClient`                | `k8s-<c>`      | cephx user with caps on its own pool only |
| `CephObjectStoreUser`       | `<c>`          | S3 credentials on RGW (GitLab, Harbor, Loki, Velero) |

Consumer clusters get the mon endpoints, `fsid` and their own cephx key. Rook's
`create-external-cluster-resources.py` generates these. They're stored as
SOPS secrets in system-manifests, and ceph-csi runs in the consumer cluster
without a Rook operator.

## Object storage

A `CephObjectStore` (RGW) in mgmt gets an address from the mgmt LB pool
(`192.168.49.40–.62`) and a name such as `s3.mgmt.bigbangfleet.lab`.
GitLab, Harbor (registry blobs) and backups use it.

## Exceptions

Workloads that replicate themselves, such as OpenSearch in siem or CNPG
Postgres clusters, would be replicated twice on Ceph. If I/O or capacity
becomes a problem, put them on node-local storage instead, for example the
Proxmox CSI plugin on LVM-thin.
