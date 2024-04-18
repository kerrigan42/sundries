### Установка и настройка PostgteSQL в контейнере Docker ###
Установим Docker Engine:
```
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
Запустим службу docker:
```
sudo systemctl start docker
```
Создадим сеть, в которой два контейнера (сервер и клиент) смогут видеть друг друга:
```
sudo docker network create postgres
```
Создадим каталог /var/lib/postgres:
```
sudo mkdir /var/lib/postgres
```
Развернём контейнер с PostgreSQL 15 смонтировав в него /var/lib/postgres:
```
sudo docker run --name pg-server --network postgres -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
```
Развернём контейнер с клиентом postgres:
```
sudo docker run --name pg-client --network postgres -e POSTGRES_PASSWORD=postgres -d postgres:15
```
Подключимся из контейнера с клиентом к контейнеру с сервером и сделаем таблицу с парой строк. При подключении будет запрошен пароль. Пароль "postgres", мы установили его при помощи переменной POSTGRES_PASSWORD, когда запускали контейнер с сервером :
```
sudo docker exec -it pg-client /bin/bash
psql -h pg-server -U postgres 
create table persons(id serial, first_name text, second_name text);
insert into persons(first_name, second_name) values('ivan', 'ivanov');
insert into persons(first_name, second_name) values('petr', 'petrov');
exit
exit
```
Проверим подключение к контейнеру с сервером с другого хоста. Для этого на хосте предварительно должен быть установлен пакет postgresql-client:
```
psql -h 192.168.56.101 -U postgres
```
Вернёмся на хост, где развёрнут контейнер с сервером и удалим его:
```
sudo docker rm -f pg-server
```
Cоздадим его заново:
```
sudo docker run --name pg-server --network postgres -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
```
Подключится снова из контейнера с клиентом к контейнеру с сервером и проверим, что данные остались на месте:
```
sudo docker exec -it pg-client /bin/bash
psql -h pg-server -U postgres 
select * from persons;
```
Видим, что таблица и две записи, которые мы создавали, на месте, потому что мы монитруем в контейнер волюм /var/lib/postgres для хранения персистентных данных, которые не удаляются при удалении контейнера.
