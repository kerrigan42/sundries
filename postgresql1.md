### Работа с уровнями изоляции транзакции в PostgreSQL ###
Установим PostgreSQL:
```
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15
```
И запустим:
```
sudo pg_ctlcluster 15 main start
```
Запустим psql из под пользователя postgres в текущей сессии и ещё во второй сессии, чтобы смотреть, как работает изоляция транзаций:
```
sudo -u postgres psql
```
Выключим auto commit:
```
\set AUTOCOMMIT OFF
```
В первой сессии создадим новую таблицу и наполним её данными:
```
create table persons(id serial, first_name text, second_name text);
insert into persons(first_name, second_name) values('ivan', 'ivanov');
insert into persons(first_name, second_name) values('petr', 'petrov');
commit;
```
Посмотрим текущий уровень изоляции: 
```
show transaction isolation level;
```
Уровень изоляции должен быть __read committed__. Это уровень изоляции по-умолчанию.
Начинаем новую транзакцию в обоих сессиях с дефолтным уровнем изоляции:
```
BEGIN;
```
В первой сессии добавим новую запись: 
```
insert into persons(first_name, second_name) values('sergey', 'sergeev');
```
Во второй сессии посмотрим содержимое таблицы:
```
select * from persons;
```
Записи, которую мы добавили в первой сессии 
