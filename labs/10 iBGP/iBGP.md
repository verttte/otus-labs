### Схема лабораторного стенда

![image](https://github.com/user-attachments/assets/cc808e77-1b48-4e9f-98f0-ab2c74dddb61)

На каждом маршрутизаторе настроен loopback интерфейс. IP-адрес взят в соответствии с IP-планом из сети 10.10.50.0/24, последний октет - номер маршрутизатора.

#### Настройка iBGP в офисе Москва между маршрутизаторами R14 и R15

Для настройки iBGP, при объявлении соседа указывается, что он находится в той же AS, что и мы.

R14  
```
router bgp 1001
neighbor 10.10.50.15 remote-as 1001
neighbor 10.10.50.15 description R15
neighbor 10.10.50.15 update-source Loopback0
```

R15  
```
router bgp 1001
neighbor 10.10.50.14 remote-as 1001
neighbor 10.10.50.14 description R15
neighbor 10.10.50.14 update-source Loopback0
```

### Настройка iBGP в провайдере Триада, с использованием RR

В качестве RR выберем R25.
Все клиенты должны иметь сессию с RR, с остальными клиентами сессий быть не должно.
Для удобства настройки на R25 объединим всех клиентов в пир-группу. На RR надо указать кто является клиентом. На самих клиентах указывать кто сервер не надо.
Так же укажем, чтоб каждый маршрутизатор передавал в качестве next-hop свой IP-адрес.

R25  
```
router bgp 520
bgp router-id 10.10.50.25
neighbor AS520 peer-group
neighbor AS520 remote-as 520
neighbor AS520 update-source Loopback0
neighbor AS520 route-reflector-client
neighbor 10.10.50.23 peer-group AS520
neighbor 10.10.50.24 peer-group AS520
neighbor 10.10.50.26 peer-group AS520
```

R23  
```
router bgp 520
bgp router-id 10.10.50.23
neighbor 10.10.50.25 remote-as 520
neighbor 10.10.50.25 description R25
neighbor 10.10.50.25 update-source Loopback0
neighbor 10.10.50.25 next-hop-self
```

R24  
```
router bgp 520
bgp router-id 10.10.50.24
neighbor 10.10.50.25 remote-as 520
neighbor 10.10.50.25 description R25
neighbor 10.10.50.25 update-source Loopback0
neighbor 10.10.50.25 next-hop-self
```

R26  
```
router bgp 520
bgp router-id 10.10.50.26
neighbor 10.10.50.25 remote-as 520
neighbor 10.10.50.25 description R25
neighbor 10.10.50.25 update-source Loopback0
neighbor 10.10.50.25 next-hop-self
```

На примере R26 убеждаемся, что сессию он имеет только с R25, но получает информацию, которая пришла с R24.

`show ip bgp summary`:  
![image](https://github.com/user-attachments/assets/64cbb79d-bb5a-4773-b1d5-39191b258e22)

`show ip bgp`:  
![image](https://github.com/user-attachments/assets/3c78213e-b6f3-45ac-bd74-e2e6c622877a)


### Настройка офиса Москва так, чтобы приоритетным провайдером стал Ламас

Необходимо настроить как исходящий, так и входящий трафик. 

Для исходящего будем использовать атрибут Local preference, так как он передается внутри AS и указывает маршрутизаторам внутри неё как выйти за её
пределы.

Задается этот атрибут с помощью route-map. Создаем его и применяем настройку с соседу по направлению на прием.

R15  
```
route-map LP-SET permit 10
set local-preference 110
exit
router bgp 1001
neighbor 10.10.100.5 description LAMAS
neighbor 10.10.100.5 route-map LP-SET in
```

Теперь вся информация, полученная от соседа имет Local preference 110 и передается на R14

`show ip bgp`:  
![image](https://github.com/user-attachments/assets/443a65a0-88df-44bd-9e44-9ae9b1963bc1)

Для настройки пути входящего трафика будем использовать атрибут AS path. С его помощью удлинним путь через соединение Киторн - Москва R14, путь Ламас - Москва R15 будет более выгодным.

Настроим route-map. Применим настройку с соседу по направлению на выход.

R14
```
route-map KITORN-OUT permit 10
set as-path prepend 1001 1001
exit
router bgp 1001
neighbor 10.10.100.1 description KITORN
neighbor 10.10.100.1 route-map KITORN-OUT out
```
В выводе информации BGP убеждаемся что пусть стал длинее
`show ip bgp`:  
![image](https://github.com/user-attachments/assets/6265d22c-e2c1-4ec0-9a87-d4cce6fad43f)


### Настройка офиса С.-Петербург так, чтобы трафик до любого офиса распределялся по двум линкам одновременно

Для этого требуется настроить BGP Multipath. Условием работы BGP Multipath является совпадение AS path, а соответственно необходимо, чтоб маршрутизатор был подключен к двумя линками к одному и тому же аплинку. В нашем случае это так.

Сперва в таблице маршрутизации присутствуют только "бесты"

`show ip bgp`:  
![image](https://github.com/user-attachments/assets/b280bb43-8429-469d-9a4d-fad10c8544e9)  
`show ip route bgp`:  
![image](https://github.com/user-attachments/assets/3f92dd3c-5488-492f-9a84-587902caae59)

Производим настройку multipath
```
R18(config)#router bgp 2042
R18(config-router)#maximum-paths 2
R18(config-router)#bgp bestpath as-path multipath-relax
```
В таблице маршрутизации появляются двойные маршруты и трафик балансируется между ними

`show ip route bgp`:  
![image](https://github.com/user-attachments/assets/ca6330bb-2ec0-4c9f-bc79-ed5f5355953d)
