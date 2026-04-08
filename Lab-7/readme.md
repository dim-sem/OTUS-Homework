# Лабораторная работа 7 - Multi-homing

Рассмотрим как работает VxLAN в случае Multi-homing.<br> 
На Cisco Nexus реализуется технология vPC - Virtual Port-Channel. Имеем в виду, что по данной технлогии мы можем соединить не более 2 узлов, т.е. сервер можем подключит только к 2 LEAF. Настроим представленную ниже топологию:

![LAP-topology](https://github.com/dim-sem/OTUS-Homework/blob/main/Lab-7/topology.png "Topology")<br>
<br>
Для этого сначала добавим поддержку vPC и LACP на LEAF-1 и LEAF-2 и сделаем аналогичные настройки на LEAF-3 и LEAF-4.<br>
```
feature vpc
feature lacp
```
Для того, чтобы vPC работал нам нужно поднять keeplive link и peer-link. Keepalive link служит для мониторинга работосопособности узлов в vPC, это обычный L3 интерфейс, поскольку нужно отделять его от data traffic поднимем его на management interface, который на Cisco Nexus находится от отдельном VRF Management:<br>
```
LEAF-1:

interface mgmt0
  vfr member management
  ip address 10.100.10.1/30

LEAF-2:

interface mgmt0
  vfr member management
  ip address 10.100.10.2/30
```
  После этого настроим vPC и peer-link (который служит для синхронизации MAC-адресов, VLAN, состояний портов и передачи служебного трафика):<br>
```
!
!  LEAF-1
!
vpc domain 1
!
! ID должен совпадать на обеих нодах vPC
!
peer-switch
peer-keepalive destination 10.100.10.2 source 10.100.10.1
peer-gateway
ip arp synchronize
```
На LEAF-2 настройки аналогичные, кроме того, что в peer-keepalive адреса указаны в обратном порядке:<br>
```
peer-keepalive destination 10.100.10.2 source 10.100.10.2
```
Создадим peer-link на узлах и настроим Port-chennel до сервера:
```

interface port-channel20
  switchport mode trunk
  vpc 20

interface port-channel1000
  switchport mode trunk
  spanning-tree port type network
  vpc peer-link

interface Ethernet1/3
  description Trunk to SERVER-1
  switchport mode trunk
  channel-group 20 mode active

interface Ethernet1/5
  switchport mode trunk
  channel-group 1000 mode active

interface Ethernet1/6
  switchport mode trunk
  channel-group 1000 mode active

interface Ethernet1/7
  description Trunk to SERVER-1
  switchport mode trunk
  channel-group 20 mode active
```
Проверим корректность настроек и работоспособность:
```

LEAF-1# sh vpc
Legend:
                (*) - local vPC is down, forwarding via vPC peer-link

vPC domain id                     : 1
Peer status                       : peer adjacency formed ok
vPC keep-alive status             : peer is alive
Configuration consistency status  : success
Per-vlan consistency status       : success
Type-2 consistency status         : success
vPC role                          : primary
Number of vPCs configured         : 1
Peer Gateway                      : Enabled
Dual-active excluded VLANs        : -
Graceful Consistency Check        : Enabled
Auto-recovery status              : Disabled
Delay-restore status              : Timer is off.(timeout = 30s)
Delay-restore SVI status          : Timer is off.(timeout = 10s)
Operational Layer3 Peer-router    : Disabled
Virtual-peerlink mode             : Disabled

vPC Peer-link status
---------------------------------------------------------------------
id    Port   Status Active vlans
--    ----   ------ -------------------------------------------------
1     Po1000 up     1,10,20,999


vPC status
----------------------------------------------------------------------------
Id    Port          Status Consistency Reason                Active vlans
--    ------------  ------ ----------- ------                ---------------
20    Po20          up     success     success               1,10,20,999
```
Как видим - vPC нормально поднялся и работает без проблем. Видно, что в нем все активные VLAN, которые мы создавали на ноде, если нужно ограничить скоп VLAN, то на интерфейсах нужно указать какие VLAN разрешены.<br>
Чтобы у нас vCPC и VxLAN жили вместе дружно и счастливо нам поребуется на интерфейсе Loobback1, который привязан к VxLAN добавить secondary IP.<br>
!!! Это нужно сделать на обеих нодаи и этот IP должен быть одинаковым!!!
```
interface Loobback1
 ip address 10.1.1.101/32 secondary
```
Аналогично настроим vPC на паре LEAF-3 и LEAF-4, добавим там хост SERVER-3 и настроим на нем интерфейсы.
Проверим доступность интерфейсов SERVER-1 и SERVER-2 с SERVER-3:
```
SERVER-3# ping 172.16.10.10
PING 172.16.10.10 (172.16.10.10): 56 data bytes
64 bytes from 172.16.10.10: icmp_seq=0 ttl=254 time=13.358 ms
64 bytes from 172.16.10.10: icmp_seq=1 ttl=254 time=10.765 ms
64 bytes from 172.16.10.10: icmp_seq=2 ttl=254 time=11.177 ms
64 bytes from 172.16.10.10: icmp_seq=3 ttl=254 time=12.434 ms
64 bytes from 172.16.10.10: icmp_seq=4 ttl=254 time=12.209 ms

--- 172.16.10.10 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 10.765/11.988/13.358 ms
SERVER-3#
SERVER-3# ping 172.16.10.11
PING 172.16.10.11 (172.16.10.11): 56 data bytes
64 bytes from 172.16.10.11: icmp_seq=0 ttl=254 time=10.451 ms
64 bytes from 172.16.10.11: icmp_seq=1 ttl=254 time=13.815 ms
64 bytes from 172.16.10.11: icmp_seq=2 ttl=254 time=9.634 ms
64 bytes from 172.16.10.11: icmp_seq=3 ttl=254 time=12.4 ms
64 bytes from 172.16.10.11: icmp_seq=4 ttl=254 time=12.351 ms

--- 172.16.10.11 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 9.634/11.73/13.815 ms

SERVER-3# ping 172.16.20.10
PING 172.16.20.10 (172.16.20.10): 56 data bytes
64 bytes from 172.16.20.10: icmp_seq=0 ttl=254 time=10.288 ms
64 bytes from 172.16.20.10: icmp_seq=1 ttl=254 time=11.608 ms
64 bytes from 172.16.20.10: icmp_seq=2 ttl=254 time=10.675 ms
64 bytes from 172.16.20.10: icmp_seq=3 ttl=254 time=12.399 ms
64 bytes from 172.16.20.10: icmp_seq=4 ttl=254 time=14.242 ms

--- 172.16.20.10 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 10.288/11.842/14.242 ms
SERVER-3#
SERVER-3# ping 172.16.20.11
PING 172.16.20.11 (172.16.20.11): 56 data bytes
64 bytes from 172.16.20.11: icmp_seq=0 ttl=254 time=13.776 ms
64 bytes from 172.16.20.11: icmp_seq=1 ttl=254 time=11.285 ms
64 bytes from 172.16.20.11: icmp_seq=2 ttl=254 time=10.685 ms
64 bytes from 172.16.20.11: icmp_seq=3 ttl=254 time=10.919 ms
64 bytes from 172.16.20.11: icmp_seq=4 ttl=254 time=11.657 ms
```
Обратно пинги тоже проходят:<br>
```
SERVER-1# ping 172.16.10.100
PING 172.16.10.100 (172.16.10.100): 56 data bytes
64 bytes from 172.16.10.100: icmp_seq=0 ttl=254 time=12.172 ms
64 bytes from 172.16.10.100: icmp_seq=1 ttl=254 time=9.064 ms
64 bytes from 172.16.10.100: icmp_seq=2 ttl=254 time=8.193 ms
64 bytes from 172.16.10.100: icmp_seq=3 ttl=254 time=7.197 ms
64 bytes from 172.16.10.100: icmp_seq=4 ttl=254 time=7.944 ms

--- 172.16.10.100 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 7.197/8.914/12.172 ms

SERVER-1# ping 172.16.20.100
PING 172.16.20.100 (172.16.20.100): 56 data bytes
64 bytes from 172.16.20.100: icmp_seq=0 ttl=254 time=9.23 ms
64 bytes from 172.16.20.100: icmp_seq=1 ttl=254 time=12.01 ms
64 bytes from 172.16.20.100: icmp_seq=2 ttl=254 time=9.17 ms
64 bytes from 172.16.20.100: icmp_seq=3 ttl=254 time=15.817 ms
64 bytes from 172.16.20.100: icmp_seq=4 ttl=254 time=11.76 ms

--- 172.16.20.100 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 9.17/11.597/15.817 ms
```
Помотрим, что нам скажет sh nve peers:
```
LEAF-4# sh nve peers
Interface Peer-IP                                 State LearnType Uptime   Route
r-Mac
--------- --------------------------------------  ----- --------- -------- -----
------------
nve1      10.1.1.101                              Up    CP        18:49:23 5004.
0000.1b08

LEAF-1# sh nve peers
Interface Peer-IP                                 State LearnType Uptime   Route
r-Mac
--------- --------------------------------------  ----- --------- -------- -----
------------
nve1      10.1.1.102                              Up    CP        19:16:20 5009.
0000.1b08
```
Обратим внимание на то, что теперь в качестве next-hop адреса пира указан secondary address, который мы указывали на Looopback!
И в выводе конамды sh bgp all это хорошо видно:
```
LEAF-4# sh bgp all
BGP routing table information for VRF default, address family VPNv4 Unicast
BGP table version is 10, Local Router ID is 10.1.1.4
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 10.1.1.4:3    (VRF L3VNI_TEST)
* i172.16.10.10/32    10.1.1.101                        100          0 i
*>i                   10.1.1.101                        100          0 i
* i172.16.10.11/32    10.1.1.101                        100          0 i
*>i                   10.1.1.101                        100          0 i
* i172.16.20.10/32    10.1.1.101                        100          0 i
*>i                   10.1.1.101                        100          0 i
* i172.16.20.11/32    10.1.1.101                        100          0 i
*>i                   10.1.1.101                        100          0 i
```
Вывод: vPC работает, но нужно настраиват дополнительный IP на loopback, который привязан к nve интерфейсу, и этот IP должен совпадать на обеих нодах в vPC паре. Падение одного из LEAF не влияет на доступность сети для подключенного по multi-homing сервера.