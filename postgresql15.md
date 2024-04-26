### Кластер Patroni ###
Для кластера у нас есть три ВМ с адресами:
ubuntu1 - 192.168.56.101
ubuntu2 - 192.168.56.102
ubuntu3 - 192.168.56.103.
И ещё две ВМ для etcd и haproxy:
etcd - 192.168.56.104
haproxy - 192.168.56.105

Устанавливаем PostgreSQL 16 и пакеты, необходимые для построения кластера, на ВМ 1, 2, 3:
```
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-16 && sudo systemctl stop postgresql && sudo ln -s /usr/lib/postgresql/14/bin/* /usr/sbin/ && sudo apt -y install python3 python3-pip && sudo apt install python3-testresources && sudo pip3 install --upgrade setuptools && sudo pip3 install psycopg2-binary && sudo apt install libpq-dev python3-dev && sudo pip3 install psycopg2 && sudo pip3 install patroni && sudo pip3 install python-etcd
```
Устанавливаем etcd на ВМ 4:
```
sudo apt -y install etcd
```
Сконфигурируем кластер etcd на ВМ 4, путём добавления в конфигурационный файл /etc/default/etcd строк:
```
ETCD_LISTEN_PEER_URLS="http://192.168.56.104:2380"
ETCD_LISTEN_CLIENT_URLS="http://localhost:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.56.104:2380"
ETCD_INITIAL_CLUSTER="default=http://192.168.56.104:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.56.104:2379"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_ENABLE_V2="true"
```
И применим конфигурацию:
```
sudo systemctl restart etcd
```
Устанавливаем haproxy на ВМ 5:
```
sudo apt -y install haproxy
```
