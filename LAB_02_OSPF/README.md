# Лабораторная работа №2 "Построение Underlay сети (OSPF)"

## Задание:
1. [Подготовка стенда](#1-подготовка-стенда)
2. [Разработка адресного плана](#2-разработка-адресного-плана)
3. [Настройка Underlay на базе OSPF](#3-настройка-underlay-на-базе-ospf)
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

## 3. Настройка Underlay на базе OSPF
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
<summary>Контекст: Процесс OSPF</summary>

```eos
router ospf 100
   router-id 10.1.0.1              # Явно настраиваем Router ID
   bfd default                     # Включаем BFD для OSPF
   passive-interface default       # Не пытаться устанавливать OSPF-соседство на всех интерфейсах по умолчанию
   no passive-interface Ethernet1  # Кроме тех интерфейсов, на которых мы это это разрешаем.
   no passive-interface Ethernet2
   no passive-interface Ethernet3
```
</details>
<details>
<summary>Контекст: Интерфейс</summary>

```eos
interface Ethernet1
   description P2P-LEAF1
   mtu 9214                                                      # Увеличиваем MTU
   no switchport 
   ip address 10.1.2.0/31
   bfd interval 100 min_rx 100 multiplier 3                      # Настраиваем таймеры BFD
   ip ospf dead-interval 3                                       # Таймеры OSPF: DEAD-интервал (Таймаут (в секундах) для признания соседа отсутствующим, 3xHELLO)
   ip ospf hello-interval 1                                      # Таймеры OSPF: HELLO-интервал (Время (в секундах) между посылками HELLO-пакетов)
   ip ospf network point-to-point                                # Тип сети OSPF точка-точка (нет выборов DR/BDR)
   ip ospf authentication message-digest                         # Включаем аутентификацию OSPF на интерфейсе
   ip ospf area 0.0.0.0                                          # Настраиваем принадлежность интерфейса Backbone-зоне
   ip ospf message-digest-key 1 md5 7 OTlMNR5qzL6ViDjejl1JwA==   # Настраиваем ключ, используемый при аутентификации
```
</details>

[Конфигурация Spine1](./configs/spine1.cfg)<br>
[Конфигурация Spine2](./configs/spine2.cfg)<br>
[Конфигурация Leaf1](./configs/leaf1.cfg)<br>
[Конфигурация Leaf2](./configs/leaf2.cfg)<br>
[Конфигурация Leaf3](./configs/leaf3.cfg)<br>

## 4. Проверка связности
<details>
<summary>Доступность транспортных адресов соседей SPINE1 -> LEAFX</summary>
  
```eos
SPINE1#ping 10.1.2.1 repeat 3
PING 10.1.2.1 (10.1.2.1) 72(100) bytes of data.
80 bytes from 10.1.2.1: icmp_seq=1 ttl=64 time=0.233 ms
80 bytes from 10.1.2.1: icmp_seq=2 ttl=64 time=0.018 ms
80 bytes from 10.1.2.1: icmp_seq=3 ttl=64 time=0.011 ms

--- 10.1.2.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.011/0.087/0.233/0.103 ms, ipg/ewma 0.183/0.181 ms
SPINE1#ping 10.1.2.3 repeat 3
PING 10.1.2.3 (10.1.2.3) 72(100) bytes of data.
80 bytes from 10.1.2.3: icmp_seq=1 ttl=64 time=0.158 ms
80 bytes from 10.1.2.3: icmp_seq=2 ttl=64 time=0.018 ms
80 bytes from 10.1.2.3: icmp_seq=3 ttl=64 time=0.015 ms

--- 10.1.2.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.015/0.063/0.158/0.066 ms, ipg/ewma 0.131/0.124 ms
SPINE1#ping 10.1.2.5 repeat 3
PING 10.1.2.5 (10.1.2.5) 72(100) bytes of data.
80 bytes from 10.1.2.5: icmp_seq=1 ttl=64 time=0.143 ms
80 bytes from 10.1.2.5: icmp_seq=2 ttl=64 time=0.015 ms
80 bytes from 10.1.2.5: icmp_seq=3 ttl=64 time=0.010 ms

--- 10.1.2.5 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.010/0.056/0.143/0.061 ms, ipg/ewma 0.115/0.112 ms
```
</details>

<details>
<summary>Доступность транспортных адресов соседей SPINE2 -> LEAFX</summary>
  
```eos
SPINE2#ping 10.1.2.7 repeat 3
PING 10.1.2.7 (10.1.2.7) 72(100) bytes of data.
80 bytes from 10.1.2.7: icmp_seq=1 ttl=64 time=0.210 ms
80 bytes from 10.1.2.7: icmp_seq=2 ttl=64 time=0.019 ms
80 bytes from 10.1.2.7: icmp_seq=3 ttl=64 time=0.017 ms

--- 10.1.2.7 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.017/0.082/0.210/0.090 ms, ipg/ewma 0.180/0.165 ms
SPINE2#ping 10.1.2.9 repeat 3
PING 10.1.2.9 (10.1.2.9) 72(100) bytes of data.
80 bytes from 10.1.2.9: icmp_seq=1 ttl=64 time=0.166 ms
80 bytes from 10.1.2.9: icmp_seq=2 ttl=64 time=0.014 ms
80 bytes from 10.1.2.9: icmp_seq=3 ttl=64 time=0.011 ms

--- 10.1.2.9 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.011/0.063/0.166/0.072 ms, ipg/ewma 0.129/0.130 ms
SPINE2#ping 10.1.2.11 repeat 3
PING 10.1.2.11 (10.1.2.11) 72(100) bytes of data.
80 bytes from 10.1.2.11: icmp_seq=1 ttl=64 time=0.192 ms
80 bytes from 10.1.2.11: icmp_seq=2 ttl=64 time=0.014 ms
80 bytes from 10.1.2.11: icmp_seq=3 ttl=64 time=0.009 ms

--- 10.1.2.11 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.009/0.071/0.192/0.085 ms, ipg/ewma 0.140/0.149 ms
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
10.1.2.1           595824028       1573309871            Ethernet1(235)       normal       08/18/26 15:49             NA       No Diagnostic       Up
10.1.2.3          3910891998        843495901            Ethernet2(236)       normal       08/18/26 15:49             NA       No Diagnostic       Up
10.1.2.5           313397289        815350499            Ethernet3(232)       normal       08/18/26 15:49             NA       No Diagnostic       Up
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
10.1.2.7           1837659122       4020764370            Ethernet1(239)       normal       08/18/26 15:49             NA       No Diagnostic       Up
10.1.2.9            338608076        456061250            Ethernet2(242)       normal       08/18/26 15:49             NA       No Diagnostic       Up
10.1.2.11          3855739668       3272348628            Ethernet3(240)       normal       08/18/26 15:49             NA       No Diagnostic       Up
```
</details>


<details>
<summary>SPINE1 / show ip ospf neighbor </summary>
  
```eos
SPINE1#show ip ospf neighbor
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.1.0.3        100      default  0   FULL                   00:00:01    10.1.2.1        Ethernet1
10.1.0.4        100      default  0   FULL                   00:00:01    10.1.2.3        Ethernet2
10.1.0.5        100      default  0   FULL                   00:00:01    10.1.2.5        Ethernet3
```
</details>

<details>
<summary>SPINE2 / show ip ospf neighbor</summary>
  
```eos
SPINE2#show ip ospf neighbor
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.1.0.3        100      default  0   FULL                   00:00:01    10.1.2.7        Ethernet1
10.1.0.4        100      default  0   FULL                   00:00:01    10.1.2.9        Ethernet2
10.1.0.5        100      default  0   FULL                   00:00:01    10.1.2.11       Ethernet3
```
</details>

<details>
<summary>SPINE1 / show ip route ospf</summary>
  
```eos
SPINE1#show ip route ospf

VRF: default
 O        10.1.0.2/32 [110/30]
           via 10.1.2.1, Ethernet1
           via 10.1.2.3, Ethernet2
           via 10.1.2.5, Ethernet3
 O        10.1.0.3/32 [110/20]
           via 10.1.2.1, Ethernet1
 O        10.1.0.4/32 [110/20]
           via 10.1.2.3, Ethernet2
 O        10.1.0.5/32 [110/20]
           via 10.1.2.5, Ethernet3
 O        10.1.2.6/31 [110/20]
           via 10.1.2.1, Ethernet1
 O        10.1.2.8/31 [110/20]
           via 10.1.2.3, Ethernet2
 O        10.1.2.10/31 [110/20]
           via 10.1.2.5, Ethernet3

```
</details>

<details>
<summary>SPINE2 / show ip route ospf</summary>
  
```eos
SPINE2#show ip route ospf

VRF: default
 O        10.1.0.1/32 [110/30]
           via 10.1.2.7, Ethernet1
           via 10.1.2.9, Ethernet2
           via 10.1.2.11, Ethernet3
 O        10.1.0.3/32 [110/20]
           via 10.1.2.7, Ethernet1
 O        10.1.0.4/32 [110/20]
           via 10.1.2.9, Ethernet2
 O        10.1.0.5/32 [110/20]
           via 10.1.2.11, Ethernet3
 O        10.1.2.0/31 [110/20]
           via 10.1.2.7, Ethernet1
 O        10.1.2.2/31 [110/20]
           via 10.1.2.9, Ethernet2
 O        10.1.2.4/31 [110/20]
           via 10.1.2.11, Ethernet3

```
</details>

<details>
<summary>SPINE1 / ping до Lo0 устройств фабрики</summary>
  
```eos
SPINE1#ping 10.1.0.2 source loopback 0 repeat 3
PING 10.1.0.2 (10.1.0.2) from 10.1.0.1 : 72(100) bytes of data.
80 bytes from 10.1.0.2: icmp_seq=1 ttl=63 time=1.67 ms
80 bytes from 10.1.0.2: icmp_seq=2 ttl=63 time=0.368 ms
80 bytes from 10.1.0.2: icmp_seq=3 ttl=63 time=0.343 ms

--- 10.1.0.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 4ms
rtt min/avg/max/mdev = 0.343/0.794/1.673/0.621 ms, ipg/ewma 2.008/1.364 ms
SPINE1#ping 10.1.0.3 source loopback 0 repeat 3
PING 10.1.0.3 (10.1.0.3) from 10.1.0.1 : 72(100) bytes of data.
80 bytes from 10.1.0.3: icmp_seq=1 ttl=64 time=0.185 ms
80 bytes from 10.1.0.3: icmp_seq=2 ttl=64 time=0.013 ms
80 bytes from 10.1.0.3: icmp_seq=3 ttl=64 time=0.010 ms

--- 10.1.0.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.010/0.069/0.185/0.081 ms, ipg/ewma 0.145/0.144 ms
SPINE1#ping 10.1.0.4 source loopback 0 repeat 3
PING 10.1.0.4 (10.1.0.4) from 10.1.0.1 : 72(100) bytes of data.
80 bytes from 10.1.0.4: icmp_seq=1 ttl=64 time=0.247 ms
80 bytes from 10.1.0.4: icmp_seq=2 ttl=64 time=0.014 ms
80 bytes from 10.1.0.4: icmp_seq=3 ttl=64 time=0.010 ms

--- 10.1.0.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.010/0.090/0.247/0.110 ms, ipg/ewma 0.178/0.192 ms
SPINE1#ping 10.1.0.5 source loopback 0 repeat 3
PING 10.1.0.5 (10.1.0.5) from 10.1.0.1 : 72(100) bytes of data.
80 bytes from 10.1.0.5: icmp_seq=1 ttl=64 time=0.157 ms
80 bytes from 10.1.0.5: icmp_seq=2 ttl=64 time=0.017 ms
80 bytes from 10.1.0.5: icmp_seq=3 ttl=64 time=0.013 ms

--- 10.1.0.5 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.013/0.062/0.157/0.066 ms, ipg/ewma 0.141/0.123 ms
```
</details>

<details>
<summary>SPINE2 / ping до Lo0 устройств фабрики</summary>
  
```eos
SPINE2#ping 10.1.0.1 source loopback 0 repeat 3
PING 10.1.0.1 (10.1.0.1) from 10.1.0.2 : 72(100) bytes of data.
80 bytes from 10.1.0.1: icmp_seq=1 ttl=63 time=0.538 ms
80 bytes from 10.1.0.1: icmp_seq=2 ttl=63 time=0.290 ms
80 bytes from 10.1.0.1: icmp_seq=3 ttl=63 time=0.290 ms

--- 10.1.0.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2ms
rtt min/avg/max/mdev = 0.290/0.372/0.538/0.116 ms, ipg/ewma 1.001/0.479 ms
SPINE2#ping 10.1.0.3 source loopback 0 repeat 3
PING 10.1.0.3 (10.1.0.3) from 10.1.0.2 : 72(100) bytes of data.
80 bytes from 10.1.0.3: icmp_seq=1 ttl=64 time=0.091 ms
80 bytes from 10.1.0.3: icmp_seq=2 ttl=64 time=0.015 ms
80 bytes from 10.1.0.3: icmp_seq=3 ttl=64 time=0.011 ms

--- 10.1.0.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.011/0.039/0.091/0.036 ms, ipg/ewma 0.098/0.072 ms
SPINE2#ping 10.1.0.4 source loopback 0 repeat 3
PING 10.1.0.4 (10.1.0.4) from 10.1.0.2 : 72(100) bytes of data.
80 bytes from 10.1.0.4: icmp_seq=1 ttl=64 time=0.146 ms
80 bytes from 10.1.0.4: icmp_seq=2 ttl=64 time=0.013 ms
80 bytes from 10.1.0.4: icmp_seq=3 ttl=64 time=0.008 ms

--- 10.1.0.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.008/0.055/0.146/0.063 ms, ipg/ewma 0.119/0.114 ms
SPINE2#ping 10.1.0.5 source loopback 0 repeat 3
PING 10.1.0.5 (10.1.0.5) from 10.1.0.2 : 72(100) bytes of data.
80 bytes from 10.1.0.5: icmp_seq=1 ttl=64 time=0.147 ms
80 bytes from 10.1.0.5: icmp_seq=2 ttl=64 time=0.020 ms
80 bytes from 10.1.0.5: icmp_seq=3 ttl=64 time=0.016 ms

--- 10.1.0.5 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.016/0.061/0.147/0.060 ms, ipg/ewma 0.135/0.116 ms
```
</details>
