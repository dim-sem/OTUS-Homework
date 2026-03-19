# Лабораторная работа 2 - Underlay ISIS

В данной работе настроим underlay сеть для работы по протоколу маршрутизации ISIS.<br>
Используем сеть, собранную в первой лабораторной работе как основу для дальнейшей модернизации.<br>
![LAP-topology](https://github.com/dim-sem/OTUS-Homework/blob/main/Lab-3/clos-lab1.png "Topology")<br>
На Spine и Leaf объявим использование протокола IS-IS, предварительно активировав данный функционал, что необходимо сделать, т.к. сеть собрана на Cisco Nexus:
```
feature isis
!
router isis UNDERLAY
net 49.0001.0100.0000.1000.00
address-family ipv4 unicast
!
```
Обратим внимание на значение после net - это CLNS (Connectionless Network Service) адрес, т.е., фактически это идентификатор ноды (Intermediate System), который формируется сдедующим образом:
49 - AFI (Authority and format identifier), в даннном случае 49 - приватный диапазон.<br>
.0001. - IDI (Initial Domain Identifier) - area 1<br>
Вместе AFI+IDI формируют IDP (Initial domain part) - фактически это ID area.<br>
0100.000.1000 - System ID, для простоты документирования и дальнейшей поддержки сформируем его из IPv4 адреса интерфейса Lo1 дополнив октеты нулями, в примере используем адрес Lo1 SPINE-1:<br>
10.0.1.0 -> 010000001000 -> 0100.0000.1000<br>
Для других нод сформируем по тому же принципу.<br>
.00 - Network Entity Title, значение 00 - признак того, что адрес относится к данной IS.

Учтем, что идентификатор в net должен быть уникален для каждой ноды.<br>
Включим IS-IS на интерфейсах, для Cisco Nexux это команда:
```
Interface Eth1/1
ip router isis UNDERLAY
isis network point-to-point
```
где UNDERLAY это тег процесса IS-IS, запущенного на ноде.<br>
И укажем тип сети point-to-point.<br>
Просле того,как процесс IS-IS запущен ноды устанавливают соседство и начинают обмениваться маршрутной информацией.<br>
Проверим работоспособность протокола:<br>
```
show isis database
show isis adjacency
show ip route isis
show isis protocol
show isis topology
```
Пример вывода с LEAF-2:<br>
```
LEAF-2# sh isis data
IS-IS Process: UNDERLAY LSP database VRF: default
IS-IS Level-1 Link State Database
  LSPID                 Seq Number   Checksum  Lifetime   A/P/O/T
  SPINE-1.00-00         0x00000007   0xD02B    719        0/0/0/3
  SPINE-2.00-00         0x0000001E   0x4384    740        0/0/0/3
  LEAF-1.00-00          0x00000011   0xAC92    949        0/0/0/3
  LEAF-2.00-00        * 0x00000010   0xA382    1077       0/0/0/3
  LEAF-3.00-00          0x0000000F   0x91B9    1001       0/0/0/3

IS-IS Level-2 Link State Database
  LSPID                 Seq Number   Checksum  Lifetime   A/P/O/T
  SPINE-1.00-00         0x00000027   0x904B    705        0/0/0/3
  SPINE-2.00-00         0x0000001B   0x4981    774        0/0/0/3
  LEAF-1.00-00          0x0000000F   0xB090    949        0/0/0/3
  LEAF-2.00-00        * 0x0000000E   0xA780    1077       0/0/0/3
  LEAF-3.00-00          0x00000027   0x61D1    1001       0/0/0/3

LEAF-2# sh isis adj
IS-IS process: UNDERLAY VRF: default
IS-IS adjacency database:
Legend: '!': No AF level connectivity in given topology
System ID       SNPA            Level  State  Hold Time  Interface
SPINE-1         N/A             1-2    UP     00:00:24   Ethernet1/1
SPINE-2         N/A             1-2    UP     00:00:29   Ethernet1/2

LEAF-2# sh ip rou isis
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

10.0.1.0/32, ubest/mbest: 1/0
    *via 10.2.1.2, Eth1/1, [115/41], 00:39:30, isis-UNDERLAY, L1
10.0.2.0/32, ubest/mbest: 1/0
    *via 10.2.2.2, Eth1/2, [115/41], 01:00:33, isis-UNDERLAY, L1
10.1.1.1/32, ubest/mbest: 2/0
    *via 10.2.1.2, Eth1/1, [115/81], 00:39:29, isis-UNDERLAY, L1
    *via 10.2.2.2, Eth1/2, [115/81], 01:08:56, isis-UNDERLAY, L1
10.2.1.0/31, ubest/mbest: 1/0
    *via 10.2.1.2, Eth1/1, [115/80], 00:39:30, isis-UNDERLAY, L1
10.2.1.4/31, ubest/mbest: 1/0
    *via 10.2.1.2, Eth1/1, [115/80], 00:39:30, isis-UNDERLAY, L1
10.2.2.0/31, ubest/mbest: 1/0
    *via 10.2.2.2, Eth1/2, [115/80], 01:08:57, isis-UNDERLAY, L1
10.2.2.4/31, ubest/mbest: 1/0
    *via 10.2.2.2, Eth1/2, [115/80], 00:23:03, isis-UNDERLAY, L1
172.16.1.0/24, ubest/mbest: 2/0
    *via 10.2.1.2, Eth1/1, [115/120], 00:08:10, isis-UNDERLAY, L1
    *via 10.2.2.2, Eth1/2, [115/120], 00:08:10, isis-UNDERLAY, L1
172.16.4.0/24, ubest/mbest: 2/0
    *via 10.2.1.2, Eth1/1, [115/120], 00:07:18, isis-UNDERLAY, L1
    *via 10.2.2.2, Eth1/2, [115/120], 00:07:18, isis-UNDERLAY, L1
172.16.5.0/24, ubest/mbest: 2/0
    *via 10.2.1.2, Eth1/1, [115/120], 00:07:18, isis-UNDERLAY, L1
    *via 10.2.2.2, Eth1/2, [115/120], 00:07:18, isis-UNDERLAY, L1

LEAF-2# sh isis top
IS-IS process: UNDERLAY
VRF: default
Topology ID: 0

IS-IS Level-1 IS routing table
SPINE-1.00, Instance 0x0000001D
   *via SPINE-1, Ethernet1/1, metric 40
SPINE-2.00, Instance 0x0000001D
   *via SPINE-2, Ethernet1/2, metric 40
LEAF-1.00, Instance 0x0000001D
   *via SPINE-1, Ethernet1/1, metric 80
   *via SPINE-2, Ethernet1/2, metric 80
LEAF-3.00, Instance 0x0000001D
   *via SPINE-1, Ethernet1/1, metric 80
   *via SPINE-2, Ethernet1/2, metric 80

IS-IS Level-2 IS routing table
SPINE-1.00, Instance 0x00000025
   *via SPINE-1, Ethernet1/1, metric 40
SPINE-2.00, Instance 0x00000025
   *via SPINE-2, Ethernet1/2, metric 40
LEAF-1.00, Instance 0x00000025
   *via SPINE-1, Ethernet1/1, metric 80
   *via SPINE-2, Ethernet1/2, metric 80
LEAF-3.00, Instance 0x00000025
   *via SPINE-1, Ethernet1/1, metric 80
   *via SPINE-2, Ethernet1/2, metric 80
```
Проверим доступность конечных сетей с помощью ping с LEAF-2:<br>
```
LEAF-2# ping 172.16.5.2
PING 172.16.5.2 (172.16.5.2): 56 data bytes
Request 0 timed out
64 bytes from 172.16.5.2: icmp_seq=1 ttl=61 time=22.29 ms
64 bytes from 172.16.5.2: icmp_seq=2 ttl=61 time=9.893 ms
64 bytes from 172.16.5.2: icmp_seq=3 ttl=61 time=6.509 ms
64 bytes from 172.16.5.2: icmp_seq=4 ttl=61 time=9.108 ms

--- 172.16.5.2 ping statistics ---
5 packets transmitted, 4 packets received, 20.00% packet loss
round-trip min/avg/max = 6.509/11.949/22.29 ms
LEAF-2#
LEAF-2# ping 172.16.1.2
PING 172.16.1.2 (172.16.1.2): 56 data bytes
64 bytes from 172.16.1.2: icmp_seq=0 ttl=61 time=18.314 ms
64 bytes from 172.16.1.2: icmp_seq=1 ttl=61 time=6.601 ms
64 bytes from 172.16.1.2: icmp_seq=2 ttl=61 time=7.6 ms
64 bytes from 172.16.1.2: icmp_seq=3 ttl=61 time=7.478 ms
64 bytes from 172.16.1.2: icmp_seq=4 ttl=61 time=6.046 ms

--- 172.16.1.2 ping statistics ---
5 packets transmitted, 5 packets received, 0.00% packet loss
round-trip min/avg/max = 6.046/9.207/18.314 ms
```
Проверим доступность VPC1 и LEAF-1 с VPC4:<br>
```
VPC4> ping 172.16.1.2

84 bytes from 172.16.1.2 icmp_seq=1 ttl=61 time=6.462 ms
84 bytes from 172.16.1.2 icmp_seq=2 ttl=61 time=5.797 ms
84 bytes from 172.16.1.2 icmp_seq=3 ttl=61 time=6.343 ms
84 bytes from 172.16.1.2 icmp_seq=4 ttl=61 time=6.322 ms
84 bytes from 172.16.1.2 icmp_seq=5 ttl=61 time=8.368 ms

VPC4> ping 172.16.1.1

84 bytes from 172.16.1.1 icmp_seq=1 ttl=253 time=5.791 ms
84 bytes from 172.16.1.1 icmp_seq=2 ttl=253 time=5.721 ms
84 bytes from 172.16.1.1 icmp_seq=3 ttl=253 time=5.860 ms
84 bytes from 172.16.1.1 icmp_seq=4 ttl=253 time=5.621 ms
84 bytes from 172.16.1.1 icmp_seq=5 ttl=253 time=5.830 ms

VPC4>
```
Вывод: Underlay сеть на IS-IS построена, маршрутизация работает. Конфиги устройств приложены в файле lab-3.zip.

UPD: Учтем пожелания руководителя курса и изменим конфигурацию: будем использовать только level-2 и добавим аутентификацию.
```
key chain ISIS
  key 100
    key-string 7 070632455d360a0014000e18
    cryptographic-algorithm MD5
```
Добавим на всех нодах key chain.<br>
На интерфейсах, обменивающихся пакетами IS-IS и для процесса IS-IS добавим настройки аутентификации:<br>
```
!
interface Eth1/1
  isis authentication-type md5
  isis authentication key-chain ISIS
!
router isis UNDERLAY
! ---- use level-2 only ------
is-type level-2
! --- authentication  ----
auuthentication-type md5
authentication key-chain ISIS level-2
!
```
Что измениось? <br>
```
SPINE-2# sh isis data
IS-IS Process: UNDERLAY LSP database VRF: default
IS-IS Level-1 Link State Database
  LSPID                 Seq Number   Checksum  Lifetime   A/P/O/T

IS-IS Level-2 Link State Database
  LSPID                 Seq Number   Checksum  Lifetime   A/P/O/T
  SPINE-1.00-00         0x0000003C   0x75B6    1128       0/0/0/3
  SPINE-2.00-00       * 0x00000036   0x8ABF    1100       0/0/0/3
  LEAF-1.00-00          0x0000003E   0x28C5    897        0/0/0/3
  LEAF-2.00-00          0x00000031   0xA53B    939        0/0/0/3
  LEAF-3.00-00          0x0000002E   0x1E4F    1102       0/0/0/3
```
Как видим в LSDB присутствуют только level-2 PDU, как мы и настраивали в протоколе маршрутизации на нодах.
```
SPINE-2#

SPINE-2#sh isis prot
<---skipped--->
  Topology : 2
  Address family IPv4 unicast :
    Number of interface : 0
    Distance : 115
    Default-information not originated
  Address family IPv6 unicast :
    Number of interface : 0
    Distance : 115
    Default-information not originated
  Level1
  No auth type
  Auth check set
  Level2
  Auth type:MD5
Auth keychain is ISIS
  Auth check set
<---skipped--->

SPINE-1# sh isis top
IS-IS process: UNDERLAY
VRF: default
Topology ID: 0

IS-IS Level-1 IS routing table

IS-IS Level-2 IS routing table
SPINE-2.00, Instance 0x00000023
   *via LEAF-1, Ethernet1/1, metric 80
   *via LEAF-2, Ethernet1/2, metric 80
   *via LEAF-3, Ethernet1/3, metric 80
LEAF-1.00, Instance 0x00000023
   *via LEAF-1, Ethernet1/1, metric 40
LEAF-2.00, Instance 0x00000023
   *via LEAF-2, Ethernet1/2, metric 40
LEAF-3.00, Instance 0x00000023
   *via LEAF-3, Ethernet1/3, metric 40
```
Ноды аутентифицируются друг с другом и при несовпадении ключей либо отсутствии/ошибки в настройке авторизации - отношения соседства не происходит. Совершим "диверсию" - изменим настройки в протоколе маршрутизации и на одном из интерфейсов на LEAF-3.
```
LEAF-3# sh isis adj
IS-IS process: UNDERLAY VRF: default
IS-IS adjacency database:
Legend: '!': No AF level connectivity in given topology
System ID       SNPA            Level  State  Hold Time  Interface
0100.0000.1000  N/A             2      UP     00:00:27   Ethernet1/1

LEAF-3#
LEAF-3# sh isis top
IS-IS process: UNDERLAY
VRF: default
Topology ID: 0

IS-IS Level-1 IS routing table

IS-IS Level-2 IS routing table
```
Неправильно написали имя key chain на LEAF-3 в настройках протокола и на интерфейсе Eth1/2 и всё поломалось...<br>

Обновленные конфиги устройств добавлены в файле lab3-auth.zip