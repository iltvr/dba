


# Домашнее задание

## Работа с уровнями изоляции транзакции в PostgreSQL

### Цель:

- научиться работать с Google Cloud Platform на уровне Google Compute Engine (IaaS)
- научиться управлять уровнем изоляции транзации в PostgreSQL и понимать особенность работы уровней read committed и repeatable read

### Описание/Пошаговая инструкция выполнения домашнего задания:

1. Создать новый проект в Google Cloud Platform, Яндекс облако или на любых ВМ, докере
2. Далее создать инстанс виртуальной машины с дефолтными параметрами
3. Добавить свой ssh ключ в metadata ВМ
4. Зайти удаленным ssh (первая сессия), не забывайте про ssh-add
5. Поставить PostgreSQL
6. Зайти вторым ssh (вторая сессия)
7. Запустить везде psql из под пользователя postgres
8. Выключить auto commit
9. Сделать:

    В первой сессии новую таблицу и наполнить ее данными
    ```sql
    begin;
    create table persons(id serial, first_name text, second_name text);
    insert into persons(first_name, second_name) values('ivan', 'ivanov');
    insert into persons(first_name, second_name) values('petr', 'petrov');
    commit;
    ```

    Посмотреть текущий уровень изоляции: `show transaction isolation level`

    Начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции

    В первой сессии добавить новую запись `insert into persons(first_name, second_name) values('sergey', 'sergeev');`

    Сделать `select * from persons` во второй сессии

    Видите ли вы новую запись и если да, то почему?

    Завершить первую транзакцию - `commit;`

    Сделать `select * from persons` во второй сессии

    Видите ли вы новую запись и если да, то почему?

    Завершить транзакцию во второй сессии

    Начать новые, но уже repeatable read транзации - `set transaction isolation level repeatable read;`

    В первой сессии добавить новую запись `insert into persons(first_name, second_name) values('sveta', 'svetova');`

    Сделать `select * from persons` во второй сессии

    Видите ли вы новую запись и если да, то почему?

    Завершить первую транзакцию - `commit;`

    Сделать `select * from persons` во второй сессии

    Видите ли вы новую запись и если да, то почему?

    Завершить вторую транзакцию

    Сделать `select * from persons` во второй сессии

    Видите ли вы новую запись и если да, то почему?

10. ДЗ сдаем в виде миниотчета в markdown в гите

### Критерии оценки:

- Выполнение ДЗ: 10 баллов
- Плюс 2 балла за красивое решение
- Минус 2 балла за рабочее решение, и недостатки указанные преподавателем не устранены

# Домашняя работа к занятию "SQL и Реляционные СУБД. Введение в PostgreSQL" oт 05.02.2024

## Init Yandex Cloud
### Add VM otus-db-pg-1 using the web interface
### Configure yc cli locally
```bash
$ yc init
```
```bash
$ yc config list
```
```bash
  token: y0_AgAAAAAWGijRAATuwQAA**************************************
  cloud-id: b1g2rj6jeai*********
  folder-id: b1gffa7irkl*********
  compute-default-zone: ru-central1-d
```

### Add otus-db-pg-1 to .ssh/config
```
Host otus-db-pg-1
  HostName: 158.160.xxx.xxx
  User user
  IdentityFile ~/.ssh/yc_key
```

### Connect to the `otus-db-pg-1` VM
```bash
$ ssh otus-db-pg-1
```
## Yandex Cloud. Install docker for ubuntu.
  > **See: https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository**
```bash
$ sudo docker run hello-world
```
```bash
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
c1ec31eb5944: Pull complete
Digest: sha256:4bd78111b6914a99dbc560e6a20eab57ff6655aea4a80c50b0c5491968cbc2e6
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.
```
## Yandex Cloud. Run PostgreSQL in a Docker container
### Create a directory `/var/lib/postgres`
```bash
$ sudo mkdir /var/lib/postgres
```
### Run a PostgreSQL container
```bash
$ sudo docker run --name otus-db-pg-1 -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
```
### Check the container status
```bash
$ sudo docker ps
```
```
CONTAINER ID   IMAGE         COMMAND                  CREATED         STATUS         PORTS                    NAMES
85d62aaa55a8   postgres:15   "docker-entrypoint.s…"   6 seconds ago   Up 5 seconds   0.0.0.0:5432->5432/tcp   otus-db-pg-1
```
### Connect to the PostgreSQL container
```bash
$ sudo docker exec -it otus-db-pg-1 psql -U postgres
```

## Add data
```sql
    begin;
    create table persons(id serial, first_name text, second_name text);
    insert into persons(first_name, second_name) values('ivan', 'ivanov');
    insert into persons(first_name, second_name) values('petr', 'petrov');
    commit;
```

## Perform transaction with read committed isolation level
### Client 1. Check the current isolation level
```sql
BEGIN;
show transaction isolation level;
```
```
 transaction_isolation
-----------------------
 read committed

```
### Client 2. Check the current isolation level
```sql
BEGIN;
show transaction isolation level;
```
```
 transaction_isolation
-----------------------
 read committed
```
### Client 1. Insert new record
```sql
insert into persons(first_name, second_name) values('sergey', 'sergeev');
```

### Client 2. Select data
```sql
select * from persons;
```
```
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```
### Client 1. Commit transaction
```sql
commit;
```
### Client 2. Select data
```sql
select * from persons;
```
```
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
Now we see the new record in the table. It is because the `read committed` isolation level allows to see the changes made by other transactions after the current transaction has started.
### Client 2. Commit transaction
```sql
commit;
```

## Perform transaction with repeatable read isolation level
## Add extension `pageinspect` to the database
```sql
create extension pageinspect;
```
### Recreate the table `persons` and insert initial data
```sql
drop table persons;
begin;
create table persons(id serial, first_name text, second_name text);
insert into persons(first_name, second_name) values('ivan', 'ivanov');
insert into persons(first_name, second_name) values('petr', 'petrov');
commit;
```
### Inspect row versioning
```sql
select lp, t_ctid, t_xmin, t_xmax, raw_flags
  from heap_page_items(get_raw_page('persons', 0)),
    lateral heap_tuple_infomask_flags(t_infomask, t_infomask2);
```
```
  lp | t_ctid | t_xmin | t_xmax |              raw_flags
----+--------+--------+--------+--------------------------------------
  1 | (0,1)  | 735081 |      0 | {HEAP_HASVARWIDTH,HEAP_XMAX_INVALID}
  2 | (0,2)  | 735081 |      0 | {HEAP_HASVARWIDTH,HEAP_XMAX_INVALID}
(2 rows)
```
- `lp` - location pointer
- `t_ctid` - tuple identifier, e.g. (0,1) means the row is located on the first page and has the first item number
- `t_xmin` - transaction id that inserted the row, 735081 is the transaction id of the last transaction that inserted the rows into the table
- `t_xmax` - transaction id that deleted the row
- `raw_flags` - flags that indicate the row state
- `HEAP_XMAX_INVALID` - the row is not deleted
- `HEAP_HASVARWIDTH` - This flag is set in the tuple header to signify that the tuple contains at least one variable-width attribute. E.g, `text`, `varchar`, and `bytea`.
### Client 1. Start transaction. Set repeatable read isolation level
```sql
begin;
set transaction isolation level repeatable read;
select txid_current();
```
```
  txid_current
--------------
       735082
(1 row)
```
### Client 2. Start transaction. Set repeatable read isolation level
```sql
begin;
set transaction isolation level repeatable read;
select txid_current();
```
```
  txid_current
--------------
       735083
```
### Client 1. Insert new record
```sql
insert into persons(first_name, second_name) values('sveta', 'svetova');
```
### Inspect row versioning
```
 lp | t_ctid | t_xmin | t_xmax |              raw_flags
----+--------+--------+--------+--------------------------------------
  1 | (0,1)  | 735081 |      0 | {HEAP_HASVARWIDTH,HEAP_XMAX_INVALID}
  2 | (0,2)  | 735081 |      0 | {HEAP_HASVARWIDTH,HEAP_XMAX_INVALID}
  3 | (0,3)  | 735082 |      0 | {HEAP_HASVARWIDTH,HEAP_XMAX_INVALID}
(3 rows)
```
Now we see the new version of the row with the `t_xmin` value equal to the transaction id of the current transaction.

### Client 1. Commit transaction
```sql
commit;
```
### Inspect row versioning
```
 lp | t_ctid | t_xmin | t_xmax |                        raw_flags
----+--------+--------+--------+----------------------------------------------------------
  1 | (0,1)  | 735081 |      0 | {HEAP_HASVARWIDTH,HEAP_XMIN_COMMITTED,HEAP_XMAX_INVALID}
  2 | (0,2)  | 735081 |      0 | {HEAP_HASVARWIDTH,HEAP_XMIN_COMMITTED,HEAP_XMAX_INVALID}
  3 | (0,3)  | 735082 |      0 | {HEAP_HASVARWIDTH,HEAP_XMAX_INVALID}
(3 rows)
```
- `HEAP_XMIN_COMMITTED` - now flag is set in the tuple header to signify that the transaction that inserted the row (1 and 2) is committed.
  > ❓ It's strange that the flag was not set for the row (3) inserted by the transaction 735082.
### Client 2. Select data
```sql
select * from persons;
```
```
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(3 rows)
```
Actually we see the same data as before. We don't see the new record added by the previous transaction (735082).
It is because the `repeatable read` isolation level doesn't allow to see any committed changes made by other transactions after the current transaction has started.

### Client 2. Commit transaction
```sql
commit;
```
### Client 2. Select data
```sql
select * from persons;
```
```
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sveta      | svetova
(3 rows)
```
Now we see the new record in the table. It is because we are outside of the transaction.
