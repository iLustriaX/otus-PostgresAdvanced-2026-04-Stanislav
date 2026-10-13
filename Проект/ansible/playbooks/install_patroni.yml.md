
```
---
- name: Install Patroni
  hosts: patroni
  become: yes

  tasks:
    - name: Install Patroni
      apt:
        update_cache: yes
        name: patroni
        state: present

    - name: Create directories
      file:
        path: "{{ item }}"
        state: directory
        owner: postgres
        group: postgres
        mode: "0700"
      loop:
        - "/etc/patroni"
        - "/var/log/patroni"
        - "/var/lib/postgresql/16/patroni"

    - name: Render Patroni config
      template:
        src: ~/ansible/templates/patroni.yml.j2
        dest: "/etc/patroni/config.yml"
        owner: postgres
        group: postgres
        mode: "0640"

    - name: Enable and start Patroni
      systemd:
        name: patroni
        enabled: yes
        state: started
        daemon_reload: yes
```