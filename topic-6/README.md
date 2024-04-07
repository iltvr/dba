# Домашнее задание

## Работа с журналами

**Цель:**
- уметь работать с журналами и контрольными точками
- уметь настраивать параметры журналов

**Описание/Пошаговая инструкция выполнения домашнего задания:**
1. Настройте выполнение контрольной точки раз в 30 секунд.
2. В течение 10 минут с помощью утилиты pgbench подавайте нагрузку.
3. Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.
4. Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?
5. Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.
6. Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? Как проигнорировать ошибку и продолжить работу?

**Критерии оценки:**
- Выполнение ДЗ: 10 баллов
- плюс 2 балла за красивое решение
- минус 2 балла за рабочее решение, и недостатки, указанные преподавателем, не устранены

# Домашняя работа к занятию "9. Журналы" от 05.03.2024
## 0. Run postgres in docker
### Create local volume directory
```bash
➜  ~ sudo mkdir -p /var/lib/postgres
```
### Add permissions to the directory
```bash
➜  ~ sudo chown -R $USER: /var/lib/postgres
```
### Run postgres in docker
```bash
➜  ~ sudo docker run --name otus-db-pg-1 -e POSTGRES_PASSWORD=postgres -e POSTGRES_USER=postgres -d -p 5433:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
```
### Check connection
```bash
➜  ~ psql -U postgres -p 5433 -h 127.0.0.1
Password for user postgres:
psql (15.5 (Homebrew))
Type "help" for help.

postgres=#
```

## 1. Setup checkpoint timeout to 30s
### Check current `checkpoint_timeout`
```psql
ppostgres=# show checkpoint_timeout;
 checkpoint_timeout
--------------------
 5min
(1 row)
```

> `checkpoint_timeout (integer)`
>
> Maximum time between automatic WAL checkpoints. If this value is specified without units, it is taken as seconds. The valid range is between 30 seconds and one day. The default is five minutes (5min). Increasing this parameter can increase the amount of time needed for crash recovery. This parameter can only be set in the postgresql.conf file or on the server command line.

### Change `checkpoint_timeout` to 30s
```psql
postgres=# alter system set checkpoint_timeout = '30s';
ALTER SYSTEM
```
### Restart postgres
```bash
➜  ~ sudo docker restart otus-db-pg-1
otus-db-pg-1
```
### Check new `checkpoint_timeout`
```psql
postgres=# show checkpoint_timeout;
 checkpoint_timeout
--------------------
 30s
```
## 2. Run benchmark for 10 minutes
### Check current WAL files
```bash
➜  ~ ls -lhT /var/lib/postgres/pg_wal
total 32768
-rw-------@ 1 iliakriachkov  wheel    16M Apr  5 09:15:55 2024 000000010000000000000001
drwx------  2 iliakriachkov  wheel    64B Apr  5 09:14:37 2024 archive_status
```
### Check WAL files size
```bash
➜  ~ sudo du -sh /var/lib/postgres/pg_wal
 16M	/var/lib/postgres/pg_wal
```
### Init `pgbench`
```bash
➜  ~ pgbench -i -s 10 -U postgres -p 5433 -h 127.0.0.1 postgres
Password:
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
1000000 of 1000000 tuples (100%) done (elapsed 2.65 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 4.11 s (drop tables 0.00 s, create tables 0.02 s, client-side generate 3.38 s, vacuum 0.13 s, primary keys 0.56 s).
```
### Check WAL files size
```bash
➜  ~ sudo du -sh /var/lib/postgres/pg_wal
 144M	/var/lib/postgres/pg_wal
```
### Check WAL files
```bash
➜  ~ ls -lhT /var/lib/postgres/pg_wal
total 294912
-rw-------@ 1 iliakriachkov  wheel    16M Apr  5 09:16:47 2024 000000010000000000000001
-rw-------  1 iliakriachkov  wheel    16M Apr  5 09:16:48 2024 000000010000000000000002
-rw-------  1 iliakriachkov  wheel    16M Apr  5 09:16:49 2024 000000010000000000000003
-rw-------  1 iliakriachkov  wheel    16M Apr  5 09:16:49 2024 000000010000000000000004
-rw-------  1 iliakriachkov  wheel    16M Apr  5 09:16:50 2024 000000010000000000000005
-rw-------  1 iliakriachkov  wheel    16M Apr  5 09:16:50 2024 000000010000000000000006
-rw-------  1 iliakriachkov  wheel    16M Apr  5 09:16:51 2024 000000010000000000000007
-rw-------  1 iliakriachkov  wheel    16M Apr  5 09:16:51 2024 000000010000000000000008
-rw-------  1 iliakriachkov  wheel    16M Apr  5 09:17:02 2024 000000010000000000000009
drwx------  2 iliakriachkov  wheel    64B Apr  5 09:14:37 2024 archive_status
```
### Check bgwriter stats
```psql
postgres=# select * from pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 3
checkpoints_req       | 2
checkpoint_write_time | 26592
checkpoint_sync_time  | 20
buffers_checkpoint    | 2177
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 30803
buffers_backend_fsync | 0
buffers_alloc         | 2769
stats_reset           | 2024-04-05 06:14:37.685566+00
```
### Reset bgwriter stats
```psql
postgres=# select pg_stat_reset_shared('bgwriter');
 pg_stat_reset_shared
----------------------

(1 row)
```
### Run `pgbench`
```bash
➜  ~ pgbench -T 600 -U postgres -p 5433 -h 127.0.0.1 postgres
Password:
pgbench (15.5 (Homebrew))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 10
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 889316
number of failed transactions: 0 (0.000%)
latency average = 0.675 ms
initial connection time = 15.718 ms
tps = 1482.224905 (without initial connection time)
```
## 3. Check WAL size
### List WAL files
```bash
➜  ~ ls -lhT /var/lib/postgres/pg_wal
total 655360
-rw-------  1 iliakriachkov  wheel    16M Apr  5 09:31:27 2024 0000000100000000000000BA
-rw-------  1 iliakriachkov  wheel    16M Apr  5 09:31:28 2024 0000000100000000000000BB
-rw-------  1 iliakriachkov  wheel    16M Apr  5 09:31:31 2024 0000000100000000000000BC
-rw-------  1 iliakriachkov  wheel    16M Apr  5 09:31:33 2024 0000000100000000000000BD
-rw-------  1 iliakriachkov  wheel    16M Apr  5 09:31:36 2024 0000000100000000000000BE
-rw-------  1 iliakriachkov  wheel    16M Apr  5 09:31:39 2024 0000000100000000000000BF
-rw-------  1 iliakriachkov  wheel    16M Apr  5 09:32:08 2024 0000000100000000000000C0
-rw-------  1 iliakriachkov  wheel    16M Apr  5 09:30:34 2024 0000000100000000000000C1
-rw-------  1 iliakriachkov  wheel    16M Apr  5 09:30:37 2024 0000000100000000000000C2
-rw-------  1 iliakriachkov  wheel    16M Apr  5 09:30:41 2024 0000000100000000000000C3
-rw-------  1 iliakriachkov  wheel    16M Apr  5 09:30:32 2024 0000000100000000000000C4
-rw-------@ 1 iliakriachkov  wheel    16M Apr  5 09:31:16 2024 0000000100000000000000C5
-rw-------  1 iliakriachkov  wheel    16M Apr  5 09:30:57 2024 0000000100000000000000C6
-rw-------  1 iliakriachkov  wheel    16M Apr  5 09:31:08 2024 0000000100000000000000C7
-rw-------  1 iliakriachkov  wheel    16M Apr  5 09:31:11 2024 0000000100000000000000C8
-rw-------  1 iliakriachkov  wheel    16M Apr  5 09:31:24 2024 0000000100000000000000C9
-rw-------  1 iliakriachkov  wheel    16M Apr  5 09:31:05 2024 0000000100000000000000CA
-rw-------  1 iliakriachkov  wheel    16M Apr  5 09:30:58 2024 0000000100000000000000CB
-rw-------  1 iliakriachkov  wheel    16M Apr  5 09:31:01 2024 0000000100000000000000CC
-rw-------  1 iliakriachkov  wheel    16M Apr  5 09:31:03 2024 0000000100000000000000CD
drwx------  2 iliakriachkov  wheel    64B Apr  5 09:14:37 2024 archive_status
```

*There we can see 20 WAL files (except the file `archive_status`)*

### Calculate files count in WAL directory
```bash
➜  ~ find /var/lib/postgres/pg_wal/ -maxdepth 1 -type f | wc -l
      20
```
### Calculate total size of WAL files
```bash
➜  ~ du -sh /var/lib/postgres/pg_wal
320M	/var/lib/postgres/pg_wal
```
*We can see total size of WAL files is 320 MB now. So it was increased by 176 MB during the 10 minutes of pgbench run*

### Check LOGs
```log
2024-04-05 09:21:56 2024-04-05 06:21:56.378 UTC [27] LOG:  checkpoint starting: time
2024-04-05 09:22:23 2024-04-05 06:22:23.084 UTC [27] LOG:  checkpoint complete: wrote 12344 buffers (75.3%); 0 WAL file(s) added, 0 removed, 7 recycled; write=26.660 s, sync=0.025 s, total=26.706 s; sync files=23, longest=0.021 s, average=0.002 s; distance=124855 kB, estimate=126010 kB
2024-04-05 09:22:26 2024-04-05 06:22:26.091 UTC [27] LOG:  checkpoint starting: time
2024-04-05 09:22:53 2024-04-05 06:22:53.137 UTC [27] LOG:  checkpoint complete: wrote 10051 buffers (61.3%); 0 WAL file(s) added, 0 removed, 10 recycled; write=26.998 s, sync=0.023 s, total=27.044 s; sync files=9, longest=0.017 s, average=0.003 s; distance=156233 kB, estimate=156233 kB
2024-04-05 09:22:56 2024-04-05 06:22:56.140 UTC [27] LOG:  checkpoint starting: time
2024-04-05 09:23:23 2024-04-05 06:23:23.119 UTC [27] LOG:  checkpoint complete: wrote 9285 buffers (56.7%); 0 WAL file(s) added, 0 removed, 8 recycled; write=26.926 s, sync=0.031 s, total=26.979 s; sync files=14, longest=0.021 s, average=0.003 s; distance=134775 kB, estimate=154087 kB
2024-04-05 09:23:26 2024-04-05 06:23:26.126 UTC [27] LOG:  checkpoint starting: time
2024-04-05 09:23:53 2024-04-05 06:23:53.059 UTC [27] LOG:  checkpoint complete: wrote 9782 buffers (59.7%); 0 WAL file(s) added, 0 removed, 9 recycled; write=26.879 s, sync=0.030 s, total=26.932 s; sync files=16, longest=0.022 s, average=0.002 s; distance=142160 kB, estimate=152894 kB
2024-04-05 09:23:56 2024-04-05 06:23:56.065 UTC [27] LOG:  checkpoint starting: time
2024-04-05 09:24:23 2024-04-05 06:24:23.143 UTC [27] LOG:  checkpoint complete: wrote 9469 buffers (57.8%); 0 WAL file(s) added, 0 removed, 8 recycled; write=27.028 s, sync=0.031 s, total=27.078 s; sync files=12, longest=0.024 s, average=0.003 s; distance=141232 kB, estimate=151728 kB
2024-04-05 09:24:26 2024-04-05 06:24:26.146 UTC [27] LOG:  checkpoint starting: time
2024-04-05 09:24:53 2024-04-05 06:24:53.060 UTC [27] LOG:  checkpoint complete: wrote 9746 buffers (59.5%); 0 WAL file(s) added, 0 removed, 9 recycled; write=26.858 s, sync=0.031 s, total=26.914 s; sync files=9, longest=0.024 s, average=0.004 s; distance=140238 kB, estimate=150579 kB
2024-04-05 09:24:56 2024-04-05 06:24:56.065 UTC [27] LOG:  checkpoint starting: time
2024-04-05 09:25:23 2024-04-05 06:25:23.077 UTC [27] LOG:  checkpoint complete: wrote 9061 buffers (55.3%); 0 WAL file(s) added, 0 removed, 8 recycled; write=26.964 s, sync=0.030 s, total=27.012 s; sync files=9, longest=0.024 s, average=0.004 s; distance=140423 kB, estimate=149564 kB
2024-04-05 09:25:26 2024-04-05 06:25:26.080 UTC [27] LOG:  checkpoint starting: time
2024-04-05 09:25:53 2024-04-05 06:25:53.129 UTC [27] LOG:  checkpoint complete: wrote 9849 buffers (60.1%); 0 WAL file(s) added, 0 removed, 9 recycled; write=26.999 s, sync=0.031 s, total=27.049 s; sync files=15, longest=0.023 s, average=0.003 s; distance=144232 kB, estimate=149030 kB
2024-04-05 09:25:56 2024-04-05 06:25:56.137 UTC [27] LOG:  checkpoint starting: time
2024-04-05 09:26:23 2024-04-05 06:26:23.112 UTC [27] LOG:  checkpoint complete: wrote 9333 buffers (57.0%); 0 WAL file(s) added, 0 removed, 9 recycled; write=26.925 s, sync=0.024 s, total=26.975 s; sync files=10, longest=0.017 s, average=0.003 s; distance=142375 kB, estimate=148365 kB
2024-04-05 09:26:26 2024-04-05 06:26:26.120 UTC [27] LOG:  checkpoint starting: time
2024-04-05 09:26:53 2024-04-05 06:26:53.096 UTC [27] LOG:  checkpoint complete: wrote 9840 buffers (60.1%); 0 WAL file(s) added, 0 removed, 8 recycled; write=26.906 s, sync=0.030 s, total=26.976 s; sync files=11, longest=0.022 s, average=0.003 s; distance=141246 kB, estimate=147653 kB
2024-04-05 09:26:56 2024-04-05 06:26:56.099 UTC [27] LOG:  checkpoint starting: time
2024-04-05 09:27:23 2024-04-05 06:27:23.135 UTC [27] LOG:  checkpoint complete: wrote 9440 buffers (57.6%); 0 WAL file(s) added, 0 removed, 9 recycled; write=26.944 s, sync=0.040 s, total=27.035 s; sync files=11, longest=0.025 s, average=0.004 s; distance=141568 kB, estimate=147045 kB
2024-04-05 09:27:26 2024-04-05 06:27:26.141 UTC [27] LOG:  checkpoint starting: time
2024-04-05 09:27:53 2024-04-05 06:27:53.109 UTC [27] LOG:  checkpoint complete: wrote 10180 buffers (62.1%); 0 WAL file(s) added, 0 removed, 9 recycled; write=26.870 s, sync=0.051 s, total=26.968 s; sync files=15, longest=0.023 s, average=0.004 s; distance=147877 kB, estimate=147877 kB
2024-04-05 09:27:56 2024-04-05 06:27:56.114 UTC [27] LOG:  checkpoint starting: time
2024-04-05 09:28:23 2024-04-05 06:28:23.054 UTC [27] LOG:  checkpoint complete: wrote 10121 buffers (61.8%); 0 WAL file(s) added, 0 removed, 9 recycled; write=26.886 s, sync=0.028 s, total=26.940 s; sync files=9, longest=0.019 s, average=0.004 s; distance=150122 kB, estimate=150122 kB
2024-04-05 09:28:26 2024-04-05 06:28:26.056 UTC [27] LOG:  checkpoint starting: time
2024-04-05 09:28:53 2024-04-05 06:28:53.107 UTC [27] LOG:  checkpoint complete: wrote 10388 buffers (63.4%); 0 WAL file(s) added, 0 removed, 10 recycled; write=26.960 s, sync=0.062 s, total=27.051 s; sync files=16, longest=0.035 s, average=0.004 s; distance=151432 kB, estimate=151432 kB
2024-04-05 09:28:56 2024-04-05 06:28:56.111 UTC [27] LOG:  checkpoint starting: time
2024-04-05 09:29:23 2024-04-05 06:29:23.097 UTC [27] LOG:  checkpoint complete: wrote 10123 buffers (61.8%); 0 WAL file(s) added, 0 removed, 9 recycled; write=26.913 s, sync=0.039 s, total=26.986 s; sync files=10, longest=0.026 s, average=0.004 s; distance=153538 kB, estimate=153538 kB
2024-04-05 09:29:26 2024-04-05 06:29:26.101 UTC [27] LOG:  checkpoint starting: time
2024-04-05 09:29:53 2024-04-05 06:29:53.140 UTC [27] LOG:  checkpoint complete: wrote 10589 buffers (64.6%); 0 WAL file(s) added, 0 removed, 9 recycled; write=26.985 s, sync=0.029 s, total=27.039 s; sync files=16, longest=0.019 s, average=0.002 s; distance=155460 kB, estimate=155460 kB
2024-04-05 09:29:56 2024-04-05 06:29:56.144 UTC [27] LOG:  checkpoint starting: time
2024-04-05 09:30:23 2024-04-05 06:30:23.119 UTC [27] LOG:  checkpoint complete: wrote 10265 buffers (62.7%); 0 WAL file(s) added, 0 removed, 10 recycled; write=26.915 s, sync=0.030 s, total=26.975 s; sync files=8, longest=0.021 s, average=0.004 s; distance=153004 kB, estimate=155215 kB
2024-04-05 09:30:26 2024-04-05 06:30:26.122 UTC [27] LOG:  checkpoint starting: time
2024-04-05 09:30:53 2024-04-05 06:30:53.097 UTC [27] LOG:  checkpoint complete: wrote 10459 buffers (63.8%); 0 WAL file(s) added, 0 removed, 9 recycled; write=26.922 s, sync=0.029 s, total=26.975 s; sync files=15, longest=0.020 s, average=0.002 s; distance=149561 kB, estimate=154649 kB
2024-04-05 09:30:56 2024-04-05 06:30:56.100 UTC [27] LOG:  checkpoint starting: time
2024-04-05 09:31:23 2024-04-05 06:31:23.116 UTC [27] LOG:  checkpoint complete: wrote 9982 buffers (60.9%); 0 WAL file(s) added, 0 removed, 9 recycled; write=26.964 s, sync=0.028 s, total=27.016 s; sync files=8, longest=0.019 s, average=0.004 s; distance=146351 kB, estimate=153819 kB
2024-04-05 09:31:26 2024-04-05 06:31:26.122 UTC [27] LOG:  checkpoint starting: time
2024-04-05 09:31:53 2024-04-05 06:31:53.104 UTC [27] LOG:  checkpoint complete: wrote 10513 buffers (64.2%); 0 WAL file(s) added, 0 removed, 9 recycled; write=26.905 s, sync=0.038 s, total=26.981 s; sync files=15, longest=0.024 s, average=0.003 s; distance=145630 kB, estimate=153000 kB
2024-04-05 09:32:26 2024-04-05 06:32:26.128 UTC [27] LOG:  checkpoint starting: time
2024-04-05 09:32:53 2024-04-05 06:32:53.151 UTC [27] LOG:  checkpoint complete: wrote 7212 buffers (44.0%); 0 WAL file(s) added, 0 removed, 6 recycled; write=26.948 s, sync=0.044 s, total=27.024 s; sync files=14, longest=0.035 s, average=0.004 s; distance=104187 kB, estimate=148119 kB
```
*There we can see that during the 10 minutes of `pgbench` run 21 checkpoints were performed. Each checkpoint took around 27 seconds to complete. The distance between checkpoints was around 140 MB. The average distance between checkpoints was around 139 MB.*

*Taking into account the fact that the total WAL size has increased by 176 MB, we can assume that each checkpoint took about (176 / 21) 8.38 MB of WAL data.*

### Calculate data written to disk
> `distance=104187 kB`
>
> This indicates the amount of data written to disk during the checkpoint.

> `estimate=148119 kB`
>
> This was the estimated amount of data to be written (around 148 MB)

*So by summing the `distance` values from the logs we can get the total amount of data written to disk during the `pgbench` run*

```psql
postgres=# with checkpoints as (
select 124855 as distance union all
select 156233 union all
select 134775 union all
select 142160 union all
select 141232 union all
select 140238 union all
select 140423 union all
select 144232 union all
select 142375 union all
select 141246 union all
select 141568 union all
select 147877 union all
select 150122 union all
select 151432 union all
select 153538 union all
select 155460 union all
select 153004 union all
select 149561 union all
select 146351 union all
select 145630 union all
select 104187
)
select sum(distance) as total_distance_kb, sum(distance)/1024 as total_distance_mb, sum(distance)/count(*) as avg_distance_kb, sum(distance)/count(*)/1024 as avg_distance_mb from checkpoints;
 total_distance_kb | total_distance_mb | avg_distance_kb | avg_distance_mb
-------------------+-------------------+-----------------+-----------------
           3006499 |              2936 |          143166 |             139
(1 row)
```
*So we the total amount of data written to disk is around 2936 MB and the average amount of data written to disk during the checkpoint is around 139 MB*

## 4. Check checkpoints stats
```psql
postgres=# select * from pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 38
checkpoints_req       | 0
checkpoint_write_time | 565355
checkpoint_sync_time  | 704
buffers_checkpoint    | 207912
buffers_clean         | 127289
maxwritten_clean      | 76
buffers_backend       | 28929
buffers_backend_fsync | 0
buffers_alloc         | 201402
stats_reset           | 2024-04-05 06:21:21.770444+00
```
> `checkpoints_timed`
>
> Number of scheduled checkpoints that have been scheduled. These checkpoints were triggered when `checkpoint_timeout` was reached. But the checkpoint might be skipped if there is no activity in the database.

> `checkpoints_req`
>
> Number of requested checkpoints. These checkpoints might be manually triggered or caused by specific database events. E.g. when `max_wal_size` is reached.

*There we can see all checkpoints were scheduled since the last reset of the stats (since run `pgbench`) to the moment of the stats query. We can see that `checkpoints_timed` is 38 but actually only 21 checkpoints were performed. The rest of the checkpoints were skipped because there was no activity in the database.*

## 5. Compare tps in sync/async mode
> **Asynchronous commit** is an option that allows transactions to complete more quickly, at the cost that the most recent transactions may be lost if the database should crash. In many applications this is an acceptable trade-off.
>
> See more: [PostgreSQL Documentation](https://www.postgresql.org/docs/current/wal-async-commit.html)

### Switch to async mode
```psql
postgres=# alter system set synchronous_commit = off;
ALTER SYSTEM
```
### Restart postgres container
```bash
➜  ~ sudo docker restart otus-db-pg-1
otus-db-pg-1
```
### Check sync mode
```psql
postgres=# show synchronous_commit;
 synchronous_commit
--------------------
 off
(1 row)
```
### Run benchmark in async mode
```bash
➜  ~ pgbench -T 600 -U postgres -p 5433 -h 127.0.0.1 postgres
Password:
pgbench (15.5 (Homebrew))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 10
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 998123
number of failed transactions: 0 (0.000%)
latency average = 0.601 ms
initial connection time = 8.607 ms
tps = 1663.548176 (without initial connection time)
```
*Tps in async mode is 1663.55 vs 1482.22 in sync mode. The difference is around 12.2%*

*So we can see that tps in async mode is around 12.2% higher than in sync mode. This is because in async mode transactions are committed without waiting for WAL to be written to disk.*

## 6. Checksums
> **Page checksums**
>
> By default, data pages are not protected by checksums, but this can optionally be enabled for a cluster. When enabled, each data page includes a checksum that is updated when the page is written and verified each time the page is read. Only data pages are protected by checksums; internal data structures and temporary files are not.
>
> See more: [PostgreSQL Documentation](https://www.postgresql.org/docs/16/checksums.html)

### Create new cluster with page checksums
```bash
➜  ~ sudo mkdir -p /var/lib/postgres-2
➜  ~ sudo chown -R $USER: /var/lib/postgres-2
➜  ~ sudo docker run --name otus-db-pg-2 -e POSTGRES_PASSWORD=postgres -e POSTGRES_USER=postgres -e POSTGRES_INITDB_ARGS="--data-checksums" -d -p 5434:5432 -v /var/lib/postgres-2:/var/lib/postgresql/data postgres:15
```
### Check page checksums
```psql
postgres=# SHOW data_checksums;
 data_checksums
----------------
 on
(1 row)
```
### Create table
```psql
postgres=# create table test_table (id serial primary key, name text);
CREATE TABLE
```
### Insert 3 rows
```psql
postgres=# insert into test_table (name) values ('test1'), ('test2'), ('test3');
INSERT 0 3
```
### Check table file path
```psql
postgres=# select pg_relation_filepath('test_table');
 pg_relation_filepath
----------------------
 base/5/16389
(1 row)
```
### Stop postgres container
```bash
➜  ~ sudo docker stop otus-db-pg-2
otus-db-pg-2
```

### Add a couple of bytes in the table file
```bash
➜  ~ sudo dd if=/dev/urandom of=/var/lib/postgres-2/base/5/16389 bs=1 count=2 seek=0 conv=notrunc
```
*We have changed the first 2 bytes of the table file*
### Start postgres container
```bash
➜  ~ sudo docker start otus-db-pg-2
otus-db-pg-2
```
### Check the table
```psql
postgres=# select * from test_table;
WARNING:  page verification failed, calculated checksum 31494 but expected 11519
ERROR:  invalid page in block 0 of relation base/5/16389
```
*We can see that the checksum verification failed and the table is corrupted*

### Ignore checksum error
```psql
postgres=# alter system set ignore_checksum_failure = on;
ALTER SYSTEM
```
> **ignore_checksum_failure**
>
> Only has effect if data checksums are enabled. Detection of a checksum failure during a read normally causes PostgreSQL to report an error, aborting the current transaction. Setting `ignore_checksum_failure` to on causes the system to ignore the failure (but still report a warning), and continue processing. This behavior may cause crashes, propagate or hide corruption, or other serious problems. However, it may allow you to get past the error and retrieve undamaged tuples that might still be present in the table if the block header is still sane. If the header is corrupt an error will be reported even if this option is enabled. The default setting is off. Only superusers and users with the appropriate SET privilege can change this setting.
>
> See more: [PostgreSQL Documentation](https://www.postgresql.org/docs/current/runtime-config-developer.html)

### Restart postgres container
```bash
➜  ~ sudo docker restart otus-db-pg-2
otus-db-pg-2
```
### Check the table
```psql
postgres=# select * from test_table;
WARNING:  page verification failed, calculated checksum 31494 but expected 11519
 id | name
----+-------
  1 | test1
  2 | test2
  3 | test3
(3 rows)
```
*We can see that the table is corrupted but we can still read the data from the table*

## Conclusion
In this task, we have learned:
- how to configure and monitor checkpoints in PostgreSQL
- how to enable page checksums and how to handle checksum errors
- that async mode can provide better performance but at the cost of data loss in case of a database crash
- page checksums can help to detect data corruption in the database













