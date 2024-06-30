#### Схема лабораторного стенда

![image](https://github.com/verttte/otus-labs/assets/165086553/b48363e9-9898-4249-a031-c04b40141b67)

#### 1. В офисе С.-Петербург настроить EIGRP.

Настройка будет производиться в режиме Named EIGRP с преминением аутентификации.

Для R18 указываем роутер ID, указываем сети, в которых будет работать EIGRP. Сети здесь присутствуют только стыковой адресации.
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
!
```

Для R17 указываются сети стыковые, а так же пользователей и управления. У нас нет необходимости в отправке пакетов EIGRP с интерфесов с пользователей и управления, так как там не предполагается соседей. Поэтому переводим их в режим passive. Заодно преднастраиваем остальные интерфесы без стыков в passive.
```
router eigrp AS2042
 !
 address-family ipv4 unicast autonomous-system 2042
  !
  af-interface Ethernet0/0
   passive-interface
  exit-af-interface
  !
  af-interface Ethernet0/2
   passive-interface
  exit-af-interface
  !
  af-interface Ethernet0/3
   passive-interface
  exit-af-interface
  !
  af-interface Ethernet0/0.2
   passive-interface
  exit-af-interface
  !
  af-interface Ethernet0/0.80
   passive-interface
  exit-af-interface
  !
  af-interface Ethernet0/0.90
   passive-interface
  exit-af-interface
  !
  af-interface Ethernet0/1
   authentication mode hmac-sha-256 otus-lab
  exit-af-interface
  !
  topology base
  exit-af-topology
  network 10.10.202.0 0.0.0.255
  network 172.16.1.0 0.0.0.255
  network 192.168.80.0
  network 192.168.90.0
  eigrp router-id 10.10.50.17
 exit-address-family
!
```

Для R16 настройка аналогична R17. Дополнительно настройка соседства с R32.
```
router eigrp AS2042
 !
 address-family ipv4 unicast autonomous-system 2042
  !
  af-interface Ethernet0/0
   passive-interface
  exit-af-interface
  !
  af-interface Ethernet0/0.2
   passive-interface
  exit-af-interface
  !
  af-interface Ethernet0/0.80
   passive-interface
  exit-af-interface
  !
  af-interface Ethernet0/0.90
   passive-interface
  exit-af-interface
  !
  af-interface Ethernet0/2
   passive-interface
  exit-af-interface
  !
  af-interface Ethernet0/1
   authentication mode hmac-sha-256 otus-lab
  exit-af-interface
  !
  af-interface Ethernet0/3
   authentication mode hmac-sha-256 otus-lab
  exit-af-interface
  !
  topology base
  exit-af-topology
  network 10.10.202.0 0.0.0.255
  network 172.16.1.0 0.0.0.255
  network 192.168.80.0
  network 192.168.90.0
  eigrp router-id 10.10.50.16
 exit-address-family
!
```

На R32 настройка соседства с R16.
```
router eigrp AS2042
 !
 address-family ipv4 unicast autonomous-system 2042
  !
  af-interface Ethernet0/1
   passive-interface
  exit-af-interface
  !
  af-interface Ethernet0/2
   passive-interface
  exit-af-interface
  !
  af-interface Ethernet0/3
   passive-interface
  exit-af-interface
  !
  af-interface Ethernet0/0
   authentication mode hmac-sha-256 otus-lab
  exit-af-interface
  !
  topology base
  exit-af-topology
  network 10.10.202.0 0.0.0.255
  eigrp router-id 10.10.50.32
 exit-address-family
!
```
После этого по всей сети начинает распространяться маршрутная информация.

![image](https://github.com/verttte/otus-labs/assets/165086553/41daa0fe-871f-4c2d-a6ba-e0d603203ba6)

#### 2. R32 получает только маршрут по умолчанию.

Для передачи маршрута по умолчанию соседям необходимо объявить его на каком-либо из роутеров. Сделаем это на R18. Так как R18 имеет два линка во внешнюю сеть, то и маршрутов будет два.
```
ip route 0.0.0.0 0.0.0.0 10.10.100.21
ip route 0.0.0.0 0.0.0.0 10.10.100.25
```

Затем настраиваем редистрибьюцию статических маршрутов в протокол EIGRP.
```
R18(config)#router eigrp AS2042
R18(config-router)#address-family ipv4 unicast autonomous-system 2042
R18(config-router-af)#topology base
R18(config-router-af-topology)#redistribute static
```

И маршрут по умолчанию появляется в таблицах остальных роутеров.
![image](https://github.com/verttte/otus-labs/assets/165086553/30c919a1-9b83-4675-98f1-4661362b7085)

Для того, чтоб R32 получал только маршрут по умолчанию необходимо настроить фильтрацию. На R16 создадим префикс лист, который отберет только маршрут по умолчанию и укажем его в конфигурации протокола на направлению в сторону R32.

```
R16(config)#ip prefix-list TO_R32 seq 5 permit 0.0.0.0/0
R16(config)#router eigrp AS2042
R16(config-router)#address-family ipv4 unicast autonomous-system 2042
R16(config-router-af)#topology base
R16(config-router-af-topology)#distribute-list prefix TO_R32 out Ethernet0/3
```

После этого в таблице маршрутизации R32 остается только маршрут по умолчанию
![image](https://github.com/verttte/otus-labs/assets/165086553/1611811b-5781-4e0f-bea9-240b18be8f50)




#### 3. R16-17 анонсируют только суммарные префиксы.

В нашем случае удобно суммировать пользовательские сети с префиксом /24 в одну с префиксом /20. Для этого вводим соответствующие команды на R16 и R17. Они одинаковые для обоих маршрутизаторов, номер интерфейса тоже совпадает. 
```
(config)#router eigrp AS2042
(config-router)# address-family ipv4 unicast autonomous-system 2042
(config-router-af)#af-interface Ethernet0/1
(config-router-af-interface)#summary-address 192.168.80.0 255.255.240.0
```

Таблица маршрутизации на вышестоящем роутере до ввода команд
![image](https://github.com/verttte/otus-labs/assets/165086553/ace158c6-a270-4908-9d4b-83fde196b698)

Таблица маршрутизации после
![image](https://github.com/verttte/otus-labs/assets/165086553/bc508507-96d5-490b-b37f-f5c92a5a9be2)

Вместо двух сетей /24 одна сеть /20.
