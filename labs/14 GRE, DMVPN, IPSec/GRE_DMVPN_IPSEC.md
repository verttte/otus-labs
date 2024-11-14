### Схема лабораторного стенда

![image](https://github.com/user-attachments/assets/87becba8-bbc2-462b-8960-b3b521ca840b)

### Настройка CA

В качетве CA будет выступать R14

```
R14(config)#ip domain name otus.ru
R14(config)#ip http server
R14(config)#crypto key generate rsa general-keys label R14CA exportable modulus 4096
R14(config)#crypto pki server R14CA
R14(cs-server)#no shut
R14(cs-server)#exit
```

### Настройка GRE поверх IPSec между офисами Москва и С.-Петербург

### Настройка DMVPN поверх IPSec между Москва и Чокурдах, Лабытнанги
