# VXVLAN EVPN



### Топология





![тут скрин 1](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_06_VXVLAN_EVPN/screenshots/Screenshot_1.jpg)



### Цели:

изучить технологию VxLAN;
рассмотреть типы маршрутов EVPN;
построить базовый overlay сети с помощью EVPN.

За underlay берем уже настроенный до этого eBGP в:



https://github.com/degreekeeper/otus-network-arch/tree/main/LAB_04_UNDERLAY_BGP



### Настрйка SPINE:



#### SPINE1



#### Включаем необходимые фичи:



nv overlay evpn
feature bgp
feature nv overlay



#### тут необходимые фильтры, префикс-листы и роут-мап, настроенные ещё в лабе по underlay ebgp



```
ip prefix-list DIRECT seq 5 permit 10.0.0.0/8 eq 24
ip prefix-list LOOPBACK seq 10 permit 10.255.0.0/16 eq 32
route-map LEAF-AS permit 10
  match as-number 65011-65019
route-map REDIST_LOOP_DIRECT permit 10
  match ip address prefix-list DIRECT LOOPBACK
```



#### **Дополнительный фильтр который настраивается на все L2VPN EVPN сессии BGP, чтобы не менялся next-hop.  Т.к. EVPN сессия должна строится между LEAF, а она строится на основании next-hop.**



```
route-map UNCHANGED permit 10
  set ip next-hop unchanged
```



#### Настройка BGP, AF L2VPN EVPN



***ebgp-multihop 2*** - нужен чтобы устанавливать сессию с source loopback, а не порта

  ***address-family l2vpn evpn***
    ***retain route-target all*** - нужен чтобы SPINE передавал транзитом информацию о RT

 ***send-community***
  ***send-community extended*** - нужный чтобы передавать эти самые RT


***route-map UNCHANGED out*** - нужен чтобы применть route-map, запрещающий изменять next-hop, на исходящие анонсы



```
router bgp 65010
  bestpath as-path multipath-relax
  log-neighbor-changes
  address-family ipv4 unicast
    redistribute direct route-map REDIST_LOOP_DIRECT
    maximum-paths 3
  address-family l2vpn evpn
    retain route-target all
  template peer CORE
    remote-as 65000
  template peer LEAF
    address-family ipv4 unicast
  template peer LEAF-EVPN
    update-source loopback0
    ebgp-multihop 2
    address-family l2vpn evpn
      send-community
      send-community extended
	  route-map UNCHANGED out
  neighbor 10.0.1.1
    inherit peer CORE
    address-family ipv4 unicast
  neighbor 10.255.0.4
    inherit peer LEAF-EVPN
    remote-as 65014
  neighbor 10.255.0.5
    inherit peer LEAF-EVPN
    remote-as 65015
  neighbor 10.255.0.6
    inherit peer LEAF-EVPN
    remote-as 65016
  neighbor 10.0.0.0/9 remote-as route-map LEAF-AS
    inherit peer LEAF
    address-family ipv4 unicast
```





#### Пример BGP-сессий в L2PVN EVPN SP1





![тут скрин 12](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_06_VXVLAN_EVPN/screenshots/Screenshot_12.jpg)





#### Пример BGP-сессий в L2PVN EVPN SP2





![тут скрин 13](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_06_VXVLAN_EVPN/screenshots/Screenshot_13.jpg)





### Настройки  LEAF



#### LEAF4



#### Необходимые фичи



```
nv overlay evpn
feature bgp
feature interface-vlan
feature vn-segment-vlan-based
feature nv overlay
```





#### Настройки фильтров, префикс-листов, роут-мап, так же настройки vlan и fabric arp



***fabric forwarding anycast-gateway-mac 0000.0000.4444*** - нужен чтобы интерфейс-влан имел мак-адрес и отвечал на ARP-запросы


***vlan 10***

 ***vn-segment 10010*** - этой командой привязываем vlan к VNI 10010



```
fabric forwarding anycast-gateway-mac 0000.0000.4444
vlan 1,10
vlan 10
  name VXVLAN
  vn-segment 10010

ip prefix-list DIRECT seq 5 permit 10.0.0.0/8 eq 24
ip prefix-list LOOPBACK seq 10 permit 10.255.0.0/16 eq 32
ip prefix-list LOOPBACK seq 20 permit 10.254.0.0/16 eq 31
route-map REDIST_LOOP_DIRECT permit 10
  match ip address prefix-list DIRECT LOOPBACK
```





#### Поднимаем интерфейсы loopback1, vlan 10, nve1



***fabric forwarding mode anycast-gateway*** - нужен, чтобы влан-интерфейс отвечал на arp



***host-reachability protocol bgp*** - через какой протокол будет работать NVE1



***source-interface loopback1*** - к какому интерфейсу привязан NVE1



  ***member vni 10010***
    ***ingress-replication protocol bgp*** - NVE1 передаёт информацию о NVI 10010, через BGP





```
interface loopback1
  description NVE LOOPBACK
  ip address 10.254.4.4/31

interface Vlan10
  no shutdown
  ip address 192.168.10.4/24
  fabric forwarding mode anycast-gateway

interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback1
  member vni 10010
    ingress-replication protocol bgp
```



#### Настройки BGP и EVPN



***evpn*
  *vni 10010 l2***
    ***rd auto***
      ***route-target import auto*** - режим auto можно указывать если в качестве underlay у нас IBGP и AS одинаковая у всех LEAF. Т.к. он должен совпадать.
      ***route-target import 9999:10010*** - статический нужно указывать если используется underlay EBGP, т.к. AS будут разные
      ***route-target export auto***
      ***route-target export 9999:10010***

​    

  ***route-target import auto*** - режим auto можно указывать, если в качестве underlay у нас IBGP и AS одинаковая у всех LEAF. RT должен совпадать с обоих сторон, а в автоматическом режиме значения RT будут браться из - AS:VNI. Так что для IBGP он подходит.

  ***route-target import 9999:10010*** - статический RT нужно указывать если используется underlay EBGP, т.к. AS будут разные. В этом случае если указать автоматический режим значения RT - AS:VNI не будут совпадать и анонсы не будут приниматься.





```
router bgp 65014
  bestpath as-path multipath-relax
  log-neighbor-changes
  address-family ipv4 unicast
    redistribute direct route-map REDIST_LOOP_DIRECT
    maximum-paths 3
  template peer SPINE
    remote-as 65010
    address-family ipv4 unicast
  template peer SPINE-EVPN
    update-source loopback0
    ebgp-multihop 2
    address-family l2vpn evpn
      send-community
      send-community extended
  neighbor 10.1.4.1
    inherit peer SPINE
    address-family ipv4 unicast
  neighbor 10.2.4.1
    inherit peer SPINE
    address-family ipv4 unicast
  neighbor 10.255.0.1
    inherit peer SPINE-EVPN
    remote-as 65010
  neighbor 10.255.0.2
    inherit peer SPINE-EVPN
    remote-as 65010
evpn
  vni 10010 l2
    rd auto
    route-target import auto
    route-target import 9999:10010
    route-target export auto
    route-target export 9999:10010
```



Настройки на портах в сторону хостов(серверов):



```
interface Ethernet1/7
  description NODE9_eth0/0_DOWNLINK
  switchport mode trunk
  switchport trunk allowed vlan 10
```





#### Посмотрим таблицу L2VPN BGP LF4, а так же EVPN-сессии, туннели EVPN:





![тут скрин 8](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_06_VXVLAN_EVPN/screenshots/Screenshot_8.jpg)





![скрин 9](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_06_VXVLAN_EVPN/screenshots/Screenshot_9.jpg)





#### Посмотрим таблицу L2VPN BGP LF5, а так же EVPN-сессии, туннели EVPN:





![](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_06_VXVLAN_EVPN/screenshots/Screenshot_10.jpg)





![](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_06_VXVLAN_EVPN/screenshots/Screenshot_11.jpg)





### Настройки на серверах(хостах):



#### NODE9





```
!
interface Ethernet0/0
 description LF4_eth1/7_UPLINK
 no ip address
!
interface Ethernet0/0.10
 description VXVLAN_LF4
 encapsulation dot1Q 10
 ip address 192.168.10.9 255.255.255.0
!
```



#### NODE10




```
!
interface Ethernet0/0
 description LF5_eth1/7_UPLINK
 no ip address
!
interface Ethernet0/0.10
 description VXVLAN_LF5
 encapsulation dot1Q 10
 ip address 192.168.10.10 255.255.255.0
!
```









### Проверка работы VXVLAN





Проверяем arp  и ping с NODE9



![скрин 2](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_06_VXVLAN_EVPN/screenshots/Screenshot_2.jpg)




Проверяем apr и ping с NODE10





![скрин 3](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_06_VXVLAN_EVPN/screenshots/Screenshot_3.jpg)





Смотрим анонсы в EVPN на Leaf 4



![скрин 4](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_06_VXVLAN_EVPN/screenshots/Screenshot_4.jpg)




смотрим анонсы в EVPN  на Leaf 5





![скирн 5](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_06_VXVLAN_EVPN/screenshots/Screenshot_5.jpg)






### Конец.