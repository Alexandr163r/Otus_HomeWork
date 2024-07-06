##  Блокировки 

Меняем настройку log_min_duration_statement
```bash
postgres=# ALTER SYSTEM SET log_min_duration_statement = '200ms';
ALTER SYSTEM
```

Перезапускаем кластер
```bash
PS C:\Windows\system32> net stop postgresql-x64-16
Служба "postgresql-x64-16 - PostgreSQL Server 16" останавливается.
Служба "postgresql-x64-16 - PostgreSQL Server 16" успешно остановлена.

PS C:\Windows\system32> net start postgresql-x64-16
Служба "postgresql-x64-16 - PostgreSQL Server 16" запускается.
Служба "postgresql-x64-16 - PostgreSQL Server 16" успешно запущена.
```
Смотрим применилось ли значение, если все норм идем дальше. 
```sql
show log_min_duration_statement;
log_min_duration_statement|
--------------------------+
200ms                     |
```
Создаем таблицу и заполняем данными и проверяем 
```bash
CREATE TABLE test_block (
    id serial PRIMARY KEY,
    value integer
);
select * from test_block
id|value|
--+-----+
 1|    1|
 2|    2|
 3|    3|
 4|    4|
 5|    5|
```

Насинаем три транзакции в разных сеансах 
```bash
BEGIN;
UPDATE test_block SET value = value + 10;
```
и не комитим, во второй и третьей серии окна висят, тоесть операция находится в блокировке.

В отдельный от трех сессиях  делаем запрос
```bash 
SELECT * FROM pg_locks WHERE granted = false;

locktype     |database|relation|page|tuple|virtualxid|transactionid|classid|objid|objsubid|virtualtransaction|pid  |mode         |granted|fastpath|waitstart                    |
-------------+--------+--------+----+-----+----------+-------------+-------+-----+--------+------------------+-----+-------------+-------+--------+-----------------------------+
transactionid|        |        |    |     |          |12707083     |       |     |        |10/80             |20056|ShareLock    |false  |false   |2024-07-06 22:45:58.688 +0400|
tuple        |   16401|   41237|   0|   21|          |             |       |     |        |13/34             |19096|ExclusiveLock|false  |false   |2024-07-06 22:46:10.839 +0400|
``` 
granted = false в данном случае что бы получить транзакции которые небыли разрешены, мы видим две блакировки, 
transactionid для первой это разрешение на чтение но запрет на изменения (Собственно что мы и пытаемя сделать), условно это в торая сессия пыталась сделать апдейт но ей не дала первая.
Вторая же блокировака типа tuple на туже строку, была созадана второй сессии для дальнейших если добавить еще одну сессию с тем же запросом на Аппдейт то будет видно 

```bash 
SELECT * FеROM pg_locks WHERE granted = false;
locktype     |database|relation|page|tuple|virtualxid|transactionid|classid|objid|objsubid|virtualtransaction|pid  |mode         |granted|fastpath|waitstart                    |
-------------+--------+--------+----+-----+----------+-------------+-------+-----+--------+------------------+-----+-------------+-------+--------+-----------------------------+
tuple        |   16401|   41237|   0|   21|          |             |       |     |        |15/155            | 4220|ExclusiveLock|false  |false   |2024-07-06 23:09:08.697 +0400|
transactionid|        |        |    |     |          |12707083     |       |     |        |10/80             |20056|ShareLock    |false  |false   |2024-07-06 22:45:58.688 +0400|
tuple        |   16401|   41237|   0|   21|          |             |       |     |        |13/34             |19096|ExclusiveLock|false  |false   |2024-07-06 22:46:10.839 +0400|
```
мы добавили еще одну сессию и видим что блакировка типа tuple, последовательность ее выполней будет уже по принципу кто первый встал того и тапки, тоесть выполнение будет идти не 3я 4я 5я и тд сессия, а чей поток раньше запросит состояние блокировки:
проснулся, спросил есть ли блакировка, если есть дальше спать, если нет, занимает таблицу.

Делаем deadlock 
1 сессия
```bash 
BEGIN;
UPDATE test_block SET value = value + 10
where id = 2;
UPDATE test_block SET value = value - 10
where id = 1;
commit;
```
2 сессия
```bash 
BEGIN;
UPDATE test_block SET value = value + 10
where id = 1;
UPDATE test_block SET value = value - 10
where id = 2;
commit;
```
3я сессия
```bash 
UPDATE test_block SET value = value + 10
where id = 1;
UPDATE test_block SET value = value - 10
where id = 2;
```
в итоге во второй сесиии получаем 
```sql
SQL Error [40P01]: ОШИБКА: обнаружена взаимоблокировка
  Подробности: Процесс 11556 ожидает в режиме ShareLock блокировку "транзакция 12707096"; заблокирован процессом 15764.
Процесс 15764 ожидает в режиме ExclusiveLock блокировку "кортеж (0,7) отношения 41237 базы данных 16401"; заблокирован процессом 21052.
Процесс 21052 ожидает в режиме ShareLock блокировку "транзакция 12707097"; заблокирован процессом 11556.
  Подсказка: Подробности запроса смотрите в протоколе сервера.
  Где: при изменении кортежа (0,55) в отношении "test_block"
```
Можно ли разобраться в блокировках в логах? без доп настрои нет, для более подробного описания блокировок в postgresql.conf или auto.conf нуэно добавить :
log_lock_waits = on
log_statement = 'all'
во всяком случае там мне подсказывает гугл, лично не проверял.

Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
Да, самый простой способ это поставить уровень изоляции Serializable и тут без вариантов)  
