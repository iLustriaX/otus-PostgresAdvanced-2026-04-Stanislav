
```
---
# install_etcd.yml

- name: Install Etcd
  hosts: dcs
  become: yes
  vars:
    etcd_version: "v3.7.1"
    etcd_install_dir: "/opt/etcd/bin"
    etcd_config_dir: "/etc/etcd"
    etcd_download_url: "https://github.com/etcd-io/etcd/releases/download/{{ etcd_version }}/etcd-{{ etcd_version }}-linux-amd64.tar.gz"

  tasks:
    - name: Update and upgrade apt packages
      apt:
        update_cache: yes
        upgrade: full

    - name: Create directory for etcd binaries
      file:
        path: "{{ etcd_install_dir }}"
        state: directory

    - name: Create etcd config directory
      file:
        path: "{{ etcd_config_dir }}"
        state: directory

    - name: Create etcd data directory
      file:
        path: /var/lib/etcd
        state: directory

    - name: Download the tarball into /tmp
      get_url:
        url: "{{ etcd_download_url }}"
        dest: /tmp/etcd.tar.gz
        force: yes

    - name: Extract the contents of the tarball
      unarchive:
        src: /tmp/etcd.tar.gz
        dest: "{{ etcd_install_dir }}"
        extra_opts:
          - --strip-components=1
        remote_src: yes

    - name: Add etcd binaries to PATH via profile.d
      copy:
        content: |
          export PATH="{{ etcd_install_dir }}:$PATH"
        dest: /etc/profile.d/etcd-path.sh

    - name: Render etcd config from template
      template:
        src: ~/ansible/templates/etcd.conf.j2
        dest: "{{ etcd_config_dir }}/etcd.conf.yml"

    - name: Create etcd systemd service
      copy:
        content: |
          [Unit]
          Description=etcd key-value store
          Documentation=https://etcd.io/docs/
          After=network-online.target
          Wants=network-online.target

          [Service]
          Type=notify
          ExecStart={{ etcd_install_dir }}/etcd --config-file={{ etcd_config_dir }}/etcd.conf.yml
          Restart=on-failure
          RestartSec=5
          LimitNOFILE=65536

          [Install]
          WantedBy=multi-user.target
        dest: /etc/systemd/system/etcd.service
      notify: Reload systemd and restart etcd

    - name: Enable and start etcd service
      systemd:
        name: etcd
        enabled: yes
        state: started
        daemon_reload: yes

    - name: Wait for etcd to be ready
      wait_for:
        host: 127.0.0.1
        port: 2379
        timeout: 30

    - name: Verify etcd cluster health
      command: "{{ etcd_install_dir }}/etcdctl endpoint health"
      environment:
        ETCDCTL_API: "3"
        ETCDCTL_ENDPOINTS: "http://{{ ansible_host }}:2379"
      register: etcd_health
      changed_when: false

    - name: Show etcd health
      debug:
        var: etcd_health.stdout

  handlers:
    - name: Reload systemd and restart etcd
      systemd:
        name: etcd
        state: restarted
        daemon_reload: yes
        
```