1. postgres установлен на компьютере
2. залила перевозки предоставленной командой
```
wget https://storage.googleapis.com/thaibus/thai_small.tar.gz && tar -xf thai_small.tar.gz && psql < thai.sql
```
3. количество поездок - 5185505
```
thai=# select count(*) from book.tickets;
  count  
---------
 5185505
(1 row)
```
