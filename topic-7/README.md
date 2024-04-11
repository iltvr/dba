# Домашнее задание

## Механизм блокировок

### Цель:

- Понимание работы механизма блокировок объектов и строк.

### Описание/Пошаговая инструкция выполнения домашнего задания:

1. Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.
2. Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении `pg_locks` и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.
3. Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?
4. Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?

### Задание со звездочкой*

Попробуйте воспроизвести такую ситуацию.

### Критерии оценки:

- Выполнение ДЗ: 10 баллов
- Плюс 2 балла за задание со *
- Плюс 2 балла за красивое решение
- Минус 2 балла за рабочее решение, и недостатки указанные преподавателем не устранены


# Домашняя работа к занятию "10. Блокировки" от 04.03.2024
## 0. Запустим PostgreSQL в Docker контейнере
### Создадим директорию для хранения данных PostgreSQL
```bash
➜  ~ sudo mkdir -p /var/lib/postgres
```
### Сменим владельца директории на текущего пользователя
```bash
➜  ~ sudo chown -R $USER: /var/lib/postgres
```
### Запустим контейнер с PostgreSQL
```bash
➜  ~ sudo docker run --name otus-db-pg-1 -e POSTGRES_PASSWORD=postgres -e POSTGRES_USER=postgres -d -p 5433:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
```
### Подключимся к кластеру PostgreSQL
```bash
➜  ~ psql -h localhost -p 5433 -U postgres -d postgres
Password for user postgres:
psql (15.5 (Homebrew))
Type "help" for help.

postgres=#
```
## 1. Настройка логирования
### Проверим текущие настройки логирования
```psql
postgres=# SHOW log_min_duration_statement;
 log_min_duration_statement
----------------------------
 -1
(1 row)
```
*Текущее значение -1 - логирование отключено*

> `log_min_duration_statement (integer)`
>
> Записывает в журнал продолжительность выполнения всех команд, время работы которых не меньше указанного. Например, при значении 250ms в журнал сервера будут записаны все команды, выполняющиеся 250 миллисекунд и дольше. Значение по умолчанию - -1 (логирование отключено).
>
> См. далее: https://postgrespro.ru/docs/postgrespro/15/runtime-config-logging#GUC-LOG-MIN-DURATION-STATEMENT

### Изменим настройки логирования
```psql
postgres=# ALTER SYSTEM SET log_min_duration_statement = 200;
ALTER SYSTEM
```
### Проверим настройку логирования блокировок
```psql
postgres=# SHOW log_lock_waits;
 log_lock_waits
----------------
 off
(1 row)
```
*Текущее значение off - логирование блокировок отключено*

> `log_lock_waits (boolean)`
>
> Определяет, нужно ли фиксировать в журнале события, когда сеанс ожидает получения блокировки дольше, чем указано в deadlock_timeout. Это позволяет выяснить, не связана ли низкая производительность с ожиданием блокировок. Значение по умолчанию - off.
>
> См. далее: https://postgrespro.ru/docs/postgrespro/15/runtime-config-logging#GUC-LOG-LOCK-WAITS

### Включим логирование блокировок
```psql
postgres=# ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM
```
### Перезагрузим конфигурацию
```psql
postgres=# SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)
```
### Проверим текущие настройки логирования
```psql
postgres=# SHOW log_min_duration_statement;
 log_min_duration_statement
----------------------------
 200ms
(1 row)
```
*Текущее значение 200ms - логирование включено*
### Проверим настройку логирования блокировок
```psql
postgres=# SHOW log_lock_waits;
 log_lock_waits
----------------
 on
(1 row)
```
### Запустим просмотр логов из контейнера в отдельном терминале
```bash
➜  ~ docker logs -tf otus-db-pg-1
```
### Создадим тестовую таблицу accounts
```psql
postgres=# CREATE TABLE accounts (id serial PRIMARY KEY, name text, balance integer NOT NULL);
CREATE TABLE
```
### Вставим тестовые данные
```psql
postgres=# INSERT INTO accounts (name, balance) VALUES ('Alice', 1000), ('Bob', 2000);
INSERT 0 2
```
### Проверим содержимое таблицы accounts
```psql
postgres=# SELECT * FROM accounts;
  id | name  | balance
----+-------+---------
  1 | Alice |    1000
  2 | Bob   |    2000
(2 rows)
```
### Воссоздадим ситуацию блокировки
```psql
postgres=# BEGIN;
BEGIN
postgres=# UPDATE accounts SET balance = balance + 100 WHERE name = 'Alice';
UPDATE 1
```
### В другом терминале выполним запрос на обновление той же строки
```psql
postgres=# BEGIN;
BEGIN
postgres=# UPDATE accounts SET balance = balance + 200 WHERE name = 'Alice';
```
### Коммитим первую транзакцию
```psql
postgres=# COMMIT;
COMMIT
```
### Коммитим вторую транзакцию
```psql
postgres=# COMMIT;
```
### Посмотрим логи в терминале
```log
2024-04-11 19:34:39.599 UTC [135] LOG:  process 135 still waiting for ShareLock on transaction 749 after 1004.316 ms
2024-04-11 19:34:39.599 UTC [135] DETAIL:  Process holding the lock: 70. Wait queue: 135.
2024-04-11 19:34:39.599 UTC [135] CONTEXT:  while updating tuple (0,9) in relation "accounts"
2024-04-11 19:34:39.599 UTC [135] STATEMENT:  UPDATE accounts SET balance = balance + 200 WHERE name = 'Alice';
2024-04-11 19:34:49.199 UTC [135] LOG:  process 135 acquired ShareLock on transaction 749 after 10604.188 ms
2024-04-11 19:34:49.199 UTC [135] CONTEXT:  while updating tuple (0,9) in relation "accounts"
2024-04-11 19:34:49.199 UTC [135] STATEMENT:  UPDATE accounts SET balance = balance + 200 WHERE name = 'Alice';
2024-04-11 19:34:49.199 UTC [135] LOG:  duration: 10605.367 ms  statement: UPDATE accounts SET balance = balance + 200 WHERE name = 'Alice';
```
*В логах видим сообщение о блокировке: `process 135 still waiting for ShareLock on transaction 749 after 1004.316 ms`. А также сообщение о длинной продолжительности выполнения запроса: `duration: 10605.367 ms  statement: UPDATE accounts SET balance = balance + 200 WHERE name = 'Alice'`. Таким образом логирование блокировок и запросов работает корректно*

## 2. Взаимоблокировка
### Настроим утилиту `psql` для вывода pid текущей сессии
```psql
postgres=# \set PROMPT1 '[%p] %/%R%#%x '
```

### Воссоздадим ситуацию взаимоблокировки, в первом терминале (pid=70) выполним запрос на обновление строки
```psql
[70] postgres=# BEGIN;
BEGIN
[70] postgres=# UPDATE accounts SET balance = balance + 100 WHERE name = 'Alice';
UPDATE 1
```
### Во втором терминале (pid=135) выполним запрос на обновление той же строки
```psql
[135] postgres=# BEGIN;
BEGIN
[135] postgres=# UPDATE accounts SET balance = balance + 200 WHERE name = 'Alice';
```
### В третьем терминале (pid=208) выполним запрос на обновление той же строки
```psql
[208] postgres=# BEGIN;
BEGIN
[208] postgres=# UPDATE accounts SET balance = balance + 300 WHERE name = 'Alice';
```
### Получим данные представления `pg_locks`
> `pg_locks` - представление, содержащее информацию о блокировках объектов и строк
>
> См. далее: https://postgrespro.ru/docs/postgrespro/15/view-pg-locks

```psql
postgres=# SELECT relation::REGCLASS, locktype, mode, granted, pid, pg_blocking_pids(pid) AS wait_for, waitstart FROM pg_locks WHERE relation = 'accounts'::regclass;
 relation | locktype |       mode       | granted | pid | wait_for |           waitstart
----------+----------+------------------+---------+-----+----------+-------------------------------
 accounts | relation | RowExclusiveLock | t       | 208 | {135}    |
 accounts | relation | RowExclusiveLock | t       | 135 | {70}     |
 accounts | relation | RowExclusiveLock | t       |  70 | {}       |
 accounts | tuple    | ExclusiveLock    | f       | 208 | {135}    | 2024-04-11 20:31:05.563304+00
 accounts | tuple    | ExclusiveLock    | t       | 135 | {70}     |
(5 rows)
```
*Видим, что сессия с pid=208 ожидает блокировку типа `ExclusiveLock` от сессии с pid=135.*
*Сессия с pid=135 получила блокировку типа `ExclusiveLock` на строке таблицы `accounts`.*
*Вместе с этим все сессии успешно получили и удерживают блокировку типа `RowExclusiveLock` на таблице `accounts`, так-как значение `granted` равно `t` (блокировку в этом режиме получает любая команда, которая изменяет данные в таблице)*

> `mode (text)`
>
> Режим блокировки. Возможные значения: AccessShareLock, RowShareLock, RowExclusiveLock, ShareUpdateExclusiveLock, ShareLock, ShareRowExclusiveLock, AccessExclusiveLock.
>
> См. далее: https://postgrespro.ru/docs/postgrespro/15/view-pg-locks#VIEW-PG-LOCKS-MODE

> `locktype (text)`
>
> Тип блокируемого объекта: relation (отношение), extend (расширение отношения), frozenid (замороженный идентификатор), page (страница), tuple (кортеж), transactionid (идентификатор транзакции), virtualxid (виртуальный идентификатор), spectoken (спекулятивный маркер), object (объект), userlock (пользовательская блокировка) или advisory (рекомендательная)
>
> См. далее: https://postgrespro.ru/docs/postgrespro/15/monitoring-stats#WAIT-EVENT-LOCK-TABLE

> `granted (boolean)`
>
> Признак того, что блокировка была успешно получена. Значение t означает, что блокировка была успешно получена, а значение f означает, что блокировка ожидается.

> `pg_blocking_pids ( integer ) → integer[]`
>
> Возвращает массив идентификаторов сеансов, которые блокируют сеанс с указанным идентификатором. Если сеанс не заблокирован, возвращается пустой массив. Один серверный процесс блокирует другой, если он либо удерживает блокировку, конфликтующую с блокировкой, запрашиваемой вторым (жёсткая блокировка), либо ожидает блокировку, которая вызвала бы конфликт с запросом блокировки заблокированного процесса и находится перед ней в очереди ожидания (мягкая блокировка).
>
> См. далее: https://postgrespro.ru/docs/postgrespro/15/functions-info

> `SHARE (ShareLock) `
>
> Блокировка `ShareLock` позволяет другим сессиям читать данные параллельного, но не позволяет им изменять данные. Пока транзакция "держит" блокировку `ShareLock`, другие транзакции не могут получить конфликтующие блокировки, такие как ROW EXCLUSIVE, SHARE UPDATE EXCLUSIVE, SHARE ROW EXCLUSIVE, EXCLUSIVE и ACCESS EXCLUSIVE.
>
> См. далее: https://postgrespro.ru/docs/postgrespro/15/explicit-locking#LOCKING-TABLES

> `EXCLUSIVE (ExclusiveLock)`
>
> Конфликтует с режимами блокировки ROW SHARE, ROW EXCLUSIVE, SHARE UPDATE EXCLUSIVE, SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE и ACCESS EXCLUSIVE. Этот режим совместим только с блокировкой ACCESS SHARE, то есть параллельно с транзакцией, получившей блокировку в этом режиме, допускается только чтение таблицы.
>
> См. далее: https://postgrespro.ru/docs/postgrespro/15/explicit-locking#LOCKING-TABLES

> `ROW EXCLUSIVE (RowExclusiveLock)`
>
> Конфликтует с режимами блокировки SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE и ACCESS EXCLUSIVE.
>
> Команды UPDATE, DELETE, INSERT и MERGE получают такую блокировку для целевой таблицы (в дополнение к блокировкам ACCESS SHARE для всех других задействованных таблиц). Вообще говоря, блокировку в этом режиме получает любая команда, которая изменяет данные в таблице.
>
> См. далее:https://postgrespro.ru/docs/postgrespro/15/explicit-locking#LOCKING-TABLES

### Посмотрим логи в терминале
```log
2024-04-11 20:31:01.772 UTC [135] LOG:  process 135 still waiting for ShareLock on transaction 762 after 1004.906 ms
2024-04-11 20:31:01.772 UTC [135] DETAIL:  Process holding the lock: 70. Wait queue: 135.
2024-04-11 20:31:01.772 UTC [135] CONTEXT:  while updating tuple (0,22) in relation "accounts"
2024-04-11 20:31:01.772 UTC [135] STATEMENT:  UPDATE accounts SET balance = balance + 200 WHERE name = 'Alice';
2024-04-11 20:31:06.564 UTC [208] LOG:  process 208 still waiting for ExclusiveLock on tuple (0,22) of relation 16400 of database 5 after 1001.379 ms
2024-04-11 20:31:06.564 UTC [208] DETAIL:  Process holding the lock: 135. Wait queue: 208.
2024-04-11 20:31:06.564 UTC [208] STATEMENT:  UPDATE accounts SET balance = balance + 300 WHERE name = 'Alice';
```
*В логах видим сообщения о блокировках строк: `process 135 still waiting for ShareLock on transaction 762 after 1004.906 ms` и `process 208 still waiting for ExclusiveLock on tuple (0,22) of relation 16400 of database 5 after 1001.379 ms`. Сессии с pid=135 и pid=208 ожидают блокировку строки от сессий с pid=70 и pid=135 соответственно.*

> ВОПРОС:
>
> почему в представлении `pg_locks` отсутствует информация о ShareLock для сессии с pid=135?

### Выполним коммит в сессии с pid=70
```psql
[70] postgres=# COMMIT;
COMMIT
```
### Посмотрим состояние блокировок
```psql
postgres=# SELECT relation::REGCLASS, locktype, mode, granted, pid, pg_blocking_pids(pid) AS wait_for, waitstart FROM pg_locks WHERE relation = 'accounts'::regclass;
 relation | locktype |       mode       | granted | pid | wait_for | waitstart
----------+----------+------------------+---------+-----+----------+-----------
 accounts | relation | RowExclusiveLock | t       | 208 | {135}    |
 accounts | relation | RowExclusiveLock | t       | 135 | {}       |
(2 rows)
```
*Видим, что количество блокировок уменьшилось, сессия с pid=135 больше не ожидает блокировку от сессии с pid=70*
### Посмотрим логи в терминале
```log
2024-04-11 20:53:22.547 UTC [135] LOG:  process 135 acquired ShareLock on transaction 762 after 1341779.632 ms
2024-04-11 20:53:22.547 UTC [135] CONTEXT:  while updating tuple (0,22) in relation "accounts"
2024-04-11 20:53:22.547 UTC [135] STATEMENT:  UPDATE accounts SET balance = balance + 200 WHERE name = 'Alice';
2024-04-11 20:53:22.548 UTC [208] LOG:  process 208 acquired ExclusiveLock on tuple (0,22) of relation 16400 of database 5 after 1336984.690 ms
2024-04-11 20:53:22.548 UTC [208] STATEMENT:  UPDATE accounts SET balance = balance + 300 WHERE name = 'Alice';
2024-04-11 20:53:22.548 UTC [135] LOG:  duration: 1341781.431 ms  statement: UPDATE accounts SET balance = balance + 200 WHERE name = 'Alice';
2024-04-11 20:53:23.549 UTC [208] LOG:  process 208 still waiting for ShareLock on transaction 763 after 1001.689 ms
2024-04-11 20:53:23.549 UTC [208] DETAIL:  Process holding the lock: 135. Wait queue: 208.
2024-04-11 20:53:23.549 UTC [208] CONTEXT:  while rechecking updated tuple (0,23) in relation "accounts"
2024-04-11 20:53:23.549 UTC [208] STATEMENT:  UPDATE accounts SET balance = balance + 300 WHERE name = 'Alice';
```
*В логах видим сообщение: `process 135 acquired ShareLock on transaction 762 after 1341779.632 ms` - сессия с pid=135 успешно получила блокировку ShareLock. В то же время сессия с pid=208 ожидает блокировку ShareLock от сессии с pid=135: `process 208 still waiting for ShareLock on transaction 763 after 1001.689 ms`*

### Выполним коммит в сессии с pid=135
```psql
[135] postgres=# COMMIT;
COMMIT
```
### Посмотрим состояние блокировок
```psql
postgres=# SELECT relation::REGCLASS, locktype, mode, granted, pid, pg_blocking_pids(pid) AS wait_for, waitstart FROM pg_locks WHERE relation = 'accounts'::regclass;
 relation | locktype |       mode       | granted | pid | wait_for | waitstart
----------+----------+------------------+---------+-----+----------+-----------
 accounts | relation | RowExclusiveLock | t       | 208 | {}       |
(1 row)
```
*Видим, что количество блокировок уменьшилось, сессия с pid=208 больше не ожидает блокировку от других сессий*
### Посмотрим логи в терминале
```log
2024-04-11 21:21:07.676 UTC [208] LOG:  process 208 acquired ShareLock on transaction 763 after 1665128.763 ms
2024-04-11 21:21:07.676 UTC [208] CONTEXT:  while rechecking updated tuple (0,23) in relation "accounts"
2024-04-11 21:21:07.676 UTC [208] STATEMENT:  UPDATE accounts SET balance = balance + 300 WHERE name = 'Alice';
2024-04-11 21:21:07.678 UTC [208] LOG:  duration: 3002116.099 ms  statement: UPDATE accounts SET balance = balance + 300 WHERE name = 'Alice';
```
*В логах видим сообщение: `process 208 acquired ShareLock on transaction 763 after 1665128.763 ms` - сессия с pid=208 успешно получила блокировку ShareLock*

### Выполним коммит в сессии с pid=208
```psql
[208] postgres=# COMMIT;
COMMIT
```
### Посмотрим состояние блокировок
```psql
postgres=# SELECT relation::REGCLASS, locktype, mode, granted, pid, pg_blocking_pids(pid) AS wait_for, waitstart FROM pg_locks WHERE relation = 'accounts'::regclass;
 relation | locktype | mode | granted | pid | wait_for | waitstart
----------+----------+------+---------+-----+----------+-----------
(0 rows)
```
*Видим, что блокировок больше нет*
