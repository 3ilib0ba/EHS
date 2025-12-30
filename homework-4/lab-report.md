# WEB. High Performance WEB и CDN

---

## Задание 1. Создание Server Pool для backend

* Создайте у себя в home dir 2 папки ~/backend1 и ~/backend2, в каждой из которых должен лежать 1 index.html
  * "\<h1>Response from Backend Server 1\</h1>" для backend1
  * "\<h2>*** Response from Backend Server 2 ***\</h2>"  для backend2

* Внутри этих папок запустите питоновый http сервер на портах 8081 и 8082 соответственно.

---

Создаю нужные файлы `index.html` и запускаю серверы на портах 8081 и 8082:

```
user@operation-highload-systems:~$ cat backend1/index.html 
"<h1>Response from Backedn Server 1</h1>"

user@operation-highload-systems:~$ cat backend2/index.html 
"<h2>*** Response from Backend Server 2 ***</h2>"
```

Сервер на 8081:
```
user@operation-highload-systems:~/backend1$ python3 -m http.server 8081
Serving HTTP on 0.0.0.0 port 8081 (http://0.0.0.0:8081/) ...
```

Сервер на 8082:
```
user@operation-highload-systems:~/backend2$ python3 -m http.server 8082
Serving HTTP on 0.0.0.0 port 8082 (http://0.0.0.0:8082/) ...
```

Из другого терминала обратимся к серверам и заметим, что в ответах есть написанные в index.html строки:
```
user@operation-highload-systems:~$ curl -v http://localhost:8081 
*   Trying 127.0.0.1:8081...
* Connected to localhost (127.0.0.1) port 8081 (#0)
> GET / HTTP/1.1
> Host: localhost:8081
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
* HTTP 1.0, assume close after body
< HTTP/1.0 200 OK
< Server: SimpleHTTP/0.6 Python/3.10.12
< Date: Mon, 29 Dec 2025 23:52:13 GMT
< Content-type: text/html
< Content-Length: 42
< Last-Modified: Mon, 29 Dec 2025 23:48:03 GMT
< 
"<h1>Response from Backedn Server 1</h1>"
* Closing connection 0


user@operation-highload-systems:~$ curl -v http://localhost:8082
*   Trying 127.0.0.1:8082...
* Connected to localhost (127.0.0.1) port 8082 (#0)
> GET / HTTP/1.1
> Host: localhost:8082
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
* HTTP 1.0, assume close after body
< HTTP/1.0 200 OK
< Server: SimpleHTTP/0.6 Python/3.10.12
< Date: Mon, 29 Dec 2025 23:52:18 GMT
< Content-type: text/html
< Content-Length: 50
< Last-Modified: Mon, 29 Dec 2025 23:48:42 GMT
< 
"<h2>*** Response from Backend Server 2 ***</h2>"
* Closing connection 0
```

---

## Задание 2. DNS Load Balancing с помощью dnsmasq

* При помощи dnsmasq создайте 2 A записи
  * my-awesome-highload-app.local,127.0.0.1
  * my-awesome-highload-app.local,127.0.0.2
* Запустите dnsmasq и при помощи dig обратитесь к 127.0.0.1 для резолва my-awesome-highload-app.local
* Проанализируйте вывод, что произойдет с DNS записями если backend2 сервер сломается?

---

Создаю конфигурацию для `dnsmasq` и проверяю, что через `dig` выдаются 127.0.0.1 и 127.0.0.2 адреса:

```
user@operation-highload-systems:~$ cat /etc/dnsmasq.d/highload-test.conf 
address=/my-awesome-highload-app.local/127.0.0.1
address=/my-awesome-highload.app.local/127.0.0.2

user@operation-highload-systems:~$ dig @127.0.0.1 my-awesome-highload-app.local A +short
127.0.0.2
127.0.0.1
```

Если backend2 сервер сломается, DNS об этом не узнает и продолжит отдавать обе A-записи, потенциально чередуя их. 
Часть запросов пойдёт на сломанный сервер.

---
