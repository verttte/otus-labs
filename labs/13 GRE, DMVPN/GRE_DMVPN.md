### Схемы лабораторного стенда

![image](https://github.com/user-attachments/assets/87becba8-bbc2-462b-8960-b3b521ca840b)

Для организации туннелей выделим новую сеть - 10.9.0.0  
 - 10.9.0.0/30 - Москва R14 - С.-Петербург R18
 - 10.9.0.4/30 - Москва R15 - С.-Петербург R18
 - 10.9.0.8/29 - Москва R15 - Чокурдах, Лабытнаги

### Настройте GRE между офисами Москва и С.-Петербург

Настроим встречные туннели на главных роутерах в городах

R14  
```
interface Tunnel1418
 description R18
 ip address 10.9.0.1 255.255.255.252
 keepalive 10 3
 tunnel source Loopback0
 tunnel destination 10.10.50.18
```

R15  
```
interface Tunnel1518
 description R18
 ip address 10.9.0.5 255.255.255.252
 keepalive 10 3
 tunnel source Loopback0
 tunnel destination 10.10.50.18
```

R18  
```
interface Tunnel1418
 description R14
 ip address 10.9.0.2 255.255.255.252
 keepalive 10 3
 tunnel source Loopback0
 tunnel destination 10.10.50.14
!
interface Tunnel1518
 description R15
 ip address 10.9.0.6 255.255.255.252
 keepalive 10 3
 tunnel source Loopback0
 tunnel destination 10.10.50.15
```

Проверим пингом. Не смотря на отсутствие анонсов в протоколах маршрутизации, адреса туннелей доступны.

```
R18#ping 10.9.0.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.9.0.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/3 ms
R18#ping 10.9.0.5
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.9.0.5, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/4 ms
R18#
```

### Настройте DMVPN между Москва и Чокурдах, Лабытнанги

В качестве HUB в Москве будет выступать R15. 
R15  
```
interface Tunnel152728
 ip address 10.9.0.9 255.255.255.248
 no ip redirects
 ip mtu 1400
 ip nhrp authentication otus
 ip nhrp map multicast dynamic
 ip nhrp network-id 152728
 ip tcp adjust-mss 1360
 tunnel source Loopback0
 tunnel mode gre multipoint
```

Spoke будут обращаться на адрес, настроенный на Loopback0 - 10.10.50.15. Этот адрес ананосируется по динамической маршрутизации, но на маршрутизаторах R27 и R28 её нет. Для R27 адрес будет доступен по настроенному дефолту. На R28 создадим два статических маршрута (по количеству аплинков).

```
ip route 10.10.50.15 255.255.255.255 10.10.100.33
ip route 10.10.50.15 255.255.255.255 10.10.100.37
```
Также для доступности самих R27 и R28 объявим сети стыков с ними по BGP на R25 и R26

R25  
```
router bgp 520
network 10.10.100.28 mask 255.255.255.252
network 10.10.100.32 mask 255.255.255.252
```
R26  
```
router bgp 520
network 10.10.100.36 mask 255.255.255.252
```

Настройка Spoke  

R27  
```
interface Tunnel152728
 ip address 10.9.0.10 255.255.255.248
 ip mtu 1400
 ip nhrp authentication otus
 ip nhrp map multicast 10.10.50.15
 ip nhrp map 10.9.0.9 10.10.50.15
 ip nhrp network-id 152728
 ip nhrp holdtime 600
 ip nhrp nhs 10.9.0.9
 ip nhrp registration no-unique
 ip nhrp shortcut
 ip nhrp redirect
 ip tcp adjust-mss 1360
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
```

R28  
```
interface Tunnel152728
 ip address 10.9.0.11 255.255.255.248
 ip mtu 1400
 ip nhrp authentication otus
 ip nhrp map multicast 10.10.50.15
 ip nhrp map 10.9.0.9 10.10.50.15
 ip nhrp network-id 152728
 ip nhrp holdtime 600
 ip nhrp nhs 10.9.0.9
 ip nhrp registration no-unique
 ip nhrp shortcut
 ip nhrp redirect
 ip tcp adjust-mss 1360
 tunnel source Ethernet0/1
 tunnel mode gre multipoint
```

Проверка  
R15  
```
R15#sh dmvpn

==========================================================================

Interface: Tunnel152728, IPv4 NHRP Details
Type:Hub, NHRP Peers:2,

 # Ent  Peer NBMA Addr Peer Tunnel Add State  UpDn Tm Attrb
 ----- --------------- --------------- ----- -------- -----
     1 10.10.100.30          10.9.0.10    UP 00:56:30     D
     1 10.10.100.34          10.9.0.11    UP 00:44:05     D

```
```
R15#show ip nhrp brief
   Target             Via            NBMA           Mode   Intfc   Claimed
        10.9.0.10/32 10.9.0.10       10.10.100.30    dynamic  Tu15272 <   >
        10.9.0.11/32 10.9.0.11       10.10.100.34    dynamic  Tu15272 <   >
```
```
R15#ping 10.9.0.10
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.9.0.10, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/2/5 ms
R15#
R15#ping 10.9.0.11
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.9.0.11, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/3 ms
```

R27  
```
R27#show dmvpn

==========================================================================

Interface: Tunnel152728, IPv4 NHRP Details
Type:Spoke, NHRP Peers:2,

 # Ent  Peer NBMA Addr Peer Tunnel Add State  UpDn Tm Attrb
 ----- --------------- --------------- ----- -------- -----
     1 10.10.50.15            10.9.0.9    UP 01:01:27     S
     1 10.10.100.34          10.9.0.11    UP 00:00:28     D
```
```
R27#show ip nhrp brief
   Target             Via            NBMA           Mode   Intfc   Claimed
         10.9.0.9/32 10.9.0.9        10.10.50.15     static   Tu15272 <   >
        10.9.0.10/32 10.9.0.10       10.10.100.30    dynamic  Tu15272 <   >
        10.9.0.11/32 10.9.0.11       10.10.100.34    dynamic  Tu15272 <   >
```

R28  
```
R28#show dmvpn

==========================================================================

Interface: Tunnel152728, IPv4 NHRP Details
Type:Spoke, NHRP Peers:2,

 # Ent  Peer NBMA Addr Peer Tunnel Add State  UpDn Tm Attrb
 ----- --------------- --------------- ----- -------- -----
     1 10.10.50.15            10.9.0.9    UP 00:49:25     S
     1 10.10.100.30          10.9.0.10    UP 00:00:51     D
```
```
R28#sh ip nhrp brief
   Target             Via            NBMA           Mode   Intfc   Claimed
         10.9.0.9/32 10.9.0.9        10.10.50.15     static   Tu15272 <   >
        10.9.0.10/32 10.9.0.10       10.10.100.30    dynamic  Tu15272 <   >
        10.9.0.11/32 10.9.0.11       10.10.100.34    dynamic  Tu15272 <   >
```
