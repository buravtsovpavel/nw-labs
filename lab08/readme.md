## Лабораторная работа 

EIGRP
---
### Цель:

Настроить EIGRP в С.-Петербург (Использовать named EIGRP)

- В офисе С.-Петербург настроить EIGRP.
- R32 получает только маршрут по умолчанию.
- R16-17 анонсируют только суммарные префиксы.
- Использовать EIGRP named-mode для настройки сети.
---

## Решение:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab08/png/topology.png)

`Перед выполнением задания внесены небольшие коррективы в схему - между SW9 и SW10 хотел переделать LAG L2 в LAG L3, но высянилось, что с LAG L3 в EVE-NG возникает баг - некоторые образы не могут корректно сгенерировать MAC-адрес для L3 Port-channel при использовании no switchport. MAC на Port-channel отображался как 0000.0000.0000 и LAG L3 не работал. Поэтому соединил SW9 и SW10 прямым линком и назначил там адреса из 10.0.15.28/30` 


### 1. Настройка EIGP named-mode.

EIGRP имеет два основных режима настройки: классический и именованный (named-mode). Классический режим – это более старый способ настройки, где конфигурация EIGRP разбросана по нескольким командам в конфигурации маршрутизатора и интерфейса. Именованный режим – это более новый и структурированный способ, позволяющий группировать все конфигурации EIGRP под общим именем процесса в режиме маршрутизатора. 

В named-mode вся конфигурация EIGRP хранится в одном блоке.

Контексты named-mode:
- Named Mode глобальный контекст
- address-family автономная система
- af-interface манипуляции с интерфейсами
- topology манипуляции с маршрутами

Задаём процесс EIGRP с именем SPB
```
(config)#router eigrp SPB
```
Далее в рамках этого процесса запускаем отдельный модуль посвящённый ipv4 в котором указываем номер автономной системы.

```
(config-router)#address-family ipv4 unicast autonomous-system 1
```

Включить EIGRP на всех интерфейсах можно командой: (все коннектед сети попадут в анонс, а на интерфейсах подсети которых не следует анонсировать можно включить passive-interface):

```
network 0.0.0.0
```

Либо указать нужные сети в которых находятся интерфейсы для анонса в EIGRP.
Анонсируем все подсети, начинающиеся с 10.0.0.0, включая все подсети внутри этого диапазона:

```
R17(config-router)#address-family ipv4 unicast autonomous-system 1
R17(config-router-af)#network 10.0.0.0 0.255.255.255
```
Можно задать общие настройки для всех интерфейсов в EIGRP:
```
R17(config-router-af)#af-interface default
R17(config-router-af-interface)#passive-interface
R17(config-router-af-interface)#exit-af-interface
```
Все настройки характерные для конкретного интерфейса будут находиться в контексте af-interface
Например, заходя на каждый необходимый интерфейс убираем пассивный режим:
```
R17(config-router-af)#af-interface ethernet 0/1
R17(config-router-af-interface)#no passive-interface
```
и при необходимости делаем какие либо другие настройки, например, включаем суммаризацию summary-address 10.0.0.0/24

R18
```
router eigrp SPB
 !
 address-family ipv4 unicast autonomous-system 1
  !
  af-interface default
   passive-interface
  exit-af-interface
  !
  af-interface Ethernet0/1
   no passive-interface
  exit-af-interface
  !
  af-interface Ethernet0/0
   no passive-interface
  exit-af-interface
  !
  topology base
  exit-af-topology
  network 10.0.0.0
  eigrp router-id 10.2.0.1
 exit-address-family
```
<details>
  <summary>R17</summary>
router eigrp SPB<br>
 !<br>
 address-family ipv4 unicast autonomous-system 1<br>
  !<br>
  af-interface default<br>
   passive-interface<br>
  exit-af-interface<br>
  !<br>
  af-interface Ethernet0/1<br>
   no passive-interface<br>
  exit-af-interface<br>
  !<br>
  af-interface Ethernet0/2<br>
   no passive-interface<br>
  exit-af-interface<br>
  !<br>
  af-interface Ethernet0/0<br>
   no passive-interface<br>
  exit-af-interface<br>
  !<br>
  topology base<br>
  exit-af-topology<br>
  network 10.0.0.0<br>
  eigrp router-id 10.2.0.3<br>
 exit-address-family<br>
</details>


<details>
   <summary>SW10</summary>
router eigrp SPB<br>
 !<br>
 address-family ipv4 unicast autonomous-system 1<br>
  !<br>
  af-interface default<br>
   passive-interface<br>
  exit-af-interface<br>
  !<br>
  af-interface Ethernet0/0<br>
   no passive-interface<br>
  exit-af-interface<br>
  !<br>
  af-interface Ethernet1/0<br>
   no passive-interface<br>
  exit-af-interface<br>
  !<br>
  af-interface Ethernet0/3<br>
   no passive-interface<br>
  exit-af-interface<br>
  !<br>
  topology base<br>
  exit-af-topology<br>
  network 10.0.0.0<br>
  network 172.16.0.0<br>
  eigrp router-id 10.2.0.6<br>
 exit-address-family<br>
</details>


На остальных роутерах аналогично

Таблица маршрутизации с маршрутами полученными по EIGRP после первичной настройки:

 на R18

 ![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab08/png/R18_befor.png)
 
 
### 2. R32 получает только маршрут по умолчанию.

В протоколе EIGRP нет возможности распространить маршрута по умолчанию как это реализовано например в OSPF (default-information originate).

Но есть два способа распространить маршрут по умолчанию на R32:

1. Настроить с помощью команды ```summary-address``` суммарный маршрут 0.0.0.0 0.0.0.0 на интерфейсе маршрутизатора направленном в сторону роутера на котором нужно получать только маршрут по умолчанию.  (на R16 e0/3)
```
 router eigrp SPB
 address-family ipv4 unicast autonomous-system 1
 af-interface e0/3 
 summary-address 0.0.0.0 0.0.0.0
```

2. Отфильтровать всё на границе с помощью distribute-list настроив соответствующий prefix-list. Либо R16 отправляет всё кроме 0.0.0.0 либо R32 принимает только 0.0.0.0.  

**Второй вариант с использованием prefix-list является предпочтительным**, будем использовать его настроив на R32 prefix-list чтобы пропускать только маршрут по умолчанию.

Сначала нужно на R18 создать статический дефолтный маршрут и сделать его редистрибуцию в EIGRP.

сделаем один маршрут до R24 в Триаде:

```
R18(config)#ip route 0.0.0.0 0.0.0.0 200.50.0.21
```

Создаём route-map и prefix-list разрешающий только один маршрут — 0.0.0.0/0, что соответствует дефолтному маршруту.

```
R18(config)#route-map ONLY-DEFAULT permit 10
R18(config-route-map)#match ip address prefix-list ONLY-DEFAULT
R18(config-route-map)#exit
R18(config)#ip prefix-list ONLY-DEFAULT seq 5 permit 0.0.0.0/0
```

Делаем редистребьюцию в EIGRP:
```
R18(config)#router eigrp SPB
R18(config-router)# address-family ipv4 unicast autonomous-system 1
R18(config-router-af)#topology base
R18(config-router-af-topology)#redistribute static route-map ONLY-DEFAULT
R18(config-router-af-topology)#end
```

на R32

Создаём prefix-list чтобы пропускать только маршрут по умолчанию (0.0.0.0/0) и отбрасывать все остальные входящие маршруты.

```
R32(config)#ip prefix-list DEFAULT_ONLY seq 5 permit 0.0.0.0/0
R32(config)#ip prefix-list DEFAULT_ONLY seq 10 deny 0.0.0.0/0 le 32
```

Далее применяем этот prefix-list к distribute-list внутри EIGRP:

```
R32(config)#router eigrp SPB
R32(config-router)#address-family ipv4 unicast autonomous-system 1
R32(config-router-af)#topology base
R32(config-router-af-topology)#distribute-list prefix DEFAULT_ONLY in
R32(config-router-af-topology)#distribute-list prefix DEFAULT_ONLY in
R32(config-router-af-topology)#end
```
В таблице маршрутизации R32 по EIGRP теперь приходит только маршрут по умолчанию:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab08/png/R32_after.png)


### 3. R16-17 анонсируют только суммарные префиксы.

В задании имеется в виду по возможности суммировать и передать все сети снизу на верх в сторону R18.

Все подсети используемые для межроутерных линков которые ниже R16 и R17 можно просуммировать в 10.0.15.0/27

Две клиентские подсети суммируются в 172.16.8.0/22

Суммирование выполняем командой ```summary-address``` на интерфейсах которые смотрят в сторону роутера которому анонсируем суммированный маршрут(R18), т.е. на e0/1 на R16 и R17

```
R16(config)#router eigrp SPB
R16(config-router)#address-family ipv4 unicast autonomous-system 1
R16(config-router-af)#AF-interface ethernet 0/1
R16(config-router-af-interface)#summary-address 10.0.15.0  255.255.255.224
R16(config-router-af-interface)#summary-address 172.16.8.0 255.255.252.0
```
```
R17(config)#router eigrp SPB
R17(config-router)# address-family ipv4 unicast autonomous-system 1
R17(config-router-af)#af-interface ethernet 0/1
R17(config-router-af-interface)#summary-address 10.0.15.0  255.255.255.224
R17(config-router-af-interface)#summary-address 172.16.8.0 255.255.252.0
```

После этого на R18 в таблице маршрутизации видны просуммированные сети:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab08/png/R18_after_2.png)

[Конфигурации](https://github.com/buravtsovpavel/nw-labs/tree/main/lab08/configs) устройств.


