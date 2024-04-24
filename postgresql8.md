### Нагрузочное тестирование и тюнинг PostgreSQL ###
Эксперимент провожу на ВМ с 2 CPU и 4 ГБ RAM.
Установим и запустим PostgreSQL 15:
```
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15
sudo pg_ctlcluster 15 main start
```
Cоздадим базу данных для тестов:
```
sudo su postgres
pgbench -i postgres
```
Запустим тест, чтобы узнать производительность кластера с дефолтными настройками:
```
postgres@ubuntu1:/home/user$ pgbench -c 50 -j 2 -P 10 -T 60 postgres
pgbench (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
starting vacuum...end.
progress: 10.0 s, 135.4 tps, lat 356.486 ms stddev 238.909, 0 failed
progress: 20.0 s, 141.7 tps, lat 354.463 ms stddev 227.017, 0 failed
progress: 30.0 s, 139.5 tps, lat 357.119 ms stddev 262.268, 0 failed
progress: 40.0 s, 141.1 tps, lat 353.982 ms stddev 223.586, 0 failed
progress: 50.0 s, 142.2 tps, lat 352.072 ms stddev 233.536, 0 failed
progress: 60.0 s, 141.5 tps, lat 353.294 ms stddev 256.182, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 8464
number of failed transactions: 0 (0.000%)
latency average = 355.015 ms
latency stddev = 240.599 ms
initial connection time = 86.977 ms
tps = 140.434249 (without initial connection time)
```
Попробуем настроить кластер PostgreSQL 15 на максимальную производительность. Для этого я использую комбинацию значений параметров, которые предлагают ресурсы https://pgconfigurator.cybertec.at/ и https://pgtune.leopard.in.ua/#/, а также параметров из файла параметры_кластера.txt.
В итоге вот такие изменения в конфигурационном файле /etc/postgresql/15/main/postgresql.conf были произведены:
max_connections - уменьшаем до 50
shared_buffers - увеличиваем до 1 Гб
wal_buffers = 16MB
maintenance_work_mem - увеличиваем до 360MB
effective_cache_size - уменьшаем до 3GB
synchronous_commit - выключаем
checkpoint_timeout - увеличиваем до 15 мин
max_wal_size - увеличиваем до 8GB
work_mem - увеличиваем до 32MB

Перезапускаем кластер для применения изменений и ещё раз запускаем тест:
```
postgres@ubuntu1:/home/user$ pgbench -c 50 -j 2 -P 10 -T 60 postgres
pgbench (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
starting vacuum...end.
progress: 10.0 s, 2594.3 tps, lat 19.070 ms stddev 13.210, 0 failed
progress: 20.0 s, 2778.7 tps, lat 17.992 ms stddev 11.888, 0 failed
progress: 30.0 s, 2727.5 tps, lat 18.334 ms stddev 11.927, 0 failed
progress: 40.0 s, 2756.8 tps, lat 18.134 ms stddev 11.873, 0 failed
progress: 50.0 s, 2777.8 tps, lat 17.996 ms stddev 11.903, 0 failed
progress: 60.0 s, 2739.0 tps, lat 18.245 ms stddev 12.939, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 163790
number of failed transactions: 0 (0.000%)
latency average = 18.300 ms
latency stddev = 12.327 ms
initial connection time = 91.249 ms
tps = 2729.611916 (without initial connection time)
```
tps после тюнинга стал почти в 20 раз больше.

### Провернём упражнение со звёздочкой ###
Для чистоты эксперимента протестируем кластер ещё утилитой https://github.com/Percona-Lab/sysbench-tpcc
Сначала установим утилиту https://github.com/akopytov/sysbench:
```
curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.deb.sh | sudo bash
sudo apt -y install sysbench
```
Создадим пользователя и базу данных для теста:
```
sudo -u postgres psql
CREATE USER test WITH PASSWORD 'test';
CREATE DATABASE test;
ALTER DATABASE test OWNER TO test;
\q
exit
```
Клонируем репозиторий с бенчмарком и переходим в директорию с ним:
```
git clone https://github.com/Percona-Lab/sysbench-tpcc.git
cd sysbench-tpcc
```
Запустим скрипт подготовки таблиц и данных:
```
./tpcc.lua --pgsql-user=test --time=60 --threads=2 --report-interval=10 --tables=10 --scale=100 --db-driver=pgsql --pgsql-password=test --pgsql-host=localhost --pgsql-db=test prepare
```
И подождём примерно вечность, пока данные нагенерятся. 
```
./tpcc.lua --pgsql-user=test --time=60 --threads=2 --report-interval=10 --tables=10 --scale=100 --db-driver=pgsql --pgsql-password=test --pgsql-host=localhost --pgsql-db=test run


