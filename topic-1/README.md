


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
