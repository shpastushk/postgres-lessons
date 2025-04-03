1. Поднятие котейнеров для мастера и реплики
```
version: "3"

services:  
  postgres17:
    image: postgres:17
    container_name: postgresql17
    restart: always
    volumes:
      - /data/postgresql17:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: password

  postgresql17r:
    image: postgres:17
    container_name: postgresql17r
    restart: always
    volumes:
      - /data/postgresql17r/:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: password
```
```
$ docker compose up
$ docker ps
CONTAINER ID   IMAGE                      COMMAND                  CREATED          STATUS              PORTS                                         NAMES
dea6f02cde77   postgres:17                "docker-entrypoint.s…"   17 seconds ago   Up 15 seconds       5432/tcp                                      postgresql17
bec0b1529c79   postgres:17                "docker-entrypoint.s…"   17 seconds ago   Up 15 seconds       5432/tcp                                      postgresql17r

postgres@dea6f02cde77:/$ cat >> /var/lib/postgresql/data/pg_hba.conf << EOL
host replication replicator 172.21.0.0/16 scram-sha-256
EOL
-- перезапускаем контейнер
$ docker restart postgresql17 
postgresql17
-- заливаем тайские перевозки
wget https://storage.googleapis.com/thaibus/thai_small.tar.gz && tar -xf thai_small.tar.gz && psql < thai.sql
```
2. Создание пользователя для реплики и слота для репликации
```
$ docker exec -it postgresql17 bash
root@dea6f02cde77:/# su postgres
postgres@dea6f02cde77:/$ psql -c "CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'secret$123';"
CREATE ROLE
postgres@dea6f02cde77:/$ psql -c "SELECT pg_create_physical_replication_slot('test');"
 pg_create_physical_replication_slot 
-------------------------------------
 (test,)
(1 row)
```
3. на реплике
```
root@5f12047f5f68:/# cd /var/lib/postgresql/data
root@5f12047f5f68:/var/lib/postgresql/data# rm -rf *
root@5f12047f5f68:/var/lib/postgresql/data# cd /
root@5f12047f5f68:/# ls /var/lib/postgresql/data
root@5f12047f5f68:/# su - postgres -c "pg_basebackup -h postgresql17 -U replicator -R -S test -D /var/lib/postgresql/data"
Password: 
```

4. состояние реплики 

```
postgres=# SELECT * FROM pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 68
usesysid         | 16387
usename          | replicator
application_name | walreceiver
client_addr      | 172.21.0.2
client_hostname  | 
client_port      | 59832
backend_start    | 2025-04-03 20:48:40.362745+00
backend_xmin     | 
state            | streaming
sent_lsn         | 0/22000168
write_lsn        | 0/22000168
flush_lsn        | 0/22000168
replay_lsn       | 0/22000168
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 0
sync_state       | async
reply_time       | 2025-04-03 20:53:40.131379+00
```

5. Остановим реплику
6. Нагрузка на запись
```
root@dea6f02cde77:/#cat > ~/workload2.sql << EOL
INSERT INTO book.tickets (fkRide, fio, contact, fkSeat)
VALUES (
	ceil(random()*100)
	, (array(SELECT fam FROM book.fam))[ceil(random()*110)]::text || ' ' ||
    (array(SELECT nam FROM book.nam))[ceil(random()*110)]::text
    ,('{"phone":"+7' || (1000000000::bigint + floor(random()*9000000000)::bigint)::text || '"}')::jsonb
    , ceil(random()*100));

EOL
```

```
root@dea6f02cde77:/# /usr/lib/postgresql/17/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload2.sql -n -U postgres -p 5432 thai
pgbench (17.4 (Debian 17.4-1.pgdg120+2))
transaction type: /root/workload2.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 9609
number of failed transactions: 0 (0.000%)
latency average = 8.315 ms
initial connection time = 17.222 ms
tps = 962.141066 (without initial connection time)
```

6. нагрузка на чтение

```
root@dea6f02cde77:/#cat > ~/workload.sql << EOL
\set r random(1, 5000000) 
SELECT id, fkRide, fio, contact, fkSeat FROM book.tickets WHERE id = :r;
EOL

root@dea6f02cde77:/# /usr/lib/postgresql/17/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload.sql -n -U postgres thai
pgbench (17.4 (Debian 17.4-1.pgdg120+2))
transaction type: /root/workload.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 210025
number of failed transactions: 0 (0.000%)
latency average = 0.379 ms
initial connection time = 50.602 ms
tps = 21108.534501 (without initial connection time)
```

7. Запустим реплику
```
postgres@5f12047f5f68:/$ psql -d thai -c "select pg_is_in_recovery();"
 pg_is_in_recovery 
-------------------
 t
(1 row)

postgres@5f12047f5f68:/$ psql -d thai -c "select count(*) from book.tickets;"
  count  
---------
 5202151
(1 row)
```
8. нагрузка на запись

```
root@dea6f02cde77:/# /usr/lib/postgresql/17/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload2.sql -n -U postgres -p 5432 thai
pgbench (17.4 (Debian 17.4-1.pgdg120+2))
transaction type: /root/workload2.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 7037
number of failed transactions: 0 (0.000%)
latency average = 11.366 ms
initial connection time = 14.553 ms
tps = 703.875476 (without initial connection time)
```

9. нагрузка на чтение

```
root@dea6f02cde77:/# /usr/lib/postgresql/17/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload.sql -n -U postgres thai
pgbench (17.4 (Debian 17.4-1.pgdg120+2))
transaction type: /root/workload.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 195696
number of failed transactions: 0 (0.000%)
latency average = 0.408 ms
initial connection time = 26.669 ms
tps = 19591.838696 (without initial connection time)
```
