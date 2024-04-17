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
sudo docker run -it --network postgres --name pg-client postgres:15 psql -h pg-server -U postgres
```
