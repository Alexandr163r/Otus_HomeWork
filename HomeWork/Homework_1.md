## ������ ��������� � postgresql 

�������� � ���������� ������� � PostgreSQL

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

--������� persons:

id|first_name|second_name|
--+----------+-----------+
 1|ivan      |ivanov     |
 2|petr      |petrov     |
```

# ��������� ������� ����������.

```sql
SHOW TRANSACTION ISOLATION LEVEL;
```

transaction_isolation|
---------------------+
read committed       |


��������� ������� ����������.

```sql
BEGIN;
insert into persons(first_name, second_name) values('sergey', 'sergeev');
```

�� ������ ������ ���������

```sql
select * from persons p 

id|first_name|second_name|
--+----------+-----------+
 1|ivan      |ivanov     |
 2|petr      |petrov     |
```

�� ����� ����� ������ ������ ��� ������� ���������� ����� read committed � ���������� � ������ ������ �� ���������. 

� ������ ������ ���������.

```sql
COMMIT;
```
�� ������ ������ ���������. 

```sql
COMMIT;
```

id|first_name|second_name|
--+----------+-----------+
 1|ivan      |ivanov     |
 2|petr      |petrov     |
 3|sergey    |sergeev    |
```

����� ���������� ������ �� ������� ���������� ���������� � ������ ������.

� ������ ������ ���������.

```sql
BEGIN;
set transaction isolation level repeatable read;
```

�� ������ ������ ���������. 
```sql
BEGIN;
set transaction isolation level repeatable read;
```
� ������ ������ ���������.

```sql
insert into persons(first_name, second_name) values('sveta', 'svetova');
```
�� ������ ������ ���������. 

```sql
select * from persons p ;

id|first_name|second_name|
--+----------+-----------+
 1|ivan      |ivanov     |
 2|petr      |petrov     |
 3|sergey    |sergeev    |
 ```
�� ����� ����� ������ ��� ��� �� �������� ������ � ������ ������.

� ������ ������ ���������.

```sql
commit;
```

�� ������ ������ ���������. 
```sql
select * from persons p ;

id|first_name|second_name|
--+----------+-----------+
 1|ivan      |ivanov     |
 2|petr      |petrov     |
 3|sergey    |sergeev    |
```

�� ����� ����� ������ ��� ��� ���������� ������� ���������� repeatable read, � ���������� ������ ������ �� ���������.

�� ������ ������ ��������� 
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
����� ����� ������ ��������� ��� ���������� ��������� ��� ������������� ����������� ������ ���������� repeatable read.