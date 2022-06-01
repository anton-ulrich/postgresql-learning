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
 0/1F75CE48
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
progress: 60.0 s, 601.4 tps, lat 13.293 ms stddev 7.980
progress: 120.0 s, 602.8 tps, lat 13.269 ms stddev 7.951
progress: 180.0 s, 611.8 tps, lat 13.074 ms stddev 7.243
progress: 240.0 s, 616.2 tps, lat 12.982 ms stddev 6.639
progress: 300.0 s, 621.3 tps, lat 12.874 ms stddev 6.181
progress: 360.0 s, 611.7 tps, lat 13.077 ms stddev 5.972
progress: 420.0 s, 603.4 tps, lat 13.256 ms stddev 6.005
progress: 480.0 s, 607.0 tps, lat 13.178 ms stddev 7.866
progress: 540.0 s, 635.0 tps, lat 12.595 ms stddev 6.958
progress: 600.0 s, 630.9 tps, lat 12.678 ms stddev 6.691
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 368504
latency average = 13.023 ms
latency stddev = 6.987 ms
initial connection time = 17.667 ms
tps = 614.173707 (without initial connection time)

 pg_current_wal_insert_lsn
---------------------------
 0/56F972C0
(1 row)
```
