## Лабораторная работа 

iBGP
---
### Цель:

- Настроить iBGP в офисе Москва
- Настроить iBGP в сети провайдера Триада
- Организовать полную IP связанность всех сетей

Описание/Пошаговая инструкция выполнения домашнего задания:

- Настроите iBGP в офисом Москва между маршрутизаторами R14 и R15.
- Настроите iBGP в провайдере Триада, с использованием RR.
- Настройте офиса Москва так, чтобы приоритетным провайдером стал Ламас.
- Настройте офиса С.-Петербург так, чтобы трафик до любого офиса распределялся по двум линкам одновременно.
- Все сети в лабораторной работе должны иметь IP связность.
- План работы и изменения зафиксированы в документации.
---

## Решение:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab10/png/topology.png)


## Введение

Два механизма защиты от петель используемые в iBGP вводят свои нюансы в настройку iBGP:

1. поскольку в iBGP в связи с работой механизма защиты от петель Split Horizon маршруты полученные от iBGP соседа роутер дальше следующему iBGP соседу не передаёт, то для передачи маршрутов можно использовать два варианта:

- FULL MESH (устанавливается соседство каждого с каждым)
- route reflector (настраиваем в п.2)

Соотвественно, что бы соседство установилось между роутерами должна быть ip-связность. Если мы устанавливаем соседство на connected-интерфейсах напрямую подключённых друг к другу, то проблем не будет, а если по loopback'ам(это приоритетный вариант), то нужно сначала организовать по ним связность, например, с помощью IGP (OSPF).


2. При передаче маршрута по iBGP мы не меняем next-hop. И может получиться там, что внутренний роутер получит от eBGP соседа маршрут и там будет next-hop про который он может не знать и это исправляется с помощью команды `next-hop-self`.
Например, `neighbor 172.16.10.2 next-hop-self`   Это команда подставляет в качестве next-hop адрес самого iBGP-соседа.
Но достижимость этого подставленного next-hop’a опять же должна быть и обычно это достигается через IGP (OSPF, EIGRP, IS-IS).


---


### 1. Настроим iBGP в офисе Москва между маршрутизаторами R14 и R15.

В целом настройка iBG соседства схожа с eBGP, только в качестве remote-as указываем свою внутреннюю AS.
И для установления соседства лучше использовать loopback-интерфейсы.

командой `update-source` указываем из под какого интрефейса устанавливать соседство
(по умолчанию BGP будет использовать IP адрес интерфейса, через который он доберётся до соседа)

`neighbor x.x.x.x update-source loopback0`

В нашем случае после того как будет установлено iBGP соседство роутеры получат по BGP маршруты до внешний подсети соседа и в дальнейшем если из вне придёт какой-нибудь анонс, то в next-hop будет стоять адрес этой внешней подсети и  next-hop-self не понадобится, т.к. они уже будут его знать. Но если понадобится передавать префиксы вниз, то нужно будет использовать next-hop-self, что бы next-hop менялся на адрес подсети по которой будет выстраиваться соседство с нижестоящими роутерами, а его они уже знают по OSPF.

R14:
```
R14(config-router)#neighbor 10.1.0.2 remote-as 1001
R14(config-router)#neighbor 10.1.0.2 update-source loopback 0
R14(config-router)#neighbor 10.1.0.2 soft-reconfiguration inbound
```
R15:
```
R15(config-router)#neighbor 10.1.0.1 remote-as 1001
R15(config-router)#neighbor 10.1.0.1 update-source loopback 0
*Jun 19 09:15:23.549: %BGP-5-ADJCHANGE: neighbor 10.1.0.1 Up
R15(config-router)#neighbor 10.1.0.1 soft-reconfiguration inbound
```

Соседство iBGP установилось и теперь R14 до подсетей С.-Петербурга знает по BGP два валидных маршрута, лучший из которых заносится в основную таблицу маршрутизации

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab10/png/R14_sh_ip_bgp_summary.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab10/png/R15_sh_ip_bgp_summary.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab10/png/R14_sh_ip_bgp_1.png)


на R14 маршруты до 10.2.0.1/32 и  200.50.0.20/30 пришедшедшие от R22 проигрывает маршрутам пришедшим от R15 по AS-Path и поэтому в итоговую таблицу маршрутизации попадают маршруты пришедшие от R15 и значит далее они должны анонсироваться по BGP, но поскольку в iBGP обратно тому от кого они пришли не анонсируются(в eBGP тоже), то в анонсе от R14 для R15 их нет. А в анонсе для R22 он предлагает в том числе префиксы пришедшие через R15.

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab10/png/R14_sh_ip_route.png)


### 2. Настроим iBGP в провайдере Триада, с использованием RR.

В качестве Route Reflector выберем R23. 


Route Reflector по умолчанию не меняет next-hop при отражении маршрутов. Он оставляет next-hop таким, каким он был в оригинальном BGP UPDATE-сообщении от клиента. Поэтому, если у клиента которому он отразит маршрут не будет маршурта до клиента отправившего UPDATE, то он будет для него не достижим. В таком случае нужно на Route Reflector использовать `neighbor X.X.X.X next-hop-self`
В нашем случае для RR это не актуально, т.к. ранее настроен ISIS и роутеры знают как друг до друга добраться.

Поскольку  при передаче префиксов для iBGP next-hop не меняется, то R24 получив префиксы по eBGP будет их передавать на R23 (Route Reflector) с ip адресом стыковочного интерфейса того роутера от которого их получил. Поэтому используем на R24 `next-hop-self`, а  маршрут до его адреса (loopback) уже известен всем роутерам внутри AS, т.к. ранее настраивали IS-IS.

При настройке Route Reflector (RR) в iBGP соседство может устанавливаться как по loopback-интерфейсам, так и по физическим интерфейсам, но т.к. рекомендуется использовать loopback-интерфейсы, то будем устанавливать соседство по ним.

Создаём peer-group AS520-RR. Это позволит упростить конфигурацию, так как вы общие настройки будут применены к группе, а не к каждому соседу по отдельности.

R23

```
router bgp 520
 bgp router-id 23.23.23.23
 bgp log-neighbor-changes
 neighbor AS520-RR peer-group
 neighbor AS520-RR remote-as 520
 neighbor AS520-RR update-source Loopback0
 neighbor AS520-RR route-reflector-client
 neighbor AS520-RR soft-reconfiguration inbound
 neighbor 10.3.0.2 peer-group AS520-RR
 neighbor 10.3.0.3 peer-group AS520-RR
 neighbor 10.3.0.4 peer-group AS520-RR
```

после того как настроили R23 RR и двух клиентов (**пока без R24**) видно, что R26 предлагает R23 два префикса, R23 их получает,но у R23 нет маршрута до их next-hop и он их себе не заносит в таблицу маршрутизации bgp


![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab10/png/RR/R26_adv_1.png)


![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab10/png/RR/R23_rec_1.png)


Теперь настроим соседство с R24(**пока без next-hop-self**) и он должен анонсировать для R23 много маршрутов

R24

```
router bgp 520
 bgp router-id 24.24.24.24
 bgp log-neighbor-changes
 neighbor 10.3.0.1 remote-as 520
 neighbor 10.3.0.1 next-hop-self
 neighbor 10.3.0.1 soft-reconfiguration inbound
 neighbor 200.50.0.17 remote-as 301
 neighbor 200.50.0.17 soft-reconfiguration inbound
 neighbor 200.50.0.22 remote-as 2042
 neighbor 200.50.0.22 soft-reconfiguration inbound
```

Видно что R24 предлагает R23 маршруты с Next-Hop до которого у R23 нет маршрутов и он не добавляет их себе в таблицу маршрутизации

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab10/png/RR/R23_rec_from_R24_befor.png)


Теперь добавляем `next-hop-self`

спустя некторое время Next-Hop изменился и маршруты добавились в остновную таблицу маршрутизации

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab10/png/RR/R23_rec_from_R24_after.png)


Так же маршруты добавились на R25 и R26

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab10/png/RR/R25_route_bgp.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab10/png/RR/R26_route_bgp.png)


у R23 три активных соседа

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab10/png/RR/R23_nei.png)


### 3. Настроим офис Москва так, чтобы приоритетным провайдером стал Ламас.

Задачу можно рассматривать в двух контекстах - сделать приоритетным для входящих маршрутов и сделать приоритетным для исходящего трафика от внутренних маршрутизаторов. В нашем случае имеется в виду первый вариант. Для настройки необходимо с помощью атрибута Local Preference указать, что через R15 выходить из AS 1001 предпочтительнее, чем через R14. 

Local Preference:
- Указывает маршрутизаторам внутри автономной системы как предпочтительнее выйти за её пределы (Чтобы выбор действительно был, нужно чтобы один и тот же маршрут пришёл с разных eBGP-соседей и попал на разные пограничные роутеры в AS. Если маршрут приходит только через одного пограничного роутера, то сравнивать Local Preference не с чем, и он просто используется по умолчанию.)
- Local Preference применяется только к входящим в AS маршрутам по BGP и передается только в пределах одной автономной системы.
- Чем выше значение Local Preference, тем предпочтительнее маршрут.

На R15 создаём route-map в которой устанавливается Local Preference 200 для маршрутов, которые мы получаем от соседа eBGP - R21 и пременяем к этому соседу на входящие анонсы. 

```
R15(config)#route-map LOCPREF_AS301
R15(config-route-map)#set local-preference 200
```
```
R15(config)#router bgp 1001
R15(config-router)#neighbor 200.50.0.9 route-map LOCPREF_AS301 in
```

После применения route-map у R14 и R15 и инициирования что бы префиксы пришли занова (отключением интерфейсов на R18) для входящих маршрутов до подсетей AS2042 Local Preference стал 200, приоритетный провайдер Ламас AS301.

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab10/png/Local%20Perf/R14_sh_bgp_loc.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab10/png/Local%20Perf/R15_sh_bgp_loc.png)


### 4. Настроим офис С.-Петербург так, чтобы трафик до любого офиса распределялся по двум линкам одновременно.

(в предыдущей ЛР eBGP не анонсировал подсеть между R18 и R24, сейчас добавляем её в анонс BGP)

До внесения изменений видно, что R24 и R26 анонсируют для R18 одинаковые префиксы c одинаковыми BGP атрибутами и до каждого префикса у R18 есть два маршрута один из которых помечен лучшим и занесён в основную таблицу маршрутизации.

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab10/png/maximum-paths/R18_befor_1.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab10/png/maximum-paths/R18_befor_2.png)

Теперь вносим изменения 

`maximum-paths 2` — разрешает BGP установить до 2 равнозначных маршрутов в RIB

`bgp bestpath as-path multipath-relax` — позволяет использовать несколько путей даже с разными next-hop AS, при прочих равных (AS-PATH одинаковый, но AS разные)

R18:

```
router bgp 2042
 bgp bestpath as-path multipath-relax
 maximum-paths 2
```

Видно, что теперь до удалённого офиса по две записи на один префикс с двумя next-hop'ами.

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab10/png/maximum-paths/R18_after.png)


### 5. Все сети в лабораторной работе должны иметь IP связность.

 Пока мы передаём loopback'и бордеров(можно убрать из анонса при необходимости) и внешние сети пограничных роутеров (доступность пограничных роутеров нужна была в п.5 задания BGP. Основы). 


Добавляем в анонс eBGP сети конечных пользователей.
Со стороны Москвы на R14 и R15:  172.16.1.0/24 и 172.16.7.0/24

Со стороны СПБ на R18:  `network 172.16.8.0 mask 255.255.252.0` 

(**мы можем анонсировать только префиксы, которые содержатся в нашей таблице маршрутизации, т.е. для BGP нужно, чтобы конкретно анонсируемый префикс (**_точный_**) находился в локальной таблице маршрутизации**, а т.к в ЛР по EIGRP анонсировали для R18 **суммарный** префикс до конечных пользователей, то у нас в таблице `172.16.8.0/22`. Его и надо анонсировать.)

R18:

```
router bgp 2042
 bgp router-id 18.18.18.18
 bgp log-neighbor-changes
 bgp bestpath as-path multipath-relax
 network 10.2.0.1 mask 255.255.255.255
 network 172.16.8.0 mask 255.255.252.0
 network 200.50.0.20 mask 255.255.255.252
 network 200.50.0.24 mask 255.255.255.252
 neighbor 200.50.0.21 remote-as 520
 neighbor 200.50.0.21 soft-reconfiguration inbound
 neighbor 200.50.0.25 remote-as 520
 neighbor 200.50.0.25 soft-reconfiguration inbound
 maximum-paths 2
```

R14:

```
router bgp 1001
 bgp router-id 14.14.14.14
 bgp log-neighbor-changes
 network 10.1.0.1 mask 255.255.255.255
 network 172.16.1.0 mask 255.255.255.0
 network 172.16.7.0 mask 255.255.255.0
 network 200.50.0.0 mask 255.255.255.252
 neighbor 10.1.0.2 remote-as 1001
 neighbor 10.1.0.2 update-source Loopback0
 neighbor 10.1.0.2 soft-reconfiguration inbound
 neighbor 200.50.0.1 remote-as 101
 neighbor 200.50.0.1 soft-reconfiguration inbound

```

R15:

```
router bgp 1001
 bgp router-id 15.15.15.15
 bgp log-neighbor-changes
 network 10.1.0.2 mask 255.255.255.255
 network 172.16.1.0 mask 255.255.255.0
 network 172.16.7.0 mask 255.255.255.0
 network 200.50.0.8 mask 255.255.255.252
 neighbor 10.1.0.1 remote-as 1001
 neighbor 10.1.0.1 update-source Loopback0
 neighbor 10.1.0.1 soft-reconfiguration inbound
 neighbor 200.50.0.9 remote-as 301
 neighbor 200.50.0.9 soft-reconfiguration inbound
 neighbor 200.50.0.9 route-map LOCPREF_AS301 in
```

Проверяем связность:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab10/png/%D1%81%D0%B2%D1%8F%D0%B7%D0%BD%D0%BE%D1%81%D1%82%D1%8C/VPC1_ping_SBP.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab10/png/%D1%81%D0%B2%D1%8F%D0%B7%D0%BD%D0%BE%D1%81%D1%82%D1%8C/VPC8_ping_MSK.png)


[Конфигурации](https://github.com/buravtsovpavel/nw-labs/tree/main/lab10/configs) устройств
