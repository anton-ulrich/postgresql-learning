# Домашняя работа 8
## Нагрузочное тестирование и тюнинг PostgreSQL


Создал инстанс VM e2-medium в GCP с именем pg-homework08 обновил пакеты и установил postgresql-14
```
sudo apt-get update && sudo apt-get upgrade -y
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt-get update && sudo apt-get install -y postgresql-14
```

Заходим под пользователем postgres, инициализируем и запускаем pgbench с дефолтными параметрами сервера
```
anton@pg-homework08:~$ sudo su postgres

postgres@pg-homework08:/home/anton$ pgbench -i postgres
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.12 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.50 s (drop tables 0.01 s, create tables 0.04 s, client-side generate 0.29 s, vacuum 0.08 s, primary keys 0.08 s).

postgres@pg-homework08:/home/anton$ pgbench -c16 -P 60 -T 600 -U postgres postgres
pgbench (14.3 (Ubuntu 14.3-1.pgdg18.04+1))
starting vacuum...end.
progress: 60.0 s, 826.8 tps, lat 19.336 ms stddev 11.512
progress: 120.0 s, 813.5 tps, lat 19.667 ms stddev 12.210
progress: 180.0 s, 804.7 tps, lat 19.879 ms stddev 11.807
progress: 240.0 s, 843.4 tps, lat 18.970 ms stddev 11.428
progress: 300.0 s, 819.5 tps, lat 19.522 ms stddev 11.798
progress: 360.0 s, 823.0 tps, lat 19.439 ms stddev 12.000
progress: 420.0 s, 840.8 tps, lat 19.026 ms stddev 11.399
progress: 480.0 s, 818.2 tps, lat 19.553 ms stddev 11.548
progress: 540.0 s, 726.2 tps, lat 22.032 ms stddev 20.922
progress: 600.0 s, 552.4 tps, lat 28.964 ms stddev 32.661
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 16
number of threads: 1
duration: 600 s
number of transactions actually processed: 472122
latency average = 20.331 ms
latency stddev = 15.377 ms
initial connection time = 28.063 ms
tps = 786.865590 (without initial connection time)
```
Получаем средний TPS 786.

Пробуем настроить сервер через сервер https://pgtune.leopard.in.ua/. получаем следующее
![Screenshot](gptune.png)

Копируем предложенную конфигурацию в конец файла /etc/postgresql/14/main/postgresql.conf
