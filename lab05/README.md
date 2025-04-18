Домашнее задание

VxLAN. L2 VNI
Цель:

Настроить Overlay на основе VxLAN EVPN для L2 связанности между клиентами

Описание/Пошаговая инструкция выполнения домашнего задания:

В этой самостоятельной работе мы ожидаем, что вы самостоятельно:

    Настроите BGP peering между Leaf и Spine в AF l2vpn evpn
    Настроите связанность между клиентами в первой зоне и убедитесь в её наличии
    Зафиксируете в документации - план работы, адресное пространство, схему сети, конфигурацию устройств

Документация оформлена на github в файле Readme.md(markdown). Каждая лабораторная работа находится в своей директории. 

### 2. Адресный план и правила именования коммутаторов:

Протокол eBGP настривается для работы с IPv6 адресацией.
      
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
| Коммутатор  | Hostname  |  IP Loopback 0 | IP Loopback 1 | BGP AS Number |
| :------------ |:---------------:| -----:| ---------------:| -------------:|
| Spine 1      | swsp-dc01-01 | 10.1.0.1 | fd:0:0:1::1/64 | 65000 |
| Spine 2      | swsp-dc01-02 |   10.1.0.2 | fd:0:0:2:1/64 | 65000 |
| Leaf 1 | swle-dc01-01 |    10.1.0.3 | fd:0:0:3::1/64 | 65001 |
| Leaf 2 | swle-dc01-02 |    10.1.0.4 | fd:0:0:4::1/64 | 65002 |
| Leaf 3 | swle-dc01-03 |    10.1.0.5 | fd:0:0:5::1/64 | 65003 |


  ![Схема](net05.png)



### 3. План выполнения работ
#### 3.1 Подготовительные работы
Выполнена коммутация согласно п.2, настроены линковые интерфейсы и интерфейсы Loopback 1 с ipv6 адресами согласно таблицы №1
Настроен eBGP в underlay, настроена сессия для AF EVPN 

### 3.1.1 Включить ipv6 форвардинг:


### 3.1.2 Включить работу расширений BGP на Arista:

```

### 3.1.2 Настроить линковые интерфейсы для ipv6 + bfd
Достаточно настройки link-local адресации для обеспечения соседской связности по BGP в пределах фабрики.
```
interface Ethernet2.10
   description - Uplink to Spine-2
   encapsulation dot1q vlan 10
   ipv6 enable
   bfd interval 300 min_rx 300 multiplier 3
```
                        

#### 3.2 Настроить процесс маршрутизации BGP (на примере Leaf-3)
- Согласно таблицы №1 65003;
- Включаем ECMP, указав maximum-path 10;
- Настраиваем редистрибуцию connected маршрутов, для адресов Loopback с применением route-map + prefix-list;
- Настраиваем соседские отношения для Spine-1 и Spine-2;
- Включаем функционал BFD для каждого соседа;
- Включаем фильтр входящих префиксов по AS-Path;
- Оптимизировать таймеры BGP (для всех устройств):
    - Route Advertisement Timer 'advertisement-interval 0'
    - Keepalive/Hold: 20/60
    - graceful-restart 'restart-time 120'

  



##### 3.2.2 Фрагмент конфигурации процесса BGP на устройстве Leaf-3:
```
router bgp 65003
   router-id 10.1.0.5
   no bgp default ipv4-unicast
   bgp default ipv6-unicast
   graceful-restart restart-time 120
   maximum-paths 10
   neighbor spine-1 peer group
   neighbor spine-1 bfd
   neighbor spine-1 timers 20 60
   neighbor spine-1 graceful-restart
   neighbor spine-1 route-map in-as-path in
   neighbor spine-2 peer group
   neighbor spine-2 bfd
   neighbor spine-2 timers 20 60
   neighbor spine-2 graceful-restart
   neighbor spine-2 route-map in-as-path in
   neighbor fe80::5200:ff:fe03:3766%Et2.10 peer group spine-2
   neighbor fe80::5200:ff:fe03:3766%Et2.10 remote-as 65000
   neighbor fe80::5200:ff:fed5:5dc0%Et1.10 peer group spine-1
   neighbor fe80::5200:ff:fed5:5dc0%Et1.10 remote-as 65000
   !
   address-family ipv6
      neighbor spine-1 activate
      neighbor spine-2 activate
      network fd::/61
      redistribute connected route-map rc6-map
```

##### 3.2.2 Фрагмент конфигурации процесса BGP на устройстве Spine-1:
```
router bgp 65000
   router-id 10.1.0.1
   no bgp default ipv4-unicast
   bgp default ipv6-unicast
   graceful-restart restart-time 120
   maximum-paths 10
   neighbor leaf-1 peer group
   neighbor leaf-1 bfd
   neighbor leaf-1 timers 20 60
   neighbor leaf-1 graceful-restart
   neighbor leaf-1 route-map in-as-path in
   neighbor leaf-2 peer group
   neighbor leaf-2 bfd
   neighbor leaf-2 timers 20 60
   neighbor leaf-2 graceful-restart
   neighbor leaf-2 route-map in-as-path in
   neighbor leaf-3 peer group
   neighbor leaf-3 bfd
   neighbor leaf-3 timers 20 60
   neighbor leaf-3 graceful-restart
   neighbor leaf-3 route-map in-as-path in
   neighbor fe80::5200:ff:fe15:f4e8%Et3.10 peer group leaf-3
   neighbor fe80::5200:ff:fe15:f4e8%Et3.10 remote-as 65003
   neighbor fe80::5200:ff:fecb:38c2%Et2.10 peer group leaf-2
   neighbor fe80::5200:ff:fecb:38c2%Et2.10 remote-as 65002
   neighbor fe80::5200:ff:fed7:ee0b%Et1.10 peer group leaf-1
   neighbor fe80::5200:ff:fed7:ee0b%Et1.10 remote-as 65001
   !
   address-family ipv6
      neighbor leaf-1 activate
      neighbor leaf-2 activate
      neighbor leaf-3 activate
      redistribute connected route-map rc6-map
!
```

### 3.4 Выполнить контроль и проверки

- Убедиться в том, что соседские отношения подняты (проверку выполняем на leaf-3):
```
swle-dc01-03#sh ipv6 bgp summary
BGP summary information for VRF default
Router identifier 10.1.0.5, local AS number 65003
Neighbor Status Codes: m - Under maintenance
  Neighbor                       V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  fe80::5200:ff:fe03:3766%Et2.10 4 65000            104       110    0    0 01:09:46 Estab   3      3
  fe80::5200:ff:fed5:5dc0%Et1.10 4 65000            127       129    0    0 01:09:45 Estab   3      3
```
- Убедиться в наличии маршрутов адресов Loopback в апдейтах от Spine, убедиться в наличии ECMP префиксов fd:0:0:3::/64 и fd:0:0:4::/64:
```
  swle-dc01-03#sh ipv6 bgp
BGP routing table information for VRF default
Router identifier 10.1.0.5, local AS number 65003
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      fd:0:0:1::/64          fe80::5200:ff:fed5:5dc0%Et1.10 0       -          100     0       65000 i
 * >      fd:0:0:2::/64          fe80::5200:ff:fe03:3766%Et2.10 0       -          100     0       65000 i
 * >Ec    fd:0:0:3::/64          fe80::5200:ff:fed5:5dc0%Et1.10 0       -          100     0       65000 65001 i
 *  ec    fd:0:0:3::/64          fe80::5200:ff:fe03:3766%Et2.10 0       -          100     0       65000 65001 i
 * >Ec    fd:0:0:4::/64          fe80::5200:ff:fed5:5dc0%Et1.10 0       -          100     0       65000 65002 i
 *  ec    fd:0:0:4::/64          fe80::5200:ff:fe03:3766%Et2.10 0       -          100     0       65000 65002 i
 * >      fd:0:0:5::/64          -                     -       -          -       0       i
```


- Убедиться в наличии маршрутов адресов Loopback 1 коммутаторов leaf и работе ECMP для префиксов fd:0:0:3::/64 и fd:0:0:4::/64:
```
swle-dc01-03#sh ipv6 route

VRF: default
Displaying 5 of 9 IPv6 routing table entries
Codes: C - connected, S - static, K - kernel, O3 - OSPFv3,
       B - Other BGP Routes, A B - BGP Aggregate, R - RIP,
       I L1 - IS-IS level 1, I L2 - IS-IS level 2, DH - DHCP,
       NG - Nexthop Group Static Route, M - Martian,
       DP - Dynamic Policy Route, L - VRF Leaked,
       RC - Route Cache Route

 B E      fd:0:0:1::/64 [200/0]
           via fe80::5200:ff:fed5:5dc0, Ethernet1.10
 B E      fd:0:0:2::/64 [200/0]
           via fe80::5200:ff:fe03:3766, Ethernet2.10
 B E      fd:0:0:3::/64 [200/0]
           via fe80::5200:ff:fed5:5dc0, Ethernet1.10
           via fe80::5200:ff:fe03:3766, Ethernet2.10
 B E      fd:0:0:4::/64 [200/0]
           via fe80::5200:ff:fed5:5dc0, Ethernet1.10
           via fe80::5200:ff:fe03:3766, Ethernet2.10
 C        fd:0:0:5::/64 [0/0]
           via Loopback1, directly connected
```

##### Наличие маршрута fd:0:0:3::/64 и fd:0:0:4::/64 (loopback 1 leaf-1 и leaf-2) через оба spine, свидетельствуют о корректной работе ECMP.

Дополнительно вывод routing information base для ipv6 префикса fd:0:0:3::/64, свидетельствует о том что маршрут попадает в таблицу маршрутизации от BGP и работает ECMP:

```
swle-dc01-03#sh rib route ipv6 fd:0:0:3::/64
VRF: default
Codes: C - Connected, S - Static, P - Route Input, G - Gribi
       B - BGP, O - Ospf, O3 - Ospf3, I - Isis, R - Rip, VL - VRF Leak
       > - Best Route, * - Unresolved Next hop
       EM - Exact match of the SR-TE Policy
       NM - Null endpoint match of the SR-TE Policy
       AM - Any endpoint match of the SR-TE Policy
       L - Part of a recursive route resolution loop
       A - Next hop not resolved in ARP/ND
       NF - Not in FEC
>B    fd:0:0:3::/64 [200 pref/0 MED] updated 00:40:36 ago
         via fe80::5200:ff:fe03:3766, Ethernet2.10
         via fe80::5200:ff:fed5:5dc0, Ethernet1.10
```

- Выполняем проверку связности между адресами Loopback 1 Leaf коммутаторов:
```
swle-dc01-03#ping ipv4 fd:0:0:3::1
PING fd:0:0:3::1(fd:0:0:3::1) 52 data bytes
60 bytes from fd:0:0:3::1: icmp_seq=1 ttl=63 time=24.0 ms
60 bytes from fd:0:0:3::1: icmp_seq=2 ttl=63 time=17.1 ms
60 bytes from fd:0:0:3::1: icmp_seq=3 ttl=63 time=20.1 ms
```

### 4 Конфигурации устройств
- Spine коммутаторы:
  - [swsp-dc1-1](configs/swsp-dc01-01.conf)
  - [swsp-dc1-2](configs/swsp-dc01-02.conf)
- Leaf коммутаторы:
  - [swle-dc1-1](configs/swle-dc01-01.conf)
  - [swle-dc1-2](configs/swle-dc01-02.conf)
  - [swle-dc1-3](configs/swle-dc01-03.conf)


