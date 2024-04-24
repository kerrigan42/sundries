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
Во второй сессии запустим обновление тех же строк, изменение которых уже запущено в транзакции в первой сессии:
```
BEGIN;
UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
```
Операция обновления не происходит, потому что транзакция, открытая в первой сессии, блокирует строки. Вернёмся в первую сессию и завершим транзакцию:
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
Смоделируем ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах и посмотрим на блокировки.
Завершим транзакцию во второй сессии.
Начнём транзакции во всех трёх сессиях и посмотрим, какие у них номера обслуживающего процесса:
```
BEGIN;
SELECT pg_backend_pid();
```
Номер процесса по сессиям: сессия1 - 902, сессия2 - 973, сессия3 - 1033.
Теперь в каждой сессии запускаем обновление одной и той же строки:
```
UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
```
В первой транзакции команда выполнилась. В остальных двух сессиях команда не выполняется из-за блокировок. Посмотрим на возникшие блокировки в представлении pg_locks в первой сессии:
```
postgres=*# SELECT pid,
       locktype,
       CASE locktype
         WHEN 'relation' THEN relation::regclass::text
         WHEN 'transactionid' THEN transactionid::text
         WHEN 'tuple' THEN relation::regclass::text||':'||tuple::text
       END AS lockid,
       mode,
       granted
FROM pg_locks
WHERE locktype in ('relation','transactionid','tuple')
AND (locktype != 'relation' OR relation = 'accounts'::regclass);
 pid  |   locktype    |   lockid   |       mode       | granted 
------+---------------+------------+------------------+---------
 1033 | relation      | accounts   | AccessShareLock  | t
 1033 | relation      | accounts   | RowExclusiveLock | t
  973 | relation      | accounts   | RowExclusiveLock | t
  902 | relation      | accounts   | RowExclusiveLock | t
 1033 | transactionid | 741        | ExclusiveLock    | t
  902 | transactionid | 739        | ExclusiveLock    | t
 1033 | tuple         | accounts:4 | ExclusiveLock    | f
  973 | transactionid | 739        | ShareLock        | f
  973 | transactionid | 740        | ExclusiveLock    | t
  973 | tuple         | accounts:4 | ExclusiveLock    | t
(10 rows)
```
В первой сессии номер процесса 902 и в выводе мы видим, что транзакция удерживает исключительную блокировку номера своей транзакции, потому что транзакцию мы открыли и ещё не закрыли. Так же эта транзакция удерживает исключительную блокировку строки в таблице accounts, потому что в транзакции мы обновляем строку.
Во второй сессии номер процесса 973 и касательно этого процесса, мы видим, что он так же исключительно блокирует номер своей транзакции и строку в таблице accounts, потому что в этой сессии транзакция тоже ещё не закрыта и тоже происходит обновление строки. Кроме того, в этой транзакции есть ещё исключительная блокировка версии строки. Версию строки ранее блокировала транзакция в первой сессии, но она её уже отпустила, и блокировку версии строки захватила транзакция во второй сессии, но увидела, что эта строка сейчас заблокирована и поэтому нельзя прописать свой собственный xmax, и пока xmax не пропишет, блокировку версии строки не отпустит. И ещё транзакция ожидает получение блокировки типа transactionid в режиме ShareLock для первой транзакции, но пока что она эту блокировку не получает(granted = f).
Что касается третьей транзакции с номером процесса 1033, то она тоже хочет поставит исключительную блокировку на номер версии строки, чтобы прописать свой xmax и тоже пока не может. Также есть исключительная блокировка номера транзакции и блокировка строки в таблице accounts. И есть блокировка AccessShareLock на таблицу accounts, которая совместима со всеми остальными блокировками.


