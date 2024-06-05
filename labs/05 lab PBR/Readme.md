Схема лабораторного стенда:

![Screenshot_4](https://github.com/verttte/otus-labs/assets/165086553/7a9f992e-11a1-4e54-92d6-229332137354)

10.10.100.28/30 - Триада - Лабытнаги

10.10.100.32/30 - Триада - Чокурдах R25-R28

10.10.100.36/30 - Триада - Чокурдах R26-R28


Пользовательские сети в Чокурдахе

192.168.30.0/24

192.168.31.0/24



#### 1. и 2. Настроить политику маршрутизации в офисе Чокурдах - распределить трафик между 2 линками.

В Чокурдахе две пользовательские подсети - VLAN 30 и VLAN 31. Для каждой из них создан подинтерфейс на маршрутизаторе.

```
interface Ethernet0/2.30
 description VLAN0030
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0
!
interface Ethernet0/2.31
 description VLAN0031
 encapsulation dot1Q 31
 ip address 192.168.31.1 255.255.255.0
!
```

Для связи во вне имеются два линка

```
interface Ethernet0/0
 description TRIADA_R26
 ip address 10.10.100.38 255.255.255.252
!
interface Ethernet0/1
 description TRIADA_R25
 ip address 10.10.100.34 255.255.255.252
!
```

Для распределения трафика между двумя линками настроим политику маршрутизации - PBR. Для этого сперва создаем ACL, которые будут выбирать для нас трафик конкретной подсети.

```
ip access-list standard NET30
 permit 192.168.30.0 0.0.0.255
ip access-list standard NET31
 permit 192.168.31.0 0.0.0.255
```

Затем создаем карту маршрутов для каждой подсети. `match` – условие подпадания трафика под данное правило. `set ip next-hop` - куда направить трафик, в какой линк.

```
route-map NET30 permit 10
 match ip address NET30
 set ip next-hop 10.10.100.37
!
route-map NET31 permit 10
 match ip address NET31
 set ip next-hop 10.10.100.33
!
```

Затем на подинтерфейсе сети определяем политику, указываем соответствующую политику.

```
interface Ethernet0/2.30
 ip policy route-map NET30
interface Ethernet0/2.31
 ip policy route-map NET31
```

#### 3. Настройка отслеживание линка через технологию IP SLA.

```
ip sla 1
 icmp-echo 10.10.100.37 source-ip 10.10.100.38
 frequency 10
ip sla schedule 1 life forever start-time now
!
```

#### 4. Настройка для офиса Лабытнанги маршрута по-умолчанию.

```
ip route 0.0.0.0 0.0.0.0 10.10.100.29
!
```

