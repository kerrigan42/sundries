### Бэкапы ###
Установим и запустим PostgreSQL 15:
```
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15
sudo pg_ctlcluster 15 main start
```
Создадим БД, схему и в ней таблицу и заполним таблицу автосгенерированными 100 записями:
```
sudo -u postgres psql
create database hw9;
\c hw9;
create schema hw9;
create table hw9.hw9 as select generate_series(1, 100) as id, md5(random()::text)::char(10) as char;
\q
```
Под пользователем postgres создадим каталог для бэкапов:
```
sudo su postgres
mkdir ~/backups
```
Сделаем логический бэкап используя утилиту COPY:
```
postgres@ubuntu2:~$ psql
psql (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
Type "help" for help.

postgres=# \c hw9;
You are now connected to database "hw9" as user "postgres".
hw9=# \copy hw9.hw9 to '/var/lib/postgresql/backups/backup.sql'
COPY 100
hw9=#
```
Восстановим в 2 таблицу данные из бэкапа:
```
hw9=# create table hw9.hw92 (id int, char char(10));
CREATE TABLE
hw9=# \copy hw9.hw92 from '/var/lib/postgresql/backups/backup.sql';
COPY 100
```
Беглая проверка при помощи select показывает, что данные в двух таблицах идентичны.
Используя утилиту pg_dump создадим бэкап в кастомном сжатом формате двух таблиц:
```
pg_dump -d hw9 --create -Fc > /tmp/backup_dump.gz
```
Используя утилиту pg_restore восстановим в новую БД только вторую таблицу.
Сначала подготовим новую БД и схему в ней:
```
psql
create database hw9restore;
\c hw9restore;
create schema hw9;
exit
```
И теперь запускаем восстановление:
```
pg_restore -d hw9restore -U postgres -t hw92 /tmp/backup_dump.gz
```
Select из восстановленной второй таблицы в новой БД соответствует select второй табдлицы из старой БД.



