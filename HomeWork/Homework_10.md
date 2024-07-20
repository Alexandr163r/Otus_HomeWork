##  Репликация
Исходные данные Windows 10, wsl ubuntu, установку всего необходимого для работы опустим 

Поднимаем 4 кластера, порты 40+ просто для удобства отделения от старых баз.

```bash
sudo pg_createcluster 16 main1 --port=5441 --start
sudo pg_createcluster 16 main2 --port=5442 --start
sudo pg_createcluster 16 main3 --port=5443 --start
sudo pg_createcluster 16 main4 --port=5444 --start
```
Проверяем 
```bash
alex@DESKTOP-G9FSIV5:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory               Log file
14  main    5432 online postgres /var/lib/postgresql/14/main  /var/log/postgresql/postgresql-14-main.log
16  main    5433 online postgres /var/lib/postgresql/16/main  /var/log/postgresql/postgresql-16-main.log
16  main1   5441 online postgres /var/lib/postgresql/16/main1 /var/log/postgresql/postgresql-16-main1.log
16  main2   5442 online postgres /var/lib/postgresql/16/main2 /var/log/postgresql/postgresql-16-main2.log
16  main3   5443 online postgres /var/lib/postgresql/16/main3 /var/log/postgresql/postgresql-16-main3.log
16  main4   5444 online postgres /var/lib/postgresql/16/main4 /var/log/postgresql/postgresql-16-main4.log
alex@DESKTOP-G9FSIV5:~$
```
Сразу идем в hba.conf и меняем для пользователя пострес доступ, неправильно но каждый раз парроль вводить надоедает.
Делаем для каждого кластера.
```bash
root@DESKTOP-G9FSIV5:/etc/postgresql/16/main3# nano pg_hba.conf
local   all             postgres                                trust
```
и рестартуем все
```bash
root@DESKTOP-G9FSIV5:/etc/postgresql/16/main3# sudo pg_ctlcluster 16 main1 restart
root@DESKTOP-G9FSIV5:/etc/postgresql/16/main3# sudo pg_ctlcluster 16 main2 restart
root@DESKTOP-G9FSIV5:/etc/postgresql/16/main3# sudo pg_ctlcluster 16 main3 restart
root@DESKTOP-G9FSIV5:/etc/postgresql/16/main3# sudo pg_ctlcluster 16 main4 restart
```
идем в main1 и main2 создаем базы, таблицы и публикации, расписывать для каждой не буду, это пример для main2
```sql
alex@DESKTOP-G9FSIV5:/mnt/c/Windows/system32$ psql -p 5442 -U postgres
psql (16.3 (Ubuntu 16.3-1.pgdg22.04+1))
Type "help" for help.

postgres=# CREATE DATABASE db2;
CREATE DATABASE
db2=# CREATE TABLE test1 (id SERIAL PRIMARY KEY,data TEXT);
CREATE TABLE
db2=# CREATE TABLE test2 (id SERIAL PRIMARY KEY,data TEXT);
CREATE TABLE
db2=# ALTER SYSTEM SET wal_level = 'logical';
ALTER SYSTEM
db2=# CREATE PUBLICATION pub_test2 FOR TABLE test2;
```

Настраиваем подписки для main1 на main2 и наоборот 
```sql
CREATE SUBSCRIPTION sub_test2 CONNECTION 'host=localhost port=5442 dbname=db2 user=postgres password=pos' PUBLICATION pub_test2;
CREATE SUBSCRIPTION sub_test1 CONNECTION 'host=localhost port=5441 dbname=db1 user=postgres password=pos' PUBLICATION pub_test1;
```
Проверяем 

```sql 
db2=# SELECT * FROM pg_publication;
  oid  |  pubname  | pubowner | puballtables | pubinsert | pubupdate | pubdelete | pubtruncate | pubviaroot
-------+-----------+----------+--------------+-----------+-----------+-----------+-------------+------------
 16407 | pub_test2 |       10 | f            | t         | t         | t         | t           | f
(1 row)

db2=# SELECT * FROM pg_stat_subscription;
 subid |  subname  |  pid  | leader_pid | relid | received_lsn |      last_msg_send_time       |     last_msg_receipt_time     | latest_end_lsn |        latest_end_time
-------+-----------+-------+------------+-------+--------------+-------------------------------+-------------------------------+----------------+-------------------------------
 16409 | sub_test1 | 13705 |            |       | 0/1968E58    | 2024-07-20 23:57:27.033908+04 | 2024-07-20 23:57:27.033946+04 | 0/1968E58      | 2024-07-20 23:57:27.033908+04
(1 row)

db2=# exit
alex@DESKTOP-G9FSIV5:/mnt/c/Windows/system32$ psql -p 5441 -U postgres -d db1
psql (16.3 (Ubuntu 16.3-1.pgdg22.04+1))
Type "help" for help.

db1=# SELECT * FROM pg_publication;
  oid  |  pubname  | pubowner | puballtables | pubinsert | pubupdate | pubdelete | pubtruncate | pubviaroot
-------+-----------+----------+--------------+-----------+-----------+-----------+-------------+------------
 16407 | pub_test1 |       10 | f            | t         | t         | t         | t           | f
(1 row)

db1=# SELECT * FROM pg_stat_subscription;
 subid |  subname  |  pid  | leader_pid | relid | received_lsn |      last_msg_send_time       |     last_msg_receipt_time     | latest_end_lsn |        latest_end_time
-------+-----------+-------+------------+-------+--------------+-------------------------------+-------------------------------+----------------+-------------------------------
 16417 | sub_test2 | 13689 |            |       | 0/1987EA8    | 2024-07-20 23:58:00.738298+04 | 2024-07-20 23:58:00.738343+04 | 0/1987EA8      | 2024-07-20 23:58:00.738298+04
(1 row)
db1=#
```
проверяем на селект и видим, в обратную сторону все тоже самое ибо настройки одинаковые. 
```sql
alex@DESKTOP-G9FSIV5:/mnt/c/Users/Alex$ psql -p 5442 -U postgres -d db2
psql (16.3 (Ubuntu 16.3-1.pgdg22.04+1))
Type "help" for help.

db2=# SELECT * from test1;
 id |  data
----+--------
  1 | первый
  2 | второй
  3 | Третий
(3 rows)

db2=#
alex@DESKTOP-G9FSIV5:/mnt/c/Windows/system32$ psql -p 5441 -U postgres -d db1
psql (16.3 (Ubuntu 16.3-1.pgdg22.04+1))
Type "help" for help.

db1=# select * from test1
;
 id |  data
----+--------
  1 | первый
  2 | второй
  3 | Третий
(3 rows)
db1=#
```

идем на третий кластер и подписываемся на первый и второй 

```sql
db3=# CREATE SUBSCRIPTION sub_test3 CONNECTION 'host=localhost port=5441 dbname=db1 user=postgres password=pos' PUBLICATION pub_test1;
NOTICE:  created replication slot "sub_test3" on publisher
CREATE SUBSCRIPTION
db3=# CREATE SUBSCRIPTION sub_test4 CONNECTION 'host=localhost port=5442 dbname=db2 user=postgres password=pos' PUBLICATION pub_test2;
NOTICE:  created replication slot "sub_test4" on publisher
CREATE SUBSCRIPTION
db3=# select * from test1;
 id |  data
----+--------
  1 | первый
  2 | второй
  3 | Третий
(3 rows)

db3=# select * from test2;
 id | data
----+------
  1 | 1
  2 | 2
  3 | 3
(3 rows)
db3=#
```
Начал делать горячую репликацию на 4ю машину, но что то как то постоянно ломается, так что без задания со звездачкой сдаю работу.
Из сложнойстей, да все как обычно, невнимательность в первую очередь, и забыл что если сходу поставить доступ trust то пароля для md5 просто не будет) 
