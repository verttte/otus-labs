### Схема лабораторного стенда

### Настроить фильтрацию в офисе Москва так, чтобы не появилось транзитного трафика (As-path)

На R22 выводим таблицу BGP и видим что префиксы от AS 2042 доступны через AS 1001.

![image](https://github.com/user-attachments/assets/2f5a6955-aa4d-4301-962a-7e432f6e38be)

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
![image](https://github.com/user-attachments/assets/99086f29-340e-4511-8eba-5473bbbc12a9)

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
![image](https://github.com/user-attachments/assets/c3c01727-1c3b-4ea5-a000-1d12e3e5a1c7)

Настраиваем на маршрутизаторе в Киторн анонсирование маршрута по умолчанию соседу R14
```
R22(config)#router bgp 101
R22(config-router)#neighbor 10.10.100.2 default-originate
```
В анонсках появился маршрут по умолчанию  
![image](https://github.com/user-attachments/assets/74c5e758-e5dd-476b-a5f9-82e72e97a5d1)

Чтоб не передавать всю другую информацию создадим карту маршрутов, в которой будет единственное запрещающее всё правило и применим её по направлению к соседу  
```
R22(config)#route-map MSC-R14-OUT deny 10
R22(config)#router bgp 101
R22(config-router)#neighbor 10.10.100.2 route-map MSC-R14-OUT out
```

После этого в таблице BGP остается только маршрут по умолчанию  
![image](https://github.com/user-attachments/assets/22c76dd7-9bbd-4019-a90c-4e47aab33b66)



### Настроить провайдера Ламас так, чтобы в офис Москва отдавался только маршрут по умолчанию и префикс офиса С.-Петербург

