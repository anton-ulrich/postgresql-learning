# Домашняя работа 11
## Секционирование таблицы

Развернул виртуалку в Yandex Cloud с именем pg-homework11-01 и установил Postgres
```
sudo apt-get update && sudo apt-get upgrade -y
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt-get update && sudo apt-get install -y postgresql-14
```

Получаем самый большой архив базы полетов и распаковываем его
```
wget https://edu.postgrespro.ru/demo-big.zip
sudo apt-get install -y unzip
unzip demo-big.zip
```

Восстановил демо БД из дампа
```
sudo -u postgres psql -f demo-big-20170815.sql
```

Заходим в psql, подключаемся к базе и ищем самую большую таблицу
```

```
