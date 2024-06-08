## Уровни транзакци в postgresql 

Создание и заполнение таблицы в PostgreSQL

```sql
BEGIN;
CREATE TABLE persons(
    id SERIAL,
    first_name TEXT,
    second_name TEXT
); 
INSERT INTO persons(first_name, second_name) VALUES('ivan', 'ivanov'); 
INSERT INTO persons(first_name, second_name) VALUES('petr', 'petrov'); 
COMMIT;
```

--Таблица persons:

id|first_name|second_name|
--+----------+-----------+
 1|ivan      |ivanov     |
 2|petr      |petrov     |
```

# Проверяем уровень транзакций.

```sql
SHOW TRANSACTION ISOLATION LEVEL;
```

transaction_isolation|
---------------------+
read committed       |


Проверяем уровень транзакций.

```sql
BEGIN;
insert into persons(first_name, second_name) values('sergey', 'sergeev');
```

Во второй сессии выполняем

```sql
select * from persons p 

id|first_name|second_name|
--+----------+-----------+
 1|ivan      |ivanov     |
 2|petr      |petrov     |
```

Не видим новую запись потому что уровень транзакции стоит read committed и транзакция в первой сессии не завершена. 

В первой сессии выполняем.

```sql
COMMIT;
```
Во второй сессии выполняем. 

```sql
COMMIT;
```

id|first_name|second_name|
--+----------+-----------+
 1|ivan      |ivanov     |
 2|petr      |petrov     |
 3|sergey    |sergeev    |
```

Видим добавление записи по причине завершения транзакции в первой сессии.

В первой сессии выполняем.

```sql
BEGIN;
set transaction isolation level repeatable read;
```

Во второй сессии выполняем. 
```sql
BEGIN;
set transaction isolation level repeatable read;
```
В первой сессии выполняем.

```sql
insert into persons(first_name, second_name) values('sveta', 'svetova');
```
Во второй сессии выполняем. 

```sql
select * from persons p ;

id|first_name|second_name|
--+----------+-----------+
 1|ivan      |ivanov     |
 2|petr      |petrov     |
 3|sergey    |sergeev    |
 ```
Не видим новую запись так как не выполнен коммит в первой сессии.

В первой сессии выполняем.

```sql
commit;
```

Во второй сессии выполняем. 
```sql
select * from persons p ;

id|first_name|second_name|
--+----------+-----------+
 1|ivan      |ivanov     |
 2|petr      |petrov     |
 3|sergey    |sergeev    |
```

Не видим новую запись так как установлен уровень транзакции repeatable read, а транзакции второй сессии не завершена.

Во второй сессии выполняем 
```sql
commit;
select * from persons p;

id|first_name|second_name|
--+----------+-----------+
 1|ivan      |ivanov     |
 2|petr      |petrov     |
 3|sergey    |sergeev    |
 4|sveta     |svetova    |
```
Видим новую запись поскольку обе транзакции завершены что соответствует требованиям уровня транзакции repeatable read.