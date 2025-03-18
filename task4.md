1. Создать таблицу accounts(id integer, amount numeric);
```
test=# CREATE TABLE accounts(id integer, amount numeric);
CREATE TABLE
```
2. Добавить несколько записей и подключившись через 2 терминала добиться ситуации взаимоблокировки (deadlock).

- добавление записей
```
test=# insert into accounts values (1, 20), (2, 30), (3,40);
INSERT 0 3
test=# select * from accounts;
 id | amount 
----+--------
  1 |     20
  2 |     30
  3 |     40
(3 rows)
```

- 1 терминал
```
test=# begin;
BEGIN
test=*# update accounts set amount = 25 where id = 1;
UPDATE 1
test=*# 
```

- 2 терминал
```
test=# begin;
BEGIN
test=*# update accounts set amount = 35 where id = 2;
UPDATE 1
test=*#
```

- 1 терминал
```
test=*# update accounts set amount = 35 where id = 2;
```

- 2 терминал
```
test=*# update accounts set amount = 25 where id = 1;
ERROR:  deadlock detected
DETAIL:  Process 52 waits for ShareLock on transaction 884; blocked by process 42.
Process 42 waits for ShareLock on transaction 885; blocked by process 52.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "accounts"
```


3. Посмотреть логи и убедиться, что информация о дедлоке туда попала.
```
2025-03-18 08:36:44.194 UTC [52] ERROR:  deadlock detected
2025-03-18 08:36:44.194 UTC [52] DETAIL:  Process 52 waits for ShareLock on transaction 884; blocked by process 42.
        Process 42 waits for ShareLock on transaction 885; blocked by process 52.
        Process 52: update accounts set amount = 25 where id = 1;
        Process 42: update accounts set amount = 35 where id = 2;
2025-03-18 08:36:44.194 UTC [52] HINT:  See server log for query details.
2025-03-18 08:36:44.194 UTC [52] CONTEXT:  while updating tuple (0,1) in relation "accounts"
```
