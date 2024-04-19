### Работа с базами данных, пользователями и правами ###
Cоздадим новый кластер PostgresSQL 14:
```
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-14
sudo pg_ctlcluster 14 main start
```
Зайдём в созданный кластер под пользователем postgres:
```
sudo -u postgres psql
```
Создадим новую базу данных testdb:
```
CREATE DATABASE testdb;
```
Зайдём в созданную базу данных под пользователем postgres:
```
\c testdb
```
Создадим новую схему testnm:
```
CREATE SCHEMA testnm;
```
Создадим новую таблицу t1 с одной колонкой c1 типа integer:
```
CREATE TABLE testnm.t1(c1 integer);
```
Вставим строку со значением c1=1:
```
INSERT INTO testnm.t1 values(1);
```
Создадим новую роль readonly:
```
CREATE role readonly;
```
Дадим новой роли право на подключение к базе данных testdb:
```
grant connect on DATABASE testdb TO readonly;
```
Дадим новой роли право на использование схемы testnm:
```
grant usage on SCHEMA testnm to readonly;
```
Дадим новой роли право на select для всех таблиц схемы testnm:
```
grant SELECT on all TABLEs in SCHEMA testnm TO readonly;
```
Создадим пользователя testread с паролем test123:
```
CREATE USER testread with password 'test123';
```
Дадим роль readonly пользователю testread:
```
grant readonly TO testread;
```
Зайдём под пользователем testread в базу данных testdb и проверим права:
```
\q
psql -h localhost -U testread -d testdb
select * from testnm.t1;
```
Select должен отработать.
Теперь попробуем такое:
```
create table t2(c1 integer);
insert into t2 values (2);
```
Такое тоже работает несмотря на то, что права мы давали только на select.
Но это из-за того, что здесь в командах не указана схема, в которой создаётся таблица и поэтому она создаётся по-умолчанию в схеме public, а не testnm.
Чтобы схема public не смущала, отзываем права у роли public для базы testdb под пользователем postgres:
```
REVOKE CREATE on SCHEMA public FROM public; 
REVOKE ALL on DATABASE testdb FROM public;
```
Подключися ещё раз пользователем testread и проверим, что таблицы больше не создаются в схеме public:
```
create table t3(c1 integer);
```
