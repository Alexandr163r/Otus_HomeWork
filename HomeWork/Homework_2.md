## Установка и настройка PostgteSQL в контейнере Docker
ubuntu 22.04 в wsl. все проводил на винде.
Устанавливаем Docker Engine на ubuntu 22.04 
```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
Проверяем установился и работает ли докер.
```bash
alex@DESKTOP-G9FSIV5:~$ sudo docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
c1ec31eb5944: Pull complete
Digest: sha256:94323f3e5e09a8b9515d74337010375a456c909543e1ff1538f5116d38ab3989
Status: Downloaded newer image for hello-world:latest
Hello from Docker!
```
Создаем дерикторию.
```bash
alex@DESKTOP-G9FSIV5:~$ sudo mkdir /var/lib/postgres
```
Проверяем дерикторию 
```bash
alex@DESKTOP-G9FSIV5:~$ ls -ld /var/lib/postgres
drwx------ 19 999 root 4096 Jun 22 16:29 /var/lib/postgres
```
Создадим и запустим контейнер с PostgreSQL 16, сократил немного вывод для краткости.
```bash
sudo docker run -d --name otus -v /var/lib/postgres:/var/lib/postgresql/data -e POSTGRES_PASSWORD=alex -p 5432:5432 postgres:16

Unable to find image 'postgres:16' locally
16: Pulling from library/postgres
2cc3ae149d28: Pull complete
 ...Pull complete 
Digest: sha256:46aa2ee5d664b275f05d1a963b30fff60fb422b4b594d509765c42db46d48881
Status: Downloaded newer image for postgres:16
763fb6031a5bb554654cfef6fb1b4d856a1a6ace4dcaeb402734703510a18d93
```
Создаем контейнер клиента.
```bash
docker run -d --name client --network host postgres:16 tail -f /dev/null
```
Проверяем контейнеры
```bash
alex@DESKTOP-G9FSIV5:/mnt/c/Users/Alex$ docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS         PORTS                                       NAMES
6cecf84a7ada   postgres:16   "docker-entrypoint.s…"   15 minutes ago   Up 5 minutes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   otus
b1d698e4b981   postgres:16   "docker-entrypoint.s…"   20 minutes ago   Up 4 minutes
```

Подключаемся из контейнера клиента к серверу 
```bash
root@DESKTOP-G9FSIV5:/# psql -h 127.0.0.1 -U postgres -d postgres
Password for user postgres:
psql: error: connection to server at "127.0.0.1", port 5432 failed: FATAL:  password authentication failed for user "postgres"
root@DESKTOP-G9FSIV5:/#
```
Не пксукает хотя пароль верный.
Идем в контейнер сервера и меняем настройки 
пароль на всякий случай
```bash
psql -U postgres 
ALTER USER postgres PASSWORD 'alex';
```
и открываем порты. так делать конечно не совсем хорошо, но для учебного проекта сойдет
```bash
echo "local all all trust" >> /var/lib/postgresql/data/pg_hba.conf
echo "host all all 127.0.0.1/32 trust" >> /var/lib/postgresql/data/pg_hba.conf
echo "host all all ::1/128 trust" >> /var/lib/postgresql/data/pg_hba.conf
```
Снова поддключаемся из контейнера клиента к серверу, работает.
```bash
psql (16.3 (Debian 16.3-1.pgdg120+1))
Type "help" for help.
postgres=#
```
Создаем таблицу 
```bash
postgres=# CREATE TABLE test_table (
postgres(#     id SERIAL PRIMARY KEY,
postgres(#     name VARCHAR(255)
postgres(# );
CREATE TABLE
```
Заполняем смотрим
```bash
postgres=# INSERT INTO test_table (name) VALUES ('1'), ('2');
INSERT 0 2
postgres=# SELECT * FROM test_table;
 id | name
----+------
  1 | 1
  2 | 2
(2 rows)
```
Удаляем контейнер сервер
```bash
alex@DESKTOP-G9FSIV5:/mnt/c/Users/Alex$ docker stop otus
otus
alex@DESKTOP-G9FSIV5:/mnt/c/Users/Alex$ docker rm otus
otus
```
Проверяем 
```bash
alex@DESKTOP-G9FSIV5:/mnt/c/Users/Alex$ docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED       STATUS             PORTS     NAMES
b1d698e4b981   postgres:16   "docker-entrypoint.s…"   2 hours ago   Up About an hour             client
```
Создаем по новой контейнер сервер 
```bash
alex@DESKTOP-G9FSIV5:/mnt/c/Users/Alex$ sudo docker run -d --name otus -v /var/lib/postgres:/var/lib/postgresql/data -e POSTGRES_PASSWORD=alex -p 5432:5432 postgres:16
[sudo] password for alex:
5e0c6e9d7245b596f5a95bb52ed9486717d7cb669431840708b7afdc7c8dd4b4
```
Проверяем 
```bash
alex@DESKTOP-G9FSIV5:/mnt/c/Users/Alex$ docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS             PORTS                                       NAMES
5e0c6e9d7245   postgres:16   "docker-entrypoint.s…"   45 seconds ago   Up 44 seconds      0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   otus
b1d698e4b981   postgres:16   "docker-entrypoint.s…"   2 hours ago      Up About an hour                                               client
```
Идем в контейнер клиет проверяем записи, 
```bash
alex@DESKTOP-G9FSIV5:/mnt/c/Users/Alex$ docker exec -it client bash
root@DESKTOP-G9FSIV5:/# psql -h 127.0.0.1 -U postgres -d postgres
Password for user postgres:
psql (16.3 (Debian 16.3-1.pgdg120+1))
Type "help" for help.

postgres=# SELECT * FROM test_table;
 id | name
----+------
  1 | 1
  2 | 2
(2 rows)
```
Они на месте.
Из проблем, если ставить в контейнер только постресс то в нем нет нано, поэтому внесение параметров через эхо, ну еще я не особо люблю работать в консоли это тоже вызвало много затруднений.
