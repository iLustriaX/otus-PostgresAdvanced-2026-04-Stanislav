
```
---
# install_haproxy.yml

- name: Install HAProxy
  hosts: balancers
  become: yes

  tasks:
    - name: Install HAProxy
      apt:
        update_cache: yes
        name: haproxy
        state: present

    - name: Render HAProxy config
      template:
        src: ~/ansible/templates/haproxy.cfg.j2
        dest: /etc/haproxy/haproxy.cfg
      notify: Restart HAProxy

    - name: Enable and start HAProxy
      systemd:
        name: haproxy
        enabled: yes
        state: started

  handlers:
    - name: Restart HAProxy
      systemd:
        name: haproxy
        state: restarted

```