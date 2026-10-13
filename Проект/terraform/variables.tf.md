
```
# variables.tf

variable "cloud_id" {
  description = "Yandex Cloud ID"
  type        = string
}

variable "folder_id" {
  description = "Yandex Cloud Folder ID"
  type        = string
}

variable "zone" {
  description = "Availability zone"
  type        = string
  default     = "ru-central1-a"
}

variable "service_account_key_file" {
  description = "Path to service account key file"
  type        = string
}

variable "ssh_public_key" {
  description = "Path to public SSH key"
  type        = string
}

variable "vm_resources" {
	description = "Resources for VMs by role"
	type = map(object({
		cores = number
		memory = number
		disk_size = number
		disk_type = string
	}))
}

variable "vm_count" {
	description = "Number of VMs by role"
	type = map(number)
}

variable "vm_ips" {
	description = "Static IPs for VMs"
	type = map(list(string))
}

variable "network_name" {
  description = "Name of existing VPC network"
  type        = string
}

variable "subnet_name" {
  description = "Name of existing subnet"
  type        = string
}
```