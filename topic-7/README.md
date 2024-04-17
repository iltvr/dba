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
*Видим, что сессия с pid=208 ожидает исключительную блокировку версии строки типа `ExclusiveLock` от сессии с pid=135.*
*Сессия с pid=135 получила исключительную блокировку версии строки типа `ExclusiveLock`.*
*Вместе с этим все сессии успешно получили и удерживают блокировку типа `RowExclusiveLock` на таблице `accounts`, так-как значение `granted` равно `t` (блокировку в этом режиме получает любая команда, которая изменяет данные в таблице)*

**Когда транзакция собирается изменить строку, она выполняет следующую последовательность действий:**
1. Захватывает исключительную блокировку изменяемой версии строки (tuple).
2. Если xmax и информационные биты говорят о том, что строка заблокирована, то запрашивает блокировку номера транзакции xmax.
3. Прописывает свой xmax и необходимые информационные биты.
4. Освобождает блокировку версии строки.

> `mode (text)`
>
> Режим блокировки. Возможные значения: AccessShareLock, RowShareLock, RowExclusiveLock, ShareUpdateExclusiveLock, ShareLock, ShareRowExclusiveLock, AccessExclusiveLock.
>
> См. далее: https://postgrespro.ru/docs/postgrespro/15/view-pg-locks#VIEW-PG-LOCKS-MODE

> `locktype (text)`
>
> Тип блокируемого объекта: `relation` (ожидание при запросе блокировки для отношения), `tuple` (ожидание при запросе блокировки для кортежа).
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

## 3. Взаимная блокировка трех транзакций
### Проверим `deadlock_timeout`
```psql
postgres=# SHOW deadlock_timeout;
 deadlock_timeout
------------------
 1s
(1 row)
```
> `deadlock_timeout (integer)`
>
> Время в миллисекундах, в течение которого сервер будет ожидать разрешения взаимоблокировки. Если взаимоблокировка не разрешится в течение этого времени, сервер прервет одну из транзакций, чтобы разрешить взаимоблокировку. Значение по умолчанию - 1 секунда.
>
> См. далее: https://postgrespro.ru/docs/postgrespro/15/runtime-config-client#GUC-DEADLOCK-TIMEOUT

### Воссоздадим ситуацию взаимоблокировки трех транзакций. Добавим данные в таблицу `accounts`
```psql
postgres=# INSERT INTO accounts (name, balance) VALUES ('Alice', 1000), ('Bob', 2000), ('Charlie', 3000);
INSERT 0 3
```
### Проверим содержимое таблицы accounts
```psql
postgres=# select * from accounts;
 id |  name   | balance
----+---------+---------
  4 | Alice   |    1000
  5 | Bob     |    2000
  6 | Charlie |    3000
(3 rows)
```

### В первом терминале (pid=70) будем переводить деньги от Alice к Bob. Уменьшим баланс Alice на 100
```psql
[70] postgres=# BEGIN;
BEGIN
[70] postgres=#* UPDATE accounts SET balance = balance - 100 WHERE name = 'Alice';
UPDATE 1
```
### Во втором терминале (pid=135) будем переводить деньги от Bob к Charlie. Уменьшим баланс Bob на 200
```psql
[135] postgres=# BEGIN;
BEGIN
[135] postgres=#* UPDATE accounts SET balance = balance - 200 WHERE name = 'Bob';
UPDATE 1
```
### В третьем терминале (pid=208) будем переводить деньги от Charlie к Alice. Уменьшим баланс Charlie на 300
```psql
[208] postgres=# BEGIN;
BEGIN
[208] postgres=#* UPDATE accounts SET balance = balance - 300 WHERE name = 'Charlie';
UPDATE 1
```
### В первом терминале (pid=70) увеличим баланс Bob на 100
```psql
[70] postgres=#* UPDATE accounts SET balance = balance + 100 WHERE name = 'Bob';
```
### Во втором терминале (pid=135) увеличим баланс Charlie на 200
```psql
[135] postgres=#* UPDATE accounts SET balance = balance + 200 WHERE name = 'Charlie';
```
### В третьем терминале (pid=208) увеличим баланс Alice на 300
```psql
[208] postgres=#* UPDATE accounts SET balance = balance + 300 WHERE name = 'Alice';
ERROR:  deadlock detected
DETAIL:  Process 208 waits for ShareLock on transaction 775; blocked by process 70.
Process 70 waits for ShareLock on transaction 776; blocked by process 135.
Process 135 waits for ShareLock on transaction 777; blocked by process 208.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "accounts"
[208] postgres=#! COMMIT;
ROLLBACK
```
### Коммитим транзакции в сессиях с pid=70 и pid=135
```psql
[70] postgres=# COMMIT;
COMMIT
```
```psql
[135] postgres=# COMMIT;
COMMIT
```
### Посмотрим состояние данных в таблице accounts
```psql
postgres=#  select * from accounts;
 id |  name   | balance
----+---------+---------
  4 | Alice   |     900
  6 | Charlie |    3200
  5 | Bob     |    1900
(3 rows)
```
*Благодаря тому, что перевод от Charlie к Alice происходил в рамках одной транзакции, данные в таблице accounts остались целостными*

### Изучим логи в терминале
```log
2024-04-16T21:17:44.251190805Z 2024-04-16 21:17:44.249 UTC [70] LOG:  process 70 still waiting for ShareLock on transaction 776 after 1006.040 ms
2024-04-16T21:17:44.252650597Z 2024-04-16 21:17:44.249 UTC [70] DETAIL:  Process holding the lock: 135. Wait queue: 70.
2024-04-16T21:17:44.252687180Z 2024-04-16 21:17:44.249 UTC [70] CONTEXT:  while updating tuple (0,2) in relation "accounts"
2024-04-16T21:17:44.252693972Z 2024-04-16 21:17:44.249 UTC [70] STATEMENT:  UPDATE accounts SET balance = balance + 100 WHERE name = 'Bob';
```
*В логах видим сообщение о взаимоблокировке: `process 70 still waiting for ShareLock on transaction 776 after 1006.040 ms`. Сессия с pid=70 ожидает блокировку ShareLock от сессии с pid=135. Это произошло в момент выполнения запроса на увеличение баланса Bob на 100*
```log
2024-04-16T21:18:17.152542042Z 2024-04-16 21:18:17.149 UTC [135] LOG:  process 135 still waiting for ShareLock on transaction 777 after 1001.348 ms
2024-04-16T21:18:17.153021667Z 2024-04-16 21:18:17.149 UTC [135] DETAIL:  Process holding the lock: 208. Wait queue: 135.
2024-04-16T21:18:17.153036750Z 2024-04-16 21:18:17.149 UTC [135] CONTEXT:  while updating tuple (0,3) in relation "accounts"
2024-04-16T21:18:17.153042667Z 2024-04-16 21:18:17.149 UTC [135] STATEMENT:  UPDATE accounts SET balance = balance + 200 WHERE name = 'Charlie';
```
*Далее логах видим сообщение о взаимоблокировке: `process 135 still waiting for ShareLock on transaction 777 after 1001.348 ms`. Сессия с pid=135 ожидает блокировку ShareLock от сессии с pid=208. Это произошло в момент выполнения запроса на увеличение баланса Charlie на 200*
```log
2024-04-16T2:51.801376586Z 2024-04-16 21:18:51.799 UTC [208] LOG:  process 208 detected deadlock while waiting for ShareLock on transaction 775 after 1002.675 ms
 2024-04-16 21:18:51.799 UTC [208] DETAIL:  Process holding the lock: 70. Wait queue: .
 2024-04-16 21:18:51.799 UTC [208] CONTEXT:  while updating tuple (0,1) in relation "accounts"
 2024-04-16 21:18:51.799 UTC [208] STATEMENT:  UPDATE accounts SET balance = balance + 300 WHERE name = 'Alice';
 2024-04-16 21:18:51.800 UTC [208] ERROR:  deadlock detected
 2024-04-16 21:18:51.800 UTC [208] DETAIL:  Process 208 waits for ShareLock on transaction 775; blocked by process 70.
 	Process 70 waits for ShareLock on transaction 776; blocked by process 135.
 	Process 135 waits for ShareLock on transaction 777; blocked by process 208.
 	Process 208: UPDATE accounts SET balance = balance + 300 WHERE name = 'Alice';
 	Process 70: UPDATE accounts SET balance = balance + 100 WHERE name = 'Bob';
 	Process 135: UPDATE accounts SET balance = balance + 200 WHERE name = 'Charlie';
 2024-04-16 21:18:51.800 UTC [208] HINT:  See server log for query details.
 2024-04-16 21:18:51.800 UTC [208] CONTEXT:  while updating tuple (0,1) in relation "accounts"
 2024-04-16 21:18:51.800 UTC [208] STATEMENT:  UPDATE accounts SET balance = balance + 300 WHERE name = 'Alice';
 2024-04-16 21:18:51.801 UTC [135] LOG:  process 135 acquired ShareLock on transaction 777 after 35653.419 ms
 2024-04-16 21:18:51.801 UTC [135] CONTEXT:  while updating tuple (0,3) in relation "accounts"
 2024-04-16 21:18:51.801 UTC [135] STATEMENT:  UPDATE accounts SET balance = balance + 200 WHERE name = 'Charlie';
 2024-04-16 21:18:51.801 UTC [135] LOG:  duration: 35658.221 ms  statement: UPDATE accounts SET balance = balance + 200 WHERE name = 'Charlie';
```
*После возникновения Deadlock в третьем терминале с pid=208, сессия была прервана. В логах видим сообщение: `process 208 detected deadlock while waiting for ShareLock on transaction 775 after 1002.675 ms`.*

### Проверим статистику `pg_stat_database`
```psql
postgres=# SELECT * FROM pg_stat_database WHERE datname = 'postgres';
-[ RECORD 1 ]------------+---------------
datid                    | 5
datname                  | postgres
numbackends              | 4
xact_commit              | 14788
xact_rollback            | 13
blks_read                | 363
blks_hit                 | 543483
tup_returned             | 6513597
tup_fetched              | 105987
tup_inserted             | 138
tup_updated              | 48
tup_deleted              | 54
conflicts                | 0
temp_files               | 0
temp_bytes               | 0
deadlocks                | 1
checksum_failures        |
checksum_last_failure    |
blk_read_time            | 0
blk_write_time           | 0
session_time             | 1755927843.626
active_time              | 5582267.269
idle_in_transaction_time | 5410610.177
sessions                 | 5
sessions_abandoned       | 0
sessions_fatal           | 0
sessions_killed          | 0
stats_reset              |
```
*Видим, что количество `deadlocks` равно 1. Это означает, что за время работы сервера произошла одна взаимоблокировка которая и была зарегистрирована в журнале сервера*

## 4. Поиск блокировок при выполнении UPDATE запроса без WHERE условия
### Воссоздадим ситуацию блокировки при выполнении UPDATE запроса без WHERE условия

Обычно для выполнения операций которые могут привести к взаимоблокировке, необходимо выполнять блокировки в одном и том же порядке. Например в таблице `accounts` блокировать строки в порядке увеличения `id`.

Поскольку UPDATE запрашивает блокировку последовательно для каждой строки, то если строки блокируются в одном порядке - взаимоблокировки не возникнут.

Следовательно, чтобы воспроизвести взаимоблокировку, нам необходимо воссоздать ситуацию, когда одна транзакция обновляет все строки в таблице в одном порядке, а другая транзакция обновляет все строки в таблице в обратном порядке.

### Установим расширение `pgrowlocks`
```psql
postgres=# CREATE EXTENSION pgrowlocks;
CREATE EXTENSION
```
### Создадим таблицу `accounts` и добавим в нее 3 строки
```psql
postgres=# CREATE TABLE accounts (id serial PRIMARY KEY, name text, balance integer);
CREATE TABLE
postgres=# INSERT INTO accounts (name, balance) VALUES ('Alice', 1000), ('Bob', 2000), ('Charlie', 3000);
INSERT 0 3
```
### Проверим содержимое таблицы `accounts`
```psql
postgres=# SELECT * FROM accounts;
 id |  name   | balance
----+---------+---------
  7 | Alice   |    1000
  8 | Bob     |    2000
  9 | Charlie |    3000
(3 rows)
```
### Проверим план выполнения запроса
```psql
postgres=# EXPLAIN (ANALYZE, BUFFERS) UPDATE accounts SET balance = balance + 1;
                                                 QUERY PLAN
-------------------------------------------------------------------------------------------------------------
 Update on accounts  (cost=0.00..25.00 rows=0 width=0) (actual time=0.341..0.343 rows=0 loops=1)
   Buffers: shared hit=7
   ->  Seq Scan on accounts  (cost=0.00..25.00 rows=1200 width=10) (actual time=0.138..0.142 rows=3 loops=1)
         Buffers: shared hit=1
 Planning Time: 0.551 ms
 Execution Time: 1.003 ms
(6 rows)
```
*Видим, что план выполнения запроса предполагает последовательное обновление всех строк в таблице `accounts`. Так-как строки таблицы вставлялись в порядке увеличения `id`, то блокировки строк будут запрашиваться в порядке увеличения `id`*

### Для изменения порядка выполнения запроса будем использовать механизм DECLARE cursor_name CURSOR.
В целях отладки для каждой сессии будем увеличивать таймаут проверки взаимоблокировок командой
```sql
SET LOCAL deadlock_timeout = '30s';
```
### В первой сессии (pid=70) добавим курсор для таблицы `accounts` в порядке увеличения `id`
```psql
[70] postgres=# BEGIN;
BEGIN
[70] postgres=#* SET LOCAL deadlock_timeout = '30s';
SET
[70] postgres=#* DECLARE cur CURSOR FOR SELECT * FROM accounts ORDER BY id FOR UPDATE;
DECLARE CURSOR
```
### Во второй сессии (pid=135) добавим курсор для таблицы `accounts` в порядке уменьшения `id`
```psql
[135] postgres=# BEGIN;
BEGIN
[135] postgres=#* SET LOCAL deadlock_timeout = '30s';
SET
[135] postgres=#* DECLARE cur CURSOR FOR SELECT * FROM accounts ORDER BY id DESC FOR UPDATE;
DECLARE CURSOR
```
### В первой сессии (pid=70) продвинем курсор на одну строку
```psql
[70] postgres=#* FETCH cur;
 id | name  | balance
----+-------+---------
  7 | Alice |    1004
(1 row)
```
### Во второй сессии (pid=135) продвинем курсор на одну строку
```psql
[135] postgres=#* FETCH cur;
 id |  name   | balance
----+---------+---------
  9 | Charlie |    3004
(1 row)
```
### В первой сессии (pid=70) продвинем курсор на следующую строку
```psql
[70] postgres=#* FETCH cur;
 id | name | balance
----+------+---------
  8 | Bob  |    2004
(1 row)
```
### Во второй сессии (pid=135) продвинем курсор на следующую строку
```psql
[135] postgres=#* FETCH cur;
-- ожидание блокировки --
```
*Видим, что сессия с pid=135 ожидает блокировку строки (name='Bob') от сессии с pid=70*
### Посмотрим состояние блокировок
```psql
postgres=# SELECT * FROM pgrowlocks('accounts') \gx
-[ RECORD 1 ]--------------
locked_row | (0,13)
locker     | 810
multi      | f
xids       | {810}
modes      | {"For Update"}
pids       | {70}
-[ RECORD 2 ]--------------
locked_row | (0,14)
locker     | 810
multi      | f
xids       | {810}
modes      | {"For Update"}
pids       | {70}
-[ RECORD 3 ]--------------
locked_row | (0,15)
locker     | 811
multi      | f
xids       | {811}
modes      | {"For Update"}
pids       | {135}
```
### В первой сессии (pid=70) продвинем курсор на следующую строку, чтобы вызвать взаимоблокировку.
```psql
[70] postgres=#* FETCH cur;
-- ожидание блокировки --
```
*Видим, что возникло ожидание блокировки строки (name='Charlie') от сессии с pid=135. Тем самым, в обеих транзакциях возникли взаимные ожидания блокировок.*

### Посмотрим состояние блокировок
```psql
postgres=# SELECT * FROM pgrowlocks('accounts') \gx
-[ RECORD 1 ]--------------
locked_row | (0,13)
locker     | 810
multi      | f
xids       | {810}
modes      | {"For Update"}
pids       | {70}
-[ RECORD 2 ]--------------
locked_row | (0,14)
locker     | 810
multi      | f
xids       | {810}
modes      | {"For Update"}
pids       | {70}
-[ RECORD 3 ]--------------
locked_row | (0,15)
locker     | 811
multi      | f
xids       | {811}
modes      | {"For Update"}
pids       | {135}
```
*Ничего не изменилось, запись о том, что сессия с pid=70 ожидает блокировку строки от сессии с pid=135 отсутствует*
### В первой сессии (pid=70) через 30 секунд видим сообщение о Deadlock
```psql
ERROR:  deadlock detected
DETAIL:  Process 70 waits for ShareLock on transaction 811; blocked by process 135.
Process 135 waits for ShareLock on transaction 810; blocked by process 70.
HINT:  See server log for query details.
CONTEXT:  while locking tuple (0,15) in relation "accounts"
```
### Проверим состояние блокировок
```psql
postgres=# SELECT * FROM pgrowlocks('accounts') \gx
-[ RECORD 1 ]--------------
locked_row | (0,14)
locker     | 811
multi      | f
xids       | {811}
modes      | {"For Update"}
pids       | {135}
-[ RECORD 2 ]--------------
locked_row | (0,15)
locker     | 811
multi      | f
xids       | {811}
modes      | {"For Update"}
pids       | {135}
```
### Коммитим транзакции в сессиях с pid=70 и pid=135
```psql
[70] postgres=#! COMMIT;
ROLLBACK
```
```psql
[135] postgres=#* COMMIT;
COMMIT
```
### Посмотрим состояние блокировок
```psql
postgres=# SELECT * FROM pgrowlocks('accounts') \gx
(0 rows)
```
*Видим, что блокировок больше нет*

### Изучим логи в терминале
```log
2024-04-17T06:49:26.158525929Z 2024-04-17 06:49:26.155 UTC [135] LOG:  process 135 still waiting for ShareLock on transaction 810 after 30003.154 ms
2024-04-17T06:49:26.159263304Z 2024-04-17 06:49:26.155 UTC [135] DETAIL:  Process holding the lock: 70. Wait queue: 135.
2024-04-17T06:49:26.159270971Z 2024-04-17 06:49:26.155 UTC [135] CONTEXT:  while locking tuple (0,14) in relation "accounts"
2024-04-17T06:49:26.159273388Z 2024-04-17 06:49:26.155 UTC [135] STATEMENT:  FETCH cur;
2024-04-17T06:51:19.496209718Z 2024-04-17 06:51:19.494 UTC [70] LOG:  process 70 detected deadlock while waiting for ShareLock on transaction 811 after 30006.983 ms
2024-04-17T06:51:19.496823634Z 2024-04-17 06:51:19.494 UTC [70] DETAIL:  Process holding the lock: 135. Wait queue: .
2024-04-17T06:51:19.496838634Z 2024-04-17 06:51:19.494 UTC [70] CONTEXT:  while locking tuple (0,15) in relation "accounts"
2024-04-17T06:51:19.496841384Z 2024-04-17 06:51:19.494 UTC [70] STATEMENT:  FETCH cur;
2024-04-17T06:51:19.496843301Z 2024-04-17 06:51:19.494 UTC [70] ERROR:  deadlock detected
2024-04-17T06:51:19.496845218Z 2024-04-17 06:51:19.494 UTC [70] DETAIL:  Process 70 waits for ShareLock on transaction 811; blocked by process 135.
2024-04-17T06:51:19.496847093Z 	Process 135 waits for ShareLock on transaction 810; blocked by process 70.
2024-04-17T06:51:19.496868634Z 	Process 70: FETCH cur;
2024-04-17T06:51:19.496870468Z 	Process 135: FETCH cur;
2024-04-17T06:51:19.496872176Z 2024-04-17 06:51:19.494 UTC [70] HINT:  See server log for query details.
2024-04-17T06:51:19.496874009Z 2024-04-17 06:51:19.494 UTC [70] CONTEXT:  while locking tuple (0,15) in relation "accounts"
2024-04-17T06:51:19.496876593Z 2024-04-17 06:51:19.494 UTC [70] STATEMENT:  FETCH cur;
2024-04-17T06:51:19.496878551Z 2024-04-17 06:51:19.495 UTC [135] LOG:  process 135 acquired ShareLock on transaction 810 after 143343.360 ms
2024-04-17T06:51:19.496880384Z 2024-04-17 06:51:19.495 UTC [135] CONTEXT:  while locking tuple (0,14) in relation "accounts"
2024-04-17T06:51:19.496882259Z 2024-04-17 06:51:19.495 UTC [135] STATEMENT:  FETCH cur;
2024-04-17T06:51:19.497865884Z 2024-04-17 06:51:19.496 UTC [135] LOG:  duration: 143344.566 ms  statement: FETCH cur;
```
*В логах видим сообщения о взаимоблокировке: `process 135 still waiting for ShareLock on transaction 810 after 30003.154 ms` и `process 70 detected deadlock while waiting for ShareLock on transaction 811 after 30006.983 ms`*

## Итоги
В ходе выполнения домашнего задания мы:
 - научились настраивать параметры журнала сервера для отслеживания длинных транзакций
 - изучили механизмы блокировок в PostgreSQL
 - изучили механизмы поиска блокировок в PostgreSQL с использованием представления `pg_locks` и расширения `pgrowlocks`
 - рассмотрели примеры взаимоблокировок
 - рассмотрели примеры записей журнала сервера при возникновении взаимоблокировок
 - рассмотрели примеры взаимоблокировок при выполнении UPDATE запроса без WHERE условия
