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
 *  ec    10.2.2.10/31           10.0.0.3              0       -          100     0       65002 i Or-ID: 10.1.0.0 C-LST: 10.1.0.1```
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
