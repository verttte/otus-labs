#### Схема лабораторного стенда


#### 1. В офисе С.-Петербург настроить EIGRP.

Настройка будет производиться в режиме Named EIGRP с преминением аутентификации.

Для R18
```
router eigrp AS2042
 !
 address-family ipv4 unicast autonomous-system 2042
  !
  af-interface Ethernet0/1
   authentication mode hmac-sha-256 otus-lab
  exit-af-interface
  !
  af-interface Ethernet0/0
   authentication mode hmac-sha-256 otus-lab
  exit-af-interface
  !
  topology base
  exit-af-topology
  network 10.10.202.0 0.0.0.255
  eigrp router-id 10.10.50.18
 exit-address-family

```

```
```

#### 2. R32 получает только маршрут по умолчанию.

```
```

```
```
#### 3. R16-17 анонсируют только суммарные префиксы.
