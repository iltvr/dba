# Домашнее задание

## Бэкапы

**Цель:**

- Применить логический бэкап. Восстановиться из бэкапа.

**Описание/Пошаговая инструкция выполнения домашнего задания:**

1. Создаем ВМ/докер с PostgreSQL.
2. Создаем БД, схему и в ней таблицу.
3. Заполним таблицы автосгенерированными 100 записями.
4. Под линукс пользователем Postgres создадим каталог для бэкапов.
5. Сделаем логический бэкап, используя утилиту COPY.
6. Восстановим во вторую таблицу данные из бэкапа.
7. Используя утилиту pg_dump, создадим бэкап в кастомном сжатом формате двух таблиц.
8. Используя утилиту pg_restore, восстановим в новую БД только вторую таблицу.

**ДЗ сдается в виде миниотчета на гитхабе с описанием шагов и с какими проблемами столкнулись.**

## Критерии оценки:

Выполнение ДЗ: 10 баллов

- 2 балла за красивое решение
- 2 балла за рабочее решение, и недостатки, указанные преподавателем, не устранены

# Домашняя работа к занятию "Резервное копирование и восстановление" от 07.03.2024

## 1. Подготовка
### 1.1 Используем ВМ из предыдущего ДЗ.
> См. шаги 1, 2 [ДЗ к занятию "Настройка PostgreSQL"](../topic-8/README.md)
### 1.2 Удалим кластер PostgreSQL и создадим новый.
```bash
yc-user@fv4nascp2arqci4kdat6:~$ sudo systemctl stop postgresql@15-main
```
```bash
yc-user@fv4nascp2arqci4kdat6:~$ sudo pg_dropcluster 15 main
```
```bash
yc-user@fv4nascp2arqci4kdat6:~$ sudo pg_createcluster 15 main
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
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```
</details>

### 1.3 Очистим кастомные конфигурационные файлы.
```bash
yc-user@fv4nascp2arqci4kdat6:~$ sudo rm -rf /etc/postgresql/15/main/conf.d/*
```
### 1.4 Запустим кластер PostgreSQL.
```bash
yc-user@fv4nascp2arqci4kdat6:~$ sudo systemctl start postgresql@15-main
```
## 2. Создание БД, тестовую схему и таблицы
### 2.1 Создадим БД `test_db` и таблицу `test_table` в схеме `test_schema`.
```bash
yc-user@fv4nascp2arqci4kdat6:~$ sudo su postgres
```
```bash
postgres@fv4nascp2arqci4kdat6:/home/yc-user$ psql
```
```psql
CREATE DATABASE test_db;
CREATE DATABASE
```
```psql
postgres=# \c test_db
You are now connected to database "test_db" as user "postgres".
```
```psql
test_db=# CREATE SCHEMA test_schema;
CREATE SCHEMA
```
```psql
test_db=# CREATE TABLE test_schema.test_table (id SERIAL PRIMARY KEY, name VARCHAR(100));
CREATE TABLE
```
## 3. Заполним таблицу.
### 3.1 Заполним таблицу `test_table` 100 записями.
```psql
test_db=# INSERT INTO test_schema.test_table (name) SELECT md5(random()::text) FROM generate_series(1, 100);
INSERT 0 100
```
### 3.2 Проверим записи в таблице.
```psql
test_db=# SELECT * FROM test_schema.test_table;
 id  |               name
-----+----------------------------------
   1 | eebbbe5a4221d0b9daed838c9f91e3a0
   2 | 4458d25a7ec41b483c27b19fea85adb4
   3 | 1b1bcaca461a503e1ec222fe8b0fba0a
   4 | d648d6b24344503591deed1e8b5e8ad2
   5 | fdd475535ea6d1785b68651dcbc20884
   ...
  95 | efb49da6220c00323cb7bc990513c7d3
  96 | 440e1d828a53d8ab5ae43c870519b234
  97 | 6e20e161e58efabc28541109b0c58fdf
  98 | 0718e3b23af72805a8aeb47597ca2020
  99 | 36e7bc0d305c47d34413fed9b25df517
 100 | bc76b43d3656c4e54b36a5c7d066f7b4
(100 rows)
```
## 4. Создание каталога для бэкапов
### 4.1 Создадим сетевой диск в YC
```bash
➜  ~ yc compute disk create --name otus-db-pg-1-hdd-1 --description "backup" --size 5 --type network-hdd
```
<details>
<summary>Детали</summary>

```bash
done (8s)
id: fv46h0kre7fva19d7lur
folder_id: b1gffa7irklogm13fdjq
created_at: "2024-05-06T20:16:26Z"
name: otus-db-pg-1-hdd-1
description: backup
type_id: network-hdd
zone_id: ru-central1-d
size: "5368709120"
block_size: "4096"
status: READY
disk_placement_policy: {}
```
</details>

### 4.2 Подключим сетевой диск к ВМ.
```bash
~ yc compute instance attach-disk otus-db-pg-1 --disk-name otus-db-pg-1-hdd-1
```
<details>
<summary>Детали</summary>

```bash
done (11s)
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
secondary_disks:
  - mode: READ_WRITE
    device_name: fv46h0kre7fva19d7lur
    disk_id: fv46h0kre7fva19d7lur
network_interfaces:
  - index: "0"
    mac_address: d0:0d:17:57:19:91
    subnet_id: fl8t7lqif5qa2aihdfvm
    primary_v4_address:
      address: 192.168.0.32
      one_to_one_nat:
        address: 158.160.156.246
        ip_version: IPV4
gpu_settings: {}
fqdn: fv4nascp2arqci4kdat6.auto.internal
scheduling_policy: {}
network_settings:
  type: STANDARD
placement_policy: {}
```
</details>

### 4.3 Проверим виден ли диск в системе.
```bash
yc-user@fv4nascp2arqci4kdat6:~$ sudo fdisk -l
```
<details>
<summary>Детали</summary>

```bash
Disk /dev/vda: 20 GiB, 21474836480 bytes, 41943040 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: gpt
Disk identifier: 25637092-3FEB-41CF-A2BB-20E068C29F21

Device     Start      End  Sectors Size Type
/dev/vda1   2048     4095     2048   1M BIOS boot
/dev/vda2   4096 41943006 41938911  20G Linux filesystem


Disk /dev/vdb: 5 GiB, 5368709120 bytes, 10485760 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
```
</details>

### 4.4 Создадим раздел и форматируем диск.
```bash
yc-user@fv4nascp2arqci4kdat6:~$ sudo fdisk /dev/vdb
```
```bash
yc-user@fv4nascp2arqci4kdat6:~$ sudo mkfs.ext4 /dev/vdb
```
### 4.5 Cмонтируем диск в каталог `/mnt/backup`.
```bash
yc-user@fv4nascp2arqci4kdat6:~$ sudo mkdir /mnt/backup
```
```bash
yc-user@fv4nascp2arqci4kdat6:~$ sudo mount /dev/vdb /mnt/backup
```
### 4.6 Добавим запись в `/etc/fstab` для автоматического монтирования диска при перезагрузке.
```bash
yc-user@fv4nascp2arqci4kdat6:~$ echo "UUID=8ba0c1b7-5ec6-4d08-8034-e5dba3567d7a /mnt/backup ext4 defaults 0 0" | sudo tee -a /etc/fstab
```
### 4.7 Проверим монтирование диска.
```bash
➜  ~ yc compute instance stop otus-db-pg-1
done (15s)
```
```bash
➜  ~ yc compute instance start otus-db-pg-1
done (16s)
...
```
```bash
yc-user@fv4nascp2arqci4kdat6:~$ df -h
```
<details>
<summary>Детали</summary>

```bash
Filesystem      Size  Used Avail Use% Mounted on
udev            1.9G     0  1.9G   0% /dev
tmpfs           392M  804K  392M   1% /run
/dev/vda2        20G  2.9G   16G  16% /
tmpfs           2.0G  1.1M  2.0G   1% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/vdb        4.9G   24K  4.6G   1% /mnt/backup
tmpfs           392M     0  392M   0% /run/user/1000
```
</details>

**Готово. Диск монтируется автоматически при перезагрузке.**

### 4.8 Создадим каталог для бэкапов.
```bash
yc-user@fv4nascp2arqci4kdat6:~$ sudo mkdir /mnt/backup/pg_backup
```
### 4.9 Дадим права на запись каталогу. Проверим права.
```bash
yc-user@fv4nascp2arqci4kdat6:~$ sudo chown postgres:postgres /mnt/backup/pg_backup
```
```bash
yc-user@fv4nascp2arqci4kdat6:~$ ls -ld /mnt/backup/pg_backup
drwxr-xr-x 2 postgres postgres 4096 May  6 20:37 /mnt/backup/pg_backup
```
## 5. Логический бэкап
### 5.1 Создадим логический бэкап с помощью утилиты `COPY`.
> `COPY` перемещает данные между таблицами Postgres Pro и обычными файлами в файловой системе. `COPY TO` копирует содержимое таблицы в файл, а `COPY FROM` — из файла в таблицу (добавляет данные к тем, что уже содержались в таблице). `COPY TO` может также скопировать результаты запроса `SELECT`.
>
> Документация: https://postgrespro.ru/docs/postgrespro/16/sql-copy

> Важно:
> Команду `COPY TO` можно использовать только с простыми таблицами, не представлениями, и при этом она не копирует строки из дочерних таблиц или секций. То есть, `COPY таблица TO` копирует те же строки, что выдаёт запрос `SELECT * FROM ONLY таблица`. Для выгрузки всех строк представления или таблицы с учётом иерархии наследования или секционирования можно применить `COPY (SELECT * FROM таблица) TO ...`.

### 5.2 Создадим бэкап `test_table` в файл `/mnt/backup/pg_backup/test_table.sql` c помощью утилиты `COPY`.
```bash
yc-user@fv4nascp2arqci4kdat6:~$ sudo su postgres
```
```bash
postgres@fv4nascp2arqci4kdat6:/home/yc-user$ psql -d test_db -c "COPY test_schema.test_table TO '/mnt/backup/pg_backup/test_table.sql';"
COPY 100
```
### 5.3 Проверим наличие файла бэкапа.
```bash
postgres@fv4nascp2arqci4kdat6:/home/yc-user$ ls -l /mnt/backup/pg_backup/test_table.sql
-rw-r--r-- 1 postgres postgres 3592 May  6 20:59 /mnt/backup/pg_backup/test_table.sql
```
## 6. Восстановление из бэкапа
### 6.1 Создадим таблицу `test_table2` в схеме `test_schema`.
```bash
postgres@fv4nascp2arqci4kdat6:/home/yc-user$ psql -d test_db
```
```psql
test_db=# CREATE TABLE test_schema.test_table2 (id SERIAL PRIMARY KEY, name VARCHAR(100));
CREATE TABLE
```
### 6.2 Восстановим данные из бэкапа в таблицу `test_table2`.
```psql
COPY test_schema.test_table2 FROM '/mnt/backup/pg_backup/test_table.sql';
COPY 100
```
### 6.3 Проверим записи в таблице `test_table2`.
```psql
test_db=# SELECT * FROM test_schema.test_table2;
  id  |               name
-----+----------------------------------
   1 | eebbbe5a4221d0b9daed838c9f91e3a0
   2 | 4458d25a7ec41b483c27b19fea85adb4
   3 | 1b1bcaca461a503e1ec222fe8b0fba0a
   4 | d648d6b24344503591deed1e8b5e8ad2
   5 | fdd475535ea6d1785b68651dcbc20884
   ...
  95 | efb49da6220c00323cb7bc990513c7d3
  96 | 440e1d828a53d8ab5ae43c870519b234
  97 | 6e20e161e58efabc28541109b0c58fdf
  98 | 0718e3b23af72805a8aeb47597ca2020
  99 | 36e7bc0d305c47d34413fed9b25df517
 100 | bc76b43d3656c4e54b36a5c7d066f7b4
(100 rows)
```
**Строки в таблице `test_table2` восстановлены из бэкапа.**

## 7. Создание кастомного бэкапа
### Подготовка. Создадим таблицу `test_table3` в схеме `test_schema` с отличающейся структурой.
```bash
postgres@fv4nascp2arqci4kdat6:/home/yc-user$ psql -d test_db
```
```psql
test_db=# CREATE TABLE test_schema.test_table3 (id SERIAL PRIMARY KEY, data VARCHAR(200));
CREATE TABLE
```
```psql
test_db=# INSERT INTO test_schema.test_table3 (data) SELECT md5(random()::text) FROM generate_series(1, 100);
INSERT 0 100
```
```psql
test_db=# SELECT * FROM test_schema.test_table3;
 id  |               data
-----+----------------------------------
   1 | 7db49a3eb86095de9f032def571280d2
   2 | 1e9a22f88c327a341642f02c70ee3e57
   3 | d854599f25b4ad73751b83d2b3656eea
   4 | 08ee94f35c881b65cc4fa19a11550f3a
   5 | 3244782b795099d9662bc2a25e6e59a
   ...
  95 | 9ce87053e897d308764e6f74fa6bfddf
  96 | 78fcd299dcde5b6ed45f6b88b9077c90
  97 | d62c7d2f9d16981bfadcde7f7d48e975
  98 | 6e4ccfcac8ac63b67ec02fa50862aa52
  99 | 02606feddf1a628a7ed74fab71f2fbbb
 100 | eb3b03e7c49574eff9ffb615ffcf278f
(100 rows)
```

### 7.1 Создадим кастомный бэкап с помощью утилиты `pg_dump`.
> `pg_dump` — утилита для резервного копирования данных PostgreSQL. Она создаёт снимок базы данных в виде скрипта SQL, который может быть использован для восстановления базы данных в исходном или другом месте.
>
> Документация: https://postgrespro.ru/docs/postgrespro/16/app-pgdump

### 7.2 Создадим кастомный бэкап таблиц в кастомном формате `test_table` и `test_table3` в файл `/mnt/backup/pg_backup/test_tables_fd.sql`.
```bash
postgres@fv4nascp2arqci4kdat6:/home/yc-user$ pg_dump -d test_db -t test_schema.test_table -t test_schema.test_table3 -Fc -f /mnt/backup/pg_backup/test_tables_fc.dump
```
### 7.3 Проверим наличие файла кастомного бэкапа.
```bash
postgres@fv4nascp2arqci4kdat6:/home/yc-user$ ls -l /mnt/backup/pg_backup/test_tables_fc.dump
-rw-rw-r-- 1 postgres postgres 9033 May 13 19:48 /mnt/backup/pg_backup/test_tables_fc.dump
```
**Кастомный бэкап создан.**

## 8. Восстановление из кастомного бэкапа
### 8.1 Создадим новую БД `test_db2`.
```bash
postgres@fv4nascp2arqci4kdat6:/home/yc-user$ psql -c "CREATE DATABASE test_db2;"
CREATE DATABASE
```
### 8.3 Создадим схему `test_schema` в БД `test_db2`.
```bash
postgres@fv4nascp2arqci4kdat6:/home/yc-user$ psql -d test_db2 -c "CREATE SCHEMA test_schema;"
CREATE SCHEMA
```
### 8.3 Восстановим данные из кастомного бэкапа в новую БД `test_db2`. Bосстановим только `test_table3`.
> `pg_restore` — утилита предназначена для восстановления базы данных Postgres Pro из архива, созданного командой pg_dump в любом из не текстовых форматов. Она выполняет команды, необходимые для восстановления того состояния базы данных, в котором база была сохранена. При наличии файлов архивов, pg_restore может восстанавливать данные избирательно или даже переупорядочить объекты перед восстановлением. Заметьте, что разработанный для файлов архива формат не привязан к архитектуре.
>
> Документация: https://postgrespro.ru/docs/postgrespro/16/app-pgrestore

### 8.4 Проверим таблицы в БД `test_db2`.
```bash
postgres@fv4nascp2arqci4kdat6:/home/yc-user$ psql -d test_db2
```
```psql
test_db2=# \dt test_schema.*
Did not find any relation named "test_schema.*".
```
### 8.5 Восстановим таблицу `test_table3` из кастомного бэкапа в БД `test_db2`.
```bash
postgres@fv4nascp2arqci4kdat6:/home/yc-user$ pg_restore -d test_db2 -j 2 -t test_schema.test_table3 -Fc /mnt/backup/pg_backup/test_tables_fc.dump
```
### 8.6 Проверим таблицы в БД `test_db2`.
```bash
postgres@fv4nascp2arqci4kdat6:/home/yc-user$ psql -d test_db2
```
```psql
test_db2=# \dt test_schema.*
              List of relations
   Schema    |    Name     | Type  |  Owner
-------------+-------------+-------+----------
 test_schema | test_table3 | table | postgres
(1 row)
```
### 8.5 Проверим записи в таблице `test_table3` в БД `test_db2`.
```bash
postgres@fv4nascp2arqci4kdat6:/home/yc-user$ psql -d test_db2
```
```psql
test_db2=# SELECT * FROM test_schema.test_table3;
 id  |               data
-----+----------------------------------
   1 | 7db49a3eb86095de9f032def571280d2
   2 | 1e9a22f88c327a341642f02c70ee3e57
   3 | d854599f25b4ad73751b83d2b3656eea
   4 | 08ee94f35c881b65cc4fa19a11550f3a
  ...
  96 | 78fcd299dcde5b6ed45f6b88b9077c90
  97 | d62c7d2f9d16981bfadcde7f7d48e975
  98 | 6e4ccfcac8ac63b67ec02fa50862aa52
  99 | 02606feddf1a628a7ed74fab71f2fbbb
 100 | eb3b03e7c49574eff9ffb615ffcf278f
```
**Данные в таблице `test_table3` восстановлены из бэкапа.**

# Вывод
## В процессе выполнения домашнего задания мы освоили:
- Создание логического бэкапа с помощью утилиты `COPY`.
- Восстановление данных из логического бэкапа.
- Создание кастомного бэкапа с помощью утилиты `pg_dump`.
- Восстановление данных из кастомного бэкапа с помощью утилиты `pg_restore`.









