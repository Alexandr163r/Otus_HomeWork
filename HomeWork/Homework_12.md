## Работа с join'ами, статистикой

За основу взял вмем известную бд Bookings (я ленивый, а она готовая)
[схема базы](demo_bd.png)

1) Реализовать прямое соединение двух или более таблиц.
Получаем какой рейс откуда куда летит.

```sql
SELECT f.flight_id, f.flight_no, a1.airport_name AS departure_airport, a2.airport_name AS arrival_airport
FROM flights f
JOIN airports_data a1 ON f.departure_airport = a1.airport_code
JOIN airports_data a2 ON f.arrival_airport = a2.airport_code;
```
Explain

```sql
QUERY PLAN                                                                         |
-----------------------------------------------------------------------------------+
Hash Join  (cost=10.68..5957.37 rows=214867 width=133)                             |
  Hash Cond: (f.arrival_airport = a2.airport_code)                                 |
  ->  Hash Join  (cost=5.34..5365.02 rows=214867 width=76)                         |
        Hash Cond: (f.departure_airport = a1.airport_code)                         |
        ->  Seq Scan on flights f  (cost=0.00..4772.67 rows=214867 width=19)       |
        ->  Hash  (cost=4.04..4.04 rows=104 width=65)                              |
              ->  Seq Scan on airports_data a1  (cost=0.00..4.04 rows=104 width=65)|
  ->  Hash  (cost=4.04..4.04 rows=104 width=65)                                    |
        ->  Seq Scan on airports_data a2  (cost=0.00..4.04 rows=104 width=65)      |
```
Описание:
Hash Join используется для соединения данных из таблицы flights с двумя экземплярами таблицы airports_data.
Пострес сначала сканирует все строки из таблиц (Seq Scan), затем строит хеш-таблицы для более эффективного соединения на основании условий соединения.
Все этапы построены на последовательных сканированиях и хеш-таблицах.

2) Реализовать левостороннее (или правостороннее) соединение двух или более таблиц
Получаем список всех билетов и соответствующие рейсы (ошибочных с null тут к сожалению нет, сразу видно таблица демо)

```sql 
explain SELECT t.ticket_no, t.passenger_name, f.flight_no, f.scheduled_departure
FROM tickets t
LEFT JOIN ticket_flights tf ON t.ticket_no = tf.ticket_no
LEFT JOIN flights f ON tf.flight_id = f.flight_id;
```
Explain
```sql 
QUERY PLAN                                                                                                                               |
-----------------------------------------------------------------------------------------------------------------------------------------+
Hash Right Join  (cost=144698.15..577113.09 rows=8391310 width=45) (actual time=771.169..6831.969 rows=8391852 loops=1)                  |
  Hash Cond: (tf.ticket_no = t.ticket_no)                                                                                                |
  ->  Hash Left Join  (cost=8717.51..284213.23 rows=8391310 width=29) (actual time=37.063..2610.802 rows=8391852 loops=1)                |
        Hash Cond: (tf.flight_id = f.flight_id)                                                                                          |
        ->  Seq Scan on ticket_flights tf  (cost=0.00..153873.10 rows=8391310 width=18) (actual time=0.011..696.998 rows=8391852 loops=1)|
        ->  Hash  (cost=4772.67..4772.67 rows=214867 width=19) (actual time=36.913..36.914 rows=214867 loops=1)                          |
              Buckets: 131072  Batches: 2  Memory Usage: 6488kB                                                                          |
              ->  Seq Scan on flights f  (cost=0.00..4772.67 rows=214867 width=19) (actual time=0.007..14.776 rows=214867 loops=1)       |
  ->  Hash  (cost=78939.84..78939.84 rows=2949984 width=30) (actual time=730.547..730.548 rows=2949857 loops=1)                          |
        Buckets: 131072  Batches: 32  Memory Usage: 6617kB                                                                               |
        ->  Seq Scan on tickets t  (cost=0.00..78939.84 rows=2949984 width=30) (actual time=0.032..334.595 rows=2949857 loops=1)         |
Planning Time: 0.303 ms                                                                                                                  |
Execution Time: 7022.228 ms                                                                                                              |
```
Описание: 
Если я провильно понял то сначала строятся хеш таблицы для обоих таблиц Hash Right Join  и  Hash Left Join, а затем уже соотносятся друг с другом, как и в придыдущем примере происходит последовательное сканирование.
При правом джойне Hash Right Join  и  Hash Left Join просто меняются местами в плане запроса.

3) Реализовать кросс соединение двух или более таблиц
Получаем все возможные комбинации пассажиров и рейсов (Декартово произведение, ниразу в работе не пользовался, видимо что то для аналитиков) 

```sql
SELECT t.passenger_name, f.flight_no
FROM tickets t
CROSS JOIN flights f;
```
Explain
```sql
QUERY PLAN                                                                 |
---------------------------------------------------------------------------+
Nested Loop  (cost=0.00..10401248461.28 rows=633854212128 width=23)        |
  ->  Seq Scan on tickets t  (cost=0.00..78939.84 rows=2949984 width=16)   |
  ->  Materialize  (cost=0.00..6687.01 rows=214867 width=7)                |
        ->  Seq Scan on flights f  (cost=0.00..4772.67 rows=214867 width=7)|
```
Описание :
Nested Loop по сути просто алгоритм включающий в себя цикл (честно стырено из инета)

    Узел Nested Loop обращается к первому (внешнему) набору за первой строкой. Здесь это - узел Index Scan по билетам.
    Затем Nested Loop обращается ко второму (внутреннему) набору и просит выдать все строки, соответствующие строке первого набора. Здесь это - узел Index Scan по перелетам.
    Процесс повторяется до тех пор, пока внешний набор не исчерпает все строки.
Materialize кеширование (хотя правильнее наверное создает представление) данных второй таблицы что бы каждый раз не сканировать по новой во время работы данного цикла.

4) Реализовать полное соединение двух или более таблиц.
Получаем все билеты и все рейсы, с указанием соответствий, должны быть null если нет соответствий но база демо не косячкая. 

```sql 
SELECT t.ticket_no, t.passenger_name, f.flight_no, f.scheduled_departure
FROM tickets t
FULL OUTER JOIN ticket_flights tf ON t.ticket_no = tf.ticket_no
FULL OUTER JOIN flights f ON tf.flight_id = f.flight_id;
```
Explain
```sql
QUERY PLAN                                                                       
---------------------------------------------------------------------------------
Hash Full Join  (cost=144698.15..593503.09 rows=8391310 width=45)                
  Hash Cond: (tf.flight_id = f.flight_id)                                        
  ->  Hash Full Join  (cost=135980.64..430382.96 rows=8391310 width=34)          
        Hash Cond: (tf.ticket_no = t.ticket_no)                                  
        ->  Seq Scan on ticket_flights tf  (cost=0.00..153873.10 rows=8391310 wid
        ->  Hash  (cost=78939.84..78939.84 rows=2949984 width=30)                
              ->  Seq Scan on tickets t  (cost=0.00..78939.84 rows=2949984 width=
  ->  Hash  (cost=4772.67..4772.67 rows=214867 width=19)                         
        ->  Seq Scan on flights f  (cost=0.00..4772.67 rows=214867 width=19)     
```
Описание : 
Мне честно говоря тут особо нечего добавить из чего то нового тут только тип соединения Hash Full Join, который просто вернет все строки из обеих таблиц.

5) Реализовать запрос, в котором будут использованы разные типы соединений
Получаем всех пассажиры с их билетами, рейсами и временем отправления и  рейсы, которые могут не иметь соответствующего билета (не уверен что точно и вообще нужно но почему бы и нет).
```sql 
explain SELECT t.passenger_name, t.ticket_no, f.flight_no, f.scheduled_departure
FROM tickets t
LEFT JOIN ticket_flights tf ON t.ticket_no = tf.ticket_no
LEFT JOIN flights f ON tf.flight_id = f.flight_id
UNION
SELECT t.passenger_name, t.ticket_no, f.flight_no, f.scheduled_departure
FROM flights f
RIGHT JOIN ticket_flights tf ON f.flight_id = tf.flight_id
RIGHT JOIN tickets t ON tf.ticket_no = t.ticket_no;
```
Explain
```sql 
QUERY PLAN                                                                                                 |
-----------------------------------------------------------------------------------------------------------+
Unique  (cost=6521761.17..6731543.92 rows=16782620 width=124)                                              |
  ->  Sort  (cost=6521761.17..6563717.72 rows=16782620 width=124)                                          |
        Sort Key: t.passenger_name, t.ticket_no, f.flight_no, f.scheduled_departure                        |
        ->  Append  (cost=144698.15..1238139.28 rows=16782620 width=124)                                   |
              ->  Hash Right Join  (cost=144698.15..577113.09 rows=8391310 width=45)                       |
                    Hash Cond: (tf.ticket_no = t.ticket_no)                                                |
                    ->  Hash Left Join  (cost=8717.51..284213.23 rows=8391310 width=29)                    |
                          Hash Cond: (tf.flight_id = f.flight_id)                                          |
                          ->  Seq Scan on ticket_flights tf  (cost=0.00..153873.10 rows=8391310 width=18)  |
                          ->  Hash  (cost=4772.67..4772.67 rows=214867 width=19)                           |
                                ->  Seq Scan on flights f  (cost=0.00..4772.67 rows=214867 width=19)       |
                    ->  Hash  (cost=78939.84..78939.84 rows=2949984 width=30)                              |
                          ->  Seq Scan on tickets t  (cost=0.00..78939.84 rows=2949984 width=30)           |
              ->  Hash Right Join  (cost=144698.15..577113.09 rows=8391310 width=45)                       |
                    Hash Cond: (tf_1.ticket_no = t_1.ticket_no)                                            |
                    ->  Hash Left Join  (cost=8717.51..284213.23 rows=8391310 width=29)                    |
                          Hash Cond: (tf_1.flight_id = f_1.flight_id)                                      |
                          ->  Seq Scan on ticket_flights tf_1  (cost=0.00..153873.10 rows=8391310 width=18)|
                          ->  Hash  (cost=4772.67..4772.67 rows=214867 width=19)                           |
                                ->  Seq Scan on flights f_1  (cost=0.00..4772.67 rows=214867 width=19)     |
                    ->  Hash  (cost=78939.84..78939.84 rows=2949984 width=30)                              |
                          ->  Seq Scan on tickets t_1  (cost=0.00..78939.84 rows=2949984 width=30)         |
```
Описание: 
Unique применяется для удаления дубликатов в результирующем наборе данных.
Sort Сортирует данные по ключам: t.passenger_name, t.ticket_no, f.flight_no, f.scheduled_departur
Append Объединяет результаты из двух разных операций Hash Right Join
ну а остальные операции уже встречались выше и были описаны. 

По метрикам с помощью pg_stat_statements, просто примеры того на что стоит взглянуть. 

1) выдаст самые медленные запросы.
```sql 
SELECT query, total_time, calls, mean_time, max_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;
```
2) выдаст самые частые запросы.
```sql 
SELECT query, total_time, calls, mean_time, max_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;
```
3) Запросы наименее эффективно используют буфер
```sql 
SELECT query, shared_blks_hit, shared_blks_read,total_time
FROM pg_stat_statements
ORDER BY shared_blks_hit ASC
LIMIT 10;
```
4) Запросы наименее эффективно используют буфер
```sql 
SELECT query, temp_blks_written, calls, total_time
FROM pg_stat_statements
ORDER BY temp_blks_written DESC
LIMIT 10;
```
