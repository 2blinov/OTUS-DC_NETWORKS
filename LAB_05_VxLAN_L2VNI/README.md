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
<details>
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

<details>
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

## 5. Настройка VxLAN L2VNI
Пояснения касательно настройки
<details>
<summary>Контекст: LEAF1/NX</summary>

```
vlan 10               # Для VLAN 10
  name VLAN10
  vn-segment 10010    # Настраиваем L2VNI 10010
vlan 20               # Для VLAN 20
  name VLAN20
  vn-segment 10020    # Настраиваем L2VNI 10020

interface nve1                           # Настройки интерфейса NVE
  no shutdown
  host-reachability protocol bgp         # Для поиска удалённых MAC/IP-хостов внутри VXLAN использовать BGP EVPN как control-plane.
  source-interface loopback0             # Туннели строим с Lo0
  member vni 10010                       
    ingress-replication protocol bgp     # Для BUM-трафика внутри VNI использовать ingress replication, а список удалённых VTEP получать через BGP EVPN.
  member vni 10020
    ingress-replication protocol bgp

evpn                                     # Секция EVPN
  vni 10010 l2                           # Для каждого L2 VNI
    rd 10.1.0.3:10010                    # Настраиваем RD (будет отдаваться вместе с маршрутами соседям через расширенные community, для того, чтобы различать EVPN-маршруты)
    route-target import 10010:10010      # Импортируем в VNI/EVPN инстанс EVPN-маршруты с RT 10010:10010
    route-target export 10010:10010      # При экспорте EVPN-маршрута для VNI 10010, добавляет к нему RT 10010:10010
  vni 10020 l2
    rd 10.1.0.3:10020
    route-target import 10020:10020
    route-target export 10020:10020
```
</details>

<details>
<summary>Контекст: LEAF2/Arista</summary>

```
vlan 10
   name VLAN10
!
vlan 20
   name VLAN20
!
interface Vxlan1                         # Настройки интерфейса NVE
   vxlan source-interface Loopback0      # Туннели строим с Lo0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010               # Привязка VNI к VLAN ID
   vxlan vlan 20 vni 10020               # Привязка VNI к VLAN ID

router bgp 65002                         
   vlan 10                               # Для VLAN
      rd 10.1.0.4:10010                  # Настраиваем RD
      route-target both 10010:10010      # Настраиваем RT на импорт/экспорт
      redistribute learned               # MAC-адреса, которые коммутатор выучил в MAC address table, и распространяем через BGP EVPN.
   !
   vlan 20
      rd 10.1.0.4:10020
      route-target both 10020:10020
      redistribute learned
```
</details>

## 6. Проверка работы L2VNI

<details>
<summary>Ping Host1 -> Host2, Host3, FGT в рамках одного VLAN</summary>

```
root@frr1:~# ip vrf exec VRF1 ping 10.10.10.1 -c 3
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.1: icmp_seq=1 ttl=255 time=11.7 ms
64 bytes from 10.10.10.1: icmp_seq=2 ttl=255 time=4.65 ms
64 bytes from 10.10.10.1: icmp_seq=3 ttl=255 time=2.48 ms

--- 10.10.10.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 2.483/6.273/11.682/3.926 ms
root@frr1:~# ip vrf exec VRF1 ping 10.10.10.11 -c 3
PING 10.10.10.11 (10.10.10.11) 56(84) bytes of data.
64 bytes from 10.10.10.11: icmp_seq=1 ttl=64 time=2.62 ms
64 bytes from 10.10.10.11: icmp_seq=2 ttl=64 time=2.25 ms
64 bytes from 10.10.10.11: icmp_seq=3 ttl=64 time=5.03 ms

--- 10.10.10.11 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 2.247/3.297/5.026/1.231 ms
root@frr1:~# ip vrf exec VRF2 ping 20.20.20.1 -c 3
PING 20.20.20.1 (20.20.20.1) 56(84) bytes of data.
64 bytes from 20.20.20.1: icmp_seq=1 ttl=255 time=7.17 ms
64 bytes from 20.20.20.1: icmp_seq=2 ttl=255 time=3.24 ms
64 bytes from 20.20.20.1: icmp_seq=3 ttl=255 time=2.88 ms

--- 20.20.20.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 2.876/4.428/7.167/1.942 ms
root@frr1:~# ip vrf exec VRF2 ping 20.20.20.11 -c 3
PING 20.20.20.11 (20.20.20.11) 56(84) bytes of data.
64 bytes from 20.20.20.11: icmp_seq=1 ttl=64 time=3.41 ms
64 bytes from 20.20.20.11: icmp_seq=2 ttl=64 time=2.27 ms
64 bytes from 20.20.20.11: icmp_seq=3 ttl=64 time=2.79 ms

--- 20.20.20.11 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 2.272/2.825/3.413/0.466 ms
```
</details>

<details>
<summary>Host1, отсутствие связности между двумя VLAN</summary>

```
root@frr1:~# ip vrf exec VRF1 ping 20.20.20.1 -c 3
PING 20.20.20.1 (20.20.20.1) 56(84) bytes of data.

--- 20.20.20.1 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2129ms

root@frr1:~# ip vrf exec VRF2 ping 10.10.10.1 -c 3
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.

--- 10.10.10.1 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2036ms
```
</details>

<details>
<summary>LEAF1 / route-type 3/ sh bgp l2vpn evpn route-type 3</summary>

```
LEAF1# sh bgp l2vpn evpn route-type 3
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 10.1.0.3:10010    (L2VNI 10010)
BGP routing table entry for [3]:[0]:[32]:[10.1.0.3]/88, version 115
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn
Multipath: eBGP

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop, is extd
  AS-Path: NONE, path locally originated
    10.1.0.3 (metric 0) from 0.0.0.0 (10.1.0.3)
      Origin IGP, MED not set, localpref 100, weight 32768
      Extcommunity: RT:10010:10010 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10010, Tunnel Id: 10.1.0.3

  Path-id 1 advertised to peers:
    10.1.0.1           10.1.0.2       
BGP routing table entry for [3]:[0]:[32]:[10.1.0.4]/88, version 116
Paths: (1 available, best #1)
Flags: (0x000012) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW
Multipath: eBGP

  Advertised path-id 1
  Path type: external, path is valid, is best path, no labeled nexthop
             Imported from 10.1.0.4:10010:[3]:[0]:[32]:[10.1.0.4]/88 
  AS-Path: 65000 65002 , path sourced external to AS
    10.1.0.4 (metric 0) from 10.1.0.1 (10.1.0.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:10010:10010 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10010, Tunnel Id: 10.1.0.4

  Path-id 1 not advertised to any peer
BGP routing table entry for [3]:[0]:[32]:[10.1.0.5]/88, version 117
Paths: (1 available, best #1)
Flags: (0x000012) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW
Multipath: eBGP

  Advertised path-id 1
  Path type: external, path is valid, is best path, no labeled nexthop
             Imported from 10.1.0.5:10010:[3]:[0]:[32]:[10.1.0.5]/88 
  AS-Path: 65000 65003 , path sourced external to AS
    10.1.0.5 (metric 0) from 10.1.0.1 (10.1.0.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:10010:10010 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10010, Tunnel Id: 10.1.0.5

  Path-id 1 not advertised to any peer

Route Distinguisher: 10.1.0.3:10020    (L2VNI 10020)
BGP routing table entry for [3]:[0]:[32]:[10.1.0.3]/88, version 119
Paths: (1 available, best #1)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn
Multipath: eBGP

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop, is extd
  AS-Path: NONE, path locally originated
    10.1.0.3 (metric 0) from 0.0.0.0 (10.1.0.3)
      Origin IGP, MED not set, localpref 100, weight 32768
      Extcommunity: RT:10020:10020 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10020, Tunnel Id: 10.1.0.3

  Path-id 1 advertised to peers:
    10.1.0.1           10.1.0.2       
BGP routing table entry for [3]:[0]:[32]:[10.1.0.4]/88, version 120
Paths: (1 available, best #1)
Flags: (0x000012) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW
Multipath: eBGP

  Advertised path-id 1
  Path type: external, path is valid, is best path, no labeled nexthop
             Imported from 10.1.0.4:10020:[3]:[0]:[32]:[10.1.0.4]/88 
  AS-Path: 65000 65002 , path sourced external to AS
    10.1.0.4 (metric 0) from 10.1.0.1 (10.1.0.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:10020:10020 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10020, Tunnel Id: 10.1.0.4

  Path-id 1 not advertised to any peer
BGP routing table entry for [3]:[0]:[32]:[10.1.0.5]/88, version 121
Paths: (1 available, best #1)
Flags: (0x000012) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW
Multipath: eBGP

  Advertised path-id 1
  Path type: external, path is valid, is best path, no labeled nexthop
             Imported from 10.1.0.5:10020:[3]:[0]:[32]:[10.1.0.5]/88 
  AS-Path: 65000 65003 , path sourced external to AS
    10.1.0.5 (metric 0) from 10.1.0.1 (10.1.0.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:10020:10020 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10020, Tunnel Id: 10.1.0.5

  Path-id 1 not advertised to any peer

Route Distinguisher: 10.1.0.4:10010
BGP routing table entry for [3]:[0]:[32]:[10.1.0.4]/88, version 122
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW
Multipath: eBGP

  Path type: external, path is valid, not best reason: newer EBGP path, no labeled nexthop, is extd
  AS-Path: 65000 65002 , path sourced external to AS
    10.1.0.4 (metric 0) from 10.1.0.2 (10.1.0.2)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:10010:10010 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10010, Tunnel Id: 10.1.0.4

  Advertised path-id 1
  Path type: external, path is valid, is best path, no labeled nexthop, is extd
             Imported to 1 destination(s)
             Imported paths list: L2-10010
  AS-Path: 65000 65002 , path sourced external to AS
    10.1.0.4 (metric 0) from 10.1.0.1 (10.1.0.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:10010:10010 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10010, Tunnel Id: 10.1.0.4

  Path-id 1 not advertised to any peer

Route Distinguisher: 10.1.0.4:10020
BGP routing table entry for [3]:[0]:[32]:[10.1.0.4]/88, version 123
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW
Multipath: eBGP

  Path type: external, path is valid, not best reason: newer EBGP path, no labeled nexthop, is extd
  AS-Path: 65000 65002 , path sourced external to AS
    10.1.0.4 (metric 0) from 10.1.0.2 (10.1.0.2)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:10020:10020 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10020, Tunnel Id: 10.1.0.4

  Advertised path-id 1
  Path type: external, path is valid, is best path, no labeled nexthop, is extd
             Imported to 1 destination(s)
             Imported paths list: L2-10020
  AS-Path: 65000 65002 , path sourced external to AS
    10.1.0.4 (metric 0) from 10.1.0.1 (10.1.0.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:10020:10020 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10020, Tunnel Id: 10.1.0.4

  Path-id 1 not advertised to any peer

Route Distinguisher: 10.1.0.5:10010
BGP routing table entry for [3]:[0]:[32]:[10.1.0.5]/88, version 124
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW
Multipath: eBGP

  Path type: external, path is valid, not best reason: newer EBGP path, no labeled nexthop, is extd
  AS-Path: 65000 65003 , path sourced external to AS
    10.1.0.5 (metric 0) from 10.1.0.2 (10.1.0.2)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:10010:10010 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10010, Tunnel Id: 10.1.0.5

  Advertised path-id 1
  Path type: external, path is valid, is best path, no labeled nexthop, is extd
             Imported to 1 destination(s)
             Imported paths list: L2-10010
  AS-Path: 65000 65003 , path sourced external to AS
    10.1.0.5 (metric 0) from 10.1.0.1 (10.1.0.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:10010:10010 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10010, Tunnel Id: 10.1.0.5

  Path-id 1 not advertised to any peer

Route Distinguisher: 10.1.0.5:10020
BGP routing table entry for [3]:[0]:[32]:[10.1.0.5]/88, version 125
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW
Multipath: eBGP

  Path type: external, path is valid, not best reason: newer EBGP path, no labeled nexthop, is extd
  AS-Path: 65000 65003 , path sourced external to AS
    10.1.0.5 (metric 0) from 10.1.0.2 (10.1.0.2)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:10020:10020 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10020, Tunnel Id: 10.1.0.5

  Advertised path-id 1
  Path type: external, path is valid, is best path, no labeled nexthop, is extd
             Imported to 1 destination(s)
             Imported paths list: L2-10020
  AS-Path: 65000 65003 , path sourced external to AS
    10.1.0.5 (metric 0) from 10.1.0.1 (10.1.0.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Extcommunity: RT:10020:10020 ENCAP:8
      PMSI Tunnel Attribute:
        flags: 0x00, Tunnel type: Ingress Replication
        Label: 10020, Tunnel Id: 10.1.0.5

  Path-id 1 not advertised to any peer
```
</details>

<details>
<summary>LEAF1 / route-type 2 / sh bgp l2vpn evpn route-type 2</summary>

```
LEAF1# sh bgp l2vpn evpn route-type 2
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 10.1.0.3:10010    (L2VNI 10010)
BGP routing table entry for [2]:[0]:[0]:[48]:[aac1.ab5d.072e]:[0]:[0.0.0.0]/216, version 167
Paths: (1 available, best #1)
Flags: (0x000102) (high32 00000000) on xmit-list, is not in l2rib/evpn
Multipath: eBGP

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    10.1.0.3 (metric 0) from 0.0.0.0 (10.1.0.3)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 10010
      Extcommunity: RT:10010:10010 ENCAP:8

  Path-id 1 advertised to peers:
    10.1.0.1           10.1.0.2       
BGP routing table entry for [2]:[0]:[0]:[48]:[aac1.abc9.261e]:[0]:[0.0.0.0]/216, version 182
Paths: (1 available, best #1)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW
Multipath: eBGP

  Advertised path-id 1
  Path type: external, path is valid, is best path, no labeled nexthop, in rib
             Imported from 10.1.0.4:10010:[2]:[0]:[0]:[48]:[aac1.abc9.261e]:[0]:[0.0.0.0]/216 
  AS-Path: 65000 65002 , path sourced external to AS
    10.1.0.4 (metric 0) from 10.1.0.1 (10.1.0.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10010
      Extcommunity: RT:10010:10010 ENCAP:8

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[aac1.abdc.3580]:[0]:[0.0.0.0]/216, version 169
Paths: (1 available, best #1)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW
Multipath: eBGP

  Advertised path-id 1
  Path type: external, path is valid, is best path, no labeled nexthop, in rib
             Imported from 10.1.0.5:10010:[2]:[0]:[0]:[48]:[aac1.abdc.3580]:[0]:[0.0.0.0]/216 
  AS-Path: 65000 65003 , path sourced external to AS
    10.1.0.5 (metric 0) from 10.1.0.1 (10.1.0.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10010
      Extcommunity: RT:10010:10010 ENCAP:8

  Path-id 1 not advertised to any peer

Route Distinguisher: 10.1.0.3:10020    (L2VNI 10020)
BGP routing table entry for [2]:[0]:[0]:[48]:[aac1.ab5d.072e]:[0]:[0.0.0.0]/216, version 163
Paths: (1 available, best #1)
Flags: (0x000102) (high32 00000000) on xmit-list, is not in l2rib/evpn
Multipath: eBGP

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    10.1.0.3 (metric 0) from 0.0.0.0 (10.1.0.3)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 10020
      Extcommunity: RT:10020:10020 ENCAP:8

  Path-id 1 advertised to peers:
    10.1.0.1           10.1.0.2       
BGP routing table entry for [2]:[0]:[0]:[48]:[aac1.abab.2dd7]:[0]:[0.0.0.0]/216, version 186
Paths: (1 available, best #1)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW
Multipath: eBGP

  Advertised path-id 1
  Path type: external, path is valid, is best path, no labeled nexthop, in rib
             Imported from 10.1.0.4:10020:[2]:[0]:[0]:[48]:[aac1.abab.2dd7]:[0]:[0.0.0.0]/216 
  AS-Path: 65000 65002 , path sourced external to AS
    10.1.0.4 (metric 0) from 10.1.0.1 (10.1.0.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10020
      Extcommunity: RT:10020:10020 ENCAP:8

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[aac1.abdc.3580]:[0]:[0.0.0.0]/216, version 184
Paths: (1 available, best #1)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW
Multipath: eBGP

  Advertised path-id 1
  Path type: external, path is valid, is best path, no labeled nexthop, in rib
             Imported from 10.1.0.5:10020:[2]:[0]:[0]:[48]:[aac1.abdc.3580]:[0]:[0.0.0.0]/216 
  AS-Path: 65000 65003 , path sourced external to AS
    10.1.0.5 (metric 0) from 10.1.0.1 (10.1.0.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10020
      Extcommunity: RT:10020:10020 ENCAP:8

  Path-id 1 not advertised to any peer

Route Distinguisher: 10.1.0.4:10010
BGP routing table entry for [2]:[0]:[0]:[48]:[aac1.abc9.261e]:[0]:[0.0.0.0]/216, version 181
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW
Multipath: eBGP

  Path type: external, path is valid, not best reason: Router Id, no labeled nexthop
  AS-Path: 65000 65002 , path sourced external to AS
    10.1.0.4 (metric 0) from 10.1.0.2 (10.1.0.2)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10010
      Extcommunity: RT:10010:10010 ENCAP:8

  Advertised path-id 1
  Path type: external, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10010
  AS-Path: 65000 65002 , path sourced external to AS
    10.1.0.4 (metric 0) from 10.1.0.1 (10.1.0.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10010
      Extcommunity: RT:10010:10010 ENCAP:8

  Path-id 1 not advertised to any peer

Route Distinguisher: 10.1.0.4:10020
BGP routing table entry for [2]:[0]:[0]:[48]:[aac1.abab.2dd7]:[0]:[0.0.0.0]/216, version 185
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW
Multipath: eBGP

  Path type: external, path is valid, not best reason: Router Id, no labeled nexthop
  AS-Path: 65000 65002 , path sourced external to AS
    10.1.0.4 (metric 0) from 10.1.0.2 (10.1.0.2)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10020
      Extcommunity: RT:10020:10020 ENCAP:8

  Advertised path-id 1
  Path type: external, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10020
  AS-Path: 65000 65002 , path sourced external to AS
    10.1.0.4 (metric 0) from 10.1.0.1 (10.1.0.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10020
      Extcommunity: RT:10020:10020 ENCAP:8

  Path-id 1 not advertised to any peer

Route Distinguisher: 10.1.0.5:10010
BGP routing table entry for [2]:[0]:[0]:[48]:[aac1.abdc.3580]:[0]:[0.0.0.0]/216, version 168
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW
Multipath: eBGP

  Path type: external, path is valid, not best reason: Router Id, no labeled nexthop
  AS-Path: 65000 65003 , path sourced external to AS
    10.1.0.5 (metric 0) from 10.1.0.2 (10.1.0.2)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10010
      Extcommunity: RT:10010:10010 ENCAP:8

  Advertised path-id 1
  Path type: external, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10010
  AS-Path: 65000 65003 , path sourced external to AS
    10.1.0.5 (metric 0) from 10.1.0.1 (10.1.0.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10010
      Extcommunity: RT:10010:10010 ENCAP:8

  Path-id 1 not advertised to any peer

Route Distinguisher: 10.1.0.5:10020
BGP routing table entry for [2]:[0]:[0]:[48]:[aac1.abdc.3580]:[0]:[0.0.0.0]/216, version 183
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not in HW
Multipath: eBGP

  Path type: external, path is valid, not best reason: Router Id, no labeled nexthop
  AS-Path: 65000 65003 , path sourced external to AS
    10.1.0.5 (metric 0) from 10.1.0.2 (10.1.0.2)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10020
      Extcommunity: RT:10020:10020 ENCAP:8

  Advertised path-id 1
  Path type: external, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10020
  AS-Path: 65000 65003 , path sourced external to AS
    10.1.0.5 (metric 0) from 10.1.0.1 (10.1.0.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10020
      Extcommunity: RT:10020:10020 ENCAP:8

  Path-id 1 not advertised to any peer
```
</details>

<details>
<summary>LEAF1 / show mac address-table</summary>

```
LEAF1# show mac address-table 
Legend: 
        * - primary entry, G - Gateway MAC, (R) - Routed MAC, O - Overlay MAC
        age - seconds since last seen,+ - primary entry using vPC Peer-Link,
        (T) - True, (F) - False, C - ControlPlane MAC, ~ - vsan,
        (NA)- Not Applicable A - ESI Active Path, S - ESI Standby Path
        TL - True Learned, PS - Peer Sync, RO - Re-originate 
   VLAN     MAC Address      Type      age     Secure NTFY Ports
---------+-----------------+--------+---------+------+----+------------------
*   10     aac1.ab5d.072e   dynamic  NA         F      F    Eth1/3
C   10     aac1.abc9.261e   dynamic  NA         F      F    nve1(10.1.0.4)
C   10     aac1.abdc.3580   dynamic  NA         F      F    nve1(10.1.0.5)
*   20     aac1.ab5d.072e   dynamic  NA         F      F    Eth1/3
C   20     aac1.abab.2dd7   dynamic  NA         F      F    nve1(10.1.0.4)
C   20     aac1.abdc.3580   dynamic  NA         F      F    nve1(10.1.0.5)
G    -     0cfc.8000.1b08   static   -         F      F    sup-eth1(R)
```
</details>


<details>
<summary>LEAF2 / route-type 3 / sh bgp evpn route-type imet</summary>

```
LEAF2#sh bgp evpn route-type imet 
BGP routing table information for VRF default
Router identifier 10.1.0.4, local AS number 65002
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.1.0.3:10010 imet 10.1.0.3
                                 10.1.0.3              -       100     0       65000 65001 i
 *  ec    RD: 10.1.0.3:10010 imet 10.1.0.3
                                 10.1.0.3              -       100     0       65000 65001 i
 * >Ec    RD: 10.1.0.3:10020 imet 10.1.0.3
                                 10.1.0.3              -       100     0       65000 65001 i
 *  ec    RD: 10.1.0.3:10020 imet 10.1.0.3
                                 10.1.0.3              -       100     0       65000 65001 i
 * >      RD: 10.1.0.4:10010 imet 10.1.0.4
                                 -                     -       -       0       i
 * >      RD: 10.1.0.4:10020 imet 10.1.0.4
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.5:10010 imet 10.1.0.5
                                 10.1.0.5              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.5:10010 imet 10.1.0.5
                                 10.1.0.5              -       100     0       65000 65003 i
 * >Ec    RD: 10.1.0.5:10020 imet 10.1.0.5
                                 10.1.0.5              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.5:10020 imet 10.1.0.5
                                 10.1.0.5              -       100     0       65000 65003 i
```
</details>

<details>
<summary>LEAF2 / route-type 2 / sh bgp evpn route-type mac-ip</summary>

```
LEAF2#sh bgp evpn route-type mac-ip 
BGP routing table information for VRF default
Router identifier 10.1.0.4, local AS number 65002
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.1.0.3:10010 mac-ip aac1.ab5d.072e
                                 10.1.0.3              -       100     0       65000 65001 i
 *  ec    RD: 10.1.0.3:10010 mac-ip aac1.ab5d.072e
                                 10.1.0.3              -       100     0       65000 65001 i
 * >Ec    RD: 10.1.0.3:10020 mac-ip aac1.ab5d.072e
                                 10.1.0.3              -       100     0       65000 65001 i
 *  ec    RD: 10.1.0.3:10020 mac-ip aac1.ab5d.072e
                                 10.1.0.3              -       100     0       65000 65001 i
 * >      RD: 10.1.0.4:10020 mac-ip aac1.abab.2dd7
                                 -                     -       -       0       i
 * >      RD: 10.1.0.4:10010 mac-ip aac1.abc9.261e
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.5:10010 mac-ip aac1.abdc.3580
                                 10.1.0.5              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.5:10010 mac-ip aac1.abdc.3580
                                 10.1.0.5              -       100     0       65000 65003 i
 * >Ec    RD: 10.1.0.5:10020 mac-ip aac1.abdc.3580
                                 10.1.0.5              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.5:10020 mac-ip aac1.abdc.3580
                                 10.1.0.5              -       100     0       65000 65003 i
```
</details>

<details>
<summary>LEAF2 / show mac address-table</summary>

```
LEAF2#show mac address-table 
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  10    aac1.ab5d.072e    DYNAMIC     Vx1        1       0:16:09 ago
  10    aac1.abc9.261e    DYNAMIC     Et3        1       0:05:02 ago
  10    aac1.abdc.3580    DYNAMIC     Vx1        1       0:15:20 ago
  20    aac1.ab5d.072e    DYNAMIC     Vx1        1       0:20:36 ago
  20    aac1.abab.2dd7    DYNAMIC     Et4        1       0:04:57 ago
  20    aac1.abdc.3580    DYNAMIC     Vx1        1       0:04:59 ago
Total Mac Addresses for this criterion: 6

```
</details>
