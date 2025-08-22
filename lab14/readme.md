## Лабораторная работа 

IPSec over DmVPN

---
### Цель:

- Настроить GRE поверх IPSec между офисами Москва и С.-Петербург
- Настроить DMVPN поверх IPSec между офисами Москва и Чокурдах, Лабытнанги

Описание/Пошаговая инструкция выполнения домашнего задания:

- Настроить GRE поверх IPSec между офисами Москва и С.-Петербург.
- Настроить DMVPN поверх IPSec между Москва и Чокурдах, Лабытнанги.
- Все узлы в офисах в лабораторной работе должны иметь IP связность.
- План работы и изменения зафиксированы в документации.
---

## Решение:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab14/png/topology.png)


### 1. Настроить GRE поверх IPSec между офисами Москва и С.-Петербург

Будем настраивать IPSec IKEv2 Route based. GRE настроен ранее в lab 13.

**Настраиваем Фазу 1 (IKEv2 — установление защищённого канала для обмена ключами. указываем какими алгоритмами и методами мы будем защищать канал обмена ключами.):**

```
crypto ikev2 proposal PHASE1
 encryption aes-cbc-128
 integrity md5
 group 2               
```

Создаём политику IKEv2. В политике указываем, какой proposal использовать:

```
crypto ikev2 policy IKEV2
 proposal PHASE1
```

Создаём профиль IKEv2:

```
crypto ikev2 profile PROFILE1
 match address local interface Ethernet0/2
 match identity remote address 200.50.0.22 255.255.255.252
 authentication remote pre-share key CISCO123
 authentication local pre-share key CISCO123
```

**Настраиваем Фазу 2 (IPSec SA — настройка защиты полезного трафика. Определяем, как шифруем и проверяем целостность полезного трафика (который пойдёт через туннель)).**

Задаём `transform-set` (набор алгоритмов защиты для ESP (или AH) в Фазе 2)

```
crypto ipsec transform-set IPSEC_TS esp-aes esp-md5-hmac
 mode transport
```

**Создаём профиль IPSec (указаем, каким transform-set (Фаза 2) пользоваться и каким IKEv2(связываем их вместе))**

```
crypto ipsec profile IPSEC_PROFILE
 set transform-set IPSEC_TS
 set ikev2-profile PROFILE1
```

**Привязываем профиль IPSec к туннельному интерфейсу**
```
interface Tunnel0
 ip address 10.200.200.1 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source 200.50.0.10
 tunnel destination 200.50.0.22
 tunnel protection ipsec profile IPSEC_PROFILE
```

На R18 аналогично:

```
crypto ikev2 proposal PHASE1
 encryption aes-cbc-128
 integrity md5
 group 2

crypto ikev2 policy IKEV2
 proposal PHASE1

crypto ikev2 profile PROFILE1
 match address local interface Ethernet0/2
 match identity remote address 200.50.0.10 255.255.255.252
 authentication remote pre-share key CISCO123
 authentication local pre-share key CISCO123

crypto ipsec transform-set IPSEC_TS esp-aes esp-md5-hmac
 mode transport

crypto ipsec profile IPSEC_PROFILE
 set transform-set IPSEC_TS
 set ikev2-profile PROFILE1

interface Tunnel0
 ip address 10.200.200.2 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source 200.50.0.22
 tunnel destination 200.50.0.10
 tunnel protection ipsec profile IPSEC_PROFILE
```

Проверка:

пинг проходит, трафик идёт через туннельный интерфейс


![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab14/png/GRE%20over%20IPSec/ping%20VPC1%20VPC8.png)


![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab14/png/GRE%20over%20IPSec/Wireshark_IPSec_over_GRE.png)



### 2. Настроить DMVPN поверх IPSec между Москва и Чокурдах, Лабытнанги.

DMVPN Phase2 уже настроен в lab13. Будем настраивать DMVPN поверх IPSec IKEV2.

Обе конфигурации на R15(одна GRE over IPSec и вторая DMVPN over IPSec) могут использовать IKEv2, но с разными профилями и match-условиями.

Настройка IPSec аналогична п.1, только создаём новый профиль и соответственно для него transform-set, IKEv2-профиль, политику.

Настройки на HUB и Spoke идентичные.

`match identity remote address 0.0.0.0 0.0.0.0` т.к. споки взаимодействуют во 2 фазе DMVPN друг с другом напрямую (теоретически их может быть много) и HUB взаимодействует со можествовом споков.

R15:
```
crypto ikev2 proposal DMVPN_PHASE2
 encryption aes-cbc-128
 integrity md5
 group 2

crypto ikev2 policy DMVPN_IKEV2
 proposal DMVPN_PHASE2

crypto ikev2 profile DMVPN_PROFILE
 match address local interface Ethernet0/2
 match identity remote address 0.0.0.0
 authentication remote pre-share key DMVPNKEY
 authentication local pre-share key DMVPNKEY

crypto ipsec transform-set DMVPN_TS esp-aes esp-md5-hmac
 mode transport

crypto ipsec profile DMVPN_IPSEC_PROFILE
 set transform-set DMVPN_TS
 set ikev2-profile DMVPN_PROFILE

interface Tunnel10
 description FOR-DMVPN-HUB
 ip address 10.115.115.1 255.255.255.0
 no ip redirects
 ip mtu 1380
 ip nhrp map multicast dynamic
 ip nhrp network-id 999
 ip tcp adjust-mss 1360
 tunnel source Ethernet0/2
 tunnel mode gre multipoint
 tunnel key 999
 tunnel protection ipsec profile DMVPN_IPSEC_PROFILE
```

R27:

```
crypto ikev2 proposal DMVPN_PHASE2
 encryption aes-cbc-128
 integrity md5
 group 2

crypto ikev2 policy DMVPN_IKEV2
 proposal DMVPN_PHASE2

crypto ikev2 profile DMVPN_PROFILE
 match address local interface Ethernet0/0
 match identity remote address 0.0.0.0
 authentication remote pre-share key DMVPNKEY
 authentication local pre-share key DMVPNKEY

crypto ipsec transform-set DMVPN_TS esp-aes esp-md5-hmac
 mode transport


crypto ipsec profile DMVPN_IPSEC_PROFILE
 set transform-set DMVPN_TS
 set ikev2-profile DMVPN_PROFILE


interface Tunnel0
 description FOR-DMVPN-SPOKE27
 ip address 10.115.115.2 255.255.255.0
 no ip redirects
 ip mtu 1380
 ip nhrp map 10.115.115.1 200.50.0.10
 ip nhrp map multicast 10.115.115.1
 ip nhrp network-id 999
 ip nhrp nhs 10.115.115.1
 ip nhrp registration no-unique
 ip tcp adjust-mss 1360 
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 999
 tunnel protection ipsec profile DMVPN_IPSEC_PROFILE
```

R28 настраиваем аналогично:
```
crypto ikev2 proposal DMVPN_PHASE2
 encryption aes-cbc-128
 integrity md5
 group 2

crypto ikev2 policy DMVPN_IKEV2
 proposal DMVPN_PHASE2

crypto ikev2 profile DMVPN_PROFILE
 match address local interface Ethernet0/0
 match identity remote address 0.0.0.0
 authentication remote pre-share key DMVPNKEY
 authentication local pre-share key DMVPNKEY


crypto ipsec transform-set DMVPN_TS esp-aes esp-md5-hmac
 mode transport

crypto ipsec profile DMVPN_IPSEC_PROFILE
 set transform-set DMVPN_TS
 set ikev2-profile DMVPN_PROFILE


interface Tunnel0
 description FOR-DMVPN-SPOKE28
 ip address 10.115.115.3 255.255.255.0
 no ip redirects
 ip mtu 1380
 ip nhrp map 10.115.115.1 200.50.0.10
 ip nhrp map multicast 10.115.115.1
 ip nhrp network-id 999
 ip nhrp nhs 10.115.115.1
 ip nhrp registration no-unique
 ip tcp adjust-mss 1360
 tunnel source Ethernet0/0
 tunnel mode gre multipoint
 tunnel key 999
 tunnel protection ipsec profile DMVPN_IPSEC_PROFILE
```

Проверка:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab14/png/DMVPN/R15_sh_crypto_ikev2_sa.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab14/png/DMVPN/R15_sh_crypto_ikev2_session.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab14/png/DMVPN/R15_sh_crypto_ipsec_profile.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab14/png/DMVPN/R15_sh_crypto_session.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab14/png/DMVPN/Wireshark_VPC1_VPC30.png)


[Конфигурации](https://github.com/buravtsovpavel/nw-labs/tree/main/lab14/configs) устройств


