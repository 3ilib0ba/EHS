# Сетевой стек

---

## Задание 1. Анализ состояний TCP-соединений

* Запустите Python HTTP сервер на порту 8080:
  * python3 -m http.server 8080

* Проверяйте слушающие TCP-сокеты с помощью утилиты ss. найдите сокет с вашим http сервером.

* Подключитесь к серверу через curl.

* Проанализируйте состояние TCP-сокетов для порта 8080, объясните, почему есть сокет в состоянии TIME-WAIT, его роль и почему его нельзя удалить.

* Опишите, к каким проблемам может привести большое количество TIME-WAIT сокетов.

---

Поднимаю в одном терминале сервер и получаю в нем запрос:

```
user@operation-highload-systems:~$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...

----- [[ПОСЛЕ ТОГО КАК ОТПРАВИЛИ ЗАПРОС]] -----
127.0.0.1 - - [29/Dec/2025 12:32:25] "GET / HTTP/1.1" 200 -
```

В другом терминале выполняю проверку всех прослушивающих портов и нахожу только что запущенный, проверяю его тестовым запросом:

```
user@operation-highload-systems:~$ ss -ltnp | grep ':8080'
LISTEN 0      5            0.0.0.0:8080      0.0.0.0:*    users:(("python3",pid=39455,fd=3))
user@operation-highload-systems:~$ curl -v http://localhost:8080
*   Trying 127.0.0.1:8080...
* Connected to localhost (127.0.0.1) port 8080 (#0)
> GET / HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
* HTTP 1.0, assume close after body
< HTTP/1.0 200 OK
< Server: SimpleHTTP/0.6 Python/3.10.12
< Date: Mon, 29 Dec 2025 09:32:25 GMT
< Content-type: text/html; charset=iso8859-1
< Content-Length: 1306

...
...
...

* Closing connection 0
```

---

Найдём состояния по всем TCP сокетам с помощью команды `ss -tan`:

```
user@operation-highload-systems:~$ ss -tan
State       Recv-Q   Send-Q     Local Address:Port        Peer Address:Port   Process  
LISTEN      0        5                0.0.0.0:8080             0.0.0.0:*               
...      
TIME-WAIT   0        0              127.0.0.1:8080           127.0.0.1:35662           
...
```

Отсюда видно, что сервер продолжает слушать порт, а также ранее было соединение, которое сейчас в состоянии 
`TIME-WAIT`, что означает состояние перед закрытием соединения (начатое самим питон-сервером), чтобы не слушать запоздалые
пакеты по соединению, если такие будут.

Так как `TIME-WAIT` занимает случайный порт, то их большое кол-во может привести к тому, что все порты будут заняты, сервер
будет больше нагружен и для обработки новых соединений будет требоваться больше времени.

---

## Задание 2. Динамическая маршрутизация с BIRD

* Создайте dummy-интерфейс с адресом 192.168.14.88/32, назовите его service_0.
* При помощи BIRD проаннонсируйте этот адрес при помощи протокола RIP v2 включенного на вашем интерфейсе (eth0/ens33), а так же любой другой будущий адрес из подсети 192.168.14.0/24 но только если у него будет маска подсети /32 и имя будет начинаться на service_
* Создайте ещё три интерфейса:
  * service_1 192.168.14.1/30
  * service_2 192.168.10.4/32
  * srv_1 192.168.14.4/32
* С помощью tcpdump докажите, что анонсируются только нужные адреса, без лишних.

---

Создаю сетевые интерфейсы:

```
user@operation-highload-systems:~$ sudo ip link add service_0 type dummy
user@operation-highload-systems:~$ sudo ip addr add 192.168.14.88/32 dev service_0
user@operation-highload-systems:~$ sudo ip link set service_0 up

user@operation-highload-systems:~$ sudo ip link add service_1 type dummy
user@operation-highload-systems:~$ sudo ip addr add 192.168.14.1/30 dev service_1
user@operation-highload-systems:~$ sudo ip link set service_1 up

user@operation-highload-systems:~$ sudo ip link add service_2 type dummy
user@operation-highload-systems:~$ sudo ip addr add 192.168.10.4/32 dev service_2
user@operation-highload-systems:~$ sudo ip link set service_2 up

user@operation-highload-systems:~$ sudo ip link add srv_1 type dummy
user@operation-highload-systems:~$ sudo ip addr add 192.168.14.4/32 dev srv_1
user@operation-highload-systems:~$ sudo ip link set srv_1 up

user@operation-highload-systems:~$ ip addr show | grep -A2 "service_\|srv_"
3: service_0: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 0a:da:eb:a2:6e:aa brd ff:ff:ff:ff:ff:ff
    inet 192.168.14.88/32 scope global service_0
       valid_lft forever preferred_lft forever
    inet6 fe80::8da:ebff:fea2:6eaa/64 scope link 
--
4: service_1: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 9e:79:fe:e2:3d:75 brd ff:ff:ff:ff:ff:ff
    inet 192.168.14.1/30 scope global service_1
       valid_lft forever preferred_lft forever
    inet6 fe80::9c79:feff:fee2:3d75/64 scope link 
--
5: service_2: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 5e:9a:8a:21:74:12 brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.4/32 scope global service_2
       valid_lft forever preferred_lft forever
    inet6 fe80::5c9a:8aff:fe21:7412/64 scope link 
--
6: srv_1: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether ce:4a:f1:e3:64:c2 brd ff:ff:ff:ff:ff:ff
    inet 192.168.14.4/32 scope global srv_1
       valid_lft forever preferred_lft forever
    inet6 fe80::cc4a:f1ff:fee3:64c2/64 scope link
```

---

Конфигурация `/etc/bird.conf`:

```
user@operation-highload-systems:~$ sudo cat /etc/bird/bird.conf
router id 192.168.14.88;

protocol device {
    scan time 10;
}

protocol direct {
    interface "service_*";
}

protocol kernel {
    persist;
    scan time 20;
    ipv4 {
        import none;
        export all;
    };
}

protocol rip {
    interface "enp0s3" {
        version 2;
        metric 1;
        authentication none;
    };
    
    ipv4 {
        import all;
        export filter {
            if (source = RTS_DEVICE) && 
               (ifname ~ "service_") && 
               (net ~ 192.168.14.0/24) &&
               (net.len = 32) then accept;
            else reject;
        };
    };
};
```

---

Проверяем, что анонсируется только service_0 с помощью `tcpdump`:

```
user@operation-highload-systems:~$ sudo tcpdump -vv -s0 -n -i enp0s3 udp port 520
tcpdump: listening on enp0s3, link-type EN10MB (Ethernet), snapshot length 262144 bytes
02:08:11.402934 IP (tos 0xc0, ttl 1, id 12258, offset 0, flags [none], proto UDP (17), length 52)
    10.0.2.15.520 > 224.0.0.9.520: [bad udp cksum 0xec49 -> 0x3e80!] 
 RIPv2, Response, length: 24, routes: 1 or less
   AFI IPv4,   192.168.14.88/32, tag 0x0000, metric: 1, next-hop: self
 0x0000:  0202 0000 0002 0000 c0a8 0e58 ffff ffff
 0x0010:  0000 0000 0000 0001
02:08:41.439327 IP (tos 0xc0, ttl 1, id 26468, offset 0, flags [none], proto UDP (17), length 52)
    10.0.2.15.520 > 224.0.0.9.520: [bad udp cksum 0xec49 -> 0x3e80!] 
 RIPv2, Response, length: 24, routes: 1 or less
   AFI IPv4,   192.168.14.88/32, tag 0x0000, metric: 1, next-hop: self
 0x0000:  0202 0000 0002 0000 c0a8 0e58 ffff ffff
 0x0010:  0000 0000 0000 0001
02:09:11.402754 IP (tos 0xc0, ttl 1, id 30746, offset 0, flags [none], proto UDP (17), length 52)
    10.0.2.15.520 > 224.0.0.9.520: [bad udp cksum 0xec49 -> 0x3e80!] 
 RIPv2, Response, length: 24, routes: 1 or less
   AFI IPv4,   192.168.14.88/32, tag 0x0000, metric: 1, next-hop: self
 0x0000:  0202 0000 0002 0000 c0a8 0e58 ffff ffff
 0x0010:  0000 0000 0000 0001
02:09:41.434005 IP (tos 0xc0, ttl 1, id 49973, offset 0, flags [none], proto UDP (17), length 52)
    10.0.2.15.520 > 224.0.0.9.520: [bad udp cksum 0xec49 -> 0x3e80!] 
 RIPv2, Response, length: 24, routes: 1 or less
   AFI IPv4,   192.168.14.88/32, tag 0x0000, metric: 1, next-hop: self
 0x0000:  0202 0000 0002 0000 c0a8 0e58 ffff ffff
 0x0010:  0000 0000 0000 0001
^C
4 packets captured
4 packets received by filter
0 packets dropped by kernel
```

---

## Задание 3. Настройка фаервола/ Host Firewalling

* С помощью iptables или nftables создайте правило, запрещающее подключения к порту 8080.
* Запустите веб сервер на питоне и продемонстрируйте работу вашего firewall при помощи tcpdump.

---

Настройка `firewall` и запуск сервера:

```
user@operation-highload-systems:~$ sudo iptables -A INPUT -p tcp --dport 8080 -j DROP

user@operation-highload-systems:~$ sudo iptables -L -n -v | grep 8080
    0     0 DROP       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:8080

user@operation-highload-systems:~$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

Тут видно, что сервер не принял ни одного запроса (из попыток следующего `curl`):

```
user@operation-highload-systems:~$ curl -v http://localhost:8080
*   Trying 127.0.0.1:8080...
*   Trying ::1:8080...
* connect to ::1 port 8080 failed: Connection refused
```

При этом при помощи `tcpdump` видно, что запросы блочатся (так как стоит правило `DROP`):

```
user@operation-highload-systems:~$ sudo tcpdump -n -i lo tcp port 8080
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on lo, link-type EN10MB (Ethernet), snapshot length 262144 bytes
02:15:52.236917 IP 127.0.0.1.60626 > 127.0.0.1.8080: Flags [S], seq 4232294095, win 65495, options [mss 65495,sackOK,TS val 975456469 ecr 0,nop,wscale 7], length 0
02:15:52.446702 IP6 ::1.35788 > ::1.8080: Flags [S], seq 1949206464, win 65476, options [mss 65476,sackOK,TS val 2286514691 ecr 0,nop,wscale 7], length 0
02:15:52.446709 IP6 ::1.8080 > ::1.35788: Flags [R.], seq 0, ack 1949206465, win 0, length 0
02:15:53.330004 IP 127.0.0.1.60626 > 127.0.0.1.8080: Flags [S], seq 4232294095, win 65495, options [mss 65495,sackOK,TS val 975457562 ecr 0,nop,wscale 7], length 0
02:15:54.363594 IP 127.0.0.1.60626 > 127.0.0.1.8080: Flags [S], seq 4232294095, win 65495, options [mss 65495,sackOK,TS val 975458595 ecr 0,nop,wscale 7], length 0
02:15:55.398100 IP 127.0.0.1.60626 > 127.0.0.1.8080: Flags [S], seq 4232294095, win 65495, options [mss 65495,sackOK,TS val 975459630 ecr 0,nop,wscale 7], length 0
02:15:56.401802 IP 127.0.0.1.60626 > 127.0.0.1.8080: Flags [S], seq 4232294095, win 65495, options [mss 65495,sackOK,TS val 975460634 ecr 0,nop,wscale 7], length 0
02:15:57.439325 IP 127.0.0.1.60626 > 127.0.0.1.8080: Flags [S], seq 4232294095, win 65495, options [mss 65495,sackOK,TS val 975461671 ecr 0,nop,wscale 7], length 0
02:15:59.496138 IP 127.0.0.1.60626 > 127.0.0.1.8080: Flags [S], seq 4232294095, win 65495, options [mss 65495,sackOK,TS val 975463728 ecr 0,nop,wscale 7], length 0
... 
...
```

---

## Задание 4. Аппаратное ускорение сетевого трафика

* С помощью ethtool исследуйте offload возможности вашего сетевого адаптера.
* Покажите включён ли TCP segmentation offload.
* Объясните, какую задачу решает TCP segmentation offload.

---

Анализ сетевого интерфейса `enp0s3`:

```
user@operation-highload-systems:~$ ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 08:00:27:95:3a:58 brd ff:ff:ff:ff:ff:ff

user@operation-highload-systems:~$ sudo ethtool -k enp0s3
Features for enp0s3:
rx-checksumming: off
tx-checksumming: on
 tx-checksum-ipv4: off [fixed]
 tx-checksum-ip-generic: on
 tx-checksum-ipv6: off [fixed]
 tx-checksum-fcoe-crc: off [fixed]
 tx-checksum-sctp: off [fixed]
scatter-gather: on
 tx-scatter-gather: on
 tx-scatter-gather-fraglist: off [fixed]
tcp-segmentation-offload: on
 tx-tcp-segmentation: on
 tx-tcp-ecn-segmentation: off [fixed]
 tx-tcp-mangleid-segmentation: off
 tx-tcp6-segmentation: off [fixed]
generic-segmentation-offload: on
generic-receive-offload: on
large-receive-offload: off [fixed]
rx-vlan-offload: on
tx-vlan-offload: on [fixed]
ntuple-filters: off [fixed]
receive-hashing: off [fixed]
highdma: off [fixed]
rx-vlan-filter: on [fixed]
vlan-challenged: off [fixed]
tx-lockless: off [fixed]
netns-local: off [fixed]
tx-gso-robust: off [fixed]
tx-fcoe-segmentation: off [fixed]
tx-gre-segmentation: off [fixed]
tx-gre-csum-segmentation: off [fixed]
tx-ipxip4-segmentation: off [fixed]
tx-ipxip6-segmentation: off [fixed]
tx-udp_tnl-segmentation: off [fixed]
tx-udp_tnl-csum-segmentation: off [fixed]
tx-gso-partial: off [fixed]
tx-tunnel-remcsum-segmentation: off [fixed]
tx-sctp-segmentation: off [fixed]
tx-esp-segmentation: off [fixed]
tx-udp-segmentation: off [fixed]
tx-gso-list: off [fixed]
fcoe-mtu: off [fixed]
tx-nocache-copy: off
loopback: off [fixed]
rx-fcs: off
rx-all: off
tx-vlan-stag-hw-insert: off [fixed]
rx-vlan-stag-hw-parse: off [fixed]
rx-vlan-stag-filter: off [fixed]
l2-fwd-offload: off [fixed]
hw-tc-offload: off [fixed]
esp-hw-offload: off [fixed]
esp-tx-csum-hw-offload: off [fixed]
rx-udp_tunnel-port-offload: off [fixed]
tls-hw-tx-offload: off [fixed]
tls-hw-rx-offload: off [fixed]
rx-gro-hw: off [fixed]
tls-hw-record: off [fixed]
rx-gro-list: off
macsec-hw-offload: off [fixed]
rx-udp-gro-forwarding: off
hsr-tag-ins-offload: off [fixed]
hsr-tag-rm-offload: off [fixed]
hsr-fwd-offload: off [fixed]
hsr-dup-offload: off [fixed]
```

Здесь же видно строку `tcp-segmentation-offload: on`, показывающую, что данный флаг включён

TCP segmentation offset решает задачу разделения пакетов на MTU-сегменты на уровень сетевой карты (без `CPU`). Также в процессе
сетевая карта выдает нужные TCP/IP заголовки каждому сегменту и проверяет их контрольные суммы

Без технологии TCP segmentation offset сетевая карта только принимает и отправляет пакеты, а этой накладной обработкой занимается процессор, 
что увеличивает нагрузку на него

---
