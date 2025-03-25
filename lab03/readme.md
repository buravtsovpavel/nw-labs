## Лабораторная работа 

### Часть 1. DHCPv4
---
### Топология:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv4/png/topology.png)

### Цели

1. Создать сеть и настроить основные параметры устройств

2. Настроить и проверить два DHCPv4 сервера на R1

3. Настроить и проверить DHCP Relay (DHCP-ретрансляцию) на R2
---

### 1. Для построения сети и настройки параметров устройств разделим сеть 192.168.1.0/24 на три подсети.
* Подсеть A на 58 хостов: 192.168.1.0/26
* Подсеть B на 28 хостов: 192.168.1.64/27
* Подсеть C на 12 хостов: 192.168.1.96/28

Таблица адресации

 Device    | Interface | IP Address   | Subnet Mask     | Default Gateway
---------- | --------- | --------     | -------------   | -----------------
R1         | e0/0      | 10.0.0.1     | 255.255.255.252 | N/A
R1         | e0/1      |      N/A     |   N/A           | N/A
R1         | e0/1.100      | 192.168.1.1     | 255.255.255.192 | N/A
R1         | e0/1.200      | 192.168.1.65     | 255.255.255.224 | N/A
R1         | e0/1.1000      | N/A     | N/A | N/A
R2         | e0/0      | 10.0.0.2     | 255.255.255.252 | N/A
R2         | e0/1      | 192.168.1.97     | 255.255.255.240 | N/A
S1         | VLAN 200      | 192.168.1.66     | 255.255.255.224 | 192.168.1.65
S2         | VLAN 1      | 192.168.1.98     | 255.255.255.240 | 192.168.1.97
PC-A       | NIC       | DHCP  | DHCP | DHCP
PC-B       | NIC       | DHCP  | DHCP | DHCP


Таблица VLAN
VLAN    | Name       | Interface Assigned
--------| ----       | ------------------
1       | N/A | S2: e0/0
100       | Clients | S1: e0/0
200       | Management | S1: VLAN 200
999       | Parking Lot | S1: e0/2, e0/3
1000      | Native      | N/A

#### 1.1. Производим базовую настройку устройств:

(hostname, отключение автоматического выполнения DNS-запросов, установка зашифрованного пароля привилегированного режима, установка пароля для доступа через консоль, установка пароля на vty для доступа по протоколам telnet или ssh, шифруем пароли, создаём баннер)

Пример настройки для S1:
```
Switch(config)#hostname S1
S1(config)#no ip domain lookup
S1(config)#enable secret class
S1(config)#line console 0
S1(config-line)#password cisco
S1(config-line)#exit
S1(config)#line vty 0 4
S1(config-line)#password cisco
S1(config-line)#login
S1(config-line)#exit
S1(config)#service password-encryption
S1(config)#banner motd #
Enter TEXT message.  End with the character '#'.
***********************************************
This is a secure system. Authorized Access Only!
***********************************************
#
S1(config)#clock timezone YEKT 5 0
S1(config)#
```
остальные устройства настраиваем схожим образом

#### 1.2 Настраиваем Inter-VLAN Routing на R1

```
R1(config)#interface ethernet0/1
R1(config-if)#no shutdown


R1(config)#int e0/1.100
R1(config-subif)#encapsulation dot1Q 100
R1(config-subif)#description CLIENTS
R1(config-subif)#ip address 192.168.1.1 255.255.255.192
R1(config-subif)#exit
R1(config)#interface e0/1.200
R1(config-subif)#encapsulation dot1Q 200
R1(config-subif)#description MANAGEMENT
R1(config-subif)#ip address 192.168.1.65 255.255.255.224
R1(config-subif)#exit
R1(config)#interface e0/1.1000
R1(config-subif)#encapsulation dot1Q 1000
R1(config-subif)#description NATIVE
```
Убедимся, что sub-интерфесы в UP'е

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv4/png/sh_ip_br_R1.png)

#### 1.3 Настраиваем интерфейсы на R2, R1 и статическую маршрутизацию.
```
R2(config)#interface e0/0
R2(config-if)#no shutdown
R2(config-if)#ip ad
*Mar 23 08:27:34.088: %LINK-3-UPDOWN: Interface Ethernet0/0, changed state to up
*Mar 23 08:27:35.093: %LINEPROTO-5-UPDOWN: Line protocol on Interface Ethernet0/0, changed state to up
R2(config-if)#ip address 10.0.0.2 255.255.255.252
R2(config-if)#exit
R2(config)#interface e0/1
R2(config-if)#no shutdown
R2(config-if)#ip address
*Mar 23 08:28:34.139: %LINK-3-UPDOWN: Interface Ethernet0/1, changed state to up
*Mar 23 08:28:35.139: %LINEPROTO-5-UPDOWN: Line protocol on Interface Ethernet0/1, changed state to up
R2(config-if)#ip address 192.168.1.97 255.255.255.240
R2(config-if)#exit
R2(config)#ip route 0.0.0.0 0.0.0.0 10.0.0.1


R1(config)#interface e0/0
R1(config-if)#no shutdown
R1(config-if)#ip address 10.0.0.1 255.255.255.252
R1(config-if)#exit
R1(config)#ip route 0.0.0.0 0.0.0.0 10.0.0.2
```
Пингуем с R1 интерфейс e0/1 на R2

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv4/png/ping%20R2%20e01%20from%20R1.png)


#### 1.4 Настраиваем коммутаторы S1 и S2
Делаем базовую настройку, создаём VLAN'ы, настриваем интерфейсы управления и шлюз по умолчанию. Назначем интерфейсы в нужные VLAN, неиспользуемые порты добавляем в VLAN PARKING LOT.

```
S1(config)#vlan 100
S1(config-vlan)#name CLIENTS
S1(config-vlan)#exit
S1(config)#vlan 200
S1(config-vlan)#name MANGEMENT
S1(config-vlan)#exit
S1(config)#vlan 999
S1(config-vlan)#name PARKING LOT
S1(config-vlan)#exit
S1(config)#vlan 1000
S1(config-vlan)#name NATIVE
S1(config-vlan)#exit
S1(config)#interface vlan 200
S1(config-if)#no shutdown
*Mar 23 09:26:21.978: %LINEPROTO-5-UPDOWN: Line protocol on Interface Vlan200, changed state to down
*Mar 23 09:26:25.430: %LINK-3-UPDOWN: Interface Vlan200, changed state to down
S1(config-if)#ip address 192.168.1.66 255.255.255.224
S1(config-if)#exit
S1(config)#ip default-gateway 192.168.1.65

S1(config)#interface range e0/2 - 3
S1(config-if-range)#switchport mode access
S1(config-if-range)#switchport access vlan 999
S1(config-if-range)#shutdown


S2(config)#interface vlan 1
S2(config-if)#no shutdown
S2(config-if)#ip address 192.168.1.98 255.255.255.240
S2(config-if)#exit
S2(config)#ip default-gateway 192.168.1.97
```

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv4/png/sh_vlan_br.png)

e0/1 пока в VLAN 1, т.к. не назначали его в другой VLAN.

#### 1.5 Настраиваем интерфейс e0/1 на S1 как транковый

```
S1(config)#interface e0/1
S1(config-if)#switchport trunk encapsulation dot1q
S1(config-if)#switchport mode trunk
*Mar 23 09:52:15.948: %LINEPROTO-5-UPDOWN: Line protocol on Interface Ethernet0/1, changed state to down
*Mar 23 09:52:18.720: %LINEPROTO-5-UPDOWN: Line protocol on Interface Ethernet0/1, changed state to up
S1(config-if)#switchport trunk allowed VLAN 100,200,1000
*Mar 23 09:52:49.961: %LINK-3-UPDOWN: Interface Vlan200, changed state to up
*Mar 23 09:52:50.966: %LINEPROTO-5-UPDOWN: Line protocol on Interface Vlan200, changed state to up
S1(config-if)#switchport trunk native vlan 1000
```
![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv4/png/S1_sh_int_tr.png)

Какой IP-адрес будет у ПК на этом этапе, если он подключен к сети с помощью DHCP?

Т.к. DHCP ещё не настоили и статически адрес не задан, то он получит его по APIPA (Automatic Private IP Addressing) из сети 169.254.0.0/16

### 2. Сконфигурируем и проверим два DHCP сервера на R1 (для подсетей A и C)

#### 2.1
- исключаем первые пять используемых адресов из каждого пула адресов
- создаём пул DHCP (используйте уникальное имя для каждого пула). R1_CLIENTS_NET_A  для подсети A и R2_Client_LAN для C.
- указываем сеть, которую поддерживает этот DHCP-сервер.
- настраиваем доменное имя как ccna-lab.com
- настраиваем соответствующий шлюз по умолчанию для каждого пула DHCP.
- настраиваем время аренды на 2 дня 12 часов и 30 минут.


```
R1(config)#ip dhcp excluded-address 192.168.1.1 192.168.1.5
R1(config)#ip dhcp pool R1_CLIENTS_NET_A
R1(dhcp-config)#network 192.168.1.0 255.255.255.192
R1(dhcp-config)#domain-name ccna-lab.com
R1(dhcp-config)#default-router 192.168.1.1
R1(dhcp-config)#lease 2 12 30
R1(dhcp-config)#exit
R1(config)#ip dhcp excluded-address 192.168.1.97 192.168.1.101
R1(config)#ip dhcp pool R2_Client_LAN
R1(dhcp-config)#default-router 192.168.1.97
R1(dhcp-config)#domain-name ccna-lab.com
R1(dhcp-config)#lease 2 12 30
```

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv4/png/sh_ip_dhcp_R1.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv4/png/sh_ip_dhcp_binding_R1.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv4/png/sh_ip_dhcp_statistics_R1.png)

#### 2.2 Получаем IP-адрес по DHCP на PC-A

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv4/png/dhcp_PC_A.png)

#### 2.3 Сконфигурируем и проверим DHCP Relay(ретрансляцию) на R2

Настраиваем R2 как DHCP relay agent для LAN сети (ip helper-address)

#### 2.4 Получаем IP-адрес по DHCP на PC-B

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv4/png/dhcp_PC_B.png)

проверяем связность до R1 

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv4/png/ping_PC_B_R1.png)

```
show ip dhcp binding 
show ip dhcp server statistics  на R1 и R2
```
![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv4/png/sh_ip_dhcp_binding_R1_2.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv4/png/sh_ip_dhcp_statistics_R1_2.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv4/png/sh_ip_dhcp_statistics_R2.png)

конфигурации устройств первой части [S1](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv4/configs_dhcpv4/DHCPv4/S1.txt), [S2](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv4/configs_dhcpv4/DHCPv4/S2.txt), [R1](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv4/configs_dhcpv4/DHCPv4/R1.txt), [R2](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv4/configs_dhcpv4/DHCPv4/R2.txt)

------

### Часть 2. DHCPv6 SLAAC

Топология

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv6/png/topology2.png)


Таблица адресации

 Device    | Interface | IPv6 Address               | 
---------- | --------- | --------                   |
R1         | e0/0      | 2001:db8:acad:2::1/64     | 
R1         | e0/0      | fe80::1                    | 
R1         | e0/1      | 2001:db8:acad:1::1/64      | 
R1         | e0/1      | fe80::1                    | 
R2         | e0/0      | 2001:db8:acad:2::2/64      | 
R2         | e0/0      | fe80::2                    | 
R2         | e0/1      | 2001:db8:acad:3::1/64     |
R2         | e0/1      | fe80::1                    | 
PC-A       | NIC       | DHCP                       |
PC-B       | NIC       | DHCP                       |

### Цели

1. Создать сеть и настроить основные параметры устройств
2. Проверить получение адреса SLAAC от R1
3. Настроить и проверить Stateless DHCPv6 сервер на R1
4. Настроить и проверить Stateful DHCPv6 сервер на R1
5. Настроить и проверить DHCPv6 Relay на R2

### 2.1. Производим базовую настройку устройств:
На коммутаторах:
hostname, отключение автоматического выполнения DNS-запросов, установка зашифрованного пароля привилегированного режима, установка пароля для доступа через консоль, установка пароля на vty для доступа по протоколам telnet или ssh, шифруем пароли, создаём баннер, отключаем все неиспользуемые порты.
*На роутерах аналогично и плюс ещё включаем маршрутизацию IPv6*
```
Router(config)#hostname R1
R1(config)#no ip domain lookup
R1(config)#enable secret class
R1(config)#line console 0
R1(config-line)#password cisco
R1(config-line)#exit
R1(config)#line vty 0 4
R1(config-line)#password cisco
R1(config-line)#login
R1(config-line)#exit
R1(config)#service password-encryption
R1(config)#banner motd #
Enter TEXT message.  End with the character '#'.
***********************************************
This is a secure system. Authorized Access Only!
***********************************************
#
R1(config)#clock timezone YEKT 5 0
R1(config)#ipv6 unicast-routing
R1(config)#
*Mar 24 14:43:47.477: %SYS-6-CLOCKUPDATE: System clock has been updated from 14:43:47 UTC Mon Mar 24 2025 to 19:43:47 YEKT Mon Mar 24 2025, configured from console by console.
```

#### 2.2 Настраиваем интерфейсы и маршрутизацию для обоих маршрутизаторов.

```
R1(config)#interface e0/0
R1(config-if)#ipv6 address 2001:db8:acad:2::1/64
R1(config-if)#ipv6 address fe80::1 link-local
R1(config-if)#no shutdown
R1(config-if)#exit
R1(config)#interface e0/1
R1(config-if)#ipv6 address 2001:db8:acad:1::1/64
R1(config-if)#ipv6 address fe80::1 link-local
R1(config-if)#no shutdown
```


```
R2(config)#interface e0/0
R2(config-if)#no shutdown
R2(config-if)#ipv6 address 2001:db8:acad:2::2/64
R2(config-if)#ipv6 address fe80::2 link-local
R2(config-if)#exit
R2(config)#interface e0/1
R2(config-if)#no shutdown
R2(config-if)#ipv6 address 2001:db8:acad:3::1/64
R2(config-if)#ipv6 address fe80::1 link-local
```

```
R1(config)#ipv6 route ::/0 2001:db8:acad:2::2
R2(config)#ipv6 route ::/0 2001:db8:acad:2::1
```
Проверим:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv6/png/ping_R1_R2.png)
![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv6/png/ping_R2_R1.png)

#### 2.3 Проверка получения IPv6 GUA адреса через SLAAC на PC-A от R1

Видно, что получен адрес из подести, которая на интерфейсе e0/1 роутера R1

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv6/png/ip_PCA.png)

Откуда взялась часть адреса host-id?

2001:db8:acad:1 - это префикс сети, переданный от R1

5111:a2d7:4bef:1805 - host-id самостоятельно cгенерированный себе хостом с помощью механизма EUI-64 (Extended Unique Identifier)

### 2.4 Настройка и проверка stateless DHCPv6-сервера на R1 (цель — предоставить PC-A информацию о DNS-сервере и домене.)

Создаём пул IPv6 DHCP на R1 с именем R1-STATELESS. В нём указываем адрес DNS-сервера 2001:db8:acad::1 и доменное имя STATELESS.com. Привязываем интерфейс к пулу с и меняем флаг O с 0 на 1.

```
R1(config)#ipv6 dhcp pool R1-STATELESS
R1(config-dhcpv6)#dns-server 2001:db8:acad::254
R1(config-dhcpv6)#domain-name STATELESS.com
R1(config-dhcpv6)#exit
R1(config)#interface e0/1
R1(config-if)#ipv6 nd other-config-flag
R1(config-if)#ipv6 dhcp server R1-STATELESS
```
Проверяем, что хост PC-A получил информацию от DHCP сервера по указанным настройкам:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv6/png/resolf_conf.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv6/png/conn_inf.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv6/png/ping_PCA_R2.png)

### 2.5 Настройка и проверка stateful DHCPv6-сервера на R1 (для ответа на запросы DHCPv6 из локальной сети на R2)

Создаём пул DHCPv6 на R1 для сети 2001:db8:acad:3:aaaa::/80 для локальной сети, подключенной к интерфейсу e0/1 на R2. В нём указываем адрес DNS-сервера 2001:db8:acad::254 и доменное имя STATEFUL.com.
```
R1(config)#ipv6 dhcp pool R2-STATEFUL
R1(config-dhcpv6)#address prefix 2001:db8:acad:3:aaa::/80
R1(config-dhcpv6)#dns-server 2001:db8:acad::254
R1(config-dhcpv6)#domain-name STATEFUL.com
R1(config-dhcpv6)#exit
R1(config)#interface ethernet0/0
R1(config-if)#ipv6 dhcp server R2-STATEFUL
```

Сконфигурируем и проверим DHCP Relay(ретрансляцию) на R2 для локальной сети на интерфейсе e0/1

```
R2(config-if)#ipv6 nd managed-config-flag
R2(config-if)#ipv6 dhcp relay destination 2001:db8:acad:2::1 e0/0
```
Хост нужные настройки получает

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv6/png/resolf_conf_PCB.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv6/png/conn_inf_PCB.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv6/png/sh_ipv6_dhcp_binding.png)

Пинг до R1 интерфейса e0/1 проходит

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv6/png/ping_PCB_R1.png)

конфигурации устройств второй части [S1](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv6/configs_dhcpv6/S1.txt), [S2](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv6/configs_dhcpv6/S2.txt), [R1](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv6/configs_dhcpv6/R1.txt), [R2](https://github.com/buravtsovpavel/nw-labs/blob/main/lab03/DHCPv6/configs_dhcpv6/R2.txt)
