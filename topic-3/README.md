# Домашнее задание
Установка и настройка PostgreSQL
## Цель:

- создавать дополнительный диск для уже существующей виртуальной машины, размечать его и делать на нем файловую систему
- переносить содержимое базы данных PostgreSQL на дополнительный диск
- переносить содержимое БД PostgreSQL между виртуальными машинами

## Описание/Пошаговая инструкция выполнения домашнего задания:

1. Создайте виртуальную машину с Ubuntu 20.04/22.04 LTS в GCE/ЯО/Virtual Box/докере.
2. Установите PostgreSQL 15 через `sudo apt`.
3. Проверьте, что кластер запущен, выполнив `sudo -u postgres pg_lsclusters`.
4. Войдите из-под пользователя postgres в psql и создайте произвольную таблицу с произвольным содержимым:
    ```sql
    postgres=# create table test(c1 text);
    postgres=# insert into test values('1');
    \q
    ```
5. Остановите PostgreSQL, например, через `sudo -u postgres pg_ctlcluster 15 main stop`.
6. Создайте новый диск для ВМ размером 10GB.
7. Добавьте свеже-созданный диск к виртуальной машине - зайдите в режим ее редактирования и выберите пункт "attach existing disk".
8. Проинициализируйте диск согласно инструкции и подмонтируйте файловую систему. Не забудьте менять имя диска на актуальное, в вашем случае это скорее всего будет `/dev/sdb`. [Инструкция](https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux).
9. Перезагрузите инстанс и убедитесь, что диск остается примонтированным (если не так, проверьте `fstab`).
10. Сделайте пользователя postgres владельцем `/mnt/data`:
    ```bash
    chown -R postgres:postgres /mnt/data/
    ```
11. Перенесите содержимое `/var/lib/postgres/15` в `/mnt/data`:
    ```bash
    mv /var/lib/postgresql/15 /mnt/data
    ```
12. Попытайтесь запустить кластер:
    ```bash
    sudo -u postgres pg_ctlcluster 15 main start
    ```
    Напишите, получилось ли и почему.
13. Найдите конфигурационный параметр в файлах, расположенных в `/etc/postgresql/15/main`, который надо поменять, и поменяйте его.
    Напишите, что и почему поменяли.
14. Попытайтесь запустить кластер:
    ```bash
    sudo -u postgres pg_ctlcluster 15 main start
    ```
    Напишите, получилось ли и почему.
15. Зайдите через psql и проверьте содержимое ранее созданной таблицы.

## Задание со звездочкой *:

Не удаляя существующий инстанс ВМ, сделайте новый. Установите на него PostgreSQL, удалите файлы с данными из `/var/lib/postgres`, перемонтируйте внешний диск, который сделали ранее от первой виртуальной машины ко второй, и запустите PostgreSQL на второй машине так, чтобы он работал с данными на внешнем диске.
Расскажите, как вы это сделали и что в итоге получилось.


# Домашняя работа к занятию "Физический уровень PostgreSQL" от 20.02.2024
## 1. Create a new VM with Ubuntu 20.04 using YC web interface.
- view networks
```bash
➜  ~ yc vpc network list
+----------------------+---------+
|          ID          |  NAME   |
+----------------------+---------+
| enpo6musfeqqdj2nhfrs | default |
+----------------------+---------+
```
- create new subnet
```bash
➜  ~ yc vpc subnet create --name otus-subnet-1 --description "otus-subnet" --range 192.168.0.0/24 --network-name default
id: fl8ogf9pp7nlijj4gl3p
folder_id: b1gffa7irklogm13fdjq
created_at: "2024-02-28T18:53:28Z"
name: otus-subnet-1
description: otus-subnet
network_id: enpo6musfeqqdj2nhfrs
zone_id: ru-central1-d
v4_cidr_blocks:
  - 192.168.0.0/24
```
## 2. Install PostgreSQL 14 using `sudo apt`.
- login using ssh `~ ssh otus-db-pg-1`
- create the file repository configuration:
```bash
$ sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
  ```
- import the repository signing key:
```bash
$ wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
Warning: apt-key is deprecated. Manage keyring files in trusted.gpg.d instead (see apt-key(8)).
OK
  ```
- update the package lists:
```bash
$ sudo apt update && sudo apt upgrade -y -q
...
Service restarts being deferred:
systemctl restart systemd-logind.service
systemctl restart unattended-upgrades.service
systemctl restart user@1000.service
```
- install PostgreSQL 14:
```bash
$ sudo apt -y install postgresql-14
```
## 3. Ensure postgres is running:
```bash
$ sudo systemctl start postgresql.service
$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5433 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
```
### 3.1 check pg firewall configuration file
Check if the postgres user can connect to the database using the `peer` authentication method.
```bash
$ sudo su postgres
$ cd /var/lib/postgresql/14/main/
```
```bash
$ sudo cat /etc/postgresql/14/main/pg_hba.conf
local   all             postgres                                peer

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            scram-sha-256
# IPv6 local connections:
host    all             all             ::1/128                 scram-sha-256
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     peer
host    replication     all             127.0.0.1/32            scram-sha-256
host    replication     all             ::1/128                 scram-sha-256
```
There we can see that the server is configured to accept connections from the local machine only.
The user `postgres` can connect to the database using the `peer` authentication method.
### 3.2 show listen addresses
```psql
postgres=# show listen_addresses;
listen_addresses
------------------
localhost
(1 row)
```
We can see that the connection to the database is successful.
## 4. Create a table and insert a row:
```bash
$ sudo -u postgres psql
```
```psql
postgres=# create table test(c1 text);
CREATE TABLE
postgres=# insert into test values('hello 1');
postgres=# select * from test;
    c1
---------
hello 1
(1 row)
```
## 5. Stop PostgreSQL:
```bash
# $ sudo -u postgres pg_ctlcluster 14 main stop
$ sudo systemctl stop postgresql@14-main
```
## 6. Create a new disk for the VM - size 10GB.
### 6.1 create a new disk in YC web interface
- check types of disks
```bash
➜  ~ yc compute disk-type list
+---------------------------+--------------------------------+
|            ID             |          DESCRIPTION           |
+---------------------------+--------------------------------+
| network-hdd               | Network storage with HDD       |
|                           | backend                        |
| network-ssd               | Network storage with SSD       |
|                           | backend                        |
| network-ssd-io-m3         | Fast network storage with      |
|                           | three replicas                 |
| network-ssd-nonreplicated | Non-replicated network storage |
|                           | with SSD backend               |
+---------------------------+--------------------------------+
```
- check existing disks
```bash
➜  ~ yc compute disk list
+----------------------+------+-------------+---------------+--------+----------------------+-----------------+-------------+
|          ID          | NAME |    SIZE     |     ZONE      | STATUS |     INSTANCE IDS     | PLACEMENT GROUP | DESCRIPTION |
+----------------------+------+-------------+---------------+--------+----------------------+-----------------+-------------+
| fv45n9bess1nuod3b7bk |      | 19327352832 | ru-central1-d | READY  | fv47thb8u3ap8ti1pk06 |                 |             |
+----------------------+------+-------------+---------------+--------+----------------------+-----------------+-------------+
```
- create a new network HDD disk
```bash
➜  ~ yc compute disk create --name otus-db-pg-1-disk-1 --description "otus-db-pg-1-disk" --size 10 --type network-hdd
done (8s)
id: fv4oj5t4ec7cvcr647ah
folder_id: b1gffa7irklogm13fdjq
created_at: "2024-02-28T20:05:17Z"
name: otus-db-pg-1-disk-1
description: otus-db-pg-1-disk
type_id: network-hdd
zone_id: ru-central1-d
size: "10737418240"
block_size: "4096"
status: READY
disk_placement_policy: {}
```
```bash
➜  ~ yc compute disk list
+----------------------+---------------------+-------------+---------------+--------+----------------------+-----------------+-------------------+
|          ID          |        NAME         |    SIZE     |     ZONE      | STATUS |     INSTANCE IDS     | PLACEMENT GROUP |    DESCRIPTION    |
+----------------------+---------------------+-------------+---------------+--------+----------------------+-----------------+-------------------+
| fv45n9bess1nuod3b7bk |                     | 19327352832 | ru-central1-d | READY  | fv47thb8u3ap8ti1pk06 |                 |                   |
| fv4oj5t4ec7cvcr647ah | otus-db-pg-1-disk-1 | 10737418240 | ru-central1-d | READY  |                      |                 | otus-db-pg-1-disk |
+----------------------+---------------------+-------------+---------------+--------+----------------------+-----------------+-------------------+
```
## 7. Attach the disk to the VM `otus-db-pg-1`
```bash
➜  ~ yc compute instance attach-disk otus-db-pg-1 --disk-name otus-db-pg-1-disk-1
done (9s)
id: fv47thb8u3ap8ti1pk06
folder_id: b1gffa7irklogm13fdjq
created_at: "2024-02-11T08:23:42Z"
name: otus-db-pg-1
zone_id: ru-central1-d
platform_id: standard-v3
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
  device_name: fv45n9bess1nuod3b7bk
  auto_delete: true
  disk_id: fv45n9bess1nuod3b7bk
secondary_disks:
  - mode: READ_WRITE
    device_name: fv4oj5t4ec7cvcr647ah
    disk_id: fv4oj5t4ec7cvcr647ah
network_interfaces:
  - index: "0"
    mac_address: d0:0d:7e:c5:68:f0
    subnet_id: fl89834amppv0hn4507t
    primary_v4_address:
      address: 10.131.0.8
      one_to_one_nat:
        address: 158.160.134.138
        ip_version: IPV4
gpu_settings: {}
fqdn: otus-db-pg-1.ru-central1.internal
scheduling_policy: {}
network_settings:
  type: STANDARD
placement_policy: {}
```
## 8. Initialize a new partition
### 8.1 Check the disk
login using ssh `~ ssh otus-db-pg-1`
```bash
$ sudo fdisk -l
...
Disk /dev/vda: 18 GiB, 19327352832 bytes, 37748736 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: gpt
Disk identifier: 76209B49-C05A-4E13-B7E6-9FFFBC929BD3

Device     Start      End  Sectors Size Type
/dev/vda1   2048     4095     2048   1M BIOS boot
/dev/vda2   4096 37748702 37744607  18G Linux filesystem


Disk /dev/vdb: 10 GiB, 10737418240 bytes, 20971520 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
```
Now we can see that the name of the disk is `/dev/vdb` and it has a size of 10GB.
Also, we can see that the disk is attached to the VM using `sudo lsblk -o NAME,FSTYPE,SIZE,MOUNTPOINT,LABEL`
```bash
$ sudo lsblk -o NAME,FSTYPE,SIZE,MOUNTPOINT,LABEL
NAME   FSTYPE     SIZE MOUNTPOINT        LABEL
loop0  squashfs  63.9M /snap/core20/2105
loop1  squashfs  63.9M /snap/core20/2182
loop2  squashfs 111.9M /snap/lxd/24322
loop3  squashfs    87M /snap/lxd/27037
loop4  squashfs  49.8M /snap/snapd/18357
loop5  squashfs  40.4M /snap/snapd/20671
vda                18G
├─vda1              1M
└─vda2 ext4        18G /
vdb                10G
```
## 8.2 Create a new partition
```bash
$ sudo fdisk /dev/sdb

Welcome to fdisk (util-linux 2.37.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0xb55eff29.

Command (m for help): n
Partition type
  p   primary (0 primary, 0 extended, 4 free)
  e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1):
First sector (2048-20971519, default 2048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-20971519, default 20971519):

Created a new partition 1 of type 'Linux' and of size 10 GiB.

Command (m for help): p
Disk /dev/vdb: 10 GiB, 10737418240 bytes, 20971520 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: dos
Disk identifier: 0xb55eff29

Device     Boot Start      End  Sectors Size Id Type
/dev/vdb1        2048 20971519 20969472  10G 83 Linux

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```
### 8.3 Format the partition
```bash
$ sudo mkfs.ext4 /dev/vdb1
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 2621184 4k blocks and 655360 inodes
Filesystem UUID: 39639b1e-0394-4e11-a99a-7e8dbf2a01ed
Superblock backups stored on blocks:
  32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done
```
### 8.4 Mount the partition to `/mnt/data`
```bash
$ sudo mkdir /mnt/data
$ sudo mount /dev/vdb1 /mnt/data
```
### 8.5 Check the partition
```bash
$ sudo lsblk --fs
NAME   FSTYPE   FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
loop0  squashfs 4.0                                                    0   100% /snap/core20/2105
loop1  squashfs 4.0                                                    0   100% /snap/core20/2182
loop2  squashfs 4.0                                                    0   100% /snap/lxd/24322
loop3  squashfs 4.0                                                    0   100% /snap/lxd/27037
loop4  squashfs 4.0                                                    0   100% /snap/snapd/18357
loop5  squashfs 4.0                                                    0   100% /snap/snapd/20671
vda
├─vda1
└─vda2 ext4     1.0         ed465c6e-049a-41c6-8e0b-c8da348a3577   10.2G    37% /
vdb
└─vdb1 ext4     1.0         39639b1e-0394-4e11-a99a-7e8dbf2a01ed    9.2G     0% /mnt/data
```
### 8.6 Make the partition mount automatically
```bash
$ sudo blkid
/dev/vdb1: UUID="39639b1e-0394-4e11-a99a-7e8dbf2a01ed" TYPE="ext4" PARTUUID="b55eff29-01"
```
```bash
$ sudo nano /etc/fstab
```
Add the following line to the file:
```bash
UUID=39639b1e-0394-4e11-a99a-7e8dbf2a01ed /mnt/data ext4 defaults 0 0
```
Save the file and exit.
```bash
$ sudo mount -a
```
Now the partition will mount automatically after the system restart.
## 9. Make sure the partition is mounted after the system restart
```bash
➜  ~ yc compute instance stop otus-db-pg-1
done (24s)
➜  ~ yc compute instance start otus-db-pg-1
done (19s)
```
login using ssh `ssh otus-db-pg-1`
```bash
$ sudo lsblk -o NAME,FSTYPE,SIZE,MOUNTPOINT,LABEL
NAME   FSTYPE     SIZE MOUNTPOINT        LABEL
loop0  squashfs  63.9M /snap/core20/2105
loop1  squashfs  63.9M /snap/core20/2182
loop2  squashfs 111.9M /snap/lxd/24322
loop3  squashfs    87M /snap/lxd/27037
loop4  squashfs  49.8M /snap/snapd/18357
loop5  squashfs  40.4M /snap/snapd/20671
vda                18G
├─vda1              1M
└─vda2 ext4        18G /
vdb                10G
└─vdb1 ext4        10G /mnt/data
```
So we can see that the partition is mounted after the system restart.
## 10. Change the owner of the partition
```bash
$ sudo chown -R postgres:postgres /mnt/data
```
## 11. Move the data from `/var/lib/postgresql/14` to `/mnt/data`
```bash
$ sudo systemctl stop postgresql@14-main
$ sudo mv /var/lib/postgresql/14 /mnt/data
```
## 12. Now if we start the PostgreSQL server, it will not be able to find the data directory.
```bash
$ sudo systemctl start postgresql@14-main
$ sudo systemctl status postgresql@14-main
$ sudo journalctl -xeu postgresql@14-main.service
Feb 28 21:09:30 otus-db-pg-1 systemd[1]: Failed to start PostgreSQL Cluster 14-main.
░░ Subject: A start job for unit postgresql@14-main.service has failed
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░
░░ A start job for unit postgresql@14-main.service has finished with a failure.
░░
░░ The job identifier is 673 and the job result is failed.
```
We can see that the server is not running.
```bash
$ sudo -u postgres pg_ctlcluster 14 main start
Error: /var/lib/postgresql/14/main is not accessible or does not exist
```
# 13. Change the data directory in the configuration file
```bash
$ sudo nano /etc/postgresql/14/main/postgresql.conf
```
Change the following line:
```bash
data_directory = '/var/lib/postgresql/14/main'
```
to:
```bash
data_directory = '/mnt/data/14/main'
```
  Save the file and exit.
## 14. Start cluster
```bash
$ sudo systemctl start postgresql@14-main
$ sudo systemctl status postgresql@14-main
...
Feb 28 21:15:48 otus-db-pg-1 systemd[1]: Started PostgreSQL Cluster 14-main.
```
We can see that the server is running.
## 15 Check the data
```bash
$ sudo -u postgres psql
```
```psql
postgres=# select * from test;
  c1
---------
hello 1
(1 row)
```
We can see that the data is still there.
## 16. Create a new VM `otus-db-pg-2`
```bash
➜  ~ yc compute instance create --name otus-db-pg-2 --zone ru-central1-d --network-interface subnet-name=otus-subnet-1,nat-ip-version=ipv4 --create-boot-disk size=15G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2004-lts --ssh-key ~/.ssh/yc_key.pub
done (27s)
```
Check the new VM
```bash
➜  ~ yc compute instance list
+----------------------+--------------+---------------+---------+-----------------+-------------+
|          ID          |     NAME     |    ZONE ID    | STATUS  |   EXTERNAL IP   | INTERNAL IP |
+----------------------+--------------+---------------+---------+-----------------+-------------+
| fv47thb8u3ap8ti1pk06 | otus-db-pg-1 | ru-central1-d | RUNNING | 158.160.140.113 | 10.131.0.8  |
| fv4ivct1vnhgkegi61v2 | otus-db-pg-2 | ru-central1-d | RUNNING | 158.160.149.67  | 192.168.0.6 |
+----------------------+--------------+---------------+---------+-----------------+-------------+
```
## 17. Install PostgreSQL 14 on `otus-db-pg-2`
login using ssh `ssh otus-db-pg-2`
```bash
$ sudo apt update && sudo apt upgrade -y -q
$ sudo apt -y install postgresql-14
```
## 18. Stop PostgreSQL
```bash
$ sudo systemctl stop postgresql@14-main
```
## 19. Remove all data from `/var/lib/postgresql/14`
```bash
$ sudo rm -rf /var/lib/postgresql/14
```
## 20. Attach YC disk `otus-db-pg-1-disk-1` to `otus-db-pg-2`
```bash
➜  ~ yc compute instance attach-disk otus-db-pg-2 --disk-name otus-db-pg-1-disk-1
ERROR: rpc error: code = FailedPrecondition desc = Disk "fv4oj5t4ec7cvcr647ah" already attached to "fv47thb8u3ap8ti1pk06"
```
We can see that the disk is already attached to the VM `otus-db-pg-1`.
### 20.1 Detach the disk from `otus-db-pg-1`
- stop PG cluster in `otus-db-pg-1`
```bash
$ sudo systemctl stop postgresql@14-main
...
Feb 28 21:48:30 otus-db-pg-1 systemd[1]: Stopped PostgreSQL Cluster 14-main.
```
- detach the disk
```bash
➜  ~ yc compute instance detach-disk otus-db-pg-1 --disk-name otus-db-pg-1-disk-1
done (16s)
```
- attach the disk to `otus-db-pg-2`
```bash
➜  ~ yc compute instance attach-disk otus-db-pg-2 --disk-name otus-db-pg-1-disk-1
done (9s)
id: fv4ivct1vnhgkegi61v2
folder_id: b1gffa7irklogm13fdjq
created_at: "2024-02-28T21:24:12Z"
name: otus-db-pg-2
zone_id: ru-central1-d
platform_id: standard-v2
resources:
  memory: "2147483648"
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
  device_name: fv4h1nta5j1mfeerrsj8
  auto_delete: true
  disk_id: fv4h1nta5j1mfeerrsj8
secondary_disks:
  - mode: READ_WRITE
    device_name: fv4oj5t4ec7cvcr647ah
    disk_id: fv4oj5t4ec7cvcr647ah
network_interfaces:
  - index: "0"
    mac_address: d0:0d:12:fb:3a:1f
    subnet_id: fl8ogf9pp7nlijj4gl3p
    primary_v4_address:
      address: 192.168.0.6
      one_to_one_nat:
        address: 158.160.149.67
        ip_version: IPV4
gpu_settings: {}
fqdn: fv4ivct1vnhgkegi61v2.auto.internal
scheduling_policy: {}
network_settings:
  type: STANDARD
placement_policy: {}
```
## 21. Mount the disk to `/mnt/data` on `otus-db-pg-2`
- login using ssh `ssh otus-db-pg-2`
- check the disk
```bash
$ sudo lsblk --fs
NAME   FSTYPE LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINT
vda
├─vda1
└─vda2 ext4         be2c7c06-cc2b-4d4b-96c6-e3700932b129   11.3G    18% /
vdb
└─vdb1 ext4         39639b1e-0394-4e11-a99a-7e8dbf2a01ed
```
As we can see the disk is attached to the VM and OS can see it as `vdb1`.
- mount the disk to `/mnt/data`
```bash
$ sudo mkdir /mnt/data
$ sudo mount /dev/vdb1 /mnt/data
```
- check the disk
```bash
$ sudo lsblk --fs
NAME   FSTYPE LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINT
vda
├─vda1
└─vda2 ext4         be2c7c06-cc2b-4d4b-96c6-e3700932b129   11.3G    18% /
vdb
└─vdb1 ext4         39639b1e-0394-4e11-a99a-7e8dbf2a01ed    9.2G     0% /mnt/data
```
## 22. Check the owner of the partition
```bash
$ sudo ls -l /mnt/data
total 20
drwx------ 2 root root 16384 Feb 28 21:24 lost+found
```
## 23. Change the owner of the partition
```bash
$ sudo chown -R postgres:postgres /mnt/data
$ sudo ls -l /mnt/data
total 20
drwxr-xr-x 3 postgres postgres  4096 Feb 28 19:17 14
drwx------ 2 postgres postgres 16384 Feb 28 20:25 lost+found
```
## 24. Configure PostgreSQL to use the new data directory
```bash
$ sudo nano /etc/postgresql/14/main/postgresql.conf
```
Change `data_directory` to the following line:
```bash
data_directory = '/mnt/data/14/main'
```
Save the file and exit.
## 25. Start the PostgreSQL server
```bash
$ sudo systemctl start postgresql@14-main
$ sudo systemctl status postgresql@14-main
Feb 28 21:58:59 fv4ivct1vnhgkegi61v2 systemd[1]: Started PostgreSQL Cluster 14-main.
```
## 26. Check the data
```bash
$ sudo -u postgres psql
```
```psql
postgres=# select * from test;
    c1
 ---------
  hello 1
(1 row)
```
WELL DONE! We can see that the data is still there!
## Conclusion
We have created a new VM with Ubuntu 20.04 on Yandex Cloud. We have practiced the following:
- installing PostgreSQL 14 using `sudo apt`
- using yc CLI to create a new subnet, a new disk, and attach it to the VM
- creating a new disk for the VM, attaching it, and mounting it to `/mnt/data`
- moving the data from `/var/lib/postgresql/14` to `/mnt/data`
- configuring PostgreSQL to use the new data directory
- starting a new PostgreSQL clustre on a new VM using the data from the old VM
