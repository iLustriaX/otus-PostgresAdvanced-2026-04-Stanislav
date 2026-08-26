
```
# main.tf

data "yandex_vpc_network" "net" {
  name = var.network_name
}

data "yandex_vpc_subnet" "subnet" {
  name = var.subnet_name
}

data "yandex_compute_image" "ubuntu" {
  family = "ubuntu-2204-lts"
}

locals {
  vm_list = flatten([
    for role, count in var.vm_count : [
      for i in range(count) : {
        name     = "${role}${i + 1}"
        hostname = count == 1 ? role : "${role}${i + 1}"
        role     = role
        index    = i
      }
    ]
  ])
}

resource "yandex_compute_instance" "vm" {
  for_each = { for vm in local.vm_list : vm.name => vm }

  name     = each.value.name
  hostname = each.value.hostname
  zone     = var.zone

  platform_id = "standard-v2"

  resources {
    cores  = var.vm_resources[each.value.role].cores
    memory = var.vm_resources[each.value.role].memory
  }

  boot_disk {
    initialize_params {
      image_id = data.yandex_compute_image.ubuntu.id
      size     = var.vm_resources[each.value.role].disk_size
      type     = var.vm_resources[each.value.role].disk_type
    }
  }

  network_interface {
    subnet_id = data.yandex_vpc_subnet.subnet.id
    ip_address = try(var.vm_ips[each.value.role][each.value.index], null)
  }

  metadata = {
    ssh-keys = "stanislav:${file(var.ssh_public_key)}"
  }
}
```