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
