
```
---
- import_playbook: install_postgresql_16.yml
- import_playbook: install_etcd.yml
- import_playbook: install_patroni.yml
- import_playbook: install_haproxy.yml
- import_playbook: install_pgbouncer.yml
- import_playbook: install_citus.yml
```