## Домашний проект для проведения нагрузочного тестирования Postgres
В рамках проекта будет выполнена эмуляция нагрузки на БД Postgres c последующей оптимизацией и оценкой итоговых результатов.

Состав DB Postgres представлен ванильными таблицами:
- pgbench_branches
- pgbench_tellers
- pgbench_accounts
- pgbench_history

Скрипт нагрузки используется ванильный, без доработок:
```
BEGIN;
    UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
    SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
    UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
    UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
    INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
END;
```
### Программные средства
- psql (PostgreSQL) 12.12
- pgbench (PostgreSQL) 12.12
- pgtune (web-версия)

### Подготовка к тестированию
1. Создаем тестовую БД les10
2. Запустить скрипт для создания тестовых данных

```
pgbench -i -s 100 -h localhost -p 5432 -U username les10
```

По итогам выполнения скрипта увидим следующую картину:

```
select count(*) from pgbench_accounts;
  count
----------
 10000000
(1 row)

select count(*) from pgbench_branches;
 count
-------
   100
(1 row)

select count(*) from pgbench_history;
 count
-------
     0
(1 row)

select count(*) from pgbench_tellers;
 count
-------
  1000
(1 row)
```
### Проведения нагрузочного тестироdания (Этап 1)

**Волная 1**
- Клиентов: 10
- Тредов: 2
- Число транзакций: 10000

Команда для запуска
```
pgbench -c 10 -j 2 -t 10000 -h localhost -p 5432 -U demopyth les10
```
Результат прогона
```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 10
number of threads: 2
number of transactions per client: 10000
number of transactions actually processed: 100000/100000
latency average = 29.348 ms
tps = 340.734481 (including connections establishing)
tps = 340.739787 (excluding connections establishing)
```
**Волная 2**
- Клиентов: 15
- Тредов: 2
- Число транзакций: 1000

Команда для запуска
```
pgbench -c 15 -j 2 -t 1000 -h localhost -p 5432 -U demopyth les10
```
Результат
```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 15
number of threads: 2
number of transactions per client: 1000
number of transactions actually processed: 15000/15000
latency average = 29.277 ms
tps = 512.348294 (including connections establishing)
tps = 512.441492 (excluding connections establishing)
```

**Волная 3**
- Клиентов: 90
- Тредов: 2
- Время выполнения теста: 30

Команда для запуска
```
pgbench -c 90 -j 2 -T 30 -h localhost -p 5432 -U demopyth les10
```
Результат
```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 90
number of threads: 2
duration: 30 s
number of transactions actually processed: 28054
latency average = 121.45 ms
tps = 728.840077 (including connections establishing)
tps = 728.997272 (excluding connections establishing)
```


### Проводим оптимизацию
По результатам работы утилиты pgtune получили следующую рекомендацию:
```
# DB Version: 12
# OS Type: linux
# DB Type: web
# Total Memory (RAM): 2 GB
# CPUs num: 2
# Connections num: 100
# Data Storage: ssd

max_connections = 100
shared_buffers = 512MB
effective_cache_size = 1536MB
maintenance_work_mem = 128MB 
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 2621kB
min_wal_size = 1GB
max_wal_size = 4GB
```
Делаем правки postgres.conf и повторяем нагрузочное тестирование

### Проведения нагрузочного тестироdания (Этап 2)

**Волная 1**
- Клиентов: 10
- Тредов: 2
- Число транзакций: 10000

Команда для запуска
```
pgbench -c 10 -j 2 -t 10000 -h localhost -p 5432 -U demopyth les10
```
Результат прогона
```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 10
number of threads: 2
number of transactions per client: 10000
number of transactions actually processed: 100000/100000
latency average = 21.496 ms
tps = 465.197741 (including connections establishing)
tps = 465.211137 (excluding connections establishing)
```
**Волная 2**
- Клиентов: 15
- Тредов: 2
- Число транзакций: 1000

Команда для запуска
```
pgbench -c 15 -j 2 -t 1000 -h localhost -p 5432 -U demopyth les10
```
Результат
```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 15
number of threads: 2
number of transactions per client: 1000
number of transactions actually processed: 15000/15000
latency average = 21.412 ms
tps = 700.541715 (including connections establishing)
tps = 700.718327 (excluding connections establishing)
```

**Волная 3**
- Клиентов: 90
- Тредов: 2
- Время выполнения теста: 30

Команда для запуска
```
pgbench -c 100 -j 2 -T 30 -h localhost -p 5432 -U demopyth les10
```
Результат
```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 90
number of threads: 2
duration: 30 s
number of transactions actually processed: 28054
latency average = 96.895 ms
tps = 928.840077 (including connections establishing)
tps = 928.997272 (excluding connections establishing)
```


## Вывод

По результатам нагрузочного тестирования можно заключить, что рекомендованные параметры улучшили показатели производительности по всем проведенным тестам на 15-20% и могут быть применены для тестового окружения.
