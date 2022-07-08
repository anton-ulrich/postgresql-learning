# Домашняя работа 9
## Репликация

Развернул 3 виртуалки в (pg-homework09-01, pg-homework09-02, pg-homework09-03) и установил на них Postgres
```
sudo apt-get update && sudo apt-get upgrade -y
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt-get update && sudo apt-get install -y postgresql-14
```

Заходим в PSQL и создаем БД replication на всех серверах и о создаем по 2 таблицы на каждом из серверов
```
anton@pg-homework09-01:~$ sudo -u postgres psql
psql (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
Type "help" for help.

postgres=# create database replication;
CREATE DATABASE

postgres=# \c replication
You are now connected to database "replication" as user "postgres".

replication=# create table test(id serial, name text);
CREATE TABLE

replication=# create table test2(id serial, name text);
CREATE TABLE
```

В postgresql.conf устанавличаем wal_level в значение logical и меняем слушаемые порты на * на всех 3х серверах
```
anton@pg-homework09-01:~$ sudo vim /etc/postgresql/14/main/postgresql.conf

...
#------------------------------------------------------------------------------
# CONNECTIONS AND AUTHENTICATION
#------------------------------------------------------------------------------

# - Connection Settings -

listen_addresses = '*'                  # what IP address(es) to listen on;
                                        # comma-separated list of addresses;
                                        # defaults to 'localhost'; use '*' for all
                                        # (change requires restart)
port = 5432                             # (change requires restart)
max_connections = 100                   # (change requires restart)
#superuser_reserved_connections = 3     # (change requires restart)
unix_socket_directories = '/var/run/postgresql' # comma-separated list of directories
...

...

#------------------------------------------------------------------------------
# WRITE-AHEAD LOG
#------------------------------------------------------------------------------

# - Settings -

wal_level = logical                     # minimal, replica, or logical
                                        # (change requires restart)
#fsync = on                             # flush data to disk for crash safety
                                        # (turning this off can cause
                                        # unrecoverable data corruption)
#synchronous_commit = on                # synchronization level;
                                        # off, local, remote_write, remote_apply, or on
...
```

Так же меняем pg_hba.conf для того что бы соседние ноды могли обращаться к базе по сети(для безопасности используется внутренняя подсеть яндекса недоступная из вне)
```
anton@pg-homework09-01:~$ sudo vim /etc/postgresql/14/main/pg_hba.conf

...
# IPv4 local connections:
host    all             all             127.0.0.1/32            scram-sha-256
host    all             all             10.129.0.0/24           scram-sha-256
# IPv6 local connections:
...
```

Перезегружаем кластер для применения настроек
```
anton@pg-homework09-01:~$ sudo systemctl stop postgresql@14-main
anton@pg-homework09-01:~$ sudo systemctl start postgresql@14-main
anton@pg-homework09-01:~$ sudo systemctl status postgresql@14-main
● postgresql@14-main.service - PostgreSQL Cluster 14-main
     Loaded: loaded (/lib/systemd/system/postgresql@.service; enabled-runtime; vendor preset: enabled)
     Active: active (running) since Fri 2022-07-08 15:18:19 UTC; 5s ago
    Process: 6505 ExecStart=/usr/bin/pg_ctlcluster --skip-systemctl-redirect 14-main start (code=exited, status=0/SU>
   Main PID: 6524 (postgres)
      Tasks: 7 (limit: 4648)
     Memory: 18.0M
     CGroup: /system.slice/system-postgresql.slice/postgresql@14-main.service
             ├─6524 /usr/lib/postgresql/14/bin/postgres -D /var/lib/postgresql/14/main -c config_file=/etc/postgresq>
             ├─6526 postgres: 14/main: checkpointer
             ├─6527 postgres: 14/main: background writer
             ├─6528 postgres: 14/main: walwriter
             ├─6529 postgres: 14/main: autovacuum launcher
             ├─6530 postgres: 14/main: stats collector
             └─6531 postgres: 14/main: logical replication launcher

Jul 08 15:18:17 pg-homework09-01 systemd[1]: Starting PostgreSQL Cluster 14-main...
Jul 08 15:18:19 pg-homework09-01 systemd[1]: Started PostgreSQL Cluster 14-main.
```

Далее приступаем к настройке публицаций
##### Сервер 1
Создаем публикацию и задаем пароль для пользователя postgress на сервере
```
anton@pg-homework09-01:~$ sudo -u postgres psql
psql (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
Type "help" for help.

postgres=# \c replication
You are now connected to database "replication" as user "postgres".

replication=# create publication pub_test for table test;
CREATE PUBLICATION

replication=# \password
Enter new password for user "postgres":
Enter it again:

replication=#
```
##### Сервер 2
Создаем публикацию на test2 и так же задаем пароль
```
anton@pg-homework09-02:~$ sudo -u postgres psql
psql (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
Type "help" for help.

postgres=# \c replication
You are now connected to database "replication" as user "postgres".

replication=# create publication pub_test2 for table test2;
CREATE PUBLICATION

replication=# \password
Enter new password for user "postgres":
Enter it again:

replication=#
```

##### Сервер 1
Подписываемся на опубликованную таблицу на сервере 2
```
replication=# create subscription sub_test2 connection 'host=10.129.0.19 port=5432 user=postgres password=123456 dbname=replication' publication pub_test2 with (copy_data=true);
NOTICE:  created replication slot "sub_test2" on publisher
CREATE SUBSCRIPTION

replication=#
```
##### Сервер 2
Подписываемся на опубликованную таблицу на сервере 1
```
replication=# create subscription sub_test connection 'host=10.129.0.14 port=5432 user=postgres password=123456 dbname=replication' publication pub_test with (copy_data=true);
NOTICE:  created replication slot "sub_test" on publisher
CREATE SUBSCRIPTION

replication=#
```
##### Сервер 3
Подписываемся на таблицу 2 на сервере 2
```
create subscription sub_test2 connection 'host=10.129.0.19 port=5432 user=postgres password=123456 dbname=replication' publication pub_test2 with (copy_data=true);

ERROR:  could not create replication slot "sub_test2": ERROR:  replication slot "sub_test2" already exists
```
Получаем ошибку т.к. данный слот репликации уже используется на сервером 1. Пробуем подписаться с другим именем
```
replication=# create subscription sub_test2_server3 connection 'host=10.129.0.19 port=5432 user=postgres password=123
456 dbname=replication' publication pub_test2 with (copy_data=true);
NOTICE:  created replication slot "sub_test2_server3" on publisher
CREATE SUBSCRIPTION

replication=# create subscription sub_test_server3 connection 'host=10.129.0.14 port=5432 user=postgres password=1234
56 dbname=replication' publication pub_test with (copy_data=true);
NOTICE:  created replication slot "sub_test_server3" on publisher
CREATE SUBSCRIPTION
```
#### Проверка работы репликации
Делаем вставку на сервере 1 в таблицу test
```
replication=# insert into test(name) values('test_insert_server_1');
INSERT 0 1
```
Делаем селект на 2м и 3м сервере из таблицы тест
```
replication=# select * from test;
 id |         name
----+----------------------
  1 | test_insert_server_1
(1 row)
```
Данные появились на обоих серверах

Если сделать в ставку в таблицу тест2 на втором сервере
```
replication=# insert into test2(name) values('server_2_insert');
INSERT 0 1
```
Аналогичным образом данные появятся в таблице test2 на первом и третьем сервере
```
replication=# select * from test2;
 id |      name
----+-----------------
  1 | server_2_insert
(1 row)
```
