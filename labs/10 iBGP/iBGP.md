#### Схема лабораторного стенда

#### Настроите iBGP в офисом Москва между маршрутизаторами R14 и R15.
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
