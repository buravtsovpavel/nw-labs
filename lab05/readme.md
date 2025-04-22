## Лабораторная работа 
PBR
---
### Цель:
Настроить политику маршрутизации в офисе Чокурдах
Распределить трафик между 2 линками


1. Настроить политику маршрутизации для сетей офиса.
2. Распределить трафик между двумя линками с провайдером.
3. Настроить отслеживание линка через технологию IP SLA.(только для IPv4)
4. Настроить для офиса Лабытнанги маршрут по-умолчанию.
5. План работы и изменения зафиксировать в документации.
---

### Решение

1-2. Policy-based routing с использованием route map для распределения трафика между двумя линками с провайдером будет настроен следующим образом: 
- Трафик хостов из VLAN 30 подсеть 172.16.30.0/24 будет направляться через интерфейс e0/3 роутера R25 ip 200.50.0.33
- Трафик хостов из VLAN 31 подсеть 172.16.31.0/24 будет направляться через интерфейс e0/1 роутера R26 ip 200.50.0.29

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab05/png/topology.png)

Сначала пропишем статические маршруты, что бы можно было проверять связанность.

R28
```
ip route 200.50.0.36 255.255.255.252 200.50.0.33 name to_labitnangi_net_2
ip route 200.50.0.36 255.255.255.252 200.50.0.29 name to_labitnangi_net_1
```
R27 Лабытнанги (это п.4 в задачах)

```
ip route 0.0.0.0 0.0.0.0 200.50.0.37
```

R25
```
ip route 172.16.0.0 255.255.0.0 200.50.0.34 name to_hosts_1
ip route 200.50.0.28 255.255.255.252 10.0.20.14 name to_200.50.0.28_net1
```

R26
```
ip route 172.16.0.0 255.255.0.0 200.50.0.30 name to_hosts_1
ip route 200.50.0.36 255.255.255.252 10.0.20.13 name to_labitnangi
```


Далее настраиваем R28. Создаём ACL для определения трафика из VLAN 30 и VLAN 31

```
ip access-list extended ACL-for-VLAN30
 permit ip 172.16.30.0 0.0.0.255 any
 deny   ip any any
ip access-list extended ACL-for-VLAN31
 permit ip 172.16.31.0 0.0.0.255 any
 deny   ip any any
```

Создаём route-map для маршрутизации по политикам

```
route-map PBR-VLAN31 permit 10
 match ip address ACL-for-VLAN31
 set ip next-hop 200.50.0.29
!
route-map PBR-VLAN30 permit 10
 match ip address ACL-for-VLAN30
 set ip next-hop 200.50.0.33
```

Применяем route-map к интерфейсам VLAN

``` 
interface Ethernet0/2.30
 ip policy route-map PBR_VLAN30

interface Ethernet0/2.31
 ip policy route-map PBR_VLAN31
```

Теперь видно, что трафик идёт разными маршрутами

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab05/png/trace2.png)


3. Настроим отслеживание линка через технологию IP SLA.(только для IPv4)

Создаём IP SLA для e0/1 R26 (подключение к R26)

```
ip sla 1
 icmp-echo 200.50.0.29 source-ip 172.16.30.1
 frequency 5
```

Создаём IP SLA для e0/3 R25 (подключение к R25)

```
ip sla 2
 icmp-echo 200.50.0.33 source-ip 172.16.31.1
 frequency 5
```
Включаем IP SLA

```
ip sla schedule 1 life forever start-time now
ip sla schedule 2 life forever start-time now
```

Создаем track'и отслеживания для каждого IP SLA
```
! подключение к R26
track 1 ip sla 1 reachability

! подключение к R25
track 2 ip sla 2 reachability
```

Вносим изменения в ранее созданные route-map с использованием отслеживания
```
route-map PBR-VLAN31 permit 10
 match ip address ACL-for-VLAN31
 set ip next-hop verify-availability 200.50.0.29 10 track 1
 set ip next-hop verify-availability 200.50.0.33 20 track 2
```

```
route-map PBR-VLAN30 permit 10
 match ip address ACL-for-VLAN30
 set ip next-hop verify-availability 200.50.0.33 10 track 2
 set ip next-hop verify-availability 200.50.0.29 20 track 1
```

----------
### Примечание 

Для тестирования настроим отслеживание состояния интерфейса e0/3 на R25, что бы когда мы его выключим трафик до сетей с конечными хостами пошёл через e0/2 на R26. Иначе, если не будет включено отслеживание останется маршрут через e0/1 R28 и связности не будет.

Создаём IP SLA для проверки доступности IP-адреса 200.50.0.33 (проверяем с lo0 10.3.0.2)

```
ip sla 1
icmp-echo 200.50.0.33 source-ip 10.3.0.2
frequency 5
```
Активируем IP SLA

```
ip sla schedule 1 life forever start-time now
```

Создаём track, который основывается на статусе IP SLA
```
track 1 ip sla 1 reachability
```
Прописываем статические маршруты с проверкой track

```
! Основной маршрут - активен при достижимости IP SLA
ip route 172.16.0.0 255.255.0.0 200.50.0.34 name to_hosts_1 track 1

! Запасной маршрут — активируется, если IP SLA недостижим
ip route 172.16.0.0 255.255.0.0 10.0.20.14 20 name to_hosts_2

! маршрут до сети 200.50.0.28/30
ip route 200.50.0.28 255.255.255.252 10.0.20.14 name to_200.50.0.28_net1
```
---

### Проверка

1. Выключаем интерфейс e0/1 на R26 

Цель ip sla 1 становится не доступна, track 1 переходит в состояние Down, оба хоста начинают ходить через e0/3 на R25 (ip 200.50.0.33)
```
R28(config-track)#
*Apr 22 13:22:10.102: %TRACK-6-STATE: 1 ip sla 1 reachability Up -> Down
R28(config-track)#
```
![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab05/png/trace_sla_1.png)

2. Выключаем интерфейс e0/3 на R25 

Цель ip sla 2 становится не доступна, track 2 переходит в состояние Down. Так же и на R25 соответствующая цель ip sla становится не доступна, track 1 переходит в состояние Down и маршрут до 172.16.0.0 меняется.

Оба хоста начинают ходить через e0/1 на R26 (ip 200.50.0.29)

```
R28(config-track)#
*Apr 22 13:35:30.557: %TRACK-6-STATE: 2 ip sla 2 reachability Up -> Down
R28(config-track)#
```

```
R25(config-if)#
*Apr 22 13:35:31.772: %TRACK-6-STATE: 1 ip sla 1 reachability Up -> Down
R25(config-if)#
```
![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab05/png/trace_sla_2.png)

[Конфиги](https://github.com/buravtsovpavel/nw-labs/tree/main/lab05/configs) устройств.
