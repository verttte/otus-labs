### Схема лабораторного стенда

![image](https://github.com/user-attachments/assets/b11d9264-f283-4678-882a-ded6e994923e)


### Настроить фильтрацию в офисе Москва так, чтобы не появилось транзитного трафика (As-path)

На R22 выводим таблицу BGP и видим, что префиксы от AS 2042 доступны через AS 1001.

![image](https://github.com/user-attachments/assets/148da2b9-9ca2-473f-bad5-c28f2bdb420c)

С помощью as-path ACL ограничим передаваемую соседям информацию.  
На R14 и R15 создаем необходимый ACL  
```
ip as-path access-list 10 permit ^$
```
Регулярное выражение `^$` отбирает префиксы с пустой строкой. Это значит, что под правило попадут локально созданные префиксы.  
Затем мы применяем фильтр к соседям в направлении `out`.

```
R14(config)#router bgp 1001
R14(config-router)#neighbor 10.10.100.1 filter-list 10 out
```
```
R15(config)#router bgp 1001
R15(config-router)#neighbor 10.10.100.5 filter-list 10 out
```

Тем самым соседи получают информацию о префиксах только из AS 1001 и не смогут отправлять трафик, предназначенный для AS 2042 через AS 1001.

Проверим таблицу BGP  
![01](https://github.com/user-attachments/assets/ce091867-5d1c-4056-be63-f5cdee7e2c39)


Для префиксов из AS 2042 больше нет next-hop через AS 1001

### Настроить фильтрацию в офисе С.-Петербург так, чтобы не появилось транзитного трафика (Prefix-list)

Создадим на R18 prefix-list, перечислив все префиксы, которые мы анонсируем нашим соседям

```
ip prefix-list FILTER-OUT seq 10 permit 10.10.18.0/29
ip prefix-list FILTER-OUT seq 20 permit 10.10.50.18/32
ip prefix-list FILTER-OUT seq 30 permit 10.10.100.20/30
ip prefix-list FILTER-OUT seq 40 permit 10.10.100.24/30
```

Применим фильтр к соседям в направлении `out`

```
R18(config)#router bgp 2042
R18(config-router)#neighbor 10.10.100.21 prefix-list FILTER-OUT out
R18(config-router)#neighbor 10.10.100.25 prefix-list FILTER-OUT out
```


### Настроить провайдера Киторн так, чтобы в офис Москва отдавался только маршрут по умолчанию

Ранее маршрут по умолчанию был настроен статикой, удаляем его  
```
R14(config)#no ip route 0.0.0.0 0.0.0.0 10.10.100.1
```
Проверим анонсы от Киторн R22  
![image](https://github.com/user-attachments/assets/a7140703-b2b1-4ed7-80ba-fdf86afe9a02)


Настраиваем на маршрутизаторе в Киторн анонсирование маршрута по умолчанию соседу R14
```
R22(config)#router bgp 101
R22(config-router)#neighbor 10.10.100.2 default-originate
```
В анонсах появился маршрут по умолчанию  
![01](https://github.com/user-attachments/assets/78c96b70-506c-4e0c-9faa-a1e38278268f)


Чтоб не передавать всю другую информацию создадим карту маршрутов, в которой будет единственное запрещающее всё правило и применим её по направлению к соседу  
```
R22(config)#route-map MSC-R14-OUT deny 10
R22(config)#router bgp 101
R22(config-router)#neighbor 10.10.100.2 route-map MSC-R14-OUT out
```

После этого в таблице BGP остается только маршрут по умолчанию  
![image](https://github.com/user-attachments/assets/dbd879b8-41ee-413e-813f-59bab232a617)

### Настроить провайдера Ламас так, чтобы в офис Москва отдавался только маршрут по умолчанию и префикс офиса С.-Петербург

Удаляем настроенный ранее статический маршрут по умолчанию
```
R15(config)#no ip route 0.0.0.0 0.0.0.0 10.10.100.5
```

Для тестирования создадим маршрут и проанонсируем сеть на R24  
```
R24(config)#ip route 24.24.24.24 255.255.255.255 null 0
R24(config)#router bgp 520
R24(config-router)#network 24.24.24.24 mask 255.255.255.255
```
Проверим, что мы получаем в Москве на R15 - информация от Триады, от С.-Петербурга, от Ламас и нет маршрута по умолчанию  
![image](https://github.com/user-attachments/assets/60f9a588-3a6b-47c5-b98e-379e3deada51)

Настраиваем на маршрутизаторе в Ламас анонсирование маршрута по умолчанию соседу R15

```
R21(config)#router bgp 301
R21(config-router)#neighbor 10.10.100.6 default-originate
```
Информация о маршруте по умолчанию появилась в таблице  
![image](https://github.com/user-attachments/assets/57f982b0-f594-44b2-8ad5-1bdf5320dfe2)

Теперь нам нужно сделать так, чтоб передавался префикс С.-Петербурга, а остальные не передавались. Создадим as-path ACL, который разрешает префиксы, произошедшие в AS 2042. Добавим его в карту маршрутов. Неявное правило deny в карте маршрутов запретит все остальные. И применим карту к соседу по направлению на выход.
```
R21(config)#ip as-path access-list 20 permit _2042$
R21(config)#route-map MSC-R15-OUT permit 10
R21(config-route-map)#match as-path 20
R21(config)
R21(config)#router bgp 301
R21(config-router)#neighbor 10.10.100.6 route-map MSC-R15-OUT out
```

Проверим. Префикс из AS 520 больше не присутствует в таблице. Только маршрут по умолчанию и префиксы из AS 2042. 
![image](https://github.com/user-attachments/assets/487a17b2-2c70-4f97-ab23-4a0d6362aedc)

