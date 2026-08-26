
```
# hosts.yml

all:
  hosts:
    dcs1:
      ansible_host: 10.10.0.10
    dcs2:
      ansible_host: 10.10.0.11
    dcs3:
      ansible_host: 10.10.0.12
    coord1:
      ansible_host: 10.10.0.20
    coord2:
      ansible_host: 10.10.0.21
    worker11:
      ansible_host: 10.10.0.30
    worker12:
      ansible_host: 10.10.0.31
    worker21:
      ansible_host: 10.10.0.40
    worker22:
      ansible_host: 10.10.0.41
    balancer:
      ansible_host: 10.10.0.50
  children:
    dcs:
      hosts:
        dcs1:
        dcs2:
        dcs3:
    coord:
      hosts:
        coord1:
        coord2:
    workers:
      hosts:
        worker11:
        worker12:
        worker21:
        worker22:
    balancers:
      hosts:
        balancer:
    postgres_cluster:
      children:
        coord:
        workers:
```