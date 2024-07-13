##  Нагрузочное тестирование и тюнинг PostgreSQL
Дано  кластер postgres 16 на Windows 10.
Скидываем все на дефолтные настройки, что бы было от чего отталкиваться,
запускаем тестирование через pgbench.
```bash
PS C:\Windows\system32> &"C:\Program Files\PostgreSQL\16\bin\pgbench" -c 8 -P 6 -T 60 -U postgres -p 5433 testdb
Password:
pgbench (16.3)
starting vacuum...end.
progress: 6.0 s, 5645.8 tps, lat 1.359 ms stddev 0.734, 0 failed
progress: 12.0 s, 5988.7 tps, lat 1.335 ms stddev 0.712, 0 failed
progress: 18.0 s, 5977.0 tps, lat 1.337 ms stddev 0.715, 0 failed
progress: 24.0 s, 5978.3 tps, lat 1.337 ms stddev 0.737, 0 failed
progress: 30.0 s, 5991.2 tps, lat 1.334 ms stddev 0.729, 0 failed
progress: 36.0 s, 5960.5 tps, lat 1.341 ms stddev 0.773, 0 failed
progress: 42.0 s, 6039.5 tps, lat 1.324 ms stddev 0.705, 0 failed
progress: 48.0 s, 5974.8 tps, lat 1.338 ms stddev 0.782, 0 failed
progress: 54.0 s, 5992.7 tps, lat 1.334 ms stddev 0.755, 0 failed
progress: 60.0 s, 5954.6 tps, lat 1.342 ms stddev 0.730, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 357112
number of failed transactions: 0 (0.000%)
latency average = 1.338 ms
latency stddev = 0.738 ms
initial connection time = 240.391 ms
tps = 5974.109171 (without initial connection time)
PS C:\Windows\system32>
```
И так мы получаем исходное tps = 5974 для сравнения.
Идем в postgresql.auto.conf и вносим изменения 

```text
max_connections = 100
shared_buffers = 8GB
effective_cache_size = 24GB
maintenance_work_mem = 2047MB
checkpoint_completion_target = 0.9
wal_buffers = 32MB
default_statistics_target = 100
random_page_cost = 1.1
work_mem = 41942kB
huge_pages = try
min_wal_size = 1GB
max_wal_size = 4GB
max_worker_processes = 16
max_parallel_workers_per_gather = 4
max_parallel_workers = 16
max_parallel_maintenance_workers = 4
checkpoint_timeout = '200s'
synchronous_commit = 'off'
fsync = 'off'
```

Перезагружам кластер и запускаем тест
```bash
PS C:\Windows\system32> net stop postgresql-x64-16
Служба "postgresql-x64-16 - PostgreSQL Server 16" останавливается.
Служба "postgresql-x64-16 - PostgreSQL Server 16" успешно остановлена.

PS C:\Windows\system32> net start postgresql-x64-16
Служба "postgresql-x64-16 - PostgreSQL Server 16" запускается.
Служба "postgresql-x64-16 - PostgreSQL Server 16" успешно запущена.

PS C:\Windows\system32> &"C:\Program Files\PostgreSQL\16\bin\pgbench" -c 8 -P 6 -T 60 -U postgres -p 5433 testdb
Password:
pgbench (16.3)
starting vacuum...end.
progress: 6.0 s, 8155.3 tps, lat 0.937 ms stddev 0.403, 0 failed
progress: 12.0 s, 8628.5 tps, lat 0.925 ms stddev 0.392, 0 failed
progress: 18.0 s, 8607.0 tps, lat 0.928 ms stddev 0.386, 0 failed
progress: 24.0 s, 8602.2 tps, lat 0.928 ms stddev 0.395, 0 failed
progress: 30.0 s, 8637.2 tps, lat 0.924 ms stddev 0.394, 0 failed
progress: 36.0 s, 8666.8 tps, lat 0.921 ms stddev 0.390, 0 failed
progress: 42.0 s, 8667.7 tps, lat 0.921 ms stddev 0.387, 0 failed
progress: 48.0 s, 8603.2 tps, lat 0.928 ms stddev 0.393, 0 failed
progress: 54.0 s, 8637.7 tps, lat 0.924 ms stddev 0.389, 0 failed
progress: 60.0 s, 8486.7 tps, lat 0.941 ms stddev 0.398, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 514215
number of failed transactions: 0 (0.000%)
latency average = 0.928 ms
latency stddev = 0.393 ms
initial connection time = 258.049 ms
tps = 8606.121318 (without initial connection time)
```
Получаем tps = 8606 против 5974 в исходном тесте, что дает нам где то 44% прироста по tps.
решил сделать еще пару тестов и убрать пару настроек 

```text
shared_buffers = 8GB
effective_cache_size = 24GB
maintenance_work_mem = 2047MB
wal_buffers = 32MB
min_wal_size = 1GB
max_wal_size = 4GB
synchronous_commit = 'off'
fsync = 'off'
```

Результат 
```bash
PS C:\Windows\system32> &"C:\Program Files\PostgreSQL\16\bin\pgbench" -c 8 -P 6 -T 60 -U postgres -p 5433 testdb
Password:
pgbench (16.3)
starting vacuum...end.
progress: 6.0 s, 8122.3 tps, lat 0.943 ms stddev 0.407, 0 failed
progress: 12.0 s, 8631.9 tps, lat 0.925 ms stddev 0.392, 0 failed
progress: 18.0 s, 8655.3 tps, lat 0.922 ms stddev 0.388, 0 failed
progress: 24.0 s, 8628.5 tps, lat 0.925 ms stddev 0.387, 0 failed
progress: 30.0 s, 8618.5 tps, lat 0.926 ms stddev 0.392, 0 failed
progress: 36.0 s, 8668.3 tps, lat 0.921 ms stddev 0.387, 0 failed
progress: 42.0 s, 8703.2 tps, lat 0.917 ms stddev 0.385, 0 failed
progress: 48.0 s, 8614.6 tps, lat 0.927 ms stddev 0.388, 0 failed
progress: 54.0 s, 8640.5 tps, lat 0.924 ms stddev 0.385, 0 failed
progress: 60.0 s, 8644.8 tps, lat 0.924 ms stddev 0.388, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 515606
number of failed transactions: 0 (0.000%)
latency average = 0.925 ms
latency stddev = 0.390 ms
initial connection time = 247.281 ms
tps = 8628.229834 (without initial connection time)
```
Даже лучше чем был немного но все же 8628 против 8606. 
Снова меняем настройки отключая только синхранизацию все остальное по дефолту 
```text
synchronous_commit = 'off'
fsync = 'off'
```
Результат 
```bash
PS C:\Windows\system32> &"C:\Program Files\PostgreSQL\16\bin\pgbench" -c 8 -P 6 -T 60 -U postgres -p 5433 testdb
Password:
pgbench (16.3)
starting vacuum...end.
progress: 6.0 s, 8154.5 tps, lat 0.939 ms stddev 0.436, 0 failed
progress: 12.0 s, 8571.7 tps, lat 0.932 ms stddev 0.393, 0 failed
progress: 18.0 s, 8574.1 tps, lat 0.931 ms stddev 0.394, 0 failed
progress: 24.0 s, 8530.0 tps, lat 0.936 ms stddev 0.393, 0 failed
progress: 30.0 s, 8527.7 tps, lat 0.936 ms stddev 0.399, 0 failed
progress: 36.0 s, 8580.2 tps, lat 0.931 ms stddev 0.392, 0 failed
progress: 42.0 s, 8603.6 tps, lat 0.928 ms stddev 0.389, 0 failed
progress: 48.0 s, 8539.5 tps, lat 0.935 ms stddev 0.391, 0 failed
progress: 54.0 s, 8528.3 tps, lat 0.936 ms stddev 0.396, 0 failed
progress: 60.0 s, 8553.8 tps, lat 0.933 ms stddev 0.394, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 511102
number of failed transactions: 0 (0.000%)
latency average = 0.934 ms
latency stddev = 0.398 ms
initial connection time = 243.733 ms
tps = 8551.008574 (without initial connection time)
```
уже немного хуже чем в предыдущем тесте 8551 против 8628.

Теперь к выводам: 
Максимальный прирост производительности дают две настройки это 
synchronous_commit = 'off' и fsync = 'off', это ассинхронная запипись wal файлов и синхранизация записи на диск, но данные параметры могут привести к потери данных при сбое системы.

Остальная поднастройка системы в плане указания конфегурации сервера тоже дает небольшой прирост это такие параметры как:

shared_buffers - выделение оперативки для кеша таблиц.
effective_cache_size - размер доступного дискового кеша.
maintenance_work_mem - выделение памяти для операций обслуживания типа вакуум и тд.
wal_buffers - размер буфера журнала файлов wal. 
min_wal_size и mas_wal_size соответсвенно размеры wal файлов.

Итог: базовые настройки в целом достаточно производительные, для действительно хорошей оптимизации нужно иметь достаточно высокий скил в данном направлении, а для большиства сойдет принцип "Работает, не трогай".
