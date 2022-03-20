# Домашняя работа 1

1. Зарегистрировался в GCP. Создал проект postgres2022-19880518
2. Дал права Editor на проект пользователю ifti@yandex.ru
3. Создал инстанс pg-test
4. Добавил свой ssh ключ в MetaData
5. подключился по SSH под пользователем anton
6. Обновил систему:
```bash
sudo apt update
sudo apt upgrade
```
6. Установил postgres-14:
```bash
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt-get update
sudo apt-get install postgresql-14
```
7. Зашел в БД
```bash
sudo -u postgres psql
```
8. Во второй консоли так же подключился по ssh и зашел в postgres
9. В первой консоли создал БД iso и подключился к ней:
```sql
create database iso;  # создал БД
\l                    # вывел список доступных БД
\c iso;               # подключился к созданной БД
```
10. Посмотрел включен ли автокоммит и выключил его т.к. он был включен
```sql
\echo :AUTOCOMMIT     # Вывел текущеее значение AUTOCOMMIT(был в статусе on)
\set AUTOCOMMIT off   # отключил AUTOCOMMIT
\echo :AUTOCOMMIT     # Проверил что значение autocommit изменилось на off
```
11. Создал таблицу и заполнил её данными
```sql
create table persons(id serial, first_name text, second_name text);
insert into persons(first_name, second_name) values('ivan', 'ivanov');
insert into persons(first_name, second_name) values('petr', 'petrov');
commit;
```
12. Проверил текущий уровень изоляции транзакций
```sql
show transaction isolation level;
```
Результат выполнения
```
transaction_isolation 
-----------------------
read committed
(1 row)
```
Уровень изоляции транзакций установлен на стандартный уровень read committed

13. Начал новую транзакцию в первой консоли. Выполнил команду
```sql
insert into persons(first_name, second_name) values('sergey', 'sergeev');
```
При выполнении select *  во второй консоли новые данные не появились, т.к. транзакция еще не завершена, а Postgres на данном уровне не допускает грязное чтение данных

14. завершил транзакцию используя commit; в первой консоли. При выполнении select * во второй консоли появились новые данные в таблице, т.к. в первой консоли транзакция завершена

15. Изменил уровень изоляции транзакций в обеих консолях на repeatable read
```sql
set transaction isolation level repeatable read;    # изменения уровня на repeatable read
show transaction isolation level;                   # прочерка что уровень изменился
```

16. Добавил новую записб в первой консоли
```sql
insert into persons(first_name, second_name) values('sveta', 'svetova');
```
Сделал select * во второй консоли. Не закоммиченные данные не видны во второй консоли т.к. грязное чтение не допускается на это уровне
17. Завершил транзакцию в первой консоли. Выполнил select * во втрой. Новые данные не появились, т.к. во втрой консоли транзакция еще не завершена, а на данном уровне транзакций не повторяющееся чтение не допускается(non-repeatable read)
18. Завершил транзакцию во второй консоли. При селекте данные добавленные в первой консоли появились, т.к. транзакция во второй консоле завершена
