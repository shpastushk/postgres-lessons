1. Установить 16 ПГ

Поднятие контейнеров для 16 и 17 версии postgresql
```
version: "3"

services:  
  postgresql16:
    image: postgres:16
    container_name: postgresql16-new
    restart: always
    volumes:
      - /data/postgresql16-new:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: password
    ports:
      - "8432:5432"

  postgresql17:
    image: postgres:17
    container_name: postgresql17-new
    restart: always
    volumes:
      - /data/postgresql17-new/:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: password
    ports:
      - "9432:5432"
```

```
$ sudo docker compose up
$ sudo docker ps
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS         PORTS                                         NAMES
625d36789de2   postgres:17   "docker-entrypoint.s…"   17 seconds ago   Up 9 seconds   0.0.0.0:9432->5432/tcp, [::]:9432->5432/tcp   postgresql17-new
9e476d749746   postgres:16   "docker-entrypoint.s…"   17 seconds ago   Up 9 seconds   0.0.0.0:8432->5432/tcp, [::]:8432->5432/tcp   postgresql16-new
```

2. Залить средние Тайские перевозки

```
$ wget https://storage.googleapis.com/thaibus/thai_medium.tar.gz && tar -xf thai_medium.tar.gz && psql -h localhost -p 8432 -d postgres -U postgres < thai.sql
$ psql -h localhost -p 8432 -d thai -U postgres -c "select count(*) from book.tickets;"
Password for user postgres: 
  count   
----------
 53997475
(1 row)

```

3. Рядом поднять кластер 17 версии

```
$ sudo docker ps
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS         PORTS                                         NAMES
625d36789de2   postgres:17   "docker-entrypoint.s…"   17 seconds ago   Up 9 seconds   0.0.0.0:9432->5432/tcp, [::]:9432->5432/tcp   postgresql17-new
9e476d749746   postgres:16   "docker-entrypoint.s…"   17 seconds ago   Up 9 seconds   0.0.0.0:8432->5432/tcp, [::]:8432->5432/tcp   postgresql16-new
```

4. Логическая репликация

Создаем юзера для репликации

```
$ sudo docker exec -it postgresql16-new bash
root@9e476d749746:/# su postgres
postgres@9e476d749746:/$ psql -c "CREATE USER replicator5 WITH REPLICATION ENCRYPTED PASSWORD 'secret123';"
CREATE ROLE
postgres@9e476d749746:/$ psql -c "grant all on all tables in schema book to replicator5;"
GRANT
postgres@9e476d749746:/$ psql -c "ALTER SYSTEM SET wal_level = logical;"
ALTER SYSTEM
$ sudo docker restart 9e476d749746
```

```
$ sudo docker exec -it postgresql17-new bash
root@625d36789de2:/# su postgres
postgres@625d36789de2:/$ psql -c "ALTER SYSTEM SET wal_level = logical;"
ALTER SYSTEM
$ sudo docker restart 625d36789de2
```

На первом сервере создаем публикацию

```
$ sudo docker exec -it postgresql16-new bash
root@9e476d749746:/# su postgres
postgres@9e476d749746:/$ psql -d thai
psql (16.6 (Debian 16.6-1.pgdg120+1))
Type "help" for help.

thai=# CREATE PUBLICATION test_pub FOR TABLES IN SCHEMA book;
CREATE PUBLICATION
```

На втором сервере создаем объекты 

```
$ pg_dumpall -s -h localhost -p 8432 -l thai -U postgres > schema.sql
$ psql -h localhost -p 9432  -U postgres < schema.sql
```

На втором сервере создаем подписку

```
thai=# CREATE SUBSCRIPTION test_sub 
CONNECTION 'host=postgresql16-new user=replicator5 password=secret123 dbname=thai' 
PUBLICATION test_pub WITH (copy_data = true);
NOTICE:  created replication slot "test_sub" on publisher
CREATE SUBSCRIPTION
Time: 414.397 ms

thai=# SELECT * FROM pg_stat_subscription \gx
-[ RECORD 1 ]---------+------------------------------
subid                 | 16526
subname               | test_sub
worker_type           | table synchronization
pid                   | 1295
leader_pid            | 
relid                 | 16431
received_lsn          | 
last_msg_send_time    | 2025-05-22 19:38:51.070008+00
last_msg_receipt_time | 2025-05-22 19:38:51.070008+00
latest_end_lsn        | 
latest_end_time       | 2025-05-22 19:38:51.070008+00
-[ RECORD 2 ]---------+------------------------------
subid                 | 16526
subname               | test_sub
worker_type           | apply
pid                   | 1291
leader_pid            | 
relid                 | 
received_lsn          | 2/8868B9C0
last_msg_send_time    | 2025-05-22 19:51:16.429222+00
last_msg_receipt_time | 2025-05-22 19:51:16.429372+00
latest_end_lsn        | 2/8868B9C0
latest_end_time       | 2025-05-22 19:51:16.429222+00

Time: 267.607 ms
```

На первом сервере

```
-[ RECORD 1 ]----+----------------------------------------
pid              | 93
usesysid         | 32903
usename          | replicator5
application_name | test_sub
client_addr      | 172.21.0.3
client_hostname  | 
client_port      | 53786
backend_start    | 2025-05-22 19:38:50.656007+00
backend_xmin     | 
state            | streaming
sent_lsn         | 2/8868B9C0
write_lsn        | 2/8868B9C0
flush_lsn        | 2/8868B9C0
replay_lsn       | 2/8868B9C0
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 0
sync_state       | async
reply_time       | 2025-05-22 20:00:17.161514+00
-[ RECORD 2 ]----+----------------------------------------
pid              | 97
usesysid         | 32903
usename          | replicator5
application_name | pg_16526_sync_16431_7507306859512381479
client_addr      | 172.21.0.3
client_hostname  | 
client_port      | 53822
backend_start    | 2025-05-22 19:38:51.075142+00
backend_xmin     | 990
state            | startup
sent_lsn         | 
write_lsn        | 
flush_lsn        | 
replay_lsn       | 
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 0
sync_state       | async
reply_time       | 

thai=# \dt+ book.*
                                         List of relations
 Schema |     Name     | Type  |  Owner   | Persistence | Access method |    Size    | Description 
--------+--------------+-------+----------+-------------+---------------+------------+-------------
 book   | bus          | table | postgres | permanent   | heap          | 16 kB      | 
 book   | busroute     | table | postgres | permanent   | heap          | 8192 bytes | 
 book   | busstation   | table | postgres | permanent   | heap          | 16 kB      | 
 book   | fam          | table | postgres | permanent   | heap          | 16 kB      | 
 book   | nam          | table | postgres | permanent   | heap          | 16 kB      | 
 book   | ride         | table | postgres | permanent   | heap          | 64 MB      | 
 book   | schedule     | table | postgres | permanent   | heap          | 128 kB     | 
 book   | seat         | table | postgres | permanent   | heap          | 40 kB      | 
 book   | seatcategory | table | postgres | permanent   | heap          | 16 kB      | 
 book   | tickets      | table | postgres | permanent   | heap          | 4796 MB    | 
(10 rows)

thai=# select count(*) from book.tickets;
  count   
----------
 53997475
(1 row)
```
Данные копировались с 2025-05-22 19:38:50 до 2025-05-22 20:09:48, где-то 30 минут вышло

На втором сервере

```
thai=# \dt+ book.*
                                         List of relations
 Schema |     Name     | Type  |  Owner   | Persistence | Access method |    Size    | Description 
--------+--------------+-------+----------+-------------+---------------+------------+-------------
 book   | bus          | table | postgres | permanent   | heap          | 16 kB      | 
 book   | busroute     | table | postgres | permanent   | heap          | 8192 bytes | 
 book   | busstation   | table | postgres | permanent   | heap          | 16 kB      | 
 book   | fam          | table | postgres | permanent   | heap          | 16 kB      | 
 book   | nam          | table | postgres | permanent   | heap          | 16 kB      | 
 book   | ride         | table | postgres | permanent   | heap          | 64 MB      | 
 book   | schedule     | table | postgres | permanent   | heap          | 128 kB     | 
 book   | seat         | table | postgres | permanent   | heap          | 40 kB      | 
 book   | seatcategory | table | postgres | permanent   | heap          | 16 kB      | 
 book   | tickets      | table | postgres | permanent   | heap          | 4795 MB    | 
(10 rows)

thai=# SELECT * FROM pg_stat_subscription \gx
-[ RECORD 1 ]---------+------------------------------
subid                 | 16526
subname               | test_sub
worker_type           | apply
pid                   | 1291
leader_pid            | 
relid                 | 
received_lsn          | 2/8868B9C0
last_msg_send_time    | 2025-05-22 20:09:48.950476+00
last_msg_receipt_time | 2025-05-22 20:09:48.950593+00
latest_end_lsn        | 2/8868B9C0
latest_end_time       | 2025-05-22 20:09:48.950476+00

Time: 0.718 ms
thai=# select count(*) from book.tickets;
  count   
----------
 53997475
(1 row)

Time: 555676.349 ms (09:15.676)
```



