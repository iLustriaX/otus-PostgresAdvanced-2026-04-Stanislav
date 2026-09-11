# План работ
1. Развернуть инфраструктуру 
2. Наполнить кластер данными
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

| Hostname | IP-адрес   | Роль     | Компоненты                      |
| -------- | ---------- | -------- | ------------------------------- |
| dcs1     | 10.10.0.10 | DCS      | Etcd                            |
| dcs2     | 10.10.0.11 | DCS      | Etcd                            |
| dcs3     | 10.10.0.12 | DCS      | Etcd                            |
| coord1   | 10.10.0.20 | Coord    | PostgreSQL, Patroni             |
| coord2   | 10.10.0.21 | Coord    | PostgreSQL, Patroni             |
| worker11 | 10.10.0.30 | Worker1  | PostgreSQL, Patroni             |
| worker12 | 10.10.0.31 | Worker1  | PostgreSQL, Patroni             |
| worker21 | 10.10.0.40 | Worker2  | PostgreSQL, Patroni             |
| worker22 | 10.10.0.41 | Worker2  | PostgreSQL, Patroni             |
| balancer | 10.10.0.50 | Balancer | HAProxy, PgBouncer              |
| client   | 10.10.0.70 | client   | ansible, terraform, yc, pgbench |

# Сетевое взаимодействие

| Источник (hostname)                                    | Получатель (hostname)                                  | Протокол / Порт | Комментарий                                                      |
| ------------------------------------------------------ | ------------------------------------------------------ | --------------- | ---------------------------------------------------------------- |
| dcs1, dcs2, dcs3                                       | dcs1, dcs2, dcs3                                       | TCP/2380        | Raft-репликация между etcd-узлами                                |
| coord1, coord2, worker11, worker12, worker21, worker22 | dcs1, dcs2, dcs3                                       | TCP/2379        | Хранение состояния кластера Patroni, выбор лидера, конфигурация  |
| coord1, coord2                                         | coord1, coord2                                         | TCP/5432        | Потоковая репликация внутри Coordinator                          |
| worker11, worker12                                     | worker11, worker12                                     | TCP/5432        | Потоковая репликация внутри Worker1                              |
| worker21, worker22                                     | worker21, worker22                                     | TCP/5432        | Потоковая репликация внутри Worker2                              |
| coord1, coord2                                         | coord1, coord2                                         | TCP/5432        | Patroni управляет локальным PostgreSQL (Coord)                   |
| worker11, worker12                                     | worker11, worker12                                     | TCP/5432        | Patroni управляет локальным PostgreSQL (Worker1)                 |
| worker21, worker22                                     | worker21, worker22                                     | TCP/5432        | Patroni управляет локальным PostgreSQL (Worker2)                 |
| balancer                                               | balancer                                               | TCP/5000        | PgBouncer → локальный HAProxy (порт 5000, для пула coord_rw)     |
| balancer                                               | balancer                                               | TCP/5001        | PgBouncer → локальный HAProxy (порт 5001, для пула coord_r)      |
| balancer                                               | coord1, coord2, worker11, worker12, worker21, worker22 | TCP/8008        | Health‑check HAProxy → Patroni всех узлов                        |
| balancer                                               | coord1, coord2                                         | TCP/5432        | Проксирование трафика от клиента к primary/standby Coordinator   |
| coord1, coord2                                         | balancer                                               | TCP/5010        | Coordinator подключается к primary Worker1 через балансировщик   |
| coord1, coord2                                         | balancer                                               | TCP/5020        | Coordinator подключается к primary Worker2 через балансировщик   |
| balancer                                               | worker11, worker12                                     | TCP/5432        | Проксирование трафика от координатора к текущему primary Worker1 |
| balancer                                               | worker21, worker22                                     | TCP/5432        | Проксирование трафика от координатора к текущему primary Worker2 |
| client                                                 | balancer                                               | TCP/6432        | Подключение клиента к PgBouncer                                  |
# Cхемы
## Информационный поток
![](../project/Pasted%20image%2020260826123510.png)
## PostgreSQL репликация
![](../project/Pasted%20image%2020260826123616.png)
## Etcd кластер
![](../project/Pasted%20image%2020260826123632.png)

## Взаимодействие Patroni и etcd
![](../project/Pasted%20image%2020260826123708.png)
# Разворачиваем инфраструктуру
## Client
Вручную разворачиваю ВМ Client в Yandex Cloud:
**ОС**: `Ubuntu 24.04`
**Зона доступности**: `ru-central1-a`
**Диск**: `HDD 20ГБ`
**Ресурсы**: `2vCPU, 2ГБ RAM`
**Подсеть**: 
	`postgresql-update (10.10.0.0)`
	`default-ru-central1-a (10.128.0.0)`
**IP-адрес**: `Без публичного, внутренний 10.10.0.70`
`Публичный динамический, внутренний 10.128.0.70`
**Имя администратора**: `stanislav`
**Имя ВМ**: `client`

Команда подключения к ВМ:
```
ssh -i /path/to/key user@ip
```
```
ssh -i C:\Users\Mina-S\.ssh\ssh-postgresql-update stanislav@51.250.76.220
```

Удалить дефолтный маршрут через сеть без внешнего IP:
```
sudo ip route del default via 10.10.0.1 dev eth0
```

Убедиться, что остался маршрут только через сеть с публичным адресом:
```
ip route show default
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

# Настройка шлюза

Включить пересылку пакетов:
```
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
```

Настроить masquerade для исходящего трафика:
```
sudo iptables -t nat -A POSTROUTING -o eth1 -j MASQUERADE
```

Сохранить правила iptables:
```
sudo apt update && sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

Создать таблицу маршрутов с default route на client:
```
yc vpc route-table create \
  --name nat-route-table \
  --network-name default \
  --route destination=0.0.0.0/0,next-hop=10.10.0.70
```

Привязать таблицу к существующей подсети:
```
yc vpc subnet update postgresql-update --route-table-name nat-route-table
```

Для отката изменений:
```
yc vpc subnet update postgresql-update --route-table-name ""
yc vpc route-table delete nat-route-table

sudo iptables -t nat -F
sudo sysctl -w net.ipv4.ip_forward=0
sudo sed -i '/net.ipv4.ip_forward/d' /etc/sysctl.conf
sudo systemctl restart systemd-networkd   # или netplan apply
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

Переношу закрытый и открытый ssh-ключи для ВМ:
```
scp -i ssh-postgresql-update ssh-postgresql-update stanislav@51.250.76.220:/home/stanislav/terraform

scp -i ssh-postgresql-update ssh-postgresql-update.pub stanislav@51.250.76.220:/home/stanislav/terraform
```

Переношу файл ключа сервиса terraform для YC:
```
mv /tmp/key.json ~/terraform/
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

Создаю файл переменных:
```bash
nano variables.tf
```

Создаю файл значений переменных:
```bash
nano terraform.tfvars
```

Создаю файл main.tf
```
nano main.tf
```

Инициализирую terraform:
```
terraform init
```

Форматирую файлы:
```
terraform fmt
```

Валидирую файлы:
```
terraform validate
```

Вывожу список планируемых ресурсов:
```
terraform plan
```

Запускаю создание ресурсов:
```
terraform apply
```

# Настройка Ansible

```
mkdir -p ~/ansible && cd ~/ansible
```

```
cp ~/terraform/ssh-postgresql-update ~/ansible
```

```
nano ansible.cfg
```

```
nano inventory.yml
```

```
mkdir -p playbooks && cd playbooks
```

```
nano install_postgresql_16.yml
```

# Сложности
1. Не понял как создать ВМ с конкретным пользователем вместо ubuntu. Час потратил на то, чтобы понять почему не могу подключиться по ssh
2. Проблема с установкой пакетов на ВМ без публичного адреса