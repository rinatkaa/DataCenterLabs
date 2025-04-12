Домашнее задание

Underlay. IS-IS
Цель:

Настроить IS-IS для Underlay сети с использованием ipv6

Описание/Пошаговая инструкция выполнения домашнего задания:

В этой самостоятельной работе мы ожидаем, что вы самостоятельно:

    Настроите ISIS в Underlay сети, для IP связанности между всеми сетевыми устройствами.
    Зафиксируете в документации - план работы, адресное пространство, схему сети, конфигурацию устройств
    Убедитесь в наличии IP связанности между устройствами в ISIS домене


### 2. Адресный план и правила именования коммутаторов:

На устройствах одновременно настраивается оба стека IPv4, IPv6 с отдельными процессами маршрутизации.
Протокол ISIS настривается для IPv6
      
- Общий план адресов ipv4: 10.0.0.0/8;
- Адреса для Loopback интерфейсов ipv4: 10.00[DC num].0.0/23, 512 устройств на 1 DC;
- #### Адреса для Loopback интерфейсов ipv6: fd::/61 в соответствии с таблицей №1;
- Линковые адреса ipv4: 10.10[DC_num].16.0/20, для линковых сетей использовать /31, младший адрес на стороне Spine;
- #### Линковые адреса ipv6: link-local
- Правила именования коммутаторов:
   - Spine Hostname: swsp-dc0[DC_num]-num
   - Leaf Hostname: swle-dc0[DC_num]-num
- Линковые интерфейсы для стека ipv4: основной интерфейс Eth [0..n] //сохраняю для будущих лаб, не используется в данной лабе
- #### Линковые интерфейсы для стека ipv6: подинтерфейс Eth [0..n].10, с тэгом 802.1q 10
  
#### Таблица №1 Имена хостов и адреса Loopback
| Коммутатор  | Hostname  |  IP Loopback 0 | IP Loopback 1 | System ID |
| :------------ |:---------------:| -----:| ---------------:| -------------:|
| Spine 1      | swsp-dc01-01 | 10.1.0.1 | fd:0:0:1::1/64 | 0000.0000.0001 |
| Spine 2      | swsp-dc01-02 |   10.1.0.2 | fd:0:0:2:1/64 |0000.0000.0002 |
| Leaf 1 | swle-dc01-01 |    10.1.0.3 | fd:0:0:3::1/64 | 0000.0001.0001 |
| Leaf 2 | swle-dc01-02 |    10.1.0.4 | fd:0:0:4::1/64 | 0000.0001.0002 |
| Leaf 3 | swle-dc01-03 |    10.1.0.5 | fd:0:0:5::1/64 | 0000.0001.0003 |


  ![Схема](net03.png)



### 3. План выполнения работ
#### 3.1 Подготовительные работы
Выполнена коммутация согласно п.2, настроены линковые интерфейсы и интерфейсы Loopback 1

### 3.1.1 Включить ipv6 форвардинг:
```
ipv6 unicast-routing
```
### 3.1.2 Настроить линковые интерфейсы для ipv6
Достаточно настройки link-local адресации для обеспечения форвардинга пакетов в пределах фабрики.
```
interface Ethernet2.10
   description - Uplink to Spine-2
   encapsulation dot1q vlan 10
   ipv6 enable
```
                        

#### 3.2 Настроить процесс маршрутизации ISIS
- Используется только Area 49.0001;
- Используется только один тип отношений между устройствами level-2 - этого достаточно для работы фабрики и редистрибьюции коннектед маршрутов;
- ECMP по умолчанию (128 одинаковых маршрутов для Arista);
- Настроена редистрибуция connected маршрутов, для адресов Loopback с применением route-map + prefix-list;
- Включить функционал BFD для интерфейсов на которых работает ISIS;

  
##### 3.2.1 Фрагмент конфигурации route-map + prefix-list:
```
ip prefix-list redistribute-connected-pl
   seq 10 permit 10.1.0.0/24 ge 32
!
ipv6 prefix-list rc6-pl
   seq 10 permit fd::/61 ge 64
```

##### 3.2.2 Фрагмент конфигурации процесса ISIS:
```
router isis 1
   net 49.0001.0000.0000.0003.00
   is-type level-2
   redistribute connected route-map rc6-map
   !
   address-family ipv6 unicast
```

#### 3.3 Настроить атрибуты протокола ISIS на интерфейсах
- включить на интерфейсе протокол ISIS;
- Включить аутентификацию сообщений с вычислением md5 хэша;
- Ключ аутентификации 'cisco' без кавычек;

##### 3.3.1 Фрагмент конфигурации интерфейсов:
```
interface Ethernet1.10
   description - Uplink to Spine-1
   encapsulation dot1q vlan 10
   ipv6 enable
   isis enable 1
   isis authentication mode md5
   isis authentication key 7 KHWA0XKKZeJG8f+/WzG88A==
```

### 3.4 Выполнить контроль и проверки


Проверка настройки ipv6 (Leaf 3): 

swle-dc01-03#sh ipv6 int br
Interface  Status    MTU   IPv6 Address                 Addr State  Addr Source
---------- ------- ------ ---------------------------- ------------ -----------
Et1.10     up       1500   fe80::5200:ff:fe15:f4e8/64   up          link local
Et2.10     up       1500   fe80::5200:ff:fe15:f4e8/64   up          link local
Lo1        up      65535   fe80::ff:fe00:0/64           up          link local
                           fd:0:0:5::1/64               up          config

Проверка связности с соседом по ipv6 по link-local:
```
swle-dc01-03#show ipv6 neighbors ethernet2.10
IPv6 Address                                  Age Hardware Addr    State Interface
fe80::5200:ff:fe03:3766                   0:00:30 5000.0003.3766   REACH Et2.10
```

```
swle-dc01-03#ping ipv6 fe80::5200:ff:fe03:3766 interface ethernet 2.10 size 1500
PING fe80::5200:ff:fe03:3766(fe80::5200:ff:fe03:3766) from fe80::5200:ff:fe15:f4e8%et2.10 et2.10: 1452 data bytes
1460 bytes from fe80::5200:ff:fe03:3766%et2.10: icmp_seq=1 ttl=64 time=14.1 ms
1460 bytes from fe80::5200:ff:fe03:3766%et2.10: icmp_seq=2 ttl=64 time=19.3 ms
```
   

- Убедиться в том, что соседские отношения подняты (проверку выполняем на spine):
```
swsp-dc1-02#sh isis neighbor
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.1.0.3        1        default  0   FULL                   00:00:01    10.101.0.3      Ethernet1
10.1.0.4        1        default  0   FULL                   00:00:01    10.101.0.7      Ethernet2
10.1.0.5        1        default  0   FULL                   00:00:01    10.101.0.11     Ethernet3
```

- Убедиться в наличии маршрутов адресов Loopback 1 коммутаторов и работе ECMP:

##### Наличие маршрута 10.1.0.1/32 (Spine-1 loopback 0) через каждый из leaf свидетельствуют о корректной работе ECMP.
```
swsp-dc1-02#sh ip route ospf

VRF: default
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

 O E2     10.1.0.1/32 [110/1] via 10.101.0.3, Ethernet1
                              via 10.101.0.7, Ethernet2
                              via 10.101.0.11, Ethernet3
 O E2     10.1.0.3/32 [110/1] via 10.101.0.3, Ethernet1
 O E2     10.1.0.4/32 [110/1] via 10.101.0.7, Ethernet2
 O E2     10.1.0.5/32 [110/1] via 10.101.0.11, Ethernet3
 O        10.101.0.0/31 [110/20] via 10.101.0.3, Ethernet1
 O        10.101.0.4/31 [110/20] via 10.101.0.7, Ethernet2
 O        10.101.0.8/31 [110/20] via 10.101.0.11, Ethernet3
```
- Аналогично убеждаемся в корректности настройки на стороне leaf:

```
swle-dc01-01#sh ip route ospf

VRF: default
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

 O E2     10.1.0.1/32 [110/1] via 10.101.0.1, Ethernet1
 O E2     10.1.0.2/32 [110/1] via 10.101.0.2, Ethernet2
 O E2     10.1.0.4/32 [110/1] via 10.101.0.1, Ethernet1
                              via 10.101.0.2, Ethernet2
 O E2     10.1.0.5/32 [110/1] via 10.101.0.1, Ethernet1
                              via 10.101.0.2, Ethernet2
 O        10.101.0.4/31 [110/20] via 10.101.0.1, Ethernet1
 O        10.101.0.6/31 [110/20] via 10.101.0.2, Ethernet2
 O        10.101.0.8/31 [110/20] via 10.101.0.1, Ethernet1
 O        10.101.0.10/31 [110/20] via 10.101.0.2, Ethernet2
```

- Обращаем внимание на наличие в таблице маршрутизации более 1 маршрута для Loopback интерфейсов двух других Leaf коммутаторов:
    - 10.1.0.4/32
    - 10.1.0.5/32

- Выполняем проверку связности между адресами Loopback Leaf коммутаторов:
```
swle-dc01-01#ping 10.1.0.4 size 1500 df-bit source loopback 0
PING 10.1.0.4 (10.1.0.4) from 10.1.0.3 : 1472(1500) bytes of data.
1480 bytes from 10.1.0.4: icmp_seq=1 ttl=63 time=22.9 ms
1480 bytes from 10.1.0.4: icmp_seq=2 ttl=63 time=31.7 ms
1480 bytes from 10.1.0.4: icmp_seq=3 ttl=63 time=34.5 ms
```

### 4 Конфигурации устройств
- Spine коммутаторы:
  - [swsp-dc1-1](configs/swsp-dc1-01-config.txt)
  - [swsp-dc1-2](configs/swsp-dc1-02-config.txt)
- Leaf коммутаторы:
  - [swle-dc1-1](configs/swle-dc1-01-config.txt)
  - [swle-dc1-2](configs/swle-dc1-02-config.txt)
  - [swle-dc1-3](configs/swle-dc1-03-config.txt)
