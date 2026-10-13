
```
scope: {{ patroni_scope }}
namespace: /postgresql-16
name: {{ inventory_hostname }}

restapi:
  listen: 0.0.0.0:8008
  connect_address: "{{ ansible_host }}:8008"

etcd3:
  hosts:
    - "{{ hostvars['dcs1'].ansible_host }}:2379"
    - "{{ hostvars['dcs2'].ansible_host }}:2379"
    - "{{ hostvars['dcs3'].ansible_host }}:2379"

bootstrap:
  initdb:
    - encoding: UTF8
    - data-checksums
  users:
    replicator:
      options:
        - replication
        - login
    rewind:
      options:
        - login
  dcs:
    postgresql:
      parameters:
        unix_socket_directories: /var/run/postgresql
        shared_preload_libraries: 'citus'
        wal_level: logical
        max_replication_slots: 20
        citus.enable_change_data_capture: on
        max_prepared_transactions: 10
        citus.node_conninfo: 'sslmode=prefer'
        max_connections: 200
        max_wal_senders: 10
        hot_standby: on
        logging_collector: on
        log_destination: 'stderr'
        log_directory: 'log'
        log_filename: 'postgresql-%a.log'
        log_rotation_age: 1440
        log_truncate_on_rotation: on
        log_rotation_size: 0
        log_replication_commands: on
        log_connections: on
        log_disconnections: on
        log_line_prefix: '%m [%p] %q%u@%d/%a %r '
      pg_hba:
        - local all all trust
        - host all all 0.0.0.0/0 trust
        - host replication replicator 0.0.0.0/0 trust
        - host replication rewind 0.0.0.0/0 trust

postgresql:
  listen: 0.0.0.0:5432
  connect_address: "{{ ansible_host }}:5432"
  data_dir: /var/lib/postgresql/16/patroni
  bin_dir: /usr/lib/postgresql/16/bin
  use_unix_socket: true

  authentication:
    replication:
      username: replicator
    superuser:
      username: postgres
    rewind:
      username: rewind

  parameters:
    unix_socket_directories: /var/run/postgresql
    shared_preload_libraries: 'citus'
    wal_level: logical
    max_replication_slots: 20
    citus.enable_change_data_capture: on
    max_prepared_transactions: 10
    citus.node_conninfo: 'sslmode=prefer'
    max_connections: 200
    max_wal_senders: 10
    hot_standby: on
    logging_collector: on
    log_destination: 'stderr'
    log_directory: 'log'
    log_filename: 'postgresql-%a.log'
    log_rotation_age: 1440
    log_truncate_on_rotation: on
    log_rotation_size: 0
    log_replication_commands: on
    log_connections: on
    log_disconnections: on
    log_line_prefix: '%m [%p] %q%u@%d/%a %r '

  pg_hba:
    - local all all trust
    - host all all 0.0.0.0/0 trust
    - host replication replicator 0.0.0.0/0 trust
    - host replication rewind 0.0.0.0/0 trust

log:
  level: INFO
  dir: /var/log/patroni
  
```