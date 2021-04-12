# VXLAN L3VNI





### Топология






![скрин 1](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_08_VXLAN_L3VNI/screenshots/Screenshot_1.jpg)





### Цели лабораторной:





1. Изучить маршрутизацию между различными VNI;
2. Настройка маршрутизации между разными подсетями с помощью L3VNI. В том числе с применением VPC
3. Настройка MULTIPOD





#### Команды:


***show ip route vrf L3VNI_9999_33333*** - посмотреть маршруты в конкретном VRF





#### Изменения в функциях хостов в лабораторной:





1. CORE-ZERO превращен в обычный коммутатор который в транке во VLAN 99 связывает все SPINE по L2. Это его единственная функция.
2. Port-Channel 7 который работает по VPC на Leaf5,6 и Port-Channel 1 который работает на NODE10 перенастроен в switchport trunk
3. На NODE10 для маршрутизации поднят SVI interface vlan 10.







### Настройка MULTIPOD



На примере SP1


POD будут связаны между собой с помощью eBGP сессий IPV4 и L2VPN EVPN через коммутатор CORE-ZERO




Настраиваем порт до SP3

```
interface Ethernet1/1.99
  description MULTIPOD_to_SP3
  encapsulation dot1q 99
  ip address 10.99.13.1/24
  no shutdown
```





Данный route-map нужен, чтобы исключить изменения next-hop на SPINE, т.к. у нас используется eBGP

```
route-map UNCHANGED permit 10
  set ip next-hop unchanged
```



тут мы применяем route-map, а так же применяем ebgp-multihop, т.к. сессии у нас подняты на loopback

```
router bgp 65010

  template peer LEAF-EVPN
    update-source loopback0
    ebgp-multihop 2
    address-family l2vpn evpn
      send-community
      send-community extended
      route-map UNCHANGED out
```



Пример настройки сессий


```
  template peer MULTIPOD
    remote-as 65020
    address-family ipv4 unicast
```

```
  neighbor 10.99.13.3
    inherit peer MULTIPOD
```

```
  neighbor 10.255.0.3
    inherit peer LEAF-EVPN
    remote-as 65020
```



#### Все сессии в IPV4 и L2VPN на SP1


![тут скрин 10](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_08_VXLAN_L3VNI/screenshots/Screenshot_10.jpg)

#### Все сессии в IPV4 и L2VPN на SP2


![тут скрин 11](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_08_VXLAN_L3VNI/screenshots/Screenshot_11.jpg)

#### Все сессии в IPV4 и L2VPN на SP3


![тут скрин 12](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_08_VXLAN_L3VNI/screenshots/Screenshot_12.jpg)






### Настройка L3VNI


Настраивается на LEAF



Можно разделить на этапы.  Возьмем для примеры LF4.



1. Создание специального ВЛАН и привязка его к отдельному VNI

   

   ```
   vlan 999
     name L3VNI_VLAN
     vn-segment 33333
   ```

   



2. Создание VRF. Привязка его к VNI. Добавление ему статического RT в EVPN, так и в IPV4.



```
vrf context L3VNI_9999_33333
  vni 33333
  rd auto
  address-family ipv4 unicast
    route-target import 9999:33333
    route-target import 9999:33333 evpn
    route-target export 9999:33333
    route-target export 9999:33333 evpn
    route-target both auto
    route-target both auto evpn
```



3. Добавления vlan-интерфейса под хосты в VRF. А так же создания специального vlan-интерфейса по L3VNI и включение его в VRF

```
interface Vlan10
  no shutdown
  vrf member L3VNI_9999_33333
  ip address 192.168.10.4/24
  fabric forwarding mode anycast-gateway

interface Vlan999
  description L3VNI_FORWARD
  no shutdown
  vrf member L3VNI_9999_33333
  ip forward
```



4. Добавление L3VNI в туннельный интерфейс NVE1 и привязка L3VNI к VRF



```
interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback1
  member vni 10010
    ingress-replication protocol bgp
  member vni 33333 associate-vrf
```





Аналогичные настройки для LF5 и  LF6




```
vlan 999
  name L3VNI_VLAN
  vn-segment 33333
```




```
vrf context L3VNI_9999_33333
  vni 33333
  rd auto
  address-family ipv4 unicast
    route-target import 9999:33333
    route-target import 9999:33333 evpn
    route-target export 9999:33333
    route-target export 9999:33333 evpn
    route-target both auto
    route-target both auto evpn
```



```
interface Vlan10
  no shutdown
  vrf member L3VNI_9999_33333
  ip address 192.168.10.99/24
  fabric forwarding mode anycast-gateway

interface Vlan999
  description L3VNI_FORWARD
  no shutdown
  vrf member L3VNI_9999_33333
  ip forward
```



```
interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback1
  member vni 10010
    ingress-replication protocol bgp
  member vni 33333 associate-vrf
```




Настройки для LF7 из второго POD



```
vlan 999
  name L3VNI_VLAN
  vn-segment 33333

```



```
vrf context L3VNI_9999_33333
  vni 33333
  rd auto
  address-family ipv4 unicast
    route-target import 9999:33333
    route-target import 9999:33333 evpn
    route-target export 9999:33333
    route-target export 9999:33333 evpn
    route-target both auto
    route-target both auto evpn
```



```
interface Vlan20
  no shutdown
  vrf member L3VNI_9999_33333
  ip address 192.168.20.7/24
  fabric forwarding mode anycast-gateway

interface Vlan999
  description L3VNI_FORWARD
  no shutdown
  vrf member L3VNI_9999_33333
  ip forward
```



```
interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback1
  member vni 20020
    ingress-replication protocol bgp
  member vni 33333 associate-vrf
```







### Проверка работы:



Обращаем внимание, что маршрутизация начинает работать только после пинга шлюза и появления в таблице маршрутизации VRF маршрута до хоста через HMM.



##### На NODE9. Хосты NODE10 и NODE11 доступны.


Обращаем внимаение, что 192.168.10.10 в одном L2-сегменте и виден его ARP. Когда как 192.168.20.11 находится в другой подсети и доступен через шлюз по умолчанию и L3VNI


![скрин 2](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_08_VXLAN_L3VNI/screenshots/Screenshot_2.jpg)



Смотрим таблицу маршрутизации на LF4 и обращаем внимание на 3 маршрута: 1. до хоста через HMM, 2. до 192.168.10.10 через L2VNI, 3. до 192.168.20.11 через L3VNI. Next-hop будет туннель VXLAN, как и энкапсуляция.


![скрин 3](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_08_VXLAN_L3VNI/screenshots/Screenshot_3.jpg)

##### Аналогичная проверка на NODE10. Хосты NODE9 и NODE11 доступны.





Обращаем внимание, что теперь есть arp от 192.168.10.9, 192.168.20.11 так же доступен через маршрутизацию



![скрин 4](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_08_VXLAN_L3VNI/screenshots/Screenshot_4.jpg)



Смотрим таблицу маршрутизации VRF и nv peers на LF5 и LF6


![скрин 5](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_08_VXLAN_L3VNI/screenshots/Screenshot_5.jpg)


![скрин 6](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_08_VXLAN_L3VNI/screenshots/Screenshot_6.jpg)



##### Проверка на NODE11. Хосты NODE9 и NODE10 доступны.




Тут не будет arp, т.к. нет ни одного хоста в L2VNI. Все хосты доступны через L3VNI.



![скрин 7](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_08_VXLAN_L3VNI/screenshots/Screenshot_7.jpg)



Посмотрим таблицу маршрутизации в VRF и увидим оба хоста доступны черзе L3VNI и туннель-VXLAN. Next-hop это ip NVE-интерфейсов удаленных VTEP.


![скрин 8](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_08_VXLAN_L3VNI/screenshots/Screenshot_8.jpg)




Тут так же просмотрим таблицу анонсов в EVPN на LF7



![скрин 9](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_08_VXLAN_L3VNI/screenshots/Screenshot_9.jpg)





### Конец.