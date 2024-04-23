Схема лабораторного стенда

![image](https://github.com/verttte/otus-labs/assets/165086553/491052cd-f67c-41d7-8e0c-6fb80f3040e9)


Таблица адресации

![image](https://github.com/verttte/otus-labs/assets/165086553/4a1fce24-c2b4-41ba-9e40-59b1359735cd)



#### Цели:

Часть 1: Настройка основных параметров устройства

Часть 2: Проверка назначения адреса SLAAC на R1

Часть 3: Настройка и проверка сервера DHCPv6 без отслеживания состояния на R1

Часть 4: Настройка и проверка сервера DHCPv6 с отслеживанием состояния на R1

Часть 5: Настройка и проверка ретранслятора DHCPv6 на R2

#### Часть 1: Настройка основных параметров устройства

Произведены базовые настройки устройств в соответствии с методичкой.

Дополнительно включаем маршрутизацию IPv6 командой `ipv6 unicast-routing`.

Настраиваем адреса на интерфейсах e0/0 и e0/1 в соответствии с таблицей.

Команда `ipv6 address <ipv6_addr>/<prefix>`

Настраиваем маршрут по умолчанию на каждом маршрутизаторе, указывающем на IP-адрес e0/0 на другом маршрутизаторе - `ipv6 route ::/0 <ip_addr>`.

Чтоб убедиться, что маршрутизация работает, отправляем запрос с R1 на адрес, настроенный на R2 e0/1:

```
R1#ping 2001:DB8:ACAD:3::1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 2001:DB8:ACAD:3::1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms
```

#### Часть 2: Проверка назначения адреса SLAAC на R1

PC-A настроен на автоматическое получение адреса. Проверяем его настройки.

```
C:\>ipv6config /all

FastEthernet0 Connection:(default port)

   Connection-specific DNS Suffix..: 
   Physical Address................: 0060.5C06.55D0
   Link-local IPv6 Address.........: FE80::260:5CFF:FE06:55D0
   IPv6 Address....................: 2001:DB8:ACAD:1:260:5CFF:FE06:55D0
   Default Gateway.................: FE80::201:C9FF:FEE6:2502
   DNS Servers.....................: ::
   DHCPv6 IAID.....................: 1108484923
   DHCPv6 Client DUID..............: 00-01-00-01-C3-D7-61-94-00-60-5C-06-55-D0
```

```diff
Вопрос из методички
- Откуда взялась часть адреса с идентификатором хоста?
```

#### Часть 3: Настройка и проверка сервера DHCPv6 без отслеживания состояния на R1

```
C:\>ipv6config /all

FastEthernet0 Connection:(default port)

   Connection-specific DNS Suffix..: 
   Physical Address................: 0060.5C06.55D0
   Link-local IPv6 Address.........: FE80::260:5CFF:FE06:55D0
   IPv6 Address....................: 2001:DB8:ACAD:1:260:5CFF:FE06:55D0
   Default Gateway.................: FE80::201:C9FF:FEE6:2502
   DNS Servers.....................: ::
   DHCPv6 IAID.....................: 1108484923
   DHCPv6 Client DUID..............: 00-01-00-01-C3-D7-61-94-00-60-5C-06-55-D0
```

Отсутствует информация о DNS.

Настроим R1 для предоставления DHCPv6 без сохранения состояния для PC-A.

```
R1(config)#ipv6 dhcp pool R1-STATELESS
R1(config-dhcpv6)#dns-server 2001:DB8:ACAD::254
R1(config-dhcpv6)#domain-name STATELESS.com
R1(config-dhcpv6)#exit
R1(config)#interface e0/1
R1(config-if)#ipv6 nd other-config-flag
R1(config-if)#ipv6 dhcp server R1-STATELESS
```

Проверим настройки на PC-A

```
C:\>ipv6config /all

FastEthernet0 Connection:(default port)

   Connection-specific DNS Suffix..: STATELESS.com 
   Physical Address................: 0060.5C06.55D0
   Link-local IPv6 Address.........: FE80::260:5CFF:FE06:55D0
   IPv6 Address....................: 2001:DB8:ACAD:1:260:5CFF:FE06:55D0
   Default Gateway.................: FE80::201:C9FF:FEE6:2502
   DNS Servers.....................: 2001:DB8:ACAD::254
   DHCPv6 IAID.....................: 1108484923
   DHCPv6 Client DUID..............: 00-01-00-01-C3-D7-61-94-00-60-5C-06-55-D0
```

Протестируем подключение, отправив запрос на IP-адрес интерфейса R2 e0/1. 

```
C:\>ping 2001:db8:acad:3::1

Pinging 2001:db8:acad:3::1 with 32 bytes of data:

Reply from 2001:DB8:ACAD:3::1: bytes=32 time<1ms TTL=254
Reply from 2001:DB8:ACAD:3::1: bytes=32 time<1ms TTL=254
```

#### Часть 4: Настройка и проверка сервера DHCPv6 с отслеживанием состояния на R1



#### Часть 5: Настройка и проверка ретранслятора DHCPv6 на R2


