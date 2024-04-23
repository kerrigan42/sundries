### Работа с журналами ###
Установим PostgreSQL 15:
```
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15
```
Настроим выполнение контрольной точки раз в 30 секунд. Для этого в файле /etc/postgresql/15/main/postgresql.conf выставляем значение параметра __checkpoint_timeout__ в __30s__.
Запустим кластер и создадим базу данных для тестов:
```
sudo pg_ctlcluster 15 main start
sudo su postgres
pgbench -i postgres
```
Посмотрим текущую позицию записи в журнале предзаписи:
```
select pg_current_wal_lsn();
exit
```
Текущая позиция 0/13DC4A78.
C помощью утилиты pgbench подадим нагрузку в течение 10 минут:
```
pgbench -P 6 -T 600 -U postgres postgres

```
И ещё раз посмотрим текущую позицию записи в журнале предзаписи. Текущая позиция 0/253673A8.
Посмотрим, какой объем журнальных файлов был сгенерирован за это время: 
```
select pg_size_pretty('0/253673A8'::pg_lsn - '0/13DC4A78'::pg_lsn) wal_size;
```
Получилось 278 MB. И поскольку контрольных точек у нас было 20 штук, то на одну точку приходится в среднем примерно 278/20=14 MB.
Проверим, все ли контрольные точки выполнялись точно по расписанию. Судя по логам в файле /var/log/postgresql/postgresql-15-main.log точки выполнялись точно по расписанию. Концовка лога:
```
2024-04-23 15:08:21.267 UTC [4659] LOG:  checkpoint starting: time
2024-04-23 15:08:48.201 UTC [4659] LOG:  checkpoint complete: wrote 1392 buffers (8.5%); 0 WAL file(s) added, 0 removed, 0 recycled; write=26.719 s, sync=0.077 s, total=26.935 s; sync files=6, longest=0.024 s, average=0.013 s; distance=13870 kB, estimate=14079 kB
2024-04-23 15:08:51.203 UTC [4659] LOG:  checkpoint starting: time
2024-04-23 15:09:18.243 UTC [4659] LOG:  checkpoint complete: wrote 1800 buffers (11.0%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.733 s, sync=0.110 s, total=27.041 s; sync files=10, longest=0.023 s, average=0.011 s; distance=13947 kB, estimate=14065 kB
2024-04-23 15:09:21.247 UTC [4659] LOG:  checkpoint starting: time
2024-04-23 15:09:48.225 UTC [4659] LOG:  checkpoint complete: wrote 1287 buffers (7.9%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.731 s, sync=0.074 s, total=26.978 s; sync files=6, longest=0.024 s, average=0.013 s; distance=14025 kB, estimate=14061 kB
2024-04-23 15:09:51.227 UTC [4659] LOG:  checkpoint starting: time
2024-04-23 15:10:18.298 UTC [4659] LOG:  checkpoint complete: wrote 1494 buffers (9.1%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.751 s, sync=0.129 s, total=27.071 s; sync files=12, longest=0.024 s, average=0.011 s; distance=14034 kB, estimate=14059 kB
2024-04-23 15:10:21.299 UTC [4659] LOG:  checkpoint starting: time
2024-04-23 15:10:48.170 UTC [4659] LOG:  checkpoint complete: wrote 1391 buffers (8.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.617 s, sync=0.091 s, total=26.872 s; sync files=6, longest=0.024 s, average=0.016 s; distance=14066 kB, estimate=14066 kB
2024-04-23 15:10:51.171 UTC [4659] LOG:  checkpoint starting: time
2024-04-23 15:11:18.244 UTC [4659] LOG:  checkpoint complete: wrote 1782 buffers (10.9%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.827 s, sync=0.095 s, total=27.074 s; sync files=10, longest=0.024 s, average=0.010 s; distance=13217 kB, estimate=13982 kB
```
Сравним tps в синхронном/асинхронном режиме.
При синхронном режиме прогон pgbench показал результат __tps = 141.351746 (without initial connection time)__
Включим асинхронный режим:
```
psql
alter SYSTEM set synchronous_commit = off;
\q
exit
sudo pg_ctlcluster 15 main restart
```
И запустим тестирование pgbench ещё раз. 
В асинхронном режиме __tps = 6237.377710 (without initial connection time)__. Сильно больше, чем в синхронном!
Создадим новый кластер с включенной контрольной суммой страниц:
```
sudo pg_createcluster 15 hw6
sudo /usr/lib/postgresql/15/bin/pg_checksums --enable -D "/var/lib/postgresql/15/hw6"
sudo pg_ctlcluster 15 hw6 start
```
Создадим таблицу:
```
sudo su postgres
psql -p 5433
CREATE TABLE hw6(char char(100));
```
Вставим несколько значений и посмотрим путь до таблицы:
```
INSERT INTO hw6(char) SELECT 'test' FROM generate_series(1,5);
SELECT pg_relation_filepath('hw6');
exit
exit
```
Выключим кластер:
```
sudo pg_ctlcluster 15 hw6 stop
```
Изменим пару байт в таблице:
```
sudo su postgres
vim /var/lib/postgresql/15/hw6/base/5/16388
exit
```
Включим кластер и сделаем выборку из таблицы:
```
sudo pg_ctlcluster 15 hw6 start
sudo su postgres
psql -p 5433
select * from hw6;
```
Получаем ошибку из-за того, что контрольная сумма изменилась:
```
WARNING:  page verification failed, calculated checksum 63580 but expected 26218
ERROR:  invalid page in block 0 of relation base/5/16388
```
Чтобы проигнорировать ошибку и продолжить работу можно отключить проверку контрольной суммы:
```
alter SYSTEM set ignore_checksum_failure = on;
\q
exit
sudo pg_ctlcluster 15 hw6 restart
```



