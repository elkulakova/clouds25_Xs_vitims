начнем с создания самоподписанных сертификатов `ssl` (господи храни хабр, я не знаю, как мир жил до создания такого прекрасного места и ему подобных по типу reddit и stackoverflow…)

создаем папку для сертификатов (можно и обычную, а не в системных файлах, которые просто так не доступны, но я что-то начала так, так что продолжу тоже так):
```
sudo mkdir /opt/homebrew/etc/nginx/ssl/ca/
cd /opt/homebrew/etc/nginx/ssl/ca/
```
создаем корневой сертификат для наших проектов: ПРО КЛЮЧИ ПОЯСНИТЬ
```
sudo openssl genrsa -out rootCA.key 2048
sudo openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 1024 -out rootCA.pem
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:RU    
State or Province Name (full name) [Some-State]:Saint Petersburg
Locality Name (eg, city) []:Saint Petersburg
Organization Name (eg, company) [Internet Widgits Pty Ltd]:ITMO haha
Organizational Unit Name (eg, section) []:FoAI
Common Name (e.g. server FQDN or YOUR name) []:ITMO DevOps Lab1 CA       
Email Address []:      
```

```
sudo ls -la /opt/homebrew/etc/nginx/ssl/ca/
total 16
drwxr-xr-x  4 root  wheel   128 Oct 11 11:44 .
drwxr-xr-x  3 root  wheel    96 Oct 11 11:37 ..
-rw-------  1 root  wheel  1704 Oct 11 11:39 rootCA.key
-rw-r--r--  1 root  wheel  1419 Oct 11 11:44 rootCA.pem
```
видим, что корневой сертификат создать получилось, ура!
теперь надо создать сертификаты для каждого домена:
* ca/ - только для корневого центра сертификации
* доменные сертификаты - создаются на одном уровне с папкой ca/
```
cd ..
sudo openssl genrsa -out aproj.local.key 2048
sudo openssl req -new -key aproj.local.key -subj "/C=RU/ST=Saint Petersburg/L=Saint Petersburg/O=ITMO haha/CN=aproj.local" -out aproj.local.csr
```

тут уже важно, чтобы `common name` совпадало с названием проекта! остальные параметры можно ввести как у корневого сертификата
подписываем сертификат:
```
sudo openssl x509 -req -in aproj.local.csr -CA ca/rootCA.pem -CAkey ca/rootCA.key -CAcreateserial -out aproj.local.crt -days 365 -sha256
Certificate request self-signature ok
subject=C=RU, ST=Saint Petersburg, L=Saint Petersburg, O=ITMO haha, CN=aproj.local
```

после подписания сертификата в папке /ca у нас автоматически появляется serial-файл `rootCA.srl`
```
sudo ls -al ./ca
total 24
drwxr-xr-x  5 root  wheel   160 Oct 11 12:06 .
drwxr-xr-x  6 root  wheel   192 Oct 11 12:06 ..
-rw-------  1 root  wheel  1704 Oct 11 11:39 rootCA.key
-rw-r--r--  1 root  wheel  1419 Oct 11 11:44 rootCA.pem
-rw-r--r--  1 root  wheel    41 Oct 11 12:06 rootCA.srl
```

точно такую же процедуру проделываем и для bproj:
```
sudo openssl genrsa -out bproj.local.key 2048
sudo openssl req -new -key bproj.local.key -subj "/C=RU/ST=Saint Petersburg/L=Saint Petersburg/O=ITMO haha/CN=bproj.local" -out bproj.local.csr
sudo openssl x509 -req -in bproj.local.csr -CA ca/rootCA.pem -CAkey ca/rootCA.key -CAcreateserial -out bproj.local.crt -days 365 -sha256
Certificate request self-signature ok
subject=C=RU, ST=Saint Petersburg, L=Saint Petersburg, O=ITMO haha, CN=bproj.local
```

теперь у нас есть вот такая структура папки:
```
elizabethkulakova@Laptop-Elizabeth ssl % sudo ls -al
total 48
drwxr-xr-x  9 root  wheel   288 Oct 11 12:10 .
drwxr-xr-x  3 root  wheel    96 Oct 11 11:30 ..
-rw-r--r--  1 root  wheel  1363 Oct 11 12:06 aproj.local.crt
-rw-r--r--  1 root  wheel  1013 Oct 11 12:02 aproj.local.csr
-rw-------  1 root  wheel  1704 Oct 11 12:00 aproj.local.key
-rw-r--r--  1 root  wheel  1363 Oct 11 12:10 bproj.local.crt
-rw-r--r--  1 root  wheel  1013 Oct 11 12:10 bproj.local.csr
-rw-------  1 root  wheel  1704 Oct 11 12:10 bproj.local.key
drwxr-xr-x  5 root  wheel   160 Oct 11 12:06 ca
```


**установка nginx с помощью команды**
```
brew install nginx
```
конфигурационный файл находится вот по такому пути: `/opt/homebrew/etc/nginx/nginx.conf`. топаем туда, чтобы внести изменения и настроить его
```
cd /opt/homebrew/etc/nginx
vim nginx.conf
```

вносим такие изменения:
```
#user nobody;
worker_processes 1; -> worker_processes auto; (ставит количество ядер процессора)

в server:
listen 8080 -> 80 default_server;
+ listen [::]:80 default_server; (for IPv6-addresses)
servername localhost -> _; (невалидное значение-заглушка, которое никогда не совпадет с реальным именем хоста из запроса. Оно используется именно в блоках по умолчанию (default_server), чтобы явно показать, что этот блок предназначен для обработки любых запросов, для которых не найдено другого, более подходящего блока. По сути, эта строка говорит: "Этот блок — ловушка для всего, что не было поймано другими правилами")
+ return 301 https://$host$request_uri; (301 moved permanently, redirects any http request to https)
```

создаем папку для содержимого сайтов
```
sudo mkdir -p /usr/local/var/www/
sudo mkdir /usr/local/var/www/aproj/
sudo mkdir /usr/local/var/www/bproj/
```
и в эти папки положим базовые файлики index.html следующего содержания
aproj/index.html:
```
<center>
  <p>
      I am aproj!! lorem ipsum, hello world!
  </p> 
  <p>just filling empty space.</p>
</center>
```

bproj/index.html:
```
<center>
  <p>
      I am bproj!! lorem ipsum, hello world!
  </p> 
  <p>just filling empty space.</p>
</center>
```

выдаем права не руту, чтобы использовать ключи сертификатов и наше содержимое сайтов и мочь запускать вообще сервак:
```
sudo chown -R elizabethkulakova /usr/local/var 
sudo chown -R elizabethkulakova /opt/homebrew/etc/nginx/ssl (чтобы на все сертификаты при использовании права были)
```

в файле etc/hosts прописываем хосты
```
cat /etc/hosts
##
# Host Database
#
# localhost is used to configure the loopback interface
# when the system is booting.  Do not change this entry.
##
127.0.0.1	localhost
255.255.255.255	broadcasthost
::1             localhost

127.0.0.1       aproj.local bproj.local **# вооот эта строка!**
```

сквозь мучения с редиректами, которые заняли часа 2, а то и больше… (а еще отец-девопс инженер тоже почти сломался)
создаем такой файл (внизу хосты прописали):
```
# user  _www;
worker_processes  auto;

error_log  /opt/homebrew/etc/nginx/logs/error.log error;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

pid        /opt/homebrew/etc/nginx/logs/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen 80 default_server;
	listen [::]:80 default_server;
        server_name _;
	return 301 https://$host$request_uri;
        #charset koi8-r;

        access_log  /opt/homebrew/etc/nginx/logs/host.access.log;

        location / {
            root   html;
            index  index.html index.htm;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }


    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}


    # HTTPS server
    
    server {
        listen       443 ssl default_server;
        server_name  _;

        ssl_certificate      ssl/ca/rootCA.pem;
        ssl_certificate_key  ssl/ca/rootCA.key;

        ssl_session_cache    shared:SSL:1m;
        ssl_session_timeout  5m;

        ssl_ciphers  HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers  on;

        location / {
            root   html;
            index  index.html index.htm;
        }
    }

    server {
        access_log  /opt/homebrew/etc/nginx/logs/aproj_access.log;
        error_log  /opt/homebrew/etc/nginx/logs/aproj_error.log error;

        listen 80;
        server_name  aproj.local;
        return 301 https://$host$request_uri;
    }

    server {
        access_log  /opt/homebrew/etc/nginx/logs/bproj_access.log;
        error_log  /opt/homebrew/etc/nginx/logs/bproj_error.log error;

        listen 80;
        server_name  bproj.local;
        return 301 https://$host$request_uri;
    }


    server {
        access_log  /opt/homebrew/etc/nginx/logs/aproj_access.log;
        error_log  /opt/homebrew/etc/nginx/logs/aproj_error.log error;

        listen 80;
        listen       443 ssl;
        server_name  aproj.local;

        ssl_certificate      ssl/aproj.local.crt;
        ssl_certificate_key  ssl/aproj.local.key;

        ssl_session_cache    shared:SSL:1m;
        ssl_session_timeout  5m;

        ssl_ciphers  HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers  on;

        root   /usr/local/var/www/aproj;

        location / {
            try_files $uri $uri/ =404;
        }

        location ^~ /images {
            alias   /usr/local/var/www/aliasa;
            try_files $uri $uri/ =404;
        }
    }

    server {
        access_log  /opt/homebrew/etc/nginx/logs/bproj_access.log;
        error_log  /opt/homebrew/etc/nginx/logs/bproj_error.log error;

        listen 80;
        listen       443 ssl;
        server_name  bproj.local;

        ssl_certificate      ssl/bproj.local.crt;
        ssl_certificate_key  ssl/bproj.local.key;

        ssl_session_cache    shared:SSL:1m;
        ssl_session_timeout  5m;

        ssl_ciphers  HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers  on;

        root   /usr/local/var/www/bproj;

        location / {
            try_files $uri $uri/ =404;
        }

        location ^~ /images {
            alias   /usr/local/var/www/aliasb;
            try_files $uri $uri/ =404;
        }
    }


    include servers/*;
}
```

также если создать папки
```
sudo mkdir /usr/local/var/www/aproj/cat
sudo mkdir /usr/local/var/www/bproj/dog
sudo mkdir /usr/local/var/www/aliasa
sudo mkdir /usr/local/var/www/aliasb
```
и в этих папках прикрепить файлы, то при обращении `http://aproj.local/cat/index.html`, например, также будет отображено содержимое файла `index.html` из папки `cat`
в папки для алиасов помещаем смешные картиночки гуся и Гуменника (да, мы на таком уровне каламбуров, но с большой любовью!) и теперь по обращению `http://aproj.local/images/gumennik.jpg` у нас выводится картиночка с красивым мальчишкой, которая хранится не в руте, а в `/usr/local/var/www/aliasa`!
