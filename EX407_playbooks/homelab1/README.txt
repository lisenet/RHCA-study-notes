Two Ansible playbooks to configure MySQL and Apache.

MySQL service is installed on a separate partition mounted on /var/lib/mysql.

Apache uses PHP to connect to the MySQL database and run a simple query.

Ansible Vault is used to store database credentials.

```
$ ansible-playbook --vault-password-file=vault play1_mysql.yml
$ ansible-playbook --vault-password-file=vault play2_apache.yml
```
