
```
[databases]
coord-16      = host=127.0.0.1 port=5000 dbname=postgres
worker1-16    = host=127.0.0.1 port=5010 dbname=postgres
worker2-16    = host=127.0.0.1 port=5020 dbname=postgres

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = trust
auth_file = /etc/pgbouncer/userlist.txt

pool_mode = transaction
max_client_conn = 200
default_pool_size = 20
reserve_pool_size = 5

ignore_startup_parameters = extra_float_digits, application_name, options

logfile = /var/log/postgresql/pgbouncer.log
pidfile = /var/run/postgresql/pgbouncer.pid
unix_socket_dir = /var/run/postgresql

admin_users = postgres


```