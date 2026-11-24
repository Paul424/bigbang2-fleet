# Step CA - ACME

step-ca runs on the `step-ca` VM (`192.168.48.20`, see [IPAM.md](IPAM.md)) and is
managed by the `step_ca` role in `providers/ansible/infra-vms`.

## Trust model

| Key                 | Where                                                   | Signs                              |
|---------------------|---------------------------------------------------------|------------------------------------|
| Root CA             | Offline: `providers/ansible/infra-vms/files/pki/root_ca_key` (gitignored) + backup | Intermediates only, rarely |
| Intermediate CA     | VM: `/etc/step-ca/secrets/intermediate_ca_key`          | Leaf certificates (ACME, 24h)      |

- The root key and the root password never stay on the VM. If the VM is
  compromised, replace the intermediate. The root, which every cluster trusts,
  stays valid.
- `root_ca.crt` is public. Distribute it via trust-manager and the Talos
  `TrustedRootsConfig`.
- Each key has its own password in the ansible-vault:
  `vault_step_ca_root_password`, `vault_step_ca_intermediate_password`,
  `vault_step_ca_provisioner_password`.

**Back up `root_ca_key` and the root password**, for example in a password
manager. Without both you can't sign a new intermediate.

## Endpoints

- ACME directory: `https://ca.bigbangfleet.lab/acme/acme/directory`
  (until DNS is up, use `https://192.168.48.20/acme/acme/directory`)
- Client bootstrap:
  `step ca bootstrap --ca-url https://192.168.48.20 --fingerprint <fp>`.
  The `step-ca.yml` playbook prints the fingerprint.

## Creating a new intermediate

### With Ansible (recommended)

Requires `step` on the controller, plus `root_ca.crt` and `root_ca_key` in
`providers/ansible/infra-vms/files/pki/`.

```sh
ansible-playbook providers/ansible/infra-vms/playbooks/step-ca.yml \
  -e step_ca_rotate_intermediate=true
```

The playbook:

1. signs a new intermediate on the controller with the offline root key
   (name `Bigbangfleet Lab Intermediate CA`, validity
   `step_ca_intermediate_validity`, default 10 years), and encrypts its key
   with the intermediate password from the vault;
2. keeps the current intermediate on the VM as
   `intermediate_ca.crt.<timestamp>` / `intermediate_ca_key.<timestamp>`;
3. copies the new intermediate to the VM, verifies the chain against the root
   and restarts step-ca;
4. removes the temporary files, including the password files, from the
   controller.

Only pass `step_ca_rotate_intermediate=true` when you want a new intermediate.
Every run with it set signs a new one.

### Manually

```sh
cd providers/ansible/infra-vms/files/pki
step certificate create "Bigbangfleet Lab Intermediate CA" \
  intermediate_ca.crt intermediate_ca_key \
  --profile intermediate-ca --ca root_ca.crt --ca-key root_ca_key \
  --not-after 87600h
```

Encrypt the new key with the intermediate password from the vault. Copy the
files to `/etc/step-ca/certs/intermediate_ca.crt` and
`/etc/step-ca/secrets/intermediate_ca_key` (owner `step`), then run
`systemctl restart step-ca`. Don't leave `intermediate_ca_key` in `files/pki/`.

### Notes

- Leaf certificates from the old intermediate stay valid until they expire
  (max. 24h by default), because they chain to the same root.
- step-ca doesn't revoke the old intermediate. After a compromise, remove the
  old intermediate from every trust bundle that contains it. Clients that
  trust only the root aren't affected.

## Working with the root key

`step` commands read the encrypted key directly and prompt for the root
password:

```sh
step crypto key inspect providers/ansible/infra-vms/files/pki/root_ca_key
```

Only decrypt the key if a tool really needs it, and never leave it on disk:

```sh
openssl pkey -in root_ca_key -out root_ca_key.plain.pem && chmod 600 root_ca_key.plain.pem
# ...use...
shred -u root_ca_key.plain.pem
```
