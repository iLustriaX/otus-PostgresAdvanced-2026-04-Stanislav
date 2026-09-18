
```
---
- name: Install etcd
  hosts: dcs
  become: yes

  tasks:
    - name: Install etcd
      apt:
        update_cache: yes
        name:
          - etcd-server
          - etcd-client
        state: present

    - name: Render etcd environment config
      template:
        src: ~/ansible/templates/etcd.conf.j2
        dest: /etc/default/etcd

    - name: Enable and start etcd
      systemd:
        name: etcd
        enabled: yes
        state: restarted
```