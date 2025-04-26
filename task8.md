1. Создать таблицу с продажами.
```
CREATE TABLE sales (itemno int, price int, quantity int, created_at timestamptz);

INSERT INTO sales VALUES (1,10,100, '2025-04-20 18:01:34.515 +0300'),
(2,20,200, '2025-06-20 18:01:34.515 +0300'), 
(3,30,300, '2025-12-20 18:01:34.515 +0300'), 
(4,40,400, null);

INSERT INTO sales VALUES (5,50,500, '2025-01-20 18:01:34.515 +0300'),
(6,60,600, '2025-11-20 18:01:34.515 +0300'), 
(7,70,700, '2025-09-20 18:01:34.515 +0300'), 
(8,80,800, null);
```


2. Реализовать функцию выбор трети года (1-4 мес - первая треть, 5-8 - вторая и т.д.)

```
CREATE or replace FUNCTION partYear(ts timestamptz) RETURNS integer AS $$
 	SELECT CASE WHEN date_part('month', ts) in (1,2,3,4) THEN 1 WHEN date_part('month', ts) in (5,6,7,8) THEN 2 ELSE 3 END;
$$ LANGUAGE sql
RETURNS NULL ON NULL INPUT;

CREATE or replace FUNCTION partYearDiv(ts timestamptz) RETURNS integer AS $$
 	SELECT (date_part('month', ts)::int - 1) / 4 + 1
$$ LANGUAGE sql
RETURNS NULL ON NULL INPUT;

CREATE or replace FUNCTION partYearShift(ts timestamptz) RETURNS integer AS $$
 	SELECT ((date_part('month', ts)::int - 1) >> 2) + 1
$$ LANGUAGE sql
RETURNS NULL ON NULL INPUT;
```

3. Вызвать эту функцию в SELECT из таблицы с продажами, убедиться, что всё отработало

```
select *, date_part('month', created_at), partYear(created_at), partYearDiv(created_at), partYearShift(created_at)  from sales order by created_at 
```

```
thai=# select *, date_part('month', created_at), partYear(created_at), partYearDiv(created_at), partYearShift(created_at)  from sales order by created_at;
 itemno | price | quantity |         created_at         | date_part | partyear | partyeardiv | partyearshift 
--------+-------+----------+----------------------------+-----------+----------+-------------+---------------
      5 |    50 |      500 | 2025-01-20 15:01:34.515+00 |         1 |        1 |           1 |             1
      1 |    10 |      100 | 2025-04-20 15:01:34.515+00 |         4 |        1 |           1 |             1
      2 |    20 |      200 | 2025-06-20 15:01:34.515+00 |         6 |        2 |           2 |             2
      7 |    70 |      700 | 2025-09-20 15:01:34.515+00 |         9 |        3 |           3 |             3
      6 |    60 |      600 | 2025-11-20 15:01:34.515+00 |        11 |        3 |           3 |             3
      3 |    30 |      300 | 2025-12-20 15:01:34.515+00 |        12 |        3 |           3 |             3
      4 |    40 |      400 |                            |           |          |             |              
      8 |    80 |      800 |                            |           |          |             |              
(8 rows)
```
