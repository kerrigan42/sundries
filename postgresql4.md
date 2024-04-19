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
