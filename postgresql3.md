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
sudo su postgres
psql
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



