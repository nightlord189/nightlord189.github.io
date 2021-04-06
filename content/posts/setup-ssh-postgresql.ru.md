+++ 
draft = false
date = 2021-04-06T00:32:32+06:00
title = "Настройка SSH и установка PostgreSQL на VPS"
slug = "" 
tags = []
categories = []
thumbnail = "/images/tn.png"
description = ""
+++

Тут вкратце расскажу об аренде и начальной настройке VPS, а также установке PostgreSQL. Если это все вы делаете с закрытыми глазами - закрывайте :)

### Аренда VPS
Тут все зависит от вашего провайдера. Здесь и далее я буду показывать примеры с [Hetzner](https://www.hetzner.com/).
1. Регистрируемся на (https://www.hetzner.com).

2. Заходим в кабинет Cloud, выбираем дефолтный проект или создаем новый:
![](/images/posts/setup-ssh-postgresql/01projects.jpg)

3. Нажимаем "Add Server", настраиваем нужную конфигурацию (расположение, ОС, тип, volume, название и др.).
В моем примере я выбираю CX11 (2.49 евро) в Хельсинки на Ubuntu 20.04, без SSH-ключа. 
![](/images/posts/setup-ssh-postgresql/02buy.jpg)

4. Сразу настроим volume (в следующих шагах настроим, чтобы данные БД хранились на нем, что обеспечит легкое масштабирование). В разделе Volume кликаем на "Create Volume", настраиваем размер:
![](/images/posts/setup-ssh-postgresql/03buyvolume.jpg)

5. После создания volume кликаем на "Create & buy now" и ожидаем в консоли, пока создастся наш VPS:
 ![](/images/posts/setup-ssh-postgresql/04created.jpg)

6. После завершения создания вам на почтовый адрес должно прийти письмо с данными для подключения:
 ![](/images/posts/setup-ssh-postgresql/05mail.jpg)

### Безопасность и настройка SSH
Да, я в курсе, что лучше настраивать доступ по SSH-ключу. Но в данном листинге я это не рассматриваю.

7. Для начала подключимся к нашему VPS по SSH. Я предпочитаю работать через MobaXterm, но есть много других аналогов - просто bash, Putty, десятки их ([можно посмотреть тут](https://ruprogi.ru/software/mobaxterm)). Хостом будет адрес, указанный в поле IPv4 в письме, портом - 22, протокол SSH.

После подключения вводим логин root и пароль из письма. Сразу после подключения система затребует от вас сменить пароль, что правильно:

 ![](/images/posts/setup-ssh-postgresql/06connect.jpg)

 8. Создадим нового пользователя (т.к. под рутом подключаться и работать не следует):
 ```
 sudo adduser user1
 usermod -aG sudo user1
 ```
 Теперь попробуйте подключиться под новым пользователем, а не под root.

 9. Поменяем порт SSH с 22 на какой-нибудь другой, например, 8906 (т.к. практически сразу после заведения VPS нас начинают сканить боты и пытаться взломать пароль брутфорсом через дефолтный 22 порт). Для этого откроем конфиг SSH через 
 ```
sudo nano /etc/ssh/sshd_config
 ```
Ищем строку "#Port 22", и меняем ее на "Port 8906". 

10. Нужно также запретить подключение под root-пользователем. Для этого в том же файле ищем строку "PermitRootLogin yes" и меняем ее на "PermitRootLogin no".

 ![](/images/posts/setup-ssh-postgresql/07ssh.jpg)

Затем сохраняем файл:   
* Ctrl+O (запись файла)  
* Enter (подтверждение)  
* Ctrl+X (закрыть редактор)

**Не забываем**, что подключаться нужно уже по новому указанному нами порту!

### Установка и настройка PostgreSQL
11. Установим PostgreSQL:
```
sudo apt update
sudo apt install postgresql postgresql-contrib
```

12. PostgreSQL установлена и к ней уже можно подключаться через консольный клиент psql. Но не думаю, что мы делали это все ради работы из консоли, так что установим пароль на дефолтного пользователя postgres:

```
sudo -u postgres psql postgres
\password postgres
#вводим пароль
\q
```
![](/images/posts/setup-ssh-postgresql/08psql.jpg)

13. Для возможности подключения извне осталось разрешить это в конфигах:

```
sudo nano /etc/postgresql/12/main/postgresql.conf
```

Меняем в редакторе строку "#listen_addresses = 'localhost'" на "listen_addresses = '*'". Сохраняем файл аналогично пункту 10.
Затем перезапускаем PostgreSQL, чтобы обновленные настройки вступили в действие:

```
sudo systemctl restart postgresql
```

14. Теперь если мы попробуем подключиться с внешнего клиента, то сетевой доступ уже будет, однако мы получим ошибку "FATAL: no pg_hba.conf entry for host <>, user "postgres", database "postgres", SSL on".
Для ее разрешения нужно модифицировать еще один конфиг:

```
sudo nano /etc/postgresql/12/main/pg_hba.conf
```
Нужно добавить в конец файла две строки (разделение табами):

```
host    all     all     0.0.0.0/0       md5
host    all     all     ::/0    md5
```
![](/images/posts/setup-ssh-postgresql/09psql.jpg)

Сохраняем файл аналогично пункту 10 и перезапускаем PostgreSQL аналогично пункту 13.


15. Теперь можно подключаться с вашего клиента БД. Я использую DBeaver:

![](/images/posts/setup-ssh-postgresql/10connect_db.jpg)

Далее уже можно создавать новых пользователей, базы данных, настраивать таблицы и др.

### Монтирование volume
Пункты 16-18 нужны только если вы собираетесь хранить данные на отдельном volume для удобства и безопасности данных.

16. Примонтируем volume на сервере. Для того чтобы посмотреть нужные команды, можно в веб-интерфейсе на volume нажать кнопку "Show configuration":

![](/images/posts/setup-ssh-postgresql/11mnt.jpg)

```
mkdir /mnt/volume-hel1-1
mount -o discard,defaults /dev/disk/by-id/scsi-0HC_Volume_10478560 /mnt/volume-hel1-1
```

17. Переместим текущие данные PostgreSQL со старого места на новое в volume:
```
sudo systemctl stop postgresql
sudo rsync -av /var/lib/postgresql /mnt/volume-hel1-1/
```

18. Изменим место хранения в конфигах:
```
sudo nano /etc/postgresql/12/main/postgresql.conf
```

Меняем строку "data_directory = '/var/lib/postgresql/12/main'" на "data_directory = '/mnt/volume-hel1-1/postgresql/12/main'"
Сохраняем файл аналогично пункту 10 и перезапускаем PostgreSQL аналогично пункту 13.
