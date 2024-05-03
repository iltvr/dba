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

***Результат составил 953.408722 TPS (без учета времени подключения)***

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
yc-user@fv4nascp2arqci4kdat6:~$ sudo systemctl restart postgresql@15-main
```
### 3.5 Проверим применение параметров в представлении `pg_settings`
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

***Судя по листингу, параметры применены успешно.***

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

***Мы видим, что новый результат составил 1540.829328 TPS, что на 61% выше, чем на стандартной конфигурации.***

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
yc-user@fv4nascp2arqci4kdat6:~$ sudo systemctl restart postgresql@15-main
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

***Видим, что новые параметры применены наряду с параметрами из `001pgtune.conf`.***

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

***Мы видим, что новый результат составил 3301.719924 TPS.***
## Вывод
 На стандартной конфигурации результат был 953.408722 TPS. При отключении синхронного реплицирования, `fsync` и `full_page_writes`, увеличении размера `wal_buffers` и `shared_buffers` результат составил 3301.719924 TPS. Тем самым, мы увеличили производительность в 3.5 раза (на 246%). Однако, стоит помнить, что такие настройки могут привести к потере данных в случае аварийной перезагрузки виртуальной машины.

## Задание со *
> Материалы взяты из [статьи](https://www.percona.com/blog/tuning-postgresql-for-sysbench-tpcc) на сайте Percona.
> А также из статьи [Тест производительности PostgreSQL на AWS EC2-инстансах на ARM](https://habr.com/ru/companies/flant/articles/547526/).
### 0. Установим `git`
```bash
yc-user@fv4nascp2arqci4kdat6:~$ sudo apt -y install git
```
### 1. Установим `sysbench` и `sysbench-tpcc`
```bash
yc-user@fv4nascp2arqci4kdat6:~$ curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.deb.sh | sudo bash
sudo apt -y install sysbench
```
### 2. Клонируем репозиторий `sysbench-tpcc`
```bash
yc-user@fv4nascp2arqci4kdat6:~$ git clone https://github.com/Percona-Lab/sysbench-tpcc.git
```
<details>
<summary>Детали</summary>

```bash
Cloning into 'sysbench-tpcc'...
remote: Enumerating objects: 226, done.
remote: Counting objects: 100% (58/58), done.
remote: Compressing objects: 100% (26/26), done.
remote: Total 226 (delta 32), reused 52 (delta 32), pack-reused 168
Receiving objects: 100% (226/226), 77.81 KiB | 866.00 KiB/s, done.
Resolving deltas: 100% (120/120), done.
```
</details>

### 3. Перейдем в директорию `sysbench-tpcc`
```bash
yc-user@fv4nascp2arqci4kdat6:~$ cd sysbench-tpcc
```

### 4. Создадим базу данных для `sysbench-tpcc`
```bash
yc-user@fv4nascp2arqci4kdat6:~$ sudo su postgres
```
```bash
postgres@fv4nascp2arqci4kdat6:/home/yc-user$ createdb tpcc
```
### 5. Инициализируем базу данных для `sysbench-tpcc`
```bash
postgres@fv4nascp2arqci4kdat6:/home/yc-user$ sysbench tpcc.lua --pgsql-host=localhost --pgsql-port=5432 --pgsql-user=postgres --pgsql-password=postgres --pgsql-db=tpcc --threads=2 --db-driver=pgsql --tables=10 --scale=10 --trx_level=RC prepare
```
### 5.1 Не можем подключиться к базе так-как не настроен `pg_hba.conf`. Добавим вход по паролю для пользователя `postgres`
```bash
postgres@fv4nascp2arqci4kdat6:/home/yc-user$ vi /etc/postgresql/15/main/pg_hba.conf
```
```text
local    all     postgres     md5
```
```bash
postgres@fv4nascp2arqci4kdat6:/home/yc-user/sysbench-tpcc$ psql
psql (15.6 (Ubuntu 15.6-1.pgdg20.04+1))
Type "help" for help.

postgres=# \password
Enter new password for user "postgres":
Enter it again:
postgres=# \q
```
### 5.2 Повторим инициализацию базы данных
```bash
postgres@fv4nascp2arqci4kdat6:/home/yc-user/sysbench-tpcc$ sysbench tpcc.lua \
  --pgsql-host=localhost \
  --pgsql-port=5432 \
  --pgsql-user=postgres \
  --pgsql-password=postgres \
  --pgsql-db=tpcc \
  --threads=2 \
  --db-driver=pgsql \
  --tables=10 \
  --scale=100 \
  --trx_level=RC \
  prepare
```
***Скрипт попытается создать таблиы и заполнить их данными. В процессе инициализации получили ошибку места на диске.***

> –threads=2 – количество потоков
>
> –tables=10 – количество таблиц
>
> –scale=100 – масштаб базы данных, размер базы данных в гигабайтах будет равен 100 * 1.1 ГБ = 110 ГБ

### 5.3 Удалим кластер
```bash
yc-user@fv4nascp2arqci4kdat6:~/sysbench-tpcc$ sudo systemctl stop postgresql@15-main
```
```bash
yc-user@fv4nascp2arqci4kdat6:~/sysbench-tpcc$ sudo -u postgres pg_dropcluster 15 main
Warning: systemd was not informed about the removed cluster yet. Operations like "service postgresql start" might fail. To fix, run:
  sudo systemctl daemon-reload
```
### 5.4 Увеличим размер диска до 20GB
```bash
➜  ~ yc compute disk list
+----------------------+------+-------------+---------------+--------+----------------------+-----------------+-------------+
|          ID          | NAME |    SIZE     |     ZONE      | STATUS |     INSTANCE IDS     | PLACEMENT GROUP | DESCRIPTION |
+----------------------+------+-------------+---------------+--------+----------------------+-----------------+-------------+
| fv4poj465cj699mght3b |      | 10737418240 | ru-central1-d | READY  | fv4nascp2arqci4kdat6 |                 |             |
+----------------------+------+-------------+---------------+--------+----------------------+-----------------+-------------+
```
```bash
➜  ~ yc compute disk resize fv4poj465cj699mght3b --size 20GB
```
<details>
<summary>Детали</summary>

```bash
done (27s)
id: fv4poj465cj699mght3b
folder_id: b1gffa7irklogm13fdjq
created_at: "2024-05-02T21:27:14Z"
type_id: network-ssd
zone_id: ru-central1-d
size: "21474836480"
block_size: "4096"
product_ids:
  - f2ejga170v8ct0nua6o6
status: READY
source_image_id: fd8cnj92ad0th7m7krqh
instance_ids:
  - fv4nascp2arqci4kdat6
disk_placement_policy: {}
```
</details>

***Теперь у нас есть 20GB диска.***

### 5.5 Создадим новый кластер
```bash
yc-user@fv4nascp2arqci4kdat6:~/sysbench-tpcc$ sudo -u postgres pg_createcluster 15 main
```
<details>
<summary>Детали</summary>

```bash
Creating new PostgreSQL cluster 15/main ...
/usr/lib/postgresql/15/bin/initdb -D /var/lib/postgresql/15/main --auth-local peer --auth-host scram-sha-256 --no-instructions
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "C.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are disabled.

fixing permissions on existing directory /var/lib/postgresql/15/main ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Etc/UTC
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Warning: systemd does not know about the new cluster yet. Operations like "service postgresql start" will not handle it. To fix, run:
  sudo systemctl daemon-reload
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```
</details>

### 5.6 Сконфигурируем `pg_hba.conf`
```bash
yc-user@fv4nascp2arqci4kdat6:~/sysbench-tpcc$ sudo vi /etc/postgresql/15/main/pg_hba.conf
```
```text
local    all     postgres    md5
```

### 5.7 Запустим кластер
```bash
yc-user@fv4nascp2arqci4kdat6:~/sysbench-tpcc$ sudo systemctl start postgresql@15-main
```
### 5.8 Создадим базу данных
```bash
yc-user@fv4nascp2arqci4kdat6:~/sysbench-tpcc$ sudo -u postgres createdb tpcc
```
### 5.9 Повторим инициализацию базы данных, но уменьшим `--scale` до 10 и уменьшим `--tables` до 1
```bash
yc-user@fv4nascp2arqci4kdat6:~/sysbench-tpcc$ sysbench tpcc.lua \
>   --pgsql-host=localhost \
>   --pgsql-port=5432 \
>   --pgsql-user=postgres \
>   --pgsql-password=postgres \
>   --pgsql-db=tpcc \
>   --threads=2 \
>   --db-driver=pgsql \
>   --tables=1 \
>   --scale=10 \
>   --trx_level=RC \
>   prepare
```
<details>
<summary>Детали</summary>

```bash
sysbench 1.0.20 (using bundled LuaJIT 2.1.0-beta2)

Initializing worker threads...

DB SCHEMA public
Creating tables: 1

DB SCHEMA public
Waiting on tables 30 sec

Adding indexes 1 ...

Adding FK 1 ...

Waiting on tables 30 sec

loading tables: 1 for warehouse: 2

loading tables: 1 for warehouse: 1

loading tables: 1 for warehouse: 4

loading tables: 1 for warehouse: 3

loading tables: 1 for warehouse: 6

loading tables: 1 for warehouse: 5

loading tables: 1 for warehouse: 8

loading tables: 1 for warehouse: 7

loading tables: 1 for warehouse: 10

loading tables: 1 for warehouse: 9
```
</details>

### 5.10 Будем следить за состоянием дискового пространства
```bash
yc-user@fv4nascp2arqci4kdat6:~/sysbench-tpcc$ watch -n 1 df -h
```

<details>
<summary>Детали</summary>

```bash
Every 1.0s: df -h                                                                                                                                                  fv4nascp2arqci4kdat6: Fri May  3 10:03:52 2024

Filesystem      Size  Used Avail Use% Mounted on
udev            1.9G     0  1.9G   0% /dev
tmpfs           392M  796K  392M   1% /run
/dev/vda2       9.8G  7.2G  2.2G  77% /
tmpfs           2.0G  1.1M  2.0G   1% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           2.0G     0  2.0G   0% /sys/fs/cgroup
tmpfs           392M     0  392M   0% /run/user/1000
```
</details>

***Видим, что после подготовки к тесту данные на диске занимают 7.2GB.***

### 5.11 Запустим тестирование на стандартной конфигурации PostgreSQL
```bash
yc-user@fv4nascp2arqci4kdat6:~/sysbench-tpcc$ sysbench tpcc.lua \
  --pgsql-host=localhost \
  --pgsql-port=5432 \
  --pgsql-user=postgres \
  --pgsql-password=postgres \
  --pgsql-db=tpcc \
  --threads=2 \
  --db-driver=pgsql \
  --tables=1 \
  --scale=10 \
  --trx_level=RC \
  --time=60 \
  run
```
<details>
<summary>Результат</summary>

```bash
sysbench 1.0.20 (using bundled LuaJIT 2.1.0-beta2)

Running the test with following options:
Number of threads: 2
Initializing random number generator from current time


Initializing worker threads...

DB SCHEMA public
DB SCHEMA public
Threads started!

SQL statistics:
    queries performed:
        read:                            235079
        write:                           244534
        other:                           35864
        total:                           515477
    transactions:                        17930  (298.80 per sec.)
    queries:                             515477 (8590.26 per sec.)
    ignored errors:                      80     (1.33 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          60.0051s
    total number of events:              17930

Latency (ms):
         min:                                    0.59
         avg:                                    6.69
         max:                                 1963.73
         95th percentile:                       15.55
         sum:                               119971.23

Threads fairness:
    events (avg/stddev):           8965.0000/62.00
    execution time (avg/stddev):   59.9856/0.00
```
</details>

***Результат составил 298.80 TPS.***

### 5.12 Проверим применение параметров в представлении `pg_settings`
```bash
yc-user@fv4nascp2arqci4kdat6:~/sysbench-tpcc$ sudo su postgres
```
```bash
postgres@fv4nascp2arqci4kdat6:/home/yc-user/sysbench-tpcc$ psql -U postgres -W -c "SELECT name, setting FROM pg_settings WHERE source = 'configuration file'"
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
 lc_messages                  | C.UTF-8
 lc_monetary                  | C.UTF-8
 lc_numeric                   | C.UTF-8
 lc_time                      | C.UTF-8
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

***Видим, что параметры остались такими-же как и при тестировании утилитой `pgbench`.***

### 5.13 Повторим тестирование c теми-же параметрами
```bash
postgres@fv4nascp2arqci4kdat6:/home/yc-user/sysbench-tpcc$ sysbench tpcc.lua \
  --pgsql-host=localhost \
  --pgsql-port=5432 \
  --pgsql-user=postgres \
  --pgsql-password=postgres \
  --pgsql-db=tpcc \
  --threads=2 \
  --db-driver=pgsql \
  --tables=1 \
  --scale=10 \
  --trx_level=RC \
  --time=60 \
  run
```

<details>
<summary>Результат</summary>

```bash
sysbench 1.0.20 (using bundled LuaJIT 2.1.0-beta2)

Running the test with following options:
Number of threads: 2
Initializing random number generator from current time


Initializing worker threads...

DB SCHEMA public
DB SCHEMA public
Threads started!

SQL statistics:
    queries performed:
        read:                            261689
        write:                           271732
        other:                           40290
        total:                           573711
    transactions:                        20143  (335.65 per sec.)
    queries:                             573711 (9559.83 per sec.)
    ignored errors:                      95     (1.58 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          60.0107s
    total number of events:              20143

Latency (ms):
         min:                                    0.61
         avg:                                    5.96
         max:                                  512.96
         95th percentile:                       15.27
         sum:                               119973.44

Threads fairness:
    events (avg/stddev):           10071.5000/61.50
    execution time (avg/stddev):   59.9867/0.00
```
</details>

***Результат составил 335.65 TPS. Таким образом, мы увеличили производительность на 12% (на 36.5 TPS). По сравнению с результатами тестирования утилитой `pgbench` мы увеличили производительность на 12%. Это может быть связано с тем, что при использовании `sysbench` характер нагрузки на кластер отличается от `pgbench`.***

## Вывод
В процессе выполнения ДЗ мы научились
  - настраивать параметры PostgreSQL для увеличения производительности
  - использовать утилиту `pgtune` для настройки параметров PostgreSQL
  - использовать утилиту `pgbench` для тестирования производительности PostgreSQL
  - использовать утилиту `sysbench` для тестирования производительности PostgreSQL
