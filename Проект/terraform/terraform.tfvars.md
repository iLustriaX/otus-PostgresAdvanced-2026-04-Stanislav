
```
# terraform.tfvars

cloud_id                  = "b1gmqqn8huu6s39112te"
folder_id                 = "b1gk5pe5h3541d01896e"
zone                      = "ru-central1-a"

service_account_key_file  = "key.json"

ssh_public_key            = "/home/stanislav/terraform/ssh-postgresql-update.pub"

network_name              = "default"
subnet_name               = "postgresql-update"

vm_resources = {
  dcs      = { cores = 2, memory = 2,  disk_size = 20, disk_type = "network-ssd" }
  coord    = { cores = 2, memory = 2,  disk_size = 30, disk_type = "network-hdd" }
  worker1  = { cores = 2, memory = 2,  disk_size = 30, disk_type = "network-hdd" }
  worker2  = { cores = 2, memory = 2,  disk_size = 30, disk_type = "network-hdd" }
  balancer = { cores = 2, memory = 2,  disk_size = 20, disk_type = "network-hdd" }
}

vm_count = {
  dcs      = 3
  coord    = 2
  worker1  = 2
  worker2  = 2
  balancer = 1
}

vm_ips = {
  dcs      = ["10.10.0.10", "10.10.0.11", "10.10.0.12"]
  coord    = ["10.10.0.20", "10.10.0.21"]
  worker1  = ["10.10.0.30", "10.10.0.31"]
  worker2  = ["10.10.0.40", "10.10.0.41"]
  balancer = ["10.10.0.50"]
}
```