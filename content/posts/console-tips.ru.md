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
remove all images: docker rmi -f $(docker images -q)  
удалить контейнеры:	`docker rm -f $$(docker ps -a -q)`  
удалить все volume:	`docker volume rm $$(docker volume ls -q)`

### Сборка
заменить значение в json-файле (предварительно нужно установить утилиту jq):  
`contents="$(jq '.Key.Key2 = "value"' file.json)" && \echo "${contents}" > file.json`

### SSH
скопировать ssh-ключ на remote-юзера на сервере: `ssh-copy-id -i ~/.ssh/id_rsa.pub <user>@<host>`