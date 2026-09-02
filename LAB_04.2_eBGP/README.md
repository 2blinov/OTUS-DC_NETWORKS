# Лабораторная работа №3 "Построение Underlay сети (eBGP)"

## Задание:
1. [Подготовка стенда](#1-подготовка-стенда)
2. [Разработка адресного плана](#2-разработка-адресного-плана)
3. [Настройка Underlay на базе eBGP](#3-настройка-underlay-на-базе-ebgp)
4. [Проверка связности](#4-проверка-связности)
5. [Особенности](#5-особенности)

## 1. Подготовка стенда
В качестве платформы для организации стенда был выбран Containerlab, развернутый на WSL, с использованием образа Arista cEOS.
Получившийся стенд выглядит следующим образом ([Топология для Containetlab](containerlab/lab04_2.yaml)):
<img width="987" height="744" alt="image" src="https://github.com/user-attachments/assets/16aaed8a-e628-4351-938a-7b36b96b88be" />

## 2. Разработка адресного плана
Для адресного плана предлагаем использовать приватную сеть 10.0.0.0/8. При этом второй октет мы будем использовать как индекс ЦОД, для которого предназначена адресация. Для Lo0 предлагаем зарезервировать подсеть /24 (сможем адресовать 256 устройств). Для транспортных подсетей /31 предлагаю зарезервировать подсеть /23 (запас вплоть до фабрики 8 Spine / 32 Leaf). Для адресации сервисов зарезервируем подсеть /21. Итого, общая адресация каждого ЦОД будет суммироваться до /20 (с учетом зарезервированных адресов для возможного расширения).

| Устройство | Подсеть     |
| ---------- | ----------- |
| Loopback0  | 10.x.0.0/24 |
| Reserved   | 10.x.1.0/24 |
| Transport  | 10.x.2.0/23 |
| Reserved   | 10.x.4.0/22 |
| Service    | 10.x.8.0/21 |

## 3. Настройка Underlay на базе eBGP
В качестве лабораторной работы настроим стенд согласно адресному плану (пусть это будет DC1)

Loopback0
| Device |    Loopback0 |
| ------ | -----------: |
| SPINE1 |  10.1.0.1/32 |
| SPINE2 |  10.1.0.2/32 |
| LEAF1  | 10.1.0.3/32 |
| LEAF2  | 10.1.0.4/32 |
| LEAF3  | 10.1.0.5/32 |

Транспортные подсети
| Link           | Network      |
| -------------- | ------------ |
| SPINE1 — LEAF1 | 10.1.2.0/31  |
| SPINE1 — LEAF2 | 10.1.2.2/31  |
| SPINE1 — LEAF3 | 10.1.2.4/31  |
| SPINE2 — LEAF1 | 10.1.2.6/31  |
| SPINE2 — LEAF2 | 10.1.2.8/31  |
| SPINE2 — LEAF3 | 10.1.2.10/31 |

Пояснения касательно настройки
<details>
<summary>Контекст: Процесс BGP/SPINE</summary>

```eos
route-map RM_REDISTRIBUTE-Lo0 permit 10            # route-map для редистрибьюции
   match interface Loopback0                       # выбираем только интерфейс Looback 0
   set origin igp                                  # устанавливаем origin в igp
   set community 65000:1                           # устанавливаем community
!
peer-filter LEAFS-AS-FILTER                        # peer-filter для фильтрации соседств с LEAF
   10 match as-range 65001-65003 result accept     # принимаем только BGP-соседей из AS 65001-65003

router bgp 65000                                                               # Процесс BGP в AS 65001
   router-id 10.1.0.1                                                          # Задаем Router ID
   maximum-paths 4                                                             # количество маршрутов для ECMP
   bgp listen range 10.1.2.0/23 peer-group LEAFS peer-filter LEAFS-AS-FILTER   # Принимаем соседей с адресами из 10.1.2.0/23 и AS 65001-65003
   neighbor LEAFS peer group                                                   # peer-group LEAFS
   neighbor LEAFS bfd                                                          # Включаем BFD
   neighbor LEAFS timers 3 9                                                   # Настраиваем таймеры протокола
   neighbor LEAFS send-community standard extended                             # Настраиваем отправку community
   !
   address-family ipv4
      neighbor LEAFS activate                                                  # активируем соседства в секции IPv4 unicast
      redistribute connected route-map RM_REDISTRIBUTE-Lo0                     # редистрибьюцируем в BGP Loopback0
```
</details>

<details>
<summary>Контекст: Процесс BGP/LEAF</summary>

```eos
route-map RM_REDISTRIBUTE-Lo0 permit 10            # route-map для редистрибьюции
   match interface Loopback0                       # выбираем только интерфейс Looback 0
   set origin igp                                  # устанавливаем origin в igp
   set community 65001:1                           # устанавливаем community
!
router bgp 65001                                                # Процесс BGP в AS 65001
   router-id 10.1.0.3                                           # Задаем Router ID
   maximum-paths 4                                              # Количество маршрутов для ECMP
   neighbor SPINE peer group                                    # peer-group SPINE
   neighbor SPINE remote-as 65000                               # AS соседа (eBGP)
   neighbor SPINE bfd                                           # Включаем BFD
   neighbor SPINE timers 3 9                                    # Настраиваем таймеры протокола
   neighbor SPINE send-community standard extended              # Настраиваем отправку community
   neighbor 10.1.2.0 peer group SPINE                           # Описываем соседей, включая их в peer-group
   neighbor 10.1.2.6 peer group SPINE
   !
   address-family ipv4
      neighbor SPINE activate                                   # активируем соседства в секции IPv4 unicast
      redistribute connected route-map RM_REDISTRIBUTE-Lo0      # редистрибьюцируем в BGP Loopback0
```
</details>

<details>
<summary>Контекст: Интерфейс</summary>

```eos
interface Ethernet1
   description P2P-LEAF1
   mtu 9214                                  # Увеличиваем MTU
   no switchport
   ip address 10.1.2.0/31
   bfd interval 100 min_rx 100 multiplier 3  # Настраиваем таймеры BFD
```
</details>

[Конфигурация Spine1](./configs/spine01.cfg)<br>
[Конфигурация Spine2](./configs/spine02.cfg)<br>
[Конфигурация Leaf1](./configs/leaf01.cfg)<br>
[Конфигурация Leaf2](./configs/leaf02.cfg)<br>
[Конфигурация Leaf3](./configs/leaf03.cfg)<br>

## 4. Проверка связности
<details>
<summary>Доступность транспортных адресов соседей SPINE1 -> LEAFX</summary>
  
```eos
SPINE1#  ping 10.1.2.1 repeat 3
PING 10.1.2.1 (10.1.2.1) 72(100) bytes of data.
80 bytes from 10.1.2.1: icmp_seq=1 ttl=64 time=0.321 ms
80 bytes from 10.1.2.1: icmp_seq=2 ttl=64 time=0.023 ms
80 bytes from 10.1.2.1: icmp_seq=3 ttl=64 time=0.020 ms

--- 10.1.2.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.020/0.121/0.321/0.141 ms, ipg/ewma 0.199/0.250 ms
SPINE1#ping 10.1.2.3 repeat 3
PING 10.1.2.3 (10.1.2.3) 72(100) bytes of data.
80 bytes from 10.1.2.3: icmp_seq=1 ttl=64 time=0.165 ms
80 bytes from 10.1.2.3: icmp_seq=2 ttl=64 time=0.016 ms
80 bytes from 10.1.2.3: icmp_seq=3 ttl=64 time=0.011 ms

--- 10.1.2.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.011/0.064/0.165/0.071 ms, ipg/ewma 0.125/0.129 ms
SPINE1#ping 10.1.2.5 repeat 3
PING 10.1.2.5 (10.1.2.5) 72(100) bytes of data.
80 bytes from 10.1.2.5: icmp_seq=1 ttl=64 time=0.207 ms
80 bytes from 10.1.2.5: icmp_seq=2 ttl=64 time=0.023 ms
80 bytes from 10.1.2.5: icmp_seq=3 ttl=64 time=0.016 ms

--- 10.1.2.5 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.016/0.082/0.207/0.088 ms, ipg/ewma 0.162/0.163 ms

```
</details>

<details>
<summary>Доступность транспортных адресов соседей SPINE2 -> LEAFX</summary>
  
```eos
SPINE2#ping 10.1.2.7 repeat 3
PING 10.1.2.7 (10.1.2.7) 72(100) bytes of data.
80 bytes from 10.1.2.7: icmp_seq=1 ttl=64 time=0.225 ms
80 bytes from 10.1.2.7: icmp_seq=2 ttl=64 time=0.029 ms
80 bytes from 10.1.2.7: icmp_seq=3 ttl=64 time=0.012 ms

--- 10.1.2.7 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.012/0.088/0.225/0.096 ms, ipg/ewma 0.171/0.177 ms
SPINE2#ping 10.1.2.9 repeat 3
PING 10.1.2.9 (10.1.2.9) 72(100) bytes of data.
80 bytes from 10.1.2.9: icmp_seq=1 ttl=64 time=0.169 ms
80 bytes from 10.1.2.9: icmp_seq=2 ttl=64 time=0.021 ms
80 bytes from 10.1.2.9: icmp_seq=3 ttl=64 time=0.020 ms

--- 10.1.2.9 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.020/0.070/0.169/0.070 ms, ipg/ewma 0.114/0.134 ms
SPINE2#ping 10.1.2.11 repeat 3
PING 10.1.2.11 (10.1.2.11) 72(100) bytes of data.
80 bytes from 10.1.2.11: icmp_seq=1 ttl=64 time=0.179 ms
80 bytes from 10.1.2.11: icmp_seq=2 ttl=64 time=0.021 ms
80 bytes from 10.1.2.11: icmp_seq=3 ttl=64 time=0.016 ms

--- 10.1.2.11 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.016/0.072/0.179/0.075 ms, ipg/ewma 0.117/0.141 ms
```
</details>

<details>
<summary>SPINE1 / show bfd peers</summary>
  
```eos
SPINE1#show bfd peers
VRF name: default
-----------------
DstAddr               MyDisc         YourDisc       Interface/Transport         Type               LastUp       LastDown            LastDiag    State
-------------- ---------------- ---------------- ------------------------- ------------ -------------------- -------------- ------------------- -----
10.1.2.1          1283456637       2005565758            Ethernet1(299)       normal       09/02/26 06:01             NA       No Diagnostic       Up
10.1.2.3          4238161139       3401854897            Ethernet2(296)       normal       09/02/26 06:10             NA       No Diagnostic       Up
10.1.2.5          1995265440       1486267061            Ethernet3(300)       normal       09/02/26 06:12             NA       No Diagnostic       Up
```
</details>

<details>
<summary>SPINE2 / show bfd peers</summary>
  
```eos
SPINE2#show bfd peers
VRF name: default
-----------------
DstAddr                MyDisc         YourDisc       Interface/Transport         Type               LastUp       LastDown            LastDiag    State
--------------- ---------------- ---------------- ------------------------- ------------ -------------------- -------------- ------------------- -----
10.1.2.7            278547670        232690230            Ethernet1(295)       normal       09/02/26 06:16             NA       No Diagnostic       Up
10.1.2.9           4198592186       1248918497            Ethernet2(302)       normal       09/02/26 06:17             NA       No Diagnostic       Up
10.1.2.11          1040641304       1259272234            Ethernet3(304)       normal       09/02/26 06:15             NA       No Diagnostic       Up
```
</details>

<details>
<summary>SPINE1 / show bgp summary</summary>
  
```eos
SPINE1#show bgp summary 
BGP summary information for VRF default
Router identifier 10.1.0.1, local AS number 65000
Neighbor          AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc   NLRI Adv
-------- ----------- ------------- ----------------------- -------------- ---------- ---------- ----------
10.1.2.1       65001 Established   IPv4 Unicast            Negotiated              1          1          3
10.1.2.3       65002 Established   IPv4 Unicast            Negotiated              1          1          3
10.1.2.5       65003 Established   IPv4 Unicast            Negotiated              1          1          3
```
</details>

<details>
<summary>SPINE2 / show bgp summary</summary>
  
```eos
SPINE2#show bgp summary 
BGP summary information for VRF default
Router identifier 10.1.0.2, local AS number 65000
Neighbor           AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc   NLRI Adv
--------- ----------- ------------- ----------------------- -------------- ---------- ---------- ----------
10.1.2.7        65001 Established   IPv4 Unicast            Negotiated              1          1          3
10.1.2.9        65002 Established   IPv4 Unicast            Negotiated              1          1          3
10.1.2.11       65003 Established   IPv4 Unicast            Negotiated              1          1          3
```
</details>

<details>
<summary> SPINE1 / show ip bgp</summary>
  
```eos
SPINE1#show ip bgp
BGP routing table information for VRF default
Router identifier 10.1.0.1, local AS number 65000
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast, q - Pending FIB install
                    % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.1.0.1/32            -                     -       -          -       0       i
 * >      10.1.0.3/32            10.1.2.1              0       -          100     0       65001 i
 * >      10.1.0.4/32            10.1.2.3              0       -          100     0       65002 i
 * >      10.1.0.5/32            10.1.2.5              0       -          100     0       65003 i
```
</details>

<details>
<summary> SPINE2 / show ip bgp</summary>
  
```eos
SPINE2#show ip bgp 
BGP routing table information for VRF default
Router identifier 10.1.0.2, local AS number 65000
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast, q - Pending FIB install
                    % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.1.0.2/32            -                     -       -          -       0       i
 * >      10.1.0.3/32            10.1.2.7              0       -          100     0       65001 i
 * >      10.1.0.4/32            10.1.2.9              0       -          100     0       65002 i
 * >      10.1.0.5/32            10.1.2.11             0       -          100     0       65003 i
```
</details>

<details>
<summary>LEAF1 / show ip bgp</summary>
  
```eos
LEAF1# show ip bgp 
BGP routing table information for VRF default
Router identifier 10.1.0.3, local AS number 65001
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast, q - Pending FIB install
                    % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.1.0.1/32            10.1.2.0              0       -          100     0       65000 i
 * >      10.1.0.2/32            10.1.2.6              0       -          100     0       65000 i
 * >      10.1.0.3/32            -                     -       -          -       0       i
 * >Ec    10.1.0.4/32            10.1.2.0              0       -          100     0       65000 65002 i
 *  ec    10.1.0.4/32            10.1.2.6              0       -          100     0       65000 65002 i
 * >Ec    10.1.0.5/32            10.1.2.0              0       -          100     0       65000 65003 i
 *  ec    10.1.0.5/32            10.1.2.6              0       -          100     0       65000 65003 i
```
</details>

<details>
<summary>LEAF1 / show ip route</summary>
  
```eos
LEAF1#show ip route 

VRF: default
Source Codes:
       C - connected, S - static, K - kernel,
       O - OSPF, O IA - OSPF inter area, O E1 - OSPF external type 1,
       O E2 - OSPF external type 2, O N1 - OSPF NSSA external type 1,
       O N2 - OSPF NSSA external type2, O3 - OSPFv3,
       O3 IA - OSPFv3 inter area, O3 E1 - OSPFv3 external type 1,
       O3 E2 - OSPFv3 external type 2,
       O3 N1 - OSPFv3 NSSA external type 1,
       O3 N2 - OSPFv3 NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route,
       CL - CBF Leaked Route

Gateway of last resort:
 S        0.0.0.0/0 [1/0]
           via 172.20.20.1, Management0

 B E      10.1.0.1/32 [200/0]
           via 10.1.2.0, Ethernet1
 B E      10.1.0.2/32 [200/0]
           via 10.1.2.6, Ethernet2
 C        10.1.0.3/32
           directly connected, Loopback0
 B E      10.1.0.4/32 [200/0]
           via 10.1.2.0, Ethernet1
           via 10.1.2.6, Ethernet2
 B E      10.1.0.5/32 [200/0]
           via 10.1.2.0, Ethernet1
           via 10.1.2.6, Ethernet2
 C        10.1.2.0/31
           directly connected, Ethernet1
 C        10.1.2.6/31
           directly connected, Ethernet2
```
</details>

<details>
<summary>LEAF1 / ping до Lo0 LEAF2,3</summary>
  
```eos
LEAF1#ping 10.1.0.4 source loopback 0 repeat 3
PING 10.1.0.4 (10.1.0.4) from 10.1.0.3 : 72(100) bytes of data.
80 bytes from 10.1.0.4: icmp_seq=1 ttl=63 time=1.10 ms
80 bytes from 10.1.0.4: icmp_seq=2 ttl=63 time=0.343 ms
80 bytes from 10.1.0.4: icmp_seq=3 ttl=63 time=0.318 ms

--- 10.1.0.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2ms
rtt min/avg/max/mdev = 0.318/0.587/1.101/0.363 ms, ipg/ewma 1.098/0.920 ms
LEAF1#ping 10.1.0.5 source loopback 0 repeat 3
PING 10.1.0.5 (10.1.0.5) from 10.1.0.3 : 72(100) bytes of data.
80 bytes from 10.1.0.5: icmp_seq=1 ttl=63 time=0.547 ms
80 bytes from 10.1.0.5: icmp_seq=2 ttl=63 time=0.330 ms
80 bytes from 10.1.0.5: icmp_seq=3 ttl=63 time=0.319 ms

--- 10.1.0.5 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2ms
rtt min/avg/max/mdev = 0.319/0.398/0.547/0.104 ms, ipg/ewma 1.004/0.494 ms
```
</details>

<details>
<summary>LEAF2 / ping до Lo0 LEAF1,3</summary>
  
```eos
LEAF2# ping 10.1.0.3 source loopback 0 repeat 3
PING 10.1.0.3 (10.1.0.3) from 10.1.0.4 : 72(100) bytes of data.
80 bytes from 10.1.0.3: icmp_seq=1 ttl=63 time=0.557 ms
80 bytes from 10.1.0.3: icmp_seq=2 ttl=63 time=0.330 ms
80 bytes from 10.1.0.3: icmp_seq=3 ttl=63 time=0.315 ms

--- 10.1.0.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2ms
rtt min/avg/max/mdev = 0.315/0.400/0.557/0.110 ms, ipg/ewma 1.001/0.502 ms
LEAF2#ping 10.1.0.5 source loopback 0 repeat 3
PING 10.1.0.5 (10.1.0.5) from 10.1.0.4 : 72(100) bytes of data.
80 bytes from 10.1.0.5: icmp_seq=1 ttl=63 time=0.649 ms
80 bytes from 10.1.0.5: icmp_seq=2 ttl=63 time=0.331 ms
80 bytes from 10.1.0.5: icmp_seq=3 ttl=63 time=0.324 ms

--- 10.1.0.5 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2ms
rtt min/avg/max/mdev = 0.324/0.434/0.649/0.151 ms, ipg/ewma 1.001/0.573 ms
```
</details>

<details>
<summary>LEAF3 / ping до Lo0 LEAF1,2</summary>
  
```eos
LEAF3# ping 10.1.0.3 source loopback 0 repeat 3
PING 10.1.0.3 (10.1.0.3) from 10.1.0.5 : 72(100) bytes of data.
80 bytes from 10.1.0.3: icmp_seq=1 ttl=63 time=0.479 ms
80 bytes from 10.1.0.3: icmp_seq=2 ttl=63 time=0.310 ms
80 bytes from 10.1.0.3: icmp_seq=3 ttl=63 time=0.324 ms

--- 10.1.0.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1ms
rtt min/avg/max/mdev = 0.310/0.371/0.479/0.076 ms, ipg/ewma 0.470/0.441 ms
LEAF3#ping 10.1.0.4 source loopback 0 repeat 3
PING 10.1.0.4 (10.1.0.4) from 10.1.0.5 : 72(100) bytes of data.
80 bytes from 10.1.0.4: icmp_seq=1 ttl=63 time=0.653 ms
80 bytes from 10.1.0.4: icmp_seq=2 ttl=63 time=0.364 ms
80 bytes from 10.1.0.4: icmp_seq=3 ttl=63 time=0.328 ms

--- 10.1.0.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2ms
rtt min/avg/max/mdev = 0.328/0.448/0.653/0.145 ms, ipg/ewma 1.001/0.580 ms
```
</details>

## 5. Особенности
У данной схемы имеются следующие особенности:
- SPINE находятся в одной AS для предотвращения path-hunting.
- Из-за отсутствия связности между SPINE и того, что они находятся в одной AS, они не имеют информации о префиксах друг друга. Но ввиду того, что мы строим Underlay для VxLAN/EVPN-фабрики нас интересует только распространение адресов VTEP (Loopback LEAF).
- Схема так же "ломается" при двойном отказе.
<img width="936" height="711" alt="image" src="https://github.com/user-attachments/assets/24a77a39-0796-45a3-8e40-37485dabfb3c" />
