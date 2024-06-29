##  Настройка autovacuum с учетом особеностей производительности

Включаем pgbench
```powershal

C:\Users\Alex>"C:\Program Files\PostgreSQL\16\bin\pgbench" -i -U postgres -p 5433 testdb
Password:
dropping old tables...
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.01 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.17 s (drop tables 0.01 s, create tables 0.01 s, client-side generate 0.08 s, vacuum 0.03 s, primary keys 0.04 s).

C:\Users\Alex>
```
Тестируем 
```bash
C:\Users\Alex>"C:\Program Files\PostgreSQL\16\bin\pgbench" -c 8 -P 6 -T 60 -U postgres -p 5433 testdb
Password:
pgbench (16.3)
starting vacuum...end.
progress: 6.0 s, 5740.3 tps, lat 1.337 ms stddev 0.675, 0 failed
progress: 12.0 s, 5977.5 tps, lat 1.337 ms stddev 0.788, 0 failed
progress: 18.0 s, 5970.4 tps, lat 1.339 ms stddev 0.781, 0 failed
progress: 24.0 s, 5984.7 tps, lat 1.336 ms stddev 0.724, 0 failed
progress: 30.0 s, 5897.2 tps, lat 1.356 ms stddev 0.763, 0 failed
progress: 36.0 s, 5890.0 tps, lat 1.357 ms stddev 0.749, 0 failed
progress: 107.2 s, 0.0 tps, lat 0.000 ms stddev 0.000, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 212768
number of failed transactions: 0 (0.000%)
latency average = 4.021 ms
latency stddev = 436.661 ms
initial connection time = 240.700 ms
tps = 1988.930944 (without initial connection time)
```
меняем настройки в postgres.conf и тестируем по новой, настройку effective_io_concurrency выставил на 0 ибо из под винды с двойкой не поднимается кластер.

```bash
C:\Users\Alex>"C:\Program Files\PostgreSQL\16\bin\pgbench" -c 8 -P 6 -T 60 -U postgres -p 5433 testdb
Password:
pgbench (16.3)
starting vacuum...end.
progress: 6.0 s, 5743.8 tps, lat 1.335 ms stddev 0.666, 0 failed
progress: 12.0 s, 6042.0 tps, lat 1.323 ms stddev 0.646, 0 failed
progress: 18.0 s, 6040.7 tps, lat 1.323 ms stddev 0.646, 0 failed
progress: 24.0 s, 6020.7 tps, lat 1.328 ms stddev 0.672, 0 failed
progress: 30.0 s, 6056.3 tps, lat 1.320 ms stddev 0.680, 0 failed
progress: 36.0 s, 6012.7 tps, lat 1.330 ms stddev 0.677, 0 failed
progress: 42.0 s, 6046.1 tps, lat 1.322 ms stddev 0.738, 0 failed
progress: 48.0 s, 6004.1 tps, lat 1.332 ms stddev 0.744, 0 failed
progress: 54.0 s, 6000.0 tps, lat 1.332 ms stddev 0.746, 0 failed
progress: 60.0 s, 5996.3 tps, lat 1.333 ms stddev 0.744, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 359784
number of failed transactions: 0 (0.000%)
latency average = 1.373 ms
latency stddev = 9.544 ms
initial connection time = 245.961 ms
tps = 5824.180723 (without initial connection time)
C:\Users\Alex>
```
Из явного это увеличение tps с 1988 до 5824, тоесть почти трехкратное увеличение проделаных транзанций в секунду.

Создаем и заполняем таблицу.

```sql
CREATE TABLE test_vac (
    id SERIAL PRIMARY KEY,
    text TEXT
);

INSERT INTO test_vac (text)
SELECT md5(random()::text)
FROM generate_series(1, 1000000);

SELECT pg_size_pretty(pg_total_relation_size('test_vac'));
pg_size_pretty|
--------------+
87 MB         |
```
Обновляем записи 5 раз и заодно проверям размер 
```sql 
DO $$
BEGIN
    FOR i IN 1..5 LOOP
        UPDATE test_vac
        SET text = text || 'Ъ';
    END LOOP;
END $$;
SELECT pg_size_pretty(pg_total_relation_size('test_vac'));
pg_size_pretty|
--------------+
508 MB        |
``
Проверяем количество мертвых строк и когда был автовакуум 
```sql
SELECT
    relname AS table,
    n_dead_tup AS dead,
    last_autovacuum
FROM
    pg_stat_all_tables
WHERE
    relname = 'test_vac';
table   |dead|last_autovacuum              |
--------+----+-----------------------------+
test_vac|   0|2024-06-29 22:00:33.059 +0400|    
```

Повторям заполнени и проверку размема и автовакуума 
```sql
DO $$
BEGIN
    FOR i IN 1..5 LOOP
        UPDATE test_vac
        SET text = text || 'Ъ';
    END LOOP;
END $$;

SELECT
    relname AS table,
    n_dead_tup AS dead,
    last_autovacuum
FROM
    pg_stat_all_tables
WHERE
    relname = 'test_vac';
    table   |dead   |last_autovacuum              |
--------+-------+-----------------------------+
test_vac|5000000|2024-06-29 22:00:33.059 +0400|

SELECT pg_size_pretty(pg_total_relation_size('test_vac'));
pg_size_pretty|
--------------+
570 MB        |
```
Отрубаем автовакуум для данной таблицы 
```sql
ALTER TABLE test_vac SET (autovacuum_enabled = false);
```

Обновляем записи 10 раз, проверяем 
```sql 
DO $$
BEGIN
    FOR i IN 1..10 LOOP
        UPDATE test_vac
        SET text = text || 'Ъ';
    END LOOP;
END $$;

SELECT pg_size_pretty(pg_total_relation_size('test_vac'));
pg_size_pretty|
--------------+
1182 MB       |

SELECT
    relname AS table,
    n_dead_tup AS dead,
    last_autovacuum
FROM
    pg_stat_all_tables
WHERE
    relname = 'test_vac';
table   |dead    |last_autovacuum              |
--------+--------+-----------------------------+
test_vac|10000000|2024-06-29 22:07:34.848 +0400|    
```

И так выводы:
1) Таблица при апдейте записей распухла увеличилась кратно количеству этих апдейтов,  тоесть размер таблицы без апдейтов плюс 5 размеров апдейтов.
2) После автовакуума размер таблицы не изменился, потому что это не фул ваккуум, он оставляет место за собой, не смотря на то что строки мертвые или помечены удаленными, это можно решить запустив фулл вакуум.
3) Конечный размер таблицы без вакуума только в 10 раз больше а не в 15, потому что новые мертвые строки занимают места старых, было уже 5 апдейтов соответветственно место под них уже есть хоть они и почищены вакууумом. как говорилось выше что бы полностью освободить место нужно запустить фул ваккуум. 
