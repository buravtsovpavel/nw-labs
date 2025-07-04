## Лабораторная работа 

BGP. Фильтрация

---
### Цель:

- Настроить фильтрацию для офисе Москва
- Настроить фильтрацию для офисе С.-Петербург

Описание/Пошаговая инструкция выполнения домашнего задания:

- Настроить фильтрацию в офисе Москва так, чтобы не появилось транзитного трафика(As-path).
- Настроить фильтрацию в офисе С.-Петербург так, чтобы не появилось транзитного трафика(Prefix-list).
- Настроить провайдера Киторн так, чтобы в офис Москва отдавался только маршрут по умолчанию.
- Настроить провайдера Ламас так, чтобы в офис Москва отдавался только маршрут по умолчанию и префикс офиса С.-Петербург.
- Все сети в лабораторной работе должны иметь IP связность.
- План работы и изменения зафиксированы в документации.
---

## Решение:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab11/png/topology.png)


### 1. Настроим фильтрацию в офисе Москва так, чтобы не появилось транзитного трафика(As-path).

Транзитный трафик — это любой трафик, идущий через нашу AS между двумя другими AS, независимо от того, являются ли они нашими соседями или нет.

Для корпоративных сетей этого следует избегать, т.к.:

- мы становимся бесплатным транспортом для других.
- можно получить гигантскую нагрузку на каналы.
- нет контроля над тем, кто и как передаёт трафик через нашу AS.

Для предотвращения транзита необходимо не допустить, чтобы маршруты, полученные по eBGP, были переданы дальше другому eBGP-соседу (через iBGP). 

Можно сделать двумя способами - либо не принимать по iBGP от соседа маршруты пришедшие по eBGP из AS на которую тот смотрит, либо не отдавать по iBGP соседу маршруты пришедшие по eBGP из AS на которую смотрит роутер с которого отдаём.

Настроим по второму варианту - будем фильтровать маршруты по AS_PATH при анонсе в iBGP. На R15 — фильтрация маршрутов от AS301 Ламас (пришедших от R21 по eBGP), что бы они не уходили на R14, а на R14 запрет отправки маршрутов от AS101 Киторн (полученные от R22 по eBGP), чтобы они не уходили на R15.

Для настройки используем механизм **AS-Path ACL**, который фильтрует маршруты на основе BGP-поля AS_PATH

<details>
  <summary>AS-Path ACL</summary>
  
  ![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab11/png/as-path-acl.png)
</details>


Настройка R14:

Составляем AS-Path ACL, который отфильтрует маршруты пришедшие от AS 101:

`ip as-path access-list 1 permit ^101`

(У AS-Path Access-List в Cisco нет явной политики по умолчанию, как у обычных ACL типа deny any)

Составляем route-map сопоставляющую условие нашего AS-Path ACL и отбрасывающего маршрут, если он попадает под это правило и разрешающую всё что не попало под правило:

```
route-map BLOCK-TRANSIT-AS101-OUT deny 10
 match as-path 1
!
route-map BLOCK-TRANSIT-AS101-OUT permit 20
```

И применяем эту route-map на выход для соседа R15:

```
router bgp 1001
  neighbor 10.1.0.2 route-map BLOCK-TRANSIT-AS101-OUT out
```

Теперь видно, что в adj-RIB-out на R14 (лучшие маршруты после применения политик к соседу на out) для R15 нет маршрутов пришедших от AS 101:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab11/png/R14_adv_after.png)


на R15 аналогично фильтруем и отбрасываем маршруты пришедшие из AS 301 и не отправляем их R14:

`ip as-path access-list 1 permit ^301`

```
route-map BLOCK-TRANSIT-AS301-OUT deny 10
 match as-path 1
route-map BLOCK-TRANSIT-AS301-OUT permit 20
```
```
router bgp 1001
  neighbor 10.1.0.1 route-map BLOCK-TRANSIT-AS301-OUT out
```

Аналогично видно, что в adj-RIB-out на R15 (лучшие маршруты после применения политик к соседу на out) для R14 нет маршрутов пришедших от AS 301:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab11/png/R15_adv_after.png)




### 2. Настроить фильтрацию в офисе С.-Петербург так, чтобы не появилось транзитного трафика(Prefix-list).

Анонсы для R24 и R26 до применения:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab11/png/2/R18_adv_befor.png)


В данном случае аналогично необходимо не допустить, чтобы маршруты, полученные по eBGP, были переданы дальше другому eBGP-соседу. Для этого разрешим передавать своим eBGP соседям только свои локальные маршруты.  (**Пока мы передаём сети конечных пользователей(п.5 lab10), loopback'и бордеров и внешние сети пограничных роутеров (доступность пограничных роутеров нужна была в п.5 задания BGP. Основы). Если что loopback'и можно будеть удалить из анонсов.**)

Создаём prefix-list`ы, разрешающие только необходимые (локальные) маршруты:
```
ip prefix-list ONLY-LOCAL seq 5 permit 10.2.0.1/32
ip prefix-list ONLY-LOCAL seq 10 permit 172.16.8.0/22
ip prefix-list ONLY-LOCAL seq 15 permit 200.50.0.20/30
ip prefix-list ONLY-LOCAL seq 20 permit 200.50.0.24/30
```

Создаём route-map в которой будем матчить маршруты из нашего prefix-list и разрешать соответствующие:

```
route-map EXPORT-ONLY-LOCAL permit 10
 match ip address prefix-list ONLY-LOCAL

route-map EXPORT-ONLY-LOCAL deny 20
```
Применяем её на исходящих eBGP-сессиях с R24 и R26:

```
router bgp 2042
 neighbor 200.50.0.21 route-map EXPORT-ONLY-LOCAL out
 neighbor 200.50.0.25 route-map EXPORT-ONLY-LOCAL out
```

Анонсы для R24 и R26 после применения:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab11/png/2/R18_adv_after.png)



### 3. Настроим провайдера Киторн так, чтобы в офис Москва отдавался только маршрут по умолчанию.

Сейчас на R14 и R15 маршруты по умолчанию настроены статикой на R22 и R21, поэтому удаляем их.
```
no ip route 0.0.0.0 0.0.0.0 200.50.0.1
no ip route 0.0.0.0 0.0.0.0 200.50.0.9
```

Маршрут по умолчанию (даже если его нет в таблице маршрутизации) анонсируется командой: ```neighbor 200.50.0.2 default-originate```
(работает и в eBGP и в iBGP)

Создаём prefix-list, разрешающий только дефолт
```
ip prefix-list DEFAULT seq 5 permit 0.0.0.0/0
```

Создаём route-map в которой будем матчить адреса из нашего prefix-list и разрешать соответствующие
(но можно и без route-map в данном случае, а просто накинуть prefix-list на R14 на out - так сделал в п.4)

```
route-map ONLY-DEFAULT permit 10
 match ip address prefix-list DEFAULT
```

Применяем эту route-map на исходящей eBGP-сессии с R14:

```
router bgp 101
  neighbor 200.50.0.2 route-map ONLY-DEFAULT out
```

Теперь R14 получает от R22 только маршрут по умолчанию и заносит его себе в таблицу маршрутизации:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab11/png/3/R14_rec_after.png)


((Команда show ip bgp neighbors ... advertised-routes не показывает "искусственные" маршруты, а только те, которые реально присутствуют в локальной BGP таблице (show ip bgp) либо были объявлены через network, aggregate-address, redistribute либо пришли от соседей, а маршрут, идущий через  default-originate, не считается локальным BGP-маршрутом — он "вшивается" в UPDATE пакет исключительно для этого соседа. ))

---------------

### 4. Настроим провайдера Ламас так, чтобы в офис Москва отдавался только маршрут по умолчанию и префикс офиса С.-Петербург.

Будет отдаваться только дефолт и префикс конечных пользователей 

На R21

Делаем prefix-list и применяем его на исходящей eBGP-сессии с R15:

```
ip prefix-list SPB-AND-DEFAULT seq 5 permit 0.0.0.0/0
ip prefix-list SPB-AND-DEFAULT seq 15 permit 172.16.8.0/22
```

```
router bgp 301
 neighbor 200.50.0.10 route-map TO-MOSCOW out
 neighbor 200.50.0.10 default-originate
```

R15 от R21 Ламас получает только дефолт и префикс конечных пользователей СПБ

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab11/png/4/R15_adv.png)


### 5. Проверка связности подсетей конечных пользователей.

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab11/png/5/VPC.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab11/png/5/VPC1.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab11/png/5/VPC7.png)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab11/png/5/VPC8.png)


[Конфигурации](https://github.com/buravtsovpavel/nw-labs/tree/main/lab11/config) устройств
