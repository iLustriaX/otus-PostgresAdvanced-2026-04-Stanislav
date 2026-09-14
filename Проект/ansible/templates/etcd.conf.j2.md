
```
data-dir: /var/lib/etcd/
name: {{ inventory_hostname }}
initial-advertise-peer-urls: http://{{ ansible_host }}:2380
listen-peer-urls: http://0.0.0.0:2380
advertise-client-urls: http://{{ ansible_host }}:2379
listen-client-urls: http://0.0.0.0:2379
initial-cluster-state: new
initial-cluster: dcs1=http://{{ hostvars['dcs1'].ansible_host }}:2380,dcs2=http://{{ hostvars['dcs2'].ansible_host }}:2380,dcs3=http://{{ hostvars['dcs3'].ansible_host }}:2380
```