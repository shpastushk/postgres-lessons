1. Развернуть ВМ (Linux) с PostgreSQL
2. Залить Тайские перевозки
https://github.com/aeuge/postgres16book/tree/main/database

Тайские перевозки были залиты в рамках первой лабораторной.

3. Проверить скорость выполнения сложного запроса (приложен в коне файла скриптов)
```
explain analyze
WITH all_place AS (
    SELECT count(s.id) as all_place, s.fkbus as fkbus
    FROM book.seat s
    group by s.fkbus
),
order_place AS (
    SELECT count(t.id) as order_place, t.fkride
    FROM book.tickets t
    group by t.fkride
)
SELECT r.id, r.startdate as depart_date, bs.city || ', ' || bs.name as busstation,  
      t.order_place, st.all_place
FROM book.ride r
JOIN book.schedule as s
      on r.fkschedule = s.id
JOIN book.busroute br
      on s.fkroute = br.id
JOIN book.busstation bs
      on br.fkbusstationfrom = bs.id
JOIN order_place t
      on t.fkride = r.id
JOIN all_place st
      on r.fkbus = st.fkbus
GROUP BY r.id, r.startdate, bs.city || ', ' || bs.name, t.order_place,st.all_place
ORDER BY r.startdate
limit 10;
```
план выполнения
```
Limit  (cost=328351.02..328351.05 rows=10 width=56) (actual time=2496.591..2496.694 rows=10 loops=1)
  ->  Sort  (cost=328351.02..328707.84 rows=142728 width=56) (actual time=2453.567..2453.670 rows=10 loops=1)
        Sort Key: r.startdate
        Sort Method: top-N heapsort  Memory: 25kB
        ->  Group  (cost=322768.98..325266.72 rows=142728 width=56) (actual time=2379.885..2427.908 rows=144000 loops=1)
              Group Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))
              ->  Sort  (cost=322768.98..323125.80 rows=142728 width=56) (actual time=2379.852..2396.789 rows=144000 loops=1)
                    Sort Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))
                    Sort Method: external merge  Disk: 7864kB
                    ->  Hash Join  (cost=257231.86..305670.39 rows=142728 width=56) (actual time=1893.435..2334.013 rows=144000 loops=1)
                          Hash Cond: (r.fkbus = s_1.fkbus)
                          ->  Nested Loop  (cost=257226.75..304259.40 rows=142728 width=84) (actual time=1893.226..2285.584 rows=144000 loops=1)
                                ->  Hash Join  (cost=257226.60..300862.68 rows=142728 width=24) (actual time=1893.186..2218.103 rows=144000 loops=1)
                                      Hash Cond: (s.fkroute = br.id)
                                      ->  Hash Join  (cost=257224.25..300459.20 rows=142728 width=24) (actual time=1893.126..2183.951 rows=144000 loops=1)
                                            Hash Cond: (r.fkschedule = s.id)
                                            ->  Merge Join  (cost=257180.85..300040.04 rows=142728 width=24) (actual time=1892.352..2148.138 rows=144000 loops=1)
                                                  Merge Cond: (r.id = t.fkride)
                                                  ->  Index Scan using ride_pkey on ride r  (cost=0.42..4555.42 rows=144000 width=16) (actual time=0.031..32.329 rows=144000 loops=1)
                                                  ->  Finalize GroupAggregate  (cost=257180.43..293340.52 rows=142728 width=12) (actual time=1892.296..2068.784 rows=144000 loops=1)
                                                        Group Key: t.fkride
                                                        ->  Gather Merge  (cost=257180.43..290485.96 rows=285456 width=12) (actual time=1892.250..2003.809 rows=431998 loops=1)
                                                              Workers Planned: 2
                                                              Workers Launched: 2
                                                              ->  Sort  (cost=256180.41..256537.23 rows=142728 width=12) (actual time=1837.386..1861.736 rows=143999 loops=3)
                                                                    Sort Key: t.fkride
                                                                    Sort Method: external merge  Disk: 3672kB
                                                                    Worker 0:  Sort Method: external merge  Disk: 3672kB
                                                                    Worker 1:  Sort Method: external merge  Disk: 3672kB
                                                                    ->  Partial HashAggregate  (cost=218994.54..241521.32 rows=142728 width=12) (actual time=1446.542..1761.624 rows=143999 loops=3)
                                                                          Group Key: t.fkride
                                                                          Planned Partitions: 4  Batches: 5  Memory Usage: 8241kB  Disk Usage: 25360kB
                                                                          Worker 0:  Batches: 5  Memory Usage: 8241kB  Disk Usage: 27496kB
                                                                          Worker 1:  Batches: 5  Memory Usage: 8241kB  Disk Usage: 27776kB
                                                                          ->  Parallel Seq Scan on tickets t  (cost=0.00..80581.88 rows=2160588 width=12) (actual time=0.070..316.091 rows=1728502 loops=3)
                                            ->  Hash  (cost=25.40..25.40 rows=1440 width=8) (actual time=0.730..0.730 rows=1440 loops=1)
                                                  Buckets: 2048  Batches: 1  Memory Usage: 73kB
                                                  ->  Seq Scan on schedule s  (cost=0.00..25.40 rows=1440 width=8) (actual time=0.010..0.296 rows=1440 loops=1)
                                      ->  Hash  (cost=1.60..1.60 rows=60 width=8) (actual time=0.048..0.049 rows=60 loops=1)
                                            Buckets: 1024  Batches: 1  Memory Usage: 11kB
                                            ->  Seq Scan on busroute br  (cost=0.00..1.60 rows=60 width=8) (actual time=0.014..0.026 rows=60 loops=1)
                                ->  Memoize  (cost=0.15..0.36 rows=1 width=68) (actual time=0.000..0.000 rows=1 loops=144000)
                                      Cache Key: br.fkbusstationfrom
                                      Cache Mode: logical
                                      Hits: 143990  Misses: 10  Evictions: 0  Overflows: 0  Memory Usage: 2kB
                                      ->  Index Scan using busstation_pkey on busstation bs  (cost=0.14..0.35 rows=1 width=68) (actual time=0.004..0.004 rows=1 loops=10)
                                            Index Cond: (id = br.fkbusstationfrom)
                          ->  Hash  (cost=5.05..5.05 rows=5 width=12) (actual time=0.191..0.192 rows=5 loops=1)
                                Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                ->  HashAggregate  (cost=5.00..5.05 rows=5 width=12) (actual time=0.181..0.183 rows=5 loops=1)
                                      Group Key: s_1.fkbus
                                      Batches: 1  Memory Usage: 24kB
                                      ->  Seq Scan on seat s_1  (cost=0.00..4.00 rows=200 width=8) (actual time=0.021..0.047 rows=200 loops=1)
Planning Time: 0.782 ms
JIT:
  Functions: 80
  Options: Inlining false, Optimization false, Expressions true, Deforming true
  Timing: Generation 13.771 ms (Deform 1.622 ms), Inlining 0.000 ms, Optimization 2.871 ms, Emission 73.568 ms, Total 90.210 ms
Execution Time: 2504.692 ms
```
время выполнения - 2504.692 ms

3. Навесить индекс на внешние ключи
```
CREATE INDEX idx_ride_fkschedule on book.ride(fkschedule);
CREATE INDEX idx_ride_fkbus on book.ride(fkbus);
CREATE INDEX idx_schedule_fkroute on book.schedule(fkroute);
CREATE INDEX idx_busroute_fkbusstationfrom on book.busroute(fkbusstationfrom);
CREATE INDEX idx_tickets_fkride on book.tickets(fkride);
CREATE INDEX idx_tickets_fkseat on book.tickets(fkseat);
CREATE INDEX idx_seat_fkbus on book.seat(fkbus);
```

4. Проверит, помогли ли индексы на внешние ключи ускориться
```
Limit  (cost=329770.03..329770.06 rows=10 width=56) (actual time=2236.373..2242.690 rows=10 loops=1)
  ->  Sort  (cost=329770.03..330123.83 rows=141518 width=56) (actual time=2202.201..2208.516 rows=10 loops=1)
        Sort Key: r.startdate
        Sort Method: top-N heapsort  Memory: 25kB
        ->  Group  (cost=277296.19..326711.88 rows=141518 width=56) (actual time=1781.854..2174.814 rows=144000 loops=1)
              Group Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))
              ->  Incremental Sort  (cost=277296.19..324589.11 rows=141518 width=56) (actual time=1781.846..2133.731 rows=144000 loops=1)
                    Sort Key: r.id, (((bs.city || ', '::text) || bs.name)), (count(t.id)), (count(s_1.id))
                    Presorted Key: r.id
                    Full-sort Groups: 4500  Sort Method: quicksort  Average Memory: 27kB  Peak Memory: 27kB
                    ->  Merge Join  (cost=277067.44..316117.54 rows=141518 width=56) (actual time=1781.713..2088.848 rows=144000 loops=1)
                          Merge Cond: (r.id = t.fkride)
                          ->  Sort  (cost=20076.71..20436.71 rows=144000 width=32) (actual time=186.479..209.115 rows=144000 loops=1)
                                Sort Key: r.id
                                Sort Method: external merge  Disk: 6496kB
                                ->  Hash Join  (cost=52.09..4291.50 rows=144000 width=32) (actual time=0.551..137.876 rows=144000 loops=1)
                                      Hash Cond: (r.fkbus = s_1.fkbus)
                                      ->  Hash Join  (cost=46.98..3587.99 rows=144000 width=28) (actual time=0.451..110.363 rows=144000 loops=1)
                                            Hash Cond: (br.fkbusstationfrom = bs.id)
                                            ->  Hash Join  (cost=45.75..3048.56 rows=144000 width=16) (actual time=0.424..81.946 rows=144000 loops=1)
                                                  Hash Cond: (s.fkroute = br.id)
                                                  ->  Hash Join  (cost=43.40..2641.51 rows=144000 width=16) (actual time=0.390..54.928 rows=144000 loops=1)
                                                        Hash Cond: (r.fkschedule = s.id)
                                                        ->  Seq Scan on ride r  (cost=0.00..2219.00 rows=144000 width=16) (actual time=0.004..12.635 rows=144000 loops=1)
                                                        ->  Hash  (cost=25.40..25.40 rows=1440 width=8) (actual time=0.367..0.368 rows=1440 loops=1)
                                                              Buckets: 2048  Batches: 1  Memory Usage: 73kB
                                                              ->  Seq Scan on schedule s  (cost=0.00..25.40 rows=1440 width=8) (actual time=0.006..0.141 rows=1440 loops=1)
                                                  ->  Hash  (cost=1.60..1.60 rows=60 width=8) (actual time=0.021..0.022 rows=60 loops=1)
                                                        Buckets: 1024  Batches: 1  Memory Usage: 11kB
                                                        ->  Seq Scan on busroute br  (cost=0.00..1.60 rows=60 width=8) (actual time=0.006..0.011 rows=60 loops=1)
                                            ->  Hash  (cost=1.10..1.10 rows=10 width=20) (actual time=0.015..0.015 rows=10 loops=1)
                                                  Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                                  ->  Seq Scan on busstation bs  (cost=0.00..1.10 rows=10 width=20) (actual time=0.008..0.010 rows=10 loops=1)
                                      ->  Hash  (cost=5.05..5.05 rows=5 width=12) (actual time=0.086..0.087 rows=5 loops=1)
                                            Buckets: 1024  Batches: 1  Memory Usage: 9kB
                                            ->  HashAggregate  (cost=5.00..5.05 rows=5 width=12) (actual time=0.080..0.082 rows=5 loops=1)
                                                  Group Key: s_1.fkbus
                                                  Batches: 1  Memory Usage: 24kB
                                                  ->  Seq Scan on seat s_1  (cost=0.00..4.00 rows=200 width=8) (actual time=0.017..0.029 rows=200 loops=1)
                          ->  Finalize GroupAggregate  (cost=256990.73..292844.27 rows=141518 width=12) (actual time=1595.188..1805.993 rows=144000 loops=1)
                                Group Key: t.fkride
                                ->  Gather Merge  (cost=256990.73..290013.91 rows=283036 width=12) (actual time=1595.132..1728.808 rows=432000 loops=1)
                                      Workers Planned: 2
                                      Workers Launched: 2
                                      ->  Sort  (cost=255990.71..256344.51 rows=141518 width=12) (actual time=1556.081..1582.406 rows=144000 loops=3)
                                            Sort Key: t.fkride
                                            Sort Method: external merge  Disk: 3672kB
                                            Worker 0:  Sort Method: external merge  Disk: 3672kB
                                            Worker 1:  Sort Method: external merge  Disk: 3672kB
                                            ->  Partial HashAggregate  (cost=218946.48..241461.40 rows=141518 width=12) (actual time=1169.960..1481.246 rows=144000 loops=3)
                                                  Group Key: t.fkride
                                                  Planned Partitions: 4  Batches: 5  Memory Usage: 8241kB  Disk Usage: 27424kB
                                                  Worker 0:  Batches: 5  Memory Usage: 8241kB  Disk Usage: 24368kB
                                                  Worker 1:  Batches: 5  Memory Usage: 8241kB  Disk Usage: 27544kB
                                                  ->  Parallel Seq Scan on tickets t  (cost=0.00..80532.14 rows=2160614 width=12) (actual time=0.080..248.008 rows=1728502 loops=3)
Planning Time: 2.242 ms
JIT:
  Functions: 81
  Options: Inlining false, Optimization false, Expressions true, Deforming true
  Timing: Generation 7.857 ms (Deform 3.043 ms), Inlining 0.000 ms, Optimization 3.795 ms, Emission 59.282 ms, Total 70.934 ms
Execution Time: 2252.010 ms
```
время выполнения - 2252.010 ms

Индексы на внешние ключи ускориться не помогли. 
В таблицах немного данных.
```
select count(*) from book.ride --144000
select count(*) from book.schedule --1440
select count(*) from book.busroute -- 60
select count(*) from book.busstation -- 10
select count(*) from book.seat --200
```
Используя хэш-соединение, PostgreSQL создает хэш-таблицу в памяти из меньшей таблицы и проверяет эту хэш-таблицу для каждой строки в большей таблице, это будет быстрее, чем поиск по индексу.
