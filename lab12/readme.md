## Лабораторная работа 

Основные протоколы сети интернет

---
### Цель:

- Настроить DHCP в офисе Москва
- Настроить синхронизацию времени в офисе Москва
- Настроить NAT в офисе Москва, C.-Перетбруг и Чокурдах

Описание/Пошаговая инструкция выполнения домашнего задания:

- Настроить NAT(PAT) на R14 и R15. Трансляция должна осуществляться в адрес автономной системы AS1001.
- Настроить NAT(PAT) на R18. Трансляция должна осуществляться в пул из 5 адресов автономной системы AS2042.
- Настроить статический NAT для R20.
- Настроить NAT так, чтобы R19 был доступен с любого узла для удаленного управления.
- 5*. Настроить статический NAT(PAT) для офиса Чокурдах.
- Настроить для IPv4 DHCP сервер в офисе Москва на маршрутизаторах R12 и R13. VPC1 и VPC7 должны получать сетевые настройки по DHCP.
- Настроить NTP сервер на R12 и R13. Все устройства в офисе Москва должны синхронизировать время с R12 и R13.
- Все офисы в лабораторной работе должны иметь IP связность.
- План работы и изменения зафиксированы в документации.
---

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab12/png/topology.png)

## Решение:

### Введение:

Поскольку сейчас R21 знает маршрут до внешнего ip R14 200.50.0.2/30 через R15, то когда хост в AS 2042 (например, 172.16.8.2) отвечает на пинг 200.50.0.2, его трафик пойдёт не к R14, а через R15 и следовательно, ответный пакет для пакета пришедшего с R14 никогда не попадает на e0/2 R14, и R14 не cможет выполнить обратную трансляцию.
Поэтому нужно сделать, что бы маршрут 200.50.0.2/30 был объявлен только с R14, а не через iBGP  для R15 и далее от R15 для R21. Для этого зафильтруем его на исходящих от R15 в сторону R21 анонсах.

на R15:

```
ip prefix-list R14-NAT seq 5 permit 200.50.0.0/30
```
```
route-map BLOCK-R14 deny 10
 match ip address prefix-list R14-NAT
!
route-map BLOCK-R14 permit 20
```
```
router bgp 1001
 neighbor 200.50.0.9 route-map BLOCK-R14 out
```
Теперь R15 получает *>i 200.50.0.0/30 от R14, но не отдаёт его R21

А R21 теперь знает маршрут до 200.50.0.0/30 через R22 и ответный пакет до 200.50.0.2 прийдёт на R14.

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab12/png/R21_after.png)


### 1. Настроить NAT(PAT) на R14 и R15. Трансляция должна осуществляться в адрес автономной системы AS1001.

Делаем PAT на R14 клиентские сети -> 200.50.0.2, на R15 клиентские сети -> 200.50.0.10

R14:

Определяем стандартный ACL, разрешающий адреса, которые должны быть преобразованы:

```
R14(config)#ip access-list standard FOR-NAT
R14(config-std-nacl)#permit 172.16.1.0 0.0.0.255
R14(config-std-nacl)#permit 172.16.7.0 0.0.0.255
R14(config-std-nacl)#exit
```
Создаём пул NAT-адресов предназначенных для NAT-трансляции исходящих соединений, и IP-адреса из этого пула будут использоваться для преобразования внутренних адресов в публичные. В нашем случае пул из одного адреса:

```
R14(config)#ip nat pool R14-NAT 200.50.0.2 200.50.0.2 netmask 255.255.255.252
```
Настраиваем динамическое преобразование адреса источника, указав параметры ACL, пул и параметры перегрузки:

```
R14(config)#ip nat inside source list FOR-NAT pool R14-NAT overload
```
Определяем внутренние и внешние интерфейсы:

```
R14(config)#interface range ethernet 0/0 - 1
R14(config-if-range)#ip nat inside
```
```
R14(config)#interface ethernet 1/0
R14(config-if)#ip nat inside
R14(config-if)#exit
R14(config)#interface ethernet 0/3
R14(config-if)#ip nat inside
```
```
R14(config)#interface ethernet 0/2
R14(config-if)#ip nat outside
```
R15 аналогично.

Проверяем, что PAT работает:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab12/png/R14_nat_tr_1.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab12/png/R15_nat_tr_1.png)

### 2. Настроить NAT(PAT) на R18. Трансляция должна осуществляться в пул из 5 адресов автономной системы AS2042.

На внешних интерфейсах R18 адреса из сетей /30, в которых свобдных адресов нет. Поэтому для AS2042 просто выберем /29 сеть с 8 адресами из которых 5 можно использовать для PAT, добавим статический маршрут для этой сети в Null0 и анонсируем эту сеть в eBGP, чтобы другие роутеры получили маршрут до этой подсети по BGP.  И далее настроим NAT-пул на R18 с 5 адресами из неё.

на R18

```
ip route 203.0.113.0 255.255.255.248 Null0
```

```
ip access-list standard FOR-NAT
 permit 172.16.8.0 0.0.3.255
```

```
ip nat pool FOR-NAT 203.0.113.1 203.0.113.5 netmask 255.255.255.248
ip nat inside source list FOR-NAT pool FOR-NAT overload
```

```
router bgp 2042
 network 203.0.113.0 mask 255.255.255.248
```

Примечание:

Поскольку в п2. работы "BGP. Фильтрация" делали prefix-list разрешающий только необходимые (локальные) маршруты использовавшиеся в route-map которую вешали на выход к соседям в Триаде, то в него следует добавить нашу новую сеть:

```
ip prefix-list ONLY-LOCAL seq 25 permit 203.0.113.0/29
```

Так же т.к. в п. 4 "BGP. Фильтрация" делали так, что бы в офис Москва отдавался только маршрут по умолчанию и префикс офиса С.-Петербург, то нужно на R21 тоже добавить новую подсеть:

на R21
```
ip prefix-list SPB-AND-DEFAULT seq 20 permit 203.0.113.0/29
```
после `clear ip bgp * soft` на R15 маршрут пришёл

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab12/png/R15_ip_route_bgp.png)

Проверяем, что PAT работает:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab12/png/2/R18_nat_tr.png)

### 3. Настроить статический NAT для R20.

Поскольку на R15 внешний ip 200.50.0.10 уже используется для PAT в п.1, то в него транслировать пакеты от R20 уже нельзя и свободных адресов в этой 200.50.0.8/30 нет. Сделаем статический NAT в адрес loopback R15. Адрес приватный серый, но для лабораторной среды не принципиально, тем более, что он уже анонсирован по eBGP.

внутренний адрес R20 e0/0 10.0.10.22 будет транслироваться в 10.1.0.2

```
R15(config)#ip nat inside source static 10.0.10.22 10.1.0.2
```

Проверяем, что статический NAT работает:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab12/png/3/R15_nat_tr_static_R20.png)


### 4. Настроить NAT так, чтобы R19 был доступен с любого узла для удаленного управления.

Создадим дополнительный Loopback 2 на R14, назначим на него ip 10.1.0.10/32 и сделаем проброс (mapping) порта 2023 в 23 telnet. Согласно настроенному правилу внешние подключения к IP 10.1.0.10 на порт 2023 будут перенаправлены (проброшены) на внутренний хост 10.1.0.6 порт 23.

Что бы пакеты до 10.1.0.10 шли только на R14 (т.к. трансляцию будем делать только на нём), то нужно, что бы маршрут 10.1.0.10/32 был всем внешним роутерам известен только через R14. Но когда мы его анонсируем, то R15 тоже его получит и перекинет на R21 а тот дальше и R21 и R18 будут идти на 10.1.0.10 через R15. Что бы этого не было запретим R15 анонсировать 10.1.0.10/32 R21, для этого в prefix-list из введения где запрещали 200.50.0.0/30 добавим ещё сеть 10.1.0.10/32.
```
ip prefix-list R14-NAT seq 5 permit 200.50.0.0/30
ip prefix-list R14-NAT seq 10 permit 10.1.0.10/32
```
Теперь R21, R18 и т.д. пойдут к 10.1.0.10/32 через R14:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab12/png/4/R21_sh_ip_route.png)

R14:

```
interface Loopback2
 ip address 10.1.0.10 255.255.255.255
```
```
ip nat inside source static tcp 10.1.0.6 23 10.1.0.10 2023 extendable
```

на R19:

```
line vty 0 4
 password 7 123A2C243124
 login
 transport input telnet
!
!
end
```
Проверяем удалённый доступ:

https://github.com/buravtsovpavel/nw-labs/blob/main/lab12/png/4/R21_telnet_2023.png

https://github.com/buravtsovpavel/nw-labs/blob/main/lab12/png/4/R18_telnet_2023.png

Посмотрим трансляцию:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab12/png/4/R14_nat_trans.png)


### 6. Настроить для IPv4 DHCP сервер в офисе Москва на маршрутизаторах R12 и R13. VPC1 и VPC7 должны получать сетевые настройки по DHCP.

R12:

```
ip dhcp excluded-address 172.16.1.50
ip dhcp excluded-address 172.16.1.1
ip dhcp excluded-address 172.16.1.51
ip dhcp pool VLAN-11
 network 172.16.1.0 255.255.255.0
 default-router 172.16.1.50
 lease 2 12 30
```
и на SW4 на интерфейсе vlan 11 являющемся шлюзом для хостов подсети 172.16.1.2/24 (VPC1) указываем `ip helper-address 10.0.10.33` - он будет пересылать DHCP запросы на R12 10.0.10.33
```
interface Vlan11
 description GW-FOR-VPC1
 ip address 172.16.1.50 255.255.255.0
 ip helper-address 10.0.10.33
 standby version 2
 standby 11 ip 172.16.1.1
 standby 11 priority 150
 standby 11 preempt
end
```

R13:

```
ip dhcp excluded-address 172.16.7.51
ip dhcp excluded-address 172.16.7.1
ip dhcp excluded-address 172.16.7.50
ip dhcp pool VLAN-7
 network 172.16.7.0 255.255.255.0
 default-router 172.16.7.1
 lease 2 12 30
```

SW4:

```
interface Vlan7
 description GW-FOR-VPC7
 ip address 172.16.7.50 255.255.255.0
 ip helper-address 10.0.10.29
 standby version 2
 standby 7 ip 172.16.7.1
 standby 7 priority 150
 standby 7 preempt
```

Проверяем получение адреса и шлюза по DHCP:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab12/png/5_DHCP/VPC1_dhcp.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab12/png/5_DHCP/VPC7_dhcp.png)

### 7. Настроить NTP сервер на R12 и R13. Все устройства в офисе Москва должны синхронизировать время с R12 и R13.

Настроим R12 как корневой сервер NTP(локальный источник времени) и сделаем peer между R12 и R13. Тогда R13 будет синхронизироваться от R12, станет тоже сервером со stratum 5 (на единицу меньше) и мы его укажем для части устройств нашей топологии.

R12:

```
ntp source Loopback0
ntp master 4
ntp update-calendar
ntp peer 10.1.0.4
```

R13:
```
ntp source Loopback0
ntp update-calendar
ntp peer 10.1.0.5
```

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab12/png/6_NTP/R12_status.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab12/png/6_NTP/R13_status.png)

R19, R14, SW4 настраиваем клиентами R12:

```
ntp source Loopback0
ntp server 10.1.0.5
```
После синхронизации они становятся stratum 5 и от них можно синхронизироваться:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab12/png/6_NTP/R19_ntp.png)

SW5, R15, R20 настраиваем клиентами R13:

```
ntp source Loopback0
ntp server 10.1.0.4
```

После синхронизации они становятся stratum 6 и от них можно синхронизироваться:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab12/png/6_NTP/SW5_status.png)

для SW3 указываем сервером SW4, а для SW2 сервером указываем SW5

`ntp server 172.16.50.4`

`ntp server 172.16.50.3`

После синхронизации они становятся stratum 6 и 7 соотвественно:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab12/png/6_NTP/SW2_status.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab12/png/6_NTP/SW3_status.png)
