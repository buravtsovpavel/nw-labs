## Лабораторная работа 

BGP. Основы
---
### Цель:

Настроить BGP между автономными системами. Организовать доступность между офисами Москва и С.-Петербург.

- Настроите eBGP между офисом Москва и двумя провайдерами - Киторн и Ламас.
- Настроите eBGP между провайдерами Киторн и Ламас.
- Настроите eBGP между Ламас и Триада.
- Настроите eBGP между офисом С.-Петербург и провайдером Триада.
- Организуете IP доступность между пограничным роутерами офисами Москва и С.-Петербург.
- План работы и изменения зафиксированы в документации.
---

## Решение:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab09/png/topology.png)

По условию требуется настроить eBGP между: 
- R14 <-> R22, R15 <-> R21
- R22 <-> R21
- R21 <-> R24
- R18 <-> R24, R18 <-> R26

и организовать IP доступность между пограничным роутерами офисами Москва и С.-Петербург. 
Сделаем ip-связность по loopback'ам между офисами Москва и С.-Петербург, а так же по ip внешних подсетей, на которых установлены eBGP-соседства с промежуточными AS.

Этапы базовой настройки BGP:
1) указать номер собственной AS;  
2) настроить идентификатор маршрутизатора (router-id); 
3) указать информацию о BGP-соседях;
4) указать, какие сети необходимо анонсировать соседним роутерам;
5) настроить политики и фильтры для принимаемых и анонсируемых маршрутов при необходимости;
6) проверить работу протокола BGP

### Настройка между Москвой R14 и провайдером Киторн R22

Команды для настройки:

`Router(config)# router bgp <ASN>`

`Router(config-router)# bgp router-id ID`

`Router(config-router)# neighbor <address> remote-as <ASN>`

Для анонса префиксов в другую AS существует три варианта:

- Определить сети командой `network`
- Импортировать из другого источника (direct, static, IGP)
- Создать агрегированный маршрут командой aggregate-address

`Router(config-router)# network <subnet> [mask <mask>]`

Так же для того, что бы в при проверке можно было просматривать префиксы предлагаемые соседу и полученные от соседа включаем настройку мягкой конфигурации - хранить в памяти и не лету обновлять анонсы принимаемых маршрутов.

`neighbor <neighbor-IP> soft-reconfiguration in`

Сначала анонсируем только loopback.

настройка R14:
```
router bgp 1001
 bgp router-id 14.14.14.14
 bgp log-neighbor-changes
 network 10.1.0.1 mask 255.255.255.255
 neighbor 200.50.0.1 remote-as 101
 neighbor 200.50.0.1 soft-reconfiguration inbound
```

настройка R22:

```
router bgp 101
 bgp router-id 22.22.22.22
 bgp log-neighbor-changes
 neighbor 200.50.0.2 remote-as 1001
 neighbor 200.50.0.2 soft-reconfiguration inbound
 neighbor 200.50.0.6 remote-as 301
 neighbor 200.50.0.6 soft-reconfiguration inbound
```

#### Настраиваем аналогичным образом eBGP между остальными AS, на R15 и R18 анонсировав loopback'и

Видно, что R22 получает один префикс от R14 и два от R21

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab09/png/R22_bgp_sum_lo.png)

А R21 от каждого соседа по одному

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab09/png/R21_bgp_sum_lo.png)


Пинг по loopback'ам проходит. (Если не указывать интерфейс-источник, то пинг не пройдёт, т.к. источником в пакете будет указан внешний интерфейс маршрутизатора, а эти префиксы пока не анонсировали и роутер получатель не будет знать куда отправить echo replay.)


![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab09/png/R14_ping_source.png)


![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab09/png/R18_ping_source.png)


Теперь для доступности между пограничным роутерами офисами Москва и С.-Петербург анонсируем внешние подсети, на которых установлены BGP-соседства с промежуточными AS.

R14:
```
router bgp 1001
 bgp router-id 14.14.14.14
 bgp log-neighbor-changes
 network 10.1.0.1 mask 255.255.255.255
 network 200.50.0.0 mask 255.255.255.252
 neighbor 200.50.0.1 remote-as 101
 neighbor 200.50.0.1 soft-reconfiguration inbound
```

R15:
```
router bgp 1001
 bgp router-id 15.15.15.15
 bgp log-neighbor-changes
 network 10.1.0.2 mask 255.255.255.255
 network 200.50.0.8 mask 255.255.255.252
 neighbor 200.50.0.9 remote-as 301
 neighbor 200.50.0.9 soft-reconfiguration inbound
```
R18:
```
router bgp 2042
 bgp router-id 18.18.18.18
 bgp log-neighbor-changes
 network 10.2.0.1 mask 255.255.255.255
 network 200.50.0.20 mask 255.255.255.252
 neighbor 200.50.0.21 remote-as 520
 neighbor 200.50.0.21 soft-reconfiguration inbound
 neighbor 200.50.0.25 remote-as 520
 neighbor 200.50.0.25 soft-reconfiguration inbound
```

### Проверка:

Проверим на R14 какие префиксы мы анонсируем для R22 (это таблица adj-RIB-out. В неё попадают те префиксы которые мы будем соседу передавать. Соседям мы будем анонсировать только лучшие маршруты, т.е. только то, что попадёт от BGP в глобальную таблицу маршрутизации (основная ТМ Routing Table `sh ip route`)) и ещё к ним применятся наши политики на out.)

`sh ip bgp neighbors 200.50.0.1 advertised-routes`

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab09/png/2/R14_adv.png)


Проверим на R14 префиксы поступаемые от R22 (таблица adj-RIB-in, ассоциированая только с этим соседом и туда попадает маршрутная информация без каких-либо обработок. Далее к этой таблице применяется политика которая у нас применена к тому или иному соседу.)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab09/png/2/R14_rec.png)



Проверяем Loc-RIB на R14 (то, что прошло фильтрацию и сохранено в BGP. из этой таблицы после применения Best Path selection лучшие маршруты попадают в основную ТМ Routing Table)


![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab09/png/2/R14_sh_bpg.png)


И маршруты полученные на R14 по BGP попавшие в основную таблицу маршрутизации Routing Table:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab09/png/2/R14_ip_route_bgp.png)


На R21:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab09/png/2/R21_ip_route_bgp.png)


На R18:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab09/png/2/R21_ip_route_bgp.png)


Проверяем связность:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab09/png/2/R14_ping.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab09/png/2/R14_trace.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab09/png/2/R18_ping.png)


[Конфигурации](https://github.com/buravtsovpavel/nw-labs/tree/main/lab09/configs) устройств
