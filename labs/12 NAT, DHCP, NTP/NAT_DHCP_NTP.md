### Схема лабораторного стенда

![image](https://github.com/user-attachments/assets/1db29e3f-bf4d-4946-8d47-dfe88a9f738e)


### Настройка NAT (PAT) на R14 и R15. Трансляция должна осуществляться в адрес автономной системы AS1001.

Для того, чтоб трансляция осуществлялась в адрес автономной системы мы будем использовать интерфейс Loopback, настроенный на маршрутизаторе. Настройки одинаковые на обоих устройствах.

При помощи ACL определяем какие IP-адреса будут попадать под NAT.

```
ip access-list standard NAT
permit 192.168.10.0 0.0.0.255
permit 192.168.70.0 0.0.0.255
```

Создаем трансляцию. В пакетах, попадающих под критерий в ACL, адрес источника будет меняться на адрес, настроенный на интерфейсе Loopback0. `overload` указывает, что будет использоваться PAT.

```
ip nat inside source list NAT interface Loopback0 overload
```

Указываем на интерфейсах какие смотрят в сторону внутренней и внешней сети.

```
interface range Ethernet 0/0-1
ip nat inside
interface Ethernet 0/2
ip nat outside
```

Для проверки результатов будем использовать `debug ip icmp`, включенный на R18. Запустим пинг с VPC1 до адреса на Loopback0 маршрутизатора R18.  
До настройки NAT запросы приходили с IP-адресом источника VPC1, маршрут до которого на R18 неизвестен. Соответственно на пинг невозможно было послать ответ.  
После настройки NAT запросы приходят с замененным IP-адресом источника на адрес, настроенный на Loopback0, маршрут до которого известен на R18. И мы получаем ответ.

![image](https://github.com/user-attachments/assets/f5f3e94e-b6b6-4cdd-ad41-fbf6daafb1e2)

На маршрутизаторе в Москве есть трансляции

![image](https://github.com/user-attachments/assets/fc8c9a57-80c4-46b9-b3d8-71ea246cdec3)


### Настройка NAT (PAT) на R18. Трансляция должна осуществляться в пул из 5 адресов автономной системы AS2042.

Будем использовать сеть 10.10.18.0/29. Настроим один интерфейс с адресом из этой сети для того, чтоб сеть появилась в таблице маршрутизации. Объявим эту сеть в BGP.

```
interface Loopback1
ip address 10.10.18.1 255.255.255.248
router bgp 2042
network 10.10.18.0 mask 255.255.255.248
```

Создадим ACL с IP-адресами источников. Настроим пул адресов, в которые будет происходить трансляция. Включим трансляцию.

```
ip access-list standard NAT
permit 192.168.80.0 0.0.0.255
permit 192.168.90.0 0.0.0.255
exit
ip nat pool NAT 10.10.18.1 10.10.18.5 netmask 255.255.255.248
ip nat inside source list NAT pool NAT overload
```

Укажем входящие и исходящие интерфейсы

```
interface range Ethernet0/0-1
ip nat inside
exit
interface range Ethernet0/2-3
ip nat outside
```

Проверка пингом. С VPC8 или VPC9 отправляем пинг до 10.10.50.15. Дебаг на стороне R15 показывает отправление ответов на IP-адрес из настроенного пула. `show ip nat translations` показывает трансляции на R18.

```
VPC8> ping 10.10.50.15

84 bytes from 10.10.50.15 icmp_seq=1 ttl=251 time=3.638 ms
84 bytes from 10.10.50.15 icmp_seq=2 ttl=251 time=4.000 ms
^C
```

```
R15#debug ip icmp
R15#
*Aug 21 18:59:59.490: ICMP: echo reply sent, src 10.10.50.15, dst 10.10.18.2, topology BASE, dscp 0 topoid 0
*Aug 21 19:00:00.495: ICMP: echo reply sent, src 10.10.50.15, dst 10.10.18.2, topology BASE, dscp 0 topoid 0
R15#
```

```
R18#sho ip nat translations
Pro Inside global      Inside local       Outside local      Outside global
icmp 10.10.18.2:29498  192.168.80.80:29498 10.10.50.15:29498 10.10.50.15:29498
icmp 10.10.18.2:29754  192.168.80.80:29754 10.10.50.15:29754 10.10.50.15:29754
icmp 10.10.18.2:30010  192.168.80.80:30010 10.10.50.15:30010 10.10.50.15:30010
icmp 10.10.18.2:30266  192.168.80.80:30266 10.10.50.15:30266 10.10.50.15:30266
```

### Настройка статического NAT для R20

Для демонстрации NAT добавим VPC2 с IP-адресом 192.168.20.20/24 и настроил под него интерфейс на маршрутизаторе.

```
R20(config)#interface Ethernet0/1
R20(config-if)#desc VPC2
R20(config-if)#ip address 192.168.20.1 255.255.255.0
R20(config-if)#no shut
```

Настраиваем статическую трансляцию одного адреса в один. Вместо конкретного IP-адреса укажем интерфейс.

```
ip nat inside source static 192.168.20.20 interface Ethernet0/0
```
Указываем входищий и исходящий интерфейсы
```
interface Ethernet0/0
ip nat outside
interface Ethernet0/1
ip nat inside
```

Проверим. Запускаем пинг с VPC2 и видим, что дебаг на R15 показывает ответы на адрес 10.10.200.22, который настроен на интерфейсе Ethernet0/0 R20.

```
VPC2> ping 10.10.50.15

84 bytes from 10.10.50.15 icmp_seq=1 ttl=254 time=1.134 ms
84 bytes from 10.10.50.15 icmp_seq=2 ttl=254 time=1.306 ms
^C
```

```
R15#debug ip icmp
R15#
*Aug 21 19:48:34.306: ICMP: echo reply sent, src 10.10.50.15, dst 10.10.200.22, topology BASE, dscp 0 topoid 0
*Aug 21 19:48:35.307: ICMP: echo reply sent, src 10.10.50.15, dst 10.10.200.22, topology BASE, dscp 0 topoid 0
```


### Настройка NAT так, чтобы R19 был доступен с любого узла для удаленного управления

### Настройка статический NAT (PAT) для офиса Чокурдах

В Чокурдах ранее была принята политика, что трафик сегмента VLAN 30 отправляется в Триада R26, а сегмента VLAN 31 в Триада R25. Настроим NAT в соответствии с этой политикой.

Создаем отдельные для каждого сегмента ACL

```
access-list 30 permit 192.168.30.0 0.0.0.255
access-list 31 permit 192.168.31.0 0.0.0.255
```

Включаем отдельные для каждого сегмена странсляции. Параметр `overload` определяет, что используется PAT.

```
ip nat inside source list 30 interface Ethernet0/0 overload
ip nat inside source list 31 interface Ethernet0/1 overload
```

Помечаем входящие и исходящие интерфейсы

```
interface Ethernet0/2.30
ip nat inside
interface Ethernet0/2.31
ip nat inside
interface range Ethernet0/0-1
ip nat outside
```

Проверяем. При пинге соседних маршрутизаторов в дебаге видим ответ на IP-адрес не устройства, с которого был пинг, а на адрес исходящего интерфейса.

```
R25#
*Aug 21 22:13:49.528: ICMP: echo reply sent, src 10.10.100.33, dst 10.10.100.34, topology BASE, dscp 0 topoid 0
*Aug 21 22:13:50.530: ICMP: echo reply sent, src 10.10.100.33, dst 10.10.100.34, topology BASE, dscp 0 topoid 0
```
```
R26#
*Aug 21 22:12:58.002: ICMP: echo reply sent, src 10.10.100.37, dst 10.10.100.38, topology BASE, dscp 0 topoid 0
*Aug 21 22:12:58.345: ICMP: echo reply sent, src 10.10.100.37, dst 10.10.100.38, topology BASE, dscp 0 topoid 0
```

### Настройка для IPv4 DHCP сервера в офисе Москва на маршрутизаторах R12 и R13. VPC1 и VPC7 должны получать сетевые настройки по DHCP.

На маршрутизаторах используется резервирование шлюза GLBP. Целесообразно DHCP серверы тоже сделать зарезервированными. В соответствии с [документацией Cisco](https://www.cisco.com/c/en/us/support/docs/ip/dynamic-address-allocation-resolution/27470-100.html#toc-hId--1017204738:~:text=At%20the%20address,from%20the%20pool.), DHCP проверяет наличие конфликтов с помощью ping и gratuitous ARP, при обнаружении конфликта адрес удаляется из пула. Соответственно мы просто создаем две одинаковых конфигурации на обоих маршрутизаторах. Адрес ПК получит у одного из серверов, а второй сервер проверив, что адрес занят не будет выдавать такой же.

Укажем адреса, не участвующие в выдаче. 1 - шлюз, 12 и 13 - адреса на подинтерфейсах каждого сегмента.

```
ip dhcp excluded-address 192.168.10.1
ip dhcp excluded-address 192.168.10.12 192.168.10.13
ip dhcp excluded-address 192.168.70.1
ip dhcp excluded-address 192.168.70.12 192.168.70.13
```

Настроим пулы

```
ip dhcp pool NET10
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 domain-name otus.ru
 dns-server 192.168.10.1
!
ip dhcp pool NET70
 network 192.168.70.0 255.255.255.0
 default-router 192.168.70.1
 domain-name otus.ru
 dns-server 192.168.70.1
```

Запрашиваем адрес на VPC. Проверяем пинг до соседнего сегмента.

```
VPC10> ip dhcp
DDORA IP 192.168.10.2/24 GW 192.168.10.1

VPC10> ping 192.168.70.2

84 bytes from 192.168.70.2 icmp_seq=1 ttl=63 time=6.116 ms
84 bytes from 192.168.70.2 icmp_seq=2 ttl=63 time=3.387 ms
^C
```
Командой `show ip dhcp server statistics` убеждаемся, что запросы приходит на оба сервера, но адрес выдает только один.

![image](https://github.com/user-attachments/assets/a07e1a4f-adbd-4336-8ed0-7b46f8daf2f9)

### Настройка NTP сервера на R12 и R13. Все устройства в офисе Москва должны синхронизировать время с R12 и R13.

На R12 и R13 установим время
```
clock set 14:59:00 22 Aug 2024
```

И произведём остальные настройки NTP. 

```
clock timezone Moscow 3 0
ntp master 3
ntp update-calendar
```

Так как NTP сервера в сети у нас два, объявим их для друг друга как пиры.

R12  
```
ntp peer 10.10.50.13
```
R13  
```
ntp peer 10.10.50.12
```

Произведем на всех остальных устройствах растройки NTP-клиента

```
clock timezone Moscow 3 0
ntp server 10.10.50.12 prefer
ntp server 10.10.50.13
```
Для устройств на левой половине схемы предпочтительный сервер 10.10.50.12, для устройств на правой половине - 10.10.50.13


