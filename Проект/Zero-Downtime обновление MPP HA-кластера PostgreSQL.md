# План работ
1. Развернуть инфраструктуру 
   - DCS: etcd (3 узла), etcd_exporter, node_exporter, Alloy
   - Coord, Worker1, Worker2: PostgreSQL 16 + Patroni + Citus, wal-g, postgres_exporter, node_exporter, Alloy
   - Balancer: HAProxy, PgBouncer, Keepalived, haproxy_exporter, pgbouncer_exporter, keepalived_exporter, node_exporter, Alloy
   - Monitor: Prometheus, Alertmanager, Grafana, Loki, Alloy, node_exporter
   - RC: MinIO, NFS-сервер, node_exporter, Alloy
   - Настроить Patroni (управление кластерами через etcd), Citus (координатор + воркеры), HAProxy (порты 5000/5001 для клиентов, 5010/5020 для воркеров), PgBouncer (пулы primary/standby для координатора)
   - Настроить Keepalived (общий VIP для балансировщиков)
   - Проверить работу репликации и переключения (ручной failover)
2. Настроить мониторинг
   - Prometheus: сбор метрик с node_exporter, postgres_exporter, etcd_exporter, haproxy_exporter, pgbouncer_exporter, keepalived_exporter
   - Grafana: дашборды для:
     - Производительность запросов: TPS, QPS, среднее время, топ медленных запросов
     - Соединения: число активных, % использования max_connections
     - Ресурсы сервера: CPU, память, swap, дисковый I/O, сеть
     - Кэш: Cache Hit Ratio (>99%)
     - Блокировки: ожидания, deadlocks, длительные транзакции
     - Репликация: lag, статус
     - Диск: размер БД, свободное место
     - Autovacuum: dead tuples, время последнего vacuum/analyze
     - Ошибки: логи PostgreSQL, отмены запросов
   - Loki + Alloy: сбор логов PostgreSQL, Patroni, HAProxy, PgBouncer
   - Alertmanager: правила алертов (например, высокая задержка репликации, переполнение диска)
3. Настроить резервное копирование
   - wal-g: полные и инкрементальные бэкапы баз данных в MinIO (S3)
   - Настроить архивирование WAL в MinIO
   - Настроить снапшоты etcd и сохранение на NFS (RC)
   - Проверить восстановление из бэкапа на тестовом узле (опционально)
4. Наполнить кластер данными
   - Загрузить тренировочную БД от Postgres Pro (например, demo-база среднего размера)
   - Распределить таблицы по воркерам с помощью Citus (create_distributed_table)
   - Убедиться, что данные корректно распределены и доступны через координатор
5. Настроить симуляцию нагрузки
   - Создать кастомные скрипты pgbench, имитирующие типичную рабочую нагрузку (смесь чтения/записи, распределённые запросы)
   - Запустить нагрузку на постоянной основе (например, 50-70% от максимума) для создания фоновой активности
   - Зафиксировать базовые показатели мониторинга (TPS, QPS, latency, ошибки)
6. Выполнить обновление кластера
   - Использовать Patroni для поочерёдного обновления узлов:
     a) Обновить standby-узлы каждого кластера (Coord, Worker1, Worker2)
     b) Переключить primary на обновлённый standby (switchover)
     c) Обновить бывшие primary (теперь standby)
     d) Повторить для каждого кластера по очереди
   - Во время обновления поддерживать нагрузку (pgbench продолжает работать)
   - Контролировать метрики: TPS, latency, ошибки подключения, lag репликации — убедиться в отсутствии даунтайма
   - Проверить корректность работы Citus после обновления всех узлов
   - Зафиксировать результаты (сравнение до/после, отсутствие простоев)

# Архитектура

| Hostname  | IP-адрес   | Роль     | Компоненты                                                                                                      |
| --------- | ---------- | -------- | --------------------------------------------------------------------------------------------------------------- |
| dcs1      | 10.10.0.10 | DCS      | Etcd, etcd_exporter, node_exporter, Alloy                                                                       |
| dcs2      | 10.10.0.11 | DCS      | Etcd, etcd_exporter, node_exporter, Alloy                                                                       |
| dcs3      | 10.10.0.12 | DCS      | Etcd, etcd_exporter, node_exporter, Alloy                                                                       |
| coord1    | 10.10.0.20 | Coord    | PostgreSQL, Patroni, wal-g, postgres_exporter, node_exporter, Alloy                                             |
| coord2    | 10.10.0.21 | Coord    | PostgreSQL, Patroni, wal-g, postgres_exporter, node_exporter, Alloy                                             |
| coord3    | 10.10.0.22 | Coord    | PostgreSQL, Patroni, wal-g, postgres_exporter, node_exporter, Alloy                                             |
| worker11  | 10.10.0.30 | Worker1  | PostgreSQL, Patroni, wal-g, postgres_exporter, node_exporter, Alloy                                             |
| worker12  | 10.10.0.31 | Worker1  | PostgreSQL, Patroni, wal-g, postgres_exporter, node_exporter, Alloy                                             |
| worker13  | 10.10.0.32 | Worker1  | PostgreSQL, Patroni, wal-g, postgres_exporter, node_exporter, Alloy                                             |
| worker21  | 10.10.0.40 | Worker2  | PostgreSQL, Patroni, wal-g, postgres_exporter, node_exporter, Alloy                                             |
| worker22  | 10.10.0.41 | Worker2  | PostgreSQL, Patroni, wal-g, postgres_exporter, node_exporter, Alloy                                             |
| worker23  | 10.10.0.42 | Worker2  | PostgreSQL, Patroni, wal-g, postgres_exporter, node_exporter, Alloy                                             |
| balancer1 | 10.10.0.50 | Balancer | HAProxy, PgBouncer, Keepalived, haproxy_exporter, pgbouncer_exporter, node_exporter, Alloy, keepalived_exporter |
| balancer2 | 10.10.0.51 | Balancer | HAProxy, PgBouncer, Keepalived, haproxy_exporter, pgbouncer_exporter, node_exporter, Alloy, keepalived_exporter |
| monitor   | 10.10.0.60 | Monitor  | Prometheus, Alertmanager, Grafana, Loki, Alloy, node_exporter                                                   |
| rc        | 10.10.0.70 | RC       | MinIO, NFS-сервер, node_exporter, Alloy                                                                         |
| client    | 10.10.0.80 | client   | ansible, terraform, yc, pgbench                                                                                 |

# Сетевое взаимодействие

| Источник (hostname)                                                                                                                     | Получатель (hostname)                                                                                                                   | Протокол / Порт | Комментарий                                                      |
| --------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- | --------------- | ---------------------------------------------------------------- |
| dcs1, dcs2, dcs3                                                                                                                        | dcs1, dcs2, dcs3                                                                                                                        | TCP/2380        | Raft-репликация между etcd-узлами                                |
| coord1, coord2, coord3, worker11, worker12, worker13, worker21, worker22, worker23                                                      | dcs1, dcs2, dcs3                                                                                                                        | TCP/2379        | Хранение состояния кластера patroni, выбор лидера, конфигурация  |
| dcs1, dcs2, dcs3                                                                                                                        | dcs1, dcs2, dcs3                                                                                                                        | TCP/2379        | Сбор метрик etcd (etcd_exporter → etcd локально)                 |
| coord1, coord2, coord3                                                                                                                  | coord1, coord2, coord3                                                                                                                  | TCP/5432        | Потоковая репликация внутри Coordinator                          |
| worker11, worker12, worker13                                                                                                            | worker11, worker12, worker13                                                                                                            | TCP/5432        | Потоковая репликация внутри Worker1                              |
| worker21, worker22, worker23                                                                                                            | worker21, worker22, worker23                                                                                                            | TCP/5432        | Потоковая репликация внутри Worker2                              |
| coord1, coord2, coord3                                                                                                                  | coord1, coord2, coord3                                                                                                                  | TCP/5432        | Patroni управляет локальным PostgreSQL (Coord)                   |
| worker11, worker12, worker13                                                                                                            | worker11, worker12, worker13                                                                                                            | TCP/5432        | Patroni управляет локальным PostgreSQL (Worker1)                 |
| worker21, worker22, worker23                                                                                                            | worker21, worker22, worker23                                                                                                            | TCP/5432        | Patroni управляет локальным PostgreSQL (Worker2)                 |
| balancer1, balancer2                                                                                                                    | balancer1, balancer2                                                                                                                    | TCP/5000        | PgBouncer → локальный HAProxy (порт 5000, для пула coord_rw)     |
| balancer1, balancer2                                                                                                                    | balancer1, balancer2                                                                                                                    | TCP/5001        | PgBouncer → локальный HAProxy (порт 5001, для пула coord_r)      |
| balancer1, balancer2                                                                                                                    | coord1, coord2, coord3, worker11, worker12, worker13, worker21, worker22, worker23                                                      | TCP/8008        | Health‑check HAProxy → Patroni всех узлов                        |
| balancer1, balancer2                                                                                                                    | coord1, coord2, coord3                                                                                                                  | TCP/5432        | Проксирование трафика от клиента к primary/standby Coordinator   |
| coord1, coord2, coord3                                                                                                                  | balancer1, balancer2                                                                                                                    | TCP/5010        | Coordinator подключается к primary Worker1 через балансировщик   |
| coord1, coord2, coord3                                                                                                                  | balancer1, balancer2                                                                                                                    | TCP/5020        | Coordinator подключается к primary Worker2 через балансировщик   |
| balancer1, balancer2                                                                                                                    | worker11, worker12, worker13                                                                                                            | TCP/5432        | Проксирование трафика от координатора к текущему primary Worker1 |
| balancer1, balancer2                                                                                                                    | worker21, worker22, worker23                                                                                                            | TCP/5432        | Проксирование трафика от координатора к текущему primary Worker2 |
| balancer1                                                                                                                               | balancer2                                                                                                                               | VRRP            | Обмен VRRP-пакетами для выбора активного узла, управляющего VIP  |
| monitor                                                                                                                                 | dcs1, dcs2, dcs3, coord1, coord2, coord3, worker11, worker12, worker13, worker21, worker22, worker23, balancer1, balancer2, monitor, rc | TCP/9100        | Сбор системных метрик (node_exporter)                            |
| monitor                                                                                                                                 | coord1, coord2, coord3, worker11, worker12, worker13, worker21, worker22, worker23                                                      | TCP/9187        | Сбор метрик PostgreSQL (postgres_exporter)                       |
| monitor                                                                                                                                 | dcs1, dcs2, dcs3                                                                                                                        | TCP/9300        | Сбор метрик etcd (etcd_exporter)                                 |
| monitor                                                                                                                                 | balancer1, balancer2                                                                                                                    | TCP/9101        | Сбор метрик HAProxy (haproxy_exporter)                           |
| monitor                                                                                                                                 | balancer1, balancer2                                                                                                                    | TCP/9127        | Сбор метрик PgBouncer (pgbouncer_exporter)                       |
| monitor                                                                                                                                 | balancer1, balancer2                                                                                                                    | TCP/9650        | Сбор метрик Keepalived (keepalived_exporter)                     |
| dcs1, dcs2, dcs3, coord1, coord2, coord3, worker11, worker12, worker13, worker21, worker22, worker23, balancer1, balancer2, monitor, rc | monitor                                                                                                                                 | TCP/3100        | Отправка логов в Loki (Alloy → Loki)                             |
| monitor                                                                                                                                 | monitor                                                                                                                                 | TCP/9090        | Grafana → Prometheus (запрос метрик)                             |
| monitor                                                                                                                                 | monitor                                                                                                                                 | TCP/3100        | Grafana → Loki (запрос логов)                                    |
| monitor                                                                                                                                 | monitor                                                                                                                                 | TCP/9093        | Prometheus → Alertmanager (отправка алертов)                     |
| coord1, coord2, coord3, worker11, worker12, worker13, worker21, worker22, worker23                                                      | rc                                                                                                                                      | TCP/9000        | wal‑g → MinIO (бэкапы СУБД)                                      |
| dcs1, dcs2, dcs3                                                                                                                        | rc                                                                                                                                      | TCP/2049        | Сохранение снапшотов etcd на NFS‑шару                            |
| client                                                                                                                                  | balancer1, balancer2                                                                                                                    | TCP/6432        | Подключение клиента к pgbouncer                                  |


# Cхемы
## Информационный поток
![](../project/Pasted%20image%2020260825151039.png)
## Мониторинг
![](../project/Pasted%20image%2020260825145120.png)
## Резервное копирование
![](../project/Pasted%20image%2020260825123932.png)
## PostgreSQL репликация
![](../project/Pasted%20image%2020260825124119.png)
## Etcd кластер
![](../project/Pasted%20image%2020260825124154.png)
## Кластер балансировщиков
![](../project/Pasted%20image%2020260825124215.png)
## Взаимодействие Patroni и etcd
![](../project/Pasted%20image%2020260825124247.png)
# Разворачиваем инфраструктуру
## Client
Вручную разворачиваю ВМ Client в Yandex Cloud:
**ОС**: `Ubuntu 24.04`
**Зона доступности**: `ru-central1-a`
**Диск**: `SSD 20ГБ`
**Ресурсы**: `2vCPU, 2ГБ RAM`
**Подсеть**: `postgresql-update (10.10.0.0)`
**IP-адрес**: `Публичный автоматический, внутренний 10.10.0.70`
**Имя администратора**: `stanislav`
**Имя ВМ**: `client`

Команда подключения к ВМ:
```
ssh -i /path/to/key user@ip
```
```
ssh -i C:\Users\Mina-S\.ssh\ssh-postgresql-update stanislav@111.88.254.70
```

### Установка Terraform

Обновление системы:
```bash
sudo apt update && sudo apt upgrade -y
```

```bash
sudo apt install -y unzip
```

```bash
cd /tmp
```

Смотрю актуальную версию Terraform по ссылке: https://hashicorp-releases.yandexcloud.net/terraform/.
Сейчас `1.15.9.

Скачиваю Terraform:
```bash
wget https://hashicorp-releases.yandexcloud.net/terraform/1.15.9/terraform_1.15.9_linux_amd64.zip
```

Распаковываю архив:
```bash
unzip terraform_1.15.9_linux_amd64.zip
```

Переношу бинарник:
```bash
sudo mv terraform /usr/local/bin/
sudo chmod +x /usr/local/bin/terraform
```

Проверка установки:
```bash
terraform -v
```
![](../project/Pasted%20image%2020260825153142.png)

### Установка Ansible

Устанавливаю Ansible:
```bash
sudo apt install -y ansible
```

Проверка:
```bash
ansible --version
```
![](../project/Pasted%20image%2020260825153321.png)

### Установка YC

```bash
curl -sSL https://storage.yandexcloud.net/yandexcloud-yc/install.sh | bash
```

Перезагрузка оболочки:
```bash
source ~/.bashrc
```

Аутентифицируюсь в Yandex CLoud по инструкции: [Аутентификация в Yandex Cloud CLI на сервере без графического интерфейса | Yandex Cloud - Документация](https://yandex.cloud/ru/docs/cli/operations/authentication/auth-without-gui#yandex-account_2)

Проверка:
```bash
yc config list
```
![](../project/Pasted%20image%2020260825154906.png)
### Настройка YC для Terraform

Создаю сервисный аккаунт для Terraform в Yandex Cloud:
```bash
yc iam service-account create --name terraform --description "Service account for Terraform"
```

Назначаю роли сервисному аккаунту:
```bash
yc resource-manager folder add-access-binding \
  --id $(yc config get folder-id) \
  --role editor \
  --service-account-name terraform
```

Создаю ключ для terraform для аутентификации в Yandex Cloud:
```bash
yc iam key create \
  --service-account-name terraform \
  --output key.json
```

### Настройка Terraform для YC

Создаю директорию проекта:
```bash
mkdir ~/terraform && cd ~/terraform
```

Получаю `cloud-id`:
```bash
yc config get cloud-id
```
```
b1gmqqn8huu6s39112te
```

Получаю `folder-id`:
```bash
yc config get folder-id
```
```
b1gk5pe5h3541d01896e
```

Переношу ssh-ключ для ВМ:
```
scp -i ssh-postgresql-update ssh-postgresql-update stanislav@111.88.254.70:/home/stanislav/terraform
```

Создаю файл конфигурации Terraform CLI:
```bash
mkdir -p ~/.terraform.d && nano ~/.terraformrc
```
```yaml
provider_installation {
    network_mirror {
        url = "https://terraform-mirror.yandexcloud.net/"
        include = ["registry.terraform.io/*/*"]
}
    direct {
        exclude = ["registry.terraform.io/*/*"]
    }
}
```

Создаю файл настроек провайдера:
```bash
nano providers.tf
```
```yaml
terraform {
  required_providers {
    yandex = {
      source = "yandex-cloud/yandex"
    }
  }
  required_version = ">= 0.13"
}

provider "yandex" {
  service_account_key_file = "key.json"
  cloud_id  = "b1gmqqn8huu6s39112te"
  folder_id = "b1gk5pe5h3541d01896e"
  zone      = "ru-central1-a"
}
```

Создаю файл переменных:
```bash
nano variables.tf
```
```yaml
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

variable "ssh_public_key" {
  description = "Path to public SSH key"
  type        = string
}

variable "subnet_cidr" {
	description = "CIDR for subnet"
	type = string
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
```

Создаю файл значений переменных:
```bash
nano terraform.tfvars
```
```yaml
cloud_id = "b1gmqqn8huu6s39112te"
folder_id = "b1gk5pe5h3541d01896e"
zone = "ru-central1-a"
ssh_public_key = "/home/stanislav/terraform/ssh-postgresql-update"
subnet_cidr = "10.10.0.0/16"

vm_resources = {
  dcs      = { cores = 2, memory = 4,  disk_size = 20, disk_type = "network-ssd", preemptible = false, platform_id = "standard-v2" }
  coord    = { cores = 4, memory = 8,  disk_size = 50, disk_type = "network-ssd", preemptible = false, platform_id = "standard-v2" }
  worker   = { cores = 4, memory = 8,  disk_size = 50, disk_type = "network-ssd", preemptible = false, platform_id = "standard-v2" }
  balancer = { cores = 2, memory = 4,  disk_size = 20, disk_type = "network-ssd", preemptible = false, platform_id = "standard-v2" }
  monitor  = { cores = 2, memory = 4,  disk_size = 20, disk_type = "network-ssd", preemptible = false, platform_id = "standard-v2" }
  rc       = { cores = 2, memory = 4,  disk_size = 100, disk_type = "network-ssd", preemptible = false, platform_id = "standard-v2" }
}
```
