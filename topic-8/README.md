# Домашнее задание

## Нагрузочное тестирование и тюнинг PostgreSQL
**Цель:**
- Сделать нагрузочное тестирование PostgreSQL.
- Настроить параметры PostgreSQL для достижения максимальной производительности.

**Описание/Пошаговая инструкция выполнения домашнего задания:**
1. Развернуть виртуальную машину любым удобным способом.
2. Поставить на неё PostgreSQL 15 любым способом.
3. Настроить кластер PostgreSQL 15 на максимальную производительность, не обращая внимание на возможные проблемы с надежностью в случае аварийной перезагрузки виртуальной машины.
4. Нагрузить кластер через утилиту pgbench ([ссылка](https://postgrespro.ru/docs/postgrespro/14/pgbench)).
5. Написать, какого значения TPS удалось достичь, показать, какие параметры в какие значения устанавливали и почему.

**Задание со *:**
Аналогично протестировать через утилиту [sysbench-tpcc](https://github.com/Percona-Lab/sysbench-tpcc) (требует установки [sysbench](https://github.com/akopytov/sysbench)).

**Критерии оценки:**
Выполнение ДЗ: 10 баллов
- 2 балла за красивое решение
- 2 балла за рабочее решение, и недостатки указанные преподавателем не устранены

# Домашняя работа к занятию "Настройка PostgreSQL" от 07.03.2024
## 1. Развернуть виртуальную машину
### 1.1 Создим инстанс ВМ с 2 ядрами и 4 Гб ОЗУ и SSD 10GB
```bash
➜  ~ yc compute instance create \
    --name otus-db-pg-1 \
    --zone ru-central1-d \
    --network-interface subnet-name=otus-subnet-1,nat-ip-version=ipv4 \
    --create-boot-disk size=10G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2004-lts \
    --memory=4 \
    --cores=2 \
    --ssh-key ~/.ssh/yc_key.pub
```
<details>
<summary>Детали</summary>

```bash
done (37s)
id: fv4nascp2arqci4kdat6
folder_id: b1gffa7irklogm13fdjq
created_at: "2024-05-02T21:26:59Z"
name: otus-db-pg-1
zone_id: ru-central1-d
platform_id: standard-v2
resources:
  memory: "4294967296"
  cores: "2"
  core_fraction: "100"
status: RUNNING
metadata_options:
  gce_http_endpoint: ENABLED
  aws_v1_http_endpoint: ENABLED
  gce_http_token: ENABLED
  aws_v1_http_token: DISABLED
boot_disk:
  mode: READ_WRITE
  device_name: fv4poj465cj699mght3b
  auto_delete: true
  disk_id: fv4poj465cj699mght3b
network_interfaces:
  - index: "0"
    mac_address: d0:0d:17:57:19:91
    subnet_id: fl8t7lqif5qa2aihdfvm
    primary_v4_address:
      address: 192.168.0.32
      one_to_one_nat:
        address: 158.160.158.134
        ip_version: IPV4
gpu_settings: {}
fqdn: fv4nascp2arqci4kdat6.auto.internal
scheduling_policy: {}
network_settings:
  type: STANDARD
placement_policy: {}
```
</details>

## 2. Поставим PostgreSQL 15.
### 2.1 Подключимся к ВМ
```bash
➜  ~ ssh otus-db-pg-1
Enter passphrase for key '/Users/iliakriachkov/.ssh/yc_key':
...
yc-user@fv4nascp2arqci4kdat6:~$
```

### 2.2 Установим PostgreSQL 15
```bash
yc-user@fv4nascp2arqci4kdat6:~$ sudo apt update
```
```bash
yc-user@fv4nascp2arqci4kdat6:~$ sudo apt install -y postgresql-15
```
### 2.3 Проверим версию PostgreSQL
```bash
yc-user@fv4nascp2arqci4kdat6:~$ psql --version
psql (PostgreSQL) 15.6 (Ubuntu 15.6-1.pgdg20.04+1)
```
### 2.4 Запустим и роверим статус PostgreSQL
```bash
yc-user@fv4nascp2arqci4kdat6:~$ sudo systemctl start postgresql
```
```bash
yc-user@fv4nascp2arqci4kdat6:~$ sudo systemctl status postgresql
```
<details>
<summary>Детали</summary>

```bash
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/lib/systemd/system/postgresql.service; enabled; vendor preset: enabled)
     Active: active (exited) since Thu 2024-05-02 21:48:56 UTC; 2min 24s ago
   Main PID: 3645 (code=exited, status=0/SUCCESS)
      Tasks: 0 (limit: 4631)
     Memory: 0B
     CGroup: /system.slice/postgresql.service

May 02 21:48:56 fv4nascp2arqci4kdat6 systemd[1]: Starting PostgreSQL RDBMS...
May 02 21:48:56 fv4nascp2arqci4kdat6 systemd[1]: Finished PostgreSQL RDBMS.
```
</details>

## 3. Настроим кластер PostgreSQL 15 на максимальную производительность
### 3.1 Создадим базу данных для pgbench
```bash
yc-user@fv4nascp2arqci4kdat6:~$ sudo su postgres
```
```bash
postgres@fv4nascp2arqci4kdat6:/home/yc-user$ pgbench -i -s 10 postgres
```
<details>
<summary>Детали</summary>

```bash
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
1000000 of 1000000 tuples (100%) done (elapsed 11.81 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 17.51 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 12.52 s, vacuum 0.12 s, primary keys 4.86 s).
```
</details>

### 3.1 Запустим утилиту pgbench на стандартной конфигурации
```bash
postgres@fv4nascp2arqci4kdat6:/home/yc-user$ pgbench -c 8 -j 2 -T 60 -U postgres postgres
```
> -c 8 - количество клиентов
>
> -j 2 - количество потоков
>
> -T 60 - время работы в секундах

**Результат составил 953.408722 TPS (без учета времени подключения)**

<details>
<summary>Результат</summary>

```bash
pgbench (15.6 (Ubuntu 15.6-1.pgdg20.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 10
query mode: simple
number of clients: 8
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 57804
number of failed transactions: 0 (0.000%)
latency average = 8.391 ms
initial connection time = 12.045 ms
tps = 953.408722 (without initial connection time)
```
</details>

### 3.2 Воспользуемся калькулятором [pgtune](https://pgtune.leopard.in.ua/#/) для настройки параметров PostgreSQL
- Version: 15
- OS Type: Linux
- DB Type: Web application
- RAM: 4GB
- Number of CPUs: 2
- Connections: 100
- Storage: SSD

<details>
<summary>Рекомендуемые параметры</summary>

```text
# DB Version: 15
# OS Type: linux
# DB Type: web
# Total Memory (RAM): 4 GB
# CPUs num: 1
# Connections num: 100
# Data Storage: ssd

max_connections = 100
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 256MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 5242kB
huge_pages = off
min_wal_size = 1GB
max_wal_size = 4GB
```
</details>

### 3.3 Разместим новые параметры в отдельном файле `001pgtune.conf` в директории `/etc/postgresql/15/main/conf.d/`
```bash
yc-user@fv4nascp2arqci4kdat6:~$ sudo nano /etc/postgresql/15/main/conf.d/001pgtune.conf
```
### 3.4 Перезапустим PostgreSQL
```bash
yc-user@fv4nascp2arqci4kdat6:~$ sudo systemctl restart postgresql
```
### 3.5 Проверим применение параметров в представлении `pg_settings`
```bash
yc-user@fv4nascp2arqci4kdat6:~$ sudo su postgres
```
```bash
postgres@fv4nascp2arqci4kdat6:/home/yc-user$ psql -c "SELECT name, setting FROM pg_settings WHERE source = 'configuration file'"
```

**Судя по листингу, параметры применены успешно:**
<details>
<summary>Детали</summary>

```bash
             name             |                setting
------------------------------+----------------------------------------
 checkpoint_completion_target | 0.9
 cluster_name                 | 15/main
 DateStyle                    | ISO, MDY
 default_statistics_target    | 100
 default_text_search_config   | pg_catalog.english
 dynamic_shared_memory_type   | posix
 effective_cache_size         | 393216
 effective_io_concurrency     | 200
 external_pid_file            | /var/run/postgresql/15-main.pid
 huge_pages                   | off
 lc_messages                  | en_US.UTF-8
 lc_monetary                  | en_US.UTF-8
 lc_numeric                   | en_US.UTF-8
 lc_time                      | en_US.UTF-8
 log_line_prefix              | %m [%p] %q%u@%d
 log_timezone                 | Etc/UTC
 maintenance_work_mem         | 262144
 max_connections              | 100
 max_wal_size                 | 4096
 min_wal_size                 | 1024
 port                         | 5432
 random_page_cost             | 1.1
 shared_buffers               | 131072
 ssl                          | on
 ssl_cert_file                | /etc/ssl/certs/ssl-cert-snakeoil.pem
 ssl_key_file                 | /etc/ssl/private/ssl-cert-snakeoil.key
 TimeZone                     | Etc/UTC
 unix_socket_directories      | /var/run/postgresql
 wal_buffers                  | 2048
 work_mem                     | 5242
(30 rows)
```
</details>

### 3.6 Повторим тестирование pgbench
```bash
postgres@fv4nascp2arqci4kdat6:/home/yc-user$ pgbench -c 8 -j 2 -T 60 -U postgres postgres
```
<details>
<summary>Результат</summary>

```bash
pgbench (15.6 (Ubuntu 15.6-1.pgdg20.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 10
query mode: simple
number of clients: 8
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 92474
number of failed transactions: 0 (0.000%)
latency average = 5.192 ms
initial connection time = 14.698 ms
tps = 1540.829328 (without initial connection time)
```
</details>

**Мы видим, что новый результат составил 1540.829328 TPS, что на 61% выше, чем на стандартной конфигурации.**

### 3.7 Теперь настроим кластер на максимальную производительность, не обращая внимание на возможные проблемы с надежностью в случае аварийной перезагрузки виртуальной машины
> Отключим синхронное реплицирование.
>
> Документация [здесь](https://postgrespro.ru/docs/postgresql/14/runtime-config-wal?ysclid=lvpugycuaw733415204#GUC-SYNCHRONOUS-COMMIT)
```text
synchronous_commit = off
```

> Отключим `fsync`. Это наряду с `synchronous_commit = off` позволит увеличить производительность, но при этом увеличит риск потери данных в случае аварийной перезагрузки. Они могут привести к потере данных в случае сбоев или аварийного отключения, поэтому их следует использовать с осторожностью.
>
> Документация [здесь](https://postgrespro.ru/docs/postgresql/14/runtime-config-wal?ysclid=lvpugycuaw733415204#GUC-FSYNC)
```text
fsync = off
```
> Отключим `full_page_writes`. По умолчанию в PostgreSQL, когда происходит изменение страницы данных, PostgreSQL записывает полную копию этой страницы в журнал WAL. Это делается для обеспечения целостности данных в случае сбоев и восстановления. Однако это также создает дополнительную нагрузку на систему, так как требуется больше операций записи на диск.
>
> Документация [здесь](https://postgrespro.ru/docs/postgresql/14/runtime-config-wal?ysclid=lvpugycuaw733415204#GUC-FULL-PAGE-WRITES)
```text
full_page_writes = off
```
> Увеличим размер `wal_buffers` до 64MB. Это может улучшить производительность операций записи на диск, особенно при интенсивной записи.
>
> Документация [здесь](https://postgrespro.ru/docs/postgresql/14/runtime-config-wal?ysclid=lvpugycuaw733415204#GUC-WAL-BUFFERS)
```text
wal_buffers = 64MB
```
> Увеличим размер `shared_buffers` до 2GB.
>
> Документация [здесь](https://postgrespro.ru/docs/postgresql/14/runtime-config-resource?ysclid=lvpugycuaw733415204#GUC-SHARED-BUFFERS)
```text
shared_buffers = 2GB
```

### 3.8 Создадим файл `002pgtune.conf` в директории `/etc/postgresql/15/main/conf.d/`.
> Параметры в новом файле будут переопределять параметры из `001pgtune.conf`.
>
> Документация [здесь](https://www.postgresql.org/docs/current/config-setting.html#CONFIG-INCLUDES)
```bash
yc-user@fv4nascp2arqci4kdat6:~$ sudo nano /etc/postgresql/15/main/conf.d/002pgtune.conf
```
<details>
<summary>Параметры</summary>

```text
# WARNING: The following settings may compromise data safety!

synchronous_commit = off
fsync = off
full_page_writes = off
wal_buffers = 64MB
shared_buffers = 2GB
```
</details>

### 3.7.3 Перезапустим PostgreSQL
```bash
yc-user@fv4nascp2arqci4kdat6:~$ sudo systemctl restart postgresql
```
### 3.7.4 Проверим применение параметров в представлении `pg_settings`
```bash
yc-user@fv4nascp2arqci4kdat6:~$ sudo su postgres
```
```bash
postgres@fv4nascp2arqci4kdat6:/home/yc-user$ psql -c "SELECT name, setting FROM pg_settings WHERE source = 'configuration file'"
```
<details>
<summary>Детали</summary>

```bash
             name             |                setting
------------------------------+----------------------------------------
 checkpoint_completion_target | 0.9
 cluster_name                 | 15/main
 DateStyle                    | ISO, MDY
 default_statistics_target    | 100
 default_text_search_config   | pg_catalog.english
 dynamic_shared_memory_type   | posix
 effective_cache_size         | 393216
 effective_io_concurrency     | 200
 external_pid_file            | /var/run/postgresql/15-main.pid
 fsync                        | off
 full_page_writes             | off
 huge_pages                   | off
 lc_messages                  | en_US.UTF-8
 lc_monetary                  | en_US.UTF-8
 lc_numeric                   | en_US.UTF-8
 lc_time                      | en_US.UTF-8
 log_line_prefix              | %m [%p] %q%u@%d
 log_timezone                 | Etc/UTC
 maintenance_work_mem         | 262144
 max_connections              | 100
 max_wal_size                 | 4096
 min_wal_size                 | 1024
 port                         | 5432
 random_page_cost             | 1.1
 shared_buffers               | 262144
 ssl                          | on
 ssl_cert_file                | /etc/ssl/certs/ssl-cert-snakeoil.pem
 ssl_key_file                 | /etc/ssl/private/ssl-cert-snakeoil.key
 synchronous_commit           | off
 TimeZone                     | Etc/UTC
 unix_socket_directories      | /var/run/postgresql
 wal_buffers                  | 8192
 work_mem                     | 5242
(33 rows)
```
</details>

**Видим, что новые параметры применены наряду с параметрами из `001pgtune.conf`.**

### 3.7.4 Повторим тестирование pgbench
```bash
postgres@fv4nascp2arqci4kdat6:/home/yc-user$ pgbench -c 8 -j 2 -T 60 -U postgres postgres
```
<details>
<summary>Результат</summary>

```bash
pgbench (15.6 (Ubuntu 15.6-1.pgdg20.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 10
query mode: simple
number of clients: 8
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 198172
number of failed transactions: 0 (0.000%)
latency average = 2.423 ms
initial connection time = 14.145 ms
tps = 3301.719924 (without initial connection time)
```
</details>

**Мы видим, что новый результат составил 3301.719924 TPS.**
## Вывод
 На стандартной конфигурации результат был 953.408722 TPS. При отключении синхронного реплицирования, `fsync` и `full_page_writes`, увеличении размера `wal_buffers` и `shared_buffers` результат составил 3301.719924 TPS. Тем самым, мы увеличили производительность в 3.5 раза (на 246%). Однако, стоит помнить, что такие настройки могут привести к потере данных в случае аварийной перезагрузки виртуальной машины.
