
```
---
# install_pgbouncer.yml

- name: Install PgBouncer
  hosts: balancers
  become: yes

  tasks:
    - name: Install PgBouncer
      apt:
        update_cache: yes
        name: pgbouncer
        state: present

    - name: Render PgBouncer config
      template:
        src: ~/ansible/templates/pgbouncer.ini.j2
        dest: /etc/pgbouncer/pgbouncer.ini
      notify: Restart PgBouncer

    - name: Create userlist with all current users
      copy:
        content: |
          "postgres" ""
          "replicator" ""
          "rewind" ""
        dest: /etc/pgbouncer/userlist.txt
      notify: Restart PgBouncer

    - name: Enable and start PgBouncer
      systemd:
        name: pgbouncer
        enabled: yes
        state: started

  handlers:
    - name: Restart PgBouncer
      systemd:
        name: pgbouncer
        state: restarted
        

```