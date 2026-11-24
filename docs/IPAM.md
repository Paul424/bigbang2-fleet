# IPAM – homelab IP plan

## Principles

- One flat LAN on the router (Asus RT-AX88U Pro, Merlin): **`192.168.48.0/22`**
  (`192.168.48.0` – `192.168.51.255`), gateway **`192.168.50.1`**.
- DHCP stays limited to `192.168.50.2` – `192.168.50.249`. DHCP **never** hands
  out the rest of the /22.
- All lab addresses are assigned explicitly (cloud-init `ipconfig0`, Talos
  machine config, LB pools). The router has no MAC-based reservations.
- Everything shares one L2 segment. The laptop (`192.168.50.42`) reaches every
  host directly, without extra routes. L2 announcements for LoadBalancer pools
  work without BGP.

### Router settings

LAN → LAN IP:

| Field         | Value            |
|---------------|------------------|
| IP address    | `192.168.50.1`   |
| Subnet mask   | `255.255.252.0`  |
| DHCP pool     | `192.168.50.2` – `192.168.50.249` |

Clients pick up the new mask on their next DHCP renewal.

## Overview

| Range              | Purpose                                    |
|--------------------|--------------------------------------------|
| `192.168.48.0/24`  | Infra: Proxmox nodes and infra VMs         |
| `192.168.49.0/24`  | Talos clusters (one /26 per cluster)       |
| `192.168.50.0/24`  | Home network + DHCP (not used by the lab)  |
| `192.168.51.0/24`  | Reserved                                   |

## 192.168.48.0/24 – infra

| Address(es)         | Host / purpose                          | Status            |
|---------------------|-----------------------------------------|-------------------|
| `.1`                | p1 (Proxmox)                            | planned¹          |
| `.2`                | p2 (Proxmox)                            | planned¹          |
| `.3`                | p3 (Proxmox)                            | planned¹          |
| `.4` – `.9`         | Reserved for more Proxmox nodes         |                   |
| `.10`               | dns1 (PowerDNS auth + recursor), on p1  | planned           |
| `.11`               | dns2 (PowerDNS auth + recursor), on p2  | planned           |
| `.12` – `.19`       | Reserved for DNS                        |                   |
| `.20`               | step-ca (ACME)                          | planned           |
| `.21` – `.29`       | Reserved for PKI                        |                   |
| `.30` – `.39`       | Other infra (bastion, backup, …)        |                   |
| `.40` – `.254`      | Free                                    |                   |

¹ The Proxmox nodes stay on their current addresses for now:

| Node | Current            | Planned          |
|------|--------------------|------------------|
| p1   | `192.168.50.170`   | `192.168.48.1`   |
| p2   | `192.168.50.171`   | `192.168.48.2`   |
| p3   | `192.168.50.172`   | `192.168.48.3`   |

The current addresses are inside the DHCP pool. Until the nodes move, make sure
the router can't hand these addresses to another client. For example, shrink
the pool to `.2` – `.169`, or move the nodes before the pool reaches them.

## 192.168.49.0/24 – clusters

Each cluster gets a /26 (64 addresses) with a fixed layout:

| Cluster | Block               | VIP     | Control planes  | Workers         | LB pool         |
|---------|---------------------|---------|-----------------|-----------------|-----------------|
| mgmt    | `192.168.49.0/26`   | `.1`    | `.10` – `.19`   | `.20` – `.39`   | `.40` – `.62`   |
| dev     | `192.168.49.64/26`  | `.65`   | `.74` – `.83`   | `.84` – `.103`  | `.104` – `.126` |
| siem    | `192.168.49.128/26` | `.129`  | `.138` – `.147` | `.148` – `.167` | `.168` – `.190` |
| wrk01   | `192.168.49.192/26` | `.193`  | `.202` – `.211` | `.212` – `.231` | `.232` – `.254` |

### Layout within a /26 (offset from the start of the block)

| Offset        | Purpose                                   |
|---------------|-------------------------------------------|
| `+0`          | Block network address, don't use          |
| `+1`          | Talos / kube-apiserver VIP                |
| `+2` – `+9`   | Reserved                                  |
| `+10` – `+19` | Control-plane nodes                       |
| `+20` – `+39` | Worker nodes                              |
| `+40` – `+62` | LoadBalancer pool (Cilium / MetalLB L2)   |
| `+63`         | Don't use                                 |

> The /26 blocks are an administrative split only. Every host uses the LAN
> netmask (`/22`) with gateway `192.168.50.1`.

## DNS

Domain: **`bigbangfleet.lab`**. Each cluster has its own subzone:
`<cluster>.bigbangfleet.lab`.

Nameservers for all lab hosts: `192.168.48.10` and `192.168.48.11` (the
recursors on dns1/dns2). Until those run, the bootstrap uses a public resolver.

| Name                          | Address          |
|-------------------------------|------------------|
| `dns1.bigbangfleet.lab`       | `192.168.48.10`  |
| `dns2.bigbangfleet.lab`       | `192.168.48.11`  |
| `ca.bigbangfleet.lab`         | `192.168.48.20`  |
| `api.mgmt.bigbangfleet.lab`   | `192.168.49.1`   |
| `api.dev.bigbangfleet.lab`    | `192.168.49.65`  |
| `api.siem.bigbangfleet.lab`   | `192.168.49.129` |
| `api.wrk01.bigbangfleet.lab`  | `192.168.49.193` |

> `.lab` isn't a public TLD, so the recursors resolve it locally. Devices that
> don't use dns1/dns2, such as the laptop through the router's DNS, need a
> conditional forward for `bigbangfleet.lab` to `192.168.48.10`/`.11`.
> Merlin can do this with dnsmasq: `server=/bigbangfleet.lab/192.168.48.10`.

## Open items

- Align the Proxmox VMID convention with this plan.
