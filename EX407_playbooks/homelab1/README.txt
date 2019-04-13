Two Ansible playbooks to configure MySQL and Apache.

MySQL service is installed on a separate partition mounted on /var/lib/mysql.

Apache uses PHP to connect to the MySQL database and run a simple query.

Ansible Vault is used to store database credentials.
