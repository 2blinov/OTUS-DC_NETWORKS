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

## 3. Настройка Underlay на базе ISIS
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
<summary>Контекст: Процесс ISIS</summary>

```eos
router isis UNDERLAY                    # Процесс IS-IS с именем UNDERLAY
   net 49.0001.0000.0000.0001.00        # Задаём NET (Network Entity Title) устройства в IS-IS.
                                        # 49.0001 — Area ID
                                        # 0000.0000.0001 — System ID устройства.
                                        # 00 — Network Selector, для IS-IS обычно 00.
   router-id ipv4 10.1.0.1              # Явно настраиваем Router ID
   is-type level-2                      # Устройство работает как IS-IS Level-2 router
   log-adjacency-changes                # Записываем изменения состояния IS-IS соседств в системный лог.
   !
   address-family ipv4 unicast
      bfd all-interfaces                # Включаем BFD для IS-IS на всех IS-IS-интерфейсах (за исключение Lo0, т.к. на нем настраиваем <b>isis passive</b>).
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
   isis enable UNDERLAY                      # Включаем IS-IS процесс UNDERLAY на интерфейсе
   isis circuit-type level-2                 # Задаем тип IS-IS circuit как Level-2
   isis network point-to-point               # Говорим IS-IS, что интерфейс point-to-point circuit, несмотря на то, что физически это Ethernet
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
80 bytes from 10.1.2.1: icmp_seq=1 ttl=64 time=0.241 ms
80 bytes from 10.1.2.1: icmp_seq=2 ttl=64 time=0.018 ms
80 bytes from 10.1.2.1: icmp_seq=3 ttl=64 time=0.013 ms

--- 10.1.2.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.013/0.090/0.241/0.106 ms, ipg/ewma 0.184/0.188 ms
SPINE1#ping 10.1.2.3 repeat 3
PING 10.1.2.3 (10.1.2.3) 72(100) bytes of data.
80 bytes from 10.1.2.3: icmp_seq=1 ttl=64 time=0.169 ms
80 bytes from 10.1.2.3: icmp_seq=2 ttl=64 time=0.015 ms
80 bytes from 10.1.2.3: icmp_seq=3 ttl=64 time=0.012 ms

--- 10.1.2.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.012/0.065/0.169/0.073 ms, ipg/ewma 0.135/0.132 ms
SPINE1#ping 10.1.2.5 repeat 3
PING 10.1.2.5 (10.1.2.5) 72(100) bytes of data.
80 bytes from 10.1.2.5: icmp_seq=1 ttl=64 time=0.173 ms
80 bytes from 10.1.2.5: icmp_seq=2 ttl=64 time=0.018 ms
80 bytes from 10.1.2.5: icmp_seq=3 ttl=64 time=0.011 ms

--- 10.1.2.5 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.011/0.067/0.173/0.074 ms, ipg/ewma 0.141/0.135 ms
```
</details>

<details>
<summary>Доступность транспортных адресов соседей SPINE2 -> LEAFX</summary>
  
```eos
SPINE2#ping 10.1.2.7 repeat 3
PING 10.1.2.7 (10.1.2.7) 72(100) bytes of data.
80 bytes from 10.1.2.7: icmp_seq=1 ttl=64 time=0.123 ms
80 bytes from 10.1.2.7: icmp_seq=2 ttl=64 time=0.014 ms
80 bytes from 10.1.2.7: icmp_seq=3 ttl=64 time=0.011 ms

--- 10.1.2.7 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.011/0.049/0.123/0.052 ms, ipg/ewma 0.105/0.097 ms
SPINE2#ping 10.1.2.9 repeat 3
PING 10.1.2.9 (10.1.2.9) 72(100) bytes of data.
80 bytes from 10.1.2.9: icmp_seq=1 ttl=64 time=0.159 ms
80 bytes from 10.1.2.9: icmp_seq=2 ttl=64 time=0.015 ms
80 bytes from 10.1.2.9: icmp_seq=3 ttl=64 time=0.011 ms

--- 10.1.2.9 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.011/0.061/0.159/0.068 ms, ipg/ewma 0.126/0.124 ms
SPINE2#ping 10.1.2.11 repeat 3
PING 10.1.2.11 (10.1.2.11) 72(100) bytes of data.
80 bytes from 10.1.2.11: icmp_seq=1 ttl=64 time=0.145 ms
80 bytes from 10.1.2.11: icmp_seq=2 ttl=64 time=0.015 ms
80 bytes from 10.1.2.11: icmp_seq=3 ttl=64 time=0.011 ms

--- 10.1.2.11 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.011/0.057/0.145/0.062 ms, ipg/ewma 0.117/0.114 ms
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
10.1.2.1          1104648192        483150795            Ethernet1(299)       normal       08/23/26 13:57             NA       No Diagnostic       Up
10.1.2.3           562675910       2285687862            Ethernet2(296)       normal       08/23/26 13:57             NA       No Diagnostic       Up
10.1.2.5          1987680360       2316176272            Ethernet3(300)       normal       08/23/26 13:57             NA       No Diagnostic       Up
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
10.1.2.7            736691354       2529852040            Ethernet1(295)       normal       08/23/26 13:56             NA       No Diagnostic       Up
10.1.2.9           3652861756       3361988941            Ethernet2(302)       normal       08/23/26 13:56             NA       No Diagnostic       Up
10.1.2.11          1735839849       3139066771            Ethernet3(304)       normal       08/23/26 13:56             NA       No Diagnostic       Up
```
</details>

<details>
<summary>SPINE1 / show isis neighbor</summary>
  
```eos
SPINE1#show isis neighbor
 
Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id          
UNDERLAY  default  LEAF1            L2   Ethernet1          P2P               UP    26          2B                  
UNDERLAY  default  LEAF2            L2   Ethernet2          P2P               UP    22          2A                  
UNDERLAY  default  LEAF3            L2   Ethernet3          P2P               UP    26          2E
```
</details>

<details>
<summary>SPINE2 / show isis neighbor</summary>
  
```eos
SPINE2#show isis neighbor
 
Instance  VRF      System Id        Type Interface          SNPA              State Hold time   Circuit Id          
UNDERLAY  default  LEAF1            L2   Ethernet1          P2P               UP    27          27                  
UNDERLAY  default  LEAF2            L2   Ethernet2          P2P               UP    27          30                  
UNDERLAY  default  LEAF3            L2   Ethernet3          P2P               UP    28          32
```
</details>


<details>
<summary> SPINE1 / show isis database detail</summary>
  
```eos
SPINE1#show isis database detail
Legend:
H - hostname conflict
U - node unreachable

IS-IS Instance: UNDERLAY VRF: default
  IS-IS Level 2 Link State Database
    LSPID                   Seq Num  Cksum  Life Length IS  Received LSPID        Flags
    SPINE1.00-00                443  65155  1095    146 L2  0000.0000.0001.00-00  <>
      LSP generation remaining wait time: 0 ms
      Time remaining until refresh: 795 s
      NLPID: 0xCC(IPv4)
      Hostname: SPINE1
      Area addresses: 49.0001
      Interface address: 10.1.2.4
      Interface address: 10.1.2.2
      Interface address: 10.1.2.0
      Interface address: 10.1.0.1
      IS Neighbor          : LEAF1.00            Metric: 10
      IS Neighbor          : LEAF3.00            Metric: 10
      IS Neighbor          : LEAF2.00            Metric: 10
      Reachability         : 10.1.2.4/31 Metric: 10 Type: 1 Up
      Reachability         : 10.1.2.2/31 Metric: 10 Type: 1 Up
      Reachability         : 10.1.2.0/31 Metric: 10 Type: 1 Up
      Reachability         : 10.1.0.1/32 Metric: 10 Type: 1 Up
      Router Capabilities: Router Id: 10.1.0.1 Flags: []
        Area leader priority: 250 algorithm: 0
    SPINE2.00-00                 31  46401   824    146 L2  0000.0000.0002.00-00  <>
      LSP received time: 2026-08-23 15:01:19
      Remaining lifetime received: 1199 s Modified to: 1200 s
      NLPID: 0xCC(IPv4)
      Hostname: SPINE2
      Area addresses: 49.0001
      Interface address: 10.1.2.10
      Interface address: 10.1.2.6
      Interface address: 10.1.2.8
      Interface address: 10.1.0.2
      IS Neighbor          : LEAF3.00            Metric: 10
      IS Neighbor          : LEAF1.00            Metric: 10
      IS Neighbor          : LEAF2.00            Metric: 10
      Reachability         : 10.1.2.10/31 Metric: 10 Type: 1 Up
      Reachability         : 10.1.2.6/31 Metric: 10 Type: 1 Up
      Reachability         : 10.1.2.8/31 Metric: 10 Type: 1 Up
      Reachability         : 10.1.0.2/32 Metric: 10 Type: 1 Up
      Router Capabilities: Router Id: 10.1.0.2 Flags: []
        Area leader priority: 250 algorithm: 0
    LEAF1.00-00                 431   2386   750    121 L2  0000.0000.0003.00-00  <>
      LSP received time: 2026-08-23 15:00:05
      Remaining lifetime received: 1199 s Modified to: 1200 s
      NLPID: 0xCC(IPv4)
      Hostname: LEAF1
      Area addresses: 49.0001
      Interface address: 10.1.2.7
      Interface address: 10.1.2.1
      Interface address: 10.1.0.3
      IS Neighbor          : SPINE1.00           Metric: 10
      IS Neighbor          : SPINE2.00           Metric: 10
      Reachability         : 10.1.2.6/31 Metric: 10 Type: 1 Up
      Reachability         : 10.1.2.0/31 Metric: 10 Type: 1 Up
      Reachability         : 10.1.0.3/32 Metric: 10 Type: 1 Up
      Router Capabilities: Router Id: 10.1.0.3 Flags: []
        Area leader priority: 250 algorithm: 0
    LEAF2.00-00                  25  18079  1004    121 L2  0000.0000.0004.00-00  <>
      LSP received time: 2026-08-23 15:04:19
      Remaining lifetime received: 1199 s Modified to: 1200 s
      NLPID: 0xCC(IPv4)
      Hostname: LEAF2
      Area addresses: 49.0001
      Interface address: 10.1.2.9
      Interface address: 10.1.2.3
      Interface address: 10.1.0.4
      IS Neighbor          : SPINE1.00           Metric: 10
      IS Neighbor          : SPINE2.00           Metric: 10
      Reachability         : 10.1.2.8/31 Metric: 10 Type: 1 Up
      Reachability         : 10.1.2.2/31 Metric: 10 Type: 1 Up
      Reachability         : 10.1.0.4/32 Metric: 10 Type: 1 Up
      Router Capabilities: Router Id: 10.1.0.4 Flags: []
        Area leader priority: 250 algorithm: 0
    LEAF3.00-00                  28  39738   699    121 L2  0000.0000.0005.00-00  <>
      LSP received time: 2026-08-23 14:59:15
      Remaining lifetime received: 1199 s Modified to: 1200 s
      NLPID: 0xCC(IPv4)
      Hostname: LEAF3
      Area addresses: 49.0001
      Interface address: 10.1.2.5
      Interface address: 10.1.2.11
      Interface address: 10.1.0.5
      IS Neighbor          : SPINE1.00           Metric: 10
      IS Neighbor          : SPINE2.00           Metric: 10
      Reachability         : 10.1.2.4/31 Metric: 10 Type: 1 Up
      Reachability         : 10.1.2.10/31 Metric: 10 Type: 1 Up
      Reachability         : 10.1.0.5/32 Metric: 10 Type: 1 Up
      Router Capabilities: Router Id: 10.1.0.5 Flags: []
        Area leader priority: 250 algorithm: 0
```
</details>

<details>
<summary> SPINE2 / show isis database detail</summary>
  
```eos
SPINE2#show isis database detail
Legend:
H - hostname conflict
U - node unreachable

IS-IS Instance: UNDERLAY VRF: default
  IS-IS Level 2 Link State Database
    LSPID                   Seq Num  Cksum  Life Length IS  Received LSPID        Flags
    SPINE1.00-00                443  65155   980    146 L2  0000.0000.0001.00-00  <>
      LSP received time: 2026-08-23 15:05:54
      Remaining lifetime received: 1199 s Modified to: 1200 s
      NLPID: 0xCC(IPv4)
      Hostname: SPINE1
      Area addresses: 49.0001
      Interface address: 10.1.2.4
      Interface address: 10.1.2.2
      Interface address: 10.1.2.0
      Interface address: 10.1.0.1
      IS Neighbor          : LEAF1.00            Metric: 10
      IS Neighbor          : LEAF3.00            Metric: 10
      IS Neighbor          : LEAF2.00            Metric: 10
      Reachability         : 10.1.2.4/31 Metric: 10 Type: 1 Up
      Reachability         : 10.1.2.2/31 Metric: 10 Type: 1 Up
      Reachability         : 10.1.2.0/31 Metric: 10 Type: 1 Up
      Reachability         : 10.1.0.1/32 Metric: 10 Type: 1 Up
      Router Capabilities: Router Id: 10.1.0.1 Flags: []
        Area leader priority: 250 algorithm: 0
    SPINE2.00-00                 31  46401   709    146 L2  0000.0000.0002.00-00  <>
      LSP generation remaining wait time: 0 ms
      Time remaining until refresh: 409 s
      NLPID: 0xCC(IPv4)
      Hostname: SPINE2
      Area addresses: 49.0001
      Interface address: 10.1.2.10
      Interface address: 10.1.2.6
      Interface address: 10.1.2.8
      Interface address: 10.1.0.2
      IS Neighbor          : LEAF3.00            Metric: 10
      IS Neighbor          : LEAF1.00            Metric: 10
      IS Neighbor          : LEAF2.00            Metric: 10
      Reachability         : 10.1.2.10/31 Metric: 10 Type: 1 Up
      Reachability         : 10.1.2.6/31 Metric: 10 Type: 1 Up
      Reachability         : 10.1.2.8/31 Metric: 10 Type: 1 Up
      Reachability         : 10.1.0.2/32 Metric: 10 Type: 1 Up
      Router Capabilities: Router Id: 10.1.0.2 Flags: []
        Area leader priority: 250 algorithm: 0
    LEAF1.00-00                 431   2386   635    121 L2  0000.0000.0003.00-00  <>
      LSP received time: 2026-08-23 15:00:10
      Remaining lifetime received: 1199 s Modified to: 1200 s
      NLPID: 0xCC(IPv4)
      Hostname: LEAF1
      Area addresses: 49.0001
      Interface address: 10.1.2.7
      Interface address: 10.1.2.1
      Interface address: 10.1.0.3
      IS Neighbor          : SPINE1.00           Metric: 10
      IS Neighbor          : SPINE2.00           Metric: 10
      Reachability         : 10.1.2.6/31 Metric: 10 Type: 1 Up
      Reachability         : 10.1.2.0/31 Metric: 10 Type: 1 Up
      Reachability         : 10.1.0.3/32 Metric: 10 Type: 1 Up
      Router Capabilities: Router Id: 10.1.0.3 Flags: []
        Area leader priority: 250 algorithm: 0
    LEAF2.00-00                  25  18079   889    121 L2  0000.0000.0004.00-00  <>
      LSP received time: 2026-08-23 15:04:23
      Remaining lifetime received: 1199 s Modified to: 1200 s
      NLPID: 0xCC(IPv4)
      Hostname: LEAF2
      Area addresses: 49.0001
      Interface address: 10.1.2.9
      Interface address: 10.1.2.3
      Interface address: 10.1.0.4
      IS Neighbor          : SPINE1.00           Metric: 10
      IS Neighbor          : SPINE2.00           Metric: 10
      Reachability         : 10.1.2.8/31 Metric: 10 Type: 1 Up
      Reachability         : 10.1.2.2/31 Metric: 10 Type: 1 Up
      Reachability         : 10.1.0.4/32 Metric: 10 Type: 1 Up
      Router Capabilities: Router Id: 10.1.0.4 Flags: []
        Area leader priority: 250 algorithm: 0
    LEAF3.00-00                  28  39738   584    121 L2  0000.0000.0005.00-00  <>
      LSP received time: 2026-08-23 14:59:19
      Remaining lifetime received: 1199 s Modified to: 1200 s
      NLPID: 0xCC(IPv4)
      Hostname: LEAF3
      Area addresses: 49.0001
      Interface address: 10.1.2.5
      Interface address: 10.1.2.11
      Interface address: 10.1.0.5
      IS Neighbor          : SPINE1.00           Metric: 10
      IS Neighbor          : SPINE2.00           Metric: 10
      Reachability         : 10.1.2.4/31 Metric: 10 Type: 1 Up
      Reachability         : 10.1.2.10/31 Metric: 10 Type: 1 Up
      Reachability         : 10.1.0.5/32 Metric: 10 Type: 1 Up
      Router Capabilities: Router Id: 10.1.0.5 Flags: []
        Area leader priority: 250 algorithm: 0
```
</details>


<details>
<summary>SPINE1 / show ip route isis</summary>
  
```eos
SPINE1#  show ip route isis

VRF: default
 I L2     10.1.0.2/32 [115/30]            # 115 - административная дистанция IS-IS, Метрика 10+10+10=30
           via 10.1.2.1, Ethernet1
           via 10.1.2.3, Ethernet2
           via 10.1.2.5, Ethernet3
 I L2     10.1.0.3/32 [115/20]
           via 10.1.2.1, Ethernet1
 I L2     10.1.0.4/32 [115/20]
           via 10.1.2.3, Ethernet2
 I L2     10.1.0.5/32 [115/20]
           via 10.1.2.5, Ethernet3
 I L2     10.1.2.6/31 [115/20]
           via 10.1.2.1, Ethernet1
 I L2     10.1.2.8/31 [115/20]
           via 10.1.2.3, Ethernet2
 I L2     10.1.2.10/31 [115/20]
           via 10.1.2.5, Ethernet3

```
</details>

<details>
<summary>SPINE2 / show ip route isis</summary>
  
```eos
SPINE2#show ip route isis 

VRF: default
 I L2     10.1.0.1/32 [115/30]
           via 10.1.2.7, Ethernet1
           via 10.1.2.9, Ethernet2
           via 10.1.2.11, Ethernet3
 I L2     10.1.0.3/32 [115/20]
           via 10.1.2.7, Ethernet1
 I L2     10.1.0.4/32 [115/20]
           via 10.1.2.9, Ethernet2
 I L2     10.1.0.5/32 [115/20]
           via 10.1.2.11, Ethernet3
 I L2     10.1.2.0/31 [115/20]
           via 10.1.2.7, Ethernet1
 I L2     10.1.2.2/31 [115/20]
           via 10.1.2.9, Ethernet2
 I L2     10.1.2.4/31 [115/20]
           via 10.1.2.11, Ethernet3
```
</details>

<details>
<summary>SPINE1 / ping до Lo0 устройств фабрики</summary>
  
```eos
SPINE1#ping 10.1.0.2 source loopback 0 repeat 3
PING 10.1.0.2 (10.1.0.2) from 10.1.0.1 : 72(100) bytes of data.
80 bytes from 10.1.0.2: icmp_seq=1 ttl=63 time=2.09 ms
80 bytes from 10.1.0.2: icmp_seq=2 ttl=63 time=0.370 ms
80 bytes from 10.1.0.2: icmp_seq=3 ttl=63 time=0.337 ms

--- 10.1.0.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 4ms
rtt min/avg/max/mdev = 0.337/0.933/2.093/0.820 ms, ipg/ewma 2.155/1.685 ms
SPINE1#ping 10.1.0.3 source loopback 0 repeat 3
PING 10.1.0.3 (10.1.0.3) from 10.1.0.1 : 72(100) bytes of data.
80 bytes from 10.1.0.3: icmp_seq=1 ttl=64 time=0.164 ms
80 bytes from 10.1.0.3: icmp_seq=2 ttl=64 time=0.018 ms
80 bytes from 10.1.0.3: icmp_seq=3 ttl=64 time=0.011 ms

--- 10.1.0.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.011/0.064/0.164/0.070 ms, ipg/ewma 0.131/0.129 ms
SPINE1#ping 10.1.0.4 source loopback 0 repeat 3
PING 10.1.0.4 (10.1.0.4) from 10.1.0.1 : 72(100) bytes of data.
80 bytes from 10.1.0.4: icmp_seq=1 ttl=64 time=0.178 ms
80 bytes from 10.1.0.4: icmp_seq=2 ttl=64 time=0.015 ms
80 bytes from 10.1.0.4: icmp_seq=3 ttl=64 time=0.011 ms

--- 10.1.0.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.011/0.068/0.178/0.077 ms, ipg/ewma 0.137/0.139 ms
SPINE1#ping 10.1.0.5 source loopback 0 repeat 3
PING 10.1.0.5 (10.1.0.5) from 10.1.0.1 : 72(100) bytes of data.
80 bytes from 10.1.0.5: icmp_seq=1 ttl=64 time=0.205 ms
80 bytes from 10.1.0.5: icmp_seq=2 ttl=64 time=0.017 ms
80 bytes from 10.1.0.5: icmp_seq=3 ttl=64 time=0.012 ms

--- 10.1.0.5 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.012/0.078/0.205/0.089 ms, ipg/ewma 0.157/0.160 ms
```
</details>

<details>
<summary>SPINE2 / ping до Lo0 устройств фабрики</summary>
  
```eos
SPINE2#ping 10.1.0.1 source loopback 0 repeat 3
PING 10.1.0.1 (10.1.0.1) from 10.1.0.2 : 72(100) bytes of data.
80 bytes from 10.1.0.1: icmp_seq=1 ttl=63 time=0.633 ms
80 bytes from 10.1.0.1: icmp_seq=2 ttl=63 time=0.356 ms
80 bytes from 10.1.0.1: icmp_seq=3 ttl=63 time=0.324 ms

--- 10.1.0.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2ms
rtt min/avg/max/mdev = 0.324/0.437/0.633/0.138 ms, ipg/ewma 1.001/0.564 ms
SPINE2#ping 10.1.0.3 source loopback 0 repeat 3
PING 10.1.0.3 (10.1.0.3) from 10.1.0.2 : 72(100) bytes of data.
80 bytes from 10.1.0.3: icmp_seq=1 ttl=64 time=0.175 ms
80 bytes from 10.1.0.3: icmp_seq=2 ttl=64 time=0.015 ms
80 bytes from 10.1.0.3: icmp_seq=3 ttl=64 time=0.009 ms

--- 10.1.0.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.009/0.066/0.175/0.076 ms, ipg/ewma 0.136/0.136 ms
SPINE2#ping 10.1.0.4 source loopback 0 repeat 3
PING 10.1.0.4 (10.1.0.4) from 10.1.0.2 : 72(100) bytes of data.
80 bytes from 10.1.0.4: icmp_seq=1 ttl=64 time=0.140 ms
80 bytes from 10.1.0.4: icmp_seq=2 ttl=64 time=0.015 ms
80 bytes from 10.1.0.4: icmp_seq=3 ttl=64 time=0.011 ms

--- 10.1.0.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.011/0.055/0.140/0.059 ms, ipg/ewma 0.117/0.110 ms
SPINE2#ping 10.1.0.5 source loopback 0 repeat 3
PING 10.1.0.5 (10.1.0.5) from 10.1.0.2 : 72(100) bytes of data.
80 bytes from 10.1.0.5: icmp_seq=1 ttl=64 time=0.159 ms
80 bytes from 10.1.0.5: icmp_seq=2 ttl=64 time=0.016 ms
80 bytes from 10.1.0.5: icmp_seq=3 ttl=64 time=0.012 ms

--- 10.1.0.5 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.012/0.062/0.159/0.068 ms, ipg/ewma 0.126/0.125 ms
```
</details>
