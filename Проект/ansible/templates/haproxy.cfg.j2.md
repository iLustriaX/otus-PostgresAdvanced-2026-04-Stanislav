
```
global
    maxconn 1000
    log stdout format raw local0

defaults
    log     global
    mode    tcp
    option  tcplog
    timeout connect 5s
    timeout client  30s
    timeout server  30s

# Coord Primary PostgreSQL-16 — port 5000
listen coord_primary_pg16
    bind *:5000
    mode tcp
    option httpchk GET /
    http-check expect status 200
    server coord1 {{ hostvars['coord1'].ansible_host }}:5432 check port 8008 inter 3s fall 3 rise 2
    server coord2 {{ hostvars['coord2'].ansible_host }}:5432 check port 8008 inter 3s fall 3 rise 2

# Worker1 Primary PostgreSQL-16 — port 5010
listen worker1_primary_pg16
    bind *:5010
    mode tcp
    option httpchk GET /
    http-check expect status 200
    server worker11 {{ hostvars['worker11'].ansible_host }}:5432 check port 8008 inter 3s fall 3 rise 2
    server worker12 {{ hostvars['worker12'].ansible_host }}:5432 check port 8008 inter 3s fall 3 rise 2

# Worker2 Primary PostgreSQL-16 — port 5020
listen worker2_primary_pg16
    bind *:5020
    mode tcp
    option httpchk GET /
    http-check expect status 200
    server worker21 {{ hostvars['worker21'].ansible_host }}:5432 check port 8008 inter 3s fall 3 rise 2
    server worker22 {{ hostvars['worker22'].ansible_host }}:5432 check port 8008 inter 3s fall 3 rise 2

```