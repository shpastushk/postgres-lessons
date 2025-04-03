
Протестировать падение производителности при исползовании pgbouncer в разных режимах: statement, transaction, session

Запрос для тестирования

```
postgres@ec7affd41acc:/$ cat ~/workload.sql 
\set r random(1, 5000000) 
SELECT t.id, t.fkRide, t.fio, t.contact, t.fkSeat 
FROM book.tickets t 
inner join book.ride r on r.id = t.fkride 
inner join book.seat s on s.id = t.fkseat 
WHERE t.id = :r;
```

1. pool_mode=session
```
postgres@ec7affd41acc:/$ /usr/lib/postgresql/17/bin/pgbench -c 25 -j 4 -T 10 -f ~/workload.sql -n -U postgres -p 6432 -h pgbouncer -d thai
pgbench (17.4 (Debian 17.4-1.pgdg120+2))
transaction type: /var/lib/postgresql/workload.sql
scaling factor: 1
query mode: simple
number of clients: 25
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 39059
number of failed transactions: 0 (0.000%)
latency average = 6.384 ms
initial connection time = 51.586 ms
tps = 3915.794822 (without initial connection time)
```

2. pool_mode=statement
```
postgres@ec7affd41acc:/$ /usr/lib/postgresql/17/bin/pgbench -c 25 -j 4 -T 10 -f ~/workload.sql -n -U postgres -p 6432 -h pgbouncer -d thai
pgbench (17.4 (Debian 17.4-1.pgdg120+2))
transaction type: /var/lib/postgresql/workload.sql
scaling factor: 1
query mode: simple
number of clients: 25
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 41129
number of failed transactions: 0 (0.000%)
latency average = 6.071 ms
initial connection time = 38.775 ms
tps = 4118.257029 (without initial connection time)
```

3. pool_mode=transaction
```
postgres@ec7affd41acc:/$ /usr/lib/postgresql/17/bin/pgbench -c 25 -j 4 -T 10 -f ~/workload.sql -n -U postgres -p 6432 -h pgbouncer -d thai
pgbench (17.4 (Debian 17.4-1.pgdg120+2))
transaction type: /var/lib/postgresql/workload.sql
scaling factor: 1
query mode: simple
number of clients: 25
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 40599
number of failed transactions: 0 (0.000%)
latency average = 6.145 ms
initial connection time = 58.093 ms
tps = 4068.585617 (without initial connection time)
```

На приведенном выше запросе разные режимы pgbouncer показывают относительно похожую производительность.

session mode – можно использовать для нечастых запросов, которые выполняются в рамках долгих сессий.

transaction mode – для коротких транзакций.

statement – подходит для использования приложениями, если много повторяющихся запросов или они используют запросы с одинаковой структурой. (в нашем эксперименте statement чуть выйграл, но правда не при всех запусках)

