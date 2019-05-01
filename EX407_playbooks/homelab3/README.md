Various Ansible playbooks to install packages, create users, use system roles, configure Apache, MySQL, HAProxy, vsftpd etc.

```
$ ansible-playbook --vault-password-file=vault_secret.txt site.yml
```

Utilise dynamic inventory:

```
$ ansible all -i dynamic_inventory/proxmox.py --limit='running,!ignore' -m ping
```
