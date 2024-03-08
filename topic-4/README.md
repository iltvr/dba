# Домашнее задание

## Работа с базами данных, пользователями и правами

### Цель:

- создание новой базы данных, схемы и таблицы
- создание роли для чтения данных из созданной схемы созданной базы данных
- создание роли для чтения и записи из созданной схемы созданной базы данных

### Описание/Пошаговая инструкция выполнения домашнего задания:

1. Создайте новый кластер PostgresSQL 14.
2. Зайдите в созданный кластер под пользователем postgres.
3. Создайте новую базу данных `testdb`.
4. Зайдите в созданную базу данных под пользователем postgres.
5. Создайте новую схему `testnm`.
6. Создайте новую таблицу `t1` с одной колонкой `c1` типа integer.
7. Вставьте строку со значением `c1=1`.
8. Создайте новую роль `readonly`.
9. Дайте новой роли право на подключение к базе данных `testdb`.
10. Дайте новой роли право на использование схемы `testnm`.
11. Дайте новой роли право на select для всех таблиц схемы `testnm`.
12. Создайте пользователя `testread` с паролем `test123`.
13. Дайте роль `readonly` пользователю `testread`.
14. Зайдите под пользователем `testread` в базу данных `testdb`.
15. Сделайте `select * from t1;`.
    - Получилось? (Могло, если вы делали сами не по шпаргалке и не упустили один существенный момент, про который позже.)
16. Напишите, что именно произошло в тексте домашнего задания.
    - У вас есть идеи, почему? Ведь права то дали?
17. Посмотрите на список таблиц.
    - Подсказка: в шпаргалке под пунктом 20.
18. А почему так получилось с таблицей (если делали сами и без шпаргалки, то может у вас все нормально)?
19. Вернитесь в базу данных `testdb` под пользователем postgres.
20. Удалите таблицу `t1`.
21. Создайте ее заново, но уже с явным указанием имени схемы `testnm`.
22. Вставьте строку со значением `c1=1`.
23. Зайдите под пользователем `testread` в базу данных `testdb`.
24. Сделайте `select * from testnm.t1;`.
    - Получилось?
    - Есть идеи, почему? Если нет - смотрите шпаргалку.
25. Как сделать так, чтобы такое больше не повторялось? Если нет идей - смотрите шпаргалку.
26. Сделайте `select * from testnm.t1;`.
    - Получилось?
    - Есть идеи, почему? Если нет - смотрите шпаргалку.
27. Сделайте `select * from testnm.t1;`.
    - Получилось?
    - Ура!
28. Теперь попробуйте выполнить команду `create table t2(c1 integer); insert into t2 values (2);`.
    - А как так? Нам же никто прав на создание таблиц и insert в них под ролью `readonly`?
29. Есть идеи, как убрать эти права? Если нет - смотрите шпаргалку.
    - Если вы справились сами, то расскажите, что сделали и почему. Если смотрели шпаргалку - объясните, что сделали и почему, выполнив указанные в ней команды.
30. Теперь попробуйте выполнить команду `create table t3(c1 integer); insert into t2 values (2);`.
31. Расскажите, что получилось и почему.


# Домашняя работа к занятию "Логический уровень PostgreSQL" от 22.02.2024
## 1. Created new cluster PostgreSQL 14.
```bash
➜  ~ sudo docker run --name otus-db-pg-1 -e POSTGRES_PASSWORD=postgres -d -p 5433:5432 postgres:15
```
## 2. Connect to the created cluster.
```bash
➜  ~ psql -U postgres -p 5433 -h 127.0.0.1
psql (15.5 (Homebrew))
Type "help" for help.

postgres=#
```
## 3. Create a new database `testdb`.
```psql
postgres=# CREATE DATABASE testdb;
CREATE DATABASE
```

Check the databases.
```psql
postgres=# SELECT oid, datname, datistemplate, datallowconn FROM pg_database;
  oid  |  datname  | datistemplate | datallowconn
-------+-----------+---------------+--------------
     5 | postgres  | f             | t
 16388 | testdb    | f             | t
     1 | template1 | t             | t
     4 | template0 | t             | f
(4 rows)
```
## 4. Connect to the created database under the postgres user.
```psql
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres"
```
## 5. Create a new schema `testnm`.
```psql
testdb=# CREATE SCHEMA testnm;
CREATE SCHEMA
```

Check the schemas in the database `testdb`.
```psql
testdb=# \dn
      List of schemas
  Name  |       Owner
--------+-------------------
 public | pg_database_owner
 testnm | postgres
(2 rows)
```
## <a id="p-6"></a> 6. Create a new table `t1` with one column `c1` of type integer.
```psql
testdb=# CREATE TABLE testnm.t1 (c1 integer);
CREATE TABLE
```

Check the tables in the schema `testnm`.
```psql
testdb=# \dt testnm.*
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 testnm | t1   | table | postgres
(1 row)
```
## 7. Insert a row with the value `c1=1`.
```psql
testdb=# INSERT INTO testnm.t1 VALUES (1);
INSERT 0 1
```

Check the data in the table `t1`.
```psql
testdb=# SELECT * FROM testnm.t1;
 c1
----
  1
(1 row)
```
## 8. Create a new role `readonly`.
```psql
testdb=# CREATE ROLE readonly;
CREATE ROLE
```

Check the roles in the database `testdb`.
```psql
testdb=# \du
                                   List of roles
 Role name |                         Attributes                         | Member of
-----------+------------------------------------------------------------+-----------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
 readonly  | Cannot login                                               | {}
```
*We can see that the role `readonly` has no login permission and is not a member of any other role.*

Check the permissions of the role `readonly` in the database `testdb`.
```psql
testdb=# \l
                                                List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+------------+------------+------------+-----------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
 testdb    | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =Tc/postgres         +
           |          |          |            |            |            |                 | postgres=CTc/postgres
(4 rows)
```
*We can see that the role `readonly` has no permissions in the database `testdb`.*
## 9. Grant the new role the right to connect to the database `testdb`.

Documentation: [GRANT](https://www.postgresql.org/docs/current/sql-grant.html)
```psql
testdb=# GRANT CONNECT ON DATABASE testdb TO readonly;
GRANT
```

Check the access permissions of the role `readonly` in the database `testdb`.
```psql
testdb=# \l
                                                List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+------------+------------+------------+-----------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            |
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/postgres          +
           |          |          |            |            |            |                 | postgres=CTc/postgres
 testdb    | postgres | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =Tc/postgres         +
           |          |          |            |            |            |                 | postgres=CTc/postgres+
           |          |          |            |            |            |                 | readonly=c/postgres
(4 rows)
```
*Now we can see access privileges is `readonly=c/postgres` in the database `testdb`.*
> NOTE:
>
> `grantee=privilege-abbreviation[*].../grantor`
>
> `c - CONNECT`


Documentation: [PRIVILEGES](https://www.postgresql.org/docs/current/ddl-priv.html#PRIVILEGE-ABBREVS-TABLE)

Try to connect to the database `testdb` under the role `readonly`.
Open a new terminal and run the following command.
```bash
➜  ~ psql -U readonly -d testdb -p 5433 -h 127.0.0.1
Password for user readonly:
psql: error: connection to server at "127.0.0.1", port 5433 failed: fe_sendauth: no password supplied
```
*We can see that the role `readonly` has no password and cannot connect to the database `testdb`.*

Set the password for the role `readonly`.
```psql
testdb=# ALTER ROLE readonly WITH PASSWORD 'readonly';
ALTER ROLE
```
Check the role can connect to the database `testdb`.
```bash
➜  ~ psql -U readonly -d testdb -p 5433 -h 127.0.0.1 -W
Password:
psql: error: connection to server at "127.0.0.1", port 5433 failed: FATAL:  role "readonly" is not permitted to log in
```
*We can see that the role `readonly` has no login permission.*

Grant the new role the permission to log in.
```psql
testdb=# ALTER ROLE readonly WITH LOGIN;
ALTER ROLE
```
Check the roles in the database `testdb`.
```psql
testdb=# \du
                                   List of roles
 Role name |                         Attributes                         | Member of
-----------+------------------------------------------------------------+-----------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
 readonly  |                                                            | {}
```
*Now we can see that the role `readonly` has login permission*

Check the role can connect to the database `testdb`.
```bash
➜  ~ psql -U readonly -d testdb -p 5433 -h 127.0.0.1 -W
Password:
psql (15.5 (Homebrew))
Type "help" for help.

testdb=>
```
*We can see that the role `readonly` can connect to the database `testdb`.*

Try to select data from the table `t1` under the role `readonly`.
```psql
testdb=> select * from testnm.t1;
ERROR:  permission denied for schema testnm
LINE 1: SELECT * FROM testnm.t1;
                      ^
testdb=>
```
*But the role `readonly` still has no permission to select data from the schema `testnm`.*

## 10. Grant the new role the right to use the schema `testnm`.
```psql
testdb=# GRANT USAGE ON SCHEMA testnm TO readonly;
GRANT
```
Check role `readonly` privileges in the database `testdb` on schema `testnm`.
```psql
testdb=# SELECT grantee, privilege_type
FROM information_schema.role_table_grants
  WHERE table_schema = 'testnm'
  AND grantee = 'readonly';
 grantee | privilege_type
---------+----------------
(0 rows)
```
> QUESTION:
>
> The query returns no rows. Why?

## 11. Grant role `readonly` the right to select data from all tables in the schema `testnm`.
```psql
testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
GRANT
```
Check privileges in the database `testdb` on schema `testnm` for the role `readonly`.
```psql
testdb=# SELECT * FROM information_schema.role_table_grants where grantee = 'readonly';
 grantor  | grantee  | table_catalog | table_schema | table_name | privilege_type | is_grantable | with_hierarchy
----------+----------+---------------+--------------+------------+----------------+--------------+----------------
 postgres | readonly | testdb        | testnm       | t1         | SELECT         | NO           | YES
(1 row)
```
*Now we can see that the role `readonly` has the `SELECT` privilege on all tables in the schema `testnm`.*

Check the role `readonly` can select data from the table `t1`.
```psql
testdb=> SELECT * FROM testnm.t1;
 c1
----
  1
(1 row)
```
## 12. Create a user `testread` with the password `test123`.
```psql
CREATE USER testread WITH PASSWORD 'test123';
CREATE ROLE
```
> NOTE:
>
> The `CREATE USER` and `CREATE ROLE` commands are equivalent.

Check the users in the database `testdb`.
```psql
testdb=# \du
                                    List of roles
 Role name |                         Attributes                         | Member of
-----------+------------------------------------------------------------+-----------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
 readonly  |                                                            | {}
 testread  |                                                            | {}
```
*We can see that the user `testread` has no attributes and is not a member of any other role.*

> NOTE:
>   ```
>    GRANT role_name [, ...] TO role_specification [, ...]
>        [ WITH { ADMIN | INHERIT | SET } { OPTION | TRUE | FALSE } ]
>        [ GRANTED BY role_specification ]
>    ```
> Documentation: [GRANT](https://www.postgresql.org/docs/current/sql-grant.html)

## 13. Grant the role `readonly` to the user `testread`.
```psql
testdb=# GRANT readonly TO testread;
GRANT ROLE
```
Check the roles of the user `testread` in the database `testdb`.
```psql
testdb=# \du
                                    List of roles
 Role name |                         Attributes                         | Member of
-----------+------------------------------------------------------------+------------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
 readonly  |                                                            | {}
 testread  |                                                            | {readonly}
```
*Now we can see that the user `testread` is a member of the role `readonly`.*

## 14. Connect to the database `testdb` under the user `testread`.
```bash
➜  ~ psql -U testread -d testdb -p 5433 -h 127.0.0.1 -W
Password:
psql (15.5 (Homebrew))
Type "help" for help.

testdb=>
```
## 15. Try to select data from the table `t1` under the user `testread`.
```psql
testdb=> SELECT * FROM testnm.t1;
 c1
----
  1
(1 row)
```
*We can see that the user `testread` can select data from the table `t1`.*

Check privileges in the database `testdb` on schema `testnm` for the role `readonly`.
```psql
testdb=# SELECT * FROM information_schema.role_table_grants where grantee = 'testread';
 grantor | grantee | table_catalog | table_schema | table_name | privilege_type | is_grantable | with_hierarchy
---------+---------+---------------+--------------+------------+----------------+--------------+----------------
(0 rows)
```
*We can see that the role `readonly` has no privileges on the schema `testnm` for the user `testread`. But the user `testread` is a member of the role `readonly`. And the role `readonly` has the `SELECT` privilege on all tables in the schema `testnm`.*

## 16/17/18. What exactly happened?
Check the tables in the schema `testnm`.
```psql
testdb=# \dt testnm.*
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 testnm | t1   | table | postgres
(1 row)
```
All the steps were performed as described in the text of the homework. The main reason something could go wrong is that the role `readonly` was granted the `SELECT` privilege on all tables in the schema `testnm` but the table `t1` could have been created in the schema `public` instead of the schema `testnm`.

## 19/20. Delete the table `t1`.
```psql
testdb=# DROP TABLE testnm.t1;
DROP TABLE
```
Check the tables in the schema `testnm`.
```psql
testdb=# \dt testnm.*
Did not find any relation named "testnm.*".
```

## 21. Create the table `t1` in the schema `testnm`.
```psql
testdb=# CREATE TABLE testnm.t1 (c1 integer);
CREATE TABLE
```
Check the tables in the schema `testnm`.
```psql
testdb=# \dt testnm.*
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 testnm | t1   | table | postgres
(1 row)
```

## 22. Insert a row with the value `c1=1`.
```psql
testdb=# INSERT INTO testnm.t1 VALUES (1);
INSERT 0 1
```
## 23. Connect to the database `testdb` under the user `testread`.
```bash
➜  ~ psql -U testread -d testdb -p 5433 -h 127.0.0.1 -W
Password:
psql (15.5 (Homebrew))
Type "help" for help.
```
## 24. Try to select data from the table `t1` under the user `testread`.
```psql
testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
```
*We can see that the user `testread` has no permission to select data from the table `t1`. But it worked before we deleted the table `t1` and created it again in the schema `testnm`.*

Check privileges in the database `testdb` for the role `testread`.
```psql
testdb=# SELECT * FROM information_schema.role_table_grants where grantee = 'testread';
 grantor | grantee | table_catalog | table_schema | table_name | privilege_type | is_grantable | with_hierarchy
---------+---------+---------------+--------------+------------+----------------+--------------+----------------
(0 rows)
```

Check privileges in the database `testdb` on schema `testnm` for the role `readonly`.
```psql
testdb=# SELECT * FROM information_schema.role_table_grants where grantee = 'readonly';
 grantor | grantee | table_catalog | table_schema | table_name | privilege_type | is_grantable | with_hierarchy
---------+---------+---------------+--------------+------------+----------------+--------------+----------------
(0 rows)
```
*We can see that after we deleted the table `t1` and created it again in the schema `testnm` the role `readonly` lost the `SELECT` privilege on the table `t1`. Thus the user `testread` also lost the `SELECT` privilege on the table `t1`.*

Check the roles.
```psql
testdb=# \du
                                    List of roles
 Role name |                         Attributes                         | Member of
-----------+------------------------------------------------------------+------------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
 readonly  |                                                            | {}
 testread  |                                                            | {readonly}
```
*Still the same roles.*

## 25. How to prevent this from happening again?
Let's try to change the default privileges for the schema `testnm` for the role `readonly`.
```psql
testdb=# ALTER DEFAULT PRIVILEGES IN SCHEMA testnm GRANT SELECT ON TABLES TO readonly;
ALTER DEFAULT PRIVILEGES
```
Check the default privileges for the schema `testnm` for the role `readonly`.
```psql
testdb=# \ddp
            Default access privileges
  Owner   | Schema | Type  |  Access privileges
----------+--------+-------+---------------------
 postgres | testnm | table | readonly=r/postgres
(1 row)
```
*We can see that the default access privileges for the schema `testnm` for the role `readonly` are set.*

Check the privileges in the database `testdb` on schema `testnm` for the role `readonly`.
```psql
testdb=# SELECT * FROM information_schema.role_table_grants where grantee = 'readonly';
 grantor | grantee | table_catalog | table_schema | table_name | privilege_type | is_grantable | with_hierarchy
---------+---------+---------------+--------------+------------+----------------+--------------+----------------
(0 rows)
```
*But the role `readonly` still has no privileges on the schema `testnm`.*

Check the role `readonly` can select data from the table `t1`.
```psql
ttestdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
```
*We can see that the role `readonly` still has no permission to select data from the table `t1`. The default access privileges are not applied to the existing tables.*

Recreate the table `t1` in the schema `testnm`.
```psql
testdb=# DROP TABLE testnm.t1;
DROP TABLE
testdb=# CREATE TABLE testnm.t1 (c1 integer);
CREATE TABLE
testdb=# INSERT INTO testnm.t1 VALUES (1);
INSERT 0 1
```

Check the privileges in the database `testdb` on schema `testnm` for the role `readonly`.
```psql
testdb=# SELECT * FROM information_schema.role_table_grants where grantee = 'readonly';
 grantor  | grantee  | table_catalog | table_schema | table_name | privilege_type | is_grantable | with_hierarchy
----------+----------+---------------+--------------+------------+----------------+--------------+----------------
 postgres | readonly | testdb        | testnm       | t1         | SELECT         | NO           | YES
(1 row)
```
*Now we can see that the role `readonly` has the `SELECT` privilege on the table `t1`. It was granted by the default access privileges.*

## 26/27. Check the role `readonly` and `testread` can select data from the table `t1`.
Check the role `readonly` can select data from the table `t1`.
```psql
testdb=> select * from testnm.t1;
 c1
----
  1
(1 row)
```

Check the role `testread` can select data from the table `t1`.
```psql
testdb=> select * from testnm.t1;
 c1
----
  1
(1 row)
```
*We can see that the role `readonly` and the user `testread` can select data from the table `t1`.*

## 28. Try to execute the command `create table t2(c1 integer); insert into t2 values (2);` under the role `readonly`.
Connect to the database `testdb` under the role `readonly`.
```bash
➜  ~ psql -U readonly -d testdb -p 5433 -h 127.0.0.1 -W
Password:
psql (15.5 (Homebrew))
Type "help" for help.

testdb=>
```
Try to execute the command `create table t2(c1 integer); insert into t2 values (2);`.
```psql
testdb=> create table t2(c1 integer); insert into t2 values (2);
ERROR:  permission denied for schema public
LINE 1: create table t2(c1 integer);
                     ^
ERROR:  relation "t2" does not exist
LINE 1: insert into t2 values (2);
                    ^
```
*We can see that the role `readonly` has no permission to create a table in the schema `public` and insert data into it.*

Change the command to `create table testnm.t2(c1 integer); insert into testnm.t2 values (2);`.
```psql
testdb=> create table testnm.t2(c1 integer); insert into testnm.t2 values (2);
ERROR:  permission denied for schema testnm
LINE 1: create table testnm.t2(c1 integer);
                     ^
ERROR:  relation "testnm.t2" does not exist
LINE 1: insert into testnm.t2 values (2);
                    ^
testdb=>
```
*Still the same error. It seems that the role `readonly` has no permission to create a table in the schema `testnm` and insert data into it.*

## Conclusion
It looks like the access privileges configured properly.

OK.

Let's check the "hints for the homework" and see what we missed.
It seems that we unintentionally used schema name `testnm` in the `CREATE TABLE` command when we dealt with ["6. Create a new table `t1` with one column `c1` of type integer."](#p-6) And we should NOT have use the schema name in the `CREATE TABLE` command. Then the table `t1` would have been created in the `public` schema instead of the `testnm` schema. Because of default `search_path` value is `"$user", public` and the `public` schema is the first in the `search_path`.

Let's check the `search_path` value.
```psql
testdb=# SHOW search_path;
   search_path
-----------------
 "$user", public
(1 row)
```
### Bonus
Grant the role `readonly` the right to create a table in the schema `testnm`.
```psql
testdb=# GRANT CREATE ON SCHEMA testnm TO readonly;
GRANT
```
Check the privileges in the database `testdb` on schema `testnm` for the role `readonly`.
```psql
testdb=# SELECT * FROM information_schema.role_table_grants where grantee = 'readonly';
 grantor  | grantee  | table_catalog | table_schema | table_name | privilege_type | is_grantable | with_hierarchy
----------+----------+---------------+--------------+------------+----------------+--------------+----------------
 postgres | readonly | testdb        | testnm       | t1         | SELECT         | NO           | YES
(1 row)
```
*Here we can see only privilege of the role `readonly` on the schema `testnm` and table `t1`. The `CREATE` privileges is not listed because after we granted the `CREATE` privilege the schema we did not create any table in the schema `testnm`.*

Check all privileges of the role `readonly` on the schema `testnm` even if there are no tables in the schema `testnm`.
```psql
testdb=# SELECT
    rolname AS grantee,
    nspname AS schema_name,
    CASE
        WHEN has_schema_privilege(rolname, nspname, 'CREATE') THEN 'CREATE'
        WHEN has_schema_privilege(rolname, nspname, 'USAGE') THEN 'USAGE'
        ELSE 'NO PRIVILEGES'
    END AS privilege_type
FROM
    pg_catalog.pg_namespace n
CROSS JOIN
    pg_catalog.pg_roles r
WHERE
    n.nspname = 'testnm'
    AND r.rolname = 'readonly';
 grantee  | schema_name | privilege_type
----------+-------------+----------------
 readonly | testnm      | CREATE
(1 row)
```

Check the role `readonly` can create a table in the schema `testnm`.
```psql
testdb=> create table testnm.t2(c1 integer); insert into testnm.t2 values (2);
CREATE TABLE
INSERT 0 1
```

Check the privileges in the database `testdb` on schema `testnm` for the role `readonly`.
```psql
testdb=# SELECT * FROM information_schema.role_table_grants where grantee = 'readonly';
 grantor  | grantee  | table_catalog | table_schema | table_name | privilege_type | is_grantable | with_hierarchy
----------+----------+---------------+--------------+------------+----------------+--------------+----------------
 postgres | readonly | testdb        | testnm       | t1         | SELECT         | NO           | YES
 readonly | readonly | testdb        | testnm       | t2         | INSERT         | YES          | NO
 readonly | readonly | testdb        | testnm       | t2         | SELECT         | YES          | YES
 readonly | readonly | testdb        | testnm       | t2         | UPDATE         | YES          | NO
 readonly | readonly | testdb        | testnm       | t2         | DELETE         | YES          | NO
 readonly | readonly | testdb        | testnm       | t2         | TRUNCATE       | YES          | NO
 readonly | readonly | testdb        | testnm       | t2         | REFERENCES     | YES          | NO
 readonly | readonly | testdb        | testnm       | t2         | TRIGGER        | YES          | NO
(8 rows)
```
*Now we can see that the role `readonly` has the `INSERT`, `SELECT` and other privileges on the table `t2` in the schema `testnm`.*

Check the role `testread` can select data from the table `t2`.
```bash
➜  ~ psql -U testread -d testdb -p 5433 -h 127.0.0.1 -W
```

```psql
testdb=> select * from testnm.t2;
 c1
----
  1
(1 row)
```
*We can see that the user `testread` can select data from the table `t2` which was created by the role `readonly`.*

Check the role `testread` can create a table `t3` and insert data into it.
```psql
testdb=> create table testnm.t3(c1 integer); insert into testnm.t3 values (2);
CREATE TABLE
INSERT 0 1
```
*We can see that the role `testread` can create a table `t3` and insert data into it.*

Check tables in the schema `testnm`.
```psql
testdb=# \dt testnm.*
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 testnm | t1   | table | postgres
 testnm | t2   | table | readonly
 testnm | t3   | table | testread
(3 rows)
```

Check the privileges in the database `testdb` on schema `testnm` for the role `testread`.
```psql
testdb=# SELECT * FROM information_schema.role_table_grants where grantee = 'testread';
 grantor  | grantee  | table_catalog | table_schema | table_name | privilege_type | is_grantable | with_hierarchy
----------+----------+---------------+--------------+------------+----------------+--------------+----------------
 testread | testread | testdb        | testnm       | t3         | INSERT         | YES          | NO
 testread | testread | testdb        | testnm       | t3         | SELECT         | YES          | YES
 testread | testread | testdb        | testnm       | t3         | UPDATE         | YES          | NO
 testread | testread | testdb        | testnm       | t3         | DELETE         | YES          | NO
 testread | testread | testdb        | testnm       | t3         | TRUNCATE       | YES          | NO
 testread | testread | testdb        | testnm       | t3         | REFERENCES     | YES          | NO
 testread | testread | testdb        | testnm       | t3         | TRIGGER        | YES          | NO
(7 rows)
```
*Now we can see that the role `testread` has the `INSERT`, `SELECT` and other privileges on the table `t3` in the schema `testnm`.*

## 29. How to remove these rights?
Revoke all privileges the role `testread` on tables of the schema `testnm`.
```psql
testdb=# REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA testnm FROM testread;
REVOKE
```
Check the privileges in the database `testdb` on schema `testnm` for the role `testread`.
```psql
testdb=# SELECT * FROM information_schema.role_table_grants where grantee = 'testread';
 grantor | grantee | table_catalog | table_schema | table_name | privilege_type | is_grantable | with_hierarchy
---------+---------+---------------+--------------+------------+----------------+--------------+----------------
(0 rows)
```
*Now we can see that the role `testread` has no privileges on the tables in the schema `testnm`.*

Check the privileges in the database `testdb` on schema `testnm` for the role `readonly`.
```psql
testdb=# SELECT * FROM information_schema.role_table_grants where grantee = 'readonly';
 grantor  | grantee  | table_catalog | table_schema | table_name | privilege_type | is_grantable | with_hierarchy
----------+----------+---------------+--------------+------------+----------------+--------------+----------------
 postgres | readonly | testdb        | testnm       | t1         | SELECT         | NO           | YES
 readonly | readonly | testdb        | testnm       | t2         | INSERT         | YES          | NO
 readonly | readonly | testdb        | testnm       | t2         | SELECT         | YES          | YES
 readonly | readonly | testdb        | testnm       | t2         | UPDATE         | YES          | NO
 readonly | readonly | testdb        | testnm       | t2         | DELETE         | YES          | NO
 readonly | readonly | testdb        | testnm       | t2         | TRUNCATE       | YES          | NO
 readonly | readonly | testdb        | testnm       | t2         | REFERENCES     | YES          | NO
 readonly | readonly | testdb        | testnm       | t2         | TRIGGER        | YES          | NO
(8 rows)
```
*Now we can see that the role `readonly` has no privileges on the table `t3` in the schema `testnm`.*

Check the role `readonly` can select data from the table `t3`.
```bash
➜  ~ psql -U readonly -d testdb -p 5433 -h 127.0.0.1 -W
```
```psql
testdb=> select * from testnm.t3;
ERROR:  permission denied for table t3
```
*We can see that the role `readonly` has no permission to select data from the table `t3`.*

Add the `SELECT` privilege on the table `t3` in the schema `testnm` for the role `readonly`.
```psql
testdb=# GRANT SELECT ON testnm.t3 TO readonly;
GRANT
```

Check the privileges in the database `testdb` on schema `testnm` for the role `readonly`.
```psql
 testdb=#  SELECT * FROM information_schema.role_table_grants where grantee = 'readonly';
 grantor  | grantee  | table_catalog | table_schema | table_name | privilege_type | is_grantable | with_hierarchy
----------+----------+---------------+--------------+------------+----------------+--------------+----------------
 postgres | readonly | testdb        | testnm       | t1         | SELECT         | NO           | YES
 readonly | readonly | testdb        | testnm       | t2         | INSERT         | YES          | NO
 readonly | readonly | testdb        | testnm       | t2         | SELECT         | YES          | YES
 readonly | readonly | testdb        | testnm       | t2         | UPDATE         | YES          | NO
 readonly | readonly | testdb        | testnm       | t2         | DELETE         | YES          | NO
 readonly | readonly | testdb        | testnm       | t2         | TRUNCATE       | YES          | NO
 readonly | readonly | testdb        | testnm       | t2         | REFERENCES     | YES          | NO
 readonly | readonly | testdb        | testnm       | t2         | TRIGGER        | YES          | NO
 testread | readonly | testdb        | testnm       | t3         | SELECT         | NO           | YES
(9 rows)
```

Now select data from the table `t3` under the role `readonly`.
```bash
➜  ~ psql -U readonly -d testdb -p 5433 -h 127.0.0.1 -W
```
```psql
testdb=> select * from testnm.t3;
 c1
----
  2
(1 row)
```
*We can see there that the role `readonly` can select data from the table `t3`.*

## Conclusion
Despite of user `testread` has granted the `readonly` role, the role `readonly` has no privileges on the tables which created by the user `testread`. So, the user `readonly` cannot select data from the table `t3` in the schema `testnm`.
