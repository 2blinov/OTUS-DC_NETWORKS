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
80 bytes from 10.1.2.1: icmp_seq=1 ttl=64 time=0.082 ms
80 bytes from 10.1.2.1: icmp_seq=2 ttl=64 time=0.014 ms
80 bytes from 10.1.2.1: icmp_seq=3 ttl=64 time=0.010 ms
--- 10.1.2.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.010/0.035/0.082/0.033 ms, ipg/ewma 0.083/0.065 ms

SPINE1#ping 10.1.2.3 repeat 3
PING 10.1.2.3 (10.1.2.3) 72(100) bytes of data.
80 bytes from 10.1.2.3: icmp_seq=1 ttl=64 time=0.089 ms
80 bytes from 10.1.2.3: icmp_seq=2 ttl=64 time=0.014 ms
80 bytes from 10.1.2.3: icmp_seq=3 ttl=64 time=0.011 ms
--- 10.1.2.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.011/0.038/0.089/0.036 ms, ipg/ewma 0.087/0.071 ms

SPINE1#ping 10.1.2.5 repeat 3
PING 10.1.2.5 (10.1.2.5) 72(100) bytes of data.
80 bytes from 10.1.2.5: icmp_seq=1 ttl=64 time=0.078 ms
80 bytes from 10.1.2.5: icmp_seq=2 ttl=64 time=0.016 ms
80 bytes from 10.1.2.5: icmp_seq=3 ttl=64 time=0.011 ms
--- 10.1.2.5 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.011/0.035/0.078/0.030 ms, ipg/ewma 0.081/0.062 ms
```
</details>

<details>
<summary>Доступность транспортных адресов соседей SPINE2 -> LEAFX</summary>
  
```eos
SPINE2#ping 10.1.2.7 repeat 3
PING 10.1.2.7 (10.1.2.7) 72(100) bytes of data.
80 bytes from 10.1.2.7: icmp_seq=1 ttl=64 time=0.217 ms
80 bytes from 10.1.2.7: icmp_seq=2 ttl=64 time=0.018 ms
80 bytes from 10.1.2.7: icmp_seq=3 ttl=64 time=0.017 ms
--- 10.1.2.7 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.017/0.084/0.217/0.094 ms, ipg/ewma 0.185/0.170 ms

SPINE2#ping 10.1.2.9 repeat 3
PING 10.1.2.9 (10.1.2.9) 72(100) bytes of data.
80 bytes from 10.1.2.9: icmp_seq=1 ttl=64 time=0.208 ms
80 bytes from 10.1.2.9: icmp_seq=2 ttl=64 time=0.017 ms
80 bytes from 10.1.2.9: icmp_seq=3 ttl=64 time=0.009 ms
--- 10.1.2.9 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.009/0.078/0.208/0.091 ms, ipg/ewma 0.141/0.162 ms

SPINE2#ping 10.1.2.11 repeat 3
PING 10.1.2.11 (10.1.2.11) 72(100) bytes of data.
80 bytes from 10.1.2.11: icmp_seq=1 ttl=64 time=0.161 ms
80 bytes from 10.1.2.11: icmp_seq=2 ttl=64 time=0.014 ms
80 bytes from 10.1.2.11: icmp_seq=3 ttl=64 time=0.012 ms
--- 10.1.2.11 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.012/0.062/0.161/0.069 ms, ipg/ewma 0.126/0.126 ms
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
10.1.2.1           112724123       1740842210            Ethernet1(235)       normal       08/12/26 11:44             NA       No Diagnostic       Up
10.1.2.3            65125797       1052311716            Ethernet2(236)       normal       08/12/26 11:44             NA       No Diagnostic       Up
10.1.2.5          1163616502       2330403263            Ethernet3(232)       normal       08/12/26 11:45             NA       No Diagnostic       Up
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
10.1.2.7           2298128045       3300776703            Ethernet1(239)       normal       08/12/26 11:44             NA       No Diagnostic       Up
10.1.2.9           1658097009       2867581067            Ethernet2(242)       normal       08/12/26 11:44             NA       No Diagnostic       Up
10.1.2.11          2165676111        515240408            Ethernet3(240)       normal       08/12/26 11:45             NA       No Diagnostic       Up
```
</details>


<details>
<summary>SPINE1 / show ip ospf neighbor </summary>
  
```eos
SPINE1#show ip ospf neighbor 
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.1.0.5        100      default  0   FULL                   00:00:38    10.1.2.5        Ethernet3
10.1.0.3        100      default  0   FULL                   00:00:29    10.1.2.1        Ethernet1
10.1.0.4        100      default  0   FULL                   00:00:31    10.1.2.3        Ethernet2
```
</details>

<details>
<summary>SPINE2 / show ip ospf neighbor</summary>
  
```eos
SPINE2#show ip ospf neighbor 
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.1.0.4        100      default  0   FULL                   00:00:32    10.1.2.9        Ethernet2
10.1.0.3        100      default  0   FULL                   00:00:33    10.1.2.7        Ethernet1
10.1.0.5        100      default  0   FULL                   00:00:33    10.1.2.11       Ethernet3
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
80 bytes from 10.1.0.2: icmp_seq=1 ttl=63 time=0.958 ms
80 bytes from 10.1.0.2: icmp_seq=2 ttl=63 time=0.667 ms
80 bytes from 10.1.0.2: icmp_seq=3 ttl=63 time=0.344 ms
--- 10.1.0.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2ms
rtt min/avg/max/mdev = 0.344/0.656/0.958/0.250 ms, ipg/ewma 1.028/0.849 ms

SPINE1#ping 10.1.0.3 source loopback 0 repeat 3
PING 10.1.0.3 (10.1.0.3) from 10.1.0.1 : 72(100) bytes of data.
80 bytes from 10.1.0.3: icmp_seq=1 ttl=64 time=0.121 ms
80 bytes from 10.1.0.3: icmp_seq=2 ttl=64 time=0.015 ms
80 bytes from 10.1.0.3: icmp_seq=3 ttl=64 time=0.011 ms
--- 10.1.0.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.011/0.049/0.121/0.050 ms, ipg/ewma 0.104/0.095 ms

SPINE1#ping 10.1.0.4 source loopback 0 repeat 3
PING 10.1.0.4 (10.1.0.4) from 10.1.0.1 : 72(100) bytes of data.
80 bytes from 10.1.0.4: icmp_seq=1 ttl=64 time=0.159 ms
80 bytes from 10.1.0.4: icmp_seq=2 ttl=64 time=0.019 ms
80 bytes from 10.1.0.4: icmp_seq=3 ttl=64 time=0.013 ms
--- 10.1.0.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.013/0.063/0.159/0.067 ms, ipg/ewma 0.131/0.125 ms

SPINE1#ping 10.1.0.5 source loopback 0 repeat 3
PING 10.1.0.5 (10.1.0.5) from 10.1.0.1 : 72(100) bytes of data.
80 bytes from 10.1.0.5: icmp_seq=1 ttl=64 time=0.229 ms
80 bytes from 10.1.0.5: icmp_seq=2 ttl=64 time=0.018 ms
80 bytes from 10.1.0.5: icmp_seq=3 ttl=64 time=0.013 ms
--- 10.1.0.5 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.013/0.086/0.229/0.100 ms, ipg/ewma 0.168/0.179 ms
```
</details>

<details>
<summary>SPINE2 / ping до Lo0 устройств фабрики</summary>
  
```eos
SPINE2#ping 10.1.0.1 source loopback 0 repeat 3
PING 10.1.0.1 (10.1.0.1) from 10.1.0.2 : 72(100) bytes of data.
80 bytes from 10.1.0.1: icmp_seq=1 ttl=63 time=0.610 ms
80 bytes from 10.1.0.1: icmp_seq=2 ttl=63 time=0.307 ms
80 bytes from 10.1.0.1: icmp_seq=3 ttl=63 time=0.323 ms
--- 10.1.0.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2ms
rtt min/avg/max/mdev = 0.307/0.413/0.610/0.139 ms, ipg/ewma 1.001/0.541 ms

SPINE2#ping 10.1.0.3 source loopback 0 repeat 3
PING 10.1.0.3 (10.1.0.3) from 10.1.0.2 : 72(100) bytes of data.
80 bytes from 10.1.0.3: icmp_seq=1 ttl=64 time=0.110 ms
80 bytes from 10.1.0.3: icmp_seq=2 ttl=64 time=0.014 ms
80 bytes from 10.1.0.3: icmp_seq=3 ttl=64 time=0.013 ms
--- 10.1.0.3 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.013/0.045/0.110/0.045 ms, ipg/ewma 0.096/0.087 ms

SPINE2#ping 10.1.0.4 source loopback 0 repeat 3
PING 10.1.0.4 (10.1.0.4) from 10.1.0.2 : 72(100) bytes of data.
80 bytes from 10.1.0.4: icmp_seq=1 ttl=64 time=0.115 ms
80 bytes from 10.1.0.4: icmp_seq=2 ttl=64 time=0.014 ms
80 bytes from 10.1.0.4: icmp_seq=3 ttl=64 time=0.011 ms
--- 10.1.0.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.011/0.046/0.115/0.048 ms, ipg/ewma 0.099/0.091 ms

SPINE2#ping 10.1.0.5 source loopback 0 repeat 3
PING 10.1.0.5 (10.1.0.5) from 10.1.0.2 : 72(100) bytes of data.
80 bytes from 10.1.0.5: icmp_seq=1 ttl=64 time=0.118 ms
80 bytes from 10.1.0.5: icmp_seq=2 ttl=64 time=0.014 ms
80 bytes from 10.1.0.5: icmp_seq=3 ttl=64 time=0.011 ms
--- 10.1.0.5 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.011/0.047/0.118/0.049 ms, ipg/ewma 0.101/0.093 ms
```
</details>
