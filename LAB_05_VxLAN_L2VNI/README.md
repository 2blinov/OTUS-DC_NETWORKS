# Лабораторная работа №5 "VxLAN L2VNI"

## Задание:
1. [Подготовка стенда](#1-подготовка-стенда)
2. [Разработка адресного плана](#2-разработка-адресного-плана)
3. [Настройка eBGP для Underlay и EVPN](#3-настройка-bgp-для-underlay-и-evpn)
4. [Проверка связности](#4-проверка-связности)
5. [Настройка VxLAN L2VNI](#4-настройка-vxlan-l2vni)
6. [Проверка работы L2VNI](#6-проверка-работы-l2vni)

## 1. Подготовка стенда
В качестве платформы для организации стенда был выбран Containerlab, развернутый на WSL, с использованием образов Cisco Nexus, Arista cEOS, Fortigate.
Получившийся стенд выглядит следующим образом ([Топология для Containetlab](containerlab/lab05.yaml)):
<img width="930" height="391" alt="image" src="https://github.com/user-attachments/assets/3ce90ccd-892c-4b02-a52b-0cee3e8908d6" />

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
| BORDER | 10.1.0.5/32 |

Транспортные подсети
| Link           | Network      |
| -------------- | ------------ |
| SPINE1 — LEAF1 | 10.1.2.0/31  |
| SPINE1 — LEAF2 | 10.1.2.2/31  |
| SPINE1 — BORDER | 10.1.2.4/31  |
| SPINE2 — LEAF1 | 10.1.2.6/31  |
| SPINE2 — LEAF2 | 10.1.2.8/31  |
| SPINE2 — BORDER | 10.1.2.10/31 |

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
   
router bgp 65000                                                                        # Процесс BGP в AS 65000
   router-id 10.1.0.1                                                                   # Задаем Router ID
   maximum-paths 4                                                                      # количество маршрутов для ECMP
   bgp listen range 10.1.0.0/24 peer-group LEAFS-EVPN peer-filter LEAFS-AS-FILTER       # Принимаем соседей EVPN с адресами из 10.1.0.0/24 и AS 65001-65003
   bgp listen range 10.1.2.0/23 peer-group LEAFS-UNDERLAY peer-filter LEAFS-AS-FILTER   # Принимаем соседей UNDERLAY с адресами из 10.1.2.0/23 и AS 65001-65003
   neighbor LEAFS-EVPN peer group                                                       # peer-group LEAFS-EVPN
   neighbor LEAFS-EVPN next-hop-unchanged                                               # Нам нужно строить туннели Leaf-Leaf, поэтому не меняем next-hop
   neighbor LEAFS-EVPN update-source Loopback0                                          # Соседство строим с Lo0
   neighbor LEAFS-EVPN ebgp-multihop 5                                                  # IP TTL для eBGP-сессии
   neighbor LEAFS-EVPN send-community extended                                          # Включаем передачу расширенных community (передача RT необходима для EVPN)
   neighbor LEAFS-UNDERLAY peer group                                                   # peer-group LEAFS-UNDERLAY
   neighbor LEAFS-UNDERLAY bfd                                                          # Включаем BFD
   neighbor LEAFS-UNDERLAY timers 3 9                                                   # Настраиваем таймеры протокола
   !
   address-family evpn
      neighbor LEAFS-EVPN activate                                                      # активируем соседства в секции evpn
   !
   address-family ipv4
      no neighbor LEAFS-EVPN activate
      neighbor LEAFS-UNDERLAY activate                                                  # активируем соседства в секции IPv4 unicast
      redistribute connected route-map RM_REDISTRIBUTE-Lo0                              # редистрибьюцируем в BGP Loopback0
```
</details>

<details>
<summary>Контекст: Процесс BGP/LEAF/NX</summary>

```eos
route-map RM_REDISTRIBUTE-Lo0 permit 10                  # route-map для редистрибьюции
  match interface loopback0                              # выбираем только интерфейс Looback 0

router bgp 65001                                         # Процесс BGP в AS 65000
  router-id 10.1.0.3                                     # Задаем Router ID
  address-family ipv4 unicast
    redistribute direct route-map RM_REDISTRIBUTE-Lo0    # редистрибьюцируем в BGP Loopback0
    maximum-paths 4                                      # количество маршрутов для ECMP
  address-family l2vpn evpn
    maximum-paths 4                                      # количество маршрутов для ECMP
  neighbor 10.1.0.1                                      # SPINE1 / EVPN
    remote-as 65000
    update-source loopback0
    ebgp-multihop 5
    address-family l2vpn evpn
      send-community
      send-community extended
  neighbor 10.1.0.2                                      # SPINE2 / EVPN
    remote-as 65000
    update-source loopback0
    ebgp-multihop 5
    address-family l2vpn evpn
      send-community
      send-community extended
  neighbor 10.1.2.0                                      # SPINE1 / UNDERLAY
    bfd
    remote-as 65000
    address-family ipv4 unicast
  neighbor 10.1.2.6                                      # SPINE2 / UNDERLAY
    bfd
    remote-as 65000
    address-family ipv4 unicast
```
</details>

<details>
<summary>Контекст: Процесс BGP/LEAF/Arista</summary>

```eos
router bgp 65002
   router-id 10.1.0.4
   maximum-paths 4
   neighbor SPINE-EVPN peer group
   neighbor SPINE-EVPN remote-as 65000
   neighbor SPINE-EVPN next-hop-unchanged
   neighbor SPINE-EVPN update-source Loopback0
   neighbor SPINE-EVPN ebgp-multihop 5
   neighbor SPINE-EVPN send-community extended
   neighbor SPINE-UNDERLAY peer group
   neighbor SPINE-UNDERLAY remote-as 65000
   neighbor SPINE-UNDERLAY bfd
   neighbor 10.1.0.1 peer group SPINE-EVPN
   neighbor 10.1.0.2 peer group SPINE-EVPN
   neighbor 10.1.2.2 peer group SPINE-UNDERLAY
   neighbor 10.1.2.8 peer group SPINE-UNDERLAY
   !
   address-family evpn
      neighbor SPINE-EVPN activate
   !
   address-family ipv4
      no neighbor SPINE-EVPN activate
      neighbor SPINE-UNDERLAY activate
      redistribute connected route-map RM_REDISTRIBUTE-Lo0
```
</details>

[Конфигурация Spine1](./configs/spine1.conf)<br>
[Конфигурация Spine2](./configs/spine2.conf)<br>
[Конфигурация Leaf1](./configs/leaf1.conf)<br>
[Конфигурация Leaf2](./configs/leaf2.conf)<br>
[Конфигурация Leaf3](./configs/border.conf)<br>

## 4. Проверка связности
<summary>LEAF1 / show ip bgp summary</summary>
  
```eos
LEAF1# show ip bgp summary 
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 10.1.0.3, local AS number 65001
BGP table version is 58, IPv4 Unicast config peers 2, capable peers 2
5 network entries and 7 paths using 1692 bytes of memory
BGP attribute entries [4/1472], BGP AS path entries [3/26]
BGP community entries [0/0], BGP clusterlist entries [0/0]

Neighbor        V    AS    MsgRcvd    MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.1.2.0        4 65000      34475      30414       58    0    0    1d01h 3         
10.1.2.6        4 65000      34564      30491       58    0    0    1d01h 3         
```
</details>

<summary>LEAF1 / show bgp l2vpn evpn summary</summary>
  
```eos
LEAF1# show bgp l2vpn evpn summary 
BGP summary information for VRF default, address family L2VPN EVPN
BGP router identifier 10.1.0.3, local AS number 65001
BGP table version is 141, L2VPN EVPN config peers 2, capable peers 2
20 network entries and 28 paths using 6000 bytes of memory
BGP attribute entries [20/7360], BGP AS path entries [2/20]
BGP community entries [0/0], BGP clusterlist entries [0/0]

Neighbor        V    AS    MsgRcvd    MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.1.0.1        4 65000       1761       1539      141    0    0    1d01h 8         
10.1.0.2        4 65000       1772       1543      141    0    0    1d01h 8         

Neighbor        T    AS Type-1     Type-2     Type-3     Type-4     Type-5     Type-12   
10.1.0.1        I 65000 0          4          4          0          0          0         
10.1.0.2        I 65000 0          4          4          0          0          0 ```
```
</details>

<details>
<summary>LEAF2 / show bgp summary</summary>
  
```eos
LEAF2#  show bgp summary 
BGP summary information for VRF default
Router identifier 10.1.0.4, local AS number 65002
Neighbor          AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc   NLRI Adv
-------- ----------- ------------- ----------------------- -------------- ---------- ---------- ----------
10.1.0.1       65000 Established   L2VPN EVPN              Negotiated              8          8          7
10.1.0.2       65000 Established   L2VPN EVPN              Negotiated              8          8          9
10.1.2.2       65000 Established   IPv4 Unicast            Negotiated              3          3          2
10.1.2.8       65000 Established   IPv4 Unicast            Negotiated              3          3          4
```
</details>

<details>
<summary> LEAF1 / show ip bgp</summary>
  
```eos
LEAF1# sh ip bgp 
BGP routing table information for VRF default, address family IPv4 Unicast
BGP table version is 58, Local Router ID is 10.1.0.3
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-injected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - best2

   Network            Next Hop            Metric     LocPrf     Weight Path
*>e10.1.0.1/32        10.1.2.0                                       0 65000 i
*>e10.1.0.2/32        10.1.2.6                                       0 65000 i
*>r10.1.0.3/32        0.0.0.0                  0        100      32768 ?
*|e10.1.0.4/32        10.1.2.0                                       0 65000 65002 i
*>e                   10.1.2.6                                       0 65000 65002 i
*>e10.1.0.5/32        10.1.2.0                                       0 65000 65003 i
*|e                   10.1.2.6                                       0 65000 65003 i
```
</details>

<details>
<summary> LEAF2 / show ip bgp</summary>
  
```eos
LEAF2#sh ip bgp 
BGP routing table information for VRF default
Router identifier 10.1.0.4, local AS number 65002
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast, q - Pending FIB install
                    % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.1.0.1/32            10.1.2.2              0       -          100     0       65000 i
 * >      10.1.0.2/32            10.1.2.8              0       -          100     0       65000 i
 * >Ec    10.1.0.3/32            10.1.2.2              0       -          100     0       65000 65001 ?
 *  ec    10.1.0.3/32            10.1.2.8              0       -          100     0       65000 65001 ?
 * >      10.1.0.4/32            -                     -       -          -       0       i
 * >Ec    10.1.0.5/32            10.1.2.2              0       -          100     0       65000 65003 i
 *  ec    10.1.0.5/32            10.1.2.8              0       -          100     0       65000 65003 i
```
</details>

<details>
<summary>LEAF1 / ping до Lo0 LEAF2, BORDER</summary>
  
```eos
LEAF1# ping 10.1.0.4 source-interface loopback 0 count 3
PING 10.1.0.4 (10.1.0.4): 56 data bytes
64 bytes from 10.1.0.4: icmp_seq=0 ttl=62 time=2.074 ms
64 bytes from 10.1.0.4: icmp_seq=1 ttl=62 time=1.245 ms
64 bytes from 10.1.0.4: icmp_seq=2 ttl=62 time=1.264 ms

--- 10.1.0.4 ping statistics ---
3 packets transmitted, 3 packets received, 0.00% packet loss
round-trip min/avg/max = 1.245/1.527/2.074 ms
LEAF1# ping 10.1.0.5 source-interface loopback 0 count 3
PING 10.1.0.5 (10.1.0.5): 56 data bytes
64 bytes from 10.1.0.5: icmp_seq=0 ttl=62 time=1.67 ms
64 bytes from 10.1.0.5: icmp_seq=1 ttl=62 time=1.264 ms
64 bytes from 10.1.0.5: icmp_seq=2 ttl=62 time=1.005 ms

--- 10.1.0.5 ping statistics ---
3 packets transmitted, 3 packets received, 0.00% packet loss
round-trip min/avg/max = 1.005/1.313/1.67 ms
```
</details>

<details>
<summary>LEAF2 / ping до Lo0 LEAF1, BORDER</summary>
  
```eos
LEAF2#ping 10.1.0.3 source loopback 0 repeat 3
PING 10.1.0.3 (10.1.0.3) from 10.1.0.4 : 72(100) bytes of data.
80 bytes from 10.1.0.3: icmp_seq=1 ttl=254 time=3.06 ms
80 bytes from 10.1.0.3: icmp_seq=2 ttl=254 time=0.926 ms
80 bytes from 10.1.0.3: icmp_seq=3 ttl=254 time=1.08 ms

--- 10.1.0.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 6ms
rtt min/avg/max/mdev = 0.926/1.686/3.056/0.970 ms, ipg/ewma 3.152/2.575 ms
LEAF2#ping 10.1.0.5 source loopback 0 repeat 3
PING 10.1.0.5 (10.1.0.5) from 10.1.0.4 : 72(100) bytes of data.
80 bytes from 10.1.0.5: icmp_seq=1 ttl=63 time=1.14 ms
80 bytes from 10.1.0.5: icmp_seq=2 ttl=63 time=0.562 ms
80 bytes from 10.1.0.5: icmp_seq=3 ttl=63 time=0.333 ms

--- 10.1.0.5 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2ms
rtt min/avg/max/mdev = 0.333/0.679/1.143/0.340 ms, ipg/ewma 1.139/0.978 ms
```
</details>

<details>
<summary>BORDER / ping до Lo0 LEAF1,2</summary>
  
```eos
BORDER#ping 10.1.0.3 source loopback 0 repeat 3
PING 10.1.0.3 (10.1.0.3) from 10.1.0.5 : 72(100) bytes of data.
80 bytes from 10.1.0.3: icmp_seq=1 ttl=254 time=2.28 ms
80 bytes from 10.1.0.3: icmp_seq=2 ttl=254 time=1.01 ms
80 bytes from 10.1.0.3: icmp_seq=3 ttl=254 time=0.819 ms

--- 10.1.0.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 4ms
rtt min/avg/max/mdev = 0.819/1.366/2.275/0.646 ms, ipg/ewma 2.243/1.954 ms
BORDER#ping 10.1.0.4 source loopback 0 repeat 3
PING 10.1.0.4 (10.1.0.4) from 10.1.0.5 : 72(100) bytes of data.
80 bytes from 10.1.0.4: icmp_seq=1 ttl=63 time=0.755 ms
80 bytes from 10.1.0.4: icmp_seq=2 ttl=63 time=0.402 ms
80 bytes from 10.1.0.4: icmp_seq=3 ttl=63 time=0.342 ms

--- 10.1.0.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2ms
rtt min/avg/max/mdev = 0.342/0.499/0.755/0.182 ms, ipg/ewma 1.001/0.664 ms

```
</details>

## 5. Особенности
У данной схемы имеются следующие особенности:
- SPINE находятся в одной AS для предотвращения path-hunting.
- Из-за отсутствия связности между SPINE и того, что они находятся в одной AS, они не имеют информации о префиксах друг друга. Но ввиду того, что мы строим Underlay для VxLAN/EVPN-фабрики нас интересует только распространение адресов VTEP (Loopback LEAF).
- Схема так же "ломается" при двойном отказе.
<img width="936" height="711" alt="image" src="https://github.com/user-attachments/assets/24a77a39-0796-45a3-8e40-37485dabfb3c" />
