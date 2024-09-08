### Схемы лабораторного стенда

Для организации туннелей выделим новую сеть - 10.9.0.0.

### Настроите GRE между офисами Москва и С.-Петербург

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

### Настроите DMVPN между Москва и Чокурдах, Лабытнанги
