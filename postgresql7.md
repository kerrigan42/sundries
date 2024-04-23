### Механизм блокировок ###
Установим и запустим postgresql 15:
```
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15
sudo pg_ctlcluster 15 main start
```
Настроим сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд:
```
sudo -u postgres psql
ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM SET deadlock_timeout = 200;
SELECT pg_reload_conf();
```
Воспроизведём ситуацию, при которой в журнале появятся такие сообщения. Для этого нам понадобится две сессии.
В первой сессии создадим таблицу, заполним её значениями и начнём транзакцию с обновлением строк:
```
CREATE TABLE accounts(
  acc_no integer PRIMARY KEY,
  amount numeric
);
INSERT INTO accounts VALUES (1,1000.00), (2,2000.00), (3,3000.00);
BEGIN;
UPDATE accounts SET amount = amount - 100.00 WHERE acc_no = 1;
```
Во второй сессии запустим обновление тех же строк, про которые уже запущена транзакция в первой сессии:
```
BEGIN;
UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
```
Операция обновления не происходит, потому что транзакция, открытая в первой сессии блокирует строки. Вернёмся в первую сессию и завершим транзакцию:
```
COMMIT;
```
Теперь во второй сессии операция обновления тоже завершилась. 
В логе можно увидеть сообщения о блокировке:
```
user@ubuntu1:~$ sudo tail -n 10 /var/log/postgresql/postgresql-15-main.log
2024-04-23 17:21:14.403 UTC [4831] user@user FATAL:  role "user" does not exist
2024-04-23 17:21:42.608 UTC [4700] LOG:  checkpoint starting: time
2024-04-23 17:21:47.020 UTC [4700] LOG:  checkpoint complete: wrote 40 buffers (0.2%); 0 WAL file(s) added, 0 removed, 0 recycled; write=3.951 s, sync=0.293 s, total=4.412 s; sync files=35, longest=0.031 s, average=0.009 s; distance=149 kB, estimate=242 kB
2024-04-23 17:21:59.428 UTC [4836] postgres@postgres LOG:  process 4836 still waiting for ShareLock on transaction 737 after 200.861 ms
2024-04-23 17:21:59.428 UTC [4836] postgres@postgres DETAIL:  Process holding the lock: 4719. Wait queue: 4836.
2024-04-23 17:21:59.428 UTC [4836] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "accounts"
2024-04-23 17:21:59.428 UTC [4836] postgres@postgres STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
2024-04-23 17:22:28.161 UTC [4836] postgres@postgres LOG:  process 4836 acquired ShareLock on transaction 737 after 28932.968 ms
2024-04-23 17:22:28.161 UTC [4836] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "accounts"
2024-04-23 17:22:28.161 UTC [4836] postgres@postgres STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
```
