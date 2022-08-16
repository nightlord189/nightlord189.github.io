+++ 
draft = false
date = 2021-08-01T23:05:42+06:00
title = "Console tips & tricks"
slug = "" 
tags = []
categories = []
thumbnail = "images/tn.png"
description = "пополняемый сборник моих консольных команд"
+++

Буду дополнять сюда разные очевидные и не очень консольные команды, которые использую/использовал.

### Docker
сборка образа: `docker build -t user/some.service:latest .`  
push в registry:  `docker push user/some.service:latest`  
pull:  `docker pull user/some.service:latest`  
запуск: `docker run -d -p 3001:3000 --restart always --name some.service user/some.service:latest`  
запуск с bash: `docker run --rm -it --entrypoint bash <image>`  
remove all images: `docker rmi -f $(docker images -q)`  
удалить контейнеры:	`docker rm -f $(docker ps -a -q)`  
удалить все volume:	`docker volume rm $(docker volume ls -q)`  
скопировать файл в контейнер: `docker cp <filename> <container>:<filename>`  
подключиться к контейнеру: `docker exec -it <container> /bin/bash`

### Сборка
заменить значение в json-файле (предварительно нужно установить утилиту jq):  
`contents="$(jq '.Key.Key2 = "value"' file.json)" && \echo "${contents}" > file.json`

### SSH
скопировать ssh-ключ на remote-юзера на сервере: `ssh-copy-id -i ~/.ssh/id_rsa.pub <user>@<host>`  
скопировать файл на удаленный сервер: `scp <file> <user>@<host>:<file>`  
скопировать директорию рекурсивно на удаленный сервер: `scp -pr <file> user1@<host>:<file>`  
скопировать файл с удаленного сервера: `scp <user>@<host>:<file> <local_path>`  
узнать инфу для known_hosts: `ssh-keyscan <host>`

### Go
очистить кэш модулей: `go clean --modcache`  
прогон тестов во всем проекте рекурсивно: `go test ./...`

### Git
удалить локальный тег: `git tag -d <tag_name>`  
удалить remote тег: `git push --delete origin <tag_name>`

### SQL
TRUNCATE всех таблиц в текущей схеме БД:
```sql
DO $$ DECLARE
    r RECORD;
BEGIN
    FOR r IN (SELECT tablename FROM pg_tables WHERE schemaname = current_schema()) LOOP
        EXECUTE 'TRUNCATE TABLE ' || quote_ident(r.tablename) || ' CASCADE';
    END LOOP;
END $$;
```

### PowerShell
вывести переменную окружения: `echo $env:ENV_NAME`

### Конфиги и пути:
конфиг OpenVPN-Server: `/etc/openvpn/server/server.conf`. 

### Gitlab CI
Очистка volume, image, containers на раннере:
```
# Remove exited containers
docker ps -a -q -f status=exited | xargs --no-run-if-empty docker rm -v

# Remove dangling images
docker images -f "dangling=true" -q | xargs --no-run-if-empty docker rmi

# Remove unused images
docker images | awk '/ago/  { print $3}' | xargs --no-run-if-empty docker rmi

# Remove dangling volumes
docker volume ls -qf dangling=true | xargs --no-run-if-empty docker volume rm
```
