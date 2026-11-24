# DNS – PowerDNS

dns1 and dns2 each run PowerDNS Authoritative and PowerDNS Recursor. They're
managed by the `powerdns` role in `providers/ansible/infra-vms`. Addresses are
in [IPAM.md](IPAM.md).

## Layout

| Service        | Listens on                 | Used by                                              |
|----------------|----------------------------|------------------------------------------------------|
| Recursor       | `<vm-ip>:53`, `127.0.0.1:53` | Clients: Talos nodes, VMs, router (conditional forward) |
| Authoritative  | `<vm-ip>:5300`, `127.0.0.1:5300` | external-dns (RFC2136 + AXFR), zone transfers dns1 → dns2 |
| Auth HTTP API  | `127.0.0.1:8081`           | Ansible only (on the VM itself)                      |

- Each recursor forwards `bigbangfleet.lab` (and all subzones) to the
  authoritative server on the same VM. Everything else is resolved
  recursively from the root servers, or through `pdns_recursor_forwarders` if
  set.
- `.lab` isn't a public TLD, so the recursor has a negative trust anchor for
  `bigbangfleet.lab`. Without it, DNSSEC validation would mark the local zone
  as bogus.
- **dns1 is primary, dns2 is secondary.** dns1 sends NOTIFY to dns2 on `:5300`,
  and dns2 pulls the zones with AXFR. Dynamic updates must go to dns1.
- Storage: sqlite3 (`/var/lib/powerdns/pdns.sqlite3`) on each server.

## Zones

Zones are defined in
`providers/ansible/infra-vms/inventory/group_vars/dns.yml`:

| Zone                         | TSIG key             | Updates allowed from     |
|------------------------------|----------------------|--------------------------|
| `bigbangfleet.lab`           | —                    | — (Ansible only)         |
| `mgmt.bigbangfleet.lab`      | `external-dns-mgmt`  | `192.168.49.0/26`        |
| `dev.bigbangfleet.lab`       | `external-dns-dev`   | `192.168.49.64/26`       |
| `siem.bigbangfleet.lab`      | `external-dns-siem`  | `192.168.49.128/26`      |
| `wrk01.bigbangfleet.lab`     | `external-dns-wrk01` | `192.168.49.192/26`      |

The role creates these records automatically:

- NS records at every zone apex (`dns1.bigbangfleet.lab.`, `dns2.bigbangfleet.lab.`);
- delegations (NS) for each subzone in its parent zone;
- A records for `dns1` and `dns2` in `bigbangfleet.lab`.

All other records come from the `records` list of each zone. Ansible only
replaces the rrsets it declares. It never deletes other records, so records
created by external-dns stay in place.

Example record entries:

```yaml
records:
  - { name: "@",   type: A,     content: 192.168.49.1 }
  - { name: api,   type: A,     content: [192.168.49.10, 192.168.49.11] }
  - { name: www,   type: CNAME, content: api.mgmt.bigbangfleet.lab. }
  - { name: txt,   type: TXT,   content: '"hello"', ttl: 60 }
```

Write `content` in PowerDNS format: names end with a dot, and TXT values are
quoted.

Adding a cluster or subzone: add an entry to `pdns_zones` (with `tsig_key` and
`update_from`), add the secret to `vault_dns_tsig_keys`, then run the playbook.

## Secrets

In the ansible-vault (see `vault.yml.example`):

```yaml
vault_pdns_api_key: "<openssl rand -base64 32>"
vault_dns_tsig_keys:
  mgmt:  "<openssl rand -base64 32>"
  dev:   "<openssl rand -base64 32>"
  siem:  "<openssl rand -base64 32>"
  wrk01: "<openssl rand -base64 32>"
```

TSIG keys are `hmac-sha256`. The same secret goes into the cluster's
external-dns secret (SOPS-encrypted in system-manifests).

## Deploy

```sh
ansible-playbook providers/ansible/infra-vms/playbooks/vms.yml --limit dns
ansible-playbook providers/ansible/infra-vms/playbooks/dns.yml
```

The playbook configures dns1 first, then dns2. It finishes by resolving
`dns1.bigbangfleet.lab` through the local recursor.

## external-dns (per cluster)

Example for `mgmt`:

```yaml
args:
  - --provider=rfc2136
  - --rfc2136-host=192.168.48.10
  - --rfc2136-port=5300
  - --rfc2136-zone=mgmt.bigbangfleet.lab
  - --rfc2136-tsig-keyname=external-dns-mgmt
  - --rfc2136-tsig-secret-alg=hmac-sha256
  - --rfc2136-tsig-axfr
  - --domain-filter=mgmt.bigbangfleet.lab
  - --txt-owner-id=mgmt
env:
  - name: EXTERNAL_DNS_RFC2136_TSIG_SECRET
    valueFrom:
      secretKeyRef: { name: external-dns-tsig, key: secret }
```

## Clients

- **Talos / VMs**: nameservers `192.168.48.10`, `192.168.48.11`.
- **Home network / laptop** (through the router): add a conditional forward
  in Merlin dnsmasq (`/jffs/configs/dnsmasq.conf.add`):
  ```
  server=/bigbangfleet.lab/192.168.48.10
  server=/bigbangfleet.lab/192.168.48.11
  ```
  Then run `service restart_dnsmasq`.

## Checks

```sh
# through the recursor
dig @192.168.48.10 api.mgmt.bigbangfleet.lab

;; ANSWER SECTION:
api.mgmt.bigbangfleet.lab. 207	IN	A	192.168.49.1

# secondary auth, check the serial
dig @192.168.48.11 -p 5300 mgmt.bigbangfleet.lab SOA

;; ANSWER SECTION:
mgmt.bigbangfleet.lab.	300	IN	SOA	dns1.bigbangfleet.lab. hostmaster.bigbangfleet.lab. 2026112402 10800 3600 604800 3600

# AXFR with TSIG
dig @192.168.48.10 -p 5300 mgmt.bigbangfleet.lab AXFR \
  -y hmac-sha256:external-dns-mgmt:<secret>   

; (1 server found)
;; global options: +cmd
mgmt.bigbangfleet.lab.	300	IN	SOA	dns1.bigbangfleet.lab. hostmaster.bigbangfleet.lab. 2026112402 10800 3600 604800 3600
<...>

# through dnsmasq
dig api.mgmt.bigbangfleet.lab 

;; ANSWER SECTION:
api.mgmt.bigbangfleet.lab. 88	IN	A	192.168.49.1
```
