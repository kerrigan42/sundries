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
Почему-то никаких особых изменений в тесте нет:(
Создадим таблицу с текстовым полем и заполним сгенерированными данными в размере 1млн строк:
```
psql
CREATE TABLE hw5(char char(100));
INSERT INTO hw5(char) SELECT 'test' FROM generate_series(1,1000000);
```
Посмотрим размер файла с таблицей:
```
SELECT pg_size_pretty(pg_TABLE_size('hw5'));
```
Размер 128 MB.
Обновим 5 раз все строчки и добавить к каждой строчке любой символ:
```
-
update hw5 set char = 'testte';
update hw5 set char = 'testtes';
update hw5 set char = 'testtest';
update hw5 set char = 'testtestt';
```
Посмотрим количество мертвых строчек в таблице и когда последний раз приходил автовакуум:
```
SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'hw5';
```
Видим, что в таблице 4999919 мертвых строк:
```
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'hw5';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 hw5     |    1000000 |    4999919 |    499 | 2024-04-23 11:41:26.913824+00
(1 row)
```
Подождём, пока пройдёт автовакуум. После автовакуума количество мёртвых строк - 0:
```
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'hw5';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 hw5     |    1000000 |          0 |      0 | 2024-04-23 11:42:29.337041+00
```
Опять обновим 5 раз строки и посмотрим размер таблицы.
После обновления размер таблицы 769 MB.
Отключим автовакуум:
```
ALTER TABLE hw5 SET (autovacuum_enabled = off);
```
10 раз обновим все строчки и посмотрим размер файла с таблицей.
Размер 1409 MB.
Размер таблицы после 5 обновлений стал на 5x128 MB больше, потому что обновление строк не перезаписывает существующие строки, а создаёт новые с новыми значениями. Старые строки становятся мёртвыми, но после автовакуума эти строки обнуляются и в них снова можно записывать данные. Но эти строки не удаляются, то есть продолжают занимать место на диске. Поэтому даже после автовакуума размер файла с таблицей это 6x128 MB (768 MB). 
После того, как мы отключили автовакууу и сделали 10 обновлений, 5 из этих обновлений записались в уже существующие строки, которые были очищены автовакуумом после предыдущих обновлений. Поэтому размер файла с таблицей должен увеличиться на 5x128 МБ. Что и получилось: размер уже был 769 МБ. Прибавляем к этому 640 МБ и получаем 1409 МБ.

Включим обратно автовакуум:
```
ALTER TABLE hw5 SET (autovacuum_enabled = on);
```
