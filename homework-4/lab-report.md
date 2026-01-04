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

## Задание 3. Балансировка Layer 4 с  с помощью IPVS

* Создайте dummy1 интерфейс с адресом 192.168.100.1/32
* Используя ipvsadm создайте VS для TCP порта 80 ведущего в 127.0.0.1:8081 и 127.0.0.1:8082 использующего round-robin тип балансировки.
  * Используя curl сходите в http://192.168.100.1 продемонстрируйте счетчики на ipvs, убедитесь, что балансировка происходит.

---

Создаю dummy1 интерфейс:

```
user@operation-highload-systems:~$ sudo ip link add dummy1 type dummy

user@operation-highload-systems:~$ sudo ip addr add 192.168.100.1/32 dev dummy1

user@operation-highload-systems:~$ sudo ip link set dummy1 up

user@operation-highload-systems:~$ ip addr show dev dummy1
3: dummy1: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
link/ether 62:5c:99:39:16:14 brd ff:ff:ff:ff:ff:ff
inet 192.168.100.1/32 scope global dummy1
valid_lft forever preferred_lft forever
inet6 fe80::605c:99ff:fe39:1614/64 scope link
valid_lft forever preferred_lft forever
```

---

Создан VS для TCP-порта 80 и назначен round-robin:

```
user@operation-highload-systems:~$ sudo modprobe ip_vs ip_vs_rr nf_conntrack

user@operation-highload-systems:~$ sudo ipvsadm --clear

user@operation-highload-systems:~$ sudo ipvsadm -a -t 192.168.100.1:80 -r 127.0.0.1:8081 -m

user@operation-highload-systems:~$ sudo ipvsadm -a -t 192.168.100.1:80 -r 127.0.0.1:8082 -m

user@operation-highload-systems:~$ sudo ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.100.1:80 rr
  -> 127.0.0.1:8081               Masq    1      0          0         
  -> 127.0.0.1:8082               Masq    1      0          0
```

---

Проверка балансировки:

```
user@operation-highload-systems:~$ for i in {1..10}; do curl -s http://192.168.100.1/ | grep -E "Response"; done
"<h2>***Response from Backend Server 2***</h2>"
"<h1>Response from Backedn Server 1</h1>"
"<h2>***Response from Backend Server 2***</h2>"
"<h1>Response from Backedn Server 1</h1>"
"<h2>***Response from Backend Server 2***</h2>"
"<h1>Response from Backedn Server 1</h1>"
"<h2>***Response from Backend Server 2***</h2>"
"<h1>Response from Backedn Server 1</h1>"
"<h2>***Response from Backend Server 2***</h2>"
"<h1>Response from Backedn Server 1</h1>"
user@operation-highload-systems:~$ sudo ipvsadm -Ln --stats
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port               Conns   InPkts  OutPkts  InBytes OutBytes
  -> RemoteAddress:Port
TCP  192.168.100.1:80                   10       63       59     4126     5468
  -> 127.0.0.1:8081                      5       31       30     2037     2740
  -> 127.0.0.1:8082                      5       32       29     2089     2728
```

Видно, что запросы распределяются равномерно, а значит все корректно

---
