# # Фильтрация трафика - firewalld, iptables.

Сценарии iptables:

- реализовать knocking port;

- centralRouter может попасть на ssh inetRouter через knock скрипт пример в материалах;

- добавить inetRouter2, который виден(маршрутизируется (host-only тип сети для виртуалки)) с хоста или форвардится порт через локалхост;

- запустить nginx на centralServer;

- пробросить 80й порт на inetRouter2 8080;

- дефолт в инет оставить через inetRouter;

Цель:

- настраивать файрвалл с использованием iptables;

- настраивать NAT;

- пробрасывать порты;

- настраивать взаимодействие с роутингом;

- понимать работу таблиц и цепочек.

## Развертывание стенда для демострации фильтрации трафика (iptables).

Стэнд состоит из хостовой машины под управлением ОС Ubuntu 20.04 на которой развернут Ansible и четырех виртуальных машин CentOS/7 2004.01.

Машины с именами: `inetRouter`, `inetRouter2`, `centralRouter` выполняют роль `роутеров`.
Роутеры объединены в сеть `192.168.255.0/28`

Машина с именем `centralServer` выполняет роль `Web-сервера`.
`centralRouter` и `centralServer` объединены в сеть `192.168.0.0/30`

Разворачиваем инфраструктуру гибридно (если можно так сказать) в Vagrant с реализацией сачти функций через Ansible.

Все коментарии по каждому блоку указаны в тексте Playbook - `routing.yml` и тексте `Vagrantfile`.

Выполняем установку стенда

```
vagrant up
```

## Результат работы

### Проверяем доступность с centralRouter по ssh inetrRouter через knock скрипт

Пробуем подключится без скрипта

```
[root@centralRouter vagrant]# ssh root@192.168.255.1
ssh: connect to host 192.168.255.1 port 22: Connection timed out
```
получаем Connection timed out.

Теперь пробуем с помощью `knock.sh`

```
[root@centralRouter vagrant]# ./knock.sh 192.168.255.1 8881 7777 9991

Starting Nmap 6.40 ( http://nmap.org ) at 2023-05-08 12:20 UTC
Warning: 192.168.255.1 giving up on port because retransmission cap hit (0).
Nmap scan report for 192.168.255.1
Host is up (0.00031s latency).
PORT     STATE    SERVICE
8881/tcp filtered unknown
MAC Address: 08:00:27:93:D4:13 (Cadmus Computer Systems)

Nmap done: 1 IP address (1 host up) scanned in 0.36 seconds

Starting Nmap 6.40 ( http://nmap.org ) at 2023-05-08 12:20 UTC
Warning: 192.168.255.1 giving up on port because retransmission cap hit (0).
Nmap scan report for 192.168.255.1
Host is up (0.00038s latency).
PORT     STATE    SERVICE
7777/tcp filtered cbt
MAC Address: 08:00:27:93:D4:13 (Cadmus Computer Systems)

Nmap done: 1 IP address (1 host up) scanned in 0.35 seconds

Starting Nmap 6.40 ( http://nmap.org ) at 2023-05-08 12:20 UTC
Warning: 192.168.255.1 giving up on port because retransmission cap hit (0).
Nmap scan report for 192.168.255.1
Host is up (0.00031s latency).
PORT     STATE    SERVICE
9991/tcp filtered issa
MAC Address: 08:00:27:93:D4:13 (Cadmus Computer Systems)

Nmap done: 1 IP address (1 host up) scanned in 0.35 seconds
[root@centralRouter vagrant]# ssh root@192.168.255.1
root@192.168.255.1's password: 
Last login: Mon May  8 12:16:14 2023 from 192.168.255.2
[root@inetRouter ~]# ll
total 16
-rw-------. 1 root root 5570 Apr 30  2020 anaconda-ks.cfg
-rw-------. 1 root root 5300 Apr 30  2020 original-ks.cfg
```
### Проверяем форвардинг inetRouter2 и проброс портв с 80 centralServer на inetRouter2 8080.

Вывод маршрутов с centralServer на inetRouter2

```
[vagrant@centralServer ~]$ ip route get 192.168.255.3
192.168.255.3 via 192.168.0.1 dev eth1 src 192.168.0.2 
    cache 
[vagrant@centralServer ~]$ ping 192.168.255.3
PING 192.168.255.3 (192.168.255.3) 56(84) bytes of data.
64 bytes from 192.168.255.3: icmp_seq=1 ttl=63 time=0.915 ms
64 bytes from 192.168.255.3: icmp_seq=2 ttl=63 time=2.29 ms
64 bytes from 192.168.255.3: icmp_seq=3 ttl=63 time=2.31 ms
^C
--- 192.168.255.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2029ms
rtt min/avg/max/mdev = 0.915/1.840/2.312/0.654 ms
```

Для наглядности проверки доступности Web-сервера через inetRouter2 является форвардинг портов на хост машину для проверки проброса nginx сервера с centralServer

```
 case boxname.to_s
            when "inetRouter" # конфигурируем inetRouter
              ...

              box.vm.network 'forwarded_port', guest: 8080, host: 8080, host_ip: '127.0.0.1'
              SHELL
            end
```
В браузере хостовой машины смотрим адрес `127.0.0.1:8080`

![Снимок экрана от 2023-05-10 18-20-41](https://github.com/skyfly535/19WH_IPTAB/assets/114483769/bd297e79-50cd-4dc5-a716-327b6115e30b)

### Проверяем дефолт в инет оставить через inetRouter.

Выполняем `traceroute 8.8.8.8`

```
[vagrant@centralServer ~]$ traceroute 8.8.8.8
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets
 1  ospd-gw.ospd.net (192.168.0.1)  0.329 ms  0.267 ms  0.169 ms
 2  192.168.255.1 (192.168.255.1)  0.627 ms  0.632 ms  0.576 ms
 3  * * *
 4  * * *
 5  * * *
 6  10.251.14.124 (10.251.14.124)  9.967 ms  8.847 ms 10.251.14.126 (10.251.14.126)  9.788 ms
 7  * * 185.140.148.153 (185.140.148.153)  127.615 ms
 8  72.14.197.6 (72.14.197.6)  127.537 ms  128.877 ms  125.675 ms
 9  * * *
10  172.253.69.88 (172.253.69.88)  195.849 ms 108.170.250.129 (108.170.250.129)  191.010 ms 216.239.46.254 (216.239.46.254)  222.702 ms
11  108.170.250.66 (108.170.250.66)  221.678 ms 108.170.250.83 (108.170.250.83)  193.493 ms 108.170.250.34 (108.170.250.34)  193.620 ms
12  209.85.255.136 (209.85.255.136)  144.613 ms 142.251.238.82 (142.251.238.82)  144.226 ms 142.251.49.78 (142.251.49.78)  216.065 ms
13  108.170.232.251 (108.170.232.251)  218.527 ms 72.14.232.76 (72.14.232.76)  221.096 ms 209.85.254.6 (209.85.254.6)  218.466 ms
14  216.239.49.107 (216.239.49.107)  220.320 ms 108.170.233.163 (108.170.233.163)  220.078 ms 142.250.56.125 (142.250.56.125)  221.547 ms
15  * * *
16  * * *
17  * * *
18  * * *
19  * * *
20  * * *
21  * * *
22  * * *
23  * * *
24  dns.google (8.8.8.8)  182.355 ms *  181.793 ms
```

## P.S.: 
Выход в internet с `inetRouter` без маскарадинга должен выглядеть примерно так

```
iptables -t nat -A POSTROUTING -o eth0 -j SNAT --to-source 10.0.2.2
```
