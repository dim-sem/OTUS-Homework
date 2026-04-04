# Лабораторная работа 5 - L2EVPN

В данной лабораторной работе посмотрим как возможно растянуть L2 сеть в пределах фабрики. <br>
Поскольку лабораторная работа собрана на Cisco Nexus учтем рекомендацию вендора, в частности рекомендуется для Underlay network использовать IGP, в частности OSPF, а для Overlay network - iBGP. Попробуем следовать рекомендациям. Возьмем в  основу лабораторную работу 2 с OSPF и настроим overlay и VxLAN<br> 

![LAP-topology](https://github.com/dim-sem/OTUS-Homework/blob/main/Lab-5/lab5-topology.png "Topology")<br>

Для Cisco Nexus требуется активировать дополнительный фунционал на узлах сети:<br>
```
! Enable VLAN-based VXLAN: 
!
feature vn-segment
!
! Enable NV overlay functionality: 
!
feature nv overlay
!
! Enable VN-Segment for VLANs: 
!
feature vn-segment-vlan-based
!
! Enable Switch Virtual Interface (SVI) support: 
!
feature interface-vlan
!
! Activate the EVPN control plane for VXLAN: 
!
nv overlay evpn
```
На LEAF-1 и LEAF-2 настроим VLAN 10 и подключим к LEAF-ам хосты, которые будут в данном VLAN (VPC1 - LEAF-1 и VPC2-LEAF-2).<br>
Настройки Overlay сети на SPINE простые и хорошо поддаются шаблонизации ( в примере SPINE-1):<br>
```
! На SPINE
!
router bgp 65501
router-id 10.0.1.0
  template peer LEAF 
    remote-as 65001
    update-source loopback1
    address-family l2vpn evpn
      send-community
      send-community extended
      route-reflector-client
  neighbor 10.1.1.1
    inherit peer LEAF
  neighbor 10.1.1.2
    inherit peer LEAF
  neighbor 10.1.1.3
    inherit peer LEAF
```
В свою очередь, настройки на LEAF тоже очень простые ( в примере, LEAF-1):<br>
```
router bgp 65001
  router-id 10.1.1.1
  template peer SPINE
    remote-as 65001
    update-source loopback1
    address-family l2vpn evpn
      send-community
      send-community extended
  neighbor 10.0.1.0
    inherit peer SPINE
  neighbor 10.0.2.0
    inherit peer SPINE
```
Проверим на SPINE, что BGP сессии поднялись и у нас есть связность в Overlay сети:<br>
```
SPINE-2# sh bgp sessions
Total peers 3, established peers 3
ASN 65501
VRF default, local ASN 65501
peers 3, established peers 3, local router-id 10.0.2.0
State: I-Idle, A-Active, O-Open, E-Established, C-Closing, S-Shutdown

Neighbor        ASN    Flaps LastUpDn|LastRead|LastWrit St Port(L/R)  Notif(S/R)
10.1.1.1        65501 0     00:00:53|00:00:52|00:00:52 E   28890/179        0/0
10.1.1.2        65501 0     00:00:53|00:00:52|00:00:52 E   48633/179        0/0
10.1.1.3        65501 0     00:00:53|00:00:52|00:00:52 E   31990/179        0/0

SPINE-1# sh bgp l2vpn evpn sum
BGP summary information for VRF default, address family L2VPN EVPN
BGP router identifier 10.0.1.0, local AS number 65501
BGP table version is 5, L2VPN EVPN config peers 3, capable peers 3
0 network entries and 0 paths using 0 bytes of memory
BGP attribute entries [0/0], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [0/0]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.1.1.1        4 65501      11      11        5    0    0 00:05:38 0
10.1.1.2        4 65501      11      11        5    0    0 00:05:32 0
10.1.1.3        4 65501      11      11        5    0    0 00:05:28 0

SPINE-2# sh bgp l2vpn evpn sum
BGP summary information for VRF default, address family L2VPN EVPN
BGP router identifier 10.0.2.0, local AS number 65501
BGP table version is 5, L2VPN EVPN config peers 3, capable peers 3
0 network entries and 0 paths using 0 bytes of memory
BGP attribute entries [0/0], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [0/0]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.1.1.1        4 65501       8       8        5    0    0 00:02:22 0
10.1.1.2        4 65501       8       8        5    0    0 00:02:22 0
10.1.1.3        4 65501       8       8        5    0    0 00:02:22 0
```
Соседство установилось. Теперь настроим VxLAN (настройки делаем на LEAF, SPINE выступают в качестве ядра и просто занимаются маршрутизацией пакетов):<br>
```
!
! добавим vlan и привяжем его к vni
!
vlan 10
name VLAN10_TEST
 vn-segment 10010
!
interface nve1
 no shutdown
 host-reachability protocol bgp
 ! используем BGP для передачи маршрутов
 source-interface loopback1
 !
 ! IMPORTANT!!!! Потратил неделю, пока не понял, что
 ! нужно это делать с отключенным feature VPC,
 ! иначе loopback не поднимается
 member vni 10010
   ingress-replication protocol bgp
! Для передачи BUM испльзуем BGP
!
 ```
Проверим видимость пиров:
```LEAF-1# sh nve peers
Interface Peer-IP                                 State LearnType Uptime   Route
r-Mac
--------- --------------------------------------  ----- --------- -------- -----
------------
nve1      10.1.1.2                                Up    CP        14:17:19 n/a

nve1      10.1.1.3                                Up    CP        13:24:12 n/a

LEAF-3# sh bgp l2vpn evpn
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 9, Local Router ID is 10.1.1.3
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 10.1.1.1:32777
*>i[3]:[0]:[32]:[10.1.1.1]/88
                      10.1.1.1                          100          0 i
* i                   10.1.1.1                          100          0 i

Route Distinguisher: 10.1.1.2:32777
*>i[3]:[0]:[32]:[10.1.1.2]/88
                      10.1.1.2                          100          0 i
* i                   10.1.1.2                          100          0 i

Route Distinguisher: 10.1.1.3:32787    (L2VNI 10010)
*>i[3]:[0]:[32]:[10.1.1.1]/88
                      10.1.1.1                          100          0 i
*>i[3]:[0]:[32]:[10.1.1.2]/88
                      10.1.1.2                          100          0 i
*>l[3]:[0]:[32]:[10.1.1.3]/88
                      10.1.1.3                          100      32768 i

```
Видим, что появились Route-type 3 маршруты (для BUM трафика). Чтобы повились Route-type 2 маршруты - для хостов, нужно дополнительно настроить на LEAF, чтобы эти маршруты импортировались и экспорировались в таблицу маршрутизации, настройки делаются для каждого vni отдельно:<br>
```
evpn
 vni 10010 l2
  route-target import auto
  route-target export auto
```
Проверим, на LEAF-1 и LEAF-2 к интерфейсу ETH1/6 подключены хосты в VLAN10:<br>
```
LEAF-1# sh bgp l2vpn evpn
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 52, Local Router ID is 10.1.1.1
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 10.1.1.1:32777    (L2VNI 10010)
*>i[2]:[0]:[0]:[48]:[0050.7966.680b]:[0]:[0.0.0.0]/216
                      10.1.1.2                          100          0 i
*>l[2]:[0]:[0]:[48]:[0050.7966.680c]:[0]:[0.0.0.0]/216
                      10.1.1.1                          100      32768 i
*>l[3]:[0]:[32]:[10.1.1.1]/88
                      10.1.1.1                          100      32768 i
*>i[3]:[0]:[32]:[10.1.1.2]/88
                      10.1.1.2                          100          0 i
*>i[3]:[0]:[32]:[10.1.1.3]/88
                      10.1.1.3                          100          0 i

Route Distinguisher: 10.1.1.2:32777
* i[2]:[0]:[0]:[48]:[0050.7966.680b]:[0]:[0.0.0.0]/216
                      10.1.1.2                          100          0 i
*>i                   10.1.1.2                          100          0 i
* i[3]:[0]:[32]:[10.1.1.2]/88
                      10.1.1.2                          100          0 i
*>i                   10.1.1.2                          100          0 i

Route Distinguisher: 10.1.1.3:32787
* i[3]:[0]:[32]:[10.1.1.3]/88
                      10.1.1.3                          100          0 i
*>i                   10.1.1.3                          100          0 i
```
Появились Type-2 маршруты с MAC адресами хостов. Проверим досупность по сети:<br>
```

NAME   IP/MASK              GATEWAY                             GATEWAY
VPCS1  172.16.100.20/24     172.16.100.1
       fe80::250:79ff:fe66:680b/64

VPCS> ping 172.16.100.10

host (172.16.100.10) not reachable

VPCS> ping 172.16.100.10

84 bytes from 172.16.100.10 icmp_seq=1 ttl=64 time=6.186 ms
84 bytes from 172.16.100.10 icmp_seq=2 ttl=64 time=6.741 ms
84 bytes from 172.16.100.10 icmp_seq=3 ttl=64 time=7.172 ms
84 bytes from 172.16.100.10 icmp_seq=4 ttl=64 time=7.037 ms
84 bytes from 172.16.100.10 icmp_seq=5 ttl=64 time=6.012 ms
```
PING проходит.

Вывод: успешно получилось настроить и проверить.
Что не получилось: настроить arp-suppression, для этого требуется добавить:
```
hardware access-list tcam region arp-ether 256
```
И при вводе данной команды появляется сообщение об ошибке, пока нет понимания как это побороть.