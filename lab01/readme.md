## Лабораторная работа 
VLAN
---
Цель: Настройка DTP. Добавление сетей VLAN и назначение портов.

*Примечание*

_Исходя из предоставленых образов IoL в исходное задание из методички внесены корректировки используемых интерфесов и соответствующие им корректировки в Таблицу VLAN_


### Дана топология:
(topologu.png)

---
Таблица адресации

 Device    | Interface | IP Address   | Subnet Mask   | Default Gateway
---------- | --------- | --------     | ------------- | -----------------
R1         | Eth 0/0.3  | 192.168.3.1  | 255.255.255.0 | N/A
 \-        | Eth 0/0.4  | 192.168.4.1  | 255.255.255.0 | 
 \-        | Eth 0/0.8  | N/A          | N/A
S1         | VLAN 3    | 192.168.3.11 | 255.255.255.0 | 192.168.3.1
S2         | VLAN 3    | 192.168.3.12 | 255.255.255.0 | 192.168.3.1
PC-A       | NIC       | 192.168.3.3  | 255.255.255.0 | 192.168.3.1
PC-B       | NIC       | 192.168.4.3  | 255.255.255.0 | 192.168.4.1


Таблица VLAN

VLAN    | Name       | Interface Assigned
--------| ----       | ------------------
3       | Management | S1: VLAN 3,  S2: VLAN 3,  S1: Eth 0/0 
4       | Operations | S2: Eth 0/0
7       | ParkingLot | S1: Eth 0/3-4;  S2: Eth 0/2-4
8       | Native     | N/A


### Цели:

* Part 1: Build the Network and Configure Basic Device Settings
* Part 2: Create VLANs and Assign Switch Ports 
* Part 3: Configure an 802.1Q Trunk between the Switches 
* Part 4: Configure Inter-VLAN Routing on the Router 
* Part 5: Verify Inter-VLAN Routing is working 
------------

### 1. Собираем топологию и производим базовую настройку устройств:

(hostname, отключение автоматического выполнения DNS-запросов, установка ашифрованного пароля привилегированного режима, установка пароля для доступа через консоль, установка пароля на vty для доступа по протоколам telnet или ssh, шифруем пароли, создаём баннер)


Пример настройки для S1

```
Switch>enable
Switch#conf t
Switch#conf terminal
Enter configuration commands, one per line.  End with CNTL/Z.
Switch(config)#hos
Switch(config)#hostname S1
S1(config)#no ip dom
S1(config)#no ip domain loo
S1(config)#no ip domain lookup
S1(config)#ena
S1(config)#enable sec
S1(config)#enable secret class
S1(config)#lin
S1(config)#line co
S1(config)#line console 0
S1(config-line)#pas
S1(config-line)#password cisco
S1(config-line)#exi
S1(config-line)#exit
S1(config)#lin
S1(config)#line vty
S1(config)#line vty 0 4
S1(config-line)#pass
S1(config-line)#password cisco
S1(config-line)#log
S1(config-line)#login
S1(config-line)#login
S1(config-line)#exi
S1(config-line)#exit
S1(config)#servi
S1(config)#service pass
S1(config)#service password-encryption
S1(config)#banner motd #
Enter TEXT message.  End with the character '#'.
***********************************************
This is a secure system. Authorized Access Only!
***********************************************
#
S1(config)#
S1(config)#clo
S1(config)#clock timezone YEKT 5 0
S1(config)#
*Mar  9 06:39:17.439: %SYS-6-CLOCKUPDATE: System clock has been updated from 06:39:17 UTC Sun Mar 9 2025 to 11:39:17 YEKT Sun Mar 9 2025, configured from console by console.
S1(config)#
S1(config)#
S1(config)#exi
S1(config)#exit
```
S2 и R1 конфигурируем схожим образом.

### 2. Создаём vlan'ы и назначаем их на порты коммутаторов.

Создаём vlan'ы в соответствии с таблицей, интерфейсы управления, шлюз по умолчанию на коммутаторах. Все неиспользуемы порты добавляем в VLAN 7  (ParkingLot) и администротивно выключаем.

S1:

```
S1(config)#vlan 3
S1(config-vlan)#name Management
S1(config-vlan)#exit
S1(config)#vlan 7
S1(config-vlan)#name ParkingLot
S1(config-vlan)#exit
S1(config)#vlan 8
S1(config-vlan)#name Native
S1(config-vlan)#exit
S1(config)#vlan 4
S1(config-vlan)#name Operations
S1(config-vlan)#exit
S1(config)#interface vlan 3
S1(config-if)#ip address 192.168.3.11 255.255.255.0
S1(config-if)#no shutdown
S1(config-if)#exit
S1(config)#ip default-gateway 192.168.3.1
S1(config)#int ethernet 0/3
S1(config-if)#sw mode access
S1(config-if)#switchport access vlan 7
S1(config-if)#shutdown
S1(config-if)#
*Mar  9 09:52:12.196: %LINK-5-CHANGED: Interface Ethernet0/3, changed state to administratively down
*Mar  9 09:52:13.196: %LINEPROTO-5-UPDOWN: Line protocol on Interface Ethernet0/3, changed state to down
S1(config-if)#
```

<details>
  <summary>S2</summary>
S2(config)#vlan 3<br>
S2(config-vlan)#name Management<br>
S2(config-vlan)#exit<br>
S2(config)#vlan 4<br>
S2(config-vlan)#name Operations<br>
S2(config-vlan)#exit<br>
S2(config)#vlan 7<br>
S2(config-vlan)#name ParkingLot<br>
S2(config-vlan)#exit<br>
S2(config)#vlan 8<br>
S2(config-vlan)#name Native<br>
S2(config-vlan)#exit<br>
S2(config)#interface vlan 3<br>
S2(config-if)#ip address 192.168.3.12 255.255.255.0<br>
S1(config-if)#no shutdown
S2(config-if)#exit<br>
S2(config)#ip default-gateway 192.168.3.1<br>
S2(config)#exit<br>
S2(config)#interface range ethernet0/2 , et0/3<br>
S2(config-if-range)#switchport mode access<br>
S2(config-if-range)#switchport access vlan 7<br>
S2(config-if-range)#shutdown<br>
</details>

Назначаем порты коммутаторов в соответствующие VLAN.

S1:
```
S1(config)#interface ethernet 0/0
S1(config-if)#switchport mode access
S1(config-if)#switchport access vlan 3
```

<details>
  <summary>S2</summary>
S2(config)#interface ethernet 0/0<br>
S2(config-if)#switchport mode access<br>
S2(config-if)#switchport access vlan 4<br>
</details>

### 3. Конфигурируем транковый канал стандарта 802.1Q между коммутаторами.

На S1:
- порт e0/2 - транковый до маршрутизатора R1 (разрешены vlan 3,4,8)
- порт e0/1 - транковый до S2 (разрешены vlan 3,4,8) 
- vlan 8 - nativ vlan

На S2:
- порт e0/1 - транковый до S1 (разрешены vlan 3,4,8)
 
```
S1(config)#interface ethernet 0/2
S1(config-if)#switchport trunk encapsulation dot1q
S1(config-if)#switchport mode trunk
S1(config-if)#switchport trunk allowed vlan 3,4,8
S1(config-if)#switchport trunk native vlan 8
S1(config-if)#do wr
Building configuration...
Compressed configuration from 1359 bytes to 890 bytes[OK]
S1(config-if)#exit
S1(config)#interface ethernet 0/1
S1(config-if)#switchport trunk encapsulation dot1q
S1(config-if)#switchport mode trunk
S1(config-if)#switchport trunk allowed vlan 3,4,8
S1(config-if)#switchport trunk native vlan 8
```

<details>
  <summary>S2</summary>
S2(config)#interface ethernet 0/1<br>
S2(config-if)#switchport trunk encapsulation dot1q<br>
S2(config-if)#switchport mode trunk<br>
S2(config-if)#switchport trunk allowed vlan 3,4,8<br>
S2(config-if)#switchport trunk native vlan 8<br>
</details>

два скрина


### 4. Конфигурируем маршрутизацию между сетями VLAN на маршрутизаторе. ("Роутер на палочке")

Настроиваем сабинтерфейсы (sub-interfaces) для каждой VLAN

```
R1(config)#interface ethernet 0/0.3
R1(config-subif)#encapsulation dot1Q 3
R1(config-subif)#ip address 192.168.3.1 255.255.255.0
R1(config-subif)#exit
R1(config)#interface ethernet 0/0.4
R1(config-subif)#encapsulation dot1Q 4
R1(config-subif)#ip address 192.168.4.1 255.255.255.0
R1(config-subif)#exit
R1(config)#interface ethernet 0/0.8
R1(config-subif)#encapsulation dot1Q 8 native
R1(config-subif)#exit
```

R1_int_br

### 5. Производим проверку маршрутизации между VLAN.

Пингуем с PC-A его шлюз по умолчанию

Пингуем с PC-A PC-B 

Пингуем с PC-A S2

Запускаем tracert с PC-B до PC-A

Запускаем tracert с PC-A до PC-B

(конфиги устройств: R1, S1, S2, PC-A, PC-B)
