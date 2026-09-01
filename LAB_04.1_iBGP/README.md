# Лабораторная работа №3 "Построение Underlay сети (iBGP)"

## Задание:
1. [Подготовка стенда](#1-подготовка-стенда)
2. [Разработка адресного плана](#2-разработка-адресного-плана)
3. [Настройка Underlay на базе iBGP](#3-настройка-underlay-на-базе-ibgp)
4. [Проверка связности](#4-проверка-связности)

## 1. Подготовка стенда
В качестве платформы для организации стенда был выбран Containerlab, развернутый на WSL, с использованием образа Arista cEOS.
Получившийся стенд выглядит следующим образом ([Топология для Containetlab](containerlab/lab02.yaml)):
<img width="638" height="488" alt="image" src="https://github.com/user-attachments/assets/bda2c84c-6cab-4758-967a-651d6b85fb95" />

## 2. Разработка адресного плана
Для адресного плана предлагаем использовать приватную сеть 10.0.0.0/8. При этом второй октет мы будем использовать как индекс ЦОД, для которого предназначена адресация. Для Lo0 предлагаем зарезервировать подсеть /24 (сможем адресовать 256 устройств). Для транспортных подсетей /31 предлагаю зарезервировать подсеть /23 (запас вплоть до фабрики 8 Spine / 32 Leaf). Для адресации сервисов зарезервируем подсеть /21. Итого, общая адресация каждого ЦОД будет суммироваться до /20 (с учетом зарезервированных адресов для возможного расширения).

| Устройство | Подсеть     |
| ---------- | ----------- |
| Loopback0  | 10.x.0.0/24 |
| Reserved   | 10.x.1.0/24 |
| Transport  | 10.x.2.0/23 |
| Reserved   | 10.x.4.0/22 |
| Service    | 10.x.8.0/21 |

## 3. Настройка Underlay на базе iBGP
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
<summary>Контекст: Процесс BGP SPINE</summary>

```eos
route-map RM_REDISTRIBUTE-Lo0 permit 10            # route-map для редистрибьюции
   match interface Loopback0                       # выбираем только интерфейс Looback 0
   set origin igp                                  # устанавливаем oridgin в ibgp
!
peer-filter LEAFS-AS-FILTER                        # peer-filter для фильтрации соседств с LEAF
   10 match as-range 65001 result accept           # принимаем только iBGP-соседей (с нашей AS)

router bgp 65001                                                               # Процесс BGP в AS 65001
   router-id 10.1.0.1                                                          # Задаем Router ID
   maximum-paths 4                                                             # количество маршрутов для ECMP
   bgp listen range 10.1.2.0/23 peer-group LEAFS peer-filter LEAFS-AS-FILTER   # Принимаем соседей с адресами из 10.1.2.0/23 и нашей AS
   neighbor LEAFS peer group                                                   # peer-group LEAFS
   neighbor LEAFS remote-as 65001                                              # AS соседа (iBGP)
   neighbor LEAFS bfd                                                          # включаем BFD
   neighbor LEAFS route-reflector-client                                       # сосед будет являться route-reflector клиентом
   !
   address-family ipv4
      neighbor LEAFS activate                                                  # активируем соседство в секции IPv4 unicast
      redistribute connected route-map RM_REDISTRIBUTE-Lo0                     # редистрибьюцируем в BGP Loopback0
      network 10.1.2.0/31                                                      # добавляем в BGP транспортные сети
      network 10.1.2.2/31
      network 10.1.2.4/31
```
</details>

<summary>Контекст: Процесс BGP LEAF</summary>

```eos
router bgp 65001                                                               # Процесс BGP в AS 65001
   router-id 10.1.0.1                                                          # Задаем Router ID
   maximum-paths 4                                                             # количество маршрутов для ECMP
   bgp listen range 10.1.2.0/23 peer-group LEAFS peer-filter LEAFS-AS-FILTER   # Принимаем соседей с адресами из 10.1.2.0/23 и нашей AS
   neighbor LEAFS peer group                                                   # peer-group LEAFS
   neighbor LEAFS remote-as 65001                                              # AS соседа (iBGP)
   neighbor LEAFS bfd                                                          # включаем BFD
   neighbor LEAFS route-reflector-client                                       # сосед будет являться route-reflector клиентом
   !
   address-family ipv4
      neighbor LEAFS activate                                                  # активируем соседство в секции IPv4 unicast
      redistribute connected route-map RM_REDISTRIBUTE-Lo0                     # редистрибьюцируем в BGP Loopback0
      network 10.1.2.0/31                                                      # добавляем в BGP транспортные сети
      network 10.1.2.2/31
      network 10.1.2.4/31
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
SPINE1#ping 10.1.2.1 repeat 3
PING 10.1.2.1 (10.1.2.1) 72(100) bytes of data.
80 bytes from 10.1.2.1: icmp_seq=1 ttl=64 time=0.256 ms
80 bytes from 10.1.2.1: icmp_seq=2 ttl=64 time=0.023 ms
80 bytes from 10.1.2.1: icmp_seq=3 ttl=64 time=0.020 ms

--- 10.1.2.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.020/0.099/0.256/0.110 ms, ipg/ewma 0.176/0.201 ms
SPINE1#ping 10.1.2.3 repeat 3
PING 10.1.2.3 (10.1.2.3) 72(100) bytes of data.
80 bytes from 10.1.2.3: icmp_seq=1 ttl=64 time=0.138 ms
80 bytes from 10.1.2.3: icmp_seq=2 ttl=64 time=0.014 ms
80 bytes from 10.1.2.3: icmp_seq=3 ttl=64 time=0.011 ms

--- 10.1.2.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.011/0.054/0.138/0.059 ms, ipg/ewma 0.113/0.108 ms
SPINE1#ping 10.1.2.5 repeat 3
PING 10.1.2.5 (10.1.2.5) 72(100) bytes of data.
80 bytes from 10.1.2.5: icmp_seq=1 ttl=64 time=0.198 ms
80 bytes from 10.1.2.5: icmp_seq=2 ttl=64 time=0.020 ms
80 bytes from 10.1.2.5: icmp_seq=3 ttl=64 time=0.019 ms

--- 10.1.2.5 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
```
</details>

<details>
<summary>Доступность транспортных адресов соседей SPINE2 -> LEAFX</summary>
  
```eos
SPINE2#ping 10.1.2.7 repeat 3
PING 10.1.2.7 (10.1.2.7) 72(100) bytes of data.
80 bytes from 10.1.2.7: icmp_seq=1 ttl=64 time=0.230 ms
80 bytes from 10.1.2.7: icmp_seq=2 ttl=64 time=0.023 ms
80 bytes from 10.1.2.7: icmp_seq=3 ttl=64 time=0.019 ms

--- 10.1.2.7 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.019/0.090/0.230/0.098 ms, ipg/ewma 0.168/0.181 ms
SPINE2#ping 10.1.2.9 repeat 3
PING 10.1.2.9 (10.1.2.9) 72(100) bytes of data.
80 bytes from 10.1.2.9: icmp_seq=1 ttl=64 time=0.178 ms
80 bytes from 10.1.2.9: icmp_seq=2 ttl=64 time=0.021 ms
80 bytes from 10.1.2.9: icmp_seq=3 ttl=64 time=0.024 ms

--- 10.1.2.9 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.021/0.074/0.178/0.073 ms, ipg/ewma 0.124/0.141 ms
SPINE2#ping 10.1.2.11 repeat 3
PING 10.1.2.11 (10.1.2.11) 72(100) bytes of data.
80 bytes from 10.1.2.11: icmp_seq=1 ttl=64 time=0.165 ms
80 bytes from 10.1.2.11: icmp_seq=2 ttl=64 time=0.021 ms
80 bytes from 10.1.2.11: icmp_seq=3 ttl=64 time=0.019 ms

--- 10.1.2.11 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.019/0.068/0.165/0.068 ms, ipg/ewma 0.117/0.131 ms
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
10.1.2.1          2920583622       2808734920            Ethernet1(299)       normal       08/25/26 13:55             NA       No Diagnostic       Up
10.1.2.3          2426130429       2220068917            Ethernet2(296)       normal       08/25/26 11:45             NA       No Diagnostic       Up
10.1.2.5          3238291849       3871824122            Ethernet3(300)       normal       08/25/26 11:46             NA       No Diagnostic       Up

</details>

<details>
<summary>SPINE2 / show bfd peers</summary>
  
```eos
SPINE2#show bfd peers
VRF name: default
-----------------
DstAddr                MyDisc         YourDisc       Interface/Transport         Type               LastUp       LastDown            LastDiag    State
--------------- ---------------- ---------------- ------------------------- ------------ -------------------- -------------- ------------------- -----
10.1.2.7           2338626086        105614829            Ethernet1(295)       normal       08/25/26 11:44             NA       No Diagnostic       Up
10.1.2.9           4212359855       4086362094            Ethernet2(302)       normal       08/25/26 11:45             NA       No Diagnostic       Up
10.1.2.11          4037889458       4030422697            Ethernet3(304)       normal       08/25/26 13:55             NA       No Diagnostic       Up
```
</details>

<details>
<summary>SPINE1 / show bgp summary</summary>
  
```eos
SPINE1#show bgp summary 
BGP summary information for VRF default
Router identifier 10.1.0.1, local AS number 65001
Neighbor          AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc   NLRI Adv
-------- ----------- ------------- ----------------------- -------------- ---------- ---------- ----------
10.1.2.1       65001 Established   IPv4 Unicast            Negotiated              1          1          3
10.1.2.3       65001 Established   IPv4 Unicast            Negotiated              1          1          3
10.1.2.5       65001 Established   IPv4 Unicast            Negotiated              1          1          3
```
</details>

<details>
<summary>SPINE2 / show bgp summary</summary>
  
```eos
SPINE2#show bgp summary
BGP summary information for VRF default
Router identifier 10.1.0.2, local AS number 65001
Neighbor           AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc   NLRI Adv
--------- ----------- ------------- ----------------------- -------------- ---------- ---------- ----------
10.1.2.7        65001 Established   IPv4 Unicast            Negotiated              1          1          3
10.1.2.9        65001 Established   IPv4 Unicast            Negotiated              1          1          3
10.1.2.11       65001 Established   IPv4 Unicast            Negotiated              1          1          3
```
</details>


<details>
<summary> SPINE1 / show ip bgp</summary>
  
```eos
SPINE1#show ip bgp 
BGP routing table information for VRF default
Router identifier 10.1.0.1, local AS number 65001
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast, q - Pending FIB install
                    % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.1.0.1/32            -                     -       -          -       0       i
 * >      10.1.0.3/32            10.1.2.1              0       -          100     0       i
 * >      10.1.0.4/32            10.1.2.3              0       -          100     0       i
 * >      10.1.0.5/32            10.1.2.5              0       -          100     0       i
```
</details>

<details>
<summary> SPINE2 / show ip bgp</summary>
  
```eos
SPINE2#show ip bgp
BGP routing table information for VRF default
Router identifier 10.1.0.2, local AS number 65001
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast, q - Pending FIB install
                    % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.1.0.2/32            -                     -       -          -       0       i
 * >      10.1.0.3/32            10.1.2.7              0       -          100     0       i
 * >      10.1.0.4/32            10.1.2.9              0       -          100     0       i
 * >      10.1.0.5/32            10.1.2.11             0       -          100     0       i
</details>

<details>
<summary>LEAF1 / show ip bgp</summary>
  
```eos
LEAF1#sh ip bgp
BGP routing table information for VRF default
Router identifier 10.1.0.3, local AS number 65001
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast, q - Pending FIB install
                    % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.1.0.1/32            10.1.2.0              0       -          100     0       i
 * >      10.1.0.2/32            10.1.2.6              0       -          100     0       i
 * >      10.1.0.3/32            -                     -       -          -       0       i
 * >E     10.1.0.4/32            10.1.2.3              0       -          100     0       i Or-ID: 10.1.0.4 C-LST: 10.1.0.1 
 *  ec    10.1.0.4/32            10.1.2.9              0       -          100     0       i Or-ID: 10.1.0.4 C-LST: 10.1.0.2 
 * >Ec    10.1.0.5/32            10.1.2.5              0       -          100     0       i Or-ID: 10.1.0.5 C-LST: 10.1.0.1 
 *  e     10.1.0.5/32            10.1.2.11             0       -          100     0       i Or-ID: 10.1.0.5 C-LST: 10.1.0.2 
```
</details>

<details>
<summary>LEAF1 / ping до Lo0 LEAF2,3</summary>
  
```eos
LEAF1#ping 10.1.0.4 source loopback 0 repeat 3
PING 10.1.0.4 (10.1.0.4) from 10.1.0.3 : 72(100) bytes of data.
80 bytes from 10.1.0.4: icmp_seq=1 ttl=63 time=2.03 ms
80 bytes from 10.1.0.4: icmp_seq=2 ttl=63 time=0.381 ms
80 bytes from 10.1.0.4: icmp_seq=3 ttl=63 time=0.319 ms

--- 10.1.0.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 4ms
rtt min/avg/max/mdev = 0.319/0.911/2.034/0.794 ms, ipg/ewma 2.120/1.638 ms
LEAF1#ping 10.1.0.5 source loopback 0 repeat 3
PING 10.1.0.5 (10.1.0.5) from 10.1.0.3 : 72(100) bytes of data.
80 bytes from 10.1.0.5: icmp_seq=1 ttl=63 time=1.42 ms
80 bytes from 10.1.0.5: icmp_seq=2 ttl=63 time=0.376 ms
80 bytes from 10.1.0.5: icmp_seq=3 ttl=63 time=0.310 ms

--- 10.1.0.5 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 3ms
rtt min/avg/max/mdev = 0.310/0.700/1.416/0.506 ms, ipg/ewma 1.287/1.164 ms
```
</details>

<details>
<summary>LEAF2 / ping до Lo0 LEAF1,3</summary>
  
```eos
LEAF2#ping 10.1.0.3 source loopback 0 repeat 3
PING 10.1.0.3 (10.1.0.3) from 10.1.0.4 : 72(100) bytes of data.
80 bytes from 10.1.0.3: icmp_seq=1 ttl=63 time=0.754 ms
80 bytes from 10.1.0.3: icmp_seq=2 ttl=63 time=0.349 ms
80 bytes from 10.1.0.3: icmp_seq=3 ttl=63 time=0.349 ms

--- 10.1.0.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2ms
rtt min/avg/max/mdev = 0.349/0.484/0.754/0.190 ms, ipg/ewma 1.023/0.659 ms
LEAF2#ping 10.1.0.5 source loopback 0 repeat 3
PING 10.1.0.5 (10.1.0.5) from 10.1.0.4 : 72(100) bytes of data.
80 bytes from 10.1.0.5: icmp_seq=1 ttl=63 time=0.745 ms
80 bytes from 10.1.0.5: icmp_seq=2 ttl=63 time=0.323 ms
80 bytes from 10.1.0.5: icmp_seq=3 ttl=63 time=0.385 ms

--- 10.1.0.5 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2ms
rtt min/avg/max/mdev = 0.323/0.484/0.745/0.186 ms, ipg/ewma 1.001/0.653 ms
```
</details>

<details>
<summary>LEAF3 / ping до Lo0 LEAF1,2</summary>
  
```eos
LEAF3#ping 10.1.0.3 source loopback 0 repeat 3
PING 10.1.0.3 (10.1.0.3) from 10.1.0.5 : 72(100) bytes of data.
80 bytes from 10.1.0.3: icmp_seq=1 ttl=63 time=0.508 ms
80 bytes from 10.1.0.3: icmp_seq=2 ttl=63 time=0.383 ms
80 bytes from 10.1.0.3: icmp_seq=3 ttl=63 time=0.340 ms

--- 10.1.0.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1ms
rtt min/avg/max/mdev = 0.340/0.410/0.508/0.071 ms, ipg/ewma 0.745/0.473 ms
LEAF3#ping 10.1.0.4 source loopback 0 repeat 3
PING 10.1.0.4 (10.1.0.4) from 10.1.0.5 : 72(100) bytes of data.
80 bytes from 10.1.0.4: icmp_seq=1 ttl=63 time=0.550 ms
80 bytes from 10.1.0.4: icmp_seq=2 ttl=63 time=0.439 ms
80 bytes from 10.1.0.4: icmp_seq=3 ttl=63 time=0.308 ms

--- 10.1.0.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2ms
rtt min/avg/max/mdev = 0.308/0.432/0.550/0.098 ms, ipg/ewma 1.001/0.507 ms
```
</details>
