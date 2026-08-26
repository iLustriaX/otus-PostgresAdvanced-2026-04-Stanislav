
```
# ansible.cfg

[defaults]
inventory = hosts.yml

remote_user = stanislav
private_key_file = ssh-postgresql-update
host_key_checking = False

forks = 20

become = True
become_method = sudo
become_user = root
become_ask_pass = False

deprecation_warnings = False

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
pipelining = True
```