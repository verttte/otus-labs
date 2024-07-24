#### Схема лабораторного стенда

![image](https://github.com/user-attachments/assets/aeb2b543-985a-4fd6-bb13-ac69b003e3f2)

На каждом маршрутизаторе настроен loopback интерфейс. IP-адрес взят в соответствии с IP-планом из сети 10.10.50.0/24, последний октет - номер маршрутизатора.

На всех маршрутизаторах аналогичные первичные настройки - включаем процесс BGP, указываем ID маршрутизатора, объявляем соседа, указываем описание соседа.

#### 1. Настройка eBGP между офисом Москва и двумя провайдерами - Киторн и Ламас

Настраиваем на R14 и R22. 

```
router bgp 1001
 bgp router-id 10.10.50.14
 neighbor 10.10.100.1 remote-as 101
 neighbor 10.10.100.1 description KITORN
```

```
router bgp 101
 bgp router-id 10.10.50.22
 neighbor 10.10.100.2 remote-as 1001
 neighbor 10.10.100.2 description MOSCOW
```

Настраиваем на R15 и R21.

```
router bgp 1001
 bgp router-id 10.10.50.15
 neighbor 10.10.100.5 remote-as 301
 neighbor 10.10.100.5 description LAMAS
```

```
router bgp 301
 bgp router-id 10.10.50.21
 neighbor 10.10.100.6 remote-as 1001
 neighbor 10.10.100.6 description MOSCOW
```

Проверяем на всех маршрутизаторах командой `show ip bgp summary` и среди прочей информации видим соседство. Вывод частично опущен.

```
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.10.100.1     4          101     127     123       13    0    0 01:48:21        0
R14#
```

```
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.10.100.2     4         1001     126     129       15    0    0 01:50:18        0
R22#
```

```
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.10.100.5     4          301     132     124       13    0    0 01:50:36        0
R15#
```

```
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.10.100.6     4         1001     126     133       15    0    0 01:51:49        0
R21#
```

#### 2. Настройка eBGP между провайдерами Киторн и Ламас

Настраиваем на R22 и R21.

```
router bgp 101
 neighbor 10.10.100.9 remote-as 301
 neighbor 10.10.100.9 description LAMAS
```

```
router bgp 301
 neighbor 10.10.100.10 remote-as 101
 neighbor 10.10.100.10 description KITORN
```

Проверяем командой `show ip bgp summary`.

```
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.10.100.2     4         1001     132     136       15    0    0 01:56:40        0
10.10.100.9     4          301     135     136       15    0    0 01:54:21        0
R22#
```

```
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.10.100.6     4         1001     126     133       15    0    0 01:51:49        0
10.10.100.10    4          101     133     132       15    0    0 01:51:49        0
R21#
```

К сессиям с маршрутизаторами в Москве добавились новые между Киторн и Ламас.

#### 3. Настройка eBGP между Ламас и Триада

Настраиваем на R21 и R24.

```
router bgp 301
 neighbor 10.10.100.17 remote-as 520
 neighbor 10.10.100.17 description TRIADA
```

```
router bgp 520
 bgp router-id 10.10.50.24
 neighbor 10.10.100.18 remote-as 301
 neighbor 10.10.100.18 description LAMAS
```

На маршрутизаторе в Ламас теперь три сесии - с Москвой, с Киторн и с Триадой.

```
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.10.100.6     4         1001     126     133       15    0    0 01:51:49        0
10.10.100.10    4          101     133     132       15    0    0 01:51:49        0
10.10.100.17    4          520     129     128       15    0    0 01:48:12        0
R21#
```

На R24 в Триаде

```
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.10.100.18    4          301     138     139       15    0    0 01:57:20        0
R24#
```

#### 4. Настройка eBGP между офисом С.-Петербург и провайдером Триада

```
router bgp 2042
 bgp router-id 10.10.50.18
 neighbor 10.10.100.21 remote-as 520
 neighbor 10.10.100.21 description TRIADA
```

```
router bgp 520
 neighbor 10.10.100.22 remote-as 2042
 neighbor 10.10.100.22 description SPB
```

В Санкт-Петербурге сеесия с Триадой

```
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.10.100.21    4          520     136     136       15    0    0 01:56:05        0
R18#
```

В Триаде две сессии - с Санкт-Петербургом и Ламас

```
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.10.100.18    4          301     138     139       15    0    0 01:57:20        0
10.10.100.22    4         2042     134     134       15    0    0 01:54:26        0
R24#
```

#### 5. Организация IP-доступности между пограничным роутерами офисами Москва и С.-Петербург

Проверку IP-доступности проведём командой ping, обращаться будем к адресу на интерфейсе loopback маршрутизатора.

Необходимо настроить маршрутизацию. Командой `network` указываем сеть, которую будем анонсировать в BGP.

R14
```
router bgp 1001
 network 10.10.50.14 mask 255.255.255.255
```

R15
```
router bgp 1001
 network 10.10.50.15 mask 255.255.255.255
```

R18
```
router bgp 2042
 network 10.10.50.18 mask 255.255.255.255
```

После этого машрутизаторы начинают передавать информацию об этих префиксах друг другу. Мы можем увидеть увеличивающийся счетчик PfxRcd в выводе команды `show ip bgp summary` по всей цепочке. А так же команда `show ip route bgp` отобразит передаваемую маршрутную информацию. Пример на одном из маршрутизаторов в середине цепочки

```
R21#show ip route bgp
<...>
B        10.10.50.14/32 [20/0] via 10.10.100.10, 01:17:34
B        10.10.50.15/32 [20/0] via 10.10.100.6, 01:08:50
B        10.10.50.18/32 [20/0] via 10.10.100.17, 01:23:15
```

Не смотря на то, что R14 и R15 в Москве получают маршрут до сети на R18 в Санкт-Петербурге, команда ping всё равно не проходит

```
R14#show ip route bgp
<...>
B        10.10.50.18/32 [20/0] via 10.10.100.1, 01:30:04
R14#
R14#ping 10.10.50.18
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.10.50.18, timeout is 2 seconds:
.....
Success rate is 0 percent (0/5)
R14#
R14#ping 10.10.50.18 source 10.10.50.14
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.10.50.18, timeout is 2 seconds:
Packet sent with a source address of 10.10.50.14
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms
R14#
```

Аналогичная ситуация с R18 в Санкт-Петербурге.

Это происходит потому, что по умолчанию в качестве источника в ping используется IP-адрес исходящего интерфейса. Пакет на целевой роутер приходит с источником, которому он не знает куда отвечать.

Для того, чтоб ping отработал без условностей добавим информацию о адресах исходящих интерфейсах в BGP.

R14
```
router bgp 1001
 network 10.10.100.0 mask 255.255.255.252
```

R15
```
router bgp 1001
 network 10.10.100.4 mask 255.255.255.252
```

R18
```
router bgp 2042
 network 10.10.100.20 mask 255.255.255.252
```

В таблицах маршрутизации появились подсети исходящих интерфейсов и ping заработал без указания источника

```
R14#show ip route bgp
<...>
B        10.10.50.18/32 [20/0] via 10.10.100.1, 02:29:45
B        10.10.100.20/30 [20/0] via 10.10.100.1, 00:14:02
```

```
R15#show ip route bgp
<...>
B        10.10.50.18/32 [20/0] via 10.10.100.5, 02:30:45
B        10.10.100.20/30 [20/0] via 10.10.100.5, 00:15:02
```

```
R18#show ip route bgp
<...>
B        10.10.50.14/32 [20/0] via 10.10.100.21, 02:23:50
B        10.10.50.15/32 [20/0] via 10.10.100.21, 02:15:07
B        10.10.100.0/30 [20/0] via 10.10.100.21, 00:19:52
B        10.10.100.4/30 [20/0] via 10.10.100.21, 02:08:52
```

```
R14#ping 10.10.50.18
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.10.50.18, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms
```

```
R18#ping 10.10.50.14
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.10.50.14, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms
```
