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
Проверяем
<details><summary>R15</summary>

```
R15#show crypto pki certificates
Certificate
  Status: Available
  Certificate Serial Number (hex): 02
  Certificate Usage: General Purpose
  Issuer:
    cn=R14CA
  Subject:
    Name: R15
    hostname=R15
    cn=R15
    ou=VPN
    o=MSK
    c=RU
  Validity Date:
    start date: 23:32:02 Moscow Nov 14 2024
    end   date: 23:32:02 Moscow Nov 14 2025
  Associated Trustpoints: VPN

CA Certificate
  Status: Available
  Certificate Serial Number (hex): 01
  Certificate Usage: Signature
  Issuer:
    cn=R14CA
  Subject:
    cn=R14CA
  Validity Date:
    start date: 21:01:48 Moscow Nov 14 2024
    end   date: 21:01:48 Moscow Nov 14 2027
  Associated Trustpoints: VPN
  Storage: nvram:R14CA#1CA.cer


R15#
```
</details>

<details><summary>R18</summary>

```
R18#show crypto pki certificates
CA Certificate
  Status: Available
  Certificate Serial Number (hex): 01
  Certificate Usage: Signature
  Issuer:
    cn=R14CA
  Subject:
    cn=R14CA
  Validity Date:
    start date: 21:01:48 MSK Nov 14 2024
    end   date: 21:01:48 MSK Nov 14 2027
  Associated Trustpoints: VPN
  Storage: nvram:R14CA#1CA.cer


Certificate
  Subject:
    Name: R18
   Status: Pending
   Key Usage: General Purpose
   Certificate Request Fingerprint MD5: ADE7603B 384EC983 E8BACA12 3C9C5458
   Certificate Request Fingerprint SHA1: C2285675 D6C31812 53BCFA45 842DA65B CFE1555F
   Associated Trustpoint: VPN

R18#
```
</details>

### Настройка GRE поверх IPSec между офисами Москва и С.-Петербург

Настройки IPSec на R15 и R18 одинаковые для обоих маршрутизаторов.  
```
(config)#crypto ikev2 proposal PH1
(config-ikev2-proposal)#encryption aes-cbc-128
(config-ikev2-proposal)#integrity sha256
(config-ikev2-proposal)#group 2
(config-ikev2-proposal)#exit
(config)#crypto ikev2 policy IK2POL
(config-ikev2-policy)#proposal PH1
(config-ikev2-policy)#exit
(config)#crypto ikev2 profile PROF1
(config-ikev2-profile)#match address local interface Loopback0
(config-ikev2-profile)#match identity remote address 0.0.0.0
(config-ikev2-profile)#authentication remote rsa-sig
(config-ikev2-profile)#authentication local rsa-sig
(config-ikev2-profile)#pki trustpoint VPN
(config-ikev2-profile)#exit
(config)#crypto ipsec transform-set IPSEC_TS esp-aes esp-sha256-hmac
(cfg-crypto-trans)#mode transport
(cfg-crypto-trans)#exit
(cfg-crypto-trans)#crypto ipsec profile protect-gre
(ipsec-profile)#set transform-set IPSEC_TS
(ipsec-profile)#set ikev2-profile PROF1
(ipsec-profile)#exit
(config)#interface Tunnel1518
(config-if)#tunnel protection ipsec profile protect-gre
(config-if)#exit
```

Проверяем   
<details><summary>R15</summary>

```
R15#sho crypto ikev2 sa
 IPv4 Crypto IKEv2  SA

Tunnel-id Local                 Remote                fvrf/ivrf            Status
1         10.10.50.15/500         10.10.50.18/500         none/none            READY  
      Encr: AES-CBC, keysize: 128, PRF: SHA256, Hash: SHA256, DH Grp:2, Auth sign: RSA, Auth verify: RSA
      Life/Active Time: 86400/112 sec

R15#sh crypto ipsec sa

interface: Tunnel1518
    Crypto map tag: Tunnel1518-head-0, local addr 10.10.50.15

   protected vrf: (none)
   local  ident (addr/mask/prot/port): (10.10.50.15/255.255.255.255/47/0)
   remote ident (addr/mask/prot/port): (10.10.50.18/255.255.255.255/47/0)
   current_peer 10.10.50.18 port 500
     PERMIT, flags={origin_is_acl,}
    #pkts encaps: 10, #pkts encrypt: 10, #pkts digest: 10
    #pkts decaps: 86, #pkts decrypt: 86, #pkts verify: 86
    #pkts compressed: 0, #pkts decompressed: 0
    #pkts not compressed: 0, #pkts compr. failed: 0
    #pkts not decompressed: 0, #pkts decompress failed: 0
    #send errors 0, #recv errors 0

     local crypto endpt.: 10.10.50.15, remote crypto endpt.: 10.10.50.18
     plaintext mtu 1458, path mtu 1500, ip mtu 1500, ip mtu idb Ethernet0/2
     current outbound spi: 0x6BC98540(1808368960)
     PFS (Y/N): N, DH group: none

     inbound esp sas:
      spi: 0xA00E6CEA(2685299946)
        transform: esp-aes esp-sha256-hmac ,
        in use settings ={Transport, }
        conn id: 1, flow_id: SW:1, sibling_flags 80000000, crypto map: Tunnel1518-head-0
        sa timing: remaining key lifetime (k/sec): (4314568/3249)
        IV size: 16 bytes
        replay detection support: Y
        Status: ACTIVE(ACTIVE)

     inbound ah sas:

     inbound pcp sas:

     outbound esp sas:
      spi: 0x6BC98540(1808368960)
        transform: esp-aes esp-sha256-hmac ,
        in use settings ={Transport, }
        conn id: 2, flow_id: SW:2, sibling_flags 80000000, crypto map: Tunnel1518-head-0
        sa timing: remaining key lifetime (k/sec): (4314578/3249)
        IV size: 16 bytes
        replay detection support: Y
        Status: ACTIVE(ACTIVE)

     outbound ah sas:

     outbound pcp sas:

R15#ping 10.9.0.6
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.9.0.6, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 6/6/6 ms
```

</details>

<details><summary>R18</summary>

```
R18#sho crypto ikev2 sa
  IPv4 Crypto IKEv2  SA

 Tunnel-id Local                 Remote                fvrf/ivrf            Status
 1         10.10.50.18/500         10.10.50.15/500         none/none            READY  
       Encr: AES-CBC, keysize: 128, PRF: SHA256, Hash: SHA256, DH Grp:2, Auth sign: RSA, Auth verify: RSA
       Life/Active Time: 86400/122 sec

R18#sh crypto ipsec sa

interface: Tunnel1518
   Crypto map tag: Tunnel1518-head-0, local addr 10.10.50.18

  protected vrf: (none)
  local  ident (addr/mask/prot/port): (10.10.50.18/255.255.255.255/47/0)
  remote ident (addr/mask/prot/port): (10.10.50.15/255.255.255.255/47/0)
  current_peer 10.10.50.15 port 500
    PERMIT, flags={origin_is_acl,}
   #pkts encaps: 139, #pkts encrypt: 139, #pkts digest: 139
   #pkts decaps: 10, #pkts decrypt: 10, #pkts verify: 10
   #pkts compressed: 0, #pkts decompressed: 0
   #pkts not compressed: 0, #pkts compr. failed: 0
   #pkts not decompressed: 0, #pkts decompress failed: 0
   #send errors 0, #recv errors 0

    local crypto endpt.: 10.10.50.18, remote crypto endpt.: 10.10.50.15
    plaintext mtu 1458, path mtu 1500, ip mtu 1500, ip mtu idb Ethernet0/2
    current outbound spi: 0xA00E6CEA(2685299946)
    PFS (Y/N): N, DH group: none

    inbound esp sas:
     spi: 0x6BC98540(1808368960)
       transform: esp-aes esp-sha256-hmac ,
       in use settings ={Transport, }
       conn id: 2, flow_id: SW:2, sibling_flags 80000000, crypto map: Tunnel1518-head-0
       sa timing: remaining key lifetime (k/sec): (4252228/3000)
       IV size: 16 bytes
       replay detection support: Y
       Status: ACTIVE(ACTIVE)

    inbound ah sas:

    inbound pcp sas:

    outbound esp sas:
     spi: 0xA00E6CEA(2685299946)
       transform: esp-aes esp-sha256-hmac ,
       in use settings ={Transport, }
       conn id: 1, flow_id: SW:1, sibling_flags 80000000, crypto map: Tunnel1518-head-0
       sa timing: remaining key lifetime (k/sec): (4252210/3000)
       IV size: 16 bytes
       replay detection support: Y
       Status: ACTIVE(ACTIVE)

    outbound ah sas:

    outbound pcp sas:

R18#ping 10.9.0.5
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.9.0.5, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 6/6/7 ms
```

</details>

### Настройка DMVPN поверх IPSec между Москва и Чокурдах, Лабытнанги

Настраиваем IPSec на R15

```
R15(config)#interface Tunnel152728
R15(config-if)#tunnel protection ipsec profile protect-gre
```

Создаем сертификаты на R27 и R28

```
crypto key generate rsa label VPN modulus 2048
crypto pki trustpoint VPN
enrollment url http://10.10.50.14
subject-name CN=R27, OU=VPN, O=LAB, C=RU

//для Чокурдах O=LAB

rsakeypair VPN
revocation-check none

crypto pki authenticate VPN
crypto pki enroll VPN
```

Подписываем сертификаты на R14  
```
R14#show crypto pki server R14CA requests

<...>

Router certificates requests:
ReqID  State      Fingerprint                      SubjectName
--------------------------------------------------------------
2      pending    2EE1EBFE669D928F09BD9005189F8458 hostname=R28,cn=R27,ou=VPN,o=CHO,c=RU
1      pending    C01DD28A58374E0B0CECA14BB64E095E hostname=R27,cn=R27,ou=VPN,o=LAB,c=RU

R14#crypto pki server R14CA grant all
```
Проверяем

<details><summary>R27</summary>

```
R27#show crypto pki certificates
Certificate
  Status: Available
  Certificate Serial Number (hex): 05
  Certificate Usage: General Purpose
  Issuer:
    cn=R14CA
  Subject:
    Name: R27
    hostname=R27
    cn=R27
    ou=VPN
    o=LAB
    c=RU
  Validity Date:
    start date: 22:21:49 MSK Nov 28 2024
    end   date: 22:21:49 MSK Nov 28 2025
  Associated Trustpoints: VPN

CA Certificate
  Status: Available
  Certificate Serial Number (hex): 01
  Certificate Usage: Signature
  Issuer:
    cn=R14CA
  Subject:
    cn=R14CA
  Validity Date:
    start date: 21:01:48 MSK Nov 14 2024
    end   date: 21:01:48 MSK Nov 14 2027
  Associated Trustpoints: VPN
  Storage: nvram:R14CA#1CA.cer
```

</details>

<details><summary>R28</summary>
  
```
R28#show crypto pki certificates
Certificate
  Status: Available
  Certificate Serial Number (hex): 04
  Certificate Usage: General Purpose
  Issuer:
    cn=R14CA
  Subject:
    Name: R28
    hostname=R28
    cn=R27
    ou=VPN
    o=CHO
    c=RU
  Validity Date:
    start date: 22:21:49 MSK Nov 28 2024
    end   date: 22:21:49 MSK Nov 28 2025
  Associated Trustpoints: VPN

CA Certificate
  Status: Available
  Certificate Serial Number (hex): 01
  Certificate Usage: Signature
  Issuer:
    cn=R14CA
  Subject:
    cn=R14CA
  Validity Date:
    start date: 21:01:48 MSK Nov 14 2024
    end   date: 21:01:48 MSK Nov 14 2027
  Associated Trustpoints: VPN
  Storage: nvram:R14CA#1CA.cer
```
</details>

Настраиваем профиль ikev2 и IPSec на R27 и R28

```
(config)#crypto ikev2 proposal PH1
(config-ikev2-proposal)#encryption aes-cbc-128
(config-ikev2-proposal)#integrity sha256
(config-ikev2-proposal)#group 2
(config-ikev2-proposal)#exit
(config)#crypto ikev2 policy IK2POL
(config-ikev2-policy)#proposal PH1
(config-ikev2-policy)#exit
(config)#crypto ikev2 profile PROF1
(config-ikev2-profile)#match address local interface Ethernet0/0

//для R28 так же строка
//(config-ikev2-profile)#match address local interface Ethernet0/1

(config-ikev2-profile)#match identity remote address 0.0.0.0
(config-ikev2-profile)#authentication remote rsa-sig
(config-ikev2-profile)#authentication local rsa-sig
(config-ikev2-profile)#pki trustpoint VPN
(config-ikev2-profile)#exit
(config)#crypto ipsec transform-set IPSEC_TS esp-aes esp-sha256-hmac
(cfg-crypto-trans)#mode transport
(cfg-crypto-trans)#exit
(cfg-crypto-trans)#crypto ipsec profile protect-gre
(ipsec-profile)#set transform-set IPSEC_TS
(ipsec-profile)#set ikev2-profile PROF1
(ipsec-profile)#exit
(config)#interface Tunnel152728
(config-if)#tunnel protection ipsec profile protect-gre
(config-if)#exit
```

Проверяем

<details><summary>R27</summary>

```
R27#show crypto ikev2 session
 IPv4 Crypto IKEv2 Session

Session-id:1, Status:UP-ACTIVE, IKE count:1, CHILD count:1

Tunnel-id Local                 Remote                fvrf/ivrf            Status
1         10.10.100.30/500      10.10.50.15/500       none/none            READY
      Encr: AES-CBC, keysize: 128, PRF: SHA256, Hash: SHA256, DH Grp:2, Auth sign: RSA, Auth verify: RSA
      Life/Active Time: 86400/1002 sec
Child sa: local selector  10.10.100.30/0 - 10.10.100.30/65535
          remote selector 10.10.50.15/0 - 10.10.50.15/65535
          ESP spi in/out: 0x448C5B0/0x41CCF09E

 IPv6 Crypto IKEv2 Session

R27#show crypto ipsec sa

interface: Tunnel152728
    Crypto map tag: Tunnel152728-head-0, local addr 10.10.100.30

   protected vrf: (none)
   local  ident (addr/mask/prot/port): (10.10.100.30/255.255.255.255/47/0)
   remote ident (addr/mask/prot/port): (10.10.50.15/255.255.255.255/47/0)
   current_peer 10.10.50.15 port 500
     PERMIT, flags={origin_is_acl,}
    #pkts encaps: 6, #pkts encrypt: 6, #pkts digest: 6
    #pkts decaps: 6, #pkts decrypt: 6, #pkts verify: 6
    #pkts compressed: 0, #pkts decompressed: 0
    #pkts not compressed: 0, #pkts compr. failed: 0
    #pkts not decompressed: 0, #pkts decompress failed: 0
    #send errors 0, #recv errors 0

     local crypto endpt.: 10.10.100.30, remote crypto endpt.: 10.10.50.15
     plaintext mtu 1458, path mtu 1500, ip mtu 1500, ip mtu idb (none)
     current outbound spi: 0x41CCF09E(1103949982)
     PFS (Y/N): N, DH group: none

     inbound esp sas:
      spi: 0x448C5B0(71878064)
        transform: esp-aes esp-sha256-hmac ,
        in use settings ={Transport, }
        conn id: 2, flow_id: SW:2, sibling_flags 80000000, crypto map: Tunnel152728-head-0
        sa timing: remaining key lifetime (k/sec): (4306744/2588)
        IV size: 16 bytes
        replay detection support: Y
        Status: ACTIVE(ACTIVE)

     inbound ah sas:

     inbound pcp sas:

     outbound esp sas:
      spi: 0x41CCF09E(1103949982)
        transform: esp-aes esp-sha256-hmac ,
        in use settings ={Transport, }
        conn id: 1, flow_id: SW:1, sibling_flags 80000000, crypto map: Tunnel152728-head-0
        sa timing: remaining key lifetime (k/sec): (4306744/2588)
        IV size: 16 bytes
        replay detection support: Y
        Status: ACTIVE(ACTIVE)

     outbound ah sas:

     outbound pcp sas:

R27#ping 10.9.0.9
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.9.0.9, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 6/8/13 ms
```

</details>


<details><summary>R28</summary>

```
R28#show crypto ikev2 session
 IPv4 Crypto IKEv2 Session

Session-id:1, Status:UP-ACTIVE, IKE count:1, CHILD count:1

Tunnel-id Local                 Remote                fvrf/ivrf            Status
1         10.10.100.34/500      10.10.50.15/500       none/none            READY
      Encr: AES-CBC, keysize: 128, PRF: SHA256, Hash: SHA256, DH Grp:2, Auth sign: RSA, Auth verify: RSA
      Life/Active Time: 86400/1215 sec
Child sa: local selector  10.10.100.34/0 - 10.10.100.34/65535
          remote selector 10.10.50.15/0 - 10.10.50.15/65535
          ESP spi in/out: 0xDAA1B0A4/0x6F0B8191

 IPv6 Crypto IKEv2 Session

R28#show crypto ipsec sa

interface: Tunnel152728
    Crypto map tag: Tunnel152728-head-0, local addr 10.10.100.34

   protected vrf: (none)
   local  ident (addr/mask/prot/port): (10.10.100.34/255.255.255.255/47/0)
   remote ident (addr/mask/prot/port): (10.10.50.15/255.255.255.255/47/0)
   current_peer 10.10.50.15 port 500
     PERMIT, flags={origin_is_acl,}
    #pkts encaps: 12, #pkts encrypt: 12, #pkts digest: 12
    #pkts decaps: 12, #pkts decrypt: 12, #pkts verify: 12
    #pkts compressed: 0, #pkts decompressed: 0
    #pkts not compressed: 0, #pkts compr. failed: 0
    #pkts not decompressed: 0, #pkts decompress failed: 0
    #send errors 0, #recv errors 0

     local crypto endpt.: 10.10.100.34, remote crypto endpt.: 10.10.50.15
     plaintext mtu 1458, path mtu 1500, ip mtu 1500, ip mtu idb (none)
     current outbound spi: 0x6F0B8191(1863025041)
     PFS (Y/N): N, DH group: none

     inbound esp sas:
      spi: 0xDAA1B0A4(3668029604)
        transform: esp-aes esp-sha256-hmac ,
        in use settings ={Transport, }
        conn id: 2, flow_id: SW:2, sibling_flags 80000000, crypto map: Tunnel152728-head-0
        sa timing: remaining key lifetime (k/sec): (4164952/2372)
        IV size: 16 bytes
        replay detection support: Y
        Status: ACTIVE(ACTIVE)

     inbound ah sas:

     inbound pcp sas:

     outbound esp sas:
      spi: 0x6F0B8191(1863025041)
        transform: esp-aes esp-sha256-hmac ,
        in use settings ={Transport, }
        conn id: 1, flow_id: SW:1, sibling_flags 80000000, crypto map: Tunnel152728-head-0
        sa timing: remaining key lifetime (k/sec): (4164952/2372)
        IV size: 16 bytes
        replay detection support: Y
        Status: ACTIVE(ACTIVE)

     outbound ah sas:

     outbound pcp sas:
R28#ping 10.9.0.9
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.9.0.9, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 7/7/8 ms
```

</details>




