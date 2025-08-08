## Лабораторная работа 

VPN. GRE. DmVPN

---
### Цель:

- Настроить GRE между офисами Москва и С.-Петербург
- Настроить DMVPN между офисами Москва и Чокурдах, Лабытнанги

Описание/Пошаговая инструкция выполнения домашнего задания:

- Настроить GRE между офисами Москва и С.-Петербург.
- Настроить DMVMN между Москва и Чокурдах, Лабытнанги.
- Все узлы в офисах в лабораторной работе должны иметь IP связность.
- План работы и изменения зафиксированы в документации.
---

## Решение:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab13/png/topology.png)

### 1. Настроить GRE между офисами Москва и С.-Петербург.

Сделаем два туннеля R15 - R18 (подсеть для туннеля 10.200.200.0/24) и R14 - R18 (подсеть для туннеля 10.210.210.0/24).
На каждом роутере прописывем статические маршруты до соответствующих подсетей удалённых офисов.
На R18 будет по два маршрута до каждой подсети Московского офиса один из которых с более худшей метрикой - в таблице маршрутизации будет маршрут с более лучшей метрикой, но если tunnel0 упадёт, то в таблицу попадёт второй резервный маршрут.

R14:
```
interface Tunnel0
 ip address 10.210.210.1 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source 200.50.0.2
 tunnel destination 200.50.0.22
end
```
```
ip route 172.16.8.0 255.255.255.0 Tunnel0
ip route 172.16.10.0 255.255.255.0 Tunnel0
```

R15:
```
interface Tunnel0
 ip address 10.200.200.1 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source 200.50.0.10
 tunnel destination 200.50.0.22
end
```
```
ip route 172.16.8.0 255.255.255.0 Tunnel0
ip route 172.16.10.0 255.255.255.0 Tunnel0
```

R18:
```
interface Tunnel0
 ip address 10.200.200.2 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source 200.50.0.22
 tunnel destination 200.50.0.10
end
```
```
interface Tunnel1
 ip address 10.210.210.2 255.255.255.0
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source 200.50.0.22
 tunnel destination 200.50.0.2
end
```
```
ip route 172.16.1.0 255.255.255.0 Tunnel0
ip route 172.16.1.0 255.255.255.0 Tunnel1 10
ip route 172.16.7.0 255.255.255.0 Tunnel0
ip route 172.16.7.0 255.255.255.0 Tunnel1 10
```
Проверяем связность

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab13/png/GRE/VPC1_ping_1.png)


![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab13/png/GRE/VPC8_ping_1.png)


В wireshrk на промежуточных интерфейсах видно, что туннель работает корректно - есть внешний заголовок IP у которого поле Protocol 47 указывает, что в нём вложение GRE ,  далее GRE заголовок и внутренний заголовок IP.


![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab13/png/GRE/wireshark_gre_1.png)


![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab13/png/GRE/wireshark_gre_2.png)


### 2. Настроить DMVMN между Москва и Чокурдах, Лабытнанги.

<details>
<summary>Перед настройкой непосредственно DMVPN для его корректной работы были внесены следующие изменения в конфигурацию:</summary>


1. Для DMVPN нам необходимо иметь underlay связность по внешним ip между офисами(R15, R27 и R28). Поскольку ранее в анонсах подсетей Чокурдах и Лабытнанги не было необходимости и мы их не анонсировали, то анонсируем сейчас. 
2. Чокурадх и Лабытнанги будут как и Москва в AS1001, поэтому, что бы по механизму защиты от петель R15, R27 и R28 не отбрасвали маршруты из eBGP в которых содержится своя AS используем на eBGP соседа команду: `neighbor 200.50.0.37 allowas-in` (и для iBGP в overlay тоже.)
3. R25 получит анонс 200.50.0.36/30 от R27 но не занесёт его себе в RIB, т.к. есть лучший маршрут - эта сеть у него connected, но он анонсирует маршруты на R23 (который в lab 10 п.2 настраивали как Route Reflector). R23 получит маршруты, но не занесёт их в RIB, т.к. не знает до них hext-hop. Поэтому на R25 пропишем `neighbor 10.3.0.1 next-hop-self`, он подставит при анонсе свой ip 10.3.0.2 до которого у R23 есть маршрут и R23 добавит себе маршруты.
3. На R21 разрешить новые префиксы для R15( п. 4 "BGP. Фильтрация" настраивали,  что бы отдавался только маршрут по умолчанию и префикс офиса С.-Петербург.)
4. **Общая политика анонсов для R15, R27 и R28(иначе были петли и bgp падал)**  - eBGP соседям анонсируем только публичные сети, серые вида 172.16.0.0 котороые будем анонсировать по iBGP в overlay не анонсируем. iBGP соседям (это будет маршрутизация в overlay) наоброт - запрещаем анонсировать сети вида 200.50.0.0, а серые сети 172.16.0.0 разрешаем. Для этого навешиваем соответствующие route-map.
5. Так же на R25, R26, R28 нам больше не нужны статические маршруты которые настраивались в ЛР lab05 PBR для проверки работы политики маршрутизации.
Удаляем эти маршруты.
6. Ухудшина стоимость линков в OSPF на R12 и R13 которые смотрят на R14, что бы трафик от 172.16.1.0/24 и 172.16.7.0/24 ходил через R15:

```
interface Ethernet0/2
 description Link to R14
 ip address 10.0.10.5 255.255.255.252
 ip ospf 1 area 10
 ip ospf cost 100
end
```
7. добавляем в Чокурдах коммутатор, VPC27 (там будет сеть пользователей 172.16.40.0/24)

topology_1.png
</details>

Будем настраивать DMVPN Phase2. R15 - HUB, R27 и R28 - Spoke. HUB - route-reflector + next-hop-self будет отражать маршруты от споков заменяя next-hop на свой туннельный интерфейс, плюс для споков анонсирует дефолт через свой туннельный интерфейс 10.115.115.1. (neighbor SPOKES default-originate)

HUB во 2 фазе - маршрутизатор маршрутов, но не маршрутизауор трафика. Через него трафик от спока к споку уже не ходит. Spoke2 сначала отправляет NHRP Resolution Request- он хочет узнать информацию о NBMA (белом адресе) адресе Spoke1, в самом Request уже содержится информация о белом адресе Spoke2. Запрос отправляется на Hub, тот его транслирует на Spoke1 и Spoke1 уже отправляет NHRP Resolution Reply напрямую Spoke2 («строй туннель до меня — вот мой белый адрес»). Каждый Spoke хранит полную таблицу маршрутизации всех сеток, которые есть за другими Spok’ами.
(в Phase3 при необходимости можно переделать тремя командами - убрать на HUB роль route-reflector и добавить `ip nhrp redirect` `ip nhrp shortcut`.)

для overlay выбрана подсеть 10.115.115.0/24

R15:
```
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
end
```

R27:
```
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
end
```

R28:
```
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
end
```

Маршрутизация. 

- устанавливаем iBGP соседство в overlay
- навешиваем route-map которые запретят передавать в underlay серые сети и в overlay внешние. (иначе возникали петли.)
- анонсируем серые пользовательские сети
- на HUB для споков делаем peer-group
- HUB - route-reflector с next-hop-self и анонсирует дефолт.

R15:
```
R15# sh run | s r b
router bgp 1001
 bgp router-id 15.15.15.15
 bgp log-neighbor-changes
 bgp listen range 10.115.115.0/24 peer-group SPOKES
 network 172.16.1.0 mask 255.255.255.0
 network 172.16.7.0 mask 255.255.255.0
 network 200.50.0.8 mask 255.255.255.252
 neighbor SPOKES peer-group
 neighbor SPOKES remote-as 1001
 neighbor SPOKES route-reflector-client
 neighbor SPOKES next-hop-self
 neighbor SPOKES default-originate
 neighbor SPOKES allowas-in
 neighbor SPOKES soft-reconfiguration inbound
 neighbor SPOKES route-map BGP-OUT-FILTER out
 neighbor 10.1.0.1 remote-as 1001
 neighbor 10.1.0.1 update-source Loopback0
 neighbor 10.1.0.1 soft-reconfiguration inbound
 neighbor 10.1.0.1 route-map BLOCK-TRANSIT-AS301-OUT out
 neighbor 200.50.0.9 remote-as 301
 neighbor 200.50.0.9 allowas-in
 neighbor 200.50.0.9 soft-reconfiguration inbound
 neighbor 200.50.0.9 route-map LOCPREF_AS301 in
 neighbor 200.50.0.9 route-map BLOCK-R14 out
```

R27:

```
R27#sh run | s r b
router bgp 1001
 bgp router-id 27.27.27.27
 bgp log-neighbor-changes
 network 172.16.40.0 mask 255.255.255.0
 network 200.50.0.36 mask 255.255.255.252
 neighbor 10.115.115.1 remote-as 1001
 neighbor 10.115.115.1 allowas-in
 neighbor 10.115.115.1 soft-reconfiguration inbound
 neighbor 10.115.115.1 route-map BGP-IN-FILTER in
 neighbor 10.115.115.1 route-map BGP-OUT-FILTER out
 neighbor 200.50.0.37 remote-as 520
 neighbor 200.50.0.37 allowas-in
 neighbor 200.50.0.37 soft-reconfiguration inbound
 neighbor 200.50.0.37 route-map BLOCK-R25 out
```

R28:

```
R28#sh run | s r b
router bgp 1001
 bgp router-id 28.28.28.28
 bgp log-neighbor-changes
 network 172.16.30.0 mask 255.255.255.0
 network 172.16.31.0 mask 255.255.255.0
 network 200.50.0.28 mask 255.255.255.252
 network 200.50.0.32 mask 255.255.255.252
 neighbor 10.115.115.1 remote-as 1001
 neighbor 10.115.115.1 allowas-in
 neighbor 10.115.115.1 soft-reconfiguration inbound
 neighbor 10.115.115.1 route-map BGP-IN-FILTER in
 neighbor 10.115.115.1 route-map BGP-OUT-FILTER out
 neighbor 200.50.0.29 remote-as 520
 neighbor 200.50.0.29 allowas-in
 neighbor 200.50.0.29 soft-reconfiguration inbound
 neighbor 200.50.0.29 route-map BLOCK-R25-R26 out
 neighbor 200.50.0.33 remote-as 520
 neighbor 200.50.0.33 allowas-in
 neighbor 200.50.0.33 soft-reconfiguration inbound
 neighbor 200.50.0.33 route-map BLOCK-R25-R26 out
```

BGP UPDATE для overlay выглядит так: внешний IP заголовок, GRE, внутренний IP заголовок, TCP и внутри полезная нагрузка - BGP UPDATE.

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab13/png/DMVPN/Wireshark_bgp_UPDATE.png)


Проверяем работу DMVPN:

HUB:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab13/png/DMVPN/R15_show_dmvpn.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab13/png/DMVPN/R15_sh_ip_nhrp.png)


Spoke:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab13/png/DMVPN/R27_show_dmvpn.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab13/png/DMVPN/R27_nhrp_brief.png)

Все конечные сети за R15, R27 и R28 пингуются и трафик со Spoke на Spoke ходит напрямую (в  промежуточных хопах нет туннельного интерфейса HUB, а сразу туннельный интерфейс нужного Spoke)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab13/png/DMVPN/VPC27_ping_VPC30.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab13/png/DMVPN/VPC30_ping_VPC27.png)

[Конфигурации](https://github.com/buravtsovpavel/nw-labs/tree/main/lab13/configs) устройств

