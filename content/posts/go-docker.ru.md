+++ 
draft = false
date = 2021-08-08T19:23:14+06:00
title = "Сборка Golang-приложения в Docker"
slug = "" 
tags = ["Docker","Go","Dockerfile"]
categories = []
thumbnail = "images/tn.png"
description = "Dockerfile, multistage и кэширование зависимостей"
+++

Допустим, мы написали [сервис](https://github.com/nightlord189/test.mc) на Go. И теперь хотим его собрать в формате Docker-контейнера.

**1.** Пишем простой Dockerfile:
```Docker
# Базовый образ, содержащий установленный Go
FROM golang:alpine

# Копируем все файлы из корня проекта в образ
# Если нужно копировать не все, то можно это указать в файле .dockerignore
COPY . .

# Запускаем скачивание всех зависимостей (если у нас проект на модулях)
RUN go mod download

# Собираем бинарник
RUN GO111MODULE=on CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o main .

# Указываем, что приложению требуется 8080 порт
EXPOSE 8080

# Запускаем приложение
ENTRYPOINT ["/main"]
```

**2.** Собираем образ:
```
docker build -t <username>/test.mc:latest .
```

В принципе, это уже рабочий вариант, который можно использовать.
Но если присмотреться, у него есть один недостаток - большой размер.  

Почему так происходит?

Потому что при сборке по сути у нас выполняются следующие шаги:
1. копируем исходники проекта
2. скачиваем все зависимости
3. собираем бинарный файл
4. запускаем собранный бинарник

Неэффективность тут в том, что в контейнере у нас остаются исходники проекта и все сторонние зависимости. Хотя для работы они уже не нужны, т.к. Go собрал нам небольшой бинарный файл.

Как организовать сборку по-другому? Логичным решением видится копировать в конечный контейнер только бинарник, и, например, конфиг, чтобы размер контейнера был минимальным. Т.е. мы скачаем зависимости и соберем приложение в одном контейнере, а запускать его будем в другом, где даже не будет установленного Go (т.к. для запуска собранного приложения он уже не нужен).

Сделать это можно с помощью технологии [multistage-билдов](https://habr.com/ru/post/349802/).

**3.** Модифицируем наш Dockerfile:

```Docker
# Базовый образ, содержащий установленный Go
# В этом образе мы будем собирать приложение
FROM golang:alpine AS builder

# Указываем директорию, где будем работать с приложением в контейнере сборки
WORKDIR /build

# Копируем все файлы из корня проекта в образ
# Если нужно копировать не все, то можно это указать в файле .dockerignore
COPY . .

# Запускаем скачивание всех зависимостей (если у нас проект на модулях)
RUN go mod download

# Собираем бинарник
RUN GO111MODULE=on CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o main .

# Указываем новый базовый образ, куда мы скопируем только итоги сборки - бинарник и конфиг
FROM scratch

# Копируем в итоговый образ нужные файлы
COPY --from=builder /build/main /
COPY --from=builder /build/config.json /

# Указываем, что приложению требуется 8080 порт
EXPOSE 8080

# Запускаем приложение
ENTRYPOINT ["/main"]
```
**4.** Собираем образ аналогично пункту 2. Как можно увидеть, размер образа значительно уменьшился и составляет теперь несколько мегабайт.

**5.** Еще одно возможное улучшение, которое можно дополнить - уменьшение скорости сборки. В тестовом сервисе всего две прямых зависимости, но в любом полноценном приложении их будет несколько десятков. Соответственно, выкачивать их все с нуля может занять несколько минут, а мы же хотим быстрый CI/CD и все дела :)

Для этого можно закэшировать основные (или все) зависимости приложения в отдельный базовый образ и собирать приложение уже в нем. Тогда т.к. скачивания зависимостей уже не будет, сборка пойдет быстрее.

Для создания такого базового образа можно использовать просто go.mod без самого приложения и отдельный Dockerfile:

go.mod:
```
module <username>/golangweb

go 1.13

require (
	github.com/alecthomas/template v0.0.0-20190718012654-fb15b899a751
	github.com/gin-contrib/cors v1.3.1 // indirect
	github.com/gin-gonic/gin v1.7.2
	github.com/go-openapi/jsonreference v0.19.6 // indirect
	github.com/go-openapi/spec v0.20.3 // indirect
	github.com/go-openapi/swag v0.19.15 // indirect
	github.com/go-playground/validator/v10 v10.6.1 // indirect
	github.com/golang-jwt/jwt v3.2.1+incompatible // indirect
	github.com/golang/protobuf v1.5.2 // indirect
	github.com/google/uuid v1.2.0
	github.com/jackc/pgproto3/v2 v2.1.0 // indirect
	github.com/json-iterator/go v1.1.11 // indirect
	github.com/leodido/go-urn v1.2.1 // indirect
	github.com/lunixbochs/struc v0.0.0-20200707160740-784aaebc1d40 // indirect
	github.com/mailru/easyjson v0.7.7 // indirect
	github.com/mattn/go-isatty v0.0.13 // indirect
	github.com/nightlord189/firect v0.0.0-20210307220534-96b18afa5c05
	github.com/pkg/errors v0.9.1 // indirect
	github.com/swaggo/gin-swagger v1.3.0
	github.com/swaggo/swag v1.7.0
	github.com/ugorji/go v1.2.6 // indirect
	golang.org/x/crypto v0.0.0-20210616213533-5ff15b29337e // indirect
	golang.org/x/net v0.0.0-20210614182718-04defd469f4e // indirect
	golang.org/x/sys v0.0.0-20210616094352-59db8d763f22 // indirect
	golang.org/x/tools v0.1.3 // indirect
	google.golang.org/protobuf v1.26.0
	gorm.io/driver/postgres v1.1.0
	gorm.io/gorm v1.21.11
)
```

Dockerfile:
```Docker
FROM golang:alpine

WORKDIR /

RUN apk add build-base

COPY go.mod .

RUN go mod download
```

Далее собираем такой образ и пушим в DockerHub:
```bash
docker login --u <username>
docker build -t <username>/golangweb:latest .
docker push <username>/golangweb:latest
```

И теперь в Dockerfile основного проекта остается поменять базовый образ с golang:alpine на свой:
```Docker
FROM <username>/golangweb:latest AS builder
```

Если в базовом образе вдруг не окажется какой-то зависимости (например, вы ее недавно добавили и она специфична для конкретного приложения), то она скачается в процессе сборки.

Ссылки:
+ [Репозиторий с примером микросервиса на Go](https://github.com/nightlord189/test.mc)
+ [Репозиторий с базовым образом](https://github.com/nightlord189/golangweb)
+ [Собранный базовый образ на DockerHub](https://hub.docker.com/repository/docker/nightlord189/golangweb)