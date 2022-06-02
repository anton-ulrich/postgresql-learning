# Домашняя работа 7
## Механизм блокировок

Создал инстанс VM e2-medium в GCP с именем pg-homework07 обновил пакеты и установил postgresql-14
```
sudo apt-get update && sudo apt-get upgrade -y
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt-get update && sudo apt-get install -y postgresql-14
```

Подключаемся через psql настраиваем логирование блокировок
```
anton@pg-homework07:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

anton@pg-homework07:~$ sudo -u postgres psql
psql (14.3 (Ubuntu 14.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# alter system set log_lock_waits to 'on';
ALTER SYSTEM

postgres=# alter system set deadlock_timeout to '200ms';
ALTER SYSTEM

postgres=# \q

anton@pg-homework07:~$ sudo pg_ctlcluster 14 main restart;
```

Создаю таблицу и заполняю её данными
```
postgres=# create table test(id serial, value int);
CREATE TABLE
postgres=# insert into test values (1),(2),(3);
INSERT 0 3
postgres=#
```

Запускаем несколько сеансов и пробуем выполнить UPDATE одной и той же записи в разных сеансах

Сеанс 1
```
postgres=# begin;
BEGIN
postgres=*# update test set value=10 where id=1;
UPDATE 1
postgres=*#
```
Сеанс 2
```
postgres=# begin;
BEGIN
postgres=*# update test set value=100 where id=1;
```
Сеанс 3
```
postgres=# begin;
BEGIN
postgres=*# update test set value=1000 where id=1;
```
Во втором и третьем сеанса приглашение командной строки не появляется т.к. возникла ситуация блокировки и сеансы ждут пока сеанс №1 завершит транзакцию.

Изучим возникшие блокировки. Для этого делаем запрос в представлении pg_locks
```
postgres=*# SELECT * FROM pg_locks;
   locktype    | database | relation | page | tuple | virtualxid | transactionid | classid | objid | objsubid | virtualtransaction |  pid  |       mode       | granted | fastpath |           waitstart
---------------+----------+----------+------+-------+------------+---------------+---------+-------+----------+--------------------+-------+------------------+---------+----------+-------------------------------
 relation      |    13726 |    16385 |      |       |            |               |         |       |          | 5/5                | 26708 | RowExclusiveLock | t       | t        |
 virtualxid    |          |          |      |       | 5/5        |               |         |       |          | 5/5                | 26708 | ExclusiveLock    | t       | t        |
 relation      |    13726 |    16385 |      |       |            |               |         |       |          | 4/6                | 26704 | RowExclusiveLock | t       | t        |
 virtualxid    |          |          |      |       | 4/6        |               |         |       |          | 4/6                | 26704 | ExclusiveLock    | t       | t        |
 relation      |    13726 |    12290 |      |       |            |               |         |       |          | 3/21               | 26699 | AccessShareLock  | t       | t        |
 relation      |    13726 |    16385 |      |       |            |               |         |       |          | 3/21               | 26699 | RowExclusiveLock | t       | t        |
 virtualxid    |          |          |      |       | 3/21       |               |         |       |          | 3/21               | 26699 | ExclusiveLock    | t       | t        |
 transactionid |          |          |      |       |            |           745 |         |       |          | 4/6                | 26704 | ShareLock        | f       | f        | 2022-06-02 09:41:12.431978+00
 transactionid |          |          |      |       |            |           747 |         |       |          | 5/5                | 26708 | ExclusiveLock    | t       | f        |
 tuple         |    13726 |    16385 |    0 |    13 |            |               |         |       |          | 4/6                | 26704 | ExclusiveLock    | t       | f        |
 transactionid |          |          |      |       |            |           746 |         |       |          | 4/6                | 26704 | ExclusiveLock    | t       | f        |
 tuple         |    13726 |    16385 |    0 |    13 |            |               |         |       |          | 5/5                | 26708 | ExclusiveLock    | f       | f        | 2022-06-02 09:41:54.178101+00
 transactionid |          |          |      |       |            |           745 |         |       |          | 3/21               | 26699 | ExclusiveLock    | t       | f        |
(13 rows)
```
Мы видим что сеанс 26699 наложил эксклюзивную блокировку на таблицу test, а сеансы 26708 и 26704 встали в очередь и ожидают снятия блокировки.

После ввода команды END в первом сеансе второй сеанс завершил ожидание и обновил данные, но третий сеанс остается заблокированным ожидая завершения транзакции во втором сеансе

Наш запрос показывает теперь новую картину
```
postgres=# SELECT * FROM pg_locks;
   locktype    | database | relation | page | tuple | virtualxid | transactionid | classid | objid | objsubid | virtualtransaction |  pid  |       mode       | granted | fastpath |          waitstart
---------------+----------+----------+------+-------+------------+---------------+---------+-------+----------+--------------------+-------+------------------+---------+----------+------------------------------
 relation      |    13726 |    16385 |      |       |            |               |         |       |          | 5/5                | 26708 | RowExclusiveLock | t       | t        |
 virtualxid    |          |          |      |       | 5/5        |               |         |       |          | 5/5                | 26708 | ExclusiveLock    | t       | t        |
 relation      |    13726 |    16385 |      |       |            |               |         |       |          | 4/6                | 26704 | RowExclusiveLock | t       | t        |
 virtualxid    |          |          |      |       | 4/6        |               |         |       |          | 4/6                | 26704 | ExclusiveLock    | t       | t        |
 relation      |    13726 |    12290 |      |       |            |               |         |       |          | 3/22               | 26699 | AccessShareLock  | t       | t        |
 virtualxid    |          |          |      |       | 3/22       |               |         |       |          | 3/22               | 26699 | ExclusiveLock    | t       | t        |
 transactionid |          |          |      |       |            |           747 |         |       |          | 5/5                | 26708 | ExclusiveLock    | t       | f        |
 transactionid |          |          |      |       |            |           746 |         |       |          | 4/6                | 26704 | ExclusiveLock    | t       | f        |
 transactionid |          |          |      |       |            |           746 |         |       |          | 5/5                | 26708 | ShareLock        | f       | f        | 2022-06-02 09:44:02.19391+00
(9 rows)
```

Завершения транзакции во втором сеансе третий сеанс смог обновить данные.

При выборке данных ожидаемо получаем значение записи с ID 1 равным 1000
```
postgres=# select * from test;
 id | value
----+-------
  2 |
  3 |
  1 |  1000
(3 rows)
```

Смотрим лог и видим наши блокировки
```
anton@pg-homework07:~$ tail -n 20 /var/log/postgresql/postgresql-14-main.log
2022-06-02 09:40:25.271 UTC [26708] postgres@postgres STATEMENT:  update test set value=1000 where id=1;
2022-06-02 09:41:12.632 UTC [26704] postgres@postgres LOG:  process 26704 still waiting for ShareLock on transaction 745 after 200.197 ms
2022-06-02 09:41:12.632 UTC [26704] postgres@postgres DETAIL:  Process holding the lock: 26699. Wait queue: 26704.
2022-06-02 09:41:12.632 UTC [26704] postgres@postgres CONTEXT:  while updating tuple (0,13) in relation "test"
2022-06-02 09:41:12.632 UTC [26704] postgres@postgres STATEMENT:  update test set value=100 where id=1;
2022-06-02 09:41:54.378 UTC [26708] postgres@postgres LOG:  process 26708 still waiting for ExclusiveLock on tuple (0,13) of relation 16385 of database 13726 after 200.224 ms
2022-06-02 09:41:54.378 UTC [26708] postgres@postgres DETAIL:  Process holding the lock: 26704. Wait queue: 26708.
2022-06-02 09:41:54.378 UTC [26708] postgres@postgres STATEMENT:  update test set value=1000 where id=1;
2022-06-02 09:44:02.193 UTC [26704] postgres@postgres LOG:  process 26704 acquired ShareLock on transaction 745 after 169761.233 ms
2022-06-02 09:44:02.193 UTC [26704] postgres@postgres CONTEXT:  while updating tuple (0,13) in relation "test"
2022-06-02 09:44:02.193 UTC [26704] postgres@postgres STATEMENT:  update test set value=100 where id=1;
2022-06-02 09:44:02.193 UTC [26708] postgres@postgres LOG:  process 26708 acquired ExclusiveLock on tuple (0,13) of relation 16385 of database 13726 after 128015.762 ms
2022-06-02 09:44:02.193 UTC [26708] postgres@postgres STATEMENT:  update test set value=1000 where id=1;
2022-06-02 09:44:02.394 UTC [26708] postgres@postgres LOG:  process 26708 still waiting for ShareLock on transaction 746 after 200.161 ms
2022-06-02 09:44:02.394 UTC [26708] postgres@postgres DETAIL:  Process holding the lock: 26704. Wait queue: 26708.
2022-06-02 09:44:02.394 UTC [26708] postgres@postgres CONTEXT:  while rechecking updated tuple (0,14) in relation "test"
2022-06-02 09:44:02.394 UTC [26708] postgres@postgres STATEMENT:  update test set value=1000 where id=1;
2022-06-02 09:45:17.075 UTC [26708] postgres@postgres LOG:  process 26708 acquired ShareLock on transaction 746 after 74881.594 ms
2022-06-02 09:45:17.075 UTC [26708] postgres@postgres CONTEXT:  while rechecking updated tuple (0,14) in relation "test"
2022-06-02 09:45:17.075 UTC [26708] postgres@postgres STATEMENT:  update test set value=1000 where id=1;
```
Пробуем вызвать взаимную блокировку.

Создаем 2 сеанса.

сеанс 1
```
postgres=# begin;
BEGIN
postgres=*# update test set value=1 where id=2;
UPDATE 1
```

сеанс 2
```
postgres=# begin;
BEGIN
postgres=*# update test set value=2 where id=3;
UPDATE 1
postgres=*# update test set value=22 where id=2;
UPDATE 1
```

теперь в сеансе 1 пытаемся обновить запись которую уже обновляет сеанс 2 и получаем взаимоблокировку
```
postgres=*# update test set value=11 where id=3;
ERROR:  deadlock detected
DETAIL:  Process 27088 waits for ShareLock on transaction 749; blocked by process 27196.
Process 27196 waits for ShareLock on transaction 748; blocked by process 27088.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,3) in relation "test"
```

Смотрим логи
```
anton@pg-homework07:~$ tail -n 12 /var/log/postgresql/postgresql-14-main.log
2022-06-02 10:37:01.410 UTC [27088] postgres@postgres STATEMENT:  update test set value=11 where id=3;
2022-06-02 10:37:01.410 UTC [27088] postgres@postgres ERROR:  deadlock detected
2022-06-02 10:37:01.410 UTC [27088] postgres@postgres DETAIL:  Process 27088 waits for ShareLock on transaction 749; blocked by process 27196.
        Process 27196 waits for ShareLock on transaction 748; blocked by process 27088.
        Process 27088: update test set value=11 where id=3;
        Process 27196: update test set value=22 where id=2;
2022-06-02 10:37:01.410 UTC [27088] postgres@postgres HINT:  See server log for query details.
2022-06-02 10:37:01.410 UTC [27088] postgres@postgres CONTEXT:  while updating tuple (0,3) in relation "test"
2022-06-02 10:37:01.410 UTC [27088] postgres@postgres STATEMENT:  update test set value=11 where id=3;
2022-06-02 10:37:01.410 UTC [27196] postgres@postgres LOG:  process 27196 acquired ShareLock on transaction 748 after 17880.589 ms
2022-06-02 10:37:01.410 UTC [27196] postgres@postgres CONTEXT:  while updating tuple (0,2) in relation "test"
2022-06-02 10:37:01.410 UTC [27196] postgres@postgres STATEMENT:  update test set value=22 where id=2;
```
В логах зафиксирована ситуация дедлока и и расписано какой из процессов и что бытался обновить в момент взаимоблокировки
