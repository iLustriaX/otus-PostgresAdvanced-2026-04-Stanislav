
```
---
- name: Install PostgreSQL 16
  hosts: postgres_cluster
  become: yes
  
  tasks:
    - name: Update and upgrade
      apt:
        update_cache: yes
        upgrade: full
	
	- name: Download PostgreSQL repository key
	  get_url:
	    url: https://www.postgresql.org/media/keys/ACCC4CF8.asc
	    dest: /etc/apt/trusted.gpg.d/postgresql.asc
	    
	- name: Add PostgreSQL repository
	  apt_repository:
	    repo: "deb http://apt.postgresql.org/pub/repos/apt {{ ansible_distribution_release }}-pgdg main"
	    filename:pgdg
	    
	- name: Update
	  apt:
	    update_cache: yes
	    
	- name: Install PostgreSQL 16
	  apt:
	    name: postgresql-16
```