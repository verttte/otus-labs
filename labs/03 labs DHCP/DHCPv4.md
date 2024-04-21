Схема лабораторного стенда

![image](https://github.com/verttte/otus-labs/assets/165086553/b7312765-280f-43af-91bb-fe1353069e52)

Таблица адресации

![image](https://github.com/verttte/otus-labs/assets/165086553/4f50fc23-1b2c-4721-ba08-56cfde894855)

Таблица VLAN

![image](https://github.com/verttte/otus-labs/assets/165086553/249a66c4-d676-4497-a16f-1bfa174b6c8d)

### Часть 1: Построение сети и настройка основных параметров устройства

Произведены базовые настройки устройств в соответствии с методичкой.

В соответствии с требованиями сеть 192.168.1.0/24 разбивается на три подсети

- 192.168.1.0/26 - Clients R1
- 192.168.1.64/27 - Mgmt R1
- 192.168.1.96/28 - Clients R2

#### Настройка маршрутизации между VLAN на R1.

- включен интерфейс Ethernet0/1 в сторону коммутатора
- настроены субинтерфейсы 
  - инкапсуляция 802.1Q
  - назначен первый полезный адрес из вычисленного пула IP-адресов
  - субинтерфейсу для native VLAN не назначен IP-адрес
  - указаны описания для каждого субинтерфейса
- убеждаемся, что субинтерфейсы работают командой `show interface ethernet 0/1.100` - в выводе видим `is up, line protocol is up`

#### Настройка связи между маршрутизаторами

На R2 порту Et0/1 в торону коммутатора назначается IP 192.168.1.97 255.255.255.240

Стыковые порты на каждом маршрутизаторе настраиваются в стыковую сеть 10.0.0.0/30

R1 Et0/0 - 10.0.0.1 255.255.255.252

R2 Et0/0 - 10.0.0.2 255.255.255.252

Настраиваем маршрут по умолчанию на каждом маршрутизаторе на IP-адрес стыкового IP соседнего маршрутизатора.

`
R1(config)#ip route 0.0.0.0 0.0.0.0 10.0.0.2
`

`
R2(config)#ip route 0.0.0.0 0.0.0.0 10.0.0.1
`

Убеждаемся, что статическая маршрутизация работает отправляя эхо-запрос с одного маршрутизатора на IP-адрес, который настроен на другом маршрутизаторе.

```
R1#ping 192.168.1.97
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.1.97, timeout is 2 seconds:
!!!!!
```

#### Настройка коммутаторов

На S1 созданы VLAN в соответствии с таблицей, настроен интерфейс управления VLAN 200, ему назначен второй IP-адрес из подсети управления. Настроен шлюз по умолчанию.

Неиспользуемым портам Et0/0-1 назначается VLAN 999, производится настройка в режим access, порты отключаются.

Используемые порты переводим в режим access. Et0/3 назначается VLAN 100.

Вопрос из методички:

>Почему интерфейс F0 / 5 указан в VLAN 1?

В нашей схеме это порт Et0/2. Ответ: порт, которому не указано однозначно какой VLAN он принадлежит помещается в умолчальную VLAN 1.

Устанавливаем для порта Et0/2 режим trunk, разрешаем VLAN 100, 200 и 1000; native VLAN указываем 1000.

Проверяем состояние транкинга.

```
S1#sh int e0/2 switchport
Name: Et0/2
Switchport: Enabled
Administrative Mode: trunk
Operational Mode: trunk
Administrative Trunking Encapsulation: dot1q
Operational Trunking Encapsulation: dot1q
Negotiation of Trunking: Off
Access Mode VLAN: 1 (default)
Trunking Native Mode VLAN: 1000 (Native)
Administrative Native VLAN tagging: enabled
<...>
Trunking VLANs Enabled: 100,200,1000
```

На S2 используется только VLAN 1. Поэтому для его управления создан интерфейс VLAN 1 и используется второй адрес из сети назначенной на порту маршрутизатора в сторону S2 - 192.168.1.98 255.255.255.240. Настроен шлюз по умолчанию.

Неиспользуемые порты Et0/0-1 отключаются.

Вопрос из методички:
>На этом этапе какой IP-адрес был бы у ПК, если бы они были подключены к сети с помощью DHCP?

Ответ: IP-адрес отсутствовал бы, либо автоматически был бы настроен link-local адрес из диапазона 169.254.0.0/16, при условии, что эта технология поддерживается ОС и включена.

### Часть 2: Настройка и проверка двух серверов DHCPv4 на R1 

Производим настройки на R1 для двух диапазонов клиентских подсетей

- исключаем первые пять адресов из пула.
- создаем и именуем пул
- указываем сеть, которую поддерживает этот DHCP-сервер
- указываем шлюз по умолчанию для каждого пула
- указываем доменное имя
- назначаем время аренды 2 дня 12 часов 30 минут

```
ip dhcp excluded-address 192.168.1.1 192.168.1.5
ip dhcp excluded-address 192.168.1.97 192.168.1.101
!
ip dhcp pool R1_Clients_LAN
 network 192.168.1.0 255.255.255.192
 default-router 192.168.1.1
 domain-name ccna-lab.com
 lease 2 12 30
!
ip dhcp pool R2_Clients_LAN
 network 192.168.1.96 255.255.255.240
 default-router 192.168.1.97
 domain-name ccna-lab.com
 lease 2 12 30
```

#### Производим проверки

```
R1#show ip dhcp pool

Pool R1_Clients_LAN :
 Utilization mark (high/low)    : 100 / 0
 Subnet size (first/next)       : 0 / 0
 Total addresses                : 62
 Leased addresses               : 0
 Pending event                  : none
 1 subnet is currently in the pool :
 Current index        IP address range                    Leased addresses
 192.168.1.1          192.168.1.1      - 192.168.1.62      0

Pool R2_Clients_LAN :
 Utilization mark (high/low)    : 100 / 0
 Subnet size (first/next)       : 0 / 0
 Total addresses                : 14
 Leased addresses               : 0
 Pending event                  : none
 1 subnet is currently in the pool :
 Current index        IP address range                    Leased addresses
 192.168.1.97         192.168.1.97     - 192.168.1.110     0
```

```
R1#show ip dhcp binding
Bindings from all pools not associated with VRF:
IP address          Client-ID/              Lease expiration        Type
                    Hardware address/
                    User name
```

```
R1#show ip dhcp server statistics
Memory usage         25177
Address pools        2
Database agents      0
Automatic bindings   0
Manual bindings      0
Expired bindings     0
Malformed messages   0
Secure arp entries   0

Message              Received
BOOTREQUEST          0
DHCPDISCOVER         0
DHCPREQUEST          0
DHCPDECLINE          0
DHCPRELEASE          0
DHCPINFORM           0

Message              Sent
BOOTREPLY            0
DHCPOFFER            0
DHCPACK              0
DHCPNAK              0
```

#### Попытка получить IP-адрес по DHCP на PC-A

```
PC-A> show ip

NAME        : PC-A[1]
IP/MASK     : 0.0.0.0/0
GATEWAY     : 0.0.0.0
DNS         :
<...>

PC-A> ip dhcp
DDORA IP 192.168.1.6/26 GW 192.168.1.1

PC-A> show ip

NAME        : PC-A[1]
IP/MASK     : 192.168.1.6/26
GATEWAY     : 192.168.1.1
DNS         :
DHCP SERVER : 192.168.1.1
DHCP LEASE  : 217795, 217800/108900/190575
DOMAIN NAME : ccna-lab.com
<...>
```

Проверяем связность

```
PC-A> ping 192.168.1.1

84 bytes from 192.168.1.1 icmp_seq=1 ttl=255 time=0.301 ms
84 bytes from 192.168.1.1 icmp_seq=2 ttl=255 time=0.435 ms
^C
```

### Часть 3: Настройка и проверка DHCP-ретранслятора на R2 

Настраиваем хелпер на интерфейсе в сторону локальной сети, указываем адрес стыкового интерфейса на R1. 

```
R2(config)#int e 0/1
R2(config-if)#ip helper-address 10.0.0.1
```

Запрашиваем адрес с PC-B

```
PC-B> show ip

NAME        : PC-B[1]
IP/MASK     : 0.0.0.0/0
GATEWAY     : 0.0.0.0
DNS         :
<...>

PC-B> ip dhcp
DDORA IP 192.168.1.102/28 GW 192.168.1.97

PC-B> show ip

NAME        : PC-B[1]
IP/MASK     : 192.168.1.102/28
GATEWAY     : 192.168.1.97
DNS         :
DHCP SERVER : 10.0.0.1
DHCP LEASE  : 217796, 217800/108900/190575
DOMAIN NAME : ccna-lab.com
<...>
```

Проверяем связность, пинг к стыковому интерфейсу на R1

```
PC-B> ping 192.168.1.1

84 bytes from 192.168.1.1 icmp_seq=1 ttl=254 time=0.416 ms
84 bytes from 192.168.1.1 icmp_seq=2 ttl=254 time=0.520 ms
^C
```

Проведем проверки

```
R1#show ip dhcp binding
Bindings from all pools not associated with VRF:
IP address          Client-ID/              Lease expiration        Type
                    Hardware address/
                    User name
192.168.1.6         0100.5079.6668.03       Apr 24 2024 11:07 AM    Automatic
192.168.1.102       0100.5079.6668.04       Apr 24 2024 11:20 AM    Automatic
```

```
R1#show ip dhcp server statistics
Memory usage         42095
Address pools        2
Database agents      0
Automatic bindings   2
Manual bindings      0
Expired bindings     0
Malformed messages   0
Secure arp entries   0

Message              Received
BOOTREQUEST          0
DHCPDISCOVER         4
DHCPREQUEST          2
DHCPDECLINE          0
DHCPRELEASE          0
DHCPINFORM           0

Message              Sent
BOOTREPLY            0
DHCPOFFER            2
DHCPACK              2
DHCPNAK              0
```
