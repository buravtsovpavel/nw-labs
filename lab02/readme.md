## Лабораторная работа 
Настройка STP
---
*Примечание*

_Исходя из предоставленых образов IoL в исходное задание из методички внесены корректировки используемых интерфесов._

### Дана топология:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab02/png/topology.png)

---
Таблица адресации

Устройство    | Интерфейс  | IP-адрес    | Маска подсети 
--------      | ----       | --------    | --------------
S1            | VLAN 1     | 192.168.1.1 | 255.255.255.0
S2            | VLAN 1     | 192.168.1.2 | 255.255.255.0
S3            | VLAN 1     | 192.168.1.3 | 255.255.255.0

### Цели:

* Часть 1. Создание сети и настройка основных параметров устройства
* Часть 2. Выбор корневого моста
* Часть 3. Наблюдение за процессом выбора протоколом STP порта, исходя из стоимости портов
* Часть 4. Наблюдение за процессом выбора протоколом STP порта, исходя из приоритета портов
---

### 1. Создаём сеть и производим настройку основных параметров устройств

Отключаем поиск DNS, присваиваем имена устройствам в соответствии с топологией, назначьте class в качестве зашифрованного пароля доступа к привилегированному режиму, назначаем cisco в качестве паролей консоли и VTY,  активируем вход для консоли и VTY каналов, настраиваем logging synchronous для консольного канала, настраиваем баннерное сообщение дня (MOTD) для предупреждения пользователей о запрете несанкционированного доступа, задаём IP-адрес, указанный в таблице адресации для VLAN 1 на всех коммутаторах, копируем текущую конфигурацию в файл загрузочной конфигурации.

Пример настройки для S1

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
S1(config)#
S1(config)#clock timezone YEKT 5 0
S1(config)#interface vlan 1
S1(config-if)#ip address 192.168.1.1 255.255.255.0
S1(config-if)#no shutdown
```
S2 и S3 конфигурируем схожим образом.

Проверяем способность компьютеров обмениваться эхо-запросами. Обмен успешный.

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab02/png/s1ping.png)
![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab02/png/s2ping.png)
![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab02/png/s3ping.png)


### 2. Определение корневого моста

Отключаем все порты на коммутаторах:

```
S1(config)#int range et0/0 - 3
S1(config-if-range)#shutdown
```
на S2 и S3 аналогично.

Настраиваем подключенные порты в качестве транковых:

```
S1(config)#int range et0/0 - 3
S1(config-if-range)#switchport trunk encapsulation dot1q
S1(config-if-range)#switchport mode trunk
S1(config-if-range)#switchport trunk allowed vlan 1
```
на S2 и S3 аналогично.

Включаем порты e0/0 и e0/2:

```
S1(config)#int range et0/0 , et0/2
S1(config-if-range)#no shutdown
```
на S2 и S3 аналогично.

Отображаем данные протокола spanning-tree  командой 
```
show spanning-tree
```
Видно, что рутом выбран коммутатор S1 с наименьшим Bridge ID. (все три коммутатора имеют равные значения приоритета идентификатора моста (32769 = 32768 + 1, где приоритет по умолчанию = 32768, номер сети VLAN = 1) следовательно, коммутатор с самым низким значением MAC-адреса становится корневым мостом.)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab02/png/shstp_1.png)

На текущий момент роли и состояния активных портов на каждом коммутаторе в топологии такие:

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab02/png/port_state_1.png)

Ответы на вопросы:
* Какой коммутатор является корневым мостом? S1 
* Почему этот коммутатор был выбран протоколом spanning-tree в качестве корневого моста? Так как у него наименьший Bridge ID
* Какие порты на коммутаторе являются корневыми портами? на S2 e0/2; на S3 e0/0
* Какие порты на коммутаторе являются назначенными портами? на S1 e0/0, e0/2; на S2 e0/0; 
* Какой порт отображается в качестве альтернативного и в настоящее время заблокирован? e0/2 на S3
* Почему протокол spanning-tree выбрал этот порт в качестве невыделенного (заблокированного) порта? На основании стоимости маршрута он не был выбран корневым и у него был не лучший путь для приёма трафика ведущего к корневому мосту, поэтому он не был выбран назначенным (Designated), а порт который не является корневым или назначенным становится альтернатиным и блокируется.


### 3. Наблюдение за процессом выбора протоколом STP порта, исходя из стоимости портов

На текущий момент заблокирован e0/2 на на коммутаторе с самым высоким идентификатором BID (S3). 

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab02/png/shstp_s3.png)

Уменьшаем стоимость корневого порта этого коммутатора(S3) до 18

```
S3(config)#interface ethernet0/0
S3(config-if)#spanning-tree cost 18
```
Теперь ранее заблокированный порт (S3 e0/2) теперь является назначенным, а заблокирован порт на другом коммутаторе некорневого моста (S2 e0/0)

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab02/png/shstp_s3_2.png)
![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab02/png/shstp_s2.png)


Почему протокол spanning-tree заменяет ранее заблокированный порт на назначенный порт и блокирует порт, который был назначенным портом на другом коммутаторе?
* До изменения стоимость Root Path Cost (до Root Bridge) коммутаторов S2 и S3 была одинаковой и Designated порт был определен на основе меньшего BID в сегменте(BID в сегменте был меньше у S2, соответственно DT был выбран на нём). После того как изменили стоимость порта e0/0 появилась разница в Root Path Cost и DT был выбран на её основании.  

Удаляем изменения стоимости порта
```
S3(config)#interface ethernet0/0
S3(config-if)#no spanning-tree cost 18
```

Убеждаемся, что протокол STP сбросил порт на коммутаторе некорневого моста, вернув исходные настройки порта.

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab02/png/shstp_s3_rollback.png)

### 4. Наблюдение за процессом выбора протоколом STP порта, исходя из приоритета портов

Включаем порты e0/1 и e0/3 на всех коммутаторах

```
S1(config)#int range et0/1 , et0/3
S1(config-if-range)#no shutdown
```
Отображаем данные протокола spanning-tree

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab02/png/shstp_s2_4.png)
![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab02/png/shstp_s3_4.png)

В качестве порта корневого моста на каждом коммутаторе некорневого моста выбраны e0/1 на S2 и e0/0 на S3

![](https://github.com/buravtsovpavel/nw-labs/blob/main/lab02/png/port_state_2.png)

Почему протокол STP выбрал эти порты в качестве портов корневого моста на этих коммутаторах?
* При определении корневых портов если стоимость линков до корневого коммутатора (Root Path Cost) и Bridge ID коммутаторов до корневого коммутатора совпадает (например, как в нашем случае, когда между двумя коммутаторами два и более линка), то тогда корневой порт выбирается на основе меньшего Port ID соседа (lowest neighbor port ID). В нашем случае e0/1 на S1 - сооветственно e0/1 root на S2 и аналогично e0/0 root на S3.

1.	Какое значение протокол STP использует первым после выбора корневого моста, чтобы определить выбор порта?
* Root Path Cost стоимость
2.	Если первое значение на двух портах одинаково, какое следующее значение будет использовать протокол STP при выборе порта?
* Bridge ID отправителя (lowest neighbor bridge ID)
3.	Если оба значения на двух портах равны, каким будет следующее значение, которое использует протокол STP при выборе порта?
* приоритет порта отправителя Port ID (lowest neighbor port ID)

---
(конфиги устройств: [S1](https://github.com/buravtsovpavel/nw-labs/blob/main/lab02/configs/S1.txt), [S2](https://github.com/buravtsovpavel/nw-labs/blob/main/lab02/configs/S2.txt), [S3](https://github.com/buravtsovpavel/nw-labs/blob/main/lab02/configs/S3.txt))
