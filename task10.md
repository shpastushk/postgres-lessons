1. Сгенерировать таблицу с 1 млн JSONB документов

```
thai=# drop table if exists t;
CREATE TABLE t AS                                             
SELECT i AS id, (SELECT jsonb_object_agg(j, j) FROM generate_series(1, 1000) j) js
FROM generate_series(1, 1000000) i;
DROP TABLE
SELECT 1000000

thai=# SELECT oid::regclass AS heap_rel,
       pg_size_pretty(pg_relation_size(oid)) AS heap_rel_size,
       reltoastrelid::regclass AS toast_rel,
       pg_size_pretty(pg_relation_size(reltoastrelid)) AS toast_rel_size
FROM pg_class WHERE relname = 't';
 heap_rel | heap_rel_size |        toast_rel         | toast_rel_size 
----------+---------------+--------------------------+----------------
 t        | 50 MB         | pg_toast.pg_toast_780565 | 7813 MB
(1 row)


```
2. Создать индекс
```
thai=# create index on t using gin(js);
CREATE INDEX

thai=# select pg_size_pretty(pg_indexes_size('t'));
 pg_size_pretty 
----------------
 2047 MB
(1 row)
```

3. Обновить 1 из полей в json
```
thai=# UPDATE t SET js = js::jsonb || '{"a":1}';
UPDATE 1000000
```

4. Убедится в блоатинге TOAST
```
thai=# SELECT oid::regclass AS heap_rel,
       pg_size_pretty(pg_relation_size(oid)) AS heap_rel_size,
       reltoastrelid::regclass AS toast_rel,
       pg_size_pretty(pg_relation_size(reltoastrelid)) AS toast_rel_size
FROM pg_class WHERE relname = 't';
 heap_rel | heap_rel_size |        toast_rel         | toast_rel_size 
----------+---------------+--------------------------+----------------
 t        | 100 MB        | pg_toast.pg_toast_780565 | 15 GB
(1 row)

thai=# select pg_size_pretty(pg_indexes_size('t'));
 pg_size_pretty 
----------------
 4570 MB
(1 row)
```

5. Придумать метод избавится от него и проверить на практике

1 - vacuum full убирает распухание
```
thai=# vacuum full;
VACUUM
thai=# SELECT oid::regclass AS heap_rel,
       pg_size_pretty(pg_relation_size(oid)) AS heap_rel_size,
       reltoastrelid::regclass AS toast_rel,
       pg_size_pretty(pg_relation_size(reltoastrelid)) AS toast_rel_size
FROM pg_class WHERE relname = 't';
 heap_rel | heap_rel_size |        toast_rel         | toast_rel_size 
----------+---------------+--------------------------+----------------
 t        | 50 MB         | pg_toast.pg_toast_780565 | 7813 MB
(1 row)

thai=# select pg_size_pretty(pg_indexes_size('t'));
 pg_size_pretty 
----------------
 2048 MB
(1 row)
```

2 - поменять стратегию хранения столбца на main
```
thai=# alter table t alter column js set storage main;
ALTER TABLE
thai=# 
thai=# SELECT oid::regclass AS heap_rel,
       pg_size_pretty(pg_relation_size(oid)) AS heap_rel_size,
       reltoastrelid::regclass AS toast_rel,
       pg_size_pretty(pg_relation_size(reltoastrelid)) AS toast_rel_size
FROM pg_class WHERE relname = 't';
 heap_rel | heap_rel_size |        toast_rel         | toast_rel_size 
----------+---------------+--------------------------+----------------
 t        | 50 MB         | pg_toast.pg_toast_780565 | 7813 MB
(1 row)

thai=# UPDATE t SET js = js::jsonb || '{"a":1}';
UPDATE 1000000
thai=# SELECT oid::regclass AS heap_rel,
       pg_size_pretty(pg_relation_size(oid)) AS heap_rel_size,
       reltoastrelid::regclass AS toast_rel,
       pg_size_pretty(pg_relation_size(reltoastrelid)) AS toast_rel_size
FROM pg_class WHERE relname = 't';
 heap_rel | heap_rel_size |        toast_rel         | toast_rel_size 
----------+---------------+--------------------------+----------------
 t        | 7862 MB       | pg_toast.pg_toast_780565 | 7813 MB
(1 row)
thai=# vacuum t;
VACUUM
thai=# SELECT oid::regclass AS heap_rel,
       pg_size_pretty(pg_relation_size(oid)) AS heap_rel_size,
       reltoastrelid::regclass AS toast_rel,
       pg_size_pretty(pg_relation_size(reltoastrelid)) AS toast_rel_size
FROM pg_class WHERE relname = 't';
 heap_rel | heap_rel_size |        toast_rel         | toast_rel_size 
----------+---------------+--------------------------+----------------
 t        | 7862 MB       | pg_toast.pg_toast_780565 | 0 bytes
(1 row)
```
