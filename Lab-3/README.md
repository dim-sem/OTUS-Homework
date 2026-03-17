# Лабораторная работа 2 - Underlay ISIS

В данной работе настроим underlay сеть для работы по протоколу маршрутизации ISIS.<br>
Используем сеть, собранную в первой лабораторной работе как основу для дальнейшей модернизации.<br>
![LAP-topology](https://github.com/dim-sem/OTUS-Homework/blob/main/Lab-3/clos-lab1.png "Topology")<br>
На Spine и Leaf объявим использование протокола IS-IS:
```
router isis UNDERLAY
net 49.XX.XXXX.XXXX.XXXX.XX
address-family ipv4 unicast
bfd