# Домашняя работа 6
## Работа с журналами

Создал инстанс VM e2-medium в GCP с именем pg-homework06 обновил пакеты и установил postgresql-14
```
sudo apt-get update && sudo apt-get upgrade -y
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt-get update
sudo apt-get install postgresql-14
```

Заходим в psql
```
anton@pg-homework06:~$ sudo -u postgres psql
psql (14.3 (Ubuntu 14.3-1.pgdg20.04+1))
Type "help" for help.

postgres=#
```

Изменяем значение создания контрольных точек на 30 секунд и включаем логирование контрольных точек
```
alter system set log_checkpoints to 'on';
alter system set checkpoint_timeout to '30s';
```

перезагружаем кластер для применения изменений
```
anton@pg-homework06:~$ sudo pg_ctlcluster 14 main restart
anton@pg-homework06:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/
postgresql/postgresql-14-main.log
anton@pg-homework06:~$
```

получаем текущую точку журнала
```
anton@pg-homework06:~$ sudo -u postgres psql -c 'SELECT pg_current_wal_insert_lsn();'
 pg_current_wal_insert_lsn
---------------------------
 0/788116F0
(1 row)
```

Логинимся под пользователем постгрес, инициализируем pgbench, запускаем нагрузочное тестирование на 10 минут и сразу же после тестирования получаем новую точку журнала
```
anton@pg-homework06:~$ sudo su postgres
postgres@pg-homework06:/home/anton$ pgbench -i postgres
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.11 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.46 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.26 s, vacuum 0.10 s, primary keys 0.08 s).

postgres@pg-homework06:/home/anton$ pgbench -c8 -P 60 -T 600 -U postgres postgres && psql -c 'SELECT pg_current_wal_insert_lsn();'
pgbench (14.3 (Ubuntu 14.3-1.pgdg20.04+1))
starting vacuum...end.
progress: 60.0 s, 643.3 tps, lat 12.429 ms stddev 7.998
progress: 120.0 s, 632.0 tps, lat 12.657 ms stddev 7.908
progress: 180.0 s, 642.9 tps, lat 12.438 ms stddev 7.234
progress: 240.0 s, 633.5 tps, lat 12.630 ms stddev 7.005
progress: 300.0 s, 625.8 tps, lat 12.781 ms stddev 6.793
progress: 360.0 s, 625.2 tps, lat 12.794 ms stddev 6.719
progress: 420.0 s, 633.6 tps, lat 12.625 ms stddev 8.052
progress: 480.0 s, 612.3 tps, lat 13.063 ms stddev 7.859
progress: 540.0 s, 619.5 tps, lat 12.912 ms stddev 7.473
progress: 600.0 s, 590.1 tps, lat 13.555 ms stddev 7.448
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 375498
latency average = 12.781 ms
latency stddev = 7.471 ms
initial connection time = 13.957 ms
tps = 625.823560 (without initial connection time)
 pg_current_wal_insert_lsn
---------------------------
 0/94246F08
(1 row)
```

Заходим в PSQL и вычисляем объем журналов за это время и получаем среднее значение размера одной контрольной точки
```
postgres@pg-homework06:/home/anton$ psql
psql (14.3 (Ubuntu 14.3-1.pgdg20.04+1))
Type "help" for help.

ppostgres=# SELECT pg_size_pretty('0/94246F08'::pg_lsn - '0/788116F0'::pg_lsn);
 pg_size_pretty
----------------
 442 MB
(1 row)

postgres=# SELECT pg_size_pretty(('0/94246F08'::pg_lsn - '0/788116F0'::pg_lsn)/20);
 pg_size_pretty
----------------
 22 MB
(1 row)
```
Получаем что за все время было сгенирировано 442 MB и т.к. за 10 минут прошло 20 контрольных точек то средний размер журнала получается около 22 MB

При проведении тестирования каких либо отклонений в времени создании контрольных точек зафиксировано не было. Теоретически при больших нагрузках оно возможно, т.к. будет писаться большой объем данных на диск, но в моем случае подобных отклонений зафиксировано небыло(пытался воспроизвести несколько раз)(в примере тест начался примерно в 16:16:40 UTC)
```
2022-06-01 16:16:05.024 UTC [18390] LOG:  checkpoint starting: time
2022-06-01 16:16:32.069 UTC [18390] LOG:  checkpoint complete: wrote 1872 buffers (11.4%); 0 WAL file(s) added, 0 removed, 2 recycled; write=27.004 s, sync=0.010 s, total=27.046 s; sync files=8, longest=0.004 s, average=0.002 s; distance=22346 kB, estimate=23215 kB
2022-06-01 16:16:35.072 UTC [18390] LOG:  checkpoint starting: time
2022-06-01 16:17:02.095 UTC [18390] LOG:  checkpoint complete: wrote 1787 buffers (10.9%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.991 s, sync=0.008 s, total=27.024 s; sync files=15, longest=0.003 s, average=0.001 s; distance=23225 kB, estimate=23225 kB
2022-06-01 16:17:05.096 UTC [18390] LOG:  checkpoint starting: time
2022-06-01 16:17:32.036 UTC [18390] LOG:  checkpoint complete: wrote 1825 buffers (11.1%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.889 s, sync=0.025 s, total=26.941 s; sync files=13, longest=0.009 s, average=0.002 s; distance=19843 kB, estimate=22887 kB
2022-06-01 16:17:35.039 UTC [18390] LOG:  checkpoint starting: time
2022-06-01 16:18:02.069 UTC [18390] LOG:  checkpoint complete: wrote 1974 buffers (12.0%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.991 s, sync=0.011 s, total=27.030 s; sync files=14, longest=0.004 s, average=0.001 s; distance=22892 kB, estimate=22892 kB
2022-06-01 16:18:05.072 UTC [18390] LOG:  checkpoint starting: time
2022-06-01 16:18:32.094 UTC [18390] LOG:  checkpoint complete: wrote 1870 buffers (11.4%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.990 s, sync=0.007 s, total=27.023 s; sync files=8, longest=0.003 s, average=0.001 s; distance=22564 kB, estimate=22859 kB
2022-06-01 16:18:35.096 UTC [18390] LOG:  checkpoint starting: time
2022-06-01 16:19:02.112 UTC [18390] LOG:  checkpoint complete: wrote 2053 buffers (12.5%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.992 s, sync=0.007 s, total=27.017 s; sync files=16, longest=0.004 s, average=0.001 s; distance=22816 kB, estimate=22855 kB
2022-06-01 16:19:05.115 UTC [18390] LOG:  checkpoint starting: time
2022-06-01 16:19:32.048 UTC [18390] LOG:  checkpoint complete: wrote 1880 buffers (11.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.899 s, sync=0.007 s, total=26.933 s; sync files=8, longest=0.003 s, average=0.001 s; distance=22364 kB, estimate=22806 kB
2022-06-01 16:19:35.051 UTC [18390] LOG:  checkpoint starting: time
2022-06-01 16:20:02.069 UTC [18390] LOG:  checkpoint complete: wrote 2046 buffers (12.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.997 s, sync=0.007 s, total=27.019 s; sync files=15, longest=0.003 s, average=0.001 s; distance=22881 kB, estimate=22881 kB
2022-06-01 16:20:05.071 UTC [18390] LOG:  checkpoint starting: time
2022-06-01 16:20:32.096 UTC [18390] LOG:  checkpoint complete: wrote 1874 buffers (11.4%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.991 s, sync=0.008 s, total=27.025 s; sync files=6, longest=0.003 s, average=0.002 s; distance=22784 kB, estimate=22872 kB
2022-06-01 16:20:35.099 UTC [18390] LOG:  checkpoint starting: time
2022-06-01 16:21:02.025 UTC [18390] LOG:  checkpoint complete: wrote 2056 buffers (12.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.892 s, sync=0.008 s, total=26.927 s; sync files=14, longest=0.004 s, average=0.001 s; distance=22664 kB, estimate=22851 kB
2022-06-01 16:21:05.028 UTC [18390] LOG:  checkpoint starting: time
2022-06-01 16:21:32.047 UTC [18390] LOG:  checkpoint complete: wrote 1876 buffers (11.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.985 s, sync=0.011 s, total=27.020 s; sync files=9, longest=0.005 s, average=0.002 s; distance=22647 kB, estimate=22830 kB
2022-06-01 16:21:35.047 UTC [18390] LOG:  checkpoint starting: time
2022-06-01 16:22:02.074 UTC [18390] LOG:  checkpoint complete: wrote 2047 buffers (12.5%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.992 s, sync=0.006 s, total=27.027 s; sync files=14, longest=0.003 s, average=0.001 s; distance=22712 kB, estimate=22819 kB
2022-06-01 16:22:05.076 UTC [18390] LOG:  checkpoint starting: time
2022-06-01 16:22:32.083 UTC [18390] LOG:  checkpoint complete: wrote 1887 buffers (11.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.984 s, sync=0.007 s, total=27.008 s; sync files=9, longest=0.004 s, average=0.001 s; distance=22527 kB, estimate=22789 kB
2022-06-01 16:22:35.084 UTC [18390] LOG:  checkpoint starting: time
2022-06-01 16:23:02.115 UTC [18390] LOG:  checkpoint complete: wrote 2041 buffers (12.5%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.991 s, sync=0.006 s, total=27.032 s; sync files=14, longest=0.004 s, average=0.001 s; distance=22777 kB, estimate=22788 kB
2022-06-01 16:23:05.115 UTC [18390] LOG:  checkpoint starting: time
2022-06-01 16:23:32.036 UTC [18390] LOG:  checkpoint complete: wrote 1876 buffers (11.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.888 s, sync=0.008 s, total=26.921 s; sync files=9, longest=0.004 s, average=0.001 s; distance=22326 kB, estimate=22742 kB
2022-06-01 16:23:35.039 UTC [18390] LOG:  checkpoint starting: time
2022-06-01 16:24:02.053 UTC [18390] LOG:  checkpoint complete: wrote 2044 buffers (12.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.989 s, sync=0.009 s, total=27.015 s; sync files=12, longest=0.005 s, average=0.001 s; distance=22743 kB, estimate=22743 kB
2022-06-01 16:24:05.056 UTC [18390] LOG:  checkpoint starting: time
2022-06-01 16:24:32.085 UTC [18390] LOG:  checkpoint complete: wrote 1883 buffers (11.5%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.997 s, sync=0.005 s, total=27.030 s; sync files=9, longest=0.005 s, average=0.001 s; distance=22635 kB, estimate=22732 kB
2022-06-01 16:24:35.087 UTC [18390] LOG:  checkpoint starting: time
2022-06-01 16:25:02.109 UTC [18390] LOG:  checkpoint complete: wrote 2281 buffers (13.9%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.995 s, sync=0.010 s, total=27.022 s; sync files=15, longest=0.005 s, average=0.001 s; distance=22221 kB, estimate=22681 kB
2022-06-01 16:25:05.111 UTC [18390] LOG:  checkpoint starting: time
2022-06-01 16:25:32.037 UTC [18390] LOG:  checkpoint complete: wrote 1892 buffers (11.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.888 s, sync=0.008 s, total=26.926 s; sync files=9, longest=0.005 s, average=0.001 s; distance=22485 kB, estimate=22661 kB
2022-06-01 16:25:35.039 UTC [18390] LOG:  checkpoint starting: time
2022-06-01 16:26:02.056 UTC [18390] LOG:  checkpoint complete: wrote 2060 buffers (12.6%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.989 s, sync=0.009 s, total=27.017 s; sync files=12, longest=0.004 s, average=0.001 s; distance=22772 kB, estimate=22772 kB
2022-06-01 16:26:05.059 UTC [18390] LOG:  checkpoint starting: time
2022-06-01 16:26:32.074 UTC [18390] LOG:  checkpoint complete: wrote 1850 buffers (11.3%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.992 s, sync=0.006 s, total=27.015 s; sync files=9, longest=0.004 s, average=0.001 s; distance=21793 kB, estimate=22674 kB
2022-06-01 16:26:35.075 UTC [18390] LOG:  checkpoint starting: time
2022-06-01 16:27:02.113 UTC [18390] LOG:  checkpoint complete: wrote 2278 buffers (13.9%); 0 WAL file(s) added, 0 removed, 2 recycled; write=27.001 s, sync=0.010 s, total=27.038 s; sync files=13, longest=0.005 s, average=0.001 s; distance=22336 kB, estimate=22640 kB
2022-06-01 16:27:35.146 UTC [18390] LOG:  checkpoint starting: time
2022-06-01 16:28:02.066 UTC [18390] LOG:  checkpoint complete: wrote 1850 buffers (11.3%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.902 s, sync=0.004 s, total=26.921 s; sync files=12, longest=0.003 s, average=0.001 s; distance=18258 kB, estimate=22202 kB
2022-06-01 16:33:05.367 UTC [18390] LOG:  checkpoint starting: time
2022-06-01 16:33:32.080 UTC [18390] LOG:  checkpoint complete: wrote 1281 buffers (7.8%); 0 WAL file(s) added, 0 removed, 0 recycled; write=26.683 s, sync=0.017 s, total=26.713 s; sync files=16, longest=0.005 s, average=0.002 s; distance=10604 kB, estimate=21042 kB
2022-06-01 16:33:35.083 UTC [18390] LOG:  checkpoint starting: time
2022-06-01 16:34:02.103 UTC [18390] LOG:  checkpoint complete: wrote 1923 buffers (11.7%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.984 s, sync=0.008 s, total=27.021 s; sync files=14, longest=0.004 s, average=0.001 s; distance=23763 kB, estimate=23763 kB
2022-06-01 16:34:05.105 UTC [18390] LOG:  checkpoint starting: time
2022-06-01 16:34:32.021 UTC [18390] LOG:  checkpoint complete: wrote 1899 buffers (11.6%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.884 s, sync=0.008 s, total=26.917 s; sync files=8, longest=0.005 s, average=0.001 s; distance=23844 kB, estimate=23844 kB
2022-06-01 16:34:35.023 UTC [18390] LOG:  checkpoint starting: time
2022-06-01 16:35:02.045 UTC [18390] LOG:  checkpoint complete: wrote 2112 buffers (12.9%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.991 s, sync=0.007 s, total=27.022 s; sync files=15, longest=0.006 s, average=0.001 s; distance=23817 kB, estimate=23841 kB
```

Проверяем TPS в синхронном режиме. Проверяем что он включен(включен по умолчанию) и запускаем тест
```
postgres@pg-homework06:~$ pgbench -c8 -T 600 -U postgres postgres
pgbench (14.3 (Ubuntu 14.3-1.pgdg20.04+1))
starting vacuum...end.

transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 379088
latency average = 12.662 ms
initial connection time = 15.923 ms
tps = 631.808969 (without initial connection time)
```
получаем TPS 631,8

Выключаем синхронный режим и снова запускаем тест
```
anton@pg-homework06:~$ sudo -u postgres psql -c 'ALTER SYSTEM SET synchronous_commit TO off;'
ALTER SYSTEM
anton@pg-homework06:~$ sudo pg_ctlcluster 14 main restart
anton@pg-homework06:~$ sudo -u postgres psql -c 'show synchronous_commit;'
 synchronous_commit
--------------------
 off
(1 row)

anton@pg-homework06:~$ sudo -u postgres pgbench -c8 -T 600 -U postgres postgres
pgbench (14.3 (Ubuntu 14.3-1.pgdg20.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 880030
latency average = 5.454 ms
initial connection time = 16.615 ms
tps = 1466.720517 (without initial connection time)
```
TPS вырос до 1466 т.к. сервер больше не ждет подтверждения записи контрольной точки на диск

создаем новый кластер с контролем четности таблиц. создадим в нем таблицу и заполним её данными
```
anton@pg-homework06:~$ sudo pg_createcluster 14 test_cluster -- --data-checksums
Creating new PostgreSQL cluster 14/test_cluster ...
/usr/lib/postgresql/14/bin/initdb -D /var/lib/postgresql/14/test_cluster --auth-local peer --auth-host scram-sha-256 --no-instructions --data-checksums
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "C.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are enabled.

fixing permissions on existing directory /var/lib/postgresql/14/test_cluster ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Etc/UTC
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Ver Cluster      Port Status Owner    Data directory                      Log file
14  test_cluster 5433 down   postgres /var/lib/postgresql/14/test_cluster /var/log/postgresql/postgresql-14-test_cluster.log
anton@pg-homework06:~$ pg_lsclusters
Ver Cluster      Port Status Owner    Data directory                      Log file
14  main         5432 online postgres /var/lib/postgresql/14/main         /var/log/postgresql/postgresql-14-main.log
14  test_cluster 5433 down   postgres /var/lib/postgresql/14/test_cluster /var/log/postgresql/postgresql-14-test_cluster.log
anton@pg-homework06:~$ sudo pg_ctlcluster 14 test_cluster start
anton@pg-homework06:~$ sudo -u postgres psql -p 5433
psql (14.3 (Ubuntu 14.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# create table test(x int);
CREATE TABLE                    ^
postgres=# insert into test values(1),(2),(3);
INSERT 0 3
postgres=# select * from test;
 x
---
 1
 2
 3
(3 rows)
```
Проверим расположение таблицы, остановим кластер и изменем несколько бит в файле таблицы и пытаемся выбрать из неё данные
```
postgres=# select pg_relation_filepath('test');
 pg_relation_filepath
----------------------
 base/13726/16384
(1 row)

postgres=# \q

anton@pg-homework06:~$ sudo pg_ctlcluster 14 test_cluster stop

anton@pg-homework06:~$ sudo pg_lsclusters
Ver Cluster      Port Status Owner    Data directory                      Log file
14  main         5432 online postgres /var/lib/postgresql/14/main         /var/log/postgresql/postgresql-14-main.log
14  test_cluster 5433 down   postgres /var/lib/postgresql/14/test_cluster /var/log/postgresql/postgresql-14-test_cluster.log

anton@pg-homework06:~$ sudo su postgres

postgres@pg-homework06:/home/anton$ cd /var/lib/postgresql/14/test_cluster/base/13726/

postgres@pg-homework06:~/14/test_cluster/base/13726$ vim 16384

postgres@pg-homework06:~/14/test_cluster/base/13726$ exit

anton@pg-homework06:~$ sudo pg_ctlcluster 14 test_cluster start

anton@pg-homework06:~$ sudo pg_lsclusters
Ver Cluster      Port Status Owner    Data directory                      Log file
14  main         5432 online postgres /var/lib/postgresql/14/main         /var/log/postgresql/postgresql-14-main.log
14  test_cluster 5433 online postgres /var/lib/postgresql/14/test_cluster /var/log/postgresql/postgresql-14-test_cluster.log

anton@pg-homework06:~$ sudo -u postgres psql -p 5433
psql (14.3 (Ubuntu 14.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | test | table | postgres
(1 row)

postgres=# select * from test;
WARNING:  page verification failed, calculated checksum 25394 but expected 38542
ERROR:  invalid page in block 0 of relation base/13726/16384
postgres=#
```
Мы получаем ошибку т.к. контрольная сумма файла таблицы не совпадает с ожидаемой

игнорировать ошибку можно установив параметр ignore_checksum_failure в on
```
postgres=# alter system set ignore_checksum_failure to 'on';
ALTER SYSTEM
postgres=# \q
anton@pg-homework06:~$ sudo pg_ctlcluster 14 test_cluster restart
anton@pg-homework06:~$ sudo -u postgres psql -p 5433
psql (14.3 (Ubuntu 14.3-1.pgdg20.04+1))
postgres=#

postgres=# show ignore_checksum_failure;
 ignore_checksum_failure
-------------------------
 on
(1 row)

postgres=# select * from test;
WARNING:  page verification failed, calculated checksum 25394 but expected 38542
ERROR:  invalid page in block 0 of relation base/13726/16384
```
Теоретически должно было получится он похоже при вставке повредил структуру таблицы из за чего данные всеравно не читаются
