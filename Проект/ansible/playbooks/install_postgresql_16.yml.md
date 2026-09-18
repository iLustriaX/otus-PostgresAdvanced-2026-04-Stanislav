
```
---
- name: Install PostgreSQL 16
  hosts: patroni
  become: yes

  tasks:
    - name: Download PostgreSQL repository key
      get_url:
        url: https://www.postgresql.org/media/keys/ACCC4CF8.asc
        dest: /etc/apt/trusted.gpg.d/postgresql.asc
        mode: "0644"

    - name: Add PostgreSQL repository
      apt_repository:
        repo: "deb http://apt.postgresql.org/pub/repos/apt {{ ansible_distribution_release }}-pgdg main"
        filename: pgdg
        state: present

    - name: Install PostgreSQL 16
      apt:
        update_cache: yes
        name: postgresql-16
        state: present

    - name: Stop and disable default PostgreSQL cluster
      systemd:
        name: postgresql
        state: stopped
        enabled: no
      ignore_errors: yes
      
```