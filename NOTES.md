

conda activate ansible


cd /home/pbox/projects/bigbang2-fleet/providers/ansible/infra-vms
echo ".venv/" >> .gitignore
V=/tmp/claude-1000/-home-pbox-projects-bigbang2-fleet/f8c7de0b-2f2b-4f43-a555-1c63b9994217/scratchpad/venv
python3 -m venv $V && $V/bin/pip -q install ansible-core proxmoxer requests 2>&1 | tail -2
$V/bin/ansible-galaxy collection install -r requirements.yml -p ./collections 2>&1 | tail -3
echo '---
vault_proxmox_api_token_secret: x' > inventory/group_vars/all/vault.yml
for p in playbooks/*.yml; do $V/bin/ansible-playbook --syntax-check $p 2>&1 | tail -2; done
$V/bin/ansible-inventory --host dns2 2>&1 | head -30


ansible-playbook providers/ansible/infra-vms/playbooks/vms-destroy.yml --limit step-ca -e confirm=yes
ansible-playbook providers/ansible/infra-vms/playbooks/vms.yml --limit step-ca

ansible-playbook providers/ansible/infra-vms/playbooks/vms.yml --limit dns
ansible-playbook providers/ansible/infra-vms/playbooks/dns.yml