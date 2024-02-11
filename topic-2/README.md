# Домашнее задание

## Установка и настройка PostgreSQL в контейнере Docker

### Цель:

- Установить PostgreSQL в Docker контейнере
- Настроить контейнер для внешнего подключения

### Описание/Пошаговая инструкция выполнения домашнего задания:

1. Создать ВМ с Ubuntu 20.04/22.04 или развернуть Docker любым удобным способом
2. Поставить на нем Docker Engine
3. Сделать каталог `/var/lib/postgres`
4. Развернуть контейнер с PostgreSQL 15, смонтировав в него `/var/lib/postgresql`
5. Развернуть контейнер с клиентом PostgreSQL
6. Подключиться из контейнера с клиентом к контейнеру с сервером и создать таблицу с парой строк
7. Подключиться к контейнеру с сервером с ноутбука/компьютера извне инстансов GCP/ЯО/места установки Docker
8. Удалить контейнер с сервером
9. Создать его заново
10. Подключиться снова из контейнера с клиентом к контейнеру с сервером
11. Проверить, что данные остались на месте
12. Оставляйте в ЛК ДЗ комментарии о том, что и как вы делали и как боролись с проблемами

# Домашняя работа к занятию "Установка PostgreSQL" от 08.02.2024

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
### Check the version of docker compose
```bash
$ docker compose version
```
```bash
Docker Compose version v2.24.5
```
### Add user to the docker group
```bash
$ sudo usermod -aG docker %username%
```
### Log out and log in to apply the changes
```bash
$ docker ps # check if the user has access to the docker
```

## Yandex Cloud. Install portainer.
  > **See: https://docs.portainer.io/start/install-ce/server/docker/linux**

### Use docker-compose.yml
```yaml
version: "3.8"
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: always
    environment:
      - TZ=Europe/Moscow
    ports:
      - 8000:8000
      - 9443:9443
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /opt/portainer/portainer_data:/data
```
```bash
$ docker ps
```
```bash
CONTAINER ID   IMAGE                           COMMAND        CREATED          STATUS          PORTS                                                                                            NAMES
9c5c72c6bd5b   portainer/portainer-ce:latest   "/portainer"   31 seconds ago   Up 28 seconds   0.0.0.0:8000->8000/tcp, :::8000->8000/tcp, 0.0.0.0:9443->9443/tcp, :::9443->9443/tcp, 9000/tcp   portainer
```

### Now you can access portainer at https://158.160.144.19:9443

![Portainer](/public/screenshots/portainer.png)

## Configure postgresql with docker-compose
### Add docker-compose.yml
```yaml
version: '3.8'
services:
  postgres:
    image: postgres:14
    container_name: postgres
    restart: always
    environment:
      POSTGRES_USER: iliakriachkov
      POSTGRES_PASSWORD: password2postgres
      POSTGRES_DB: otus-db
      PGDATA: /var/lib/postgresql/data/pgdata
    ports:
      - 5432:5432
    volumes:
      - /opt/postgres/postgres_data:/var/lib/postgresql/data
```
```bash
$ docker-compose up -d
```
## Login to the postgres container
```bash
$ docker exec -it postgres /bin/bash
```
### Inside the postgres container
```bash
psql -U iliakriachkov -d postgres
```
### Create database
```bash
postgres=# create database otus;
```
### Check if the database is created
```bash
postgres=# \l
                                        List of databases
   Name    |     Owner     | Encoding |  Collate   |   Ctype    |        Access privileges
-----------+---------------+----------+------------+------------+---------------------------------
 otus      | iliakriachkov | UTF8     | en_US.utf8 | en_US.utf8 |
 postgres  | iliakriachkov | UTF8     | en_US.utf8 | en_US.utf8 |
 template0 | iliakriachkov | UTF8     | en_US.utf8 | en_US.utf8 | =c/iliakriachkov               +
           |               |          |            |            | iliakriachkov=CTc/iliakriachkov
 template1 | iliakriachkov | UTF8     | en_US.utf8 | en_US.utf8 | =c/iliakriachkov               +
           |               |          |            |            | iliakriachkov=CTc/iliakriachkov
(4 rows)
```

## Setup connections from the outside of the container
### Change `pg_hba.conf`
```bash
$ sudo vi /opt/postgres/postgres_data/pg_hba.conf
```
```bash
host    all             all            0.0.0.0/0               md5
```
### Install client
```bash
$ sudo apt-get install postgresql-client
```
### Connect to the database from the outside of the container, check if it works
```bash
$ psql -h localhost -p 5432 -d postgres -U iliakriachkov -W
```
## Setup connectection from the remote host (vscode extension)
  ![Database Client](/public/screenshots/database-client.png)

## Check pgdata volume
### Stop and remove containers
```bash
$ cd /opt/postgres
$ docker compose down
```
### Check if postgres container was removed
```bash
$ docker ps
```
```bash
CONTAINER ID   IMAGE                           COMMAND        CREATED       STATUS       PORTS                                                                                            NAMES
1a48c60a55d4   portainer/portainer-ce:latest   "/portainer"   3 hours ago   Up 3 hours   0.0.0.0:8000->8000/tcp, :::8000->8000/tcp, 0.0.0.0:9443->9443/tcp, :::9443->9443/tcp, 9000/tcp   portainer
```
### Run containers
```bash
$ docker compose up -d
```
```bash
CONTAINER ID   IMAGE                           COMMAND                  CREATED          STATUS          PORTS                                                                                            NAMES
3c6ffea37863   postgres:14                     "docker-entrypoint.s…"   18 seconds ago   Up 17 seconds   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp                                                        postgres
1a48c60a55d4   portainer/portainer-ce:latest   "/portainer"             3 hours ago      Up 3 hours      0.0.0.0:8000->8000/tcp, :::8000->8000/tcp, 0.0.0.0:9443->9443/tcp, :::9443->9443/tcp, 9000/tcp   portainer
```
### Check if the database is available and the data is there
```bash
$ psql -h localhost -p 5432 -d postgres -U iliakriachkov -W
```
```bash
postgres=# \l
                                        List of databases
   Name    |     Owner     | Encoding |  Collate   |   Ctype    |        Access privileges
-----------+---------------+----------+------------+------------+---------------------------------
 otus      | iliakriachkov | UTF8     | en_US.utf8 | en_US.utf8 |
 postgres  | iliakriachkov | UTF8     | en_US.utf8 | en_US.utf8 |
 template0 | iliakriachkov | UTF8     | en_US.utf8 | en_US.utf8 | =c/iliakriachkov               +
           |               |          |            |            | iliakriachkov=CTc/iliakriachkov
 template1 | iliakriachkov | UTF8     | en_US.utf8 | en_US.utf8 | =c/iliakriachkov               +
           |               |          |            |            | iliakriachkov=CTc/iliakriachkov
(4 rows)
```
