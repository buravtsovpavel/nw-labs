## Лабораторная работа 
OSPF
---
### Цель:
Настроить OSPF офисе Москва
Разделить сеть на зоны
Настроить фильтрацию между зонами


1. Маршрутизаторы R14-R15 находятся в зоне 0 - backbone.
2. Маршрутизаторы R12-R13 находятся в зоне 10. Дополнительно к маршрутам должны получать маршрут по умолчанию.
3. Маршрутизатор R19 находится в зоне 101 и получает только маршрут по умолчанию.
4. Маршрутизатор R20 находится в зоне 102 и получает все маршруты, кроме маршрутов до сетей зоны 101.
5. План работы и изменения зафиксированы в документации .
---

## Решение

### 1. Маршрутизаторы R14-R15 находятся в зоне 0 - backbone.

Поскольку в пределах области backbone area 0 все роутеры должны иметь прямые соединения или маршрутизироваться через соседние роутеры без разрывов, то в изначальную топологию добаляем линк между R14 и R15 и назначаем на него ip адреса из сети 10.0.10.40/30. На R14 10.0.10.41/30, на R15 10.0.10.42/30

Получается топология:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab06/png/OSPF_Moscow.png)


Включить OSPF на интерфейсе можно двумя способами: через команду network в процессе настройки OSPF и прямо на интерфейсе командой ip ospf. Команда ip ospf area на интерфейсе имеет абсолютный приоритет и явно заставляет интерфейс участвовать в OSPF в указанной области, независимо от настроек network.

Включаем OSPF на loopback'ах через network в area 0, а на остальных интерфейсах через настройку непосредственно на интерфейсе.

на R14:

```
router ospf 1
 router-id 14.14.14.14
 network 10.1.0.1 0.0.0.0 area 0
```

```
interface Ethernet1/0
 description Link to R15
 ip address 10.0.10.41 255.255.255.252
 ip ospf 1 area 0
```
на R15 аналогично.

Теперь R14 и R15 в одной зоне 0 - backbone, между ними сформировано соседство

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab06/png/neig_area0.png)

На остальных интерфейсах включаем OSPF в соответствующие зоны.


### 2.  Маршрутизаторы R12-R13 находятся в зоне 10. Дополнительно к маршрутам должны получать маршрут по умолчанию.

Включаем на всех интерфейсах роутеров R12, R13 OSPF в area 10. Для того что бы они получали ещё маршрут по умолчанию на R14 и R15 допишем default-information originate always (инжектировать default route в OSPF всегда. вне зависимости присутстсвует он в ТМ или нет.)

Проверяем, что в таблице маршрутизации появился маршрут по умолчанию:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab06/png/R12_R13_sh_route_1.png)

Сразу на SW4 и SW5 включаем OSPF через "network" в area 10 на всех интерфейсах включая loopback, интерфейсы которые смотрят на конечных пользователей делаем пассивными (OSPF не отправляет OSPF UPDATE через эти интерфейсы, предотвращая распространение маршрутов и отправку Hello сообщений.)

SW4

```
router ospf 1
 router-id 4.4.4.4
 passive-interface Ethernet0/0
 passive-interface Ethernet0/1
 network 10.0.10.32 0.0.0.3 area 10
 network 10.0.10.36 0.0.0.3 area 10
 network 10.1.0.8 0.0.0.0 area 10
 network 172.16.1.0 0.0.0.255 area 10
 network 172.16.7.0 0.0.0.255 area 10
 network 172.16.50.0 0.0.0.255 area 10
```

SW5

```
router ospf 1
 router-id 5.5.5.5
 passive-interface Ethernet0/0
 passive-interface Ethernet0/1
 network 10.0.10.24 0.0.0.3 area 10
 network 10.0.10.28 0.0.0.3 area 10
 network 10.1.0.7 0.0.0.0 area 10
 network 172.16.1.0 0.0.0.255 area 10
 network 172.16.7.0 0.0.0.255 area 10
 network 172.16.50.0 0.0.0.255 area 10
```

Проверяем, что маршруты и маршрут по умолчанию получены:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab06/png/ip_route_area_10.png)


### 3. Маршрутизатор R19 находится в зоне 101 и получает только маршрут по умолчанию.

Для выполнения данного условия зона 101 будет totally stubb area (блокируются LSA 3,4,5)

Конфигурацию производим на ABR - роутер R14 

```
are 101 stub no-summary      
```
```
router ospf 1
 router-id 14.14.14.14
 area 101 stub no-summary
 network 10.1.0.1 0.0.0.0 area 0
 default-information originate always
```

и на всех маршрутизаторах зоны - в нашем случае это только единственный R19

```
area 101 stub
```

Включаем OSPF на loopback'ах через network в area 0, а на интерфейсе e0/0 через настройку непосредственно на интерфейсе.

```
router ospf 1
 router-id 19.19.19.19
 area 101 stub
 network 10.1.0.6 0.0.0.0 area 101
```

```
interface Ethernet0/0
 description Link to R14
 ip address 10.0.10.1 255.255.255.252
 ip ospf 1 area 101
```
Убеждаемся, что он получает только маршрут по умолчанию.

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab06/png/R19_ip_route_ospf.png)

### 4. Маршрутизатор R20 находится в зоне 102 и получает все маршруты, кроме маршрутов до сетей зоны 101.

Сначала настраиваем OSPF на R20 - включаем его интерфейсы OSPF в area 102.

он полчает все маршруты, в том числе из area 101

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab06/png/R20_befor.png)

Type 3 LSA создаются ABR, поэтому фильтрация возможна только на ABR. 
На R15 создаём prefix-list которые запретят подсети из area 101 и применим их к OSPF

```
ip prefix-list BLOCK_NET1_101 seq 5 deny 10.0.10.0/30
ip prefix-list BLOCK_NET2_101 seq 10 deny 10.1.0.1/32
ip prefix-list BLOCK_NETS seq 20 permit 0.0.0.0/0 le 32
```
```
router ospf 1
 router-id 15.15.15.15
 area 102 filter-list prefix BLOCK_AREA101 in
 network 10.1.0.2 0.0.0.0 area 0
 default-information originate always
```


![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab06/png/R20_after.png)


Маршруты до 10.0.10.0/30 и 10.1.0.6/32 исчезли из таблицы маршрутизации R19.

[Конфиги](https://github.com/buravtsovpavel/nw-labs/tree/main/lab06/configs) устройств.
