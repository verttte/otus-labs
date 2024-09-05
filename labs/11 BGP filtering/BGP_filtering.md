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

### Настроить провайдера Ламас так, чтобы в офис Москва отдавался только маршрут по умолчанию и префикс офиса С.-Петербург

