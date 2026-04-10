# Лабораторная работа 8 - VxLAN routing

Рассмотрим EVPN Route-Type 5 маршруты.<br>
Такие маршуты используются для передачи в VxLAN маршрутной информации и сетях и в том числе с их помощью можно передавать маршрут по умолчанию, что мы и попробуем сделать.<br>
Для этого немного переделаем топологии, использованую в прошлой лабораторной работе, в частности добавим внешнее устроство - FW (Firewall), через который VxLAN фабрика общается с внешним миром. Кроме того поместим VLAN 10 и VLAN 20 в разные VRF, так чтобы взаимодействие между ними проходило через FW.<br>
Топология представлена на следующей схеме:
![LAP-topology](https://github.com/dim-sem/OTUS-Homework/blob/main/Lab-8/topology.png "Topology")<br>
<br>
На SERVER-1 создадим отдельные VRF VLAN 10 и VLAN 20 и привяжем VLAN к соответсвующим VRF, аналогичные настройки сделаем на SERVER-2 и SERVER-3. Отличия будут только в IP адресах на соответсвующих интерфейсах на хостах.<br>
```
vrf context VLAN_10
 ip route 0.0.0.0/0 172.16.10.254
vrf context VLAN_20
 ip route 0.0.0.0/0 172.16.20.254
!
! Пропишем дефолтный маршрут в VRF на серверах.
!

interface Vlan 10
 no shutdown
 vrf member VLAN_10
 ip address 172.16.10.10/24
interface Vlan 20
 no shutdown
 vrf member VLAN_20
 ip address 172.16.20.10/24
```
Теперь перейдём к настройкам на LEAF-ах. Поскольку мы помещаем каждый VLAN в отдельный VRF нужно будет создать свой vni для каждого из них. Настройки покажем на примере LEAF-1.<br>
```
!
! создали отдельные Vlan и привязали к каждому свой vni
!

vlan 888
 name OTUS-VRF1
 vn-segment 1000888
vlan 999
 name OTUS-VRF2
 vn-segment 1000999

!
! Создали отдельные VRF
!

vrf context OTUS-VRF1
 vni 1000888
 rd auto
 address-fammily ipv4 unicast
  route-target both auto
  route-target both auto evpn
vrf context OTUS-VRF2
 vni 1000999
 rd auto
 address-fammily ipv4 unicast
  route-target both auto
  route-target both auto evpn

!
! назначили VRF соответсвующим SVI
!

interface Vlan10
 no shutdown
 vrf member OTUS_VRF1
 no ip redirects
 ip address 172.16.10.254/24
 no ipv6 rediects
 fabric forwarding mode anycast-gateway

interface Vlan20
 no shutdown
 vrf member OTUS_VRF2
 no ip redirects
 ip address 172.16.20.254/24
 no ipv6 rediects
 fabric forwarding mode anycast-gateway

!
! Для L3 VNI тоже добавим SVI
!

interface Vlan888
 description OTUS-VRF1 VNI
 no shutdown
 vrf member OTUS-VRF1
 no ip redirects
 ip forward
 no ipv6 redirects

interface Vlan999
 description OTUS-VRF2 VNI
 no shutdown
 vrf member OTUS-VRF2
 no ip redirects
 ip forward
 no ipv6 redirects

!
!  на NVE интерфейс дабавим нужные L3 VNI 
!

interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback1
  member vni 10010
    ingress-replication protocol bgp
  member vni 10020
  member vni 1000888 associate-vrf
  member vni 1000999 associate-vrf
```
Аналогично настроим на остальных LEAF.<br>
На LEAF-3 и LEAF-4 нам потребуются дополнительный настройки, поскольку они выступают в качестве boarder leaf. Добавим на них дополнительные SVI для транзита трафика на FW и привяжем их к VRF.
```
vlan 1110
 name Transit_to_FW_Vlan10 
vlan 1120
 name Transit_to_FW_Vlan20

interface Vlan1110
  no shurdown
  vrf member OTUS-VRF1
  no ip redirects
  ip address 172.16.108.1/30
  no ipv6 redirects

interface Vlan1120
  no shurdown
  vrf member OTUS-VRF2
  no ip redirects
  ip address 172.16.109.1/30
  no ipv6 redirects

!
! анонсируем default route в BGP в соответсвующих VRF
!

router BGP 65501
vrf OTUS-VRF1
  address-family ipv4 unicast
    network 0.0.0.0/0
vrf OTUS-VRF2
  address-family ipv4 unicast
    network 0.0.0.0/0
```
Проверим, что же получилось и получилось ли, что-нибудь.<br>
Как нам известно, при анонсе сетей создаются маршруты типа-5, и действительно, смотрим на LEAF-1:
```
LEAF-1# sh bgp l2vpn evpn route-type 5 vrf OTUS-VRF2
Route Distinguisher: 10.1.1.1:4    (L3VNI 1000999)
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 32
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Path type: internal, path is valid, not best reason: Router Id, no labeled nex
thop
             Imported from 10.1.1.4:4:[5]:[0]:[0]:[0]:[0.0.0.0]/224
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    10.1.1.102 (metric 81) from 10.0.1.0 (10.0.1.0)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 1000999
      Extcommunity: RT:65501:1000999 ENCAP:8 Router MAC:5009.0000.1b08
      Originator: 10.1.1.4 Cluster list: 10.0.1.0

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 10.1.1.3:4:[5]:[0]:[0]:[0]:[0.0.0.0]/224
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    10.1.1.102 (metric 81) from 10.0.1.0 (10.0.1.0)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 1000999
      Extcommunity: RT:65501:1000999 ENCAP:8 Router MAC:5006.0000.1b08
      Originator: 10.1.1.3 Cluster list: 10.0.1.0

  Path-id 1 not advertised to any peer

LEAF-1# sh bgp l2vpn evpn route-type 5 vrf OTUS-VRF1
Route Distinguisher: 10.1.1.1:3    (L3VNI 1000888)
BGP routing table entry for [5]:[0]:[0]:[0]:[0.0.0.0]/224, version 59
Paths: (2 available, best #2)
Flags: (0x000002) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Path type: internal, path is valid, not best reason: Router Id, no labeled nex
thop
             Imported from 10.1.1.4:3:[5]:[0]:[0]:[0]:[0.0.0.0]/224
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    10.1.1.102 (metric 81) from 10.0.1.0 (10.0.1.0)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 1000888
      Extcommunity: RT:65501:1000888 ENCAP:8 Router MAC:5009.0000.1b08
      Originator: 10.1.1.4 Cluster list: 10.0.1.0

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 10.1.1.3:3:[5]:[0]:[0]:[0]:[0.0.0.0]/224
  Gateway IP: 0.0.0.0
  AS-Path: NONE, path sourced internal to AS
    10.1.1.102 (metric 81) from 10.0.1.0 (10.0.1.0)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 1000888
      Extcommunity: RT:65501:1000888 ENCAP:8 Router MAC:5006.0000.1b08
      Originator: 10.1.1.3 Cluster list: 10.0.1.0

  Path-id 1 not advertised to any peer
```
Как видим появились маршруты к сети 0.0.0.0/0 (т.е. Default route) и данные анонсы прилетели с LEAF-3 и LEAF-4 в каждом, созданном нами VRF:
```
      Originator: 10.1.1.4 Cluster list: 10.0.1.0

      Originator: 10.1.1.3 Cluster list: 10.0.1.0      
```
Проверим доступность по сети с сервера SERVER-1:
```
SERVER-1# ping 8.8.8.8 vrf VLAN_10
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: icmp_seq=0 ttl=252 time=12.434 ms
64 bytes from 8.8.8.8: icmp_seq=1 ttl=252 time=13.268 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=252 time=12.974 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=252 time=9.885 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=252 time=9.82 ms

--- 8.8.8.8 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 9.82/11.676/13.268 ms
SERVER-1#
SERVER-1# ping 8.8.8.8 vrf VLAN_20
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: icmp_seq=0 ttl=252 time=14.909 ms
64 bytes from 8.8.8.8: icmp_seq=1 ttl=252 time=11.037 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=252 time=13.135 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=252 time=16.78 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=252 time=11.298 ms

--- 8.8.8.8 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 11.037/13.431/16.78 ms
SERVER-1#
```
"Внешняя" сеть - доступна!<br>
Но как у нас с доступностью хостов в VRF и между VRF?
```
SERVER-1# ping 172.16.10.100 vrf VLAN_10
PING 172.16.10.100 (172.16.10.100): 56 data bytes
64 bytes from 172.16.10.100: icmp_seq=0 ttl=254 time=24.305 ms
64 bytes from 172.16.10.100: icmp_seq=1 ttl=254 time=15.699 ms
64 bytes from 172.16.10.100: icmp_seq=2 ttl=254 time=16.403 ms
64 bytes from 172.16.10.100: icmp_seq=3 ttl=254 time=14.667 ms
64 bytes from 172.16.10.100: icmp_seq=4 ttl=254 time=12.922 ms

--- 172.16.10.100 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 12.922/16.799/24.305 ms
SERVER-1#
SERVER-1# ping 172.16.20.100 vrf VLAN_20
PING 172.16.20.100 (172.16.20.100): 56 data bytes
36 bytes from 172.16.20.10: Destination Host Unreachable
Request 0 timed out
64 bytes from 172.16.20.100: icmp_seq=1 ttl=254 time=10.953 ms
64 bytes from 172.16.20.100: icmp_seq=2 ttl=254 time=14.736 ms
64 bytes from 172.16.20.100: icmp_seq=3 ttl=254 time=12.584 ms
64 bytes from 172.16.20.100: icmp_seq=4 ttl=254 time=12.921 ms

--- 172.16.20.100 ping statistics ---
5 packets transmitted, 4 packets received, 20.00% packet loss
round-trip min/avg/max = 10.953/12.798/14.736 ms
SERVER-1# 
SERVER-1# ping 172.16.20.100 vrf VLAN_10
PING 172.16.20.100 (172.16.20.100): 56 data bytes
36 bytes from 172.16.20.10: Destination Host Unreachable
Request 0 timed out
64 bytes from 172.16.20.100: icmp_seq=1 ttl=254 time=10.953 ms
64 bytes from 172.16.20.100: icmp_seq=2 ttl=254 time=14.736 ms
64 bytes from 172.16.20.100: icmp_seq=3 ttl=254 time=12.584 ms
64 bytes from 172.16.20.100: icmp_seq=4 ttl=254 time=12.921 ms

--- 172.16.20.100 ping statistics ---
5 packets transmitted, 4 packets received, 20.00% packet loss
round-trip min/avg/max = 10.953/12.798/14.736 ms
SERVER-1# 
```
Тоже всё ОК!

Вывод: успешно построили сеть и проверили работу Route-type 5 в VxLAN.