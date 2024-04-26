### Триггеры, поддержка заполнения витрин ###
Установим и запустим PostgreSQL15 и подготовим тестовые данные:
```
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15
sudo pg_ctlcluster 15 main start
```
В БД создим структуру, описывающую товары (таблица goods) и продажи (таблица sales):
```
sudo -u postgres psql
DROP SCHEMA IF EXISTS pract_functions CASCADE;
CREATE SCHEMA pract_functions;
SET search_path = pract_functions, publ;
CREATE TABLE goods
(
    goods_id    integer PRIMARY KEY,
    good_name   varchar(63) NOT NULL,
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
);
INSERT INTO goods (goods_id, good_name, good_price)
VALUES 	(1, 'Спички хозайственные', .50),
		(2, 'Автомобиль Ferrari FXX K', 185000000.01);
CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);
INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);
```
Есть запрос для генерации отчета – сумма продаж по каждому товару:
```
SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;
```
C увеличением объёма данных отчет стал создаваться медленно, поэтому было принято решение денормализовать БД.
Создадим таблицу (витрину), структура которой повторяет структуру отчета:
```
CREATE TABLE good_sum_mart
(
	good_name   varchar(63) NOT NULL,
	sum_sale	numeric(16, 2)NOT NULL
);
```
И заполним её данными:
```
insert into good_sum_mart(good_name, sum_sale)
    select g.good_name, sum(g.good_price * s.sales_qty) from goods g
    inner join sales s
        on s.good_id = g.goods_id
    group by g.good_name;
```
Создим триггер на таблице продаж, для поддержки данных в витрине в актуальном состоянии (вычисляющий при каждой продаже сумму и записывающий её в витрину):
```
create or replace function update_sum() returns trigger as
$$
declare
    modifying_good_name varchar(63);
    good_price_old numeric(12, 2);
    good_price_new numeric(12, 2);
    sum_increment numeric(16, 2);
begin
    sum_increment := 0;
   
    if (old is not null) then
        select good_name, good_price into modifying_good_name, good_price_old from pract_functions.goods where goods_id = old.good_id;
    end if;

    if (new is not null) then
        select good_name, good_price into modifying_good_name, good_price_new from pract_functions.goods where goods_id = new.good_id;
    end if; 

    select
        case
            when tg_op = 'INSERT' then new.sales_qty * good_price_new
            when tg_op = 'DELETE' then -old.sales_qty * good_price_old
            when tg_op = 'UPDATE' then (new.sales_qty * good_price_new) - (old.sales_qty * good_price_old)
            else 0
        end
    into sum_increment;

    if exists (select 1 from pract_functions.good_sum_mart where good_name = modifying_good_name)
    then 
        update pract_functions.good_sum_mart
        set sum_sale = pract_functions.good_sum_mart.sum_sale + sum_increment
        where good_name = modifying_good_name;
    else
        insert into pract_functions.good_sum_mart (good_name, sum_sale)
        values (modifying_good_name, sum_increment);
    end if;  
  return new;
end;
$$
language plpgsql;
create trigger trigger_sales
after insert or delete or update on pract_functions.sales
for each row execute function update_sum();
```
Теперь будем продавать спички и смотреть, как это влияет на отчёт. 
Текущее состояние отчёта:
```
postgres=# select * from good_sum_mart;
        good_name         |   sum_sale   
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)
```
Продадим спички и ещё раз посмотрим на отчёт:
```
postgres=# insert into sales (good_id, sales_qty) values (1, 1);
INSERT 0 1
postgres=# select * from good_sum_mart;
        good_name         |   sum_sale   
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        66.00
(2 rows)
```
Отчёт отражает продажу спичек.

Проверим, как отчёт отреагирует на обновление сделки по продаже спичек:
```
postgres=# select * from sales;
 sales_id | good_id |          sales_time           | sales_qty 
----------+---------+-------------------------------+-----------
        1 |       1 | 2024-04-26 11:41:43.821795+00 |        10
        2 |       1 | 2024-04-26 11:41:43.821795+00 |         1
        3 |       1 | 2024-04-26 11:41:43.821795+00 |       120
        4 |       2 | 2024-04-26 11:41:43.821795+00 |         1
        5 |       1 | 2024-04-26 11:45:05.318272+00 |         1
(5 rows)

postgres=# update sales set sales_qty = 2 where sales_id = 5;
UPDATE 1
postgres=# select * from good_sum_mart;
        good_name         |   sum_sale   
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        66.50
(2 rows)
```
Увеличение количества проданных спичек на 1 в последней продаже привело к увеличению общей суммы продаж спичек на 0,50.

И проверка отмены сделки по продаже спичек:
```
postgres=# select * from sales;
 sales_id | good_id |          sales_time           | sales_qty 
----------+---------+-------------------------------+-----------
        1 |       1 | 2024-04-26 11:41:43.821795+00 |        10
        2 |       1 | 2024-04-26 11:41:43.821795+00 |         1
        3 |       1 | 2024-04-26 11:41:43.821795+00 |       120
        4 |       2 | 2024-04-26 11:41:43.821795+00 |         1
        5 |       1 | 2024-04-26 11:45:05.318272+00 |         2
(5 rows)

postgres=# delete from sales where sales_id = 5;
DELETE 1
postgres=# select * from good_sum_mart;
        good_name         |   sum_sale   
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)
```
Удаляем из таблицы продаж продажу 2 спичек по 0.50, и в отчёте общая сумма выручки от продажи спичек уменьшается на 1.

