### Схемы лабораторного стенда

### Настроите GRE между офисами Москва и С.-Петербург.

R14
```
interface Tunnel1418
 description R18
 ip address 10.9.0.1 255.255.255.252
 keepalive 10 3
 tunnel source Loopback0
 tunnel destination 10.10.50.18
```
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

### Настроите DMVMN между Москва и Чокурдах, Лабытнанги.
