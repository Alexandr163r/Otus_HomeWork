## Cекционирование таблицы

Начнем, с таблицей и схемой думаю все понятно, база данных демо, схема bookings, таблица ticket_flights, выбор на мой взгляд немного странный но как сказал преподаватель так и сделаем. 

1) Создаем партиции, делить буду по бизнесовой логике, а точнее по билетам купленым в разные классы это эконом, комфорт и бизнес, столбец fare_conditions.

```sql 
CREATE TABLE ticket_flights_partitioned (
    ticket_no bpchar(13) NOT NULL,
    flight_id int4 NOT NULL,
    fare_conditions varchar(10) NOT NULL,
    amount numeric(10, 2) NOT NULL,
    CONSTRAINT ticket_flights_amount_check CHECK ((amount >= (0)::numeric)),
    CONSTRAINT ticket_flights_fare_conditions_check CHECK (((fare_conditions)::text = ANY (ARRAY[('Economy'::character varying)::text, ('Comfort'::character varying)::text, ('Business'::character varying)::text]))),
    CONSTRAINT ticket_flights_flight_id_fkey FOREIGN KEY (flight_id) REFERENCES flights(flight_id),
    CONSTRAINT ticket_flights_ticket_no_fkey FOREIGN KEY (ticket_no) REFERENCES tickets(ticket_no)
) PARTITION BY LIST (fare_conditions);



CREATE TABLE ticket_flights_economy
    PARTITION OF ticket_flights_partitioned
    FOR VALUES IN ('Economy');

CREATE TABLE ticket_flights_comfort
    PARTITION OF ticket_flights_partitioned
    FOR VALUES IN ('Comfort');

CREATE TABLE ticket_flights_business
    PARTITION OF ticket_flights_partitioned
    FOR VALUES IN ('Business');
```

2) копируем данные в нашу партицию из первоисточника
```sql
INSERT INTO ticket_flights_partitioned
SELECT * FROM ticket_flights;
```

3) дропаем исходную таблицу 
```sql 
DROP TABLE ticket_flights;
```
тут возник нюанс, к сожалению ошибку я не записал, зависимость первичного ключа, и придложение все дропнуть каскадно, такого нам разумеется не нужно, так что дропаем ключ 

```
ALTER TABLE boarding_passes
DROP CONSTRAINT boarding_passes_ticket_no_fkey;
```
а затем и таблицу 
```sql
DROP TABLE ticket_flights;
```
меняем имя нашей партиции на имя бывшего источника 
```sql 
ALTER TABLE ticket_flights_partitioned
RENAME TO ticket_flights;
```
уже все работает, но нужно вернуть связку ключей.
```sql 
ALTER TABLE bookings.boarding_passes
ADD CONSTRAINT boarding_passes_ticket_no_fkey
FOREIGN KEY (ticket_no) REFERENCES bookings.ticket_flights(ticket_no);
```
теперь вроде все на месте проверяем 

```sql
SELECT fare_conditions, count(*)
FROM ticket_flights
GROUP BY fare_conditions  

fare_conditions|count  |
---------------+-------+
Business       | 859656|
Comfort        | 139965|
Economy        |7392231|
```
explain разумеется тоже 
```sql
explain SELECT fare_conditions, count(*)
FROM ticket_flights
GROUP BY fare_conditions;   

QUERY PLAN                                                                                                                            |
--------------------------------------------------------------------------------------------------------------------------------------+
Finalize GroupAggregate  (cost=141114.89..141165.56 rows=200 width=16)                                                                |
  Group Key: ticket_flights.fare_conditions                                                                                           |
  ->  Gather Merge  (cost=141114.89..141161.56 rows=400 width=16)                                                                     |
        Workers Planned: 2                                                                                                            |
        ->  Sort  (cost=140114.87..140115.37 rows=200 width=16)                                                                       |
              Sort Key: ticket_flights.fare_conditions                                                                                |
              ->  Partial HashAggregate  (cost=140105.23..140107.23 rows=200 width=16)                                                |
                    Group Key: ticket_flights.fare_conditions                                                                         |
                    ->  Parallel Append  (cost=0.00..122622.21 rows=3496604 width=8)                                                  |
                          ->  Parallel Seq Scan on ticket_flights_economy ticket_flights_3  (cost=0.00..92402.96 rows=3080096 width=8)|
                          ->  Parallel Seq Scan on ticket_flights_business ticket_flights_1  (cost=0.00..10745.90 rows=358190 width=9)|
                          ->  Parallel Seq Scan on ticket_flights_comfort ticket_flights_2  (cost=0.00..1990.32 rows=82332 width=8)   |
```

Ну вроде все, с ключем что то мог пропустить, не всегда записываю пока решаю проблему, за это прошу извинить.
