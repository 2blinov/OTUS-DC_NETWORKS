# Проектная работа "Проектирование мультитенантной сетевой фабрики EVPN Multi-Site для двух ЦОД с контролируемым взаимодействием сетевых сегментов и централизованной защитой периметра средствами МСЭ"

## Оглавление:
1. [Подготовка стенда](#1-подготовка-стенда)
2. [Разработка адресного плана](#2-разработка-адресного-плана)
3. [Настройка стенда](#3-настройка-стенда)
4. [Проверка связности](#4-проверка-связности)
5. [Проверка работы фабрики](#5-проверка-работы-фабрики)
6. [Политика прохождения трафика](#6-политика-прохождения-трафика)
7. [Политика фильтрации трафика](#7-политика-фильтрации-трафика)
8. [Проверка отказоустойчивости](#8-проверка-отказоустойчивости)
9. [Конфигурации устройств фабрики](#9-конфигурации-устройств-фабрики)
10. [Презентация](#10-презентация)

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
| BGW2 – SPINE21 | 10.2.2.0/31  |
| BGW2 – SPINE22 | 10.2.2.2/31  |
| SPINE21 – LEAF21 | 10.2.2.4/31  |
| SPINE21 – LEAF22 | 10.2.2.6/31  |
| SPINE22 – LEAF21 | 10.2.2.8/31  |
| SPINE22 – LEAF22 | 10.2.2.10/31 |
| LEAF22 – FW2 (TENANT1) | 10.2.2.12/31 |
| LEAF22 – FW2 (TENANT2) | 10.2.2.14/31 |

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
<summary>Контекст: Настройки SPINE</summary>

```eos
ip routing
!
peer-filter LEAFS-AS-FILTER
   10 match as-range 65001 result accept
!
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
<summary>Контекст: Настройкb BGW</summary>

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
<summary>Контекст: Настройка FW</summary>

```eos
hostname fw1
domainname localdomain
log syslog informational
service integrated-vtysh-config
!
ip prefix-list PL-IN-TENANT1 seq 10 permit 10.10.10.0/24 le 32
ip prefix-list PL-IN-TENANT2 seq 10 permit 10.10.20.0/24 le 32
!
route-map RM-IN-TENANT1 permit 10
 match ip address prefix-list PL-IN-TENANT1
exit
!
route-map RM-IN-TENANT2 permit 10
 match ip address prefix-list PL-IN-TENANT2
exit
!
vrf FW
exit-vrf
!
interface lo
 ip address 10.0.1.1/32
exit
!
router bgp 65101 vrf FW
 bgp router-id 10.0.1.1
 no bgp ebgp-requires-policy
 neighbor 10.0.2.1 remote-as 65100
 neighbor 10.0.2.1 bfd
 neighbor 10.0.2.1 bfd profile P-BFD
 neighbor 10.1.2.12 remote-as 65001
 neighbor 10.1.2.12 bfd
 neighbor 10.1.2.12 bfd profile P-BFD
 neighbor 10.1.2.14 remote-as 65001
 neighbor 10.1.2.14 bfd
 neighbor 10.1.2.14 bfd profile P-BFD
 !
 address-family ipv4 unicast
  network 10.0.1.1/32
  network 10.0.2.0/31
  network 10.1.2.12/31
  network 10.1.2.14/31
  neighbor 10.1.2.12 soft-reconfiguration inbound
  neighbor 10.1.2.12 route-map RM-IN-TENANT1 in
  neighbor 10.1.2.14 soft-reconfiguration inbound
  neighbor 10.1.2.14 route-map RM-IN-TENANT2 in
 exit-address-family
exit
!
bfd
 profile P-BFD
 exit
 !
exit
!
end

```
</details>

## 4. Проверка связности
Сайт 1:
<details>
<summary>LEAF11 / show bgp summary vrf all</summary>
  
```eos
LEAF11#show bgp summary vrf all 
BGP summary information for VRF default
Router identifier 10.1.0.3, local AS number 65001
Neighbor          AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc   NLRI Adv
-------- ----------- ------------- ----------------------- -------------- ---------- ---------- ----------
10.1.0.1       65001 Established   L2VPN EVPN              Negotiated             12         12          6
10.1.0.2       65001 Established   L2VPN EVPN              Negotiated             12         12          6
10.1.2.4       65001 Established   IPv4 Unicast            Negotiated             22         22          3
10.1.2.8       65001 Established   IPv4 Unicast            Negotiated             22         22          3
```
</details>

<details>
<summary>LEAF12 / show bgp summary vrf all</summary>
  
```eos
LEAF12#show bgp summary vrf all
BGP summary information for VRF default
Router identifier 10.1.0.4, local AS number 65001
Neighbor           AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc   NLRI Adv
--------- ----------- ------------- ----------------------- -------------- ---------- ---------- ----------
10.1.0.1        65001 Established   L2VPN EVPN              Negotiated             16         16          2
10.1.0.2        65001 Established   L2VPN EVPN              Negotiated             16         16          2
10.1.2.6        65001 Established   IPv4 Unicast            Negotiated             22         22          3
10.1.2.10       65001 Established   IPv4 Unicast            Negotiated             22         22          3

BGP summary information for VRF TENANT1
Router identifier 10.1.0.4, local AS number 65001
Neighbor           AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc   NLRI Adv
--------- ----------- ------------- ----------------------- -------------- ---------- ---------- ----------
10.1.2.13       65101 Established   IPv4 Unicast            Negotiated             13          1          2

BGP summary information for VRF TENANT2
Router identifier 10.1.0.4, local AS number 65001
Neighbor           AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc   NLRI Adv
--------- ----------- ------------- ----------------------- -------------- ---------- ---------- ----------
10.1.2.15       65101 Established   IPv4 Unicast            Negotiated             13          1          2
```
</details>

<details>
<summary>BGW1 / show bgp summary vrf all</summary>
  
```eos
BGW1#show bgp summary vrf all 
BGP summary information for VRF default
Router identifier 10.1.0.0, local AS number 65001
Neighbor          AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc   NLRI Adv
-------- ----------- ------------- ----------------------- -------------- ---------- ---------- ----------
10.0.0.3       65002 Established   IPv4 Unicast            Negotiated             13         13         13
10.0.0.3       65002 Established   L2VPN EVPN              Negotiated              9          9          9
10.1.0.1       65001 Established   L2VPN EVPN              Negotiated              8          8         10
10.1.0.2       65001 Established   L2VPN EVPN              Negotiated              8          8         10
10.1.2.1       65001 Established   IPv4 Unicast            Negotiated              8          8         17
10.1.2.3       65001 Established   IPv4 Unicast            Negotiated              8          8         17
```
</details>

<details>
<summary> LEAF11 / show ip bgp</summary>
  
```eos
LEAF11#show ip bgp 
BGP routing table information for VRF default
Router identifier 10.1.0.3, local AS number 65001
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast, q - Pending FIB install
                    % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >Ec    10.0.0.0/32            10.1.2.0              0       -          100     0       i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 *  ec    10.0.0.0/32            10.1.2.2              0       -          100     0       i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 * >Ec    10.0.0.1/32            10.0.0.3              0       -          100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 *  ec    10.0.0.1/32            10.0.0.3              0       -          100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 * >Ec    10.0.0.2/31            10.1.2.2              0       -          100     0       i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 *  ec    10.0.0.2/31            10.1.2.0              0       -          100     0       i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 * >Ec    10.1.0.0/32            10.1.2.0              0       -          100     0       i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 *  ec    10.1.0.0/32            10.1.2.2              0       -          100     0       i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 * >      10.1.0.1/32            10.1.2.4              0       -          100     0       i
 * >      10.1.0.2/32            10.1.2.8              0       -          100     0       i
 * >      10.1.0.3/32            -                     -       -          -       0       i
 * >Ec    10.1.0.4/32            10.1.2.7              0       -          100     0       i Or-ID: 10.1.0.4 C-LST: 10.1.0.1 
 *  ec    10.1.0.4/32            10.1.2.11             0       -          100     0       i Or-ID: 10.1.0.4 C-LST: 10.1.0.1 
 * >      10.1.2.0/31            10.1.2.4              0       -          100     0       i
 *        10.1.2.0/31            10.1.2.2              0       -          100     0       i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 * >      10.1.2.2/31            10.1.2.8              0       -          100     0       i
 *        10.1.2.2/31            10.1.2.0              0       -          100     0       i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 * >      10.1.2.4/31            -                     -       -          -       0       i
 *        10.1.2.4/31            10.1.2.4              0       -          100     0       i
 * >      10.1.2.6/31            10.1.2.4              0       -          100     0       i
 *        10.1.2.6/31            10.1.2.11             0       -          100     0       i Or-ID: 10.1.0.4 C-LST: 10.1.0.1 
 * >      10.1.2.8/31            -                     -       -          -       0       i
 *        10.1.2.8/31            10.1.2.8              0       -          100     0       i
 * >      10.1.2.10/31           10.1.2.8              0       -          100     0       i
 *        10.1.2.10/31           10.1.2.7              0       -          100     0       i Or-ID: 10.1.0.4 C-LST: 10.1.0.1 
 * >Ec    10.2.0.0/32            10.0.0.3              0       -          100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 *  ec    10.2.0.0/32            10.0.0.3              0       -          100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 * >Ec    10.2.0.1/32            10.0.0.3              0       -          100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 *  ec    10.2.0.1/32            10.0.0.3              0       -          100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 * >Ec    10.2.0.2/32            10.0.0.3              0       -          100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 *  ec    10.2.0.2/32            10.0.0.3              0       -          100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 * >Ec    10.2.0.3/32            10.0.0.3              0       -          100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 *  ec    10.2.0.3/32            10.0.0.3              0       -          100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 * >Ec    10.2.0.4/32            10.0.0.3              0       -          100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 *  ec    10.2.0.4/32            10.0.0.3              0       -          100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 * >Ec    10.2.2.0/31            10.0.0.3              0       -          100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 *  ec    10.2.2.0/31            10.0.0.3              0       -          100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 * >Ec    10.2.2.2/31            10.0.0.3              0       -          100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 *  ec    10.2.2.2/31            10.0.0.3              0       -          100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 * >Ec    10.2.2.4/31            10.0.0.3              0       -          100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 *  ec    10.2.2.4/31            10.0.0.3              0       -          100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 * >Ec    10.2.2.6/31            10.0.0.3              0       -          100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 *  ec    10.2.2.6/31            10.0.0.3              0       -          100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 * >Ec    10.2.2.8/31            10.0.0.3              0       -          100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 *  ec    10.2.2.8/31            10.0.0.3              0       -          100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 * >Ec    10.2.2.10/31           10.0.0.3              0       -          100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 *  ec    10.2.2.10/31           10.0.0.3              0       -          100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1
```
</details>

<details>
<summary>LEAF11 / ping до Lo0 LEAF12, BGW1, Lo1 BGW1</summary>
  
```eos
LEAF11#ping 10.1.0.4 source loopback 0 repeat 3
PING 10.1.0.4 (10.1.0.4) from 10.1.0.3 : 72(100) bytes of data.
80 bytes from 10.1.0.4: icmp_seq=1 ttl=63 time=0.520 ms
80 bytes from 10.1.0.4: icmp_seq=2 ttl=63 time=0.322 ms
80 bytes from 10.1.0.4: icmp_seq=3 ttl=63 time=0.317 ms

--- 10.1.0.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1ms
rtt min/avg/max/mdev = 0.317/0.386/0.520/0.094 ms, ipg/ewma 0.698/0.473 ms
LEAF11#ping 10.1.0.0 source loopback 0 repeat 3
PING 10.1.0.0 (10.1.0.0) from 10.1.0.3 : 72(100) bytes of data.
80 bytes from 10.1.0.0: icmp_seq=1 ttl=63 time=0.640 ms
80 bytes from 10.1.0.0: icmp_seq=2 ttl=63 time=0.315 ms
80 bytes from 10.1.0.0: icmp_seq=3 ttl=63 time=0.325 ms

--- 10.1.0.0 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2ms
rtt min/avg/max/mdev = 0.315/0.426/0.640/0.150 ms, ipg/ewma 1.001/0.565 ms
LEAF11#ping 10.0.0.0 source loopback 0 repeat 3
PING 10.0.0.0 (10.0.0.0) from 10.1.0.3 : 72(100) bytes of data.
80 bytes from 10.0.0.0: icmp_seq=1 ttl=63 time=0.440 ms
80 bytes from 10.0.0.0: icmp_seq=2 ttl=63 time=0.346 ms
80 bytes from 10.0.0.0: icmp_seq=3 ttl=63 time=0.354 ms

--- 10.0.0.0 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1ms
rtt min/avg/max/mdev = 0.346/0.380/0.440/0.042 ms, ipg/ewma 0.507/0.419 ms
```
</details>

Сайт 2:
<details>
<summary>LEAF21 / show bgp summary vrf all</summary>
  
```eos
LEAF21#show bgp summary vrf all 
BGP summary information for VRF default
Router identifier 10.2.0.3, local AS number 65002
Neighbor          AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc   NLRI Adv
-------- ----------- ------------- ----------------------- -------------- ---------- ---------- ----------
10.2.0.1       65002 Established   L2VPN EVPN              Negotiated             12         12          6
10.2.0.2       65002 Established   L2VPN EVPN              Negotiated             12         12          6
10.2.2.4       65002 Established   IPv4 Unicast            Negotiated             22         22          3
10.2.2.8       65002 Established   IPv4 Unicast            Negotiated             22         22          3
```
</details>

<details>
<summary>LEAF22 / show bgp summary vrf all</summary>
  
```eos
LEAF22#show bgp summary vrf all 
BGP summary information for VRF default
Router identifier 10.2.0.4, local AS number 65002
Neighbor           AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc   NLRI Adv
--------- ----------- ------------- ----------------------- -------------- ---------- ---------- ----------
10.2.0.1        65002 Established   L2VPN EVPN              Negotiated             16         16          2
10.2.0.2        65002 Established   L2VPN EVPN              Negotiated             16         16          2
10.2.2.6        65002 Established   IPv4 Unicast            Negotiated             22         22          3
10.2.2.10       65002 Established   IPv4 Unicast            Negotiated             22         22          3

BGP summary information for VRF TENANT1
Router identifier 10.2.0.4, local AS number 65002
Neighbor           AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc   NLRI Adv
--------- ----------- ------------- ----------------------- -------------- ---------- ---------- ----------
10.2.2.13       65102 Established   IPv4 Unicast            Negotiated             15          1          2

BGP summary information for VRF TENANT2
Router identifier 10.2.0.4, local AS number 65002
Neighbor           AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc   NLRI Adv
--------- ----------- ------------- ----------------------- -------------- ---------- ---------- ----------
10.2.2.15       65102 Established   IPv4 Unicast            Negotiated             15          1          2
```
</details>

<details>
<summary>BGW2 / show bgp summary vrf all</summary>
  
```eos
BGW2#show bgp summary vrf all 
BGP summary information for VRF default
Router identifier 10.2.0.0, local AS number 65002
Neighbor          AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc   NLRI Adv
-------- ----------- ------------- ----------------------- -------------- ---------- ---------- ----------
10.0.0.2       65001 Established   IPv4 Unicast            Negotiated             13         13         13
10.0.0.2       65001 Established   L2VPN EVPN              Negotiated              9          9          9
10.2.0.1       65002 Established   L2VPN EVPN              Negotiated              8          8         10
10.2.0.2       65002 Established   L2VPN EVPN              Negotiated              8          8         10
10.2.2.1       65002 Established   IPv4 Unicast            Negotiated              8          8         17
10.2.2.3       65002 Established   IPv4 Unicast            Negotiated              8          8         17
```
</details>

<details>
<summary> LEAF21 / show ip bgp</summary>
  
```eos
LEAF21#show ip bgp
BGP routing table information for VRF default
Router identifier 10.2.0.3, local AS number 65002
Route status codes: s - suppressed contributor, * - valid, > - active, E - ECMP head, e - ECMP
                    S - Stale, c - Contributing to ECMP, b - backup, L - labeled-unicast, q - Pending FIB install
                    % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI Origin Validation codes: V - valid, I - invalid, U - unknown
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  AIGP       LocPref Weight  Path
 * >Ec    10.0.0.0/32            10.0.0.2              0       -          100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 *  ec    10.0.0.0/32            10.0.0.2              0       -          100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 * >Ec    10.0.0.1/32            10.2.2.0              0       -          100     0       i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 *  ec    10.0.0.1/32            10.2.2.2              0       -          100     0       i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 * >Ec    10.0.0.2/31            10.2.2.0              0       -          100     0       i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 *  ec    10.0.0.2/31            10.2.2.2              0       -          100     0       i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 * >Ec    10.1.0.0/32            10.0.0.2              0       -          100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 *  ec    10.1.0.0/32            10.0.0.2              0       -          100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 * >Ec    10.1.0.1/32            10.0.0.2              0       -          100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 *  ec    10.1.0.1/32            10.0.0.2              0       -          100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 * >Ec    10.1.0.2/32            10.0.0.2              0       -          100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 *  ec    10.1.0.2/32            10.0.0.2              0       -          100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 * >Ec    10.1.0.3/32            10.0.0.2              0       -          100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 *  ec    10.1.0.3/32            10.0.0.2              0       -          100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 * >Ec    10.1.0.4/32            10.0.0.2              0       -          100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 *  ec    10.1.0.4/32            10.0.0.2              0       -          100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 * >Ec    10.1.2.0/31            10.0.0.2              0       -          100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 *  ec    10.1.2.0/31            10.0.0.2              0       -          100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 * >Ec    10.1.2.2/31            10.0.0.2              0       -          100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 *  ec    10.1.2.2/31            10.0.0.2              0       -          100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 * >Ec    10.1.2.4/31            10.0.0.2              0       -          100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 *  ec    10.1.2.4/31            10.0.0.2              0       -          100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 * >Ec    10.1.2.6/31            10.0.0.2              0       -          100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 *  ec    10.1.2.6/31            10.0.0.2              0       -          100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 * >Ec    10.1.2.8/31            10.0.0.2              0       -          100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 *  ec    10.1.2.8/31            10.0.0.2              0       -          100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 * >Ec    10.1.2.10/31           10.0.0.2              0       -          100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 *  ec    10.1.2.10/31           10.0.0.2              0       -          100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 * >Ec    10.2.0.0/32            10.2.2.0              0       -          100     0       i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 *  ec    10.2.0.0/32            10.2.2.2              0       -          100     0       i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 * >      10.2.0.1/32            10.2.2.4              0       -          100     0       i
 * >      10.2.0.2/32            10.2.2.8              0       -          100     0       i
 * >      10.2.0.3/32            -                     -       -          -       0       i
 * >Ec    10.2.0.4/32            10.2.2.7              0       -          100     0       i Or-ID: 10.2.0.4 C-LST: 10.2.0.1 
 *  ec    10.2.0.4/32            10.2.2.11             0       -          100     0       i Or-ID: 10.2.0.4 C-LST: 10.2.0.1 
 * >      10.2.2.0/31            10.2.2.4              0       -          100     0       i
 *        10.2.2.0/31            10.2.2.2              0       -          100     0       i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 * >      10.2.2.2/31            10.2.2.8              0       -          100     0       i
 *        10.2.2.2/31            10.2.2.0              0       -          100     0       i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 * >      10.2.2.4/31            -                     -       -          -       0       i
 *        10.2.2.4/31            10.2.2.4              0       -          100     0       i
 * >      10.2.2.6/31            10.2.2.4              0       -          100     0       i
 *        10.2.2.6/31            10.2.2.11             0       -          100     0       i Or-ID: 10.2.0.4 C-LST: 10.2.0.1 
 * >      10.2.2.8/31            -                     -       -          -       0       i
 *        10.2.2.8/31            10.2.2.8              0       -          100     0       i
 * >      10.2.2.10/31           10.2.2.8              0       -          100     0       i
 *        10.2.2.10/31           10.2.2.7              0       -          100     0       i Or-ID: 10.2.0.4 C-LST: 10.2.0.1
```
</details>

<details>
<summary>LEAF21 / ping до Lo0 LEAF22, BGW2, Lo1 BGW2</summary>
  
```eos
LEAF21#ping 10.2.0.4 source loopback 0 repeat 3
PING 10.2.0.4 (10.2.0.4) from 10.2.0.3 : 72(100) bytes of data.
80 bytes from 10.2.0.4: icmp_seq=1 ttl=63 time=0.666 ms
80 bytes from 10.2.0.4: icmp_seq=2 ttl=63 time=0.341 ms
80 bytes from 10.2.0.4: icmp_seq=3 ttl=63 time=0.302 ms

--- 10.2.0.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2ms
rtt min/avg/max/mdev = 0.302/0.436/0.666/0.163 ms, ipg/ewma 1.001/0.585 ms
LEAF21#ping 10.2.0.0 source loopback 0 repeat 3
PING 10.2.0.0 (10.2.0.0) from 10.2.0.3 : 72(100) bytes of data.
80 bytes from 10.2.0.0: icmp_seq=1 ttl=63 time=0.654 ms
80 bytes from 10.2.0.0: icmp_seq=2 ttl=63 time=0.381 ms
80 bytes from 10.2.0.0: icmp_seq=3 ttl=63 time=0.311 ms

--- 10.2.0.0 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2ms
rtt min/avg/max/mdev = 0.311/0.448/0.654/0.147 ms, ipg/ewma 1.001/0.581 ms
LEAF21#ping 10.0.0.1 source loopback 0 repeat 3
PING 10.0.0.1 (10.0.0.1) from 10.2.0.3 : 72(100) bytes of data.
80 bytes from 10.0.0.1: icmp_seq=1 ttl=63 time=0.718 ms
80 bytes from 10.0.0.1: icmp_seq=2 ttl=63 time=0.351 ms
80 bytes from 10.0.0.1: icmp_seq=3 ttl=63 time=0.303 ms

--- 10.0.0.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2ms
rtt min/avg/max/mdev = 0.303/0.457/0.718/0.185 ms, ipg/ewma 1.001/0.626 ms
```
</details>

## 5. Проверка работы фабрики
<details>
<summary>LEAF11 / route-type 2 / show bgp evpn route-type mac-ip</summary>
   
```
LEAF11#show bgp evpn route-type mac-ip 
BGP routing table information for VRF default
Router identifier 10.1.0.3, local AS number 65001
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.1.0.3:20 mac-ip aac1.ab2d.c08f
                                 -                     -       -       0       i
 * >      RD: 10.1.0.3:20 mac-ip aac1.ab2d.c08f 10.10.20.1
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.0:10 mac-ip aac1.ab46.07d3
                                 10.0.0.0              -       100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 *  ec    RD: 10.1.0.0:10 mac-ip aac1.ab46.07d3
                                 10.0.0.0              -       100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 * >Ec    RD: 10.1.0.0:10 mac-ip aac1.ab46.07d3 10.10.10.2
                                 10.0.0.0              -       100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 *  ec    RD: 10.1.0.0:10 mac-ip aac1.ab46.07d3 10.10.10.2
                                 10.0.0.0              -       100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 * >      RD: 10.1.0.3:10 mac-ip aac1.ab6c.483a
                                 -                     -       -       0       i
 * >      RD: 10.1.0.3:10 mac-ip aac1.ab6c.483a 10.10.10.1
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.0:20 mac-ip aac1.abc6.faf7
                                 10.0.0.0              -       100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 *  ec    RD: 10.1.0.0:20 mac-ip aac1.abc6.faf7
                                 10.0.0.0              -       100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 * >Ec    RD: 10.1.0.0:20 mac-ip aac1.abc6.faf7 10.10.20.2
                                 10.0.0.0              -       100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 *  ec    RD: 10.1.0.0:20 mac-ip aac1.abc6.faf7 10.10.20.2
                                 10.0.0.0              -       100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
```
</details>

<details>
<summary>LEAF11 / route-type 5 / show bgp evpn route-type ip-prefix</summary>

```
LEAF11#show bgp evpn route-type ip-prefix
BGP routing table information for VRF default
Router identifier 10.1.0.3, local AS number 65001
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.1.0.4:50001 ip-prefix 0.0.0.0/0
                                 10.1.0.4              -       100     0       65101 65100 i Or-ID: 10.1.0.4 C-LST: 10.1.0.1 
 *  ec    RD: 10.1.0.4:50001 ip-prefix 0.0.0.0/0
                                 10.1.0.4              -       100     0       65101 65100 i Or-ID: 10.1.0.4 C-LST: 10.1.0.1 
 * >Ec    RD: 10.1.0.4:50002 ip-prefix 0.0.0.0/0
                                 10.1.0.4              -       100     0       65101 65100 i Or-ID: 10.1.0.4 C-LST: 10.1.0.1 
 *  ec    RD: 10.1.0.4:50002 ip-prefix 0.0.0.0/0
                                 10.1.0.4              -       100     0       65101 65100 i Or-ID: 10.1.0.4 C-LST: 10.1.0.1 
 * >Ec    RD: 10.2.0.3:1 ip-prefix 0.0.0.0/0
                                 10.0.0.0              -       100     0       65002 65102 65100 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 *  ec    RD: 10.2.0.3:1 ip-prefix 0.0.0.0/0
                                 10.0.0.0              -       100     0       65002 65102 65100 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 * >Ec    RD: 10.2.0.3:2 ip-prefix 0.0.0.0/0
                                 10.0.0.0              -       100     0       65002 65102 65100 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 *  ec    RD: 10.2.0.3:2 ip-prefix 0.0.0.0/0
                                 10.0.0.0              -       100     0       65002 65102 65100 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 * >Ec    RD: 65001:102 ip-prefix 10.10.20.0/24
                                 10.0.0.0              -       100     0       i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 *  ec    RD: 65001:102 ip-prefix 10.10.20.0/24
                                 10.0.0.0              -       100     0       i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 * >Ec    RD: 65002:102 ip-prefix 10.10.20.0/24
                                 10.0.0.0              -       100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
 *  ec    RD: 65002:102 ip-prefix 10.10.20.0/24
                                 10.0.0.0              -       100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1 
```
</details>

<details>
<summary>LEAF11 / show mac address-table</summary>

```
LEAF11#show mac address-table 
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  10    aac1.ab46.07d3    DYNAMIC     Vx1        1       8:18:12 ago
  10    aac1.ab6c.483a    DYNAMIC     Et3        1       8:18:12 ago
  20    aac1.ab2d.c08f    DYNAMIC     Et4        1       8:19:57 ago
  20    aac1.abc6.faf7    DYNAMIC     Vx1        1       8:18:53 ago
Total Mac Addresses for this criterion: 4
```
</details>

<details>
<summary>LEAF11 / show ip route vrf all</summary>

```
LEAF11#show ip route vrf all

VRF: default
Gateway of last resort is not set

 B I      10.0.0.0/32 [200/0]
           via 10.1.2.4, Ethernet1
           via 10.1.2.8, Ethernet2
 B I      10.0.0.1/32 [200/0]
           via 10.1.2.4, Ethernet1
           via 10.1.2.8, Ethernet2
 B I      10.0.0.2/31 [200/0]
           via 10.1.2.4, Ethernet1
           via 10.1.2.8, Ethernet2
 B I      10.1.0.0/32 [200/0]
           via 10.1.2.4, Ethernet1
           via 10.1.2.8, Ethernet2
 B I      10.1.0.1/32 [200/0]
           via 10.1.2.4, Ethernet1
 B I      10.1.0.2/32 [200/0]
           via 10.1.2.8, Ethernet2
 C        10.1.0.3/32
           directly connected, Loopback0
 B I      10.1.0.4/32 [200/0]
           via 10.1.2.4, Ethernet1
           via 10.1.2.8, Ethernet2
 B I      10.1.2.0/31 [200/0]
           via 10.1.2.4, Ethernet1
 B I      10.1.2.2/31 [200/0]
           via 10.1.2.8, Ethernet2
 C        10.1.2.4/31
           directly connected, Ethernet1
 B I      10.1.2.6/31 [200/0]
           via 10.1.2.4, Ethernet1
 C        10.1.2.8/31
           directly connected, Ethernet2
 B I      10.1.2.10/31 [200/0]
           via 10.1.2.8, Ethernet2
 B I      10.2.0.0/32 [200/0]
           via 10.1.2.4, Ethernet1
           via 10.1.2.8, Ethernet2
 B I      10.2.0.1/32 [200/0]
           via 10.1.2.4, Ethernet1
           via 10.1.2.8, Ethernet2
 B I      10.2.0.2/32 [200/0]
           via 10.1.2.4, Ethernet1
           via 10.1.2.8, Ethernet2
 B I      10.2.0.3/32 [200/0]
           via 10.1.2.4, Ethernet1
           via 10.1.2.8, Ethernet2
 B I      10.2.0.4/32 [200/0]
           via 10.1.2.4, Ethernet1
           via 10.1.2.8, Ethernet2
 B I      10.2.2.0/31 [200/0]
           via 10.1.2.4, Ethernet1
           via 10.1.2.8, Ethernet2
 B I      10.2.2.2/31 [200/0]
           via 10.1.2.4, Ethernet1
           via 10.1.2.8, Ethernet2
 B I      10.2.2.4/31 [200/0]
           via 10.1.2.4, Ethernet1
           via 10.1.2.8, Ethernet2
 B I      10.2.2.6/31 [200/0]
           via 10.1.2.4, Ethernet1
           via 10.1.2.8, Ethernet2
 B I      10.2.2.8/31 [200/0]
           via 10.1.2.4, Ethernet1
           via 10.1.2.8, Ethernet2
 B I      10.2.2.10/31 [200/0]
           via 10.1.2.4, Ethernet1
           via 10.1.2.8, Ethernet2

VRF: TENANT1
Gateway of last resort:
 B I      0.0.0.0/0 [200/0]
           via VTEP 10.1.0.4 VNI 50001 router-mac 00:1c:73:87:df:c6 local-interface Vxlan1

 B I      10.10.10.2/32 [200/0]
           via VTEP 10.0.0.0 VNI 50001 router-mac 00:00:00:00:01:01 local-interface Vxlan1
 C        10.10.10.0/24
           directly connected, Vlan10


VRF: TENANT2
Gateway of last resort:
 B I      0.0.0.0/0 [200/0]
           via VTEP 10.1.0.4 VNI 50002 router-mac 00:1c:73:87:df:c6 local-interface Vxlan1

 B I      10.10.20.2/32 [200/0]
           via VTEP 10.0.0.0 VNI 50002 router-mac 00:00:00:00:01:01 local-interface Vxlan1
 C        10.10.20.0/24
           directly connected, Vlan20
```
</details>

<details>
<summary>LEAF21 / route-type 2 / show bgp evpn route-type mac-ip</summary>
   
```
LEAF21#show bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 10.2.0.3, local AS number 65002
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.2.0.0:20 mac-ip aac1.ab2d.c08f
                                 10.0.0.1              -       100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 *  ec    RD: 10.2.0.0:20 mac-ip aac1.ab2d.c08f
                                 10.0.0.1              -       100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 * >Ec    RD: 10.2.0.0:20 mac-ip aac1.ab2d.c08f 10.10.20.1
                                 10.0.0.1              -       100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 *  ec    RD: 10.2.0.0:20 mac-ip aac1.ab2d.c08f 10.10.20.1
                                 10.0.0.1              -       100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 * >      RD: 10.2.0.3:10 mac-ip aac1.ab46.07d3
                                 -                     -       -       0       i
 * >      RD: 10.2.0.3:10 mac-ip aac1.ab46.07d3 10.10.10.2
                                 -                     -       -       0       i
 * >Ec    RD: 10.2.0.0:10 mac-ip aac1.ab6c.483a
                                 10.0.0.1              -       100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 *  ec    RD: 10.2.0.0:10 mac-ip aac1.ab6c.483a
                                 10.0.0.1              -       100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 * >Ec    RD: 10.2.0.0:10 mac-ip aac1.ab6c.483a 10.10.10.1
                                 10.0.0.1              -       100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 *  ec    RD: 10.2.0.0:10 mac-ip aac1.ab6c.483a 10.10.10.1
                                 10.0.0.1              -       100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 * >      RD: 10.2.0.3:20 mac-ip aac1.abc6.faf7
                                 -                     -       -       0       i
 * >      RD: 10.2.0.3:20 mac-ip aac1.abc6.faf7 10.10.20.2
                                 -                     -       -       0       i
```
</details>

<details>
<summary>LEAF11 / route-type 5 / show bgp evpn route-type ip-prefix</summary>

```
LEAF21#show bgp evpn route-type ip-prefix
BGP routing table information for VRF default
Router identifier 10.2.0.3, local AS number 65002
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.1.0.4:50001 ip-prefix 0.0.0.0/0
                                 10.0.0.1              -       100     0       65001 65101 65100 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 *  ec    RD: 10.1.0.4:50001 ip-prefix 0.0.0.0/0
                                 10.0.0.1              -       100     0       65001 65101 65100 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 * >Ec    RD: 10.1.0.4:50002 ip-prefix 0.0.0.0/0
                                 10.0.0.1              -       100     0       65001 65101 65100 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 *  ec    RD: 10.1.0.4:50002 ip-prefix 0.0.0.0/0
                                 10.0.0.1              -       100     0       65001 65101 65100 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 * >Ec    RD: 10.2.0.3:1 ip-prefix 0.0.0.0/0
                                 10.2.0.4              -       100     0       65102 65100 i Or-ID: 10.2.0.4 C-LST: 10.2.0.1 
 *  ec    RD: 10.2.0.3:1 ip-prefix 0.0.0.0/0
                                 10.2.0.4              -       100     0       65102 65100 i Or-ID: 10.2.0.4 C-LST: 10.2.0.1 
 * >Ec    RD: 10.2.0.3:2 ip-prefix 0.0.0.0/0
                                 10.2.0.4              -       100     0       65102 65100 i Or-ID: 10.2.0.4 C-LST: 10.2.0.1 
 *  ec    RD: 10.2.0.3:2 ip-prefix 0.0.0.0/0
                                 10.2.0.4              -       100     0       65102 65100 i Or-ID: 10.2.0.4 C-LST: 10.2.0.1 
 * >Ec    RD: 65001:102 ip-prefix 10.10.20.0/24
                                 10.0.0.1              -       100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 *  ec    RD: 65001:102 ip-prefix 10.10.20.0/24
                                 10.0.0.1              -       100     0       65001 i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 * >Ec    RD: 65002:102 ip-prefix 10.10.20.0/24
                                 10.0.0.1              -       100     0       i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
 *  ec    RD: 65002:102 ip-prefix 10.10.20.0/24
                                 10.0.0.1              -       100     0       i Or-ID: 10.2.0.0 C-LST: 10.2.0.1 
```
</details>

<details>
<summary>LEAF11 / show mac address-table</summary>

```
LEAF21#show mac address-table 
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  10    aac1.ab46.07d3    DYNAMIC     Et3        1       8:26:15 ago
  10    aac1.ab6c.483a    DYNAMIC     Vx1        1       8:26:15 ago
  20    aac1.ab2d.c08f    DYNAMIC     Vx1        1       8:28:00 ago
  20    aac1.abc6.faf7    DYNAMIC     Et4        1       8:26:56 ago
Total Mac Addresses for this criterion: 4
```
</details>

<details>
<summary>LEAF11 / show ip route vrf all</summary>

```
LEAF21#show ip route vrf all

VRF: default
Gateway of last resort is not set

 B I      10.0.0.0/32 [200/0]
           via 10.2.2.4, Ethernet1
           via 10.2.2.8, Ethernet2
 B I      10.0.0.1/32 [200/0]
           via 10.2.2.4, Ethernet1
           via 10.2.2.8, Ethernet2
 B I      10.0.0.2/31 [200/0]
           via 10.2.2.4, Ethernet1
           via 10.2.2.8, Ethernet2
 B I      10.1.0.0/32 [200/0]
           via 10.2.2.4, Ethernet1
           via 10.2.2.8, Ethernet2
 B I      10.1.0.1/32 [200/0]
           via 10.2.2.4, Ethernet1
           via 10.2.2.8, Ethernet2
 B I      10.1.0.2/32 [200/0]
           via 10.2.2.4, Ethernet1
           via 10.2.2.8, Ethernet2
 B I      10.1.0.3/32 [200/0]
           via 10.2.2.4, Ethernet1
           via 10.2.2.8, Ethernet2
 B I      10.1.0.4/32 [200/0]
           via 10.2.2.4, Ethernet1
           via 10.2.2.8, Ethernet2
 B I      10.1.2.0/31 [200/0]
           via 10.2.2.4, Ethernet1
           via 10.2.2.8, Ethernet2
 B I      10.1.2.2/31 [200/0]
           via 10.2.2.4, Ethernet1
           via 10.2.2.8, Ethernet2
 B I      10.1.2.4/31 [200/0]
           via 10.2.2.4, Ethernet1
           via 10.2.2.8, Ethernet2
 B I      10.1.2.6/31 [200/0]
           via 10.2.2.4, Ethernet1
           via 10.2.2.8, Ethernet2
 B I      10.1.2.8/31 [200/0]
           via 10.2.2.4, Ethernet1
           via 10.2.2.8, Ethernet2
 B I      10.1.2.10/31 [200/0]
           via 10.2.2.4, Ethernet1
           via 10.2.2.8, Ethernet2
 B I      10.2.0.0/32 [200/0]
           via 10.2.2.4, Ethernet1
           via 10.2.2.8, Ethernet2
 B I      10.2.0.1/32 [200/0]
           via 10.2.2.4, Ethernet1
 B I      10.2.0.2/32 [200/0]
           via 10.2.2.8, Ethernet2
 C        10.2.0.3/32
           directly connected, Loopback0
 B I      10.2.0.4/32 [200/0]
           via 10.2.2.4, Ethernet1
           via 10.2.2.8, Ethernet2
 B I      10.2.2.0/31 [200/0]
           via 10.2.2.4, Ethernet1
 B I      10.2.2.2/31 [200/0]
           via 10.2.2.8, Ethernet2
 C        10.2.2.4/31
           directly connected, Ethernet1
 B I      10.2.2.6/31 [200/0]
           via 10.2.2.4, Ethernet1
 C        10.2.2.8/31
           directly connected, Ethernet2
 B I      10.2.2.10/31 [200/0]
           via 10.2.2.8, Ethernet2

VRF: TENANT1
Gateway of last resort:
 B I      0.0.0.0/0 [200/0]
           via VTEP 10.2.0.4 VNI 50001 router-mac 00:1c:73:24:5f:da local-interface Vxlan1

 B I      10.10.10.1/32 [200/0]
           via VTEP 10.0.0.1 VNI 50001 router-mac 00:00:00:00:02:02 local-interface Vxlan1
 C        10.10.10.0/24
           directly connected, Vlan10


VRF: TENANT2
Gateway of last resort:
 B I      0.0.0.0/0 [200/0]
           via VTEP 10.2.0.4 VNI 50002 router-mac 00:1c:73:24:5f:da local-interface Vxlan1

 B I      10.10.20.1/32 [200/0]
           via VTEP 10.0.0.1 VNI 50002 router-mac 00:00:00:00:02:02 local-interface Vxlan1
 C        10.10.20.0/24
           directly connected, Vlan20```
</details>

<details>
<summary>FW1 / show ip route bgp</summary>

```
fw1# show ip route vrf FW bgp 
Codes: K - kernel route, C - connected, L - local, S - static,
       R - RIP, O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric, t - Table-Direct,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

IPv4 unicast VRF FW:
B>* 0.0.0.0/0 [20/0] via 10.0.2.1, eth3, weight 1, 09:00:36
B>* 8.8.8.8/32 [20/0] via 10.0.2.1, eth3, weight 1, 09:00:36
B>* 10.0.1.0/32 [20/0] via 10.0.2.1, eth3, weight 1, 09:00:36
B>* 10.0.1.2/32 [20/0] via 10.0.2.1, eth3, weight 1, 08:55:48
B>* 10.0.2.2/31 [20/0] via 10.0.2.1, eth3, weight 1, 09:00:36
B>* 10.2.2.12/31 [20/0] via 10.0.2.1, eth3, weight 1, 08:55:48
B>* 10.2.2.14/31 [20/0] via 10.0.2.1, eth3, weight 1, 08:55:48
B>* 10.10.10.0/24 [20/0] via 10.1.2.12, eth1.110, weight 1, 09:00:37
B>* 10.10.10.1/32 [20/0] via 10.1.2.12, eth1.110, weight 1, 08:28:28
B>* 10.10.10.2/32 [20/0] via 10.0.2.1, eth3, weight 1, 02:01:41
B>* 10.10.20.0/24 [20/0] via 10.1.2.14, eth1.120, weight 1, 09:00:37
B>* 10.10.20.1/32 [20/0] via 10.1.2.14, eth1.120, weight 1, 08:28:09
B>* 10.10.20.2/32 [20/0] via 10.0.2.1, eth3, weight 1, 02:01:42
```
</details>

<details>

<details>
<summary>FW2 / show ip route</summary>

```
fw2# show ip route vrf FW bgp 
Codes: K - kernel route, C - connected, L - local, S - static,
       R - RIP, O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric, t - Table-Direct,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

IPv4 unicast VRF FW:
B>* 0.0.0.0/0 [20/0] via 10.0.2.3, eth3, weight 1, 08:56:38
B>* 8.8.8.8/32 [20/0] via 10.0.2.3, eth3, weight 1, 08:56:38
B>* 10.0.1.0/32 [20/0] via 10.0.2.3, eth3, weight 1, 08:56:38
B>* 10.0.1.1/32 [20/0] via 10.0.2.3, eth3, weight 1, 08:56:38
B>* 10.0.2.0/31 [20/0] via 10.0.2.3, eth3, weight 1, 08:56:38
B>* 10.1.2.12/31 [20/0] via 10.0.2.3, eth3, weight 1, 08:56:38
B>* 10.1.2.14/31 [20/0] via 10.0.2.3, eth3, weight 1, 08:56:38
B>* 10.10.10.0/24 [20/0] via 10.0.2.3, eth3, weight 1, 04:50:23
B>* 10.10.10.1/32 [20/0] via 10.0.2.3, eth3, weight 1, 08:29:18
B>* 10.10.10.2/32 [20/0] via 10.2.2.12, eth1.110, weight 1, 02:02:31
B>* 10.10.20.0/24 [20/0] via 10.0.2.3, eth3, weight 1, 04:50:23
B>* 10.10.20.1/32 [20/0] via 10.0.2.3, eth3, weight 1, 08:28:59
B>* 10.10.20.2/32 [20/0] via 10.2.2.14, eth1.120, weight 1, 02:02:32
```
</details>

<details>
<summary>ping HOST3 -> HOST1, HOST2, HOST4, 8.8.8.8</summary>

```
host1:/# ping -c 3 10.10.10.2
PING 10.10.10.2 (10.10.10.2): 56 data bytes
64 bytes from 10.10.10.2: seq=0 ttl=64 time=4.615 ms
64 bytes from 10.10.10.2: seq=1 ttl=64 time=3.401 ms
64 bytes from 10.10.10.2: seq=2 ttl=64 time=3.453 ms

--- 10.10.10.2 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 3.401/3.823/4.615 ms
host1:/# ping -c 3 10.10.20.1
PING 10.10.20.1 (10.10.20.1): 56 data bytes
64 bytes from 10.10.20.1: seq=0 ttl=59 time=3.071 ms
64 bytes from 10.10.20.1: seq=1 ttl=59 time=3.388 ms
64 bytes from 10.10.20.1: seq=2 ttl=59 time=3.124 ms

--- 10.10.20.1 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 3.071/3.194/3.388 ms
host1:/# ping -c 3 10.10.20.2
PING 10.10.20.2 (10.10.20.2): 56 data bytes
64 bytes from 10.10.20.2: seq=0 ttl=57 time=3.245 ms
64 bytes from 10.10.20.2: seq=1 ttl=57 time=3.603 ms
64 bytes from 10.10.20.2: seq=2 ttl=57 time=3.453 ms

--- 10.10.20.2 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 3.245/3.433/3.603 ms
host1:/# ping -c 3 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=61 time=1.689 ms
64 bytes from 8.8.8.8: seq=1 ttl=61 time=1.682 ms
64 bytes from 8.8.8.8: seq=2 ttl=61 time=1.492 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 1.492/1.621/1.689 ms
```
</details>

<details>
<summary>ping HOST3 -> HOST1, HOST2, HOST4, 8.8.8.8</summary>

```
host3:/# ping -c 3 10.10.10.1
PING 10.10.10.1 (10.10.10.1): 56 data bytes
64 bytes from 10.10.10.1: seq=0 ttl=64 time=4.013 ms
64 bytes from 10.10.10.1: seq=1 ttl=64 time=3.566 ms
64 bytes from 10.10.10.1: seq=2 ttl=64 time=3.413 ms

--- 10.10.10.1 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 3.413/3.664/4.013 ms
host3:/# ping -c 3 10.10.20.1
PING 10.10.20.1 (10.10.20.1): 56 data bytes
64 bytes from 10.10.20.1: seq=0 ttl=57 time=3.471 ms
64 bytes from 10.10.20.1: seq=1 ttl=57 time=3.958 ms
64 bytes from 10.10.20.1: seq=2 ttl=57 time=4.217 ms

--- 10.10.20.1 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 3.471/3.882/4.217 ms
host3:/# ping -c 3 10.10.20.2
PING 10.10.20.2 (10.10.20.2): 56 data bytes
64 bytes from 10.10.20.2: seq=0 ttl=59 time=3.798 ms
64 bytes from 10.10.20.2: seq=1 ttl=59 time=2.985 ms
64 bytes from 10.10.20.2: seq=2 ttl=59 time=3.448 ms

--- 10.10.20.2 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 2.985/3.410/3.798 ms
host3:/# ping -c 3 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=61 time=1.499 ms
64 bytes from 8.8.8.8: seq=1 ttl=61 time=2.050 ms
64 bytes from 8.8.8.8: seq=2 ttl=61 time=1.926 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 1.499/1.825/2.050 ms
```
</details>

<details>
<summary>Трассировка / Host1 -> Host4</summary>

```
host1:/# traceroute -n 10.10.20.2
traceroute to 10.10.20.2 (10.10.20.2), 30 hops max, 46 byte packets
 1  10.10.10.254  0.880 ms  0.387 ms  0.394 ms
 2  10.1.2.12  1.689 ms  1.389 ms  1.384 ms
 3  10.1.2.13  1.536 ms  1.291 ms  1.412 ms
 4  10.0.2.1  1.410 ms  1.402 ms  1.422 ms
 5  10.0.2.2  1.489 ms  1.324 ms  1.394 ms
 6  10.2.2.14  2.091 ms  1.809 ms  1.728 ms
 7  10.10.20.254  3.270 ms  2.591 ms  2.646 ms
 8  10.10.20.2  2.904 ms  2.852 ms  2.825 ms
```
</details>

<details>
<summary>Трассировка / Host3 -> Host2</summary>

```
host3:/# traceroute -n 10.10.20.1
traceroute to 10.10.20.1 (10.10.20.1), 30 hops max, 46 byte packets
 1  10.10.10.254  0.840 ms  0.493 ms  0.431 ms
 2  10.2.2.12  1.812 ms  1.282 ms  1.224 ms
 3  10.2.2.13  1.406 ms  1.291 ms  1.312 ms
 4  10.0.2.3  1.370 ms  1.274 ms  1.345 ms
 5  10.0.2.0  1.404 ms  1.321 ms  1.398 ms
 6  10.1.2.14  1.968 ms  1.749 ms  1.668 ms
 7  10.10.20.254  3.357 ms  2.595 ms  2.470 ms
 8  10.10.20.1  2.825 ms  2.685 ms  2.613 ms```
</details>

## 6. Политика прохождения трафика
Определяем следующие правила прохождения трафика:
1. В рамках одного VRF трафик маршрутизируется через DCI.
2.1. Трафик между VRF для устройств одного сайта маршрутизируется через МСЭ этого сайта.
2.2. Трафик между VRF для устройств разных сайта маршрутизируется через МСЭ обоих сайтов (используется анонс в сторону МСЭ /32 своего сайта).
3. Трафик между VRF и внешним миром маршрутизируется через МСЭ этого сайта.
4. При аварийной ситуации и недоступности МСЭ своего сайта, маршрутизация между разными VRF и внешним миром (для аварийного сайта) осуществляется через DCI и МСЭ соседнего сайта.

## 7. Политика фильтрации трафика
В качестве МСЭ в стенде используются контейнеры alpine с развернутыми FRR, nftables и iptables.
Для демонстрации возможности фильтрации трафика определяем следующие правила доступа, настроенные на каждом МСЭ:
1. Блокируем ICMP-запросы из сегмента 10.10.20.0/24 в 10.10.10.0/24
2. Блокируем трафик из сегмента 10.10.20.0/24 до 8.8.8.8
   
FW1:
```
root@fw1:/# iptables -L FORWARD -n -v --line-numbers
Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 DROP       icmp --  *      *       10.10.20.0/24        10.10.10.0/24        icmptype 8
2        0     0 DROP       all  --  *      *       10.10.20.0/24        8.8.8.8
```

FW2:
```
root@fw2:/# iptables -L FORWARD -n -v --line-numbers
Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 DROP       icmp --  *      *       10.10.20.0/24        10.10.10.0/24        icmptype 8
2        0     0 DROP       all  --  *      *       10.10.20.0/24        8.8.8.8
```
Демонстрация фильтрации трафика:<\br>
[![Демонстрация фильтрации трафика](https://img.youtube.com/vi/uzf7huBhPa8/default.jpg)](https://youtu.be/uzf7huBhPa8&vq=hd720)</br>

<img width="1390" height="157" alt="image" src="https://github.com/user-attachments/assets/cd25cf57-6070-4474-bada-93d05493674c" />

Наблюдаем, что заявленные правила фильтрации отрабатывают корректно:
1. Доступ к 8.8.8.8 есть только у сегмента 10.10.10.0/24
2. ICMP-запросы 10.10.10.0/24 -> 10.10.20.0/24 проходят успешно
3. ICMP-запросы 10.10.20.0/24 -> 10.10.10.0/24 блокируются на МСЭ

## 8. Проверка отказоустойчивости
На видео показан вариант с имитацией отказа линка между Leaf22 и FW2.</br>
[![Тестирование отключения линка LEAF22 -- FW2](https://img.youtube.com/vi/a_ydqCp0GHg/default.jpg)](https://youtu.be/a_ydqCp0GHg&vq=hd720)</br>
Наблюдаем, что  происходит следующее:</br>
0. В штатной ситуации default указывает на Border LEAF своего сfйта (получаем с FW)
```
LEAF21#sh ip route vrf TENANT1 0.0.0.0/0
VRF: TENANT1
Gateway of last resort:
 B I      0.0.0.0/0 [200/0]
           via VTEP 10.2.0.4 VNI 50001 router-mac 00:1c:73:24:5f:da local-interface Vxlan1
```
2. При отключении интерфейса на сайте 2 пропадает default, полученный от FW2 (происходит быстро из-за настроенног BFD) и default из сайта 1 (который получаем с BGW2) становится в таблицу маршрутизации.
```
LEAF21#sh ip route vrf TENANT1 0.0.0.0/0
VRF: TENANT1
Source Codes:
Gateway of last resort:
 B I      0.0.0.0/0 [200/0]
           via VTEP 10.0.0.1 VNI 50001 router-mac 00:00:00:00:02:02 local-interface Vxlan1
```
3. Выход во внешний мир и трафик между VRF для сайта 2 теперь идет через МСЭ сайта 1 (BGW при этом не отвечают в трассировке. Вероятно, это особенность cEOS, т.к. в дампе трафика фиксируем запрос с TTL=1, но него нет ответа).
```
host3:/# traceroute -n -w 1 10.10.20.1
traceroute to 10.10.20.1 (10.10.20.1), 30 hops max, 46 byte packets
 1  10.10.10.254  0.645 ms  0.443 ms  0.439 ms
 2  *  *  *
 3  *  *  *
 4  10.1.2.12  3.691 ms  2.597 ms  2.543 ms
 5  10.1.2.13  2.794 ms  2.634 ms  2.904 ms
 6  10.1.2.14  2.992 ms  2.876 ms  2.834 ms
 7  10.10.20.254  3.958 ms  3.943 ms  3.709 ms
 8  10.10.20.1  3.932 ms  3.838 ms  3.939 ms
```   
* При этом при аварийной ситуации с продолжающимися проверками с помощью ICMP фиксируем пропадание не более одного пакета.
** Обратное переключение происходит также безболезненно.

## 9. Конфигурации устройств фабрики
[Топология ContainerLab](./configs/clab.yml)<br>
[Конфигурация SPINE11](./configs/spine11.conf)<br>
[Конфигурация SPINE12](./configs/spine12.conf)<br>
[Конфигурация SPINE21](./configs/spine21.conf)<br>
[Конфигурация SPINE22](./configs/spine22.conf)<br>
[Конфигурация LEAF11](./configs/leaf11.conf)<br>
[Конфигурация LEAF12](./configs/leaf12.conf)<br>
[Конфигурация LEAF21](./configs/leaf21.conf)<br>
[Конфигурация LEAF22](./configs/leaf22.conf)<br>
[Конфигурация BGW1](./configs/bgw1.conf)<br>
[Конфигурация BGW2](./configs/bgw2.conf)<br>
[Конфигурация FW1](./configs/fw1.conf)<br>
[Конфигурация FW2](./configs/fw2.conf)<br>
[Конфигурация GW](./configs/gw.conf)<br>

## 10. Презентация
[Презентация к проекту](./project.pptx)<br>
