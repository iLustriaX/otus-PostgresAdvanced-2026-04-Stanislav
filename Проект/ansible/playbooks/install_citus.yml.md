
```
---
- name: Install Citus package
  hosts: patroni
  become: true
  tasks:
    - name: Add Citus repo
      shell: curl https://install.citusdata.com/community/deb.sh | bash
      args:
        creates: /etc/apt/sources.list.d/citusdata_community.list

    - name: Install postgresql-16-citus-14.2
      apt:
        update_cache: true
        name: postgresql-16-citus-14.2
        state: present

- name: Create citus extension on worker1
  hosts: worker_cluster1[0]
  become: true
  tasks:
    - name: Create extension citus
      shell: |
        psql -d postgres -U postgres -h balancer -p 5010 -v ON_ERROR_STOP=1 \
          -c "CREATE EXTENSION IF NOT EXISTS citus;"
          
- name: Create citus extension on worker1
  hosts: worker_cluster2[0]
  become: true
  tasks:
    - name: Create extension citus
      shell: |
        psql -d postgres -U postgres -h balancer -p 5020 -v ON_ERROR_STOP=1 \
          -c "CREATE EXTENSION IF NOT EXISTS citus;"

- name: Init Citus extension and register nodes
  hosts: coord[0]
  become: true
  tasks:
    - name: Run init SQL
      shell: |
        psql -d coord-16 -U postgres -h balancer -p 6432 -v ON_ERROR_STOP=1 <<'SQL'
        CREATE EXTENSION IF NOT EXISTS citus;
        SELECT citus_set_coordinator_host('balancer', 5000);
        SELECT citus_add_node('balancer', 5010);
        SELECT citus_add_node('balancer', 5020);
        SQL

- name: Create tables and load data
  hosts: coord[0]
  become: true
  become_user: postgres
  tasks:
    - name: Download CSVs
      get_url:
        url: "https://api.pgxn.org/src/citus/citus-11.3.0/src/test/regress/data/{{ item }}"
        dest: "/tmp/{{ item }}"
      loop: [companies.csv, campaigns.csv, ads.csv]

    - name: Create tables
      shell: |
        psql -d coord-16 -U postgres -h balancer -p 6432 -v ON_ERROR_STOP=1 <<'SQL'
        CREATE TABLE IF NOT EXISTS companies (
            id bigint NOT NULL,
            name text NOT NULL,
            image_url text,
            created_at timestamp without time zone NOT NULL,
            updated_at timestamp without time zone NOT NULL
        );
        CREATE TABLE IF NOT EXISTS campaigns (
            id bigint NOT NULL,
            company_id bigint NOT NULL,
            name text NOT NULL,
            cost_model text NOT NULL,
            state text NOT NULL,
            monthly_budget bigint,
            blacklisted_site_urls text[],
            created_at timestamp without time zone NOT NULL,
            updated_at timestamp without time zone NOT NULL
        );
        CREATE TABLE IF NOT EXISTS ads (
            id bigint NOT NULL,
            company_id bigint NOT NULL,
            campaign_id bigint NOT NULL,
            name text NOT NULL,
            image_url text,
            target_url text,
            impressions_count bigint DEFAULT 0,
            clicks_count bigint DEFAULT 0,
            created_at timestamp without time zone NOT NULL,
            updated_at timestamp without time zone NOT NULL
        );
        SQL

    - name: Add primary keys
      shell: |
        psql -d coord-16 -U postgres -h balancer -p 6432 -v ON_ERROR_STOP=1 <<'SQL'
        ALTER TABLE companies ADD PRIMARY KEY (id);
        ALTER TABLE campaigns ADD PRIMARY KEY (id, company_id);
        ALTER TABLE ads       ADD PRIMARY KEY (id, company_id);
        SQL

    - name: Distribute tables
      shell: |
        psql -d coord-16 -U postgres -h balancer -p 6432 -v ON_ERROR_STOP=1 <<'SQL'
        SELECT create_distributed_table('companies', 'id');
        SELECT create_distributed_table('campaigns', 'company_id');
        SELECT create_distributed_table('ads',       'company_id');
        SQL

    - name: Load CSVs
      shell: |
        psql -d coord-16 -U postgres -h balancer -p 6432 -v ON_ERROR_STOP=1 <<'SQL'
        \copy companies from '/tmp/companies.csv' with csv
        \copy campaigns from '/tmp/campaigns.csv' with csv
        \copy ads       from '/tmp/ads.csv'       with csv
        SQL

```