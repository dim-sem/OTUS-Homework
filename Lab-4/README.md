# Лабораторная работа 2 - Underlay BGP

В данной работе настроим underlay сеть для работы по протоколу маршрутизации BGP.<br>
Используем сеть, собранную в первой лабораторной работе как основу для дальнейшей модернизации.<br>
![LAP-topology](https://github.com/dim-sem/OTUS-Homework/blob/main/Lab-4/clos-lab1.png "Topology")<br>
На Spine и Leaf объявим использование протокола BGP, предварительно активировав данный функционал, что необходимо сделать, т.к. сеть собрана на Cisco Nexus:
```
feature BGP
```
У нас есть два варианта для организации опорной сети: iBGP или eBGP.<br>
Попробуем оба варианта:<br>
# iBGP<br>
В случае iBGP топология между нодами будет представлять full-mesh, в силу того, что  маршрутизатор не может передавать маршрут полученный от одного соседа другим соседям. Впрочем это ограничение мы можем обойти с помощью Route-reflector, т.е. выделенных маршрутизаторов, с которыми будут организовывать соседство другие маршрутизаторы в сети. В CLOS-сети в качестве Route-reflector будут выступать SPINE, а LEAF будут в роли клиентов.<br>
![LAP-topology](https://github.com/dim-sem/OTUS-Homework/blob/main/Lab-4/lab-4.png "Topology")<br>
На Spine добавим следуюшие настройки (на примере SPINE-1):<br>
```
router bgp 65500
router-id 10.0.1.0
!
! router-id = Loopback1 IP address
!
timers bgp 3 9
reconnect-interval 10
!
! tuning default bgp timers
!
log-neighbor-changes
!
neighbor 10.2.1.0/24
  bfd
  remote-as 65500
  timers 3 9
  maximum-paths 10
  address-family ipv4 unicast
  route-reflector-client
  next-hop-self all
```
И на LEAF настроим следующее (на примере LEAF-1):<br>
```

!
route-mac CONNECTED permit 10
  match interface Loopback 1
!
!   для анонсирования Loopback
!
router bgp 65500
  router=id 10.1.1.1
  timers bgp 3 9
  reconnect-interval 10
  log-neighbor-changes
  address-family ipv4 unicast
  redistribute direct route-map CONNECTED
  maximum-paths ibgp 10
!
! анонсируем Loopback 1
!
template peer UNDERLAY
  remote-as 65500
  timers 3 9
  address-family ipv4 unicast
neighbor 10.2.1.0
  inherit peer UNDERLAY
  bfd
  remote-as 65500
neighbor 10.2.2.0
  inherit peer UNDERLAY
  bfd
  remote-as 65500
```
Проверим, что у нас получилось:<br>
```
На SPINE-1 -
SPINE-1# sh ip bgp sum
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 10.0.1.0, local AS number 65500
BGP table version is 10, IPv4 Unicast config peers 4, capable peers 3
4 network entries and 4 paths using 976 bytes of memory
BGP attribute entries [2/344], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [0/0]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.2.1.1        4 65500    2886    2882       10    0    0 02:24:12 2
10.2.1.3        4 65500    2671    2669       10    0    0 02:13:29 1
10.2.1.5        4 65500    2610    2609       10    0    0 02:10:27 1

Установлено соседство со всеми LEAF-ами.

Посмотрим, что у нас на LEAF-1:

LEAF-1# sh ip bgp sum
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 10.1.1.1, local AS number 65500
BGP table version is 13, IPv4 Unicast config peers 2, capable peers 2
4 network entries and 6 paths using 1216 bytes of memory
BGP attribute entries [6/1032], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [4/16]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.2.1.0        4 65500   21827   21825       13    0    0 18:13:04 2
10.2.2.0        4 65500   21733   21752       13    0    0 18:08:22 2

LEAF-1# sh bgp sessions
Total peers 2, established peers 2
ASN 65500
VRF default, local ASN 65500
peers 2, established peers 2, local router-id 10.1.1.1
State: I-Idle, A-Active, O-Open, E-Established, C-Closing, S-Shutdown

Neighbor        ASN    Flaps LastUpDn|LastRead|LastWrit St Port(L/R)  Notif(S/R)
10.2.1.0        65500 0     18:11:20|0.196921|0.649180 E   53277/179        0/0
10.2.2.0        65500 0     18:06:38|0.747531|0.649474 E   39230/179        0/0

Сессии в состоянии Established.
Смотрим таблицу маршрутизации на LEAF-1:

LEAF-1# sh ip route bgp
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

10.1.1.2/32, ubest/mbest: 2/0
    *via 10.2.1.0, [200/0], 02:37:08, bgp-65500, internal, tag 65500
    *via 10.2.2.0, [200/0], 02:36:43, bgp-65500, internal, tag 65500
10.1.1.3/32, ubest/mbest: 2/0
    *via 10.2.1.0, [200/0], 02:34:05, bgp-65500, internal, tag 65500
    *via 10.2.2.0, [200/0], 02:33:52, bgp-65500, internal, tag 65500

Видим, что Loopback интерейсы LEAF-2 и LEAF-3 должны быть доступны. 
Проверим:
LEAF-1# ping 10.1.1.3
PING 10.1.1.3 (10.1.1.3): 56 data bytes
2026 Mar 24 22:52:18.904234 netstack: [30222] (default) Send packet (mbuf_prty 0): s=10.2.2.1, d=10.1.1.3, proto=1 (icmp
), ICMP_ECHO, tos/dscp=0x0/0x0, ip_len=84, id=0000, ttl=255
Request 0 timed out
2026 Mar 24 22:52:20.907128 netstack: [30222] (default) Send packet (mbuf_prty 0): s=10.2.2.1, d=10.1.1.3, proto=1 (icmp
), ICMP_ECHO, tos/dscp=0x0/0x0, ip_len=84, id=0000, ttl=255
Request 1 timed out

Не работает. Т.к. мы анонсируем только Lo1, а в данном случае SRC interface для ping - eth1/1 (IP 10.2.2.1), о котором LEAF-3 ничего не знает.
А вот так будет работать:
LEAF-1# ping 10.1.1.3 source 10.1.1.1
PING 10.1.1.3 (10.1.1.3) from 10.1.1.1: 56 data bytes
64 bytes from 10.1.1.3: icmp_seq=0 ttl=253 time=6.908 ms
64 bytes from 10.1.1.3: icmp_seq=1 ttl=253 time=4.696 ms
64 bytes from 10.1.1.3: icmp_seq=2 ttl=253 time=5.183 ms
64 bytes from 10.1.1.3: icmp_seq=3 ttl=253 time=4.222 ms
64 bytes from 10.1.1.3: icmp_seq=4 ttl=253 time=3.545 ms
2026 Mar 24 22:53:31.546330 netstack: [30222] (default) Send packet (mbuf_prty 0): s=10.1.1.1, d=10.1.1.3, proto=1 (icmp
), ICMP_ECHO, tos/dscp=0x0/0x0, ip_len=84, id=0000, ttl=255

И если мы явно проанонсируем сеть 172.16.1.0/24 на LEAF-1:

router bgp 65500
  address-family ipv4 unicast
   network 172.16.1.0/24

То эта сеть будет доступна, например, с LEAF-3:
LEAF-3# ping 172.16.1.2 source 10.1.1.3
PING 172.16.1.2 (172.16.1.2) from 10.1.1.3: 56 data bytes
64 bytes from 172.16.1.2: icmp_seq=0 ttl=61 time=6.688 ms
64 bytes from 172.16.1.2: icmp_seq=1 ttl=61 time=5.84 ms
64 bytes from 172.16.1.2: icmp_seq=2 ttl=61 time=6.263 ms
64 bytes from 172.16.1.2: icmp_seq=3 ttl=61 time=5.916 ms
64 bytes from 172.16.1.2: icmp_seq=4 ttl=61 time=8.004 ms
````
Вывод: всё работает, но помним, что
1. сети нужно анонсировать, если хотим, чтобы они были доступны.
2. поскольку увеличение количества SPINE и LEAF приводит к почти квадратичному увеличению соседств, нужно использовать Route Reflector, в качестве которых будут выступать SPINE.

Переделаем нашу топологию на использование eBGP. В данном случае у каждого LEAF будет своя AS, а вот все SPINE в POD будут находитьс в общей для них AS.
![LAP-topology](https://github.com/dim-sem/OTUS-Homework/blob/main/Lab-4/lab-4-ebgp.png "Topology")<br>
На SPINE настроим т.о. (на примере SPINE-1):
````
route-map RM_LEAVES_BGP permit 10
  match as-number 65551-65590
!
router bgp 65500
  router-id 10.0.1.0
  reconnect-interval 10
  log-neighbor-changes 
  address-family ipv4 unicast
    maximum-paths 10
 neighbor 10.2.1.0/24 remote-as route-map RM_LEAVES_BGP
   bfd
   address-family ipv4 unicast
!
````
На LEAF-s (на примере LEAF-1):
```
router bgp 65551
  router-id 10.1.1.1
  reconnect-interval 10
  log-neighbor-changes
  address-family ipv4 unicast
    redistribute direct route-map CONNECTED
    maximum-paths 10
  template peer SPINES
    bfd
    remote-as 65500
    timers 3 9
    address-family ipv4 unicast
  neighbor 10.2.1.0
    inherit peer SPINES
  neighbor 10.2.2.0
    inherit peer SPINES
```
Проверим работоспособность с LEAF-1:
```
LEAF-1# sh bgp sess
Total peers 2, established peers 2
ASN 65551
VRF default, local ASN 65551
peers 2, established peers 2, local router-id 10.1.1.1
State: I-Idle, A-Active, O-Open, E-Established, C-Closing, S-Shutdown

Neighbor        ASN    Flaps LastUpDn|LastRead|LastWrit St Port(L/R)  Notif(S/R)
10.2.1.0        65500 0     00:03:46|0.326308|0.386244 E   46442/179        0/0
10.2.2.0        65500 0     00:02:14|00:00:01|00:00:01 E   49435/179        0/52

LEAF-1# sh ip rou bgp
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

10.1.1.2/32, ubest/mbest: 2/0
    *via 10.2.1.0, [20/0], 00:04:44, bgp-65551, external, tag 65500
    *via 10.2.2.0, [20/0], 00:03:12, bgp-65551, external, tag 65500
10.1.1.3/32, ubest/mbest: 2/0
    *via 10.2.1.0, [20/0], 00:04:45, bgp-65551, external, tag 65500
    *via 10.2.2.0, [20/0], 00:03:12, bgp-65551, external, tag 65500

LEAF-1# sh ip bgp summ
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 10.1.1.1, local AS number 65551
BGP table version is 9, IPv4 Unicast config peers 2, capable peers 2
3 network entries and 5 paths using 972 bytes of memory
BGP attribute entries [3/516], BGP AS path entries [2/20]
BGP community entries [0/0], BGP clusterlist entries [0/0]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.2.1.0        4 65500     109     155        9    0    0 00:05:05 2
10.2.2.0        4 65500     182     180        9    0    0 00:03:33 2

LEAF-1# ping 10.1.1.3 source 10.1.1.1
PING 10.1.1.3 (10.1.1.3) from 10.1.1.1: 56 data bytes
64 bytes from 10.1.1.3: icmp_seq=0 ttl=253 time=5.548 ms
64 bytes from 10.1.1.3: icmp_seq=1 ttl=253 time=5.734 ms
64 bytes from 10.1.1.3: icmp_seq=2 ttl=253 time=5.328 ms
64 bytes from 10.1.1.3: icmp_seq=3 ttl=253 time=5.92 ms
64 bytes from 10.1.1.3: icmp_seq=4 ttl=253 time=5.014 ms

--- 10.1.1.3 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 5.014/5.508/5.92 ms
```

Вывод: всё работает, анонсоруемые сети доступны. Настройка проще, чем iBGP, но нужно продумать возможность масштабирования.