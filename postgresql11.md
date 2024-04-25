### Работа с индексами ###
Установим и запустим PostgreSQL 15:
```
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15
sudo pg_ctlcluster 15 main start
```
Создадим таблицу статей с текстовыми данными:
```
sudo -u postgres psql
CREATE TABLE articles (id SERIAL PRIMARY KEY, title TEXT, content TEXT);
INSERT INTO articles (title, content) VALUES
('Как инженеры GitHub используют GitHub Copilot: 4 способа', 'Узнаем, как GitHub Copilot повышает эффективность работы инженеров из GitHub, позволяя автоматизировать повторяющиеся задачи, сохранять концентрацию и многое другое.'),
('Кратко про микросервисы на Scala и Erlang', 'В статье рассмотрим два языка программирования, которые выделяются своим функциональным подходом и широким применением в микросервисной архитектуре: Scala и Erlang.'),
('Интероперабельность с нативным кодом через платформу .NET', 'Часто некоторые проекты требуют от нас все более новых подходов к решению задач. '),
('Важные элементы при работе в Scrum', 'В этой статье мы разберем наиболее важные элементы, которые позволяют балансировать предсказуемость и гибкость во благо доставки наивысшей ценности клиентам.'),
('Компьютерное зрение в 2024 году: Главные задачи и направления', 'Рынок компьютерного зрения сейчас переживает бурный рост с прогнозируемым увеличением с 22 миллиардов долларов в 2023 году до 50 миллиардов к 2030 году при 21.4% совокупного годового прироста с 2024 по 2030 год.'),
('Оптимизация запросов в ClickHouse с помощью создания цепочки материализованных представлений', 'В ClickHouse материализованные представления (materialized views) являются механизмом, автоматически выполняющим запросы к исходным таблицам при поступлении новых данных.'),
('Роботы наступают. И это хорошо', 'В этом отрывке из новой книги «Сердце и чип: Наше светлое будущее вместе с роботами» (“The Heart and the Chip: Our Bright Future with Robots”) директор Лаборатории компьютерных наук и искусственного интеллекта при MIT (CSAIL) Даниэла Рус рассказывает о том, как роботы могут расширить возможности человека.'),
('Обеспечение безопасности загрузчика GRUB в Linux', 'В данной статье мы рассмотрим шаги по обеспечению безопасности загрузчика GRUB, начиная с генерации зашифрованного пароля и заканчивая его внедрением в систему.'),
('Async iterator timeout в Python', 'Представим следующую задачу: у нас есть микросервисная архитектура, в которой сервисы взаимодействуют через брокер сообщений, или через gRPC.'),
('Деплоим приложение в k8s через Jenkins+Helm3+ArgoCD', 'В этой статье мы рассмотрим каждый из этих инструментов и покажем, как объединить их в единую систему для создания надежного и автоматизированного процесса развертывания приложений в Kubernetes.');
```
Подкинем сгенерированных строк:
```
insert into articles(id,title,content)
select s.id, chr((32+random()*94)::integer), chr((32+random()*94)::integer)
from generate_series(11,100000) as s(id)
order by random();
```
Создадим индекс к таблице:
```
create index id_index on articles(id);
ANALYZE articles;
```
Экспланация запроса, использующего созданный индекс:
```
postgres=# EXPLAIN SELECT * FROM articles WHERE id < 100;
                               QUERY PLAN                                
-------------------------------------------------------------------------
 Bitmap Heap Scan on articles  (cost=5.07..244.62 rows=100 width=8)
   Recheck Cond: (id < 100)
   ->  Bitmap Index Scan on id_index  (cost=0.00..5.04 rows=100 width=0)
         Index Cond: (id < 100)
(4 rows)
```
Здесь у нас большая выборка, поэтому оптимизатор использовал сканирование по битовой карте, а не индексное.

Реализуем индекс для полнотекстового поиска.
Добавим столбец tsvector с автоматическим заполнением для полнотекстового поиска:
```
ALTER TABLE articles ADD COLUMN content_tsvector TSVECTOR GENERATED ALWAYS
AS (to_tsvector('russian', content)) STORED;
```
Создадим GIN индекс для столбца tsvector:
```
CREATE INDEX idx_articles_content_tsvector ON articles USING gin
(content_tsvector);
```
Отключим последовательное сканирование:
```
SET enable_seqscan = OFF;
```
Экспланация запроса полнотекстового поиска по статьям:
```
postgres=# EXPLAIN SELECT title, content FROM articles WHERE content_tsvector @@
to_tsquery('russian', 'Erlang');
                                          QUERY PLAN                                           
-----------------------------------------------------------------------------------------------
 Bitmap Heap Scan on articles  (cost=15.88..619.80 rows=500 width=4)
   Recheck Cond: (content_tsvector @@ '''erlang'''::tsquery)
   ->  Bitmap Index Scan on idx_articles_content_tsvector  (cost=0.00..15.75 rows=500 width=0)
         Index Cond: (content_tsvector @@ '''erlang'''::tsquery)
(4 rows)
```

Реализуем индекс на часть таблицы.
Добавим к нашей таблице столбец boolean с рандомными значениями:
```
ALTER TABLE articles ADD COLUMN c boolean;
update articles set c = random() > 0.5;
```
Создадим условный индекс для булевого поля:
```
CREATE INDEX ON articles(c) WHERE c;
```
План запроса для булевого поля:
```
postgres=# EXPLAIN SELECT * FROM articles WHERE c = TRUE;
                                    QUERY PLAN                                     
-----------------------------------------------------------------------------------
 Bitmap Heap Scan on articles  (cost=437.32..2128.52 rows=49720 width=19)
   Recheck Cond: c
   ->  Bitmap Index Scan on articles_c_idx  (cost=0.00..424.89 rows=49720 width=0)
(3 rows)
```

Создим индекс на несколько полей:
```
CREATE INDEX hw11_index ON articles(id, title);
```
План запроса для составного индекса:
```
postgres=# EXPLAIN SELECT * FROM articles WHERE id <= 100 AND
title = 'a';
                                 QUERY PLAN                                 
----------------------------------------------------------------------------
 Index Scan using hw11_index on articles  (cost=0.29..9.34 rows=1 width=19)
   Index Cond: ((id <= 100) AND (title = 'a'::text))
(2 rows)
```





