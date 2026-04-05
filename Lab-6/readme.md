# Лабораторная работа 6 - L3 VNI

Изменим предыдущую лабораторную работу для проверки работы L3 VNI.<br>
Топология сети немного изменится, но только со стороны клиентских хостов. Подключим к LEAF-1 и LEAF-2 хосты виртуализации через интерфейсы Eth1/3, настроим на них транк и создадим 2 VLAN: Vlan 10 и Vlan 20. Схема сети с хосами представлена на иллюстрации ниже:<br>

![LAP-topology](https://github.com/dim-sem/OTUS-Homework/blob/main/Lab-6/lab6-topology.png "Topology")<br>
<br>
Все настройки будут производиться на LEAF и на хостах. На LEAF производим ледющие настройки:
```
!
vlan 1,10,20,999
vlan 10
  name VLAN10_TEST
  vn-segment 10010
vlan 20
  name VLAN20
  vn-segment 10020
vlan 999
  name L3_VNI
  vn-segment 1000999
!
```
Для L3 VNI добавим отдельный vlan 999 и SVI Vlan999 и привязем к vlan 999 VNI 1000999.<br>
Добавили VLAN 10 и VLAN 20 и привязали их к своим VNI. VNI 10010 был добавлен в evpn ранее, поэтому сразу же, чтобы не забыть, добавим VNI 10020 в evpn:
```
evpn
  vni 10020
   route-target import auto
   route-target export auto
```
Теперь нам нужно создать отдельный VRF и привязать к нему соответствующие SVI.<br>
```
!
! создали VRF
!
vrf context L3VNI_TEST
  vni 1000999
  rd auto
  address-family ipv4 unicast
    route-target both auto
    route-target both auto evpn

interface Vlan10
  no shutdown
  vrf member L3VNI_TEST
  ip address 172.16.10.254/24
  fabric forwarding mode anycast-gateway

interface Vlan20
  no shutdown
  vrf member L3VNI_TEST
  ip address 172.16.20.254/24
  fabric forwarding mode anycast-gateway

interface Vlan999
  description L3_VNI
  no shutdown
  vrf member L3VNI_TEST
  ip forward
! разрешаем маршрутизацию через underlay
```
На SVI VLAN10 и VLAN20 мы добавили специальную команду, чтобы включить возможност использовать anycast-gateway, но предварительно добавим на каждом LEAF виртуальный МАС адрес для anycast-gateway:<br>
```
fabric forwarding anycast-gateway-mac 0000.1000.9999
```
Этот MAC адрес должен быть одинаковый на всех LEAF, поскольку IP адрес SVI , которые будут служить шлюзами для подсетей на хостах мы указываем одинаковые на каждом LEAF, т.о. в ARP таблицах на хостах будут одинаковые данные.<br>
Отмечу, что снчала привяжем интерфейсы к VRF, т.к. на Cisco Nexus при этом удалаются все L3 настройки и IP адрес нужно будетзадавать заново.<br>
Пора настроить NVE интерфес:
```
interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback1
  member vni 10010
    ingress-replication protocol bgp
  member vni 10020
   ingress-replication protocol bgp
  member vni 1000999 associate-vrf
  ```
  Добавили VNI 100020 и VNI 1000999. Пора проверить, что у нас получилось.
  ```
  Пингуем хосты SERVER-1 и SERVER-2.

  SERVER-1# ping 172.16.20.11
PING 172.16.20.11 (172.16.20.11): 56 data bytes
64 bytes from 172.16.20.11: icmp_seq=0 ttl=254 time=11.748 ms
64 bytes from 172.16.20.11: icmp_seq=1 ttl=254 time=9.139 ms
64 bytes from 172.16.20.11: icmp_seq=2 ttl=254 time=10.008 ms
64 bytes from 172.16.20.11: icmp_seq=3 ttl=254 time=9.518 ms
64 bytes from 172.16.20.11: icmp_seq=4 ttl=254 time=10.092 ms

--- 172.16.20.11 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 9.139/10.1/11.748 ms

SERVER-2# ping 172.16.10.10
PING 172.16.10.10 (172.16.10.10): 56 data bytes
64 bytes from 172.16.10.10: icmp_seq=0 ttl=254 time=10.196 ms
64 bytes from 172.16.10.10: icmp_seq=1 ttl=254 time=12.3 ms
64 bytes from 172.16.10.10: icmp_seq=2 ttl=254 time=9.487 ms
64 bytes from 172.16.10.10: icmp_seq=3 ttl=254 time=11.045 ms
64 bytes from 172.16.10.10: icmp_seq=4 ttl=254 time=11.245 ms

--- 172.16.10.10 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 9.487/10.854/12.3 ms
SERVER-2#
SERVER-2# ping 172.16.20.10
PING 172.16.20.10 (172.16.20.10): 56 data bytes
64 bytes from 172.16.20.10: icmp_seq=0 ttl=254 time=10.288 ms
64 bytes from 172.16.20.10: icmp_seq=1 ttl=254 time=7.913 ms
64 bytes from 172.16.20.10: icmp_seq=2 ttl=254 time=10.055 ms
64 bytes from 172.16.20.10: icmp_seq=3 ttl=254 time=8.836 ms
64 bytes from 172.16.20.10: icmp_seq=4 ttl=254 time=8.283 ms

--- 172.16.20.10 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 7.913/9.075/10.288 ms
```
Связность есть! Что же у нас при этом на LEAF-1?
```
LEAF-1# sh bgp l2vpn evpn
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 31, Local Router ID is 10.1.1.1
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 10.1.1.1:32777    (L2VNI 10010)
*>l[2]:[0]:[0]:[48]:[5007.0000.1b08]:[0]:[0.0.0.0]/216
                      10.1.1.1                          100      32768 i
*>i[2]:[0]:[0]:[48]:[5008.0000.1b08]:[0]:[0.0.0.0]/216
                      10.1.1.2                          100          0 i
*>l[2]:[0]:[0]:[48]:[5007.0000.1b08]:[32]:[172.16.10.10]/272
                      10.1.1.1                          100      32768 i
*>i[2]:[0]:[0]:[48]:[5008.0000.1b08]:[32]:[172.16.10.11]/272
                      10.1.1.2                          100          0 i
*>l[3]:[0]:[32]:[10.1.1.1]/88
                      10.1.1.1                          100      32768 i
*>i[3]:[0]:[32]:[10.1.1.2]/88
                      10.1.1.2                          100          0 i
*>i[3]:[0]:[32]:[10.1.1.3]/88
                      10.1.1.3                          100          0 i
*>i[2]:[0]:[0]:[48]:[5008.0000.1b08]:[32]:[172.16.10.11]/272
                      10.1.1.2                          100          0 i
* i                   10.1.1.2                          100          0 i
*>i[3]:[0]:[32]:[10.1.1.2]/88
                      10.1.1.2                          100          0 i
* i                   10.1.1.2                          100          0 i

Route Distinguisher: 10.1.1.1:32787    (L2VNI 10020)
*>l[2]:[0]:[0]:[48]:[5007.0000.1b08]:[0]:[0.0.0.0]/216
                      10.1.1.1                          100      32768 i
*>i[2]:[0]:[0]:[48]:[5008.0000.1b08]:[0]:[0.0.0.0]/216
                      10.1.1.2                          100          0 i
*>l[2]:[0]:[0]:[48]:[5007.0000.1b08]:[32]:[172.16.20.10]/272
                      10.1.1.1                          100      32768 i
*>i[2]:[0]:[0]:[48]:[5008.0000.1b08]:[32]:[172.16.20.11]/272
                      10.1.1.2                          100          0 i
*>l[3]:[0]:[32]:[10.1.1.1]/88
                      10.1.1.1                          100      32768 i
*>i[3]:[0]:[32]:[10.1.1.2]/88
                      10.1.1.2                          100          0 i

Route Distinguisher: 10.1.1.2:32777
* i[2]:[0]:[0]:[48]:[5008.0000.1b08]:[0]:[0.0.0.0]/216
                      10.1.1.2                          100          0 i
*>i                   10.1.1.2                          100          0 i
*>i[2]:[0]:[0]:[48]:[5008.0000.1b08]:[32]:[172.16.10.11]/272
                      10.1.1.2                          100          0 i
* i                   10.1.1.2                          100          0 i
*>i[3]:[0]:[32]:[10.1.1.2]/88
                      10.1.1.2                          100          0 i
* i                   10.1.1.2                          100          0 i

Route Distinguisher: 10.1.1.2:32787
*>i[2]:[0]:[0]:[48]:[5008.0000.1b08]:[0]:[0.0.0.0]/216
                      10.1.1.2                          100          0 i
* i                   10.1.1.2                          100          0 i
* i[2]:[0]:[0]:[48]:[5008.0000.1b08]:[32]:[172.16.20.11]/272
                      10.1.1.2                          100          0 i
*>i                   10.1.1.2                          100          0 i
*>i[3]:[0]:[32]:[10.1.1.2]/88
                      10.1.1.2                          100          0 i
* i                   10.1.1.2                          100          0 i

Route Distinguisher: 10.1.1.3:32787
*>i[3]:[0]:[32]:[10.1.1.3]/88
                      10.1.1.3                          100          0 i
* i                   10.1.1.3                          100          0 i

Route Distinguisher: 10.1.1.1:3    (L3VNI 1000999)
*>i[2]:[0]:[0]:[48]:[5008.0000.1b08]:[32]:[172.16.10.11]/272
                      10.1.1.2                          100          0 i
*>i[2]:[0]:[0]:[48]:[5008.0000.1b08]:[32]:[172.16.20.11]/272
                      10.1.1.2                          100          0 i

LEAF-1#
```
В BGP таблице видим по 2 Type-2 маршрута, один только с MAC, а вот второй - MAC-IP, т.е. фактически - ARP-запись.
Вывод - работает!<br>
P.S. Не получилось проверить ARP supression, требует ввести команду hardware access-list tcam region arp-ether size double-wide, но она выдает ошибку Пока нет понимания, как это побороть.
