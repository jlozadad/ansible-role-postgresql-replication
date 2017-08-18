
PostgreSQL Streaming Replication
=========
[![Galaxy](https://img.shields.io/badge/galaxy-samdoran.postgresql--replication-blue.svg?style=flat)](https://galaxy.ansible.com/samdoran/postgresql-replication)

Configure PostgreSQL streaming replication between two nodes. This role was developed and tested for use on PostgreSQL 9.4 for setting up a redundant database backend for [Ansible Tower](https://www.ansible.com/tower). This will not configure advanced clustering but will configure two PostgreSQL nodes in a master/slave configuration.

Thes role depends on the roles included with the Ansible Tower installer.

Requirements
------------

Ansible Tower installer roles in your `roles_path` as well as the Ansible Tower inventory file.

Add the Client database node to the Ansible Tower inventory file and define `postgresrep_role` for each database host.

```
[tower]
tower1 ansible_connection=local
tower2
tower3

[database]
db-main postgresrep_role=main

[database_client]
db-client postgresrep_role=client

...

```



Role Variables
--------------

| Name              | Default Value       | Description          |
|-------------------|---------------------|----------------------|
| `pg_port` | `5432` | PostgreSQL port |
| `bundle_install` | `False` | Set to `True` if using the Bundle Installer |
| `postgresrep_role` | `skip` | `main` or `client`, which determinse which tasks run on the host |
| `postgresrep_user` | `replicator` | User account that will be created and used for replication. |
| `postgresrep_password` | `[undefined]` | Password for replication account |
| `postgresrep_wal_level` | `hot_standby` | WAL level |
| `postgresrep_max_wal_senders` | `2` | Max number of WAL senders. Don't set this less than two otherwise the initial sync will fail. |
| `postgresrep_wal_keep_segments` | `100` | Max number of WAL segments |
| `postgresrep_synchronous_commit` | `local` | Set to `on`, `local`, or `off`. Setting to `on` will cause the main to stop accepting writes in the client goes down. See [documentation](https://www.postgresql.org/docs/9.1/static/runtime-config-wal.html#GUC-SYNCHRONOUS-COMMIT) |
| `postgresrep_application_name` | `awx` | Application name used for synchronization. |
| `postgresrep_group_name` | `database_client` | Name of the group that contains the client database. |
| `postgresrep_group_name_main` | `database` | Name of the gorup that contains the main database. |
| `postgresrep_main_address` | `[default IPv4 of the main]` | If you need something other than the default IPv4 address, for expample, FQDN, define it here. |
| `postgresrep_client_address` | `[default IPv4 of the client` | If you need something other than the default IPv4 address, for expample, FQDN, define it here. |
| `postgresrep_postgres_conf_lines` | `[see defaults/main.yml]` | Lines in `postgres.conf` that are set in order to enable streaming replication. |
| `postgresrep_pg_hba_conf_lines` | `[see defaults/main.yml]` | Lines to add to `pg_hba.conf` that allow client to connect to main. |


Dependencies
------------

- postgresql
- firewall

Example Playbook
----------------

Install this role alongside the roles used by the Anisble Tower installer (bundled or standalone). Then run the example playbook.

```
ansible-galaxy install 
ansible-playbook -b -i inventory psql-replication.yml
```

```yaml
- name: Configure PostgreSQL streaming replication
  hosts: database_client

  pre_tasks:
    - name: Remove recovery.conf
      file:
        path: /var/lib/pgsql/9.4/data/recovery.conf
        state: absent

    - name: Add client to database group
      add_host:
        name: "{{ inventory_hostname }}"
        groups: database
      tags:
        - always

  roles:
    - role: packages_el
      packages_el_install_tower: false
      packages_el_install_postgres: true
      when: ansible_os_family == "RedHat"

    - role: postgres
      tags: postgresql_database
      postgres_allowed_ipv4: "0.0.0.0/0"
      postgres_allowed_ipv6: "::/0"
      postgres_username: "{{ pg_username }}"
      postgres_password: "{{ pg_password }}"
      postgres_database: "{{ pg_database }}"
      max_postgres_connections: 1024
      postgres_shared_memory_size: "{{ (ansible_memtotal_mb*0.3)|int }}"
      postgres_work_mem: "{{ (ansible_memtotal_mb*0.03)|int }}"
      postgres_maintenance_work_mem: "{{ (ansible_memtotal_mb*0.04)|int }}"

    - role: firewall
      tags: firewall
      firewalld_http_port: "{{ nginx_http_port }}"
      firewalld_https_port: "{{ nginx_https_port }}"
      when: ansible_os_family == 'RedHat'

- name: Configure main server
  hosts: database[0]

  roles:
    - samdoran.postgresql-replication

- name: Configure clinet server
  hosts: database_clinet

  roles:
    - samdoran.postgresql-replication
```

This playbook can be run multiple times. Each time, it erases all the data on the client node and creates a fresh copy of the database from the master.

If the primary database node goes now, here is a playbook that can be used to fail over to the secondary node.

```yaml
- name: Gather facts
  hosts: all
  become: yes

- name: Failover PostgreSQL
  hosts: database_client
  become: yes

  tasks:
    - name: Promote secondary PostgreSQL server to primary
      command: /usr/pgsql-9.4/bin/pg_ctl promote
      become_user: postgres
      environment:
        PGDATA: /var/lib/pgsql/9.4/data
      ignore_errors: yes

- name: Update Ansible Tower database configuration
  hosts: tower
  become: yes

  tasks:
    - name: Update Tower postgres.py
      lineinfile:
        dest: /etc/tower/conf.d/postgres.py
        regexp: "^(.*'HOST':)"
        line: "\\1 '{{ hostvars[groups['database_client'][0]].ansible_default_ipv4.address }}',"
        backrefs: yes
      notify: restart tower

  handlers:
    - name: restart tower
      command: ansible-tower-service restart
```

License
-------

MIT
