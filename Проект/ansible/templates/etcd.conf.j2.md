
```
ETCD_NAME={{ inventory_hostname }}
ETCD_DATA_DIR=/var/lib/etcd

ETCD_LISTEN_PEER_URLS=http://0.0.0.0:2380
ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379

ETCD_INITIAL_ADVERTISE_PEER_URLS=http://{{ ansible_host }}:2380
ETCD_ADVERTISE_CLIENT_URLS=http://{{ ansible_host }}:2379

ETCD_INITIAL_CLUSTER={{ hostvars['dcs1'].inventory_hostname }}=http://{{ hostvars['dcs1'].ansible_host }}:2380,{{ hostvars['dcs2'].inventory_hostname }}=http://{{ hostvars['dcs2'].ansible_host }}:2380,{{ hostvars['dcs3'].inventory_hostname }}=http://{{ hostvars['dcs3'].ansible_host }}:2380
ETCD_INITIAL_CLUSTER_TOKEN=patroni-cluster
ETCD_INITIAL_CLUSTER_STATE=new
```