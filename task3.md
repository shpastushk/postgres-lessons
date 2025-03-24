1. Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1 млн строк

```
test=# CREATE TABLE test(value text);
CREATE TABLE
test=# INSERT INTO test SELECT s.id FROM generate_series(1,1000000) AS s(id);
INSERT 0 1000000
test=# select count(*) from test;
  count  
---------
 1000000
(1 row)

test=# select value from test limit 3;
 value 
-------
 1
 2
 3
(3 rows)
```

2. Посмотреть размер файла с таблицей

```
test=# \dt+ test
                                  List of relations
 Schema | Name | Type  |  Owner   | Persistence | Access method | Size  | Description 
--------+------+-------+----------+-------------+---------------+-------+-------------
 public | test | table | postgres | permanent   | heap          | 35 MB | 
(1 row)
```

3. 5 раз обновить все строчки и добавить к каждой строчке любой символ

```
test=# update test set value = value || 'a';
UPDATE 1000000
test=# update test set value = value || 'b';
UPDATE 1000000
test=# update test set value = value || 'c';
UPDATE 1000000
test=# update test set value = value || 'd';
UPDATE 1000000
test=# update test set value = value || 'e';
UPDATE 1000000
```

4. Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум

```
test=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum
FROM pg_stat_user_tables WHERE relname = 'test';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 test    |    1000000 |    4999870 |    499 | 2025-03-22 18:13:08.634023+00
(1 row)
```

5. Подождать некоторое время, проверяя, пришел ли автовакуум

```
test=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum
FROM pg_stat_user_tables WHERE relname = 'test';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 test    |    1001020 |          0 |      0 | 2025-03-22 18:14:14.367616+00
(1 row)
```

6. 5 раз обновить все строчки и добавить к каждой строчке любой символ

```
test=# update test set value = value || 'a';
UPDATE 1000000
test=# update test set value = value || 'b';
UPDATE 1000000
test=# update test set value = value || 'c';
UPDATE 1000000
test=# update test set value = value || 'd';
UPDATE 1000000
test=# update test set value = value || 'e';
UPDATE 1000000
```

7. Посмотреть размер файла с таблицей

```
test=# \dt+ test
                                   List of relations
 Schema | Name | Type  |  Owner   | Persistence | Access method |  Size  | Description 
--------+------+-------+----------+-------------+---------------+--------+-------------
 public | test | table | postgres | permanent   | heap          | 260 MB | 
(1 row)
```

8. Отключить Автовакуум на конкретной таблице

```
test=# ALTER TABLE test SET (autovacuum_enabled = off);
ALTER TABLE
```

9. 10 раз обновить все строчки и добавить к каждой строчке любой символ

```
test=# DO
$do$
BEGIN
    FOR i IN 1..10
    LOOP
        update test set value = value || 'm';
        RAISE NOTICE 'i %', i;
    END LOOP;
END
$do$;
NOTICE:  i 1
NOTICE:  i 2
NOTICE:  i 3
NOTICE:  i 4
NOTICE:  i 5
NOTICE:  i 6
NOTICE:  i 7
NOTICE:  i 8
NOTICE:  i 9
NOTICE:  i 10
DO
```

10. Посмотреть размер файла с таблицей

```
test=# \dt+ test
                                   List of relations
 Schema | Name | Type  |  Owner   | Persistence | Access method |  Size  | Description 
--------+------+-------+----------+-------------+---------------+--------+-------------
 public | test | table | postgres | permanent   | heap          | 780 MB | 
(1 row)
```

11. Объясните полученный результат

Автовакуум находит все мертвые кортежи и запоминает свободные участки в существующих файлах. При этом сам размер таблицы не уменьшается. Последующие апдейты будут записываться в свободные участки. Размер таблицы не будет меняться во столько же раз как при первых апдейтах. 

12. Не забудьте включить автовакуум

Задание со *:

Написать анонимную процедуру, в которой в цикле 10 раз обновятся все строчки в искомой таблице. 
Не забыть вывести номер шага цикла.

```
test=# DO
$do$
BEGIN
    FOR i IN 1..10
    LOOP
        update test set value = value || 'm';
        RAISE NOTICE 'i %', i;
    END LOOP;
END
$do$;
NOTICE:  i 1
NOTICE:  i 2
NOTICE:  i 3
NOTICE:  i 4
NOTICE:  i 5
NOTICE:  i 6
NOTICE:  i 7
NOTICE:  i 8
NOTICE:  i 9
NOTICE:  i 10
DO
```
