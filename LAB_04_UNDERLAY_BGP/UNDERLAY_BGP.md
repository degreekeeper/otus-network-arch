# UNDERLAY. BGP



### Топология



![тут скриншот 1](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_04_UNDERLAY_BGP/screenshots/Screenshot_1.jpg)





#### Цель: Настроить BGP для Underlay сети



1. настроить BGP в Underlay сети, для IP связанности между всеми устройствами NXOS
2. План работы, адресное пространство, схема сети, настройки - зафиксированы в документации





#### Настройка CORE-ZERO



#### Настройка BGP



```
!
interface Loopback0 - лупбак для BGP
 description BGP LOOPBACK
 ip address 10.255.0.0 255.255.255.255
!
router bgp 65000
 bgp router-id 10.255.0.0 - роутер-id равен адресу loopback
 bgp log-neighbor-changes - логируем изменения в сессиях
 no bgp default ipv4-unicast - отключаем автоматическое прописывание active AF
 neighbor 10.0.1.2 remote-as 65010
 neighbor 10.0.2.2 remote-as 65010
 neighbor 10.0.3.2 remote-as 65020
 !
 address-family ipv4
  redistribute connected route-map TO-SPINE - редистрибуция коннектед сетей в соответсвии с роут-мап
  redistribute static route-map TO-SPINE - редистрибуция статик сетей в соответсвии с роут-мап
  neighbor 10.0.1.2 activate
  neighbor 10.0.2.2 activate
  neighbor 10.0.3.2 activate
  maximum-paths 3 - включение мульти-path
  default-information originate - анонс дефолта
 exit-address-family


```



#### Настройка фильтрации редистрибуции через префикс-листы и роут-мап



```
ip route 0.0.0.0 0.0.0.0 Loopback8888 - маршрут по умолчанию
!
!
ip prefix-list 8888 seq 5 permit 8.8.8.8/32 - префикс-лист для исключения анонас 8.8.8.8/32
!
ip prefix-list DEFAULT seq 5 permit 0.0.0.0/0 - префикс лист для анонса дефолта
!
route-map TO-SPINE permit 5 - роут-мап для редистрибуции в BGP коннектед сетей, дефолта и запрета 8.8.8.8/32
 match ip address prefix-list DEFAULT - разрешаем дефолт
!
route-map TO-SPINE deny 10 - запрещаем 8.8.8.8/32
 match ip address prefix-list 8888
!
route-map TO-SPINE permit 20 - разрешаем все коннектед
 match source-protocol connected
!
```



### Настройка SPINE на примере SP1



#### Настройка BGP



```
interface loopback0 - лупбак для BGP
  description BGP LOOPBACK
  ip address 10.255.0.1/32

router bgp 65010
  router-id 10.255.0.1
  bestpath as-path multipath-relax - включение multi-pathing при разных AS-PATH
  address-family ipv4 unicast
    redistribute direct route-map REDIST_LOOP_DIRECT - редистрибуция в BGP коннектед сетей и loopback через роут-мап
    maximum-paths 3 - включение multi-pathing
  template peer CORE - шаблон для подключения к CORE
    remote-as 65000
  template peer LEAF - шаблон для подключения к LEAF
    address-family ipv4 unicast
  neighbor 10.0.1.1 - нейбор до CORE
    inherit peer CORE - настройки нейбора в соответствии с шаблоном
    address-family ipv4 unicast 
  neighbor 10.0.0.0/8 remote-as route-map LEAF-AS - динамический диапазон нейборов с диапазоном как адресов так и диапазоном AS
    inherit peer LEAF - включение шаблона для всего диапазона нейборов
    address-family ipv4 unicast
```



#### Настройка префикс-листов и роут-мап



```
ip prefix-list DIRECT seq 5 permit 10.0.0.0/8 eq 24 - префикс-лист для редистрибуции коннектед сетей
ip prefix-list LOOPBACK seq 10 permit 10.255.0.0/16 eq 32 - префикс лист для редистрибуции loopback
route-map LEAF-AS permit 10 - роут-мап для указания нейборов до LEAF сразу диапазоноам
  match as-number 65011-65019 - диапазон AS для LEAF в первом POD
route-map REDIST_LOOP_DIRECT permit 10 - роут-мап для редистрибуции коннектед сетей и loopback в BGP
  match ip address prefix-list DIRECT LOOPBACK - префикс-листы DIRECT и LOOPBACK
```



### Настройка LEAF на примере LF4





#### Настройка BGP



```
interface loopback0 - лупбак для BGP
  description BGP LOOPBACK
  ip address 10.255.0.4/32



router bgp 65014
  router-id 10.255.0.4
  bestpath as-path multipath-relax  - включение multi-pathing при разных AS-PATH
  address-family ipv4 unicast
    redistribute direct route-map REDIST_LOOP_DIRECT - редистрибуция в BGP коннектед сетей и loopback через роут-мап
    maximum-paths 3  - включение multi-pathing
  template peer SPINE - шаблон для нейбора до SPINE
    remote-as 65010
    address-family ipv4 unicast
  neighbor 10.1.4.1
    inherit peer SPINE - включение настроек нейбора через шаблон SPINE
    address-family ipv4 unicast
  neighbor 10.2.4.1
    inherit peer SPINE
    address-family ipv4 unicast
```



Настройки префикс-листов и роут-мап



```
ip prefix-list DIRECT seq 5 permit 10.0.0.0/8 eq 24 - префикс-лист для редистрибуции коннектед сетей
ip prefix-list LOOPBACK seq 10 permit 10.255.0.0/16 eq 32 - префикс лист для редистрибуции loopback
route-map REDIST_LOOP_DIRECT permit 10 - роут-мап для редистрибуции коннектед сетей и loopback в BGP
  match ip address prefix-list DIRECT LOOPBACK - префикс-листы DIRECT и LOOPBACK
```





### Проверка связанности



#### CORE-ZERO



##### Таблица BGP



![тут скрин 2](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_04_UNDERLAY_BGP/screenshots/Screenshot_2.jpg)



##### Таблица соседств BGP



![тут скрин 3](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_04_UNDERLAY_BGP/screenshots/Screenshot_3.jpg)



##### Таблица маршрутизации



![туту скрин 4](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_04_UNDERLAY_BGP/screenshots/Screenshot_4.jpg)



### SPINE - SP1



##### Таблица BGP



![тут скрин 5](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_04_UNDERLAY_BGP/screenshots/Screenshot_5.jpg)



##### Таблица соседств BGP



![тут скрин 6](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_04_UNDERLAY_BGP/screenshots/Screenshot_6.jpg)



##### Таблица маршрутизации



![туту скрин 7](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_04_UNDERLAY_BGP/screenshots/Screenshot_7.jpg)





### LEAF - LF4



##### Таблица BGP



![тут скрин 8](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_04_UNDERLAY_BGP/screenshots/Screenshot_8.jpg)



##### Таблица соседств BGP



![тут скрин 9](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_04_UNDERLAY_BGP/screenshots/Screenshot_9.jpg)



##### Таблица маршрутизации



```
LF4# show ip route
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

0.0.0.0/0, ubest/mbest: 2/0
    *via 10.1.4.1, [20/0], 07:20:17, bgp-65014, external, tag 65010
    *via 10.2.4.1, [20/0], 03:27:24, bgp-65014, external, tag 65010
10.0.1.0/24, ubest/mbest: 1/0
    *via 10.1.4.1, [20/0], 10:28:42, bgp-65014, external, tag 65010
10.0.2.0/24, ubest/mbest: 1/0
    *via 10.2.4.1, [20/0], 03:27:24, bgp-65014, external, tag 65010
10.0.3.0/24, ubest/mbest: 2/0
    *via 10.1.4.1, [20/0], 06:58:16, bgp-65014, external, tag 65010
    *via 10.2.4.1, [20/0], 03:27:24, bgp-65014, external, tag 65010
10.1.4.0/24, ubest/mbest: 1/0, attached
    *via 10.1.4.2, Eth1/1, [0/0], 3d10h, direct
10.1.4.2/32, ubest/mbest: 1/0, attached
    *via 10.1.4.2, Eth1/1, [0/0], 3d10h, local
10.1.5.0/24, ubest/mbest: 1/0
    *via 10.1.4.1, [20/0], 10:28:42, bgp-65014, external, tag 65010
10.1.6.0/24, ubest/mbest: 1/0
    *via 10.1.4.1, [20/0], 10:28:42, bgp-65014, external, tag 65010
10.2.4.0/24, ubest/mbest: 1/0, attached
    *via 10.2.4.2, Eth1/2, [0/0], 3d10h, direct
10.2.4.2/32, ubest/mbest: 1/0, attached
    *via 10.2.4.2, Eth1/2, [0/0], 3d10h, local
10.2.5.0/24, ubest/mbest: 1/0
    *via 10.2.4.1, [20/0], 03:27:24, bgp-65014, external, tag 65010
10.2.6.0/24, ubest/mbest: 1/0
    *via 10.2.4.1, [20/0], 03:27:24, bgp-65014, external, tag 65010
10.3.7.0/24, ubest/mbest: 2/0
    *via 10.1.4.1, [20/0], 03:08:38, bgp-65014, external, tag 65010
    *via 10.2.4.1, [20/0], 03:08:38, bgp-65014, external, tag 65010
10.4.9.0/24, ubest/mbest: 1/0, attached
    *via 10.4.9.1, Eth1/7, [0/0], 3d10h, direct
10.4.9.1/32, ubest/mbest: 1/0, attached
    *via 10.4.9.1, Eth1/7, [0/0], 3d10h, local
10.5.6.0/24, ubest/mbest: 2/0
    *via 10.1.4.1, [20/0], 02:45:30, bgp-65014, external, tag 65010
    *via 10.2.4.1, [20/0], 02:45:30, bgp-65014, external, tag 65010
10.5.10.0/24, ubest/mbest: 2/0
    *via 10.1.4.1, [20/0], 02:45:24, bgp-65014, external, tag 65010
    *via 10.2.4.1, [20/0], 02:45:24, bgp-65014, external, tag 65010
10.6.10.0/24, ubest/mbest: 2/0
    *via 10.1.4.1, [20/0], 03:19:16, bgp-65014, external, tag 65010
    *via 10.2.4.1, [20/0], 03:19:16, bgp-65014, external, tag 65010
10.7.11.0/24, ubest/mbest: 2/0
    *via 10.1.4.1, [20/0], 02:36:29, bgp-65014, external, tag 65010
    *via 10.2.4.1, [20/0], 02:36:29, bgp-65014, external, tag 65010
10.255.0.0/32, ubest/mbest: 2/0
    *via 10.1.4.1, [20/0], 06:58:16, bgp-65014, external, tag 65010
    *via 10.2.4.1, [20/0], 03:27:24, bgp-65014, external, tag 65010
10.255.0.1/32, ubest/mbest: 1/0
    *via 10.1.4.1, [20/0], 10:28:42, bgp-65014, external, tag 65010
10.255.0.2/32, ubest/mbest: 1/0
    *via 10.2.4.1, [20/0], 03:27:24, bgp-65014, external, tag 65010
10.255.0.3/32, ubest/mbest: 2/0
    *via 10.1.4.1, [20/0], 03:08:38, bgp-65014, external, tag 65010
    *via 10.2.4.1, [20/0], 03:08:38, bgp-65014, external, tag 65010
10.255.0.4/32, ubest/mbest: 2/0, attached
    *via 10.255.0.4, Lo0, [0/0], 11:33:38, local
    *via 10.255.0.4, Lo0, [0/0], 11:33:38, direct
10.255.0.5/32, ubest/mbest: 2/0
    *via 10.1.4.1, [20/0], 02:45:24, bgp-65014, external, tag 65010
    *via 10.2.4.1, [20/0], 02:45:24, bgp-65014, external, tag 65010
10.255.0.6/32, ubest/mbest: 2/0
    *via 10.1.4.1, [20/0], 03:19:16, bgp-65014, external, tag 65010
    *via 10.2.4.1, [20/0], 03:19:16, bgp-65014, external, tag 65010
10.255.0.7/32, ubest/mbest: 2/0
    *via 10.1.4.1, [20/0], 02:36:29, bgp-65014, external, tag 65010
    *via 10.2.4.1, [20/0], 02:36:29, bgp-65014, external, tag 65010
```





### Пинги и трейсы с хостов(NODE)



#### NODE9



![тут скрин 10](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_04_UNDERLAY_BGP/screenshots/Screenshot_10.jpg)



#### NODE10



![тут скрин 11](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_04_UNDERLAY_BGP/screenshots/Screenshot_11.jpg)



#### NODE11



![тут скрин 12](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_04_UNDERLAY_BGP/screenshots/Screenshot_12.jpg)







### Конец.