# Лабораторная работа №5 "VxLAN L3VNI"

## Задание:
1. [Подготовка стенда](#1-подготовка-стенда)
2. [Разработка адресного плана](#2-разработка-адресного-плана)
3. [Настройка eBGP для Underlay и EVPN](#3-настройка-bgp-для-underlay-и-evpn)
4. [Проверка связности](#4-проверка-связности)
5. [Настройка для асимметричного IRB](#5-настройка-для-асимметричного-irb)
6. [Проверка работы асимметричного IRB](#6-проверка-работы-асимметричного-irb)
7. [Настройка для имметричного IRB](#5-настройка-для-симметричного-irb)
8. [Проверка работы симметричного IRB](#6-проверка-работы-симметричного-irb)

## 1. Подготовка стенда
В качестве платформы для организации стенда был выбран PNETlab, развернутый на WSL, с использованием образов Arista cEOS и alpine. Интересно было развернуть стенд с помощью альтернативного инструмента, а так же в PNETlab удобнее собирать и анализировать трафик, проходящий через устройства. На стенде дополнительно хотелось реализовать следующие моменты:
* настройка в Linux VRF и агрегации (SRV3)
* настройка паре LEAF2/LEAF3 multihoming для сервера SRV3

Получившийся стенд выглядит следующим образом:
<img width="1360" height="411" alt="image" src="https://github.com/user-attachments/assets/dac9c80b-7caa-49be-bb58-917d541c05ff" />

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
| LEAF4  | 10.1.0.6/32 |

Транспортные подсети
| Link           | Network      |
| -------------- | ------------ |
| SPINE1 — LEAF1 | 10.1.2.0/31  |
| SPINE1 — LEAF2 | 10.1.2.2/31  |
| SPINE1 — LEAF3 | 10.1.2.4/31  |
| SPINE2 — LEAF1 | 10.1.2.6/31  |
| SPINE2 — LEAF2 | 10.1.2.8/31  |
| SPINE2 — LEAF3 | 10.1.2.10/31 |
| SPINE1 — LEAF4 | 10.1.2.12/31 |
| SPINE2 — LEAF4 | 10.1.2.14/31 |

Ключевые моменты настройки:
<details>
<summary>Контекст: Процесс BGP SPINE</summary>

```eos
route-map RM_REDISTRIBUTE-Lo0 permit 10            # route-map для редистрибьюции
   match interface Loopback0                       # выбираем только интерфейс Looback 0
   set origin igp                                  # устанавливаем origin в igp
!
peer-filter LEAFS-AS-FILTER                        # peer-filter для фильтрации соседств с LEAF
   10 match as-range 65001-65004 result accept     # принимаем только BGP-соседей из AS 65001-65004
   
router bgp 65000                                                                        # Процесс BGP в AS 65000
   router-id 10.1.0.1                                                                   # Задаем Router ID
   maximum-paths 4                                                                      # количество маршрутов для ECMP
   bgp listen range 10.1.0.0/24 peer-group LEAFS-EVPN peer-filter LEAFS-AS-FILTER       # Принимаем соседей EVPN с адресами из 10.1.0.0/24 и AS 65001-65004
   bgp listen range 10.1.2.0/23 peer-group LEAFS-UNDERLAY peer-filter LEAFS-AS-FILTER   # Принимаем соседей UNDERLAY с адресами из 10.1.2.0/23 и AS 65001-65004
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
<summary>Контекст: Процесс BGP LEAF/summary>

```eos
route-map RM_REDISTRIBUTE-Lo0 permit 10
   match interface Loopback0
   set community 65001:1
   set origin igp

router bgp 65001
   router-id 10.1.0.3
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
   neighbor 10.1.2.0 peer group SPINE-UNDERLAY
   neighbor 10.1.2.6 peer group SPINE-UNDERLAY
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

<details>
<summary>Настройка SRV1/summary>

```eos
SRV1:/# ip a
106: eth1@if105: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP qlen 1000
    link/ether 02:00:00:00:00:01 brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.1/24 scope global eth1
       valid_lft forever preferred_lft forever
SRV1:/# ip route
default via 10.10.10.254 dev eth1 
10.10.10.0/24 dev eth1 scope link  src 10.10.10.1 
```
</details>

<details>
<summary>Настройкb SRV2/summary>

```eos
SRV1:/# ip addr show dev eth1
106: eth1@if105: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP qlen 1000
    link/ether 02:00:00:00:00:01 brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.1/24 scope global eth1
       valid_lft forever preferred_lft forever
SRV1:/# ip route
default via 10.10.10.254 dev eth1
```
</details>

<details>
<summary>Настройкb SRV2/summary>

```eos
SRV2:/# ip addr show dev eth1
99: eth1@if98: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP qlen 1000
    link/ether 02:00:00:00:00:02 brd ff:ff:ff:ff:ff:ff
    inet 20.20.20.1/24 scope global eth1
       valid_lft forever preferred_lft forever
SRV2:/# ip route
default via 20.20.20.254 dev eth1
```
</details>

<details>
<summary>Настройкb SRV3/summary>

```eos
SRV3:/# ip a
5: bond0: <BROADCAST,MULTICAST,UP,LOWER_UP400> mtu 1500 qdisc noqueue state UP qlen 1000
    link/ether 50:00:00:09:00:01 brd ff:ff:ff:ff:ff:ff
6: VRF1: <NOARP,UP,LOWER_UP400> mtu 65536 qdisc noqueue state UP qlen 1000
    link/ether be:29:ea:10:00:c9 brd ff:ff:ff:ff:ff:ff
7: VRF2: <NOARP,UP,LOWER_UP400> mtu 65536 qdisc noqueue state UP qlen 1000
    link/ether 1a:31:9b:3a:0e:3e brd ff:ff:ff:ff:ff:ff
8: bond0.10@bond0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master VRF1 state UP qlen 1000
    link/ether 50:00:00:09:00:01 brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.3/24 scope global bond0.10
       valid_lft forever preferred_lft forever
9: bond0.20@bond0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master VRF2 state UP qlen 1000
    link/ether 50:00:00:09:00:01 brd ff:ff:ff:ff:ff:ff
    inet 20.20.20.3/24 scope global bond0.20
       valid_lft forever preferred_lft forever
114: eth1@if113: <BROADCAST,MULTICAST,UP,LOWER_UP800,M-DOWN> mtu 1500 qdisc noqueue master bond0 state UP qlen 1000
    link/ether 02:00:00:00:01:03 brd ff:ff:ff:ff:ff:ff
116: eth2@if115: <BROADCAST,MULTICAST,UP,LOWER_UP800,M-DOWN> mtu 1500 qdisc noqueue master bond0 state UP qlen 1000
    link/ether 02:00:00:00:02:03 brd ff:ff:ff:ff:ff:ff
SRV3:/# ip route show table 10
default via 10.10.10.254 dev bond0.10 
broadcast 10.10.10.0 dev bond0.10 scope link  src 10.10.10.3 
10.10.10.0/24 dev bond0.10 scope link  src 10.10.10.3 
local 10.10.10.3 dev bond0.10 scope host  src 10.10.10.3 
broadcast 10.10.10.255 dev bond0.10 scope link  src 10.10.10.3 
SRV3:/# ip route show table 20
default via 20.20.20.254 dev bond0.20 
broadcast 20.20.20.0 dev bond0.20 scope link  src 20.20.20.3 
20.20.20.0/24 dev bond0.20 scope link  src 20.20.20.3 
local 20.20.20.3 dev bond0.20 scope host  src 20.20.20.3 
broadcast 20.20.20.255 dev bond0.20 scope link  src 20.20.20.3 
```
</details>

<details>
<summary>Настройкb SRV4/summary>

```eos
SRV4:/# ip addr show dev eth1
135: eth1@if134: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP qlen 1000
    link/ether 02:00:00:00:00:04 brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.4/24 scope global eth1
       valid_lft forever preferred_lft forever
SRV4:/# ip route
default via 10.10.10.254 dev eth1 
10.10.10.0/24 dev eth1 scope link  src 10.10.10.4 
```
</details>

<details>
<summary>Настройкb SRV5/summary>

```eos
SRV5:/# ip addr show dev eth1
141: eth1@if140: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP qlen 1000
    link/ether 02:00:00:00:00:05 brd ff:ff:ff:ff:ff:ff
    inet 20.20.20.5/24 scope global eth1
       valid_lft forever preferred_lft forever
SRV5:/# ip route
default via 20.20.20.254 dev eth1 
20.20.20.0/24 dev eth1 scope link  src 20.20.20.5 
```
</details>

## 4. Проверка связности
<details>
<summary>LEAF1 / show bgp summary</summary>
  
```eos
LEAF1# show bgp summary 
BGP summary information for VRF default
Router identifier 10.1.0.3, local AS number 65001
Neighbor          AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
-------- ----------- ------------- ----------------------- -------------- ---------- ----------
10.1.0.1       65000 Established   L2VPN EVPN              Negotiated              7          7
10.1.0.2       65000 Established   L2VPN EVPN              Negotiated              7          7
10.1.2.0       65000 Established   IPv4 Unicast            Negotiated              4          4
10.1.2.6       65000 Established   IPv4 Unicast            Negotiated              4          4
```
</details>

<details>
<summary> LEAF1 / show ip bgp</summary>
  
```eos
LEAF1#show ip bgp
BGP routing table information for VRF default
Router identifier 10.1.0.3, local AS number 65001
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast
                    % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >      10.1.0.1/32            10.1.2.0              0       -          100     0       65000 i
 * >      10.1.0.2/32            10.1.2.6              0       -          100     0       65000 i
 * >      10.1.0.3/32            -                     -       -          -       0       i
 * >Ec    10.1.0.4/32            10.1.2.6              0       -          100     0       65000 65002 i
 *  ec    10.1.0.4/32            10.1.2.0              0       -          100     0       65000 65002 i
 * >Ec    10.1.0.5/32            10.1.2.6              0       -          100     0       65000 65003 i
 *  ec    10.1.0.5/32            10.1.2.0              0       -          100     0       65000 65003 i
 * >Ec    10.1.0.6/32            10.1.2.6              0       -          100     0       65000 65004 i
 *  ec    10.1.0.6/32            10.1.2.0              0       -          100     0       65000 65004 i
```
</details>

<details>
<summary>LEAF1 / ping до Lo0 LEAF2-4</summary>
  
```eos
LEAF1#ping 10.1.0.4 source loopback 0 repeat 3
PING 10.1.0.4 (10.1.0.4) from 10.1.0.3 : 72(100) bytes of data.
80 bytes from 10.1.0.4: icmp_seq=1 ttl=63 time=6.42 ms
80 bytes from 10.1.0.4: icmp_seq=2 ttl=63 time=3.89 ms
80 bytes from 10.1.0.4: icmp_seq=3 ttl=63 time=4.30 ms

--- 10.1.0.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 13ms
rtt min/avg/max/mdev = 3.897/4.875/6.426/1.109 ms, ipg/ewma 6.752/5.884 ms
LEAF1#ping 10.1.0.5 source loopback 0 repeat 3
PING 10.1.0.5 (10.1.0.5) from 10.1.0.3 : 72(100) bytes of data.
80 bytes from 10.1.0.5: icmp_seq=1 ttl=63 time=4.51 ms
80 bytes from 10.1.0.5: icmp_seq=2 ttl=63 time=4.50 ms
80 bytes from 10.1.0.5: icmp_seq=3 ttl=63 time=3.84 ms

--- 10.1.0.5 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 11ms
rtt min/avg/max/mdev = 3.849/4.287/4.513/0.314 ms, ipg/ewma 5.755/4.428 ms
LEAF1#ping 10.1.0.6 source loopback 0 repeat 3
PING 10.1.0.6 (10.1.0.6) from 10.1.0.3 : 72(100) bytes of data.
80 bytes from 10.1.0.6: icmp_seq=1 ttl=63 time=5.63 ms
80 bytes from 10.1.0.6: icmp_seq=2 ttl=63 time=5.78 ms
80 bytes from 10.1.0.6: icmp_seq=3 ttl=63 time=4.33 ms

--- 10.1.0.6 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 13ms
rtt min/avg/max/mdev = 4.332/5.251/5.783/0.652 ms, ipg/ewma 6.906/5.490 ms```
</details>

<details>
<summary>LEAF2 / ping до Lo0 LEAF1,LEAF3-4</summary>
  
```eos
LEAF2#ping 10.1.0.3 source loopback 0 repeat 3
PING 10.1.0.3 (10.1.0.3) from 10.1.0.4 : 72(100) bytes of data.
80 bytes from 10.1.0.3: icmp_seq=1 ttl=63 time=6.43 ms
80 bytes from 10.1.0.3: icmp_seq=2 ttl=63 time=4.14 ms
80 bytes from 10.1.0.3: icmp_seq=3 ttl=63 time=4.64 ms

--- 10.1.0.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 13ms
rtt min/avg/max/mdev = 4.148/5.077/6.439/0.987 ms, ipg/ewma 6.730/5.964 ms
LEAF2#ping 10.1.0.5 source loopback 0 repeat 3
PING 10.1.0.5 (10.1.0.5) from 10.1.0.4 : 72(100) bytes of data.
80 bytes from 10.1.0.5: icmp_seq=1 ttl=63 time=5.31 ms
80 bytes from 10.1.0.5: icmp_seq=2 ttl=63 time=3.87 ms
80 bytes from 10.1.0.5: icmp_seq=3 ttl=63 time=4.41 ms

--- 10.1.0.5 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 11ms
rtt min/avg/max/mdev = 3.878/4.532/5.310/0.596 ms, ipg/ewma 5.794/5.040 ms
LEAF2#ping 10.1.0.6 source loopback 0 repeat 3
PING 10.1.0.6 (10.1.0.6) from 10.1.0.4 : 72(100) bytes of data.
80 bytes from 10.1.0.6: icmp_seq=1 ttl=63 time=7.14 ms
80 bytes from 10.1.0.6: icmp_seq=2 ttl=63 time=4.29 ms
80 bytes from 10.1.0.6: icmp_seq=3 ttl=63 time=4.30 ms

--- 10.1.0.6 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 15ms
rtt min/avg/max/mdev = 4.298/5.251/7.149/1.342 ms, ipg/ewma 7.636/6.481 ms
```
</details>

<details>
<summary>LEAF3 / ping до Lo0 LEAF1-2, LEAF4</summary>
  
```eos
LEAF3#ping 10.1.0.3 source loopback 0 repeat 3
PING 10.1.0.3 (10.1.0.3) from 10.1.0.5 : 72(100) bytes of data.
80 bytes from 10.1.0.3: icmp_seq=1 ttl=63 time=5.02 ms
80 bytes from 10.1.0.3: icmp_seq=2 ttl=63 time=3.87 ms
80 bytes from 10.1.0.3: icmp_seq=3 ttl=63 time=7.17 ms

--- 10.1.0.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 11ms
rtt min/avg/max/mdev = 3.879/5.360/7.174/1.365 ms, ipg/ewma 5.532/5.169 ms
LEAF3#ping 10.1.0.4 source loopback 0 repeat 3
PING 10.1.0.4 (10.1.0.4) from 10.1.0.5 : 72(100) bytes of data.
80 bytes from 10.1.0.4: icmp_seq=1 ttl=63 time=6.00 ms
80 bytes from 10.1.0.4: icmp_seq=2 ttl=63 time=6.14 ms
80 bytes from 10.1.0.4: icmp_seq=3 ttl=63 time=4.42 ms

--- 10.1.0.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 14ms
rtt min/avg/max/mdev = 4.420/5.522/6.144/0.781 ms, ipg/ewma 7.215/5.819 ms
LEAF3#ping 10.1.0.6 source loopback 0 repeat 3
PING 10.1.0.6 (10.1.0.6) from 10.1.0.5 : 72(100) bytes of data.
80 bytes from 10.1.0.6: icmp_seq=1 ttl=63 time=4.99 ms
80 bytes from 10.1.0.6: icmp_seq=2 ttl=63 time=4.54 ms
80 bytes from 10.1.0.6: icmp_seq=3 ttl=63 time=5.19 ms

--- 10.1.0.6 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 11ms
rtt min/avg/max/mdev = 4.543/4.910/5.197/0.272 ms, ipg/ewma 5.736/4.967 ms
```
</details>

<details>
<summary>LEAF4 / ping до Lo0 LEAF1-3</summary>
  
```eos
LEAF4#ping 10.1.0.3 source loopback 0 repeat 3
PING 10.1.0.3 (10.1.0.3) from 10.1.0.6 : 72(100) bytes of data.
80 bytes from 10.1.0.3: icmp_seq=1 ttl=63 time=6.21 ms
80 bytes from 10.1.0.3: icmp_seq=2 ttl=63 time=4.43 ms
80 bytes from 10.1.0.3: icmp_seq=3 ttl=63 time=4.32 ms

--- 10.1.0.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 13ms
rtt min/avg/max/mdev = 4.329/4.992/6.215/0.867 ms, ipg/ewma 6.944/5.784 ms
LEAF4#ping 10.1.0.4 source loopback 0 repeat 3
PING 10.1.0.4 (10.1.0.4) from 10.1.0.6 : 72(100) bytes of data.
80 bytes from 10.1.0.4: icmp_seq=1 ttl=63 time=7.80 ms
80 bytes from 10.1.0.4: icmp_seq=2 ttl=63 time=5.32 ms
80 bytes from 10.1.0.4: icmp_seq=3 ttl=63 time=4.94 ms

--- 10.1.0.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 15ms
rtt min/avg/max/mdev = 4.945/6.025/7.805/1.269 ms, ipg/ewma 7.879/7.176 ms
LEAF4#ping 10.1.0.5 source loopback 0 repeat 3
PING 10.1.0.5 (10.1.0.5) from 10.1.0.6 : 72(100) bytes of data.
80 bytes from 10.1.0.5: icmp_seq=1 ttl=63 time=5.19 ms
80 bytes from 10.1.0.5: icmp_seq=2 ttl=63 time=4.75 ms
80 bytes from 10.1.0.5: icmp_seq=3 ttl=63 time=4.35 ms

--- 10.1.0.5 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 11ms
rtt min/avg/max/mdev = 4.356/4.767/5.196/0.347 ms, ipg/ewma 5.873/5.042 ms
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

## 6. Итоговые конфигурации устройств фабрики
[Конфигурация Spine1](./configs/spine1.conf)<br>
[Конфигурация Spine2](./configs/spine2.conf)<br>
[Конфигурация Leaf1](./configs/leaf1.conf)<br>
[Конфигурация Leaf2](./configs/leaf2.conf)<br>
[Конфигурация Leaf3](./configs/border.conf)<br>
