### Репликация ###
Развернём свой миникластер на 3 ВМ.
Адреса машин: ВМ1 - 192.168.56.101, ВМ2 - 192.168.56.102, ВМ3 - 192.168.56.103.
На всех машинах устанавливаем и запускаем PostgreSQL 15 и включаем логический уровень для журнала предзаписи:
```
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15
sudo pg_ctlcluster 15 main start
sudo -u postgres psql
alter system set wal_level = logical;
exit
```
А также на всех машинах включаем прослушивание подключений на всех интерфейсах (по умолчанию прослушивание включено только для локалхоста), для этого в конфигурационном файле /etc/postgresql/15/main/postgresql.conf параметр __listen_addresses__ приводим к виду:
```
listen_addresses = '0.0.0.0'
```
И добавим пользователю postgres возможность подключаться не только локально, но и из подсети 192.168.56.0/24. Для этого в файл /etc/postgresql/15/main/pg_hba.conf добавляем строку:
```
host    postgres        postgres        192.168.56.0/24         trust
```
И применим изменения:
```
sudo pg_ctlcluster 15 main restart
```
На 1 ВМ создаем таблицы test для записи, test2 для запросов на чтение:
```
sudo -u postgres psql
create table test (int int, char char(10));
create table test2 (int int, char char(10));
```
Создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ 2:
```
create publication test_pub for table test;
create subscription test2_sub connection 'host=192.168.56.102' publication test2_pub with (copy_data = true);
```
На 2 ВМ создаем таблицы test2 для записи, test для запросов на чтение:
```
sudo -u postgres psql
create table test (int int, char char(10));
create table test2 (int int, char char(10));
```
Создадим публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ 1:
```
create publication test2_pub for table test2;
create subscription test_sub connection 'host=192.168.56.101' publication test_pub with (copy_data = true);
```
3 ВМ будем использовать как реплику для чтения и бэкапов (подпишемся на таблицы из ВМ 1 и ВМ 2 ):
```
user@node3:/etc/postgresql/15/main$ sudo -u postgres psql
psql (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
Type "help" for help.

postgres=# create table test (int int, char char(10));
CREATE TABLE
postgres=# create table test2 (int int, char char(10));
CREATE TABLE
postgres=# create subscription test_subhw10 connection 'host=192.168.56.101' publication test_pub with (copy_data = true);
NOTICE:  created replication slot "test_subhw10" on publisher
CREATE SUBSCRIPTION
postgres=# create subscription test2_subhw10 connection 'host=192.168.56.102' publication test2_pub with (copy_data = true);
NOTICE:  created replication slot "test2_subhw10" on publisher
CREATE SUBSCRIPTION
```
И теперь проверим, как работает репликация. Добавим данных в таблицу test на ВМ1 и данных в таблицу test2 на ВМ2. Добавленные данные должны среплицироваться по всем машинам.
Добавляем данные на ВМ1:
```
insert into test values (1, 'qwe');
```
Добавляем данные на ВМ2:
```
insert into test2 values (2, 'asd');
```
Проверяем на ВМ2 и ВМ3 содержимое таблицы test:
```
postgres=# select * from test;
 int |    char    
-----+------------
   1 | qwe       
(1 row)
```
На обеих машинах данные среплицировались с ВМ 1.
Проверяем на ВМ1 и ВМ3 содержимое таблицы test2:
На ВМ 3 данные среплицировались:
```
postgres=# select * from test2;
 int |    char    
-----+------------
   2 | asd       
(1 row)
```
А на ВМ данных нет. Так получилось из-за того, что на ВМ 1 мы создали подписку до того, как создали саму таблицу на ВМ2. Поэтому нужно удалить подписку и создать ещё раз и проверить репликацию данных:
```
postgres=# drop subscription test2_sub;
NOTICE:  dropped replication slot "test2_sub" on publisher
DROP SUBSCRIPTION
postgres=# create subscription test2_sub connection 'host=192.168.56.102' publication test2_pub with (copy_data = true);
NOTICE:  created replication slot "test2_sub" on publisher
CREATE SUBSCRIPTION
postgres=# select * from test2;
 int |    char    
-----+------------
   2 | asd       
(1 row)
```




