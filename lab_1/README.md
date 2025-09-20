# Лабораторная работа 1


## Шаг 1. Получение информации + некоторые раздумья

Начнем с того, что тема была для меня довольно неизвестная, а знания по ней поверхностные, поэтому, чтобы лучше понимать что происходит, а не просто такать запросами GPT, я посмотрела дополнительные видео.

В лабораторной работе необходимо сделать https-запрос, для этого необходимо иметь ssl-сертификат.(Все для безопастности)

Прочитав информацию в интернете, я узнала, что в моем случае можно ssl-сертификат можно получить двумя способами:

1) Сделать самоподписанные сертификаты(в этом случае будут некоторые НО, о которых  рассскажу позже)

2) Использовать бесплатные сертификаты с сайта Let's Encrypt(к сожалению, у меня почему-то не удалось реализовать таким способом)

Решением для этой лабы я решила сделать путь через самоподписанный сертификат.


### 1 Директории

Создаем 2 директории, одну будем использовать для ключей, другую - для сертификатов.


```
sudo mkdir -p /etc/ssl/private
sudo mkdir -p /etc/ssl/certs
```


### 2 Команда для самоподписанного сертификата

Создаем самоподписанные сертификаты(здесь помогла ии)

Для проекта 1:

```
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/ssl/private/project1.key \
    -out /etc/ssl/certs/project1.crt \
    -subj "/CN=project1.local/O=Project1/C=RU"
```

Для проекта 2:

```
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/ssl/private/project2.key \
    -out /etc/ssl/certs/project2.crt \
    -subj "/CN=project2.local/O=Project2/C=RU"
```

<details><summary><b>
(На будущее, если я забуду что это за нечто)</b></summary>

На примере запроса для 1 проекта:

`openssl req` - утилита для работы с ssl-запросами

(Она может создавать самоподписанные сертификаты для использования в качестве корневых центров сертификации)

`nodes` -  No DES - не шифровать приватный ключ паролем

`days 365` - срок действия сертификата

`newkey rsa:2048` - создать новый RSA ключ длиной 2048 бит (современный стандарт безопасности)

`keyout /etc/ssl/private/project1.key` - путь для сохранения приватного ключа

`out /etc/ssl/certs/project1.crt` - путь для сохранения сертификата

`subj "/CN=project1.local/O=Project1/C=RU"` - информация о владельце, где 

`CN=project1.local` - указываем доменное имя, для которого выдается сертифкат

`O=Project1` - название организации, для которой выдается сертификат(пусть будет просто Project1)

`C=RU` - соответственно, страна
</details>

## Шаг 2. Содержимое проектов


Я решила, для своего удобства, сделать два файла с приветствием, чтобы была возможность быстрой проверки корректности работы в терминале.
<details><summary><b>
Прежде чем начнем, важно задать следующий вопрос: "Какие папки и файлы есть у nginx?"</b></summary>


-В данной лабе стоит обратить внимание на 3 папки nginx

1 папка - там хранятся файлы сайта по твоему домену(/var/www/yourdomain.ru)

2 папка - лежит главный файл настройки nginx (/etc/nginx/nginx.conf)

3 папка - конфигурации для сайтов (/etc/nginx/sites-available)
</details>

### 1 Первый проект

```
sudo mkdir -p /var/www/project1/public
sudo nano /var/www/project1/public/index.html
```

```
<!DOCTYPE html>
<html>
<head>
    <title>Project 1</title>
</head>
<body>
    <h1>Добро пожаловать в Project 1</h1>
    <p>Цель проекта: натыкать ошибки, чтобы прокачать скилл</p>
</body>
</html>
```

### 2 Проект


```
sudo mkdir -p /var/www/project2/public
sudo nano /var/www/project2/public/index.html
```

```
<!DOCTYPE html>
<html>
<head>
    <title>Project 2</title>
</head>
<body>
    <h1>Добро пожаловать в Project 2</h1>
    <p>Ay, ay, ay
I'm your little butterfly
Green, black and blue
Make the colours in the sky
Ay, ay, ay, I'm your little butterfly
Green, black and blue
Make the colours in the sky</p>
</body>
</html>
```

Решила все не расписывать, но у нас еще будет папка `about`(с информацией о проекте 1), а также `contact` для контакной информации проект 2.


## Шаг 3. Настройка ввиртуальных хостов

### 1 Первый проект

```
sudo nano /etc/nginx/sites-available/project1.local
```

```
server {
    listen 80;
    server_name project1.local;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name project1.local;

    #Наши самоподписанные сертификаты
    ssl_certificate /etc/ssl/certs/nginx-selfsigned1.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned1.key;

    # Корневая директория
    root /var/www/project1/public;
    index index.html;


    # Обработка статических файлов
    location / {
        try_files $uri $uri/ =404;
    }

    # Пример использования alias для специального пути
    location /special {
        alias /var/www/project1/public/about;
        try_files $uri $uri/ /about/index.html;
    }
}
```

```sudo nano /etc/nginx/sites-available/project2.local```

```
server {
    listen 80;
    server_name project2.local;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name project2.local;

    ssl_certificate /etc/ssl/certs/nginx-selfsigned2.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned2.key;
    root /var/www/project2/public;
    index index.html; #Это главный html-файл
    # Обработка статических файлов
    location / {
        try_files $uri $uri/ =404;
    }


    location /contact {
        alias /var/www/project2/public/contact;
        try_files $uri $uri/ /contact/index.html;
    }
}
```

`listen 80` - ждем запросы пользователя на порту 80(виртуальный порт, их может быть оооочень много)

`server_name` - домен нашего сайта

первая секция сервер редиректик на вторую секцию сервер, с помощью оператора return(о редиректе говорит число 301)

порт `443` - большинство сайтов в интернете 




### Активация

Активируем виртуальные хосты

```
sudo ln -s /etc/nginx/sites-available/project1.local /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/project2.local /etc/nginx/sites-enabled/
```

Для того, чтобы не было возможных конфликта в конфигурациях, необходимо удалять дефолтный конфиг

```
sudo rm /etc/nginx/sites-enabled/default
```

Важно проверть, что с конфигурацией все окей 

```
$ sudo nginx -t
---------------------------------------------------------------
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```



Смотрим порты 

```
$ sudo netstat -tulpn | grep nginx
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      15789/nginx: master
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      15789/nginx: master
```


```$ curl -k https://project1.local
<!DOCTYPE html>
<html>
<head>
    <title>Project 1</title>
</head>
<body>
    <h1>Добро пожаловать в Project 1</h1>
    <p>Цель проекта: натыкать ошибки, чтобы прокачать скилл</p>
</body>
</html>
```


```
$ curl -k https://project2.local
<!DOCTYPE html>
<html>
<head>
    <title>Project 2</title>
</head>
<body>
    <h1>Добро пожаловать в Project 2</h1>
    <p>Ay, ay, ay
I'm your little butterfly
Green, black and blue
Make the colours in the sky
Ay, ay, ay, I'm your little butterfly
Green, black and blue
Make the colours in the sky</p>
</body>
</html>
```

```
$ curl -I http://project1.local
HTTP/1.1 301 Moved Permanently
Server: nginx/1.18.0 (Ubuntu)
Date: Sun, 07 Sep 2025 13:31:38 GMT
Content-Type: text/html
Content-Length: 178
Connection: keep-alive
Location: https://project1.local/
```
О чем это говорит, да это говрит нам о том, что все отлично, сервер принудительно и всегда переводит на https!!!

Протокол сменился на https


А теперь сделаем вот такую штуку

```
$ curl -I https://project1.local
curl: (60) SSL certificate problem: self-signed certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```

В начале я испугалась и подумала, что  опять наткнулась на какие-то грабли и что-то не поняла, НО(как раз к этому НО мы и должны были прийти) ЭТО НОРМАЛЬНО.

Почему?

Все просто, браузер не дурачок и он не доверяет самоподписанным сертификатам, поэтому выходит ошибка подлинности, по моему мнению, в данном случае было бы намного лучше использовать какие-то купленные и подвержденные сертификаты, либо бесплатные с Let's Encrypt.



<details><summary><b>Из интересного</b></summary>
В nginx может обрабатываться быть статический контент, для того, чтобы пользователь каждый раз не загружал картинки и т.д. при входе на сайт, или проще говоря, чтобы добиться цели быстрого открывания сайта, нужно сохранять информацию, которую уже открывал пользователь, показывать ее, не загрузжа повторно - Кеширование.

```
location ~* ^.+.(jpgljpeg|gif|png|css|js)${
    root /var/www/damain.ru
    expires 1d;
}

```

для кеширования добавим expires 1d, где 1d - будет храниться 1 день
```


## Вывод:

Данная лаба показалась интересной, очень круто узнать что-то новое, теперь стало более ясно как работают сайты и как они взаимодействуют с браузерами.

Поначалу, правда, было не совсем понятно как все реализовывать, пришлось много доп. инфы искать, чтобы лучше понять работу, однако мне очень помог код, который был показан на лекции, потому что как-то отложилась структура и некоторые команды стали привычными(мои ощущения).

Зато теперь я знаю как делать кеширование, умею получать ssl-сертификаты несколькими способами, принудительно перенапралять http-запросы на https.

