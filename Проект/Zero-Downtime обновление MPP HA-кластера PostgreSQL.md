# План работ
1. Развернуть инфраструктуру
2. Наполнить данными (взять тренировочную БД от Postgres Pro)
3. Настроить симуляцию работы с СУБД (воспользоваться pgbench с кастомными запросами)
4. Настроить мониторинг (postgres_exporter, node_exporter, prometheus, Grafana, Loki + promtail)
	1. Производительность запросов: TPS, QPS, среднее время, топ медленных запросов. 
	2. Соединения: число активных, % использования `max_connections`
	3. Ресурсы сервера: CPU, память, swap, дисковый I/O, сеть.
	4. Кэш: Cache Hit Ratio (желательно >99%).
	5. Блокировки: ожидания, deadlocks, длительные транзакции.
	6. Репликация: lag, статус.
	7. Диск: размер БД, свободное место.
	8. Autovacuum: dead tuples, время последнего vacuum/analyze.
	9. Ошибки: логи PostgreSQL, отмены запросов.
5. Настроить РК (wal-g)
6. Обновить кластер

# Архитектура

- DCS --- 3 узла --- Etcd, etcd_exporter, node_exporter, Alloy
- Координатор Citus --- 3 узла (primary, sync standby, async standby) --- PostgreSQL, Patroni, wal-g, postgres_exporter, node_exporter, Alloy
- Worker1 Citus --- 3 узла (primary, sync standby, async standby) --- PostgreSQL, Patroni, wal-g, postgres_exporter, node_exporter, Alloy
- Worker2 Citus --- 3 узла (primary, sync standby, async standby) --- PostgreSQL, Patroni, wal-g, postgres_exporter, node_exporter, Alloy
- Балансировщики --- 2 узла --- HAProxy, PgBouncer, Keepalived, haproxy_exporter, pgbouncer_exporter, node_exporter, Alloy, keepalived_exporter
- Мониторинг --- 1 узел --- Prometheus, Alertmanager, Grafana, Loki, Alloy, node_exporter, docker
- Резервное копирование --- 1 узел --- MinIO, NFS-сервер, node-exporter, Alloy