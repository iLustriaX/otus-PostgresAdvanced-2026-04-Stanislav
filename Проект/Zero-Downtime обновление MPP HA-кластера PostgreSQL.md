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
# Разворачиваю инфраструктуру
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

Обновление системы:
```bash
sudo apt update && sudo apt upgrade -y
```

Установка клиента psql и pgbench:
```
sudo apt install postgresql
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
mkdir -p ~/ansible
```

```
cp ~/terraform/ssh-postgresql-update ~/ansible
```

```
nano ~/ansible/ansible.cfg
```

```
nano ~/ansible/inventory.yml
```

```
mkdir -p ~/ansible/playbooks
```

```
nano ~/ansible/playbooks/install_postgresql_16.yml
```

```
nano ~/ansible/playbooks/install_etcd.yml
```

```
nano ~/ansible/playbooks/install_patroni.yml
```

```
nano ~/ansible/playbooks/install_haproxy.yml
```

```
nano ~/ansible/playbooks/install_pgbouncer.yml
```

```
nano ~/ansible/playbooks/install_citus.yml
```

```
nano ~/ansible/playbooks/master.yml
```

```
mkdir ~/ansible/templates
```

```
nano ~/ansible/templates/etcd.conf.j2
```

```
nano ~/ansible/templates/patroni.yml.j2
```

```
nano ~/ansible/templates/haproxy.cfg.j2
```

```
nano ~/ansible/templates/pgbouncer.ini.j2
```

```
cd ~/ansible
```

```
ansible-playbook ~/ansible/playbooks/master.yml
```

Проверки:
```
ssh -i ~/ansible/ssh-postgresql-update ubuntu@dcs1

sudo ETCDCTL_API=3 etcdctl --endpoints="http://10.10.0.10:2379,http://10.10.0.11:2379,http://10.10.0.12:2379" endpoint status -w table 
```

```
ssh -i ~/ansible/ssh-postgresql-update ubuntu@coord1

sudo patronictl -c /etc/patroni/config.yml topology
```

```
psql -d coord-16 -U postgres -p 6432 -h balancer
```

```
SELECT * FROM citus_check_cluster_node_health();
```

# pgbench

```
nano /tmp/select.sql
```
```
-- 1. Список кампаний с ненулевым бюджетом (простой фильтр + сортировка)
SELECT id, name, state, monthly_budget
FROM campaigns
WHERE monthly_budget IS NOT NULL
ORDER BY monthly_budget DESC
LIMIT 10;
-- 2. Количество рекламных объявлений по штатам кампаний (агрегация + JOIN)
SELECT cmp.state, count(*) AS ads_count
FROM ads a
JOIN campaigns cmp ON cmp.id = a.campaign_id AND cmp.company_id = a.company_id
GROUP BY cmp.state;
```

```
nano /tmp/insert.sql
```
```
\set id random(1, 99999999)

INSERT INTO companies (id, name, created_at, updated_at)
VALUES (:client_id * 100000000000 + :id,
        'Company_' || :id,
        now(), now());
```

```
pgbench -f /tmp/select.sql -c 10 -j 2 -T 600 -P 5 -n -h balancer -p 6432 -U postgres coord-16
```
```
pgbench -f /tmp/insert.sql -c 10 -j 2 -T 600 -P 5 -n -h balancer -p 6432 -U postgres coord-16
```

```
ssh -i ~/ansible/ssh-postgresql-update ubuntu@balancer
```

```
psql -h balancer -p 6432 -U postgres pgbouncer
```
```
PAUSE coord;
```

```
ssh -i ~/ansible/ssh-postgresql-update ubuntu@coord1
```
```
sudo patronictl -c /etc/patroni/config.yml switchover
```

```
RESUME coord;
```

# pg_upgrade

```
ssh -i ~/ansible/ssh-postgresql-update ubuntu@balancer
```

```
sudo nano /etc/haproxy/haproxy.cfg
```
```
global
    maxconn 1000
    log stdout format raw local0

defaults
    log     global
    mode    tcp
    option  tcplog
    timeout connect 5s
    timeout client  30s
    timeout server  30s

# Coord Primary PostgreSQL-16 — port 5000
listen coord_primary_pg16
    bind *:5000
    mode tcp
    option httpchk GET /
    http-check expect status 200
    server coord1 10.10.0.20:5432 check port 8008 inter 3s fall 3 rise 2

# Worker1 Primary PostgreSQL-16 — port 5010
listen worker1_primary_pg16
    bind *:5010
    mode tcp
    option httpchk GET /
    http-check expect status 200
    server worker11 10.10.0.30:5432 check port 8008 inter 3s fall 3 rise 2

# Worker2 Primary PostgreSQL-16 — port 5020
listen worker2_primary_pg16
    bind *:5020
    mode tcp
    option httpchk GET /
    http-check expect status 200
    server worker21 10.10.0.40:5432 check port 8008 inter 3s fall 3 rise 2


# Coord Primary PostgreSQL-18 — port 5001
listen coord_primary_pg18
    bind *:5001
    mode tcp
    option httpchk GET /
    http-check expect status 200
    server coord1 10.10.0.21:5432 check port 8008 inter 3s fall 3 rise 2

# Worker1 Primary PostgreSQL-18 — port 5011
listen worker1_primary_pg18
    bind *:5011
    mode tcp
    option httpchk GET /
    http-check expect status 200
    server worker11 10.10.0.31:5432 check port 8008 inter 3s fall 3 rise 2

# Worker2 Primary PostgreSQL-18 — port 5021
listen worker2_primary_pg18
    bind *:5021
    mode tcp
    option httpchk GET /
    http-check expect status 200
    server worker21 10.10.0.41:5432 check port 8008 inter 3s fall 3 rise 2
```

```
sudo systemctl reload haproxy
```

```
sudo nano /etc/pgbouncer/pgbouncer.ini
```
```
coord-18      = host=127.0.0.1 port=5001 dbname=postgres
worker1-18    = host=127.0.0.1 port=5011 dbname=postgres
worker2-18    = host=127.0.0.1 port=5021 dbname=postgres
```
```
sudo systemctl reload pgbouncer
```

```
ssh -i ~/ansible/ssh-postgresql-update ubuntu@coord2
ssh -i ~/ansible/ssh-postgresql-update ubuntu@worker12
ssh -i ~/ansible/ssh-postgresql-update ubuntu@worker22
```

```
sudo patronictl -c /etc/patroni/config.yml topology
sudo patronictl -c /etc/patroni/config.yml switchover --force
```

```
sudo systemctl stop patroni
sudo systemctl disable patroni
```

```
sudo apt-get install postgresql-18 postgresql-18-citus-14.2 -y
```

```
sudo mkdir -p /var/lib/postgresql/18/patroni
sudo chown -R postgres:postgres /var/lib/postgresql/18
sudo chmod 700 /var/lib/postgresql/18/patroni
```

```
sudo nano /etc/patroni/config.yml
```
```
namespace: /postgresql-18

postgresql:
  data_dir: /var/lib/postgresql/18/patroni
  bin_dir: /usr/lib/postgresql/18/bin
```

```
sudo systemctl start patroni
sudo systemctl enable patroni
```

```
psql -d coord-18 -U postgres -p 6432 -h balancer
psql -d worker1-18 -U postgres -p 6432 -h balancer
psql -d worker2-18 -U postgres -p 6432 -h balancer
```
```
CREATE EXTENSION citus;
```

```
psql -d coord-18 -U postgres -p 6432 -h balancer
```
```
SELECT citus_set_coordinator_host('balancer', 5001);
SELECT citus_add_node('balancer', 5011);
SELECT citus_add_node('balancer', 5021);

SELECT * FROM citus_check_cluster_node_health();
```

```
ssh -i ~/ansible/ssh-postgresql-update ubuntu@coord2
```
```
sudo apt install pgcopydb
```
```
sudo pgcopydb clone --follow \
  --source "postgres://postgres@balancer:5000/postgres" \
  --target "postgres://postgres@balancer:5001/postgres" \
  --table-jobs 8 \
  --index-jobs 8 \
  --slot-name pgcopydb_slot \
  --create-slot
```

# Логическая репликация

```
psql -d coord-16 -p 6432 -U postgres -h balancer
```
```
CREATE PUBLICATION companies_pub FOR TABLE companies;
CREATE PUBLICATION campaigns_pub FOR TABLE campaigns;
CREATE PUBLICATION ads_pub FOR TABLE ads;
```
```
SELECT * FROM run_command_on_all_nodes($$ SELECT pg_create_logical_replication_slot('cdc_slot', 'pgoutput', false) $$);
```

```
psql -d coord-18 -p 6432 -U postgres -h balancer
```
```
CREATE SUBSCRIPTION coord_sub
    CONNECTION 'host=balancer port=5000 user=postgres dbname=postgres'
    PUBLICATION companies_pub, campaigns_pub, ads_pub
    WITH (copy_data = true, create_slot = false, slot_name = 'cdc_slot');
    
CREATE SUBSCRIPTION worker1_sub
    CONNECTION 'host=balancer port=5010 user=postgres dbname=postgres'
    PUBLICATION companies_pub, campaigns_pub, ads_pub
    WITH (copy_data = false, create_slot = false, slot_name = 'cdc_slot');
    
CREATE SUBSCRIPTION worker2_sub
    CONNECTION 'host=balancer port=5020 user=postgres dbname=postgres'
    PUBLICATION companies_pub, campaigns_pub, ads_pub
    WITH (copy_data = false, create_slot = false, slot_name = 'cdc_slot');
```

```
psql -d worker1-18 -p 6432 -U postgres -h balancer
```
```
CREATE SUBSCRIPTION coord_sub
    CONNECTION 'host=balancer port=5000 user=postgres dbname=postgres'
    PUBLICATION companies_pub, campaigns_pub, ads_pub
    WITH (copy_data = false, create_slot = false, slot_name = 'cdc_slot_worker1');
    
CREATE SUBSCRIPTION worker1_sub
    CONNECTION 'host=balancer port=5010 user=postgres dbname=postgres'
    PUBLICATION companies_pub, campaigns_pub, ads_pub
    WITH (copy_data = false, create_slot = false, slot_name = 'cdc_slot_worker1');
    
CREATE SUBSCRIPTION worker2_sub
    CONNECTION 'host=balancer port=5020 user=postgres dbname=postgres'
    PUBLICATION companies_pub, campaigns_pub, ads_pub
    WITH (copy_data = false, create_slot = false, slot_name = 'cdc_slot_worker1');
```

```
psql -d worker2-18 -p 6432 -U postgres -h balancer
```
```
CREATE SUBSCRIPTION coord_sub
    CONNECTION 'host=balancer port=5000 user=postgres dbname=postgres'
    PUBLICATION companies_pub, campaigns_pub, ads_pub
    WITH (copy_data = false, create_slot = false, slot_name = 'cdc_slot_worker2');
    
CREATE SUBSCRIPTION worker1_sub
    CONNECTION 'host=balancer port=5010 user=postgres dbname=postgres'
    PUBLICATION companies_pub, campaigns_pub, ads_pub
    WITH (copy_data = false, create_slot = false, slot_name = 'cdc_slot_worker2');
    
CREATE SUBSCRIPTION worker2_sub
    CONNECTION 'host=balancer port=5020 user=postgres dbname=postgres'
    PUBLICATION companies_pub, campaigns_pub, ads_pub
    WITH (copy_data = false, create_slot = false, slot_name = 'cdc_slot_worker2');
```

```
ALTER SUBSCRIPTION companies_sub DISABLE;
ALTER SUBSCRIPTION companies_sub SET (slot_name = NONE);
DROP SUBSCRIPTION companies_sub;

ALTER SUBSCRIPTION campaigns_sub DISABLE;
ALTER SUBSCRIPTION campaigns_sub SET (slot_name = NONE);
DROP SUBSCRIPTION campaigns_sub;

ALTER SUBSCRIPTION ads_sub DISABLE;
ALTER SUBSCRIPTION ads_sub SET (slot_name = NONE);
DROP SUBSCRIPTION ads_sub;
```

```
SELECT count(*) FROM companies;
SELECT count(*) FROM campaigns;
SELECT count(*) FROM ads;
```

```
Создание дампа схемы со старого кластера:
pg_dump --format=plain --no-owner --schema-only --file=schema.sql postgres://user:pass@old_coordinator:5432/dbname

Применение схемы на новом координаторе:
psql -f schema.sql postgres://user:pass@new_coordinator:5432/dbname

Распределение таблиц на новом кластере:  
SELECT create_distributed_table('companies', 'id');
SELECT create_distributed_table('campaigns', 'company_id');
SELECT create_distributed_table('ads', 'company_id');


На координаторе старого кластера создать публикации:
CREATE PUBLICATION companies_pub FOR TABLE companies;
CREATE PUBLICATION campaigns_pub FOR TABLE campaigns;
CREATE PUBLICATION ads_pub FOR TABLE ads;

#### 3.2. Создание слотов репликации на всех узлах старого кластера

На координаторе старого кластера создать слоты репликации:

SELECT * FROM run_command_on_all_nodes($$ SELECT pg_create_logical_replication_slot('cdc_slot', 'pgoutput', false) $$);

На координаторе нового кластера:

CREATE SUBSCRIPTION sub_from_coord
    CONNECTION 'host=old-coord port=5432 user=repl_user password=... dbname=your_db'
    PUBLICATION items_pub
    WITH (copy_data = true, create_slot = false, slot_name = 'cdc_slot');

На воркере 1:

CREATE SUBSCRIPTION sub_from_worker1
    CONNECTION 'host=old-worker-1 port=5432 user=repl_user password=... dbname=your_db'
    PUBLICATION items_pub
    WITH (copy_data = false, create_slot = false, slot_name = 'cdc_slot');

На воркере 2:
CREATE SUBSCRIPTION sub_from_worker2
    CONNECTION 'host=old-worker-2 port=5432 user=repl_user password=... dbname=your_db'
    PUBLICATION items_pub
    WITH (copy_data = false, create_slot = false, slot_name = 'cdc_slot');



#### 3.4. Мониторинг репликации
-- Статус подписок
SELECT * FROM pg_stat_subscription;
-- Прогресс начальной синхронизации
SELECT * FROM pg_stat_subscription_stats;
```
# Сложности
1. Не понял как создать ВМ с конкретным пользователем вместо ubuntu. Час потратил на то, чтобы понять почему не могу подключиться по ssh
2. Проблема с установкой пакетов на ВМ без публичного адреса

# Заметки:

[External Integrations - Citus for PostgreSQL | Microsoft Learn](https://learn.microsoft.com/en-us/postgresql/citus/integrations?view=citus-14)
[Big Database Migration - Citus for PostgreSQL | Microsoft Learn](https://learn.microsoft.com/en-us/postgresql/citus/migrate/migration-data-big?view=citus-14)
[Small Database Migration - Citus for PostgreSQL | Microsoft Learn](https://learn.microsoft.com/en-us/postgresql/citus/migrate/migration-data-small?view=citus-14)

Можно использовать pgcopydb для логической репликации

Используется 14 версия citus, т.к. она поддерживается в pg18.

Чтобы подключение клиента не разрывалось при переключении, нужно остановить пул в pgbouncer

При создании подписок нельзя подключаться через pgbouncer:
```
CREATE SUBSCRIPTION companies_sub
    CONNECTION 'host=balancer port=5000 user=postgres dbname=postgres'
    PUBLICATION companies_pub
    WITH (copy_data = true, create_slot = false, slot_name = 'cdc_slot');
```

Сообщение `unsupported startup parameter: replication` означает, что при установке соединения для логической репликации PostgreSQL передает в стартовом пакете параметр `replication`. PgBouncer не умеет обрабатывать этот параметр и разрывает соединение.
![](../project/Pasted%20image%2020260918104314.png)


Создавал по отдельнйо подписке на каждую публикацию. При этом использовал один слот публикации.
![](../project/Pasted%20image%2020260918115801.png)

```
SELECT count(*) FROM companies;
SELECT count(*) FROM campaigns;
SELECT count(*) FROM ads;
```

```
SELECT pg_size_pretty(citus_total_relation_size('companies'));
SELECT pg_size_pretty(citus_total_relation_size('ads'));
SELECT pg_size_pretty(citus_total_relation_size('campaigns'));
```

pgcopydb идёт вместе с репозиторием pgdg