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
```
</details>

# Звездочка *
Для этой лабораторной выбрали сайт чая https://tea-24.ru/

## ffuf
Для работы с ffuf установила Golang и настроила переменные среды. Также командой 
```
git clone https://github.com/danielmiessler/SecLists.git
```
установила необходимы для работы словари, которые выглядят так (файл web-extensions.txt):
<img width="1044" height="1070" alt="image" src="https://github.com/user-attachments/assets/3a923d36-d787-4959-8832-b20715449907" />


### Ошибка конфигурации 1
Начала с проверкии расширений страницы:

```
C:\Users\Liudmila>ffuf -w "C:\Users\Liudmila\SecLists\Discovery\Web-Content\web-extensions.txt" -u "https://tea-24.ru/indexFUZZ" -ic

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : https://tea-24.ru/indexFUZZ
 :: Wordlist         : FUZZ: C:\Users\Liudmila\SecLists\Discovery\Web-Content\web-extensions.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

.cgi                    [Status: 403, Size: 199, Words: 14, Lines: 8, Duration: 46ms]
.pcap                   [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 112ms]::
.php                    [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 110ms]::
.jhtml                  [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 117ms]::
.htm                    [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 115ms]::
.log                    [Status: 403, Size: 199, Words: 14, Lines: 8, Duration: 89ms]
.php4                   [Status: 403, Size: 199, Words: 14, Lines: 8, Duration: 97ms]
.aspx                   [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 122ms]::
.asp                    [Status: 403, Size: 199, Words: 14, Lines: 8, Duration: 104ms]
.php5                   [Status: 403, Size: 199, Words: 14, Lines: 8, Duration: 102ms]
.php2                   [Status: 403, Size: 199, Words: 14, Lines: 8, Duration: 119ms]
.c                      [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 155ms]::
.cfm                    [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 119ms]::
.inc                    [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 129ms]::
.phtml                  [Status: 403, Size: 199, Words: 14, Lines: 8, Duration: 133ms]
.js                     [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 140ms]
.jsa                    [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 133ms]::
.shtml                  [Status: 403, Size: 199, Words: 14, Lines: 8, Duration: 150ms]
.phps                   [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 173ms]::
.html                   [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 176ms]::
.nsf                    [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 201ms]::
.com                    [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 207ms]::
.hta                    [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 217ms]::
.sql                    [Status: 403, Size: 199, Words: 14, Lines: 8, Duration: 182ms]
.php7                   [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 199ms]::
.pht                    [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 231ms]::
.pl                     [Status: 403, Size: 199, Words: 14, Lines: 8, Duration: 191ms]
.exe                    [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 234ms]::
.jsp                    [Status: 403, Size: 199, Words: 14, Lines: 8, Duration: 203ms]
.php6                   [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 219ms]::
.php3                   [Status: 403, Size: 199, Words: 14, Lines: 8, Duration: 206ms]
.rb                     [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 220ms]::
.reg                    [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 237ms]::
.mdb                    [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 256ms]::
.css                    [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 268ms]::
.dll                    [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 282ms]::
.sh                     [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 284ms]::
.phar                   [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 288ms]::
.bat                    [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 287ms]::
.json                   [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 297ms]::
.swf                    [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 292ms]::
.txt                    [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 270ms]::
.xml                    [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 283ms]::
:: Progress: [43/43] :: Job [1/1] :: 0 req/sec :: Duration: [0:00:00] :: Errors: 0 ::


```


Страница не вернула нигде код `200`, только `301` - редирект и `403` - запрещено (причем размер у таких ответов одинаковй, значит единая заглушка). Далее посмотрю, куда редиректит по .php:

```
curl -I https://tea-24.ru/index.php -> Location: /
```
перенаправляет на главную страницу
и по .html
```
curl -I https://tea-24.ru/index.html
HTTP/1.1 301 Moved Permanently
Server: nginx/1.14.1
Date: Sat, 11 Oct 2025 17:33:41 GMT
Content-Type: text/html; charset=UTF-8
Connection: keep-alive
X-Powered-By: PHP/7.3.33
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Status: 301 Moved Permanently
Set-Cookie: PHPSESSID=26a844a95d34185a4af5416acf3dce59; expires=Sat, 25-Oct-2025 17:33:41 GMT; Max-Age=1209600; path=/; HttpOnly
Location: /shophophophophophophophophophophophophophophophophophophophopndex.html
Strict-Transport-Security: max-age=31536000;
```
с очень интригующей Location, попробую ее изучить
```
C:\Users\Liudmila>curl -I "https://tea-24.ru/shophophophophophophophophophophophophophophophophophophophophopndex.html"
HTTP/1.1 301 Moved Permanently
Server: nginx/1.14.1
Date: Sat, 11 Oct 2025 17:39:35 GMT
Content-Type: text/html; charset=UTF-8
Connection: keep-alive
X-Powered-By: PHP/7.3.33
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Status: 301 Moved Permanently
Set-Cookie: PHPSESSID=3360e1bb021c46c982da0dd44bb1952d; expires=Sat, 25-Oct-2025 17:39:34 GMT; Max-Age=1209599; path=/; HttpOnly
Location: /shophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophopndex.html
Strict-Transport-Security: max-age=31536000;
```

```
curl -L -v "https://tea-24.ru/index.html"
...
< Location: /shophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophopndex.html
< Strict-Transport-Security: max-age=31536000;
* Ignoring the response-body
< * Connection #0 to host tea-24.ru left intact
* Issue another request to this URL: 'https://tea-24.ru/shophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophopndex.html'
* Re-using existing https: connection with host tea-24.ru
> GET /shophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophophopndex.html HTTP/1.1
...
```

Итого получаем: при каждой новой попытке создается новая сессия, редирект на ту же строку постоянно повторяется (бесконечный цикл). Но в конце сервер-таки вернул 
```
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>403 Forbidden</title>
</head><body>
<h1>Forbidden</h1>
<p>You don't have permission to access this resource.</p>
</body></html>
* Connection #0 to host tea-24.ru left intact
```
это означает, что сервер все же не допускает прямой доступ к бесконечному URL.
УРА! ОШИБКА КОНФИГУРАЦИИ НАЙДЕНА!

### Проверка конфигурации 2
Хотелось бы, конечно, попробовать проверить директории /admin и подобные, а также поделать POST запросы, НО эти действия будут иметь больший вес ответственности, т.к. первое - попытка достичь скрытую информацию и второе - возможное изменениие состояния. Поэтому постараюсь аккуратно.
Раз не хочется POST, то посмотрим, принимает ли его сервер
```
curl -X OPTIONS -i https://tea-24.ru/
```
по идее, должно было вывести Allow:, но возвращает html и сессию:
```
HTTP/1.1 200 Ok
Server: nginx/1.14.1
Date: Sat, 11 Oct 2025 19:55:19 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 63267
Connection: keep-alive
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Status: 200 Ok
X-Powered-By: PHP/7.3.33
X-Generated-By: UMI.CMS
X-CMS-Version: 22
X-XSS-Protection: 0
Set-Cookie: PHPSESSID=42ca8fd51197edb1272fe8db03d1a78e; expires=Sat, 25-Oct-2025 19:55:19 GMT; Max-Age=1209600; path=/; HttpOnly
Strict-Transport-Security: max-age=31536000;
```
Что здесь печально: `X-XSS-Protection: 0`. Этот заголовок отключает встроенный браузерный фильтр против вредоносного кода (нет остановки загрузки страниц при обнаружении XSS атаки). 
ПОТЕНЦИАЛЬНАЯ УГРОЗА!

### Проверка 3
Проверка phpAdmin - веб-интерфейса для управления базой данных.
```
curl -I -L --max-redirs 3 https://tea-24.ru/phpmyadmin
HTTP/1.1 301 Moved Permanently
Server: nginx/1.14.1
Date: Sat, 11 Oct 2025 20:22:48 GMT
Content-Type: text/html
Content-Length: 185
Location: https://tea-24.ru/phpmyadmin/
Connection: keep-alive
Strict-Transport-Security: max-age=31536000;

HTTP/1.1 200 OK
Server: nginx/1.14.1
Date: Sat, 11 Oct 2025 20:22:48 GMT
Content-Type: text/html; charset=utf-8
Connection: keep-alive
X-Powered-By: PHP/7.2.24
Set-Cookie: pma_lang_https=en; expires=Mon, 10-Nov-2025 20:22:48 GMT; Max-Age=2592000; path=/phpmyadmin/; SameSite=Strict; secure; HttpOnly
Set-Cookie: phpMyAdmin_https=77f3626f580184cb39f507308e78ce42; path=/phpmyadmin/; SameSite=Strict; secure; HttpOnly
X-ob_mode: 1
X-Frame-Options: DENY
Referrer-Policy: no-referrer
Content-Security-Policy: default-src 'self' ;script-src 'self' 'unsafe-inline' 'unsafe-eval' ;style-src 'self' 'unsafe-inline' ;img-src 'self' data:  *.tile.openstreetmap.org;object-src 'none';
X-Content-Security-Policy: default-src 'self' ;options inline-script eval-script;referrer no-referrer;img-src 'self' data:  *.tile.openstreetmap.org;object-src 'none';
X-WebKit-CSP: default-src 'self' ;script-src 'self'  'unsafe-inline' 'unsafe-eval';referrer no-referrer;style-src 'self' 'unsafe-inline' ;img-src 'self' data:  *.tile.openstreetmap.org;object-src 'none';
X-XSS-Protection: 1; mode=block
X-Content-Type-Options: nosniff
X-Permitted-Cross-Domain-Policies: none
X-Robots-Tag: noindex, nofollow
Expires: Sat, 11 Oct 2025 20:22:48 +0000
Cache-Control: no-store, no-cache, must-revalidate, pre-check=0, post-check=0, max-age=0
Pragma: no-cache
Last-Modified: Sat, 11 Oct 2025 20:22:48 +0000
Vary: Accept-Encoding
Strict-Transport-Security: max-age=31536000;
```
Из Интернета мимокрокодил может попасть на эту страницу. Из минусов: нет ограничения на попытки входа, нет двухфакторной аутентификации и видно весь стек разработки. Из плюсов: хорошая защита. 
Из конкретных уязвимостей:
- веб-сервер и его уязвимости доступны всем и каждому по ~~взмаху волшебной палочки~~ легкому запросу к нейросети или гуглу > с легкостью можно узнать уязвимости веб-сервера, особенно если это довольно старая версия (а здесь именно она!). за это сильный ай-ай-ай
- `X-Powered_By` тоже не должен быть известен, как и веб-сервер!! а тут еще и старая его версия, которая на последнем издыхании живет и, может, даже не поддерживается.

## Проверка 4
Сайт оказался довольно интересным в изучении, поэтому я решила продолжить добывать все новые не-баги-а-фичи
На очереди у нас запросы на несуществующие доменные имена
```
elizabethkulakova@Laptop-Elizabeth ~ % curl -I "https://tea-24.ru/несуществующая-страница-12345"
HTTP/2 301 
server: nginx/1.14.1
date: Sat, 11 Oct 2025 23:57:37 GMT
content-type: text/html; charset=UTF-8
x-powered-by: PHP/7.3.33
expires: Thu, 19 Nov 1981 08:52:00 GMT
cache-control: no-store, no-cache, must-revalidate
pragma: no-cache
status: 301 Moved Permanently
set-cookie: PHPSESSID=fa436e4d14aad72538c028a35f51c0fa; expires=Sat, 25-Oct-2025 23:57:37 GMT; Max-Age=1209600; path=/; HttpOnly
location: /shophophophophophophophophophophophophophophophophophophophopесуществующая-страница-12345
strict-transport-security: max-age=31536000;
```
Довольно сомнительно, что сервис не выдает ошибку 404, как должен это делать, а делает редирект. И очень смущает формирование пути: если все пути на сайте формируются таким образом, то это очень приятная работа для хакеров, которые захотят "восстановить" пароли от админских учетных записей или же просто попортить сайт. А еще по активировавшимся куки видно создание сессии, что приводит к бессмысленной трате ресурсов на неверные запросы и снова хоршей демонстрации паттерна формирования сессии.

Попробуем постучаться в конфигурационные файлы, часть из которых предложила нейросеть, и увидим следующую картину:
```
elizabethkulakova@Laptop-Elizabeth ~ % curl -I "https://tea-24.ru/nginx_status" HTTP/2 301 server: nginx/1.14.1 date: Sat, 11 Oct 2025 23:58:22 GMT content-type: text/html; charset=UTF-8 x-powered-by: PHP/7.3.33 expires: Thu, 19 Nov 1981 08:52:00 GMT cache-control: no-store, no-cache, must-revalidate pragma: no-cache status: 301 Moved Permanently set-cookie: PHPSESSID=a19f5a734afc48a1abd71b7895ad1443; expires=Sat, 25-Oct-2025 23:58:22 GMT; Max-Age=1209600; path=/; HttpOnly location: /shophophophophophophophophophophophophophophophophophophophopginx_status strict-transport-security: max-age=31536000;

elizabethkulakova@Laptop-Elizabeth ~ % curl -I "https://tea-24.ru/web.config" HTTP/2 301 server: nginx/1.14.1 date: Sat, 11 Oct 2025 23:59:13 GMT content-type: text/html; charset=UTF-8 x-powered-by: PHP/7.3.33 expires: Thu, 19 Nov 1981 08:52:00 GMT cache-control: no-store, no-cache, must-revalidate pragma: no-cache status: 301 Moved Permanently set-cookie: PHPSESSID=f571e92ac838aa7bfcdbfc89652a3c4a; expires=Sat, 25-Oct-2025 23:59:13 GMT; Max-Age=1209600; path=/; HttpOnly location: /shophophophophophophophophophophophophophophophophophophophopeb.config strict-transport-security: max-age=31536000;
```
Вероятно, никакие файлы nginx не должны быть доступны простым смертным и уводить пользователя в редирект вместо запрета на просмотр. И снова видим пример создания сессии тогда, когда это делать абсолютно не нужно, чтобы не тратить ресурсы и не рисковать безопасностью

## Проверка 5
К слову о безопасности. Есть в этом ответе что-то такое, что цепляет за душу
```
elizabethkulakova@Laptop-Elizabeth ~ % curl "https://tea-24.ru/sitemap.xml"
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9"><url>
  <loc>https://www.tea-24.ru/</loc>
  <lastmod>2022-09-15T15:22:58+03:00</lastmod>
  <priority>0.3</priority>
  <changefreq>weekly</changefreq>
</url><url>
  <loc>https://www.tea-24.ru/about/</loc>
  <lastmod>2021-12-22T12:20:18+03:00</lastmod>
  <priority>1</priority>
  <changefreq>weekly</changefreq>
</url><url>
  <loc>https://www.tea-24.ru/delivery_and_payment/</loc>
  <lastmod>2023-10-12T13:53:32+03:00</lastmod>
  <priority>1</priority>
  <changefreq>weekly</changefreq>
</url><url>
  <loc>http://dev.tea-24.ru/help/</loc>
  <lastmod>2021-12-12T08:22:53+03:00</lastmod>
  <priority>1</priority>
  <changefreq>weekly</changefreq>
</url><url>
  <loc>http://dev.tea-24.ru/remont_bytovoj_tehniki_na_domu/s_chego_nachat/</loc>
  <lastmod>2021-12-12T08:21:25+03:00</lastmod>
  <priority>0.5</priority>
  <changefreq>weekly</changefreq>
</url><url>
  <loc>https://www.tea-24.ru/pioneer_club/ya_vybral_m-150_chto_skazhete1/</loc>
  <lastmod>2025-10-10T09:53:55+03:00</lastmod>
  <priority>0.5</priority>
  <changefreq>weekly</changefreq>
</url><url>
  <loc>http://dev.tea-24.ru/super-podarki/whatsapp-image-2021-12-10-at-12-58-27-1/</loc>
  <lastmod>2021-12-13T20:46:45+03:00</lastmod>
  <priority>0.5</priority>
  <changefreq>weekly</changefreq>
</url><url>
  <loc>http://dev.tea-24.ru/super-podarki/whatsapp-image-2021-12-10-at-12-58-27-2/</loc>
  <lastmod>2021-12-13T20:46:49+03:00</lastmod>
  <priority>0.5</priority>
  <changefreq>weekly</changefreq>
</url><url>
  <loc>http://dev.tea-24.ru/super-podarki/whatsapp-image-2021-12-10-at-12-58-27/</loc>
  <lastmod>2021-12-13T20:47:08+03:00</lastmod>
  <priority>0.5</priority>
  <changefreq>weekly</changefreq>
...
```
А за душу цепляет вот такое непосредственное и честное наличие ссылок на девелоперскую версию прямо в карте сайта. А как мы знаем по dev.my.itmo.su, версия разработчиков может содержать много интересных штучек в открытом доступе, о которых простым любителям согревающих напитков знать совершенно не нужно :) Это может повлечь за собой утечку данных, кражу идей (а код писать не так-то и просто) и все сопутсвующие прелести.

Ну и также отмечу вот такой момент:
```
elizabethkulakova@Laptop-Elizabeth ~ % curl "https://tea-24.ru/robots.txt"
User-Agent: Googlebot
Disallow: /?
Disallow: /notfound/
Disallow: /admin
Disallow: /index.php
Disallow: /emarket/addToCompare
Disallow: /emarket/basket
Disallow: /emarket/gateway
Disallow: /go-out.php
Disallow: /cron.php
Disallow: /filemonitor.php
Disallow: /search
Disallow: /captcha.php
Disallow: /counter.php
Disallow: /license_restore.php
Disallow: /packer.php
Disallow: /save_domain_keycode.php
Disallow: /session.php
Disallow: /standalone.php
Disallow: /static_banner.php
Disallow: /updater.php
Disallow: /users/login_do
Disallow: /autothumbs.php
Disallow: /~/
Disallow: /install.php
Disallow: /installer.php

User-Agent: Yandex
Disallow: /?
Disallow: /notfound/
Disallow: /admin
Disallow: /index.php
Disallow: /emarket/addToCompare
Disallow: /emarket/basket
Disallow: /emarket/gateway
Disallow: /go-out.php
Disallow: /cron.php
Disallow: /filemonitor.php
Disallow: /search
Disallow: /captcha.php
Disallow: /counter.php
Disallow: /license_restore.php
Disallow: /packer.php
Disallow: /save_domain_keycode.php
Disallow: /session.php
Disallow: /standalone.php
Disallow: /static_banner.php
Disallow: /updater.php
Disallow: /users/login_do
Disallow: /autothumbs.php
Disallow: /~/
Disallow: /install.php
Disallow: /installer.php

User-Agent: *
Disallow: /?
Disallow: /notfound/
Disallow: /admin
Disallow: /index.php
Disallow: /emarket/addToCompare
Disallow: /emarket/basket
Disallow: /emarket/gateway
Disallow: /go-out.php
Disallow: /cron.php
Disallow: /filemonitor.php
Disallow: /search
Disallow: /captcha.php
Disallow: /counter.php
Disallow: /license_restore.php
Disallow: /packer.php
Disallow: /save_domain_keycode.php
Disallow: /session.php
Disallow: /standalone.php
Disallow: /static_banner.php
Disallow: /updater.php
Disallow: /users/login_do
Disallow: /autothumbs.php
Disallow: /~/
Disallow: /install.php
Disallow: /installer.php

Host: https://tea-24.ru
Sitemap: https://tea-24.ru/sitemap.xml
Sitemap: https://tea-24.ru/sitemap-images.xml

Crawl-delay: 3
```
Здесь отчетливо видно, что пути, которые должны быть в приватном доступе, открыты нараспашку, как русская душа. Как я поняла, все файлы, связанные с установкой, обновлением и хранением старых версий, должны быть под строгим контролем и удаляться сразу же после применения во избежание всякого рода перезагрузок от доброжелателей. Ну и всякие служебные пути вроде `admin` тоже не должны быть доступны, так как перейти по ним вполне себе можно, но очень сильно не нужно
<img width="1440" height="900" alt="Screenshot 2025-10-12 at 03 49 54" src="https://github.com/user-attachments/assets/8f100b58-ff6c-4fdf-bfeb-7fa930e49eff" />

## **Вывод по звездной части лабораторной:** если хотим быть бизнесвуменами и бизнесменаями, стоит не жалеть денег, сил и времени на разработку качественного и надежного сервиса. А то какие-нибудь безобидно выглядищие Катя, Люда и Лиза могут незаметно для себя увлечься и...
## На самом деле дествительно было интересно выполнять обе части! Да, были некоторые затыки и проблемы на этом пути воина, но тем было интереснее и познавательнее) Как минимум уже начинает складываться какое-никакое понимание того, как должны обстоять дела за пределами кода, что очень прокачивает нас как специалистов и позволяет открывать для себя новые грани мира ИТ!
