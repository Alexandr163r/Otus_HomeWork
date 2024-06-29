##  Физический уровень PostgreSQL Установка и настройка 
Установку я пропущу есть в прошлых домашках, смысла посторяться нет.

Проверяем кластер и заходим под постресом

```bash
root@87cc2b9431cb:/# su - postgres -c 'psql'
psql (16.3 (Debian 16.3-1.pgdg120+1))
Type "help" for help.
postgres=#
```
Создаем таблицу test и заполняем 
```bash
postgres=# create table test(c1 text);
CREATE TABLE

postgres=# insert into test values('1');
INSERT 0 1

postgres\q
root@87cc2b9431cb:/#
```

Создаем новый диск размером 10г.
```bash
alex@DESKTOP-G9FSIV5:/mnt/c/Users/Alex$ fallocate -l 5G new_disk.img
fallocate: fallocate failed: Operation not supported
```
Через fallocate не вышло идем другим путем 

```bash
alex@DESKTOP-G9FSIV5:/mnt/c/Users/Alex$ dd if=/dev/zero of=new_disk.img bs=1G count=10
10+0 records in
10+0 records out
10737418240 bytes (11 GB, 10 GiB) copied, 31.8118 s, 338 MB/s
alex@DESKTOP-G9FSIV5:/mnt/c/Users/Alex$ ls -lh new_disk.img
-rwxrwxrwx 1 alex alex 10G Jun 29 16:27 new_disk.img
alex@DESKTOP-G9FSIV5:/mnt/c/Users/Alex$
```

Монтируем и проверяем новый диск 

```bash
alex@DESKTOP-G9FSIV5:/mnt/c/Users/Alex$ sudo losetup -f
/dev/loop3
alex@DESKTOP-G9FSIV5:/mnt/c/Users/Alex$ sudo mount /dev/loop3 /mnt/new_disk
alex@DESKTOP-G9FSIV5:/mnt/c/Users/Alex$ ls /mnt/new_disk
lost+found
```

Устанавливаем ступ к диску и переносим данные из старой дериктории на новый диск
```bash
root@DESKTOP-G9FSIV5:/mnt/c/Users/Alex# sudo mv /var/lib/postgres /mnt/new_disk
```
Перезапускаем кластер и получаем ошибку не поднялось, видимо ищит файлы в прошлой дериктории 
```bash
2024-06-29 17:48:16 PostgreSQL Database directory appears to contain a database; Skipping initialization
```

Находим postgresql.conf и меняем строку на 
data_directory = '/mnt/new_disk'

Перезапкаем кластер заходим в пострес и делаем селект 

```bash
root@87cc2b9431cb:/# su - postgres -c 'psql'

postgres=# SELECT * FROM test;
 c1 
----
 1
(1 row)
```
Видим что данные на месте.

За задание со звездочкой не брался ибо очень долго провозился с основным в основном по собственной глупости.
За краткость изложения прошу извинить, но к концу уже сам забыл обо что спотыкался раз за разом.

