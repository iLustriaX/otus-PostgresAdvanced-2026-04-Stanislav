
```
# terraform.tfvars

cloud_id                  = "b1gmqqn8huu6s39112te"
folder_id                 = "b1gk5pe5h3541d01896e"
zone                      = "ru-central1-a"

service_account_key_file  = "key.json"

ssh_public_key            = "~/terraform/ssh-postgresql-update"

network_name              = "default"
subnet_name               = "postgresql-update"

vm_resources = {
  dcs      = { cores = 2, memory = 4,  disk_size = 20, disk_type = "network-ssd" }
  coord    = { cores = 2, memory = 4,  disk_size = 20, disk_type = "network-ssd" }
  worker1  = { cores = 2, memory = 4,  disk_size = 20, disk_type = "network-ssd" }
  worker2  = { cores = 2, memory = 4,  disk_size = 20, disk_type = "network-ssd" }
  balancer = { cores = 2, memory = 4,  disk_size = 20, disk_type = "network-ssd" }
  monitor  = { cores = 2, memory = 4,  disk_size = 20, disk_type = "network-ssd" }
  rc       = { cores = 2, memory = 4,  disk_size = 30, disk_type = "network-ssd" }
}

vm_count = {
  dcs      = 3
  coord    = 3
  worker1  = 3
  worker2  = 3
  balancer = 2
  monitor  = 1
  rc       = 1
}

vm_ips = {
  dcs      = ["10.10.0.10", "10.10.0.11", "10.10.0.12"]
  coord    = ["10.10.0.20", "10.10.0.21", "10.10.0.22"]
  worker1  = ["10.10.0.30", "10.10.0.31", "10.10.0.32"]
  worker2  = ["10.10.0.40", "10.10.0.41", "10.10.0.42"]
  balancer = ["10.10.0.50", "10.10.0.51"]
  monitor  = ["10.10.0.60"]
  rc       = ["10.10.0.80"]
}
```