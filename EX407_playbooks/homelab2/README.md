Two Ansible playbooks to configure Apache and HAProxy.

HAProxy Ansible role is defined in the roles/requirements.yml file.

The role is used to install and configure HAProxy to load balance two Apache servers.

```
$ ansible-galaxy install -r ./roles/requirements.yml -p roles/
$ ansible-playbook site.yml
```
