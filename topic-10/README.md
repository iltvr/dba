# Домашнее задание

## Репликация

### Цель:
реализовать свой миникластер на 3 ВМ.

### Описание/Пошаговая инструкция выполнения домашнего задания:

1. **На 1 ВМ создаем таблицы:**
    - `test` для записи
    - `test2` для запросов на чтение.
2. **Создаем публикацию таблицы `test` и подписываемся на публикацию таблицы `test2` с ВМ №2.**
3. **На 2 ВМ создаем таблицы:**
    - `test2` для записи
    - `test` для запросов на чтение.
4. **Создаем публикацию таблицы `test2` и подписываемся на публикацию таблицы `test` с ВМ №1.**
5. **3 ВМ использовать как реплику для чтения и бэкапов:**
    - подписаться на таблицы из ВМ №1 и №2.

Домашнее задание сдается в виде миниотчета на GitHub с описанием шагов и с указанием проблем, с которыми столкнулись.

* **Дополнительное задание:**
  - реализовать горячее реплицирование для высокой доступности на 4ВМ. Источником должна выступать ВМ №3. Написать о проблемах, с которыми столкнулись.

### Критерии оценки:

**Выполнение ДЗ:** 10 баллов

- 5 баллов за задание со *
- Плюс 2 балла за красивое решение
- Минус 2 балла за рабочее решение, если недостатки, указанные преподавателем, не устранены

# Домашняя работа к занятию "Виды и устройство репликации в PostgreSQL. Практика применения." от 14.03.2024
# 1. Подготовка окружения
## 1.1 Создадим в Docker 3 виртуальные машины на базе образа ubuntu 20.04
```bash
➜  ~ docker run -it -p 5551:5432 --network pg-cluster-network --rm --name vm1 --hostname vm1 ubuntu:20.04 bash
➜  ~ docker run -it -p 5552:5432 --network pg-cluster-network --rm --name vm2 --hostname vm2 ubuntu:20.04 bash
➜  ~ docker run -it -p 5553:5432 --network pg-cluster-network --rm --name vm3 --hostname vm3 ubuntu:20.04 bash
```
## 1.2 Установим PostgreSQL 16 на каждую из виртуальных машин
```bash
root@vm1:/# apt-get update
root@vm1:/# apt-get install -y lsb-release && apt-get clean all
root@vm1:/# apt-get install -y iputils-ping sudo vim netcat wget gnupg2
root@vm1:/# wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add -
root@vm1:/# echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list
root@vm1:/# apt-get update
root@vm1:/# apt-get install -y postgresql-16
```
### 1.2.1. Проверим статус службы PostgreSQL
```bash
root@vm1:/# pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 down   postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```
### 1.2.2. Проверим ip адреса виртуальных машин
```bash
docker inspect vm1 | grep IPAddress
"IPAddress": "172.20.0.2"
docker inspect vm2 | grep IPAddress
"IPAddress": "172.20.0.3"
docker inspect vm3 | grep IPAddress
"IPAddress": "172.20.0.4"
```
### 1.2.3. Проверим состояние сети
```bash
root@vm1:/# nc -vz 127.20.0.3 5432
Connection to 127.20.0.3 5432 port [tcp/postgresql] succeeded!
```
```bash
root@vm1:/# nc -vz 127.20.0.4 5432
Connection to 127.20.0.4 5432 port [tcp/postgresql] succeeded!
```

**Вывод:** Сеть настроена корректно. Машины видят друг друга.

## 2.1 Создание таблиц и публикаций
### 2.1.1 Создадим базу данных с таблицами `test` и `test2` на виртуальной машине vm1
```bash
root@vm1:/# sudo -u postgres psql -c "CREATE DATABASE test;"
root@vm1:/# sudo -u postgres psql -d test -c "CREATE TABLE test (id serial PRIMARY KEY, name VARCHAR (50));"
root@vm1:/# sudo -u postgres psql -d test -c "INSERT INTO test (name) VALUES ('Jon Doe');"
root@vm1:/# sudo -u postgres psql -d test -c "CREATE TABLE test2 (id serial PRIMARY KEY, name VARCHAR (50));"
root@vm1:/# sudo -u postgres psql -d test -c "INSERT INTO test2 (name) VALUES ('Jane Doe');"
```
### 2.1.2 Настроим тип репликации `wal_level` на виртуальной машине vm1 и vm2
```bash
root@vm1:/# sudo -u postgres psql -d test -c "ALTER SYSTEM SET wal_level = logical;"
```
```bash
root@vm2:/# sudo -u postgres psql -d test -c "ALTER SYSTEM SET wal_level = logical;"
```

### 2.1.3 Создадим публикацию таблицы `test` на виртуальной машине vm1
```bash
root@vm1:/# sudo -u postgres psql -d test -c "CREATE PUBLICATION test_pub FOR TABLE test;"
WARNING:  wal_level is insufficient to publish logical changes
HINT:  Set wal_level to "logical" before creating subscriptions.
CREATE PUBLICATION
```
### 2.1.4 Перезапустим класстер PostgreSQL на виртуальной машине vm1
```bash
root@vm1:/# sudo -u postgres pg_ctlcluster 16 main restart
```
### 2.1.5 Проверим статус публикации таблицы `test` на виртуальной машине vm1
```bash
root@vm1:/# sudo -u postgres psql -d test -c "\dRp+ test_pub"
```
<details>
<summary>Детали</summary>

```bash
                            Publication test_pub
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "public.test"
```
</details>
<br>

**Вывод:** Изначально мы забыли перезапустить кластер и получили предупреждение. После перезапуска публикация таблицы `test` была настроена корректно.

### 2.1.6 Аналогичным образом создадим публикацию таблицы `test2` на виртуальной машине vm2.
```bash
root@vm2:/# sudo -u postgres psql -d test -c "CREATE PUBLICATION test2_pub FOR TABLE test2;"
CREATE PUBLICATION
```
### 2.1.7  Проверим статус публикации таблицы `test2` на виртуальной машине vm2
```bash
root@vm2:/# sudo -u postgres psql -d test -c "\dRp+ test2_pub"
```
<details>
<summary>Детали</summary>

```bash
                             Publication test2_pub
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "public.test2"
```
</details>
<br>

**Вывод:** Публикация таблицы `test2` настроена корректно.

### 2.2.1 Создадим подписку на публикацию таблицы `test` на виртуальной машине vm2
```bash
root@vm2:/# sudo -u postgres psql -d test -c "CREATE SUBSCRIPTION test_sub CONNECTION 'host=127.20.0.2 port=5432 dbname=test user=postgres password=postgres' PUBLICATION test_pub;"
ERROR:  could not connect to the publisher: connection to server at "127.20.0.2", port 5432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?
```
*Публикация не была создана так-как не был настроен `listen_addresses` в `postgresql.conf` на виртуальной машине vm1*

### 2.2.2 Настроим `listen_addresses` в `postgresql.conf` на виртуальных машинах vm1 и vm2.
```bash
root@vm1:/# echo "listen_addresses = '*'" >> /etc/postgresql/16/main/postgresql.conf
```
```bash
root@vm2:/# echo "listen_addresses = '*'" >> /etc/postgresql/16/main/postgresql.conf
```
### 2.2.4 Включим авторизацию в подсети 172.20.0.0/16 в `pg_hba.conf` на виртуальных машинах vm1 и vm2.
```bash
root@vm1:/# echo "host    all     all        172.20.0.0/16           md5" >> /etc/postgresql/16/main/pg_hba.conf
```
```bash
root@vm:/# echo "host    all      all        172.20.0.0/16           md5" >> /etc/postgresql/16/main/pg_hba.conf
```

### 2.2.3 Перезапустим кластер PostgreSQL на виртуальной машине vm1 и vm2
```bash
root@vm1:/# sudo -u postgres pg_ctlcluster 16 main restart
```
```bash
root@vm2:/# sudo -u postgres pg_ctlcluster 16 main restart
```
### 2.2.4 Повторим попытку создания подписки на публикацию таблицы `test` на виртуальной машине vm2
```bash
root@vm2:/# sudo -u postgres psql -d test -c "CREATE SUBSCRIPTION test_sub CONNECTION 'host=172.20.0.2 port=5432 dbname=test user=postgres password=postgres' PUBLICATION test_pub WITH (copy_data = false);"
NOTICE:  created replication slot "test_sub" on publisher
CREATE SUBSCRIPTION
```
### 2.2.5 Проверим статус подписки на виртуальной машине vm2
```bash
root@vm2:/# sudo -u postgres psql -d test -c "\dRs+ test_sub"
```
<details>
<summary>Детали</summary>

```bash
                                                                                                                    List of subscriptions
   Name   |  Owner   | Enabled | Publication | Binary | Streaming | Two-phase commit | Disable on error | Origin | Password required | Run as owner? | Synchronous commit |                               Conninfo                                | Skip LSN
----------+----------+---------+-------------+--------+-----------+------------------+------------------+--------+-------------------+---------------+--------------------+-----------------------------------------------------------------------+----------
 test_sub | postgres | t       | {test_pub}  | f      | off       | d                | f                | any    | t                 | f             | off                | host=172.20.0.2 port=5432 dbname=test user=postgres password=postgres | 0/0
(1 row)
```
</details>
<br>

### 2.2.6 Проверим работу подписки на виртуальной машине vm2
**Добавим данные в таблицу `test` на виртуальной машине vm1**
```bash
root@vm1:/# sudo -u postgres psql -d test -c "INSERT INTO test (name) VALUES ('John Doe');"
INSERT 0 1
```
**Проверим данные в таблице `test` на виртуальной машине vm2**
```bash
root@vm2:/# sudo -u postgres psql -d test -c "SELECT * FROM test;"
```
<details>
<summary>Детали</summary>

```bash
  id |   name
----+----------
  1 | Jon Doe
  2 | John Doe
(2 rows)
```
</details>
<br>

**Вывод:** Подписка на публикацию таблицы `test` настроена корректно.

### 2.2.7 Аналогичным образом создадим подписку на публикацию таблицы `test2` на виртуальной машине vm1
```bash
root@vm1:/# sudo -u postgres psql -d test -c "CREATE SUBSCRIPTION test2_sub CONNECTION 'host=172.20.0.3 port=5432 dbname=test user=postgres password=postgres' PUBLICATION test2_pub WITH (copy_data = false);"
NOTICE:  created replication slot "test2_sub" on publisher
CREATE SUBSCRIPTION
```
### 2.2.8 Проверим статус подписки на виртуальной машине vm1
```bash
root@vm1:/# sudo -u postgres psql -d test -c "\dRs+ test2_sub"
```
<details>
<summary>Детали</summary>

```bash
                                                                                                                       List of subscriptions
   Name    |  Owner   | Enabled | Publication | Binary | Streaming | Two-phase commit | Disable on error | Origin | Password required | Run as owner? | Synchronous commit |                               Conninfo                                | Skip LSN
-----------+----------+---------+-------------+--------+-----------+------------------+------------------+--------+-------------------+---------------+--------------------+-----------------------------------------------------------------------+----------
 test2_sub | postgres | t       | {test2_pub} | f      | off       | d                | f                | any    | t                 | f             | off                | host=172.20.0.3 port=5432 dbname=test user=postgres password=postgres | 0/0
(1 row)
```
</details>
<br>

### 2.2.9 Проверим работу подписки на виртуальной машине vm1
**Добавим данные в таблицу `test2` на виртуальной машине vm2**
```bash
root@vm2:/# sudo -u postgres psql -d test -c "INSERT INTO test2 (name) VALUES ('Jane Doe');"
INSERT 0 1
```
**Проверим данные в таблице `test2` на виртуальной машине vm1**
```bash
root@vm1:/# sudo -u postgres psql -d test -c "SELECT * FROM test2;"
```
<details>
<summary>Детали</summary>

```bash
   id |   name
----+----------
  1 | Jane Doe
  2 | Jane Doe
(2 rows)
```
</details>

**Вывод:** Подписка на публикацию таблицы `test2` настроена корректно.

### 2.3.1 Создадим базу данных с таблицами `test` и `test2` на виртуальной машине vm3
```bash
root@vm3:/# sudo -u postgres psql -c "CREATE DATABASE test;"
root@vm3:/# sudo -u postgres psql -d test -c "CREATE TABLE test (id serial PRIMARY KEY, name VARCHAR (50));"
root@vm3:/# sudo -u postgres psql -d test -c "CREATE TABLE test2 (id serial PRIMARY KEY, name VARCHAR (50));"
```

### 2.3.2 Создадим подписку на публикацию таблицы `test` на виртуальной машине vm3
```bash
root@vm3:/# sudo -u postgres psql -d test -c "CREATE SUBSCRIPTION test_sub_vm3 CONNECTION 'host=172.20.0.2 port=5432 dbname=test user=postgres password=postgres' PUBLICATION test_pub;"
NOTICE:  created replication slot "test_sub_vm3" on publisher
CREATE SUBSCRIPTION
```
### 2.3.3 Проверим статус подписки на виртуальной машине vm3
```bash
root@vm3:/# sudo -u postgres psql -d test -c "\dRs+ test_sub_vm3"
```
<details>
<summary>Детали</summary>

```bash
                                                                                                                         List of subscriptions
     Name     |  Owner   | Enabled | Publication | Binary | Streaming | Two-phase commit | Disable on error | Origin | Password required | Run as owner? | Synchronous commit |                               Conninfo
                                | Skip LSN
--------------+----------+---------+-------------+--------+-----------+------------------+------------------+--------+-------------------+---------------+--------------------+---------------------------------------
--------------------------------+----------
 test_sub_vm3 | postgres | t       | {test_pub}  | f      | off       | d                | f                | any    | t                 | f             | off                | host=172.20.0.2 port=5432 dbname=test
user=postgres password=postgres | 0/0
(1 row)
```
</details>
<br>

### 2.3.4 Проверим работу подписки на виртуальной машине vm3
**Добавим данные в таблицу `test` на виртуальной машине vm1**
```bash
root@vm1:/# sudo -u postgres psql -d test -c "INSERT INTO test (name) VALUES ('John Johnson');"
INSERT 0 1
```
**Проверим данные в таблице `test` на виртуальной машине vm3**
```bash
root@vm3:/# sudo -u postgres psql -d test -c "SELECT * FROM test;"
```
<details>
<summary>Детали</summary>

```bash
 id |     name
----+--------------
  1 | Jon Doe
  2 | John Doe
  3 | John Johnson
(3 rows)
```
</details>
<br>

**Вывод:** Подписка на публикацию таблицы `test` настроена корректно.

### 2.3.5 Аналогичным образом создадим подписку на публикацию таблицы `test2` из vm2 на виртуальной машине vm3
```bash
root@vm3:/# sudo -u postgres psql -d test -c "CREATE SUBSCRIPTION test2_sub_vm3 CONNECTION 'host=172.20.0.3 port=5432 dbname=test user=postgres password=postgres' PUBLICATION test2_pub WITH (copy_data = false);"
NOTICE:  created replication slot "test2_sub_vm3" on publisher
CREATE SUBSCRIPTION
```
### 2.3.6 Проверим статус подписки на виртуальной машине vm3
```bash
root@vm3:/# sudo -u postgres psql -d test -c "\dRs+ test2_sub_vm3"
```
<details>
<summary>Детали</summary>

```bash
                                                                                                                      List of subscriptions
     Name      |  Owner   | Enabled | Publication | Binary | Streaming | Two-phase commit | Disable on error | Origin | Password required | Run as owner? | Synchronous commit |                               Conninf
o                                | Skip LSN
---------------+----------+---------+-------------+--------+-----------+------------------+------------------+--------+-------------------+---------------+--------------------+--------------------------------------
---------------------------------+----------
 test2_sub_vm3 | postgres | t       | {test2_pub} | f      | off       | d                | f                | any    | t                 | f             | off                | host=172.20.0.3 port=5432 dbname=test
 user=postgres password=postgres | 0/0
(1 row)
```
</details>
<br>

### 2.3.7 Проверим работу подписки на виртуальной машине vm3
**Добавим данные в таблицу `test2` на виртуальной машине vm2**
```bash
root@vm2:/# sudo -u postgres psql -d test -c "INSERT INTO test2 (name) VALUES ('Jane Johnson');"
INSERT 0 1
```

**Проверим данные в таблице `test2` на виртуальной машине vm3**
```bash
root@vm3:/# sudo -u postgres psql -d test -c "SELECT * FROM test2;"
```

<details>
<summary>Детали</summary>

```bash
 id |     name
----+--------------
  3 | Jane Johnson
(1 row)
```
</details>
<br>

**Вывод:** Подписка на публикацию таблицы `test2` из vm2 настроена корректно.
Тепрь vm3 подписана на таблицы из vm1 и vm2 и ее можно использовать как реплику для чтения и бэкапов.

## 3. Дополнительное задание. Настраиваем горячее реплицирование для высокой доступности на 4 ВМ
### 3.1 Создадим виртуальную машину vm4 аналогично vm1, vm2 и vm3
```bash
➜  ~ docker run -it -p 5554:5432 --network pg-cluster-network --rm --name vm4 --hostname vm4 ubuntu:20.04 bash
```
### 3.2 Установим PostgreSQL 16 на виртуальной машине vm4 согластно пункту 1.2
### 3.3 Сконфигурируем доступы и адреса с которых можно подключаться к PostgreSQL на виртульной машине vm4
```bash
root@vm4:/# echo "listen_addresses = '*'" >> /etc/postgresql/16/main/postgresql.conf
root@vm4:/# echo "host    replication     replica     172.20.0.0/16           md5" >> /etc/postgresql/16/main/pg_hba.conf
root@vm4:/# echo "host    all             rewind      172.20.0.0/16           md5" >> /etc/postgresql/16/main/pg_hba.conf
```
### 3.4 Включим `wal_log_hints` в `postgresql.conf` на виртуальной машине vm4
```bash
root@vm4:/# echo "wal_log_hints = on" >> /etc/postgresql/16/main/postgresql.conf
```
> `wal_log_hints` - включает запись подсказок в WAL логи, которые позволяют уменьшить количество запросов к диску при чтении данных из таблицы.
>
> Дополнительная информация: https://www.postgresql.org/docs/16/runtime-config-wal.html#GUC-WAL-LOG-HINTS

### 3.5 Включим архивацию WAL логов на виртуальной машине vm4
```bash
root@vm4:/# echo "archive_mode = on" >> /etc/postgresql/16/main/postgresql.conf
root@vm4:/# echo "archive_command = 'test ! -f /var/lib/postgresql/16/archive/%f && cp %p /var/lib/postgresql/16/archive/%f'" >> /etc/postgresql/16/main/postgresql.conf
```
> `archive_mode` - включает архивацию WAL логов. При включении этой опции PostgreSQL будет копировать WAL логи в архивные файлы.
>
> Дополнительная информация: https://www.postgresql.org/docs/16/runtime-config-wal.html#GUC-ARCHIVE-MODE

> `archive_command` - команда, которая будет выполнена для архивации WAL логов.
>
> Дополнительная информация: https://www.postgresql.org/docs/16/runtime-config-wal.html#GUC-ARCHIVE-COMMAND

### 3.6 Создадим директорию для архивации WAL логов на виртуальной машине vm4
```bash
root@vm4:/# mkdir -p /var/lib/postgresql/16/archive
root@vm4:/# chown -R postgres:postgres /var/lib/postgresql/16/archive
```
### 3.7 Аналоично настроим vm3
### 3.8 Проверим доступность vm4 с vm3
```bash
root@vm3:/# ping 172.20.0.4
```
<details>
<summary>Детали</summary>

```bash
PING 172.20.0.4 (172.20.0.4) 56(84) bytes of data.
64 bytes from 172.20.0.4: icmp_seq=1 ttl=64 time=0.330 ms
64 bytes from 172.20.0.4: icmp_seq=2 ttl=64 time=0.095 ms
64 bytes from 172.20.0.4: icmp_seq=3 ttl=64 time=0.089 ms
64 bytes from 172.20.0.4: icmp_seq=4 ttl=64 time=0.125 ms
64 bytes from 172.20.0.4: icmp_seq=5 ttl=64 time=0.096 ms
^C
--- 172.20.0.4 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4113ms
rtt min/avg/max/mdev = 0.089/0.147/0.330/0.092 ms
```
</details>
<br>

### 3.9 Перезапустим кластер PostgreSQL на виртуальной машине vm3
```bash
root@vm3:/# sudo -u postgres pg_ctlcluster 16 main restart
```
### 3.10 Создадим пользователей для репликации и восстановления на виртуальной машине vm3
```bash
root@vm3:/# sudo -u postgres psql -c "CREATE USER replica REPLICATION LOGIN ENCRYPTED PASSWORD 'replica';"
root@vm3:/# sudo -u postgres psql -c "CREATE USER rewind SUPERUSER LOGIN ENCRYPTED PASSWORD 'rewind';"
```
### 3.11 Проверяем доступность vm3 с vm4
```bash
root@vm4:/# nc -vz 172.20.0.3 5432
Connection to 172.20.0.3 5432 port [tcp/postgresql] succeeded!
```
### 3.12 Удалим все данные на виртуальной машине vm4
```bash
root@vm4:/# rm -rf /var/lib/postgresql/16/main/*
```
### 3.13 Восстановим данные на виртуальной машине vm4 из архива на виртуальной машине vm3
```bash
root@vm4:/# sudo -u postgres pg_basebackup -h 172.20.0.4 -D /var/lib/postgresql/16/main -U replica -P -R -C -S replica1 -X stream
Password:
30736/30736 kB (100%), 1/1 tablespace
```
### 3.14 Проверим статус кластера PostgreSQL на виртуальной машине vm4
```bash
root@vm4:/# sudo -u postgres pg_ctlcluster 16 main status
pg_ctl: no server running
```
### 3.15 Запустим кластер PostgreSQL на виртуальной машине vm4
```bash
root@vm4:/# sudo -u postgres pg_ctlcluster 16 main start
```
### 3.16 Проверим данные на виртульной машине vm4
```bash
root@vm4:/# sudo -u postgres psql -d test -c "SELECT * FROM test;"
```
<details>
<summary>Детали</summary>

```bash
 id |     name
----+--------------
  1 | Jon Doe
  2 | John Doe
  3 | John Johnson
(3 rows)
```
</details>
<br>

### 3.17 Проверим информацию о репликации на виртуальной машине vm4
```bash
root@vm4:/# cat /var/lib/postgresql/16/main/postgresql.auto.conf
```
<details>
<summary>Детали</summary>

```bash
# Do not edit this file manually!
# It will be overwritten by the ALTER SYSTEM command.
primary_conninfo = 'user=replica password=replica channel_binding=prefer host=172.20.0.4 port=5432 sslmode=prefer sslcompression=0 sslcertmode=allow sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres gssdelegation=0 target_session_attrs=any load_balance_hosts=disable'
primary_slot_name = 'replica1'
```
</details>
<br>

### 3.18 Убедимся что был создан файл `standby.signal` на виртуальной машине vm4
```bash
root@vm4:/# ls /var/lib/postgresql/16/main/ | grep standby.signal
standby.signal
```
> Файл `standby.signal` создается автоматически при восстановлении из архива и указывает на то, что кластер находится в режиме standby. В этом режиме кластер ожидает и применяет новые WAL логи от основного кластера.

### 3.19 Проверим статус кластера PostgreSQL на виртуальной машине vm4
```bash
root@vm4:/# sudo -u postgres pg_lsclusters
Ver Cluster Port Status          Owner    Data directory              Log file
16  main    5432 online,recovery postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```
*Мы видим, что кластер на виртуальной машине vm4 находится в режиме `online,recovery`*

### 3.20 Проверим слоты репликации на виртуальной машине vm3
```bash
root@vm3:/# sudo -u postgres psql -c "SELECT * FROM pg_replication_slots;"
```
<details>
<summary>Детали</summary>

```bash
 slot_name | plugin | slot_type | datoid | database | temporary | active | active_pid | xmin | catalog_xmin | restart_lsn | confirmed_flush_lsn | wal_status | safe_wal_size | two_phase | conflicting
-----------+--------+-----------+--------+----------+-----------+--------+------------+------+--------------+-------------+---------------------+------------+---------------+-----------+-------------
 replica1  |        | physical  |        |          | f         | t      |      12947 |      |              | 0/5000148   |                     | reserved   |               | f         |
(1 row)
```
</details>
<br>

### 3.21 Проверим статитику по слотам репликации на виртуальной машине vm3
```bash
root@vm3:/# sudo -u postgres psql -c "SELECT * FROM pg_stat_replication;"
```
<details>
<summary>Детали</summary>

```bash
  pid  | usesysid | usename | application_name | client_addr | client_hostname | client_port |         backend_start         | backend_xmin |   state   | sent_lsn  | write_lsn | flush_lsn | replay_lsn | write_lag |
 flush_lag | replay_lag | sync_priority | sync_state |          reply_time
-------+----------+---------+------------------+-------------+-----------------+-------------+-------------------------------+--------------+-----------+-----------+-----------+-----------+------------+-----------+
-----------+------------+---------------+------------+-------------------------------
 12947 |    16412 | replica | 16/main          | 172.20.0.5  |                 |       53074 | 2024-06-14 00:22:24.440195+03 |              | streaming | 0/5000148 | 0/5000148 | 0/5000148 | 0/5000148  |           |
           |            |             0 | async      | 2024-06-14 00:35:42.317373+03
(1 row)
```
</details>
<br>

*Слот репликации `replica1` на виртуальной машине vm3 активен и находится в режиме `streaming`*

### 3.22 Проверим репликацию данных между виртуальными машинами vm3 и vm4
```bash
root@vm3:/# sudo -u postgres psql -d test -c "CREATE TABLE test3 (id serial PRIMARY KEY, name VARCHAR (50));"
CREATE TABLE
root@vm3:/# sudo -u postgres psql -d test -c "INSERT INTO test3 (name) VALUES ('John Brown');"
INSERT 0 1
```
```bash
root@vm4:/# sudo -u postgres psql -d test -c "SELECT * FROM test3;"
```
<details>
<summary>Детали</summary>

```bash
 id |    name
----+------------
  1 | John Brown
(1 row)
```
</details>
<br>

*Репликация данных между виртуальными машинами vm3 и vm4 настроена корректно.*

# Выводы
В процессе выполнения данной лабораторной работы были выполнены следующие задачи:
1. Настроена логическая репликация данных между виртуальными машинами vm1, vm2 и vm3.
2. Настроено горячее реплицирование для высокой доступности между виртуальными машинами vm3 и vm4.

