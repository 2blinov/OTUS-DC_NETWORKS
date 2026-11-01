# Проектная работа "Проектирование мультитенантной сетевой фабрики EVPN Multi-Site для двух ЦОД с контролируемым взаимодействием сетевых сегментов и централизованной защитой периметра средствами МСЭ"

## Оглавление:
1. [Подготовка стенда](#1-подготовка-стенда)
2. [Разработка адресного плана](#2-разработка-адресного-плана)
3. [Настройка стенда](#3-настройка-стенда)
4. [Проверка связности](#4-проверка-связности)
5. [Проверка работы фабрики](#5-проверка-работы-фабрики)
6. [Конфигурации устройств фабрики](#6-конфигурации-устройств-фабрики)

## 1. Подготовка стенда
В качестве платформы для организации стенда был выбран ContainerLab, развернутый на WSL, с использованием образов Arista cEOS, контейнеров с Ubuntu/FRR и alpine.
Получившийся стенд выглядит следующим образом:
<img width="5688" height="3288" alt="image" src="https://github.com/user-attachments/assets/58dd2bca-199d-4b9f-8712-69a8c29a3d95" />

* VLAN10 и VLAN20 находятся в разных VRF (TENANT1, TENANT2).
* BGW1, BGW2 являются пограничными устройствами сайта для обеспечения DCI.
* LEAF12, LEAF22 являются пограничными, которые соседствует с внешним (для фабрики) миром через МСЭ своей площадки. 
* Связность VRF с внешним миром осуществляется посредством анонса с FW1, FW2 маршрута по умолчанию 0.0.0.0/0
* GW связывает две площадки и эмулирует внешний мир.

## 2. Разработка адресного плана
Для адресного плана предлагаем использовать приватную сеть 10.0.0.0/8. При этом второй октет мы будем использовать как индекс сайта, для которого предназначена адресация. Для Lo0 предлагаем зарезервировать подсеть /24 (сможем адресовать 256 устройств). Для транспортных подсетей /31 предлагаю зарезервировать подсеть /23 (запас вплоть до фабрики 8 Spine / 32 Leaf). Для адресации сервисов зарезервируем подсеть /21. Итого, общая адресация каждого ЦОД будет суммироваться до /20 (с учетом зарезервированных адресов для возможного расширения).

| Устройство | Подсеть     |
| ---------- | ----------- |
| Loopback0  | 10.x.0.0/24 |
| Reserved   | 10.x.1.0/24 |
| Transport  | 10.x.2.0/23 |
| Reserved   | 10.x.4.0/22 |
| Service    | 10.x.8.0/21 |

## 3. Настройка стенда
Для проектной работы настроим стенд согласно адресному плану (DC1, DC2)

DC1 -- Loopback0
| Device |    Loopback0 |
| ------ | -----------: |
| BGW1    |  10.1.0.0/32 |
| SPINE11 |  10.1.0.1/32 |
| SPINE12 |  10.1.0.2/32 |
| LEAF11  | 10.1.0.3/32 |
| LEAF12  | 10.1.0.4/32 |

DC1 -- Транспортные подсети
| Link           | Network      |
| -------------- | ------------ |
| BGW1 – SPINE11 | 10.1.2.0/31  |
| BGW1 – SPINE12 | 10.1.2.2/31  |
| SPINE11 – LEAF11 | 10.1.2.4/31  |
| SPINE11 – LEAF12 | 10.1.2.6/31  |
| SPINE12 – LEAF11 | 10.1.2.8/31  |
| SPINE12 – LEAF12 | 10.1.2.10/31 |
| LEAF12 – FW1 (TENANT1) | 10.1.2.12/31 |
| LEAF12 – FW1 (TENANT2) | 10.1.2.14/31 |

DC2 -- Loopback0
| Device |    Loopback0 |
| ------ | -----------: |
| BGW2    |  10.2.0.0/32 |
| SPINE21 |  10.2.0.1/32 |
| SPINE22 |  10.2.0.2/32 |
| LEAF21  | 10.2.0.3/32 |
| LEAF22  | 10.2.0.4/32 |

DC2 -- Транспортные подсети
| Link           | Network      |
| -------------- | ------------ |
| BGW1 – SPINE11 | 10.2.2.0/31  |
| BGW1 – SPINE12 | 10.2.2.2/31  |
| SPINE11 – LEAF11 | 10.2.2.4/31  |
| SPINE11 – LEAF12 | 10.2.2.6/31  |
| SPINE12 – LEAF11 | 10.2.2.8/31  |
| SPINE12 – LEAF12 | 10.2.2.10/31 |
| LEAF12 – FW1 (TENANT1) | 10.2.2.12/31 |
| LEAF12 – FW1 (TENANT2) | 10.2.2.14/31 |

DCI -- Loopback1
| Device |    Loopback1 |
| ------ | -----------: |
| BGW1  |  10.0.0.0/32 |
| BGW2  |  10.0.0.1/32 |

DCI -- Транспортные подсети
| Link           | Network      |
| -------------- | ------------ |
| BGW1 – BGW2    | 10.0.0.2/31  |

Intra -- Loopback0
| Device |    Loopback0 |
| ------ | -----------: |
| GW   |  10.0.1.0/32 |
| FW1  |  10.0.1.1/32 |
| FW1  |  10.0.1.2/32 |

Intra -- Транспортные подсети
| Link           | Network      |
| -------------- | ------------ |
| GW – FW1    | 10.0.2.0/31  |
| GW – FW2    | 10.0.2.2/31  |

Ключевые моменты настройки:
<details>
<summary>Контекст: Процесс BGP SPINE</summary>

```eos
router bgp 65001
   router-id 10.1.0.1
   maximum-paths 4 ecmp 4
   bgp listen range 10.1.0.0/24 peer-group LEAFS-EVPN peer-filter LEAFS-AS-FILTER
   bgp listen range 10.1.2.0/23 peer-group LEAFS-UNDERLAY peer-filter LEAFS-AS-FILTER
   neighbor LEAFS-EVPN peer group
   neighbor LEAFS-EVPN next-hop-unchanged
   neighbor LEAFS-EVPN update-source Loopback0
   neighbor LEAFS-EVPN bfd
   neighbor LEAFS-EVPN ebgp-multihop 5
   neighbor LEAFS-EVPN route-reflector-client
   neighbor LEAFS-EVPN send-community extended
   neighbor LEAFS-UNDERLAY peer group
   neighbor LEAFS-UNDERLAY remote-as 65001
   neighbor LEAFS-UNDERLAY bfd
   neighbor LEAFS-UNDERLAY route-reflector-client
   !
   address-family evpn
      neighbor LEAFS-EVPN activate
   !
   address-family ipv4
      no neighbor LEAFS-EVPN activate
      neighbor LEAFS-UNDERLAY activate
      network 10.1.0.1/32
      network 10.1.2.0/31
      network 10.1.2.4/31
      network 10.1.2.6/31
```
</details>

<details>
<summary>Контекст: Настройки LEAF</summary>

```eos
vrf instance TENANT1
!
vrf instance TENANT2
!
interface Vlan10
   description VLAN10
   vrf TENANT1
   ip address 10.10.10.254/24
!
interface Vlan20
   description VLAN20
   vrf TENANT2
   ip address 10.10.20.254/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vrf TENANT1 vni 50001
   vxlan vrf TENANT2 vni 50002
!
ip routing
ip routing vrf TENANT1
ip routing vrf TENANT2
!
router bgp 65001
   router-id 10.1.0.3
   maximum-paths 4 ecmp 4
   neighbor SPINE-EVPN peer group
   neighbor SPINE-EVPN remote-as 65001
   neighbor SPINE-EVPN next-hop-unchanged
   neighbor SPINE-EVPN update-source Loopback0
   neighbor SPINE-EVPN bfd
   neighbor SPINE-EVPN ebgp-multihop 5
   neighbor SPINE-EVPN send-community extended
   neighbor SPINE-UNDERLAY peer group
   neighbor SPINE-UNDERLAY remote-as 65001
   neighbor SPINE-UNDERLAY bfd
   neighbor 10.1.0.1 peer group SPINE-EVPN
   neighbor 10.1.0.2 peer group SPINE-EVPN
   neighbor 10.1.2.4 peer group SPINE-UNDERLAY
   neighbor 10.1.2.8 peer group SPINE-UNDERLAY
   !
   vlan 10
      rd auto
      route-target both 10:10010
      redistribute learned
   !
   vlan 20
      rd auto
      route-target both 20:10020
      redistribute learned
   !
   address-family evpn
      neighbor SPINE-EVPN activate
   !
   address-family ipv4
      no neighbor SPINE-EVPN activate
      neighbor SPINE-UNDERLAY activate
      network 10.1.0.3/32
      network 10.1.2.4/31
      network 10.1.2.8/31
   !
   vrf TENANT1
      rd 10.1.0.3:1
      route-target import evpn 1:50001
      route-target export evpn 1:50001
   !
   vrf TENANT2
      rd 10.1.0.3:2
      route-target import evpn 1:50002
      route-target export evpn 1:50002
```
</details>

<details>
<summary>Контекст: Настройки Border LEAF</summary>

```eos
vlan 110
   name TR-FW-TENANT1
!
vlan 120
   name TR-FW-TENANT2
!
vrf instance TENANT1
!
vrf instance TENANT2
!
interface Vlan110
   description TR-FW-TENANT1
   vrf TENANT1
   ip address 10.1.2.12/31
!
interface Vlan120
   description TR-FW-TENANT2
   vrf TENANT2
   ip address 10.1.2.14/31
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vrf TENANT1 vni 50001
   vxlan vrf TENANT2 vni 50002
!
ip routing
ip routing vrf TENANT1
ip routing vrf TENANT2
!
ip as-path access-list LOCAL-ORIGIN permit ^$ any
!
ip prefix-list PL_DEFAULT
   seq 10 permit 0.0.0.0/0
!
ip prefix-list PL_TENANT1
   seq 10 permit 10.10.10.0/24
   seq 20 permit 10.10.10.0/24 ge 32 le 32
!
ip prefix-list PL_TENANT2
   seq 10 permit 10.10.20.0/24 le 32
   seq 20 permit 10.10.20.0/24 ge 32 le 32
!
ip route vrf TENANT1 10.10.10.0/24 Null0
ip route vrf TENANT2 10.10.20.0/24 Null0
!
route-map RM_EXPORT_EVPN permit 10
   match ip address prefix-list PL_DEFAULT
!
route-map RM_TENANT1-OUT permit 10
   match ip address prefix-list PL_TENANT1
   match as-path LOCAL-ORIGIN
!
route-map RM_TENANT2-OUT permit 10
   match ip address prefix-list PL_TENANT2
   match as-path LOCAL-ORIGIN
!
route-map SITE1-HOSTS permit 10
   match as-path LOCAL-ORIGIN
   set community 65001:5
!
router bgp 65001
   router-id 10.1.0.4
   maximum-paths 4 ecmp 4
   neighbor SPINE-EVPN peer group
   neighbor SPINE-EVPN remote-as 65001
   neighbor SPINE-EVPN next-hop-unchanged
   neighbor SPINE-EVPN update-source Loopback0
   neighbor SPINE-EVPN bfd
   neighbor SPINE-EVPN ebgp-multihop 5
   neighbor SPINE-EVPN send-community extended
   neighbor SPINE-UNDERLAY peer group
   neighbor SPINE-UNDERLAY remote-as 65001
   neighbor SPINE-UNDERLAY bfd
   neighbor 10.1.0.1 peer group SPINE-EVPN
   neighbor 10.1.0.2 peer group SPINE-EVPN
   neighbor 10.1.2.6 peer group SPINE-UNDERLAY
   neighbor 10.1.2.10 peer group SPINE-UNDERLAY
   !
   address-family evpn
      neighbor SPINE-EVPN activate
   !
   address-family ipv4
      no neighbor SPINE-EVPN activate
      neighbor SPINE-UNDERLAY activate
      network 10.1.0.4/32
      network 10.1.2.6/31
      network 10.1.2.10/31
   !
   vrf TENANT1
      rd 10.1.0.4:50001
      route-target import evpn 1:50001
      route-target export evpn 1:50001
      route-target export evpn route-map RM_EXPORT_EVPN
      router-id 10.1.0.4
      maximum-paths 4 ecmp 4
      neighbor 10.1.2.13 remote-as 65101
      neighbor 10.1.2.13 bfd
      redistribute attached-host route-map SITE1-HOSTS
      !
      address-family ipv4
         neighbor 10.1.2.13 activate
         neighbor 10.1.2.13 route-map RM_TENANT1-OUT out
         neighbor 10.1.2.13 prefix-list PL_DEFAULT in
         network 10.10.10.0/24
   !
   vrf TENANT2
      rd 10.1.0.4:50002
      route-target import evpn 1:50002
      route-target export evpn 1:50002
      route-target export evpn route-map RM_EXPORT_EVPN
      router-id 10.1.0.4
      maximum-paths 4 ecmp 4
      neighbor 10.1.2.15 remote-as 65101
      neighbor 10.1.2.15 bfd
      redistribute attached-host route-map SITE1-HOSTS
      !
      address-family ipv4
         neighbor 10.1.2.15 activate
         neighbor 10.1.2.15 route-map RM_TENANT2-OUT out
         neighbor 10.1.2.15 prefix-list PL_DEFAULT in
         network 10.10.20.0/24
```
</details>

<details>
<summary>Настройкb BGW</summary>

```eos
vlan 10
   name VLAN10
!
vlan 20
   name VLAN20
!
vrf instance TENANT1
!
vrf instance TENANT2
!
interface Loopback1
   description VIP-DCI
   ip address 10.0.0.0/32
!
interface Vlan10
   vrf TENANT1
   ip address 10.10.10.251/24
!
interface Vlan20
   vrf TENANT2
   ip address 10.10.20.251/24
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan virtual-router encapsulation mac-address 00:00:00:00:01:01
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vrf TENANT1 vni 50001
   vxlan vrf TENANT2 vni 50002
   vxlan learn-restrict any
!
ip routing
ip routing vrf TENANT1
ip routing vrf TENANT2
!
router bgp 65001
   router-id 10.1.0.0
   no bgp default ipv4-unicast
   timers bgp 1 3
   distance bgp 20 200 200
   maximum-paths 4 ecmp 4
   neighbor DCI peer group
   neighbor DCI remote-as 65002
   neighbor DCI send-community extended
   neighbor SPINE-EVPN peer group
   neighbor SPINE-EVPN remote-as 65001
   neighbor SPINE-EVPN update-source Loopback0
   neighbor SPINE-EVPN send-community extended
   neighbor UNDERLAY peer group
   neighbor UNDERLAY remote-as 65001
   neighbor 10.0.0.3 peer group DCI
   neighbor 10.1.0.1 peer group SPINE-EVPN
   neighbor 10.1.0.2 peer group SPINE-EVPN
   neighbor 10.1.2.1 peer group UNDERLAY
   neighbor 10.1.2.3 peer group UNDERLAY
   !
   vlan 10
      rd evpn domain all 10.1.0.0:10
      route-target both 10:10010
      route-target import export evpn domain remote 10:10
      redistribute learned
   !
   vlan 20
      rd evpn domain all 10.1.0.0:20
      route-target both 20:10020
      route-target import export evpn domain remote 20:20
      redistribute learned
   !
   address-family evpn
      neighbor DCI activate
      neighbor DCI domain remote
      neighbor SPINE-EVPN activate
      domain identifier 1:1
      domain identifier 2:2 remote
      neighbor default next-hop-self received-evpn-routes route-type ip-prefix inter-domain
   !
   address-family ipv4
      neighbor DCI activate
      neighbor UNDERLAY activate
      network 10.0.0.0/32
      network 10.0.0.2/31
      network 10.1.0.0/32
      network 10.1.2.0/31
      network 10.1.2.2/31
   !
   vrf TENANT1
      rd 65001:101
      rd evpn domain remote 65002:101
      route-target import evpn 1:50001
      route-target export evpn 1:50001
   !
   vrf TENANT2
      rd 65001:102
      rd evpn domain remote 65002:102
      route-target import evpn 1:50002
      route-target export evpn 1:50002
      redistribute connected
!
```
</details>

<details>
<summary>Настройкb LEAF4</summary>

```eos
!
vlan 110                                    # Транспортный VLAN vrf TENANT1 -> FW
   name TR-FW-TENANT1
!
vlan 120                                    # Транспортный VLAN vrf TENANT2 -> FW
   name TR-FW-TENANT2
!
vrf instance TENANT1
!
vrf instance TENANT2
!
interface Ethernet1
   description P2P-SPINE1
   mtu 9214
   no switchport
   ip address 10.1.2.13/31
!
interface Ethernet2
   description P2P-SPINE2
   mtu 9214
   no switchport
   ip address 10.1.2.15/31
!
interface Ethernet3                         # Транковый интерфейс LEAF4 -> FW
   description TRUNK-2-FW
   switchport trunk allowed vlan 110,120
   switchport mode trunk
!
interface Loopback0
   description ROUTER-ID
   ip address 10.1.0.6/32
!
interface Vlan110                           # SVI для транспорта в vrf TENANT1
   description TR-FW-TENANT1
   vrf TENANT1
   ip address 10.1.2.16/31
!
interface Vlan120                           # SVI для транспорта в vrf TENANT2
   description TR-FW-TENANT2
   vrf TENANT2
   ip address 10.1.2.18/31
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vrf TENANT1 vni 50001              # L3VNI для vrf TENANT1
   vxlan vrf TENANT2 vni 50002              # L3VNI для vrf TENANT2
!
ip routing
ip routing vrf TENANT1
ip routing vrf TENANT2
!
ip prefix-list PL_DEFAULT                   # Префикс-лист, разрешающий только маршрут по умолчанию
   seq 10 permit 0.0.0.0/0
!
ip prefix-list PL_TENANT1                   # Префикс-лист для сетей TENANT1
   seq 10 permit 10.10.10.0/24
!
ip prefix-list PL_TENANT2                   # Префикс-лист для сетей TENANT2
   seq 10 permit 20.20.20.0/24
!
ip route vrf TENANT1 10.10.10.0/24 Null0    # Статика для постоянного анонса маршрута в сторону FW (TENANT1)
ip route vrf TENANT2 20.20.20.0/24 Null0    # Статика для постоянного анонса маршрута в сторону FW (TENANT2)
!
route-map RM_EXPORT_EVPN permit 10          # Роут-мап для маршрутов, отдаваемых с BORDER LEAF в EVPN-фабрику (хотим отдавать только дефолт, полученный от FW)
   match ip address prefix-list PL_DEFAULT
!
route-map RM_REDISTRIBUTE-Lo0 permit 10
   match interface Loopback0
   set origin igp
!
router bgp 65004
   router-id 10.1.0.6
   maximum-paths 4
   neighbor SPINE-EVPN peer group
   neighbor SPINE-EVPN remote-as 65000
   neighbor SPINE-EVPN next-hop-unchanged
   neighbor SPINE-EVPN update-source Loopback0
   neighbor SPINE-EVPN bfd
   neighbor SPINE-EVPN ebgp-multihop 5
   neighbor SPINE-EVPN send-community extended
   neighbor SPINE-UNDERLAY peer group
   neighbor SPINE-UNDERLAY remote-as 65000
   neighbor SPINE-UNDERLAY bfd
   neighbor 10.1.0.1 peer group SPINE-EVPN
   neighbor 10.1.0.2 peer group SPINE-EVPN
   neighbor 10.1.2.12 peer group SPINE-UNDERLAY
   neighbor 10.1.2.14 peer group SPINE-UNDERLAY
   !
   vlan 110
      rd 10.1.0.6:10110
      route-target both 10110:10110
      redistribute learned
   !
   vlan 120
      rd 10.1.0.6:10120
      route-target both 10120:10120
      redistribute learned
   !
   address-family evpn
      neighbor SPINE-EVPN activate
   !
   address-family ipv4
      no neighbor SPINE-EVPN activate
      neighbor SPINE-UNDERLAY activate
      redistribute connected route-map RM_REDISTRIBUTE-Lo0
   !
   vrf TENANT1
      rd 10.1.0.6:50001
      route-target import evpn 50001:50001
      route-target export evpn 50001:50001
      route-target export evpn route-map RM_EXPORT_EVPN         # Отдаем с BORDER LEAF в EVPN-фабрику только дефолт, полученный от FW
      maximum-paths 4 ecmp 4
      neighbor 10.1.2.17 remote-as 65500                        # Соседство с FW в vrf TENANT1
      !
      address-family ipv4
         no neighbor 10.0.1.17 activate
         neighbor 10.1.2.17 activate
         neighbor 10.1.2.17 prefix-list PL_DEFAULT in           # Принимаем только дефолт
         neighbor 10.1.2.17 prefix-list PL_TENANT1 out          # Отдаем только 10.10.10.0/24
         network 10.10.10.0/24
   !
   vrf TENANT2
      rd 10.1.0.6:50002
      route-target import evpn 50002:50002
      route-target export evpn 50002:50002
      route-target export evpn route-map RM_EXPORT_EVPN         # Отдаем с BORDER LEAF в EVPN-фабрику только дефолт, полученный от FW
      maximum-paths 4 ecmp 4
      neighbor 10.1.2.19 remote-as 65500                        # Соседство с FW в vrf TENANT1
      !
      address-family ipv4
         no neighbor 10.0.1.19 activate
         neighbor 10.1.2.19 activate
         neighbor 10.1.2.19 prefix-list PL_DEFAULT in           # Принимаем только дефолт
         neighbor 10.1.2.19 prefix-list PL_TENANT2 out          # Отдаем только 10.10.10.0/24
         network 20.20.20.0/24
```
</details>

<details>
<summary>Настройкb FW</summary>

```eos
!
hostname FW
!
vlan 110
   name TR-FW-TENANT1
!
vlan 120
   name TR-FW-TENANT2
!
interface Ethernet1
   description TRUNK-2-BLEAF
   switchport trunk allowed vlan 110,120
   switchport mode trunk
!
interface Loopback0
   description ROUTER-ID
   ip address 10.1.0.7/32
!
interface Loopback1
   description 8.8.8.8
   ip address 8.8.8.8/32
!
interface Vlan110
   description TR-FW-TENANT1
   ip address 10.1.2.17/31
!
interface Vlan120
   description TR-FW-TENANT2
   ip address 10.1.2.19/31
!
ip routing
!
router bgp 65500
   router-id 10.1.0.7
   maximum-paths 4 ecmp 4
   neighbor BORDER peer group
   neighbor BORDER remote-as 65004
   neighbor BORDER bfd
   neighbor 10.1.2.16 peer group BORDER
   neighbor 10.1.2.18 peer group BORDER
   !
   address-family ipv4
      neighbor BORDER activate
      neighbor BORDER default-originate always
```
</details>

## 4. Проверка связности
<details>
<summary>LEAF1 / show bgp summary</summary>
  
```eos
LEAF1#show bgp summary
BGP summary information for VRF default
Router identifier 10.1.0.3, local AS number 65001
Neighbor          AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
-------- ----------- ------------- ----------------------- -------------- ---------- ----------
10.1.0.1       65000 Established   L2VPN EVPN              Negotiated              4          4
10.1.0.2       65000 Established   L2VPN EVPN              Negotiated              4          4
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
80 bytes from 10.1.0.4: icmp_seq=1 ttl=63 time=5.10 ms
80 bytes from 10.1.0.4: icmp_seq=2 ttl=63 time=4.14 ms
80 bytes from 10.1.0.4: icmp_seq=3 ttl=63 time=3.76 ms

--- 10.1.0.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 11ms
rtt min/avg/max/mdev = 3.763/4.338/5.104/0.568 ms, ipg/ewma 5.593/4.832 ms
LEAF1#ping 10.1.0.5 source loopback 0 repeat 3
PING 10.1.0.5 (10.1.0.5) from 10.1.0.3 : 72(100) bytes of data.
80 bytes from 10.1.0.5: icmp_seq=1 ttl=63 time=4.96 ms
80 bytes from 10.1.0.5: icmp_seq=2 ttl=63 time=7.78 ms
80 bytes from 10.1.0.5: icmp_seq=3 ttl=63 time=4.27 ms

--- 10.1.0.5 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 15ms
rtt min/avg/max/mdev = 4.276/5.676/7.788/1.521 ms, ipg/ewma 7.666/5.188 ms
LEAF1#ping 10.1.0.6 source loopback 0 repeat 3
PING 10.1.0.6 (10.1.0.6) from 10.1.0.3 : 72(100) bytes of data.
80 bytes from 10.1.0.6: icmp_seq=1 ttl=63 time=5.06 ms
80 bytes from 10.1.0.6: icmp_seq=2 ttl=63 time=3.78 ms
80 bytes from 10.1.0.6: icmp_seq=3 ttl=63 time=4.30 ms

--- 10.1.0.6 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 10ms
rtt min/avg/max/mdev = 3.787/4.387/5.067/0.525 ms, ipg/ewma 5.495/4.832 ms
```
</details>

<details>
<summary>LEAF2 / ping до Lo0 LEAF1,LEAF3-4</summary>
  
```eos
LEAF2#ping 10.1.0.3 source loopback 0 repeat 3
PING 10.1.0.3 (10.1.0.3) from 10.1.0.4 : 72(100) bytes of data.
80 bytes from 10.1.0.3: icmp_seq=1 ttl=63 time=4.87 ms
80 bytes from 10.1.0.3: icmp_seq=2 ttl=63 time=5.60 ms
80 bytes from 10.1.0.3: icmp_seq=3 ttl=63 time=4.14 ms

--- 10.1.0.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 12ms
rtt min/avg/max/mdev = 4.142/4.874/5.606/0.603 ms, ipg/ewma 6.300/4.864 ms
LEAF2#ping 10.1.0.5 source loopback 0 repeat 3
PING 10.1.0.5 (10.1.0.5) from 10.1.0.4 : 72(100) bytes of data.
80 bytes from 10.1.0.5: icmp_seq=1 ttl=63 time=4.17 ms
80 bytes from 10.1.0.5: icmp_seq=2 ttl=63 time=3.95 ms
80 bytes from 10.1.0.5: icmp_seq=3 ttl=63 time=3.98 ms

--- 10.1.0.5 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 10ms
rtt min/avg/max/mdev = 3.955/4.038/4.177/0.111 ms, ipg/ewma 5.305/4.128 ms
LEAF2#ping 10.1.0.6 source loopback 0 repeat 3
PING 10.1.0.6 (10.1.0.6) from 10.1.0.4 : 72(100) bytes of data.
80 bytes from 10.1.0.6: icmp_seq=1 ttl=63 time=4.45 ms
80 bytes from 10.1.0.6: icmp_seq=2 ttl=63 time=4.38 ms
80 bytes from 10.1.0.6: icmp_seq=3 ttl=63 time=3.94 ms

--- 10.1.0.6 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 10ms
rtt min/avg/max/mdev = 3.947/4.262/4.457/0.237 ms, ipg/ewma 5.484/4.385 ms
```
</details>

<details>
<summary>LEAF3 / ping до Lo0 LEAF1-2, LEAF4</summary>
  
```eos
LEAF3#ping 10.1.0.3 source loopback 0 repeat 3
PING 10.1.0.3 (10.1.0.3) from 10.1.0.5 : 72(100) bytes of data.
80 bytes from 10.1.0.3: icmp_seq=1 ttl=63 time=5.18 ms
80 bytes from 10.1.0.3: icmp_seq=2 ttl=63 time=5.38 ms
80 bytes from 10.1.0.3: icmp_seq=3 ttl=63 time=5.37 ms

--- 10.1.0.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 13ms
rtt min/avg/max/mdev = 5.184/5.314/5.385/0.109 ms, ipg/ewma 6.880/5.229 ms
LEAF3#ping 10.1.0.4 source loopback 0 repeat 3
PING 10.1.0.4 (10.1.0.4) from 10.1.0.5 : 72(100) bytes of data.
80 bytes from 10.1.0.4: icmp_seq=1 ttl=63 time=5.04 ms
80 bytes from 10.1.0.4: icmp_seq=2 ttl=63 time=4.44 ms
80 bytes from 10.1.0.4: icmp_seq=3 ttl=63 time=4.17 ms

--- 10.1.0.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 11ms
rtt min/avg/max/mdev = 4.175/4.553/5.043/0.367 ms, ipg/ewma 5.683/4.868 ms
LEAF3#ping 10.1.0.6 source loopback 0 repeat 3
PING 10.1.0.6 (10.1.0.6) from 10.1.0.5 : 72(100) bytes of data.
80 bytes from 10.1.0.6: icmp_seq=1 ttl=63 time=4.77 ms
80 bytes from 10.1.0.6: icmp_seq=2 ttl=63 time=3.96 ms
80 bytes from 10.1.0.6: icmp_seq=3 ttl=63 time=4.27 ms

--- 10.1.0.6 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 11ms
rtt min/avg/max/mdev = 3.961/4.337/4.772/0.337 ms, ipg/ewma 5.590/4.621 ms
```
</details>

<details>
<summary>LEAF4 / ping до Lo0 LEAF1-3</summary>
  
```eos
LEAF4# ping 10.1.0.3 source loopback 0 repeat 3
PING 10.1.0.3 (10.1.0.3) from 10.1.0.6 : 72(100) bytes of data.
80 bytes from 10.1.0.3: icmp_seq=1 ttl=63 time=4.57 ms
80 bytes from 10.1.0.3: icmp_seq=2 ttl=63 time=4.05 ms
80 bytes from 10.1.0.3: icmp_seq=3 ttl=63 time=3.76 ms

--- 10.1.0.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 10ms
rtt min/avg/max/mdev = 3.767/4.131/4.574/0.342 ms, ipg/ewma 5.289/4.416 ms
LEAF4#ping 10.1.0.4 source loopback 0 repeat 3
PING 10.1.0.4 (10.1.0.4) from 10.1.0.6 : 72(100) bytes of data.
80 bytes from 10.1.0.4: icmp_seq=1 ttl=63 time=4.86 ms
80 bytes from 10.1.0.4: icmp_seq=2 ttl=63 time=3.85 ms
80 bytes from 10.1.0.4: icmp_seq=3 ttl=63 time=4.24 ms

--- 10.1.0.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 10ms
rtt min/avg/max/mdev = 3.857/4.323/4.864/0.414 ms, ipg/ewma 5.415/4.676 ms
LEAF4#ping 10.1.0.5 source loopback 0 repeat 3
PING 10.1.0.5 (10.1.0.5) from 10.1.0.6 : 72(100) bytes of data.
80 bytes from 10.1.0.5: icmp_seq=1 ttl=63 time=5.59 ms
80 bytes from 10.1.0.5: icmp_seq=2 ttl=63 time=3.87 ms
80 bytes from 10.1.0.5: icmp_seq=3 ttl=63 time=4.71 ms

--- 10.1.0.5 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 12ms
rtt min/avg/max/mdev = 3.874/4.728/5.593/0.701 ms, ipg/ewma 6.155/5.295 ms
```
</details>

## 5. Проверка работы фабрики
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
```
</details>

<details>
<summary>LEAF1 / route-type 2 / show bgp evpn route-type mac-ip</summary>
   
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
 * >      RD: 10.1.0.3:10020 mac-ip 0200.0000.0002
                                 -                     -       -       0       i
 * >      RD: 10.1.0.3:10020 mac-ip 0200.0000.0002 20.20.20.1
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.5:10010 mac-ip 0200.0001.0003
                                 10.1.0.5              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.5:10010 mac-ip 0200.0001.0003
                                 10.1.0.5              -       100     0       65000 65003 i
 * >Ec    RD: 10.1.0.4:10010 mac-ip 0200.0001.0003 10.10.10.3
                                 10.1.0.4              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.4:10010 mac-ip 0200.0001.0003 10.10.10.3
                                 10.1.0.4              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.5:10010 mac-ip 0200.0001.0003 10.10.10.3
                                 10.1.0.5              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.5:10010 mac-ip 0200.0001.0003 10.10.10.3
                                 10.1.0.5              -       100     0       65000 65003 i
 * >Ec    RD: 10.1.0.5:10020 mac-ip 0200.0002.0003
                                 10.1.0.5              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.5:10020 mac-ip 0200.0002.0003
                                 10.1.0.5              -       100     0       65000 65003 i
 * >Ec    RD: 10.1.0.4:10020 mac-ip 0200.0002.0003 20.20.20.3
                                 10.1.0.4              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.4:10020 mac-ip 0200.0002.0003 20.20.20.3
                                 10.1.0.4              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.5:10020 mac-ip 0200.0002.0003 20.20.20.3
                                 10.1.0.5              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.5:10020 mac-ip 0200.0002.0003 20.20.20.3
                                 10.1.0.5              -       100     0       65000 65003 i
```
</details>

<details>
<summary>LEAF1 / route-type 5 / show bgp evpn route-type ip-prefix 0.0.0.0/0</summary>

```
LEAF1#show bgp evpn | i prefix
 * >Ec    RD: 10.1.0.6:50001 ip-prefix 0.0.0.0/0
 *  ec    RD: 10.1.0.6:50001 ip-prefix 0.0.0.0/0
 * >Ec    RD: 10.1.0.6:50002 ip-prefix 0.0.0.0/0
 *  ec    RD: 10.1.0.6:50002 ip-prefix 0.0.0.0/0

LEAF1#show bgp evpn route-type ip-prefix 0.0.0.0/0
BGP routing table information for VRF default
Router identifier 10.1.0.3, local AS number 65001
BGP routing table entry for ip-prefix 0.0.0.0/0, Route Distinguisher: 10.1.0.6:50001
 Paths: 2 available
  65000 65004 65500
    10.1.0.6 from 10.1.0.2 (10.1.0.2)
      Origin INCOMPLETE, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:50001:50001 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:a4:00:11:d2:94
      VNI: 50001
  65000 65004 65500
    10.1.0.6 from 10.1.0.1 (10.1.0.1)
      Origin INCOMPLETE, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:50001:50001 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:a4:00:11:d2:94
      VNI: 50001
BGP routing table entry for ip-prefix 0.0.0.0/0, Route Distinguisher: 10.1.0.6:50002
 Paths: 2 available
  65000 65004 65500
    10.1.0.6 from 10.1.0.1 (10.1.0.1)
      Origin INCOMPLETE, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP head, ECMP, best, ECMP contributor
      Extended Community: Route-Target-AS:50002:50002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:a4:00:11:d2:94
      VNI: 50002
  65000 65004 65500
    10.1.0.6 from 10.1.0.2 (10.1.0.2)
      Origin INCOMPLETE, metric -, localpref 100, weight 0, tag 0, valid, external, ECMP, ECMP contributor
      Extended Community: Route-Target-AS:50002:50002 TunnelEncap:tunnelTypeVxlan EvpnRouterMac:50:a4:00:11:d2:94
      VNI: 50002
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
  10    0200.0000.0001    DYNAMIC     Et3        1       0:02:28 ago
  10    0200.0001.0003    DYNAMIC     Vx1        1       0:06:45 ago
  20    0000.2222.3333    STATIC      Cpu
  20    0200.0000.0002    DYNAMIC     Et4        1       0:01:58 ago
  20    0200.0002.0003    DYNAMIC     Vx1        1       0:06:45 ago
4093    0000.2222.3333    STATIC      Cpu
4093    5079.f2b7.ebe3    DYNAMIC     Vx1        1       0:06:45 ago
4093    50a4.0011.d294    DYNAMIC     Vx1        1       0:06:49 ago
4093    50a4.2945.b244    DYNAMIC     Vx1        1       0:06:45 ago
4094    0000.2222.3333    STATIC      Cpu
4094    5079.f2b7.ebe3    DYNAMIC     Vx1        1       0:06:45 ago
4094    50a4.0011.d294    DYNAMIC     Vx1        1       0:06:47 ago
4094    50a4.2945.b244    DYNAMIC     Vx1        1       0:06:45 ago
```
</details>

<details>
<summary>LEAF4 / show ip route vrf all</summary>

```
LEAF4#show ip route vrf all

VRF: default
Gateway of last resort is not set

 B E      10.1.0.1/32 [200/0] via 10.1.2.12, Ethernet1
 B E      10.1.0.2/32 [200/0] via 10.1.2.14, Ethernet2
 B E      10.1.0.3/32 [200/0] via 10.1.2.12, Ethernet1
                              via 10.1.2.14, Ethernet2
 B E      10.1.0.4/32 [200/0] via 10.1.2.12, Ethernet1
                              via 10.1.2.14, Ethernet2
 B E      10.1.0.5/32 [200/0] via 10.1.2.12, Ethernet1
                              via 10.1.2.14, Ethernet2
 C        10.1.0.6/32 is directly connected, Loopback0
 C        10.1.2.12/31 is directly connected, Ethernet1
 C        10.1.2.14/31 is directly connected, Ethernet2


VRF: TENANT1
Gateway of last resort:
 B E      0.0.0.0/0 [200/0] via 10.1.2.17, Vlan110

 C        10.1.2.16/31 is directly connected, Vlan110
 B E      10.10.10.1/32 [200/0] via VTEP 10.1.0.3 VNI 50001 router-mac 50:2a:2f:4a:0c:3a local-interface Vxlan1
 B E      10.10.10.3/32 [200/0] via VTEP 10.1.0.5 VNI 50001 router-mac 50:a4:29:45:b2:44 local-interface Vxlan1
                                via VTEP 10.1.0.4 VNI 50001 router-mac 50:79:f2:b7:eb:e3 local-interface Vxlan1
 S        10.10.10.0/24 is directly connected, Null0

VRF: TENANT2
Gateway of last resort:
 B E      0.0.0.0/0 [200/0] via 10.1.2.19, Vlan120

 C        10.1.2.18/31 is directly connected, Vlan120
 B E      20.20.20.1/32 [200/0] via VTEP 10.1.0.3 VNI 50002 router-mac 50:2a:2f:4a:0c:3a local-interface Vxlan1
 B E      20.20.20.3/32 [200/0] via VTEP 10.1.0.5 VNI 50002 router-mac 50:a4:29:45:b2:44 local-interface Vxlan1
                                via VTEP 10.1.0.4 VNI 50002 router-mac 50:79:f2:b7:eb:e3 local-interface Vxlan1
 S        20.20.20.0/24 is directly connected, Null0
```
</details>

<details>
<summary>FW / show ip route</summary>

```
FW#show ip route

VRF: default
Gateway of last resort is not set

 C        8.8.8.8/32 is directly connected, Loopback1
 C        10.1.0.7/32 is directly connected, Loopback0
 C        10.1.2.16/31 is directly connected, Vlan110
 C        10.1.2.18/31 is directly connected, Vlan120
 B E      10.10.10.0/24 [200/0] via 10.1.2.16, Vlan110
 B E      20.20.20.0/24 [200/0] via 10.1.2.18, Vlan120
```
</details>

<details>
<summary>ping SRV1 -> SRV3, 8.8.8.8</summary>

```
SRV1:/# ping -c 3 20.20.20.3
PING 20.20.20.3 (20.20.20.3): 56 data bytes
64 bytes from 20.20.20.3: seq=0 ttl=59 time=71.412 ms
64 bytes from 20.20.20.3: seq=1 ttl=59 time=31.888 ms
64 bytes from 20.20.20.3: seq=2 ttl=59 time=30.876 ms

--- 20.20.20.3 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 30.876/44.725/71.412 ms
SRV1:/# ping -c 3 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=62 time=13.797 ms
64 bytes from 8.8.8.8: seq=1 ttl=62 time=12.160 ms
64 bytes from 8.8.8.8: seq=2 ttl=62 time=12.847 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 12.160/12.934/13.797 ms
```
</details>

В дампе на LEAF4 видим, что запрос 10.10.10.1 -> 20.20.20.3, проходящий в разных VxLAN с разными L3VNI (50001 - от LEAF1 до LEAF4, 50002 - от LEAF4 до LEAF3). Ответ проходят аналогичным образом.
<img width="928" height="269" alt="image" src="https://github.com/user-attachments/assets/140ba28e-d1e9-4660-ac52-8ee63765e09c" />
<img width="927" height="269" alt="image" src="https://github.com/user-attachments/assets/e5a310e4-4911-4641-aff8-e6051a05f7ed" />

При ping 8.8.8.8 видим, соответственно, запрос/ответ.
<img width="918" height="306" alt="image" src="https://github.com/user-attachments/assets/e2fcf5bc-ca4a-4df4-88b8-2647cf3a96d8" />

<details>
<summary>ping SRV2 -> SRV3, 8.8.8.8</summary>

```
SRV2:/# ping 10.10.10.3 -c 3
PING 10.10.10.3 (10.10.10.3): 56 data bytes
64 bytes from 10.10.10.3: seq=0 ttl=59 time=28.307 ms
64 bytes from 10.10.10.3: seq=1 ttl=59 time=30.267 ms
64 bytes from 10.10.10.3: seq=2 ttl=59 time=25.047 ms

--- 10.10.10.3 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 25.047/27.873/30.267 ms
SRV2:/# ping 8.8.8.8 -c 3
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=62 time=12.251 ms
64 bytes from 8.8.8.8: seq=1 ttl=62 time=12.159 ms
64 bytes from 8.8.8.8: seq=2 ttl=62 time=11.118 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 11.118/11.842/12.251 ms
```
</details>
Проверки с SRV2 также проходят без потерь.

[![Тестирование отключения линка LEAF22 -- FW2](https://img.youtube.com/vi/a_ydqCp0GHg/default.jpg)](https://youtu.be/a_ydqCp0GHg&vq=hd720)

## 6. Конфигурации устройств фабрики
[Конфигурация SPINE1](./configs/spine1.conf)<br>
[Конфигурация SPINE2](./configs/spine2.conf)<br>
[Конфигурация LEAF1](./configs/leaf1.conf)<br>
[Конфигурация LEAF2](./configs/leaf2.conf)<br>
[Конфигурация LEAF3](./configs/leaf3.conf)<br>
[Конфигурация LEAF4](./configs/leaf4.conf)<br>
[Конфигурация FW](./configs/fw.conf)<br>
