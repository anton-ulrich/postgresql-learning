# Домашняя работа 4
## Работа с базами данных, пользователями и правами

Развернул виртуальную машину в GCP с именем pg-homework04.
Обновил пакеты и установил postgresql 13(либо в инструкции опечатка, либо для ДЗ реально нужен 13й постгресс)
```
sudo apt-get update
sudo apt-get upgrade
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt-get update
sudo apt-get install postgresql-13
```

Проверяю что кластер стартовал и подключаюсь юзером postgres через psql
```
anton@pg-homework04:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
13  main    5432 online postgres /var/lib/postgresql/13/main /var/log/
postgresql/postgresql-13-main.log

anton@pg-homework04:~$ sudo -u postgres psql
psql (13.7 (Ubuntu 13.7-1.pgdg20.04+1))
Type "help" for help.

postgres=#
```

Создал новую базу testdb, подключился к ней пользователем postgres создал схему testnm, создал таблицу t1 c одним полем с1 типа integer и вставил в данную таблицу запись со значением 1
```
  postgres=# create database testdb;
CREATE DATABASE
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# create schema testnm;
CREATE SCHEMA
testdb=# create table t1(c1 integer);
CREATE TABLE
testdb=# insert into t1 values(1);
INSERT 0 1
```

Создал роль readonly, дал ей право на подключение к БД testdb, дал право на использование новой схему testnm и и дал ей право select для всех таблиц схемы testnm
```
testdb=# CREATE role readonly;
CREATE ROLE
testdb=# grant connect on database testdb to readonly;
GRANT
  testdb=# grant usage on schema testnm to readonly;
GRANT
testdb=# grant SELECT on all TABLEs in SCHEMA testnm TO readonly;
GRANT
```

Создаю пользователя testread  с паролем, и даю пользователю роль readonly. пробую подключиться и получаю ошибку
```
testdb=# create user testread with password 'test123';
CREATE ROLE
testdb=# grant readonly to testread;
GRANT ROLE
testdb=# \c testdb testread
connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  Peer authentication failed for user "testread"
Previous connection kept
```

Проблема вызвана настройками в pg_hba. Что бы не лезть в настройки явно подключаемся через IP 127.0.0.1
```
testdb=# \c testdb testread 127.0.0.1
Password for user testread:
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
You are now connected to database "testdb" as user "testread" on host "127.0.0.1" at port "5432".
```

Пытаемся выполнить Select *  для созданной таблицы и получаем ошибку доступ запрещен. Если посмотреть таблицы в БД то видим что таблица создана в схеме t1, в то время как роль readonly, к которой относится наш тестовый юзер, имеет права select только в схеме testns
```
testdb=> select * from t1;
ERROR:  permission denied for table t1

testdb=> \dt
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 public | t1   | table | postgres
(1 row)
```

Что бы это устранить необходимо пересоздать таблицу в схеме testns. Для этого подключаемся к базе пользователем postgres(нужно будет выйти и зайти из psql т.к. будет просить пароль от пользователя postgres) и пересоздаем таблицу в нужной схеме
```
testdb-> \q

anton@pg-homework04:~$ sudo -u postgres psql
psql (13.7 (Ubuntu 13.7-1.pgdg20.04+1))
Type "help" for help.

postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".

testdb=# \dt
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 public | t1   | table | postgres
(1 row)

testdb=# drop table t1;
DROP TABLE

testdb=# create table testnm.t1(c1 integer);
CREATE TABLE

testdb=# insert into testnm.t1 values(1);
INSERT 0 1
```

Заходим под пользователем testread в базу testdb и снова пытаемся сделать SELECT
```
testdb=# \c testdb testread localhost
Password for user testread:
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
You are now connected to database "testdb" as user "testread" on host "localhost" (address "127.0.0.1") at port "5432".

testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
```
Ошибка возникла т.к. новая таблица была создана после назначения прав для роли

Снова подключаемся пользователем postgres, меняем разрешения и пытаемся сделать select
```
testdb=> \q

anton@pg-homework04:~$ sudo -u postgres psql
psql (13.7 (Ubuntu 13.7-1.pgdg20.04+1))
Type "help" for help.

postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".

testdb=# ALTER default privileges in SCHEMA testnm grant SELECT on TABLEs to readonly;
ALTER DEFAULT PRIVILEGES

testdb=# \c testdb testread localhost
Password for user testread:
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
You are now connected to database "testdb" as user "testread" on host "localhost" (address "127.0.0.1") at port "5432".

testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
```
Снова получаем доступ запрещен.

Проблема вызвана тем что новые разрешения будут действовать только для вновь созданных таблиц. Т.к. таблица уже существует то необходимо вновь назначить привилегии на конкретно эту таблицу или пересоздать её.

Меняем привилегии и снова пытаемся сделать select
```
testdb=> \q

anton@pg-homework04:~$ sudo -u postgres psql
psql (13.7 (Ubuntu 13.7-1.pgdg20.04+1))
Type "help" for help.

postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# grant SELECT on all TABLEs in SCHEMA testnm TO readonly;
GRANT

testdb=# \c testdb testread localhost;
Password for user testread:
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
You are now connected to database "testdb" as user "testread" on host "localhost" (address "127.0.0.1") at port "5432".

testdb=> select * from testnm.t1;
 c1
----
  1
(1 row)

```
Всё получилось, select прошел

Пытаемся создать таблицу от имени нового пользователя
```
testdb=> create table t2(c1 integer);
CREATE TABLE

testdb=> insert into t2 values (2);
INSERT 0 1
```
Команда прошла? хотя у пользователя нет прав create. Если посмотреть список таблиц то видно, что таблица создана в схеме public, а права в ней имеют все пользователи
```
testdb=> \dt
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 public | t2   | table | testread
(1 row)
```
Для того что бы убрать подобное поведение необходимо отозвать права Create у роли public из схемы public. Так же удаляем все привилегии у роли public для базы testdb
```
testdb-# \q
anton@pg-homework04:~$ sudo -u postgres psql
psql (13.7 (Ubuntu 13.7-1.pgdg20.04+1))
Type "help" for help.

postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".

testdb=# revoke CREATE on SCHEMA public FROM public;
REVOKE

testdb=# revoke all on DATABASE testdb FROM public;
REVOKE
```

Теперь пробуем подключиться к базе testdb пользователем testread, создать новую таблицу и сделать insert в существующую
```
testdb=# \c testdb testread localhost;
Password for user testread:
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
You are now connected to database "testdb" as user "testread" on host "localhost" (address "127.0.0.1") at port "5432".

testdb=> create table t3(c1 integer);
ERROR:  permission denied for schema public
LINE 1: create table t3(c1 integer);
                     ^
testdb=> insert into t2 values (2);
INSERT 0 1
```
Создать базу не удалось т.к. права на создание таблиц для схему public мы забрали.
INSERT же пройдет т.к. таблица уже создана, до того как были отозваны права, а явно права инсерт мы с данной таблицы не забирали, что подтверждает запрос ниже
```
testdb=> select * from information_schema.role_table_grants where grantee='testread';
 grantor  | grantee  | table_catalog | table_schema | table_name | privilege_type | is_grantable | with_hierarchy
----------+----------+---------------+--------------+------------+----------------+--------------+----------------
 testread | testread | testdb        | public       | t2         | INSERT         | YES          | NO
 testread | testread | testdb        | public       | t2         | SELECT         | YES          | YES
 testread | testread | testdb        | public       | t2         | UPDATE         | YES          | NO
 testread | testread | testdb        | public       | t2         | DELETE         | YES          | NO
 testread | testread | testdb        | public       | t2         | TRUNCATE       | YES          | NO
 testread | testread | testdb        | public       | t2         | REFERENCES     | YES          | NO
 testread | testread | testdb        | public       | t2         | TRIGGER        | YES          | NO
(7 rows)

```
