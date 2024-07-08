#### Схема лабораторного стенда

![image](https://github.com/verttte/otus-labs/assets/165086553/7acfec58-cf33-4e89-813f-82e7d09b91a6)

#### Настройка IS-IS в ISP Триада

Задаем настройки протокола IS-IS. NET адрес указывает принадлежность к той или иной зоне. Указываем, что по умолчанию работа протокола на всех интерфесах отключена. Затем включаем его работу только на интерфейсах в сторону тех маршрутизаторов, с которыми будем устанавливать соседство.

#### R23 и R25 находятся в зоне 2222

```
R23(config)#router isis
R23(config-router)#net 49.2222.2323.2323.2323.00
R23(config-router)#passive-interface default
R23(config-router)#exit
R23(config)#interface Ethernet0/1
R23(config-if)#ip router isis
R23(config-if)#interface Ethernet0/2
R23(config-if)#ip router isis
```

```
R25(config)#router isis
R25(config-router)#net 49.2222.2525.2525.2525.00
R25(config-router)#passive-interface default
R25(config-router)#exit
R25(config)#interface Ethernet0/0
R25(config-if)#ip router isis
R25(config-if)#interface Ethernet0/2
R25(config-if)#ip router isis
```

#### R24 находится в зоне 24

```
R24(config)#router isis
R24(config-router)#net 49.0024.2424.2424.2424.00
R24(config-router)#passive-interface default
R24(config-router)#exit
R24(config)#interface Ethernet0/1
R24(config-if)#ip router isis
R24(config-if)#interface Ethernet0/2
R24(config-if)#ip router isis

```

#### R26 находится в зоне 26

```
R26(config)#router isis
R26(config-router)#net 49.0026.2626.2626.2626.00
R26(config-router)#passive-interface default
R26(config-router)#exit
R26(config)#interface Ethernet0/0
R26(config-if)#ip router isis
R26(config-if)#interface Ethernet0/2
R26(config-if)#ip router isis
```

#### Проверка

Командами `show isis neighbors`, `show isis database` и `sh ip route` проверяем работу протокола

![image](https://github.com/verttte/otus-labs/assets/165086553/24029922-f6b2-4bb9-b4ee-e358e36053f3)

![image](https://github.com/verttte/otus-labs/assets/165086553/7fe9dcf4-5297-4321-88ad-b61de1edc6a0)

![image](https://github.com/verttte/otus-labs/assets/165086553/445b4e67-29a8-483c-a527-29c738a79e9a)

![image](https://github.com/verttte/otus-labs/assets/165086553/fe9ae259-b773-476e-acac-6705c53876fd)






