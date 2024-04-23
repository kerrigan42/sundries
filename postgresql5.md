### Настройка autovacuum с учетом особеностей производительности ###
Проводим эксперименты в ВМ с такими параметрами: 2 ядра и 4 Гб ОЗУ и SSD 10GB.
Установим PostgreSQL 15 с дефолтными настройками и запустим его:
```
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15
sudo pg_ctlcluster 15 main start
```
Создадим БД для тестов: 
```
sudo su postgres
pgbench -i postgres
```
Запустим нагрузочный тест pgbench:
```
pgbench -c8 -P 6 -T 60 -U postgres postgres
```
Результаты теста:
```
pgbench (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 140.5 tps, lat 56.474 ms stddev 28.424, 0 failed
progress: 12.0 s, 143.8 tps, lat 55.603 ms stddev 28.958, 0 failed
progress: 18.0 s, 144.7 tps, lat 55.353 ms stddev 28.048, 0 failed
progress: 24.0 s, 141.0 tps, lat 56.617 ms stddev 29.387, 0 failed
progress: 30.0 s, 139.2 tps, lat 57.469 ms stddev 32.924, 0 failed
progress: 36.0 s, 144.8 tps, lat 55.301 ms stddev 27.592, 0 failed
progress: 42.0 s, 144.0 tps, lat 55.510 ms stddev 27.948, 0 failed
progress: 48.0 s, 144.5 tps, lat 55.380 ms stddev 28.742, 0 failed
progress: 54.0 s, 143.8 tps, lat 55.571 ms stddev 28.575, 0 failed
progress: 60.0 s, 145.8 tps, lat 54.824 ms stddev 29.081, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 8601
number of failed transactions: 0 (0.000%)
latency average = 55.807 ms
latency stddev = 29.000 ms
initial connection time = 18.679 ms
tps = 143.269934 (without initial connection time)
```
Внесём параметры настройки PostgreSQL из файла параметры_кластера.txt в файл /etc/postgresql/15/main/postgresql.conf.
Перезагрузим кластер для применения изменений и ещё раз запустим тест:
```
exit
sudo pg_ctlcluster 15 main restart
sudo su postgres
pgbench -c8 -P 6 -T 60 -U postgres postgres
```
Результаты теста:
```
pgbench (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 137.7 tps, lat 57.517 ms stddev 31.970, 0 failed
progress: 12.0 s, 141.5 tps, lat 56.643 ms stddev 28.430, 0 failed
progress: 18.0 s, 142.7 tps, lat 56.020 ms stddev 28.176, 0 failed
progress: 24.0 s, 141.5 tps, lat 56.461 ms stddev 29.067, 0 failed
progress: 30.0 s, 142.8 tps, lat 56.088 ms stddev 27.383, 0 failed
progress: 36.0 s, 142.8 tps, lat 56.005 ms stddev 31.355, 0 failed
progress: 42.0 s, 141.2 tps, lat 56.705 ms stddev 30.649, 0 failed
progress: 48.0 s, 144.3 tps, lat 55.363 ms stddev 28.585, 0 failed
progress: 54.0 s, 143.3 tps, lat 55.833 ms stddev 30.555, 0 failed
progress: 60.0 s, 143.3 tps, lat 55.776 ms stddev 29.898, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 8535
number of failed transactions: 0 (0.000%)
latency average = 56.247 ms
latency stddev = 29.637 ms
initial connection time = 16.887 ms
tps = 142.146610 (without initial connection time)
```
