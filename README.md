# toGoTech_TestCase
![Apache](https://img.shields.io/badge/-Apache-D22128?style=flat&logo=apache&logoColor=white)
![Nginx](https://img.shields.io/badge/-Nginx-269539?style=flat&logo=nginx&logoColor=white)
![MySQL](https://img.shields.io/badge/-MySQL-4479A1?style=flat&logo=mysql&logoColor=white)
![MongoDB](https://img.shields.io/badge/-MongoDB-47A248?style=flat&logo=mongodb&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/-PostgreSQL-336791?style=flat&logo=postgresql&logoColor=white)
![PHP](https://img.shields.io/badge/-PHP-777BB4?style=flat&logo=php&logoColor=white)
![HTML](https://img.shields.io/badge/-HTML-E34F26?style=flat&logo=html5&logoColor=white)

### Тестовое задание выполнялось на ОС **Ubuntu 22.04.3 LTS**

## Настройка сервера
- Настройте сервер для предоставления услуг веб-хостинга (Apache, Nginx, MySQL и PHP)

Прежде чем идти на сервер было принято решение получить для веб-хостинга доменное имя. Для наших задач вполне подойдет сервис с возможностью зарегистрировать домен бесплатно. Выбор пал на [No-IP](https://www.noip.com/). Пару кликов спустя домен **testcase-togotech.ddns.net** был успешно создан. Пока информация о новом домене доходит до серверов DNS займемся основной задачей.

Веб-хостинг, который нам следует развернуть не заявлен явно. Это не проблема, создадим его с нуля! Это будет простой, одностраничный сервис для публикации записей, которые будут храниться в БД на MySQL. Поехали! 

Авторизовавшись на сервере по ssh сразу же обновляем пакетный менеджер ```apt```:
```
sudo apt update
```
Затем устанавливаем весь необходимый стек технологий одной командой:
```
sudo apt install apache2 mysql-server php libapache2-mod-php php-mysql nginx
```
Заходим в MySQL Shell:
```
mysql -u root -p

# MySQL попросит пароль. Вводим пароль от нашего сервера

Enter password:

# И можем работать с СУБД
```
Создаем нашу БД:
```
CREATE DATABASE test_db;
```
Выбираем ее для работы и создаем таблицу: 
```
USE test_db;

CREATE TABLE posts (
    id INT AUTO_INCREMENT PRIMARY KEY,
    content TEXT
);
```
Выходим:
```
exit;
```
Теперь настраиваем Apache.
Создаём для него виртуальный хост:
```
sudo nano /etc/apache2/sites-available/test-site.conf
```
Прописываем конфиг и сохраняем его:
```
<VirtualHost *:86>
    ServerAdmin yaroslav.belyanski@yandex.ru
    DocumentRoot /var/www/html/test-site
    ServerName testcase-togotech.ddns.net

    <Directory /var/www/html/test-site/>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
Дабы **Аpache2** не конфликтовал с **Nginx** из-за 80 порта, то и в остальных конфигурационных файлах пропишем 86 порт. Идем в конфиг **Apache2** и заменяем эти настройи...
```
nano /etc/apache2/ports.conf
```
```
Listen 80

<IfModule ssl_module>
        Listen 443
</IfModule>

<IfModule mod_gnutls.c>
        Listen 443
</IfModule>

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet

```
... на эти.
```
Listen 86

<IfModule ssl_module>
        Listen 7443
</IfModule>

<IfModule mod_gnutls.c>
        Listen 7443
</IfModule>

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet

```
Активируем сайт и перезапускаем Apache последовательным выполнением следующих команд:
```
sudo systemctl start apache2
sudo a2ensite test-site
sudo systemctl restart apache2
```
Настройка Nginx.
Создаем конфиг.
```
sudo nano /etc/nginx/sites-available/test-site
```
Прописываем его:
```
server {
    listen 80;
    server_name testcase-togotech.ddns.net;

    root /var/www/html/test-site;
    index index.php index.html index.htm;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    error_log /var/log/nginx/test-site_error.log;
    access_log /var/log/nginx/test-site_access.log;
}
```
Создаем символическую ссылку и перезапускаем Nginx двумя последовательными командами:
```
sudo ln -s /etc/nginx/sites-available/test-site /etc/nginx/sites-enabled
sudo systemctl restart nginx
```

Создаем директорию для PHP-страницы, а затем сам файл страницы:
```
sudo mkdir /var/www/html/test-site
sudo nano /var/www/html/test-site/index.php
```
Наполняем ее кодом:
```
<?php
$conn = new mysqli("localhost", "<имя_пользователя_БД>", "<пароль_БД>", "test_db");

if ($conn->connect_error) {
    die("Connection failed: " . $conn->connect_error);
}

if ($_SERVER["REQUEST_METHOD"] == "POST" && isset($_POST["content"])) {
    $content = $_POST["content"];
    $sql = "INSERT INTO posts (content) VALUES ('$content')";
    $conn->query($sql);
}

$result = $conn->query("SELECT * FROM posts");
?>

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Test Page</title>
</head>
<body>
    <h1>Simple Blog</h1>
    <form method="post" action="">
        <label for="content">Add a Post:</label><br>
        <textarea id="content" name="content" rows="4" cols="50"></textarea><br>
        <input type="submit" value="Submit">
    </form>

    <h2>Posts:</h2>
    <ul>
        <?php while ($row = $result->fetch_assoc()) : ?>
            <li><?= $row["content"] ?></li>
        <?php endwhile; ?>
    </ul>
</body>
</html>
```
**Примечание:** указания учетных данных БД плохая и очень плохая практика.



Убеждаемся, что у проекта есть правильные разрешения:
```
sudo chown -R www-data:www-data /var/www/html/test-site
```
Выйдем за рамки задания и настроим SSL
Установка certbot.
Чтобы установить certbot, понадобится пакетный менеджер snap. Установливаем его командой:
```
sudo apt install snapd
```


Возможно сервер попросит вам перезагрузить операционную систему. Сделаем это, а потом последовательно выполняем команды:
```
# Установка и обновление зависимостей для пакетного менеджера snap.
sudo snap install core; 
sudo snap refresh core
# При успешной установке зависимостей в терминале выведется:
# core 16-2.58.2 from Canonical✓ installed 

# Установка пакета certbot.
sudo snap install --classic certbot
# При успешной установке пакета в терминале выведется:
# certbot 2.3.0 from Certbot Project (certbot-eff✓) installed

# Создание ссылки на certbot в системной директории,
# чтобы у пользователя с правами администратора был доступ к этому пакету.
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```
Запускаем certbot и получаем SSL-сертификат

Чтобы начать процесс получения сертификата, вводим команду:
```
sudo certbot --nginx
```

В процессе оформления сертификата нужно будет указать свою электронную почту и ответить на несколько вопросов.

Далее система оповестит нас о том, что учётная запись зарегистрирована и попросит указать имена, для которых вы хотели бы активировать HTTPS:
```
Account registered.

Which names would you like to activate HTTPS for?
We recommend selecting either all domains, or all domains in a VirtualHost/server block.

1: <доменное_имя_проекта>

Select the appropriate numbers separated by commas and/or spaces, or leave input
blank to select all options shown (Enter 'c' to cancel):
```
Вводим 1 или не вводим ничего и нажимаем Enter.

Перезагружаем Nginx:
```
sudo systemctl reload nginx
```
И тут бы всему заработать, но нет. При первом прогоне, на другом сервере оно действительно работало. Однако при повторном разворачивании сервис наотрез отказался отображаться в браузере. Могу со 100% уверенностью сказать, что проблема с подключением к **MySQL**, поскольку статичная страница, без инструкций по подключению к **MySQL**, на **PHP** по всем адресам отображается корректно. Как бы то ни было, [ссылка работает](http://testcase-togotech.ddns.net//) и там размещен код на **HTML**, который демонстрирует как это должно было выглядеть.


# Сетевая инфраструктура
- Настройте базовый файрвол с использованием iptables или ufw для ограничения входящего трафика, разрешив только необходимые порты

Выбираем ufw 

Разрешим следующие порты:
```
80 — HTTP;
443 — HTTPS;
22 — SSH.
```
С портами 80 и 443 будут работать пользователи, делая запросы к приложению, запущенному на удалённом сервере. Порт 22 нужен, чтобы вы могли подключаться к серверу по SSH. 

С Ubuntu файрвол ufw идёт в комплекте, но по умолчанию он выключен. Наша задача его включить.

Указываем файрволу, какие порты должны остаться открытыми. Для этого выполняем на сервере две команды по очереди:
```
sudo ufw allow 'Nginx Full'
sudo ufw allow OpenSSH
```
Команда sudo ufw allow 'Nginx Full' активирует разрешение принимать запросы на порты 80 и 443.
Команда sudo ufw allow OpenSSH активирует разрешение для порта 22 — для соединения по SSH. Если этот порт не открыть, то доступ к удалённому серверу будет закрыт сразу после включения файрвола, и вы туда больше не попадёте.

Вводим
```
sudo ufw enable
```
В терминале выведется запрос на подтверждение операции с предупреждением, что команда может оборвать SSH-соединение
```
Command may disrupt existing ssh connections. Proceed with operation (y|n)?
```
Подтверждаем операцию — вводим **y** и нажимаем Enter.

Получаем вот такое сообщение…
```
Firewall is active and enabled on system startup
```

Проверяем внесённые изменения:
```
sudo ufw status
```
Файрвол ufw сообщит вам, что он «активен» и разрешает принимать запросы на порты, которые мы указали:
```
Status: active

To                         Action      From
--                         ------      ----
Nginx Full                 ALLOW       Anywhere
OpenSSH                    ALLOW       Anywhere
Nginx Full (v6)            ALLOW       Anywhere (v6)
OpenSSH (v6)               ALLOW       Anywhere (v6)

```
# Базы данных
- Установите MongoDB, настройте аутентификацию и примените меры
безопасности.
Устанавливаем ключ для **MongoDB**:
```
curl -fsSL https://www.mongodb.org/static/pgp/server-4.4.asc | sudo apt-key add -
```
На этом этапе может возникнуть ошибка, вызванная отсутсвием актуальной сборки **Mongo DB** для имеющейся версии Ubuntu.

Решить ее можно принудительной установкой **libssl1.1**:
```
echo "deb http://security.ubuntu.com/ubuntu focal-security main" | sudo tee /etc/apt/sources.list.d/focal-security.list

sudo apt-get update
sudo apt-get install libssl1.1
```
Затем используем команду для установки ключа **Mongo DB** и удалаем созданный файл списка фокусной безопасности:
```
sudo rm /etc/apt/sources.list.d/focal-security.list
```
После ключа мы получим такой вывод:
```
Output
OK
```
Затем запускаем команду, которая создаст файл **sources.list.d** в каталоге с именем **mongodb-org-4.4.list**:
```
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/4.4 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.4.list
```
После этого обновляем индекс пакетов сервера, чтобы **APT** знал, где найти пакет **mongodb-org**:
```
sudo apt update
```
Устанавливаем **MongoDB**:
```
sudo apt install mongodb-org
```
Запускаем службу **Mongo DB**:
```
sudo systemctl start mongod.service
```
Проверяем статус:
```
sudo systemctl status mongod
```
И получаем примерно такой ответ: 
```
● mongod.service - MongoDB Database Server
     Loaded: loaded (/lib/systemd/system/mongod.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2023-11-13 09:36:46 MSK; 2h 34min ago
       Docs: https://docs.mongodb.org/manual
   Main PID: 41330 (mongod)
     Memory: 163.1M
        CPU: 17.411s
     CGroup: /system.slice/mongod.service
             └─41330 /usr/bin/mongod --config /etc/mongod.conf

```
После этого можно включить запуск службы **Mongo DB** при загрузке системы: 
```
sudo systemctl enable mongod
```
Настраиваем безопасность.
Нам требуется добавить пользователя с правами администратора. Поскольку аутентификация отключена, мы можем подключиться к оболочке **Mongo** выполнив команду:
```
mongo
```
И да, служебную информацию мы узнаем также без всяких препятствий:
```
>show dbs
```
```
Output
admin 0.000GB
config 0.000GB
local 0.000GB
```
Чтобы устранить эту уязвимость подключаемся к базе данных **admin**:
```
>use admin
```
```
Output
switched to db admin
```
Далее используем методо ```db.createUser```:
```
db.createUser(
```
Дальнейшие действия состоят в формировании JSON-объекта, построчный ввод которого по итогу должен примерно выглядеть так:
```
> db.createUser(
... {
... user: "admin_db",
# после ввода имени пользователя необходимо написать: ...pwd: "<ваш_пароль>",
# это позволит не хранить пароль в текстовом виде
... pwd: passwordPrompt(),
... roles: [ { role: "userAdminAnyDatabase", db: "admin" }, "readWriteAnyDatabase" ]
... }
... )
```
Если синтаксис не будет иметь ошибок, нам будет предложено ввести пароль:
```
Output
Enter password:
```
Далее мы получим подтверждение, что пользователь был добавлен:
```
Output
Successfully added user: {
	"user" : "admin_db",
	"roles" : [
		{
			"role" : "userAdminAnyDatabase",
			"db" : "admin"
		},
		"readWriteAnyDatabase"
	]
}
```
Выходим:
```
exit
```
Активируем аутификацию.
Открываем конфигурационный файл:
```
sudo nano /etc/mongod.conf
```
Находим эту часть конфига:
```
# how the process runs
processManagement:
  timeZoneInfo: /usr/share/zoneinfo

#security:

#operationProfiling:

#replication:

```
Раскомментируем строку ```security:```, строкой ниже жмем два пробела и пишем ```authorization: "enabled"```. Должно получится так:
```
# how the process runs
processManagement:
  timeZoneInfo: /usr/share/zoneinfo

security:
  authorization: "enabled"

#operationProfiling:

#replication:
```
Перезапускаем демона, для вступления изменений в силу:
```
sudo systemctl restart mongod
```
Проверяем аутентификацию.
Открываем оболочку:
```
mongo
```
Просим показать нам базы данных:
```
>show dbs
```
Если все было сделано правильно, то **Mongo DB** будет партизанить и не давать никакой информации. Этого мы и добивались. 

Проверяем пользователя с правами администратора.
```
mongo -u admin_db -p --authenticationDatabase admin
```
Тут вводим пароль:
```
MongoDB shell version v4.4.25
Enter password:
```
Делаем запрос информации...
```
>show dbs
```
...и получаем ответ:
```
admin 0.000GB
config 0.000GB
local 0.000GB
```

- Установите любую SQL базу данных (на выбор: PostgreSQL, MySQL, MariaDB и др.) и убедитесь, что она корректно работает и защищена.

Выбираем **PostgreSQL**.

Устанавливаем СУБД на сервер:
```
sudo apt-get install postgresql postgresql-contrib
```
При устновке будет создан пользователь **postgres**, авторизуемся под данным пользователем:
```
sudo su - postgres
```
Затем запускаем сессию **PostgreSQL**:
```
psql
```
Все работает. Закрываем сессию и переключаемся на предыдущего пользователя:
```
exit
```
Блокируем удаленные подключения. Эта настройка задаётся по умолчанию при установке **PostgreSQL** из репозиториев Ubuntu. Но на всякий случай провериv, так ли это. Открываем следующий файл:
```
sudo cat /etc/postgresql/14/main/pg_hba.conf
```
Как мы видим, все подключения и репликации возможны лишь по локальной сети:
```
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            scram-sha-256
# IPv6 local connections:
host    all             all             ::1/128                 scram-sha-256
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     peer
host    replication     all             127.0.0.1/32            scram-sha-256
host    replication     all             ::1/128                 scram-sha-256
```
Блокировка доступа в **PostgreSQL** осуществляется через роли.
По умолчанию пользователь **postgres** имеет права суперпользователя.
Заходим в **postgres** если вышли:
```
sudo su - postgres
```
Открываем СУБД:
```
psql
```
Создадим две роли:
```
CREATE ROLE login_role WITH login;
CREATE ROLE access_role;
```
Убедимся что они появились: 
```
\du
```
```
                                      List of roles
  Role name  |                         Attributes                         |   Member of
-------------+------------------------------------------------------------+---------------
 access_role | Cannot login                                               | {}
 login_role  |                                                            | {access_role}
 postgres    | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
```
Можно создать роли для каждого пользователя (приложения) и тем самым обезопасить **PostgreSQL**.

Доступные права и роли можно посмотреть через команду:
```
\h CREATE ROLE
```
```
Command:     CREATE ROLE
Description: define a new database role
Syntax:
CREATE ROLE name [ [ WITH ] option [ ... ] ]

where option can be:

      SUPERUSER | NOSUPERUSER
    | CREATEDB | NOCREATEDB
    | CREATEROLE | NOCREATEROLE
    | INHERIT | NOINHERIT
    | LOGIN | NOLOGIN
    | REPLICATION | NOREPLICATION
    | BYPASSRLS | NOBYPASSRLS
    | CONNECTION LIMIT connlimit
    | [ ENCRYPTED ] PASSWORD 'password' | PASSWORD NULL
    | VALID UNTIL 'timestamp'
    | IN ROLE role_name [, ...]
    | IN GROUP role_name [, ...]
    | ROLE role_name [, ...]
    | ADMIN role_name [, ...]
    | USER role_name [, ...]
    | SYSID uid

URL: https://www.postgresql.org/docs/14/sql-createrole.html
```
# Безопасность
- Реализуйте базовые меры укрепления безопасности сервера (например отключение root-логина через SSH, настройка fail2ban)

Подключим **fail2ban**.

Для устновки выполним следующую команду:
```
sudo apt install fail2ban -y
```
После установки последовательно запускаем и включаем службу: 
```
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```
Проверка работоспособности...
```
sudo systemctl status fail2ban
```
... ответит нам таким сообщением:
```
root@cv3365285:/etc/fail2ban# sudo systemctl status fail2ban
● fail2ban.service - Fail2Ban Service
     Loaded: loaded (/lib/systemd/system/fail2ban.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2023-11-13 09:36:47 MSK; 4h 11min ago
       Docs: man:fail2ban(1)
   Main PID: 41492 (fail2ban-server)
      Tasks: 5 (limit: 9494)
     Memory: 12.3M
        CPU: 3.260s
     CGroup: /system.slice/fail2ban.service
             └─41492 /usr/bin/python3 /usr/bin/fail2ban-server -xf start

Nov 13 09:36:47 cv3365285.novalocal systemd[1]: Started Fail2Ban Service.
Nov 13 09:36:47 cv3365285.novalocal fail2ban-server[41492]: Server ready
```
Все параметры конфигурации хранятся в файле **jail.conf**. Если редактировать **jail.conf**, то при следующем обновлении системы содержимое файла может вернуться к состоянию по умолчанию. Поэтому мы сделаем копию оригинального **jail.conf** с новым именем **jail.local**:
```
cd /etc/fail2ban/
cp jail.conf jail.local
```
Открываем файл:
```
sudo nano jail.local
```
Мы видим, что файл конфигураций содержит множество параметров. Например, мы можем изменить значение параметров **findtime** и **maxretry** на следующие:
```
findtime = 5m
maxretry = 3
```
Таким образом, если в течении 5 минут будет произведено три провальных попытки входа, то пользователь будет заблокирован.

Давайте проверим это.

Создадим пользователя **new-user**:
```
sudo adduser new-user
```
После этого установим ему пароль, а остальные параметры (имя, номер телефона и т. д.) можно пропустить.

При желании его можно его в группу sudo, чтобы он имел права администратора (если необходимо):
```
sudo usermod -aG sudo new-user
```
В файле ```/etc/ssh/sshd_config``` находим строку ```#PasswordAuthentication yes``` и раскомментируем ее.

Перезапускаем службу **ssh**:
```
sudo systemctl restart ssh
```
Проверяем работает ли блокировка пользователя:
```
$ ssh new-user@79.174.94.83
new-user@79.174.94.83's password:
Permission denied, please try again.
new-user@79.174.94.83's password:
Permission denied, please try again.
new-user@79.174.94.83's password:
new-user@79.174.94.83: Permission denied (publickey,password).
```

## Что в итоге?
В итоге история терминала содержит ```873``` команды. Хотелось бы сказать, что это от продуктивности, но где-то не хватило знаний, где-то шла длительная борьба с ошибками. Несмотря на это, я знаю точно что постарался и теперь понимаю в каком направлении стоит углублять свои знания и повышать квалификацию. Тестовое мне понравилось. 