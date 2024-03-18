### Кластер Clickhouse из трёх реплик с одним шардом
# 1. Кроме трёх нод самого clickhouse понадобится ещё кластер zookeeper, установленный отдельно, но в рамках теста у меня zookeeper на этих же трёх нодах.
1.1. Соответственно,на всех трёх нодах ставим java:
```
sudo apt install default-jre
```
1.2. Создаём пользователя и группу, от которого будет запускаться zookeeper:
```
sudo groupadd -r zookeeper
sudo useradd -c "ZooKeeper" -s /sbin/nologin -g zookeeper -r zookeeper
```
1.3. Скачиваем и распаковываем архив с бинарниками и назначаем необходимые права:
```
cd /tmp
curl -O https://archive.apache.org/dist/zookeeper/zookeeper-3.9.1/apache-zookeeper-3.9.1-bin.tar.gz
sudo tar zxvf apache-zookeeper-3.9.1-bin.tar.gz -C /usr/local
sudo ln -s /usr/local/apache-zookeeper-3.9.1-bin /usr/local/zookeeper
sudo chown -R zookeeper:zookeeper /usr/local/apache-zookeeper-3.9.1-bin
sudo chown zookeeper:zookeeper /usr/local/zookeeper
```
1.4. Создаём необходимые каталоги с нужными правами:
```
sudo mkdir /var/lib/zookeeper
sudo mkdir /var/log/zookeeper
sudo chown zookeeper:zookeeper /var/lib/zookeeper
sudo chown zookeeper:zookeeper /var/log/zookeeper
```
1.5. Создаём файл /usr/local/zookeeper/conf/zoo.cfg. У меня он выглядит вот так:
```
tickTime=2000
initLimit=10
syncLimit=1
dataDir=/var/lib/zookeeper
dataLogDir=/var/lib/zookeeper
clientPort=2181
maxClientCnxns=10000
autopurge.snapRetainCount=100
autopurge.purgeInterval=24
autopurge.snapRetainCount=3
server.0=192.168.56.101:2888:3888
server.1=192.168.56.102:2888:3888
server.2=192.168.56.103:2888:3888
```
1.6. И ещё немного нужных прав и симлинков:
```
sudo chown zookeeper:zookeeper /usr/local/zookeeper/conf/zoo.cfg
sudo mkdir /etc/zookeeper
sudo chown zookeeper:zookeeper /etc/zookeeper
sudo ln -s /usr/local/zookeeper/conf/zoo.cfg /etc/zookeeper/zoo.cfg
sudo chown zookeeper:zookeeper /etc/zookeeper/zoo.cfg
```
1.7. На каждой ноде создаём файл /var/lib/zookeeper/myid с номером этой ноды в кластере Zookeeper (от 0 до 2, в соответствии с файлом конфигурации).
Соответственно, для первой ноды так:
```
user@node1:~$ cat /var/lib/zookeeper/myid
0
```
1.8. Создаём unit-файл для службы:
unit-файл /lib/systemd/system/zookeeper.service:
```
[Unit]
Description=zookeeper:3.9.1
After=network.target
[Service]
Type=forking
User=zookeeper
Group=zookeeper
WorkingDirectory=/usr/local/zookeeper
ExecStart=/usr/local/zookeeper/bin/zkServer.sh start
ExecStop=/usr/local/zookeeper/bin/zkServer.sh stop
ExecReload=/usr/local/zookeeper/bin/zkServer.sh restart
RestartSec=30
Restart=always
PrivateTmp=yes
PrivateDevices=yes
LimitCORE=infinity
LimitNOFILE=500000
[Install]
WantedBy=multi-user.target
Alias=zookeeper.service
```
Перед запуском возможно ещё нужно будет сделать
```
sudo chown -R zookeeper:zookeeper version-2
sudo chown -R zookeeper:zookeeper logs
```
1.9. Запускаем службу:
```
sudo systemctl start zookeeper
```
1.10. Чтобы проверить работоспособность кластера можно подключиться к нему клиентом и посмотреть, что находится внутри:
```
/usr/local/zookeeper/bin/zkCli.sh -server localhost:2181 get /zookeeper/config
```
# 2. На всех нодах записываем хостнеймы трёх нод в /etc/hosts:
```
cat /etc/hosts
127.0.0.1 localhost
127.0.1.1 node1

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

192.168.56.101 node1
192.168.56.102 node2
192.168.56.103 node3
```
# 3. Устанавливаем на всех трёх нодах clickhouse
```
curl -fsSL 'https://packages.clickhouse.com/rpm/lts/repodata/repomd.xml.key' | sudo gpg --dearmor -o /usr/share/keyrings/clickhouse-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/clickhouse-keyring.gpg] https://packages.clickhouse.com/deb stable main" | sudo tee     /etc/apt/sources.list.d/clickhouse.list
sudo apt-get update
sudo apt-get install -y clickhouse-server clickhouse-client
```
# 4. Создаём в /etc/clickhouse-server/config.d файл конфигурации. У меня он выглядит вот так:
```
user@node1:~$ sudo cat /etc/clickhouse-server/config.d/remote_servers.xml
[sudo] password for user: 
<clickhouse>
  <listen_host>0.0.0.0</listen_host>
  <interserver_http_host>node1</interserver_http_host>
  <remote_servers>
    <default>
      <shard>
        <internal_replication>true</internal_replication>
        <weight>1</weight>
        <replica>
          <host>node1</host>
          <port>9000</port>
        </replica>
        <replica>
          <host>node2</host>
          <port>9000</port>
        </replica>
        <replica>
          <host>node3</host>
          <port>9000</port>
        </replica>
      </shard>
    </default>
  </remote_servers>
  <zookeeper>
    <node>
      <host>192.168.56.101</host>
      <port>2181</port>
    </node>
    <node>
      <host>192.168.56.102</host>
      <port>2181</port>
    </node>
    <node>
      <host>192.168.56.103</host>
      <port>2181</port>
    </node>
  </zookeeper>
  <macros>
    <replica>node1</replica>
    <shard>01</shard>
  </macros>
</clickhouse>
```
В listen_host вносим ту подсеть, из которой нужен доступ к клику. У меня открыто для всех, но вы будьте лучше, чем я:)
Этот конфиг для всех нод одинаков за исключением параметра interserver_http_host и секции macros.
Секция macros для node2 выглядит так:
```
<macros>
    <replica>node2</replica>
    <shard>01</shard>
</macros>
```
А для node3 вот так:
```
<macros>
    <replica>node3</replica>
    <shard>01</shard>
</macros>
```
# 5. Запускаем clickhouse-server
```
sudo systemctl start clickhouse-server
```
# 6. Создаём реплицированную таблицу, что-нибудь в неё вставляем и проверяем, что вставилось:
```
CREATE TABLE replica_test ON CLUSTER 'default' (`a` String) ENGINE = ReplicatedMergeTree('/clickhouse/shard_{shard}/{database}/{table}', '{replica}') ORDER BY a
INSERT INTO replica_test (*) VALUES ('test')
SELECT * FROM replica_test
```
# 7. Проверить статус репликации можно вот так:
```
SELECT *
FROM system.replicas
WHERE table = 'replica_test'
FORMAT Vertical
```
Параметры total_replicas и active_replicas должны быть равны 3, а в параметре replica_is_active должны быть перечислены три реплики. Ещё можно посмотреть на параметр is_leader. У меня версия 24.2.1 и лидеры - все три ноды.
# 8. Проверяем на остальных двух нодах, что там появилась реплицированная таблица и в ней есть запись:
```
show tables
SELECT * FROM replica_test
```
# 9. Останавливаем на первой ноде службу clickhouse-server. После этого на других двух нодах по-прежнему доступны вставка и чтение, а при запуске службы на первой ноде, на ней подтягиваются изменения.




