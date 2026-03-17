# Лабораторная работа 2 - Underlay OSPF

В данной работе настроим underlay сеть для работы по протоколу маршрутизации OSPF.<br>
Используем сеть, собранную в первой лабораторной работе как основу для дальнейшей модернизации.<br>
![LAP-topology](https://github.com/dim-sem/OTUS-Homework/blob/main/Lab-2/clos-lab1.png "Topology")<br>
<br>
Все ноды сети будут в одной Area 0.<br>
По умолчанию включим для всех интерфейсов passive default.<br>
Анонсируемые сети укажем непосредственнно на интерфейсах на нодах.
Минимальная конфигурация OSPF на нодах будет содержать объявление протокола:<br>
```
router ospf OSPF_INSTANCE_ID
passive-interface default 
router-id _ROUTER-ID_
```
Здесь:<br>
_OSPF_INSTANCE_ID_ - ID OSPF процесса<br>
_ROUTER_ID_ - укажем принудительно router-id, для документирования и облегчения дальнейшей поддержки сети.<br>
И на интерфейсах включаем OSPF:
```
interface Ethernet1/1
  description TO LEAF-1
  no switchport
  mtu 9000
  ip address 10.2.1.0/31
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf 10 area 0.0.0.0
  ip ospf bfd
  no shutdown
  ```
  Как пример - настройки одного из интерфейсов SPINE-1, для остальных интерфейсов, связывающих SPINE и LEAF, настройки будет отличаться только адресацией. <br>
  Т.е. как отмечено ранее - отключаем passive-interface, чтобы интерфейс участвовал в обмене LSA и формиовании соседства. Явно указываем на интерфейсе, что он участвует в OSPF и относится к Area 0 и укажем тип сети point-to=point, т.к. на за интерфейсом только одно устройство.
  ```
  ip ospf network point-to-point
  no ip ospf passive-interface
  ```
 Проверим, работает ли OSPF:
```
SPINE-1# sh ip ospf nei
 OSPF Process ID 10 VRF default
 Total number of neighbors: 3
 Neighbor ID     Pri State            Up Time  Address         Interface
 10.1.1.1          1 FULL/ -          00:48:18 10.2.1.1        Eth1/1
 10.1.1.2          1 FULL/ -          00:22:05 10.2.1.3        Eth1/2
 10.1.1.3          1 FULL/ -          01:48:13 10.2.1.5        Eth1/3

SPINE-2# sh ip ospf nei
 OSPF Process ID 10 VRF default
 Total number of neighbors: 3
 Neighbor ID     Pri State            Up Time  Address         Interface
 10.1.1.1          1 FULL/ -          00:49:02 10.2.2.1        Eth1/1
 10.1.1.2          1 FULL/ -          00:22:49 10.2.2.3        Eth1/2
 10.1.1.3          1 FULL/ -          01:48:51 10.2.2.5        Eth1/3

LEAF-3# sh ip ospf nei
 OSPF Process ID 10 VRF default
 Total number of neighbors: 2
 Neighbor ID     Pri State            Up Time  Address         Interface
 10.0.1.0          1 FULL/ -          01:43:49 10.2.1.4        Eth1/1
 10.0.2.0          1 FULL/ -          01:43:45 10.2.2.4        Eth1/2
``` 
На нодах успешно сформировано соседство и прошел обмен LSA.<br>
Проверим таблицу маршрутизации (например на LEAF-1):
```
LEAF-1# sh ip ospf rou
 OSPF Process ID 10 VRF default, Routing Table
  (D) denotes route is directly attached      (R) denotes route is in RIB
  (L) denotes route label is in ULIB          (NHR) denotes next-hop is in RIB
10.0.1.0/32 (intra)(R) area 0.0.0.0
     via 10.2.1.0/Eth1/1  , cost 41 distance 110 (NHR)
10.0.2.0/32 (intra)(R) area 0.0.0.0
     via 10.2.2.0/Eth1/2  , cost 41 distance 110 (NHR)
10.1.1.1/32 (intra)(D) area 0.0.0.0
     via 10.1.1.1/Lo1*  , cost 1 distance 110 (NHR)
10.1.1.2/32 (intra)(R) area 0.0.0.0
     via 10.2.1.0/Eth1/1  , cost 81 distance 110 (NHR)
     via 10.2.2.0/Eth1/2  , cost 81 distance 110 (NHR)
10.1.1.3/32 (intra)(R) area 0.0.0.0
     via 10.2.1.0/Eth1/1  , cost 81 distance 110 (NHR)
     via 10.2.2.0/Eth1/2  , cost 81 distance 110 (NHR)
10.2.1.0/31 (intra)(D) area 0.0.0.0
     via 10.2.1.0/Eth1/1*  , cost 40 distance 110 (NHR)
10.2.1.2/31 (intra)(R) area 0.0.0.0
     via 10.2.1.0/Eth1/1  , cost 80 distance 110 (NHR)
10.2.1.4/31 (intra)(R) area 0.0.0.0
     via 10.2.1.0/Eth1/1  , cost 80 distance 110 (NHR)
10.2.2.0/31 (intra)(D) area 0.0.0.0
     via 10.2.2.0/Eth1/2*  , cost 40 distance 110 (NHR)
10.2.1.0/31 (intra)(D) area 0.0.0.0
     via 10.2.1.0/Eth1/1*  , cost 40 distance 110 (NHR)
10.2.1.2/31 (intra)(R) area 0.0.0.0
     via 10.2.1.0/Eth1/1  , cost 80 distance 110 (NHR)
10.2.1.4/31 (intra)(R) area 0.0.0.0
     via 10.2.1.0/Eth1/1  , cost 80 distance 110 (NHR)
10.2.2.0/31 (intra)(D) area 0.0.0.0
     via 10.2.2.0/Eth1/2*  , cost 40 distance 110 (NHR)
10.2.2.2/31 (intra)(R) area 0.0.0.0
     via 10.2.2.0/Eth1/2  , cost 80 distance 110 (NHR)
10.2.2.4/31 (intra)(R) area 0.0.0.0
     via 10.2.2.0/Eth1/2  , cost 80 distance 110 (NHR)
172.16.1.0/24 (intra)(D) area 0.0.0.0
     via 172.16.1.0/Eth1/4*  , cost 40 distance 110 (NHR)
172.16.2.0/24 (intra)(R) area 0.0.0.0
     via 10.2.1.0/Eth1/1  , cost 120 distance 110 (NHR)
     via 10.2.2.0/Eth1/2  , cost 120 distance 110 (NHR)
172.16.3.0/24 (intra)(R) area 0.0.0.0
     via 10.2.1.0/Eth1/1  , cost 120 distance 110 (NHR)
     via 10.2.2.0/Eth1/2  , cost 120 distance 110 (NHR)
172.16.4.0/24 (intra)(R) area 0.0.0.0
     via 10.2.1.0/Eth1/1  , cost 120 distance 110 (NHR)
     via 10.2.2.0/Eth1/2  , cost 120 distance 110 (NHR)
172.16.5.0/24 (intra)(R) area 0.0.0.0
     via 10.2.1.0/Eth1/1  , cost 120 distance 110 (NHR)
     via 10.2.2.0/Eth1/2  , cost 120 distance 110 (NHR)
```
Как видим все сети, которые анонсоровали видны в таблице маршрутизации.<br>
Проверим, правда ли это:
```
VPC1> ping 172.16.5.2

84 bytes from 172.16.5.2 icmp_seq=1 ttl=61 time=12.824 ms
84 bytes from 172.16.5.2 icmp_seq=2 ttl=61 time=10.546 ms
84 bytes from 172.16.5.2 icmp_seq=3 ttl=61 time=8.583 ms
84 bytes from 172.16.5.2 icmp_seq=4 ttl=61 time=8.360 ms
84 bytes from 172.16.5.2 icmp_seq=5 ttl=61 time=9.185 ms

VPC5> ping 172.16.1.2

84 bytes from 172.16.1.2 icmp_seq=1 ttl=61 time=15.020 ms
84 bytes from 172.16.1.2 icmp_seq=2 ttl=61 time=15.514 ms
84 bytes from 172.16.1.2 icmp_seq=3 ttl=61 time=8.960 ms
84 bytes from 172.16.1.2 icmp_seq=4 ttl=61 time=9.064 ms
84 bytes from 172.16.1.2 icmp_seq=5 ttl=61 time=10.117 ms

VPC1> ping 10.2.1.3

84 bytes from 10.2.1.3 icmp_seq=1 ttl=253 time=7.577 ms
84 bytes from 10.2.1.3 icmp_seq=2 ttl=253 time=7.699 ms
84 bytes from 10.2.1.3 icmp_seq=3 ttl=253 time=8.959 ms
84 bytes from 10.2.1.3 icmp_seq=4 ttl=253 time=7.976 ms
84 bytes from 10.2.1.3 icmp_seq=5 ttl=253 time=7.159 ms
```
Хосты, подключенные к разным LEAF видят друг друга, т.е. всё работает.<br>

Вывод: OSPF - это быстро и достаточно просто, если помнить о некотором отличии синтаксиса и полноты команд между разными устройствами даже у одного производителя.<br>


