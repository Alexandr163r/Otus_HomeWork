##  Работа с базами данных, пользователями и правами
Создаем тестовую БД и схуму 
```sql
CREATE DATABASE testdb;
CREATE SCHEMA testnm;
```
Создаем и заполняем таблицу, Все делал в dbeaver по этому схему указал по умолчанию testnm.

```sql
CREATE TABLE testnm.t1 (
    c1 INTEGER
);
INSERT INTO t1 (c1) VALUES (1);

select * from t1 
c1|
--+
 1|
```
Создаем новую роль readonly.
```sql
CREATE ROLE readonly;
GRANT CONNECT ON DATABASE testdb TO readonly;
GRANT USAGE ON SCHEMA testnm TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
```
Создаем пользователя выдаем ему роль readonly.
```sql
CREATE USER testread WITH PASSWORD 'test123';
GRANT readonly TO testread;
```
подключаемся к бд и делает селект от тестового пользователя 
```sql
SELECT x.* FROM testnm.t1 x

c1|
--+
 1|
```
я так понял что тут должна быть ошибка, но так как схему по умолчанию я поставил сразу, то таблица создалась не в паблике и выборка прошла успешно.

Заходим снова под постресом удаляем и создаем таблицу по новой 
```sql
DROP TABLE testnm.t1;
CREATE TABLE testnm.t1 (c1 INTEGER);
INSERT INTO t1 (c1) VALUES (1);
```
Заходим снова под тестовым юзером и делаем запрос 
```sql
SELECT x.* FROM testnm.t1 x
```
Ловим ошибку: 
SQL Error [42501]: ОШИБКА: нет доступа к таблице t1, 
потому что GRANT действует только на уже существующие таблицы, для новых нужно выдавать его по новой.

в избежание повторения данной ошибки делаем так
```sql
GRANT CONNECT ON DATABASE testdb TO readonly;
GRANT USAGE ON SCHEMA testnm TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
GRANT readonly TO testread;
ALTER DEFAULT PRIVILEGES IN SCHEMA testnm
GRANT SELECT ON TABLES TO readonly;
ALTER DEFAULT PRIVILEGES
GRANT USAGE ON SCHEMAS TO readonly;
```
Теперь чтение у testread работает и для новых таблиц (t2 была создана под постресом после выдачи прав).
```sql 
select * from t1 
c1|
--+
 1|
 1|
select * from t2 
c1|
--+
```
создаем таблицу от пользователя  testread 
```sql 
create table t3(c1 integer); 
SQL Error [42501]: ОШИБКА: нет доступа к схеме testnm
  Позиция: 14
Позиция ошибки: line: 1 pos: 13
```
Если я правильно понял то опять же из за не жесткого указания схем мне должно было это позволить, но так как в самом начале для удобства я жестко указал схемы и выставил DEFAULT PRIVILEGES  выше, этого не произошло.
