# Лабораторная работа №5 "VxLAN L3VNI + Multihoming"

## Задание:
1. [Подготовка стенда](#1-подготовка-стенда)
2. [Разработка адресного плана](#2-разработка-адресного-плана)
3. [Настройка eBGP для Underlay и EVPN](#3-настройка-bgp-для-underlay-и-evpn)
4. [Проверка связности](#4-проверка-связности)
5. [Настройки для асимметричного IRB](#5-настройки-для-асимметричного-irb)
6. [Проверка работы асимметричного IRB](#6-проверка-работы-асимметричного-irb)
7. [Настройки для cимметричного IRB](#7-настройки-для-симметричного-irb)
8. [Проверка работы симметричного IRB](#8-проверка-работы-симметричного-irb)
9. [Настройка Multihoming](#9-настройка-multihoming)
10. [Проверка Multihoming](#10-проверка-multihoming)
11. [Отказоустойчивость Multihoming](#11-отказоустойчивость-multihoming)
12. [Конфигурации устройств фабрики](#12-конфигурации-устройств-фабрики)

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
<summary>Контекст: Процесс BGP LEAF</summary>

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
<summary>Настройка SRV1</summary>

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
<summary>Настройкb SRV2</summary>

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
<summary>Настройкb SRV2</summary>

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
<summary>Настройкb SRV3</summary>

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
<summary>Настройкb SRV4</summary>

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
<summary>Настройкb SRV5</summary>

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

## 5. Настройки для асимметричного IRB
<details>
<summary>Контекст: LEAF2/Arista</summary>

```
vlan 10
   name VLAN10
!
vlan 20
   name VLAN20
!
interface Vlan10
   description VLAN10
   ip address virtual 10.10.10.254/24
!
interface Vlan20
   description VLAN20
   ip address virtual 20.20.20.254/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
!
ip virtual-router mac-address 00:00:22:22:33:33
!
ip routing
!
router bgp 65001
   vlan 10
      rd 10.1.0.3:10010
      route-target both 10010:10010
      redistribute learned
   !
   vlan 20
      rd 10.1.0.3:10020
      route-target both 10020:10020
      redistribute learned
```
</details>

## 6. Проверка работы асимметричного IRB
<details>
<summary>LEAF1 / route-type 3 / sh bgp evpn route-type imet</summary>

```
LEAF1#show bgp evpn route-type imet 
BGP routing table information for VRF default
Router identifier 10.1.0.3, local AS number 65001
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.1.0.3:10010 imet 10.1.0.3
                                 -                     -       -       0       i
 * >      RD: 10.1.0.3:10020 imet 10.1.0.3
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.4:10010 imet 10.1.0.4
                                 10.1.0.4              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.4:10010 imet 10.1.0.4
                                 10.1.0.4              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.4:10020 imet 10.1.0.4
                                 10.1.0.4              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.4:10020 imet 10.1.0.4
                                 10.1.0.4              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.5:10010 imet 10.1.0.5
                                 10.1.0.5              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.5:10010 imet 10.1.0.5
                                 10.1.0.5              -       100     0       65000 65003 i
 * >Ec    RD: 10.1.0.5:10020 imet 10.1.0.5
                                 10.1.0.5              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.5:10020 imet 10.1.0.5
                                 10.1.0.5              -       100     0       65000 65003 i
 * >Ec    RD: 10.1.0.6:10010 imet 10.1.0.6
                                 10.1.0.6              -       100     0       65000 65004 i
 *  ec    RD: 10.1.0.6:10010 imet 10.1.0.6
                                 10.1.0.6              -       100     0       65000 65004 i
 * >Ec    RD: 10.1.0.6:10020 imet 10.1.0.6
                                 10.1.0.6              -       100     0       65000 65004 i
 *  ec    RD: 10.1.0.6:10020 imet 10.1.0.6
                                 10.1.0.6              -       100     0       65000 65004 i
```
</details>

<details>
<summary>LEAF1 / route-type 2 / sh bgp evpn route-type mac-ip</summary>

```
LEAF1#show bgp evpn route-type mac-ip 
BGP routing table information for VRF default
Router identifier 10.1.0.3, local AS number 65001
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.1.0.3:10010 mac-ip 0200.0000.0001
                                 -                     -       -       0       i
 * >      RD: 10.1.0.3:10010 mac-ip 0200.0000.0001 10.10.10.1
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.6:10010 mac-ip 0200.0000.0004
                                 10.1.0.6              -       100     0       65000 65004 i
 *  ec    RD: 10.1.0.6:10010 mac-ip 0200.0000.0004
                                 10.1.0.6              -       100     0       65000 65004 i
 * >Ec    RD: 10.1.0.6:10010 mac-ip 0200.0000.0004 10.10.10.4
                                 10.1.0.6              -       100     0       65000 65004 i
 *  ec    RD: 10.1.0.6:10010 mac-ip 0200.0000.0004 10.10.10.4
                                 10.1.0.6              -       100     0       65000 65004 i
 * >Ec    RD: 10.1.0.6:10020 mac-ip 0200.0000.0005
                                 10.1.0.6              -       100     0       65000 65004 i
 *  ec    RD: 10.1.0.6:10020 mac-ip 0200.0000.0005
                                 10.1.0.6              -       100     0       65000 65004 i
 * >Ec    RD: 10.1.0.6:10020 mac-ip 0200.0000.0005 20.20.20.5
                                 10.1.0.6              -       100     0       65000 65004 i
 *  ec    RD: 10.1.0.6:10020 mac-ip 0200.0000.0005 20.20.20.5
                                 10.1.0.6              -       100     0       65000 65004 i
```
</details>

<details>
<summary>LEAF1 / show mac address-table</summary>

```
LEAF1#show mac address-table
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
   1    0000.2222.3333    STATIC      Cpu
  10    0000.2222.3333    STATIC      Cpu
  10    0200.0000.0001    DYNAMIC     Et3        1       0:03:55 ago
  10    0200.0000.0004    DYNAMIC     Vx1        1       0:03:55 ago
  20    0000.2222.3333    STATIC      Cpu
  20    0200.0000.0005    DYNAMIC     Vx1        1       0:03:50 ago
Total Mac Addresses for this criterion: 6
```
</details>


<details>
<summary>ping SRV1 -> SRV4, SRV5</summary>

```
SRV1:/# ping -c 3 10.10.10.4
PING 10.10.10.4 (10.10.10.4): 56 data bytes
64 bytes from 10.10.10.4: seq=0 ttl=64 time=8.321 ms
64 bytes from 10.10.10.4: seq=1 ttl=64 time=8.568 ms
64 bytes from 10.10.10.4: seq=2 ttl=64 time=10.561 ms

--- 10.10.10.4 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 8.321/9.150/10.561 ms
SRV1:/# ping -c 3 20.20.20.5
PING 20.20.20.5 (20.20.20.5): 56 data bytes
64 bytes from 20.20.20.5: seq=0 ttl=63 time=17.633 ms
64 bytes from 20.20.20.5: seq=1 ttl=63 time=9.663 ms
64 bytes from 20.20.20.5: seq=2 ttl=63 time=11.137 ms

--- 20.20.20.5 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 9.663/12.811/17.633 ms
```
</details>

В дампе на LEAF4 видим, что маршрутизация происходит на VTEP-источнике, на VTEP-получателе пакет видим уже в целевом VNI.
<img width="779" height="398" alt="image" src="https://github.com/user-attachments/assets/eca4d5c6-eb81-43e2-9427-615a55b78d87" />
<img width="753" height="396" alt="image" src="https://github.com/user-attachments/assets/59244a4d-8aaf-43b3-8f86-5cefa2b521e6" />


## 7. Настройки для симметричного IRB
<details>
<summary>Контекст: LEAF1</summary>

```
vrf instance TENANT1
!
interface Vlan10
   vrf TENANT1
   ip address 10.10.10.254/24
!
interface Vlan20
   vrf TENANT1
   ip address 20.20.20.254/24
!
interface Vxlan1
   vxlan vrf TENANT1 vni 50001
!
ip virtual-router mac-address 00:00:22:22:33:33
!
ip routing vrf TENANT1
!
router bgp 65001
   vrf TENANT1
      rd 10.1.0.5:50001
      route-target import evpn 50001:50001
      route-target export evpn 50001:50001
      redistribute connected
```
</details>

## 8. Проверка работы симметричного IRB

Отличие - появились route type 5 маршруты.

<details>
<summary>LEAF1 / show bgp evpn</summary>

```
LEAF1#show bgp evpn
BGP routing table information for VRF default
Router identifier 10.1.0.3, local AS number 65001
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.1.0.3:10010 mac-ip 0200.0000.0001
                                 -                     -       -       0       i
 * >      RD: 10.1.0.3:10010 mac-ip 0200.0000.0001 10.10.10.1
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.6:10020 mac-ip 0200.0000.0005
                                 10.1.0.6              -       100     0       65000 65004 i
 *  ec    RD: 10.1.0.6:10020 mac-ip 0200.0000.0005
                                 10.1.0.6              -       100     0       65000 65004 i
 * >Ec    RD: 10.1.0.6:10020 mac-ip 0200.0000.0005 20.20.20.5
                                 10.1.0.6              -       100     0       65000 65004 i
 *  ec    RD: 10.1.0.6:10020 mac-ip 0200.0000.0005 20.20.20.5
                                 10.1.0.6              -       100     0       65000 65004 i
 * >      RD: 10.1.0.3:10010 imet 10.1.0.3
                                 -                     -       -       0       i
 * >      RD: 10.1.0.3:10020 imet 10.1.0.3
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.4:10010 imet 10.1.0.4
                                 10.1.0.4              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.4:10010 imet 10.1.0.4
                                 10.1.0.4              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.4:10020 imet 10.1.0.4
                                 10.1.0.4              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.4:10020 imet 10.1.0.4
                                 10.1.0.4              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.5:10010 imet 10.1.0.5
                                 10.1.0.5              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.5:10010 imet 10.1.0.5
                                 10.1.0.5              -       100     0       65000 65003 i
 * >Ec    RD: 10.1.0.5:10020 imet 10.1.0.5
                                 10.1.0.5              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.5:10020 imet 10.1.0.5
                                 10.1.0.5              -       100     0       65000 65003 i
 * >Ec    RD: 10.1.0.6:10010 imet 10.1.0.6
                                 10.1.0.6              -       100     0       65000 65004 i
 *  ec    RD: 10.1.0.6:10010 imet 10.1.0.6
                                 10.1.0.6              -       100     0       65000 65004 i
 * >Ec    RD: 10.1.0.6:10020 imet 10.1.0.6
                                 10.1.0.6              -       100     0       65000 65004 i
 *  ec    RD: 10.1.0.6:10020 imet 10.1.0.6
                                 10.1.0.6              -       100     0       65000 65004 i
 * >      RD: 10.1.0.3:50001 ip-prefix 10.10.10.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.4:50001 ip-prefix 10.10.10.0/24
                                 10.1.0.4              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.4:50001 ip-prefix 10.10.10.0/24
                                 10.1.0.4              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.5:50001 ip-prefix 10.10.10.0/24
                                 10.1.0.5              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.5:50001 ip-prefix 10.10.10.0/24
                                 10.1.0.5              -       100     0       65000 65003 i
 * >Ec    RD: 10.1.0.6:50001 ip-prefix 10.10.10.0/24
                                 10.1.0.6              -       100     0       65000 65004 i
 *  ec    RD: 10.1.0.6:50001 ip-prefix 10.10.10.0/24
                                 10.1.0.6              -       100     0       65000 65004 i
 * >      RD: 10.1.0.3:50001 ip-prefix 20.20.20.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.4:50001 ip-prefix 20.20.20.0/24
                                 10.1.0.4              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.4:50001 ip-prefix 20.20.20.0/24
                                 10.1.0.4              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.5:50001 ip-prefix 20.20.20.0/24
                                 10.1.0.5              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.5:50001 ip-prefix 20.20.20.0/24
                                 10.1.0.5              -       100     0       65000 65003 i
 * >Ec    RD: 10.1.0.6:50001 ip-prefix 20.20.20.0/24
                                 10.1.0.6              -       100     0       65000 65004 i
 *  ec    RD: 10.1.0.6:50001 ip-prefix 20.20.20.0/24
                                 10.1.0.6              -       100     0       65000 65004 i
```
</details>

<details>
<summary>ping SRV1 -> SRV5</summary>

```
SRV1:/# ping -c 3 20.20.20.5
PING 20.20.20.5 (20.20.20.5): 56 data bytes
64 bytes from 20.20.20.5: seq=0 ttl=62 time=12.825 ms
64 bytes from 20.20.20.5: seq=1 ttl=62 time=10.855 ms
64 bytes from 20.20.20.5: seq=2 ttl=62 time=10.713 ms

--- 20.20.20.5 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 10.713/11.464/12.825 ms
```
</details>

В дампе на LEAF4 видим, что симметричный IRB работает: трафик между VLAN ходит в рамках L3VNI 50001.
<img width="1240" height="403" alt="image" src="https://github.com/user-attachments/assets/8a7d51fd-da7a-470d-851f-330bae35f2a3" />

## 9. Настройка Multihoming
Настройка multihoming  на LEAF2/LEAF3 для сервера SRV3
<details>
<summary>Настройка LEAF</summary>

```
link tracking group CORE-TRACKING                                    # Трэкинг-группа для мониторинга апоинков
   recovery delay 1
!
interface Port-Channel3                                              # Агрегированный интерфейс в сторону SRV3
   description "Trunk to SRV3"
   switchport trunk allowed vlan 10,20                               # Настроен как транк
   switchport mode trunk                                             # Поданы VL10, 20
   !
   evpn ethernet-segment                                             # Настройки Ethernet-сегмента
      identifier 0000:0000:0000:0000:0001                            # ESI
      designated-forwarder election algorithm preference 100         # Приоритет устройства при выборе Designated Forwarder
      route-target import 00:00:00:00:00:01                          # Импорт EVPN route type 4 маршрутов, относящихся к нашему сегменту.
   lacp system-id 0000.0000.3333                                     # System ID для LACP
   spanning-tree portfast
!
interface Ethernet1
   description P2P-SPINE1
   link tracking group CORE-TRACKING upstream                        # Настройка аплинка для трекинга
!
interface Ethernet2
   description P2P-SPINE2
   link tracking group CORE-TRACKING upstream                        # Настройка аплинка для трекинга
!
interface Ethernet3
   description "SRV3"
   switchport trunk allowed vlan 10,20
   switchport mode trunk
   channel-group 3 mode active                                       # Включаем порт в агрегированный интерфейс
   link tracking group CORE-TRACKING downstream                      # Настройка даунлинка для трекинга
```
</details>

## 10. Проверка Multihoming

<details>
<summary>LEAF2 / sh bgp evpn instance</summary>

```
LEAF2#sh bgp evpn instance 
EVPN instance: VLAN 10
  Route distinguisher: 10.1.0.4:10010
  Route target import: Route-Target-AS:10010:10010
  Route target export: Route-Target-AS:10010:10010
  Service interface: VLAN-based
  Local VXLAN IP address: 10.1.0.4
  VXLAN: enabled
  MPLS: disabled
  Local ethernet segment:
    ESI: 0000:0000:0000:0000:0001
      Interface: Port-Channel3
      Mode: all-active
      State: up
      ES-Import RT: 00:00:00:00:00:01
      DF election algorithm: preference
      Designated forwarder: 10.1.0.4
      Non-Designated forwarder: 10.1.0.5
EVPN instance: VLAN 20
  Route distinguisher: 10.1.0.4:10020
  Route target import: Route-Target-AS:10020:10020
  Route target export: Route-Target-AS:10020:10020
  Service interface: VLAN-based
  Local VXLAN IP address: 10.1.0.4
  VXLAN: enabled
  MPLS: disabled
  Local ethernet segment:
    ESI: 0000:0000:0000:0000:0001
      Interface: Port-Channel3
      Mode: all-active
      State: up
      ES-Import RT: 00:00:00:00:00:01
      DF election algorithm: preference
      Designated forwarder: 10.1.0.4
      Non-Designated forwarder: 10.1.0.5
```
</details>

Маршруты route type 4 от второго устройства ESI-LAG:<br>
EvpnEsImportRt:00:00:00:00:00:01<br>
DF Election: Preference 50<br>

<details>
<summary>LEAF2 / route type 4 / sh bgp evpn route-type ethernet-segment detail</summary>

```
LEAF2#sh bgp evpn route-type ethernet-segment detail
BGP routing table information for VRF default
Router identifier 10.1.0.4, local AS number 65002
BGP routing table entry for ethernet-segment 0000:0000:0000:0000:0001 10.1.0.4, Route Distinguisher: 10.1.0.4:1
 Paths: 1 available
  Local
    - from - (0.0.0.0)
      Origin IGP, metric -, localpref -, weight 0, tag 0, valid, local, best
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:00:00:00:00:01 DF Election: Preference 100
BGP routing table entry for ethernet-segment 0000:0000:0000:0000:0001 10.1.0.5, Route Distinguisher: 10.1.0.5:1
 Paths: 2 available
  65000 65003
    10.1.0.5 from 10.1.0.2 (10.1.0.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:00:00:00:00:01 DF Election: Preference 50
  65000 65003
    10.1.0.5 from 10.1.0.1 (10.1.0.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: TunnelEncap:tunnelTypeVxlan EvpnEsImportRt:00:00:00:00:00:01 DF Election: Preference 50
```
</details>

Маршруты route type 1 от второго устройства ESI-LAG:<br>
per-EVI Type-1: для VNI 10010 и 10020.<br>
ESI Type-1: с указанием VNI = 0<br>
<details>
<summary>LEAF2 / route type 1 / sh bgp evpn route-type auto-discovery detail</summary>

```
LEAF2#sh bgp evpn route-type auto-discovery detail 
BGP routing table information for VRF default
Router identifier 10.1.0.4, local AS number 65002
BGP routing table entry for auto-discovery 0 0000:0000:0000:0000:0001, Route Distinguisher: 10.1.0.4:10010
 Paths: 1 available
  Local
    - from - (0.0.0.0)
      Origin IGP, metric -, localpref -, weight 0, tag 0, valid, local, best
      Extended Community: Route-Target-AS:10010:10010 TunnelEncap:tunnelTypeVxlan
      VNI: 10010
BGP routing table entry for auto-discovery 0 0000:0000:0000:0000:0001, Route Distinguisher: 10.1.0.4:10020
 Paths: 1 available
  Local
    - from - (0.0.0.0)
      Origin IGP, metric -, localpref -, weight 0, tag 0, valid, local, best
      Extended Community: Route-Target-AS:10020:10020 TunnelEncap:tunnelTypeVxlan
      VNI: 10020
BGP routing table entry for auto-discovery 0 0000:0000:0000:0000:0001, Route Distinguisher: 10.1.0.5:10010
 Paths: 2 available
  65000 65003
    10.1.0.5 from 10.1.0.2 (10.1.0.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:10010:10010 TunnelEncap:tunnelTypeVxlan
      VNI: 10010
  65000 65003
    10.1.0.5 from 10.1.0.1 (10.1.0.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:10010:10010 TunnelEncap:tunnelTypeVxlan
      VNI: 10010
BGP routing table entry for auto-discovery 0 0000:0000:0000:0000:0001, Route Distinguisher: 10.1.0.5:10020
 Paths: 2 available
  65000 65003
    10.1.0.5 from 10.1.0.2 (10.1.0.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:10020:10020 TunnelEncap:tunnelTypeVxlan
      VNI: 10020
  65000 65003
    10.1.0.5 from 10.1.0.1 (10.1.0.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:10020:10020 TunnelEncap:tunnelTypeVxlan
      VNI: 10020
BGP routing table entry for auto-discovery 0000:0000:0000:0000:0001, Route Distinguisher: 10.1.0.4:1
 Paths: 1 available
  Local
    - from - (0.0.0.0)
      Origin IGP, metric -, localpref -, weight 0, tag 0, valid, local, best
      Extended Community: Route-Target-AS:10010:10010 Route-Target-AS:10020:10020 TunnelEncap:tunnelTypeVxlan EvpnEsiLabel:0
BGP routing table entry for auto-discovery 0000:0000:0000:0000:0001, Route Distinguisher: 10.1.0.5:1
 Paths: 2 available
  65000 65003
    10.1.0.5 from 10.1.0.2 (10.1.0.2)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:10010:10010 Route-Target-AS:10020:10020 TunnelEncap:tunnelTypeVxlan EvpnEsiLabel:0
      VNI: 0
  65000 65003
    10.1.0.5 from 10.1.0.1 (10.1.0.1)
      Origin IGP, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:10010:10010 Route-Target-AS:10020:10020 TunnelEncap:tunnelTypeVxlan EvpnEsiLabel:0
      VNI: 0
```
</details>

<details>
<summary>ping SRV1 -> SRV3</summary>

```
SRV1:/# ping -c 3 10.10.10.3
PING 10.10.10.3 (10.10.10.3): 56 data bytes
64 bytes from 10.10.10.3: seq=0 ttl=64 time=14.132 ms
64 bytes from 10.10.10.3: seq=1 ttl=64 time=11.758 ms
64 bytes from 10.10.10.3: seq=2 ttl=64 time=8.562 ms

--- 10.10.10.3 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 8.562/11.484/14.132 ms
SRV1:/# ping -c 3 20.20.20.3
PING 20.20.20.3 (20.20.20.3): 56 data bytes
64 bytes from 20.20.20.3: seq=0 ttl=62 time=15.363 ms
64 bytes from 20.20.20.3: seq=1 ttl=62 time=13.453 ms
64 bytes from 20.20.20.3: seq=2 ttl=62 time=47.762 ms

--- 20.20.20.3 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 13.453/25.526/47.762 ms
```
</details>

<details>
<summary>ping SRV2 -> SRV3</summary>

```
SRV2:/#  ping -c 3 10.10.10.3
PING 10.10.10.3 (10.10.10.3): 56 data bytes
64 bytes from 10.10.10.3: seq=0 ttl=62 time=31.818 ms
64 bytes from 10.10.10.3: seq=1 ttl=62 time=17.610 ms
64 bytes from 10.10.10.3: seq=2 ttl=62 time=19.113 ms

--- 10.10.10.3 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 17.610/22.847/31.818 ms
SRV2:/#  ping -c 3 20.20.20.3
PING 20.20.20.3 (20.20.20.3): 56 data bytes
64 bytes from 20.20.20.3: seq=0 ttl=64 time=10.705 ms
64 bytes from 20.20.20.3: seq=1 ttl=64 time=8.896 ms
64 bytes from 20.20.20.3: seq=2 ttl=64 time=8.558 ms

--- 20.20.20.3 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 8.558/9.386/10.705 ms
```
</details>

<details>
<summary>LEAF1 / sh bgp evpn route-type mac-ip </summary>

```
LEAF1#sh bgp evpn route-type mac-ip 
BGP routing table information for VRF default
Router identifier 10.1.0.3, local AS number 65001
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.1.0.3:10020 mac-ip 0200.0000.0002
                                 -                     -       -       0       i
 * >      RD: 10.1.0.3:10020 mac-ip 0200.0000.0002 20.20.20.1
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.4:10010 mac-ip 0200.0000.0103
                                 10.1.0.4              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.4:10010 mac-ip 0200.0000.0103
                                 10.1.0.4              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.5:10010 mac-ip 0200.0000.0103
                                 10.1.0.5              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.5:10010 mac-ip 0200.0000.0103
                                 10.1.0.5              -       100     0       65000 65003 i
 * >Ec    RD: 10.1.0.4:10010 mac-ip 0200.0000.0103 10.10.10.3
                                 10.1.0.4              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.4:10010 mac-ip 0200.0000.0103 10.10.10.3
                                 10.1.0.4              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.5:10010 mac-ip 0200.0000.0103 10.10.10.3
                                 10.1.0.5              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.5:10010 mac-ip 0200.0000.0103 10.10.10.3
                                 10.1.0.5              -       100     0       65000 65003 i
 * >Ec    RD: 10.1.0.4:10020 mac-ip 0200.0000.0203
                                 10.1.0.4              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.4:10020 mac-ip 0200.0000.0203
                                 10.1.0.4              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.5:10020 mac-ip 0200.0000.0203
                                 10.1.0.5              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.5:10020 mac-ip 0200.0000.0203
                                 10.1.0.5              -       100     0       65000 65003 i
 * >Ec    RD: 10.1.0.4:10020 mac-ip 0200.0000.0203 20.20.20.3
                                 10.1.0.4              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.4:10020 mac-ip 0200.0000.0203 20.20.20.3
                                 10.1.0.4              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.5:10020 mac-ip 0200.0000.0203 20.20.20.3
                                 10.1.0.5              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.5:10020 mac-ip 0200.0000.0203 20.20.20.3
                                 10.1.0.5              -       100     0       65000 65003 i
 * >Ec    RD: 10.1.0.6:10020 mac-ip 26ee.25d2.599c
                                 10.1.0.6              -       100     0       65000 65004 i
 *  ec    RD: 10.1.0.6:10020 mac-ip 26ee.25d2.599c
                                 10.1.0.6              -       100     0       65000 65004 i
 * >      RD: 10.1.0.3:10010 mac-ip 5000.0008.0001
                                 -                     -       -       0       i
 * >      RD: 10.1.0.3:10010 mac-ip 5000.0008.0001 10.10.10.1
                                 -                     -       -       0       i
```
</details>


<details>
<summary>LEAF2 / sh bgp evpn route-type mac-ip </summary>

```
LEAF2#sh bgp evpn route-type mac-ip 
BGP routing table information for VRF default
Router identifier 10.1.0.4, local AS number 65002
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.1.0.3:10020 mac-ip 0200.0000.0002
                                 10.1.0.3              -       100     0       65000 65001 i
 *  ec    RD: 10.1.0.3:10020 mac-ip 0200.0000.0002
                                 10.1.0.3              -       100     0       65000 65001 i
 * >Ec    RD: 10.1.0.3:10020 mac-ip 0200.0000.0002 20.20.20.1
                                 10.1.0.3              -       100     0       65000 65001 i
 *  ec    RD: 10.1.0.3:10020 mac-ip 0200.0000.0002 20.20.20.1
                                 10.1.0.3              -       100     0       65000 65001 i
 * >      RD: 10.1.0.4:10010 mac-ip 0200.0000.0103
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.5:10010 mac-ip 0200.0000.0103
                                 10.1.0.5              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.5:10010 mac-ip 0200.0000.0103
                                 10.1.0.5              -       100     0       65000 65003 i
 * >      RD: 10.1.0.4:10010 mac-ip 0200.0000.0103 10.10.10.3
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.5:10010 mac-ip 0200.0000.0103 10.10.10.3
                                 10.1.0.5              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.5:10010 mac-ip 0200.0000.0103 10.10.10.3
                                 10.1.0.5              -       100     0       65000 65003 i
 * >      RD: 10.1.0.4:10020 mac-ip 0200.0000.0203
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.5:10020 mac-ip 0200.0000.0203
                                 10.1.0.5              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.5:10020 mac-ip 0200.0000.0203
                                 10.1.0.5              -       100     0       65000 65003 i
 * >      RD: 10.1.0.4:10020 mac-ip 0200.0000.0203 20.20.20.3
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.5:10020 mac-ip 0200.0000.0203 20.20.20.3
                                 10.1.0.5              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.5:10020 mac-ip 0200.0000.0203 20.20.20.3
                                 10.1.0.5              -       100     0       65000 65003 i
 * >Ec    RD: 10.1.0.3:10010 mac-ip 5000.0008.0001
                                 10.1.0.3              -       100     0       65000 65001 i
 *  ec    RD: 10.1.0.3:10010 mac-ip 5000.0008.0001
                                 10.1.0.3              -       100     0       65000 65001 i
 * >Ec    RD: 10.1.0.3:10010 mac-ip 5000.0008.0001 10.10.10.1
                                 10.1.0.3              -       100     0       65000 65001 i
 *  ec    RD: 10.1.0.3:10010 mac-ip 5000.0008.0001 10.10.10.1
                                 10.1.0.3              -       100     0       65000 65001 i
```
</details>

<details>
<summary>LEAF1 / sh mac address-table </summary>

```
LEAF1#sh mac address-table 
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
   1    0000.2222.3333    STATIC      Cpu
  10    0000.2222.3333    STATIC      Cpu
  10    0200.0000.0103    DYNAMIC     Vx1        2       0:03:35 ago
  10    5000.0008.0001    DYNAMIC     Et3        1       0:03:14 ago
  20    0000.2222.3333    STATIC      Cpu
  20    0200.0000.0002    DYNAMIC     Et4        1       0:02:33 ago
  20    0200.0000.0203    DYNAMIC     Vx1        1       0:03:35 ago
  20    26ee.25d2.599c    DYNAMIC     Vx1        1       0:01:23 ago
4094    0000.2222.3333    STATIC      Cpu
4094    5010.07b7.82f4    DYNAMIC     Vx1        1       1:18:41 ago
4094    5037.1a89.95a2    DYNAMIC     Vx1        1       0:03:35 ago
4094    50a6.2620.e7e9    DYNAMIC     Vx1        1       0:16:53 ago
Total Mac Addresses for this criterion: 12
```
</details>

<details>
<summary>LEAF2 / sh mac address-table </summary>

```
LEAF2# sh mac address-table 
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  10    0000.2222.3333    STATIC      Cpu
  10    0200.0000.0103    DYNAMIC     Po3        2       0:04:38 ago
  10    5000.0008.0001    DYNAMIC     Vx1        1       0:04:38 ago
  20    0000.2222.3333    STATIC      Cpu
  20    0200.0000.0002    DYNAMIC     Vx1        1       0:03:56 ago
  20    0200.0000.0203    DYNAMIC     Po3        2       0:04:26 ago
  20    26ee.25d2.599c    DYNAMIC     Vx1        1       0:02:47 ago
4093    0000.2222.3333    STATIC      Cpu
4093    5037.1a89.95a2    DYNAMIC     Vx1        1       0:04:59 ago
4093    504a.d149.f27a    DYNAMIC     Vx1        1       2:26:23 ago
4093    50a6.2620.e7e9    DYNAMIC     Vx1        1       0:18:17 ago
Total Mac Addresses for this criterion: 11
```
</details>

## 12. Отказоустойчивость Multihoming

Ставим бесконечный ping SRV1 (10.10.10.1) -> SRV3 (20.20.20.3)

<details>
<summary>LEAF1  / sh bgp evpn route-type mac-ip 20.20.20.3</summary>

```
LEAF1#sh bgp evpn route-type mac-ip 20.20.20.3
BGP routing table information for VRF default
Router identifier 10.1.0.3, local AS number 65001
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.1.0.4:10020 mac-ip 0200.0000.0203 20.20.20.3
                                 10.1.0.4              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.4:10020 mac-ip 0200.0000.0203 20.20.20.3
                                 10.1.0.4              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.5:10020 mac-ip 0200.0000.0203 20.20.20.3
                                 10.1.0.5              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.5:10020 mac-ip 0200.0000.0203 20.20.20.3
                                 10.1.0.5              -       100     0       65000 65003 i
```
</details>

1. Отказ даунлинка LEAF2 Eth3
- Отключаем на LEAF2 интерфейс Eth3
```
LEAF2(config)#int e3
LEAF2(config-if-Et3)#shutdown 
```
- Проверяем статус агрегированного интерфейса:
```
LEAF2#sh int po3
Port-Channel3 is down, line protocol is lowerlayerdown (notconnect)
  Hardware is Port-Channel, address is 5010.0700.0403
  Description: "Trunk to SRV3"
  Ethernet MTU 9214 bytes
  Full-duplex, Unconfigured 
  Active members in this channel: 0
  Fallback mode is: off
  Down 1 minute, 3 seconds
  40 link status changes since last clear
  Last clearing of "show interface" counters never
  5 minutes input rate 0 bps (- with framing overhead), 0 packets/sec
  5 minutes output rate 0 bps (- with framing overhead), 0 packets/sec
     88 packets input, 10908 bytes
     Received 4 broadcasts, 53 multicast
     0 input errors, 0 input discards
     1702 packets output, 201089 bytes
     Sent 103 broadcasts, 1232 multicast
     0 output errors, 0 output discards
```
- Фиксируем отсутствие mac-ip маршрута для 20.20.20.3 от LEAF2
```
LEAF1#sh bgp evpn route-type mac-ip 20.20.20.3
BGP routing table information for VRF default
Router identifier 10.1.0.3, local AS number 65001
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.1.0.5:10020 mac-ip 0200.0000.0203 20.20.20.3
                                 10.1.0.5              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.5:10020 mac-ip 0200.0000.0203 20.20.20.3
                                 10.1.0.5              -       100     0       65000 65003 i
```
- Фиксируем отсутствие потерянных паектов
```
SRV1:/# ping -c 10000 20.20.20.3
PING 20.20.20.3 (20.20.20.3): 56 data bytes
64 bytes from 20.20.20.3: seq=0 ttl=62 time=11.750 ms
...
64 bytes from 20.20.20.3: seq=105 ttl=62 time=11.135 ms
^C
--- 20.20.20.3 ping statistics ---
106 packets transmitted, 106 packets received, 0% packet loss
round-trip min/avg/max = 9.932/11.976/39.116 ms
```
2. Отказ одного аплинка LEAF2 -> SPINE1
- Отключаем линк LEAF2 -> SPINE1
```
LEAF2#conf t
LEAF2(config)#int e1
LEAF2(config-if-Et1)#shutdown 
```
- Фиксируем отсутствие маршрута через SPINE1
```
LEAF1#sh bgp evpn route-type mac-ip 20.20.20.3
BGP routing table information for VRF default
Router identifier 10.1.0.3, local AS number 65001
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.1.0.4:10020 mac-ip 0200.0000.0203 20.20.20.3
                                 10.1.0.4              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.5:10020 mac-ip 0200.0000.0203 20.20.20.3
                                 10.1.0.5              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.5:10020 mac-ip 0200.0000.0203 20.20.20.3
                                 10.1.0.5              -       100     0       65000 65003 i
```
- На доступность адреса 20.20.20.3 влияния не оказывает.

3. Отказ всех аплинков на LEAF2
- Отключаем аплинки на LEAF2
```
LEAF2#conf t
LEAF2(config)#int e1-2
LEAF2(config-if-Et1-2)#shutdown 
```
- Фиксируем отсутствие маршрутов от LEAF2
```
LEAF1#sh bgp evpn route-type mac-ip 20.20.20.3
BGP routing table information for VRF default
Router identifier 10.1.0.3, local AS number 65001
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.1.0.5:10020 mac-ip 0200.0000.0203 20.20.20.3
                                 10.1.0.5              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.5:10020 mac-ip 0200.0000.0203 20.20.20.3
                                 10.1.0.5              -       100     0       65000 65003 i
```
- Проверяем статус link tracking
```
LEAF2#show link tracking group detail
Link State Group: CORE-TRACKING Status: down
Upstream Interfaces : Ethernet2 Ethernet1
Downstream Interfaces : Ethernet3
Number of times disabled : 13
Last disabled 0:02:21 ago
```
- фиксируем, что link tracking отправил downstream интерфейс в errdisabled
```
LEAF2#sh int e3
Ethernet3 is down, line protocol is down (errdisabled)
  Hardware is Ethernet, address is 5010.0700.0403 (bia 5010.0700.0403)
  Description: "SRV3"
  Member of Port-Channel3
  Ethernet MTU 9214 bytes, BW 1000000 kbit
  Full-duplex, 1Gb/s, auto negotiation: off, uni-link: n/a
  Down 3 minutes, 52 seconds
  Loopback Mode : None
  33 link status changes since last clear
  Last clearing of "show interface" counters never
  5 minutes input rate 0 bps (0.0% with framing overhead), 0 packets/sec
  5 minutes output rate 0 bps (0.0% with framing overhead), 0 packets/sec
     155 packets input, 20342 bytes
     Received 4 broadcasts, 120 multicast
     0 runts, 0 giants
     0 input errors, 0 CRC, 0 alignment, 0 symbol, 0 input discards
     0 PAUSE input
     2136 packets output, 255330 bytes
     Sent 111 broadcasts, 1604 multicast
     0 output errors, 0 collisions
     0 late collision, 0 deferred, 0 output discards
     0 PAUSE output
```
- соответствуено, Po3 на LEAF2 уходит в DOWN
```
LEAF2#sh int po3
Port-Channel3 is down, line protocol is lowerlayerdown (notconnect)
  Hardware is Port-Channel, address is 5010.0700.0403
  Description: "Trunk to SRV3"
  Ethernet MTU 9214 bytes
  Full-duplex, Unconfigured 
  Active members in this channel: 0
  Fallback mode is: off
  Down 3 minutes, 54 seconds
  49 link status changes since last clear
  Last clearing of "show interface" counters never
  5 minutes input rate 0 bps (- with framing overhead), 0 packets/sec
  5 minutes output rate 0 bps (- with framing overhead), 0 packets/sec
     129 packets input, 16975 bytes
     Received 4 broadcasts, 94 multicast
     0 input errors, 0 input discards
     2071 packets output, 246388 bytes
     Sent 111 broadcasts, 1545 multicast
     0 output errors, 0 output discards
```
- Потери тестовых пакетов не зафиксировано
```
SRV1:/# ping -c 10000 20.20.20.3
PING 20.20.20.3 (20.20.20.3): 56 data bytes
64 bytes from 20.20.20.3: seq=0 ttl=63 time=19.274 ms
...
64 bytes from 20.20.20.3: seq=407 ttl=63 time=10.506 ms
^C
...
--- 20.20.20.3 ping statistics ---
408 packets transmitted, 408 packets received, 0% packet loss
round-trip min/avg/max = 10.260/13.219/46.997 ms
```

Проведенные эксперименты показывают корректную работу отказоустойчивости при следующих сценариях:
- отказ даунлинка на одном из LEAF
- потери одного из аплинков на одном из LEAF
- потери всех аплинков на одном из LEAF
- выход из строя одного из LEAF

## 12. Конфигурации устройств фабрики
[Конфигурация SPINE1](./configs/spine1.conf)<br>
[Конфигурация SPINE2](./configs/spine2.conf)<br>
[Конфигурация LEAF1](./configs/leaf1.conf)<br>
[Конфигурация LEAF2](./configs/leaf2.conf)<br>
[Конфигурация LEAF3](./configs/leaf3.conf)<br>
[Конфигурация LEAF4](./configs/leaf4.conf)<br>
