# Лабораторная работа 1 - Проектирование адресного пространства

Для представленной на схеме ниже топологии зарезервируем адресное пространство для Spine, Leaf и хостов.

![CLOS](https://github.com/dim-sem/OTUS-Homework/blob/main/Lab-1/stand-22636-d9822e.avif "Схема CLOS")

Для практической работы используем эмуляцию сети в ПО EVE-NG, сеть будем собирать на Cisco Nexus.<br>
При подключении сетевых устройств следуем следующему соглашению:
```
Spine-1 Eth1/1 -> Leaf-1 Eth1/1
Spine-1 Eth1/2 -> Leaf-2 Eth1/1
Spine-1 Eth1/3 -> Leaf-3 Eth1/1

Spine-2 Eth1/1 -> Leaf-1 Eth1/2
Spine-2 Eth1/2 -> Leaf-2 Eth1/2
Spine-2 Eth1/3 -> Leaf-3 Eth1/2
```
Для Loopback интерфейсов IPv4 адреса выбираем по схеме: <br>
10.0.X.Y/32, для Spine<br>
10.1.X.Y/32, для Leaf<br>
10.2.X.Y/31 - p2p links<br>

Т.о. имеем следующие IPv4 адреса для узлов:
```
10.0.1.0/32 - Lo1 Spine-1
10.0.2.0/32 - Lo1 Spine-3

10.1.1.1/32 - Lo1 Leaf-1
10.1.1.2/32 - Lo1 Leaf-2
10.1.1.3/32 - Lo1 Leaf-3

10.2.1.0/31 - p2p S1 - L1
10.2.1.2/31 - p2p S1 - L2
10.2.1.4/31 - p2p S1 - L3

10.2.2.0/31 - p2p S2 - L1
10.2.2.2/31 - p2p S2 - L2
10.2.2.4/31 - p2p S2 - L3
```
Т.о. образом получается такая схема:
![clos-lab](https://github.com/dim-sem/OTUS-Homework/blob/main/Lab-1/clos-lab1.png "Итоговая схема")

Для подключенных хостов будем использовать адресное пространство:
```
172.16.1.0/24 - network A (VPC1)
172.16.2.0/24 - network B (VPC2)
172.16.3.0/24 - network C (VPC3)
172.16.4.0/24 - network D (VPC4)
172.16.5.0/24 - network E (VPC5)
```
