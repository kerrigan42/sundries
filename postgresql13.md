### Секционирование таблицы ###
Секционируем большую таблицу из демо базы flights.
Установим и запустим PostgreSQL 15:
```
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15
sudo pg_ctlcluster 15 main start
```
Загрузим демо базу flights:
```
wget https://edu.postgrespro.com/demo-big-en.zip
sudo apt install unzip
unzip demo-big-en.zip
chown postgres:postgres
sudo mv demo-big-en-20170815.sql /var/lib/postgresql/
sudo su postgres
cd ~
psql -f demo-big-en-20170815.sql
```
Будем секционировать таблицу ticket_flights по столбцу ticket_no. Значения этого столбца имеют значения от 0005432000284 до 0005435999873, поэтому будем разбивать на секции по 800000:
```
create table hw13 (like ticket_flights) partition by range(ticket_no);
create table tickets_no1 partition of hw13 for values from ('0005432000000') to ('0005432800000');
create table tickets_no2 partition of hw13 for values from ('0005432800000') to ('0005433600000');
create table tickets_no3 partition of hw13 for values from ('0005433600000') to ('0005434400000');
create table tickets_no4 partition of hw13 for values from ('0005434400000') to ('0005435200000');
create table tickets_no5 partition of hw13 for values from ('0005435200000') to ('0005436000000');
insert into hw13 select * from ticket_flights;
```
Теперь посмотрим экспланацию для определённого диапазона номеров билетов:
```
demo=# explain select * from hw13 where ticket_no > '0005433200000' and ticket_no < '0005434800000';
                                            QUERY PLAN                                             
---------------------------------------------------------------------------------------------------
 Append  (cost=0.00..143152.79 rows=3606007 width=32)
   ->  Seq Scan on tickets_no2 hw13_1  (cost=0.00..41197.46 rows=935655 width=32)
         Filter: ((ticket_no > '0005433200000'::bpchar) AND (ticket_no < '0005434800000'::bpchar))
   ->  Seq Scan on tickets_no3 hw13_2  (cost=0.00..42379.96 rows=1815901 width=32)
         Filter: ((ticket_no > '0005433200000'::bpchar) AND (ticket_no < '0005434800000'::bpchar))
   ->  Seq Scan on tickets_no4 hw13_3  (cost=0.00..41545.33 rows=854451 width=32)
         Filter: ((ticket_no > '0005433200000'::bpchar) AND (ticket_no < '0005434800000'::bpchar))
 JIT:
   Functions: 6
   Options: Inlining false, Optimization false, Expressions true, Deforming true
(10 rows)
```
Видно, что поиск был только по трём секциям, а не по всей таблице. 

