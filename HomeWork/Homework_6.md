##  Работа с журналами

Настройте выполнение контрольной точки раз в 30 секунд

```bash
postgres=# ALTER SYSTEM SET checkpoint_timeout = '30s';
```
Перезагружаем и проверяем настройки 
```bash
PS C:\Windows\system32> net stop postgresql-x64-16
Служба "postgresql-x64-16 - PostgreSQL Server 16" останавливается.
Служба "postgresql-x64-16 - PostgreSQL Server 16" успешно остановлена.
PS C:\Windows\system32> net start postgresql-x64-16
Служба "postgresql-x64-16 - PostgreSQL Server 16" запускается.
Служба "postgresql-x64-16 - PostgreSQL Server 16" успешно запущена.

PS C:\Windows\system32> &"C:\Program Files\PostgreSQL\16\bin\psql.exe" -p 5433 -U postgres -d postgres

postgres=# SHOW checkpoint_timeout;
 checkpoint_timeout
--------------------
 30s
```
Смотрим где хранятся логи базы, по умолчанию для винды это C:\Program Files\PostgreSQL\16\data\log

Проверяем режим синхронный либо асинхронный 
```bash
SHOW synchronous_commit;
synchronous_commit|
------------------+
on                |
```
видим что синхронный.

Запускаем генерацию нагрузки с помощью pgbench на 10 минут 
```bash
PS C:\Windows\system32> &"C:\Program Files\PostgreSQL\16\bin\pgbench" -c 8 -P 6 -T 600 -U postgres -p 5433 testdb
Password:
pgbench (16.3)
starting vacuum...end.
progress: 6.0 s, 5513.7 tps, lat 1.390 ms stddev 0.762, 0 failed
progress: 12.0 s, 5791.2 tps, lat 1.380 ms stddev 0.723, 0 failed
progress: 18.0 s, 5769.3 tps, lat 1.385 ms stddev 0.746, 0 failed
progress: 24.0 s, 5771.5 tps, lat 1.385 ms stddev 0.724, 0 failed
progress: 30.0 s, 5820.5 tps, lat 1.373 ms stddev 0.722, 0 failed
progress: 36.0 s, 5730.3 tps, lat 1.395 ms stddev 0.744, 0 failed
progress: 42.0 s, 5746.0 tps, lat 1.391 ms stddev 0.705, 0 failed
progress: 48.0 s, 5760.2 tps, lat 1.388 ms stddev 0.737, 0 failed
progress: 54.0 s, 5874.2 tps, lat 1.361 ms stddev 0.684, 0 failed
progress: 60.0 s, 5817.7 tps, lat 1.374 ms stddev 0.695, 0 failed
progress: 66.0 s, 5784.1 tps, lat 1.382 ms stddev 0.700, 0 failed
progress: 72.0 s, 5915.2 tps, lat 1.351 ms stddev 0.689, 0 failed
progress: 78.0 s, 5999.0 tps, lat 1.332 ms stddev 0.682, 0 failed
progress: 84.0 s, 6024.2 tps, lat 1.327 ms stddev 0.686, 0 failed
progress: 90.0 s, 5984.8 tps, lat 1.336 ms stddev 0.678, 0 failed
progress: 96.0 s, 5921.6 tps, lat 1.350 ms stddev 0.700, 0 failed
progress: 102.0 s, 5903.7 tps, lat 1.354 ms stddev 0.720, 0 failed
progress: 108.0 s, 5947.0 tps, lat 1.344 ms stddev 0.710, 0 failed
progress: 114.0 s, 5973.0 tps, lat 1.338 ms stddev 0.715, 0 failed
progress: 120.0 s, 6035.3 tps, lat 1.325 ms stddev 0.701, 0 failed
progress: 126.0 s, 5910.2 tps, lat 1.353 ms stddev 0.724, 0 failed
progress: 132.0 s, 6002.3 tps, lat 1.332 ms stddev 0.707, 0 failed
progress: 138.0 s, 5831.0 tps, lat 1.371 ms stddev 0.729, 0 failed
progress: 144.0 s, 5838.8 tps, lat 1.369 ms stddev 0.718, 0 failed
progress: 150.0 s, 5759.9 tps, lat 1.388 ms stddev 0.734, 0 failed
progress: 156.0 s, 5884.7 tps, lat 1.358 ms stddev 0.761, 0 failed
progress: 162.0 s, 5975.0 tps, lat 1.338 ms stddev 0.705, 0 failed
progress: 168.0 s, 5964.7 tps, lat 1.340 ms stddev 0.752, 0 failed
progress: 174.0 s, 5980.5 tps, lat 1.337 ms stddev 0.709, 0 failed
progress: 180.0 s, 6021.8 tps, lat 1.327 ms stddev 0.702, 0 failed
progress: 186.0 s, 5949.3 tps, lat 1.344 ms stddev 0.715, 0 failed
progress: 192.0 s, 5808.3 tps, lat 1.376 ms stddev 0.783, 0 failed
progress: 198.0 s, 5825.8 tps, lat 1.372 ms stddev 0.723, 0 failed
progress: 204.0 s, 5905.7 tps, lat 1.354 ms stddev 0.722, 0 failed
progress: 210.0 s, 6021.8 tps, lat 1.327 ms stddev 0.699, 0 failed
progress: 216.0 s, 5918.2 tps, lat 1.351 ms stddev 0.723, 0 failed
progress: 222.0 s, 5935.5 tps, lat 1.347 ms stddev 0.714, 0 failed
progress: 228.0 s, 5770.8 tps, lat 1.385 ms stddev 0.711, 0 failed
progress: 234.0 s, 5910.7 tps, lat 1.353 ms stddev 0.701, 0 failed
progress: 240.0 s, 6015.8 tps, lat 1.329 ms stddev 0.671, 0 failed
progress: 246.0 s, 5971.5 tps, lat 1.339 ms stddev 0.690, 0 failed
progress: 252.0 s, 5982.2 tps, lat 1.336 ms stddev 0.692, 0 failed
progress: 258.0 s, 6030.3 tps, lat 1.326 ms stddev 0.681, 0 failed
progress: 264.0 s, 5967.3 tps, lat 1.340 ms stddev 0.694, 0 failed
progress: 270.0 s, 6022.9 tps, lat 1.327 ms stddev 0.675, 0 failed
progress: 276.0 s, 5941.6 tps, lat 1.345 ms stddev 0.702, 0 failed
progress: 282.0 s, 5870.5 tps, lat 1.362 ms stddev 0.703, 0 failed
progress: 288.0 s, 5939.8 tps, lat 1.346 ms stddev 0.695, 0 failed
progress: 294.0 s, 5913.8 tps, lat 1.352 ms stddev 0.693, 0 failed
progress: 300.0 s, 6045.3 tps, lat 1.322 ms stddev 0.671, 0 failed
progress: 306.0 s, 5956.5 tps, lat 1.342 ms stddev 0.689, 0 failed
progress: 312.0 s, 5973.9 tps, lat 1.338 ms stddev 0.688, 0 failed
progress: 318.0 s, 6002.7 tps, lat 1.332 ms stddev 0.677, 0 failed
progress: 324.0 s, 5939.7 tps, lat 1.346 ms stddev 0.691, 0 failed
progress: 330.0 s, 6007.3 tps, lat 1.331 ms stddev 0.672, 0 failed
progress: 336.0 s, 5969.7 tps, lat 1.339 ms stddev 0.691, 0 failed
progress: 342.0 s, 5956.2 tps, lat 1.342 ms stddev 0.693, 0 failed
progress: 348.0 s, 5947.0 tps, lat 1.344 ms stddev 0.691, 0 failed
progress: 354.0 s, 5969.0 tps, lat 1.339 ms stddev 0.678, 0 failed
progress: 360.0 s, 5942.2 tps, lat 1.345 ms stddev 0.678, 0 failed
progress: 366.0 s, 5840.8 tps, lat 1.368 ms stddev 0.695, 0 failed
progress: 372.0 s, 5935.8 tps, lat 1.347 ms stddev 0.692, 0 failed
progress: 378.0 s, 5974.5 tps, lat 1.338 ms stddev 0.672, 0 failed
progress: 384.0 s, 5923.2 tps, lat 1.350 ms stddev 0.699, 0 failed
progress: 390.0 s, 5954.8 tps, lat 1.342 ms stddev 0.682, 0 failed
progress: 396.0 s, 5941.1 tps, lat 1.346 ms stddev 0.694, 0 failed
progress: 402.0 s, 5939.4 tps, lat 1.346 ms stddev 0.698, 0 failed
progress: 408.0 s, 5926.3 tps, lat 1.349 ms stddev 0.646, 0 failed
progress: 414.0 s, 5931.0 tps, lat 1.348 ms stddev 0.691, 0 failed
progress: 420.0 s, 6006.7 tps, lat 1.331 ms stddev 0.671, 0 failed
progress: 426.0 s, 5954.0 tps, lat 1.343 ms stddev 0.691, 0 failed
progress: 432.0 s, 5943.6 tps, lat 1.345 ms stddev 0.683, 0 failed
progress: 438.0 s, 5994.7 tps, lat 1.333 ms stddev 0.677, 0 failed
progress: 444.0 s, 5936.3 tps, lat 1.347 ms stddev 0.687, 0 failed
progress: 450.0 s, 5983.5 tps, lat 1.336 ms stddev 0.673, 0 failed
progress: 456.0 s, 5935.7 tps, lat 1.347 ms stddev 0.697, 0 failed
progress: 462.0 s, 5923.5 tps, lat 1.350 ms stddev 0.712, 0 failed
progress: 468.0 s, 5899.5 tps, lat 1.355 ms stddev 0.702, 0 failed
progress: 474.0 s, 5837.3 tps, lat 1.369 ms stddev 0.687, 0 failed
progress: 480.0 s, 5969.7 tps, lat 1.339 ms stddev 0.676, 0 failed
progress: 486.0 s, 5933.5 tps, lat 1.347 ms stddev 0.695, 0 failed
progress: 492.0 s, 5935.5 tps, lat 1.347 ms stddev 0.710, 0 failed
progress: 498.0 s, 5897.5 tps, lat 1.355 ms stddev 0.689, 0 failed
progress: 504.0 s, 5934.5 tps, lat 1.347 ms stddev 0.689, 0 failed
progress: 510.0 s, 5985.7 tps, lat 1.336 ms stddev 0.679, 0 failed
progress: 516.0 s, 5935.8 tps, lat 1.347 ms stddev 0.693, 0 failed
progress: 522.0 s, 5909.6 tps, lat 1.353 ms stddev 0.721, 0 failed
progress: 528.0 s, 5896.9 tps, lat 1.356 ms stddev 0.649, 0 failed
progress: 534.0 s, 5944.8 tps, lat 1.345 ms stddev 0.676, 0 failed
progress: 540.0 s, 5972.4 tps, lat 1.338 ms stddev 0.674, 0 failed
progress: 546.0 s, 5906.7 tps, lat 1.353 ms stddev 0.680, 0 failed
progress: 552.0 s, 5910.5 tps, lat 1.352 ms stddev 0.680, 0 failed
progress: 558.0 s, 5929.8 tps, lat 1.348 ms stddev 0.677, 0 failed
progress: 564.0 s, 5956.3 tps, lat 1.342 ms stddev 0.679, 0 failed
progress: 570.0 s, 5963.5 tps, lat 1.340 ms stddev 0.680, 0 failed
progress: 576.0 s, 5867.7 tps, lat 1.362 ms stddev 0.695, 0 failed
progress: 582.0 s, 5922.0 tps, lat 1.350 ms stddev 0.705, 0 failed
progress: 588.0 s, 5907.3 tps, lat 1.353 ms stddev 0.741, 0 failed
progress: 594.0 s, 5831.5 tps, lat 1.371 ms stddev 0.686, 0 failed
progress: 600.0 s, 5956.2 tps, lat 1.342 ms stddev 0.674, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 3551491
number of failed transactions: 0 (0.000%)
latency average = 1.350 ms
latency stddev = 0.699 ms
initial connection time = 244.681 ms
tps = 5921.525083 (without initial connection time)
PS C:\Windows\system32>
```
Проверяем сколько файлов в папке pg_wal сгенерировалось в мегабайтах, дата тут до запуска pgbench для фильтрации
```bash
SELECT SUM(size) / (1024 * 1024) AS total_size_mb
FROM pg_ls_waldir() AS a
WHERE a.modification > '2024-07-06 18:44:29.000 +0400';

total_size_mb        |
---------------------+
1856.0000000000000000|
```

Проверяем работу контрольных точек идет в папку логов пострес и смотрим 
```text
2024-07-06 18:52:55.093 +04 [14924] СООБЩЕНИЕ:  начата контрольная точка: time
2024-07-06 18:53:22.131 +04 [14924] СООБЩЕНИЕ:  контрольная точка завершена: записано буферов: 6690 (5.1%); добавлено файлов WAL 0, удалено: 0, переработано: 6; запись=26.936 сек., синхр.=0.016 сек., всего=27.039 сек.; синхронизировано_файлов=18, самая_долгая_синхр.=0.005 сек., средняя=0.001 сек.; расстояние=94820 kB, ожидалось=95730 kB; lsn=2/440016F0, lsn redo=2/3EC85878
2024-07-06 18:53:25.138 +04 [14924] СООБЩЕНИЕ:  начата контрольная точка: time
2024-07-06 18:53:52.186 +04 [14924] СООБЩЕНИЕ:  контрольная точка завершена: записано буферов: 3310 (2.5%); добавлено файлов WAL 0, удалено: 0, переработано: 6; запись=26.952 сек., синхр.=0.009 сек., всего=27.049 сек.; синхронизировано_файлов=11, самая_долгая_синхр.=0.002 сек., средняя=0.001 сек.; расстояние=93589 kB, ожидалось=95516 kB; lsn=2/49D282F0, lsn redo=2/447EAFB0
2024-07-06 18:53:55.187 +04 [14924] СООБЩЕНИЕ:  начата контрольная точка: time
2024-07-06 18:54:22.140 +04 [14924] СООБЩЕНИЕ:  контрольная точка завершена: записано буферов: 4362 (3.3%); добавлено файлов WAL 0, удалено: 0, переработано: 6; запись=26.851 сек., синхр.=0.013 сек., всего=26.953 сек.; синхронизировано_файлов=19, самая_долгая_синхр.=0.004 сек., средняя=0.001 сек.; расстояние=95249 kB, ожидалось=95489 kB; lsn=2/4F77A760, lsn redo=2/4A4EF3C0
2024-07-06 18:54:25.151 +04 [14924] СООБЩЕНИЕ:  начата контрольная точка: time
2024-07-06 18:54:52.094 +04 [14924] СООБЩЕНИЕ:  контрольная точка завершена: записано буферов: 3372 (2.6%); добавлено файлов WAL 0, удалено: 0, переработано: 5; запись=26.862 сек., синхр.=0.009 сек., всего=26.943 сек.; синхронизировано_файлов=13, самая_долгая_синхр.=0.002 сек., средняя=0.001 сек.; расстояние=92537 kB, ожидалось=95194 kB; lsn=2/54C5C7F0, lsn redo=2/4FF4DA28
2024-07-06 18:55:55.111 +04 [14924] СООБЩЕНИЕ:  начата контрольная точка: time
```
Сбоев в расписании я не заметил, на вид отработало штатно, возможно нужно было задать более высокую нагрузку, тогда насколько я знаю пострес мог бы делать больше контрольных точек, но это уже что то подкопотное.

Меняем режим на асинхронный и перезапускаем кластер
```bash 
postgres=# ALTER SYSTEM SET synchronous_commit = 'off';
ALTER SYSTEM
postgres=# ALTER SYSTEM SET fsync = 'off';
ALTER SYSTEM
postgres=# exit
PS C:\Windows\system32> net stop postgresql-x64-16
Служба "postgresql-x64-16 - PostgreSQL Server 16" останавливается.
Служба "postgresql-x64-16 - PostgreSQL Server 16" успешно остановлена.

PS C:\Windows\system32> net start postgresql-x64-16
Служба "postgresql-x64-16 - PostgreSQL Server 16" запускается.
Служба "postgresql-x64-16 - PostgreSQL Server 16" успешно запущена.
```
Снова проверяем размер файлов 

```bash
SELECT SUM(size) / (1024 * 1024) AS total_size_mb
FROM pg_ls_waldir() AS a
WHERE a.modification > '2024-07-06 19:36:29.000 +0400';

total_size_mb        |
---------------------+
2480.0000000000000000|
```
И видим что общий размер сгенерированных файлов увеличился, в целом все просто количество завписей wal файлов зависит от количества транзакций, в синхроном режиме tps в тесте было  5906 в среднем, в асинхронном 8500, больше транзакций больше файлов, по контрольным точкам сдвигов не заметил.

Создаем новый кластер с параметром --data_checksums
теста и так многовато выходит так что опишу словами.
созадем таблицу и заполняем парой значений.
через запрос выясняим уид таблицы.  
```bash
SELECT oid, relname FROM pg_class WHERE relname = 'test';
```
идем по пути PostgreSQL\16\data\base\16401
и по номеру определяем наш файл таблицы.
открываем с помощью тестового редактора в моем случае это notepad++, и меняем пару значений, созраняем
перезапускаем кластер  
ERROR:  page verification failed, calculated checksum
для ее устранения, меняем data_checksums на off и перезапускаем класстер, тем самым игнорируя проверку.
Для проверки текущего состояния data_checksums используем
```bash
show data_checksums;
data_checksums
--------------
off           
```
