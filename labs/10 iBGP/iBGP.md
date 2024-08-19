#### Схема лабораторного стенда

![image](https://github.com/user-attachments/assets/cc808e77-1b48-4e9f-98f0-ab2c74dddb61)

На каждом маршрутизаторе настроен loopback интерфейс. IP-адрес взят в соответствии с IP-планом из сети 10.10.50.0/24, последний октет - номер маршрутизатора.

#### Настройка iBGP в офисе Москва между маршрутизаторами R14 и R15.

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


#### Настроите iBGP в провайдере Триада, с использованием RR.
#### Настройте офис Москва так, чтобы приоритетным провайдером стал Ламас.
#### Настройте офис С.-Петербург так, чтобы трафик до любого офиса распределялся по двум линкам одновременно

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
