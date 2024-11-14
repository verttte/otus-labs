### Схема лабораторного стенда

![image](https://github.com/user-attachments/assets/87becba8-bbc2-462b-8960-b3b521ca840b)

### Настройка CA, создание сертификатов

В качетве CA будет выступать R14

```
R14(config)#ip domain name otus.ru
R14(config)#ip http server
R14(config)#crypto key generate rsa general-keys label R14CA exportable modulus 4096
R14(config)#crypto pki server R14CA
R14(cs-server)#no shut
R14(cs-server)#exit
```

R15  
```
R15(config)#crypto key generate rsa label VPN modulus 4096
R15(config)#crypto pki trustpoint VPN
R15(ca-trustpoint)#enrollment url http://10.10.50.14
R15(ca-trustpoint)#subject-name CN=R15, OU=VPN, O=MSK, C=RU
R15(ca-trustpoint)#rsakeypair VPN
R15(ca-trustpoint)#revocation-check none
R15(ca-trustpoint)#exit
R15(config)#crypto pki authenticate VPN
R15(config)#crypto pki enroll VPN
```
R18  
```
R18(config)#crypto key generate rsa label VPN modulus 4096
R18(config)#crypto pki trustpoint VPN
R18(ca-trustpoint)#enrollment url http://10.10.50.14
R18(ca-trustpoint)#subject-name CN=R18, OU=VPN, O=SPB, C=RU
R18(ca-trustpoint)#rsakeypair VPN
R18(ca-trustpoint)#revocation-check none
R18(ca-trustpoint)#exit
R18(config)#crypto pki authenticate VPN
R18(config)#crypto pki enroll VPN
```

Подписываем сертификаты на R14  
```
R14#show crypto pki server R14CA requests

<...>

Router certificates requests:
ReqID  State      Fingerprint                      SubjectName
--------------------------------------------------------------
3      pending    7EACEF59EF80BE2CBF10E4F633E1905A hostname=R15,cn=R15,ou=VPN,o=MSK,c=RU
2      pending    ADE7603B384EC983E8BACA123C9C5458 hostname=R18,cn=R18,ou=VPN,o=SPB,c=RU
```
```
R14#crypto pki server R14CA grant all
R14#show crypto pki server R14CA requests

<...>

Router certificates requests:
ReqID  State      Fingerprint                      SubjectName
--------------------------------------------------------------
3      granted    7EACEF59EF80BE2CBF10E4F633E1905A hostname=R15,cn=R15,ou=VPN,o=MSK,c=RU
2      granted    ADE7603B384EC983E8BACA123C9C5458 hostname=R18,cn=R18,ou=VPN,o=SPB,c=RU
```

### Настройка GRE поверх IPSec между офисами Москва и С.-Петербург

```

```

```

```

### Настройка DMVPN поверх IPSec между Москва и Чокурдах, Лабытнанги

```

```

```

```
