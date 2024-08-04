## Работа с индексами

Кластер есть, база есть, так что просто создаем таблицу создаем таблицу, оригинальничить не буду так что просто таблица продукты.
Cоздаем, заполняемb проверяем на всякий случай.

```sql 
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT,
    description TEXT,
    price NUMERIC,
    created TIMESTAMP
);

DO $$
BEGIN
    INSERT INTO products (name, description, price, created)
    SELECT
        'Продукт № ' || i,
        'Описание Продукта # ' || i,
        10 * i,
        NOW() - (i || ' days')::interval
    FROM generate_series(1, 1000) AS s(i);
END;
$$;

select * from products 

id |name         |description            |price|created                |
---+-------------+-----------------------+-----+-----------------------+
  1|Продукт № 1  |Описание Продукта # 1  |   10|2024-08-03 14:47:24.512|
  2|Продукт № 2  |Описание Продукта # 2  |   20|2024-08-02 14:47:24.512|
  3|Продукт № 3  |Описание Продукта # 3  |   30|2024-08-01 14:47:24.512|
  4|Продукт № 4  |Описание Продукта # 4  |   40|2024-07-31 14:47:24.512|
  5|Продукт № 5  |Описание Продукта # 5  |   50|2024-07-30 14:47:24.512|
  6|Продукт № 6  |Описание Продукта # 6  |   60|2024-07-29 14:47:24.512|
  7|Продукт № 7  |Описание Продукта # 7  |   70|2024-07-28 14:47:24.512|
  8|Продукт № 8  |Описание Продукта # 8  |   80|2024-07-27 14:47:24.512|
  9|Продукт № 9  |Описание Продукта # 9  |   90|2024-07-26 14:47:24.512|
 10|Продукт № 10 |Описание Продукта # 10 |  100|2024-07-25 14:47:24.512|
```

Так как при создание первичного ключа автоматически создается индекс то в данной таблице достаточно просто добавить в условие WHERE id =.

Проверяем
```sql 
explain select * from products p
where p.id = 1

QUERY PLAN                                                                     |
-------------------------------------------------------------------------------+
Index Scan using products_pkey on products p  (cost=0.28..8.29 rows=1 width=78)|
  Index Cond: (id = 1)                                                         |
```

Теперь создаем индекс для полнотекстового поиска.
```sql 
CREATE INDEX idx_description
ON products
USING gin(to_tsvector('russian', description));
```
Проверяем план запроса.
```sql
explain SELECT * FROM products WHERE to_tsvector('russian', description) @@ to_tsquery('russian', '12');

QUERY PLAN                                                                               |
-----------------------------------------------------------------------------------------+
Bitmap Heap Scan on products  (cost=8.58..20.93 rows=5 width=78)                         |
  Recheck Cond: (to_tsvector('russian'::regconfig, description) @@ '''12'''::tsquery)    |
  ->  Bitmap Index Scan on idx_description  (cost=0.00..8.58 rows=5 width=0)             |
        Index Cond: (to_tsvector('russian'::regconfig, description) @@ '''12'''::tsquery)|
```
Как видим работает сразу уточню два момента:
1) 12 в поиске просто потому что таблица сгенерирована и текс там одинаковый а хотелось бы получить не все строки а какую то определенную, а ручками править лень.
2) 'russian': Указывает использование русского конфигурационного словаря для токенизации и нормализации текста. (честно стырил из гугла, решил проверить работает ли).

Создаем инденксы на часть таблицы и функцию 
```sql
CREATE INDEX idx_price
ON products (price)
WHERE price > 100;

CREATE INDEX idx_lower_name
ON products (LOWER(name));
```
Проверяем 
```sql
explain SELECT *
FROM products
WHERE price > 100
AND LOWER(name) = 'продукт № 11';

QUERY PLAN                                                                 |
---------------------------------------------------------------------------+
Bitmap Heap Scan on products  (cost=4.31..15.44 rows=5 width=78)           |
  Recheck Cond: (lower(name) = 'продукт № 11'::text)                       |
  Filter: (price > '100'::numeric)                                         |
  ->  Bitmap Index Scan on idx_lower_name  (cost=0.00..4.31 rows=5 width=0)|
        Index Cond: (lower(name) = 'продукт № 11'::text)                   |
```
Работает с like правда не захотел Seq Scan получил, печалька.

Создаем индекс из нескольких полей, он же составной.
```sql
CREATE INDEX idx_price_created
ON products(price, created DESC);
```
Применять индекс почему пострес не хотел, пришлось заставить
```sql
SET enable_seqscan = off;
```
Проверяем 
```sql
EXPLAIN SELECT * FROM products 
WHERE price > 100 
ORDER BY created DESC;
QUERY PLAN                                                                        |
----------------------------------------------------------------------------------+
Sort  (cost=101.38..103.86 rows=990 width=78)                                     |
  Sort Key: created DESC                                                          |
  ->  Index Scan using idx_price on products  (cost=0.28..52.12 rows=990 width=78)|
```
Ну вроде все работает.

Все индексы применяются и ускоряют работу. это тестовая работа и тут можно, на вид все хорошо и быстро но есть одно но
```sql
SELECT 
    pg_size_pretty(pg_total_relation_size('products')) AS total_size,
    pg_size_pretty(pg_relation_size('products')) AS table_size,
    pg_size_pretty(pg_indexes_size('products')) AS indexes_size;
    
total_size|table_size|indexes_size|
----------+----------+------------+
400 kB    |112 kB    |256 kB      |
```
Вывод: нет ничего бесплатного) 
