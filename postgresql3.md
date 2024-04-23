### Физический уровень PostgreSQL ###
Установим PostgreSQL 15 и запустим его:
```
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15
sudo pg_ctlcluster 15 main start
```
Проверим, что кластер запущен:
```
sudo -u postgres pg_lsclusters
```
Зайдём из под пользователя postgres в psql и сделаем произвольную таблицу с произвольным содержимым:
```
sudo -u postgres psql
create table test(c1 text);
insert into test values('1');
\q
```
Остановим postgres:
```
pg_ctlcluster 15 main stop
```
Cоздадим новый диск размером 10GB и добавим его к виртуальной машине. В VirtualBox для этого необходимо выключить машину, зайти в её настройки, далее в пункт "Носители" и в пункте "Контроллер SATA" добавить новый диск.
После включения машины проинициализируем диск и создадим файловую систему и точку монтирования:
```
sudo parted /dev/sdb mklabel gpt
sudo parted -a opt /dev/sdb mkpart primary ext4 0% 100%
sudo mkfs.ext4 /dev/sdb1
sudo mkdir -p /mnt/data
```
Внесём информацию о монтировании нового диска при включении в файл /etc/fstab и проверим, что после перезагрузки диск подмонтирован в /mnt/data. Для этого я добавляю в конец /etc/fstab строку:
```
/dev/disk/by-uuid/2f003b45-3d27-4176-a841-792e02412f30 /mnt/data ext4 defaults 0 1
```
uuid диска в каждой системе будет отличатся. 
После перезагрузки команда ```df -h``` должена показывать в выводе диск /dev/sda1, примонтированный к /mnt/data/.
Cделаем пользователя postgres владельцем /mnt/data:
```
sudo chown -R postgres:postgres /mnt/data
```
Перенесём содержимое /var/lib/postgres/15 в /mnt/data:
```
sudo mv /var/lib/postgresql/15/mnt/data
```
Попробуем запустить кластер:
```
sudo -u postgres pg_ctlcluster 15 main start
```
Кластер не запускается с ошибкой __Error: /var/lib/postgresql/15/main is not accessible or does not exist__. 
Что ожидаемо, поскольку судя по выводу команды ```sudo -u postgres pg_lsclusters``` кластер ожидает, что данные хранятся по пути /var/lib/postgresql/15/main, но мы их переместили в /mnt/data. Поэтому теперь для запуска нужно поменять в конфигурационном файле путь директории для хранения данных. Для этого в конфигурационном файле /etc/postgresql/15/main/postgresql.conf меняем значение параметра __data_directory__ с _/var/lib/postgresql/15/main_ на _/mnt/data/15/main_.
Пробуем запустить кластер ещё раз. Теперь видим, что кластер запустился и в списке кластеров его путь хранения данных теперь указан как _/mnt/data/15/main_ .
Зайдём через psql и проверим, что таблица test, которую мы создали ранее, существует и содержит данные:
```
sudo -u postgres psql
select * from test;
```
Таблица и её содержимое на месте.

## Упражнение со звёздочкой ##
Попробуем создать вторую ВМ и подключить к ней диск с содержимым базы postgresql.
Экспериментирую в VirtualBox. 
Создаём вторую ВМ. Первую ВМ выключаем. В настройках второй ВМ в разделе __Носители__-->__Контроллер SATA добавляем к машине диск на 10 Гб, который мы создавали для монтирования к точке /mnt/data первой ВМ.
Включаем вторую ВМ и устанавливаем на ней postgres.
Удалим файлы с данными из /var/lib/postgresql:
```
sudo rm -rf /var/lib/postgresql/*
```
Внесём информацию о монтировании диска 10Гб в файл /etc/fstab:
```
/dev/disk/by-uuid/2f003b45-3d27-4176-a841-792e02412f30 /var/lib/postgresql ext4 defaults 0 1
```
uuid диска в каждой системе будет отличатся. 
Перезагружаемся и запускаем кластер postgres:
```
sudo pg_ctlcluster 15 main start
```
Проверим, что он запустился:
```
sudo -u postgres pg_lsclusters
```
В выводе __Status__ _online_. 
Затем проверяем, что таблица test существует и содержит одну строку.
```
sudo -u postgres psql
select * from test;
```
Вывод:
```
 c1 
----
 1
(1 row)
```







