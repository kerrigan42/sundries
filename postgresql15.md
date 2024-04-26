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
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-16 && sudo systemctl stop postgresql && sudo ln -s /usr/lib/postgresql/16/bin/* /usr/sbin/ && sudo apt -y install python3 python3-pip && sudo apt install python3-testresources && sudo pip3 install --upgrade setuptools && sudo pip3 install psycopg2-binary && sudo apt install libpq-dev python3-dev && sudo pip3 install psycopg2 && sudo pip3 install patroni && sudo pip3 install python-etcd
```
Сконфигурируем кластер patroni на этих ВМ, путём добавления в конфигурационный файл /etc/patroni.yml строк:
```
scope: postgres
namespace: /db/
name: ubuntu1

restapi:
    listen: 0.0.0.0:8008
    connect_address: 192.168.56.101:8008

etcd:
    host: 192.168.56.104:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:

  initdb:
  - encoding: UTF8
  - data-checksums

  pg_hba:
  - host replication replicator 127.0.0.1/32 scram-sha-256
  - host replication replicator 192.168.56.0/24 scram-sha-256
  - host all all 0.0.0.0/0 scram-sha-256

  users:
    admin:
      password: admin
      options:
        - createrole
        - createdb

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 192.168.56.101:5432
  data_dir: /data/patroni
  pgpass: /tmp/pgpass
  authentication:
    replication:
      username: replicator
      password: replicator
    superuser:
      username: postgres
      password: postgres
  parameters:
      unix_socket_directories: '.'

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
```
На ВМ 2 и 3 указываем в этом конфиге соответствующие хостнеймы и адреса.
Создадим необходимые директории для patroni на ВМ 1, 2, 3:
```
sudo mkdir -p /data/patroni && sudo chown -R postgres:postgres /data && sudo chmod 700 /data/patroni
```
Создадим на ВМ 1, 2, 3 unit-файл для сервиса Patroni /etc/systemd/system/patroni.service следующего содержания:
```
[Unit]
Description=High availability PostgreSQL Cluster
After=syslog.target network.target

[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/usr/local/bin/patroni /etc/patroni.yml
KillMode=process
TimeoutSec=30
Restart=no

[Install]
WantedBy=multi-user.targ
```
И запускаем сервис Patroni на трёх ВМ:
```
sudo systemctl start patroni
```

Устанавливаем etcd на ВМ 4:
```
sudo apt -y install etcd
```
Сконфигурируем кластер etcd на ВМ 4, путём добавления в конфигурационный файл /etc/default/etcd строк:
```
ETCD_LISTEN_PEER_URLS="http://192.168.56.104:2380"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
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
