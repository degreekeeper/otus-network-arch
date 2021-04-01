# VXLAN VPC





### Топология




![тут скрин 1](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_07_VXLAN_VPC/screenshots/Screenshot_1.jpg)




## Цели лабораторной:

рассмотреть технологию VPC;
разобрать изменения в логике VxLAN в связке с VPC.







### Команды:



***show running-config vpc*** - посмотреть конфигурацию VPC

***show vpc role*** - посмотреть является ли коммутатор primary/secondary

***show vpc peer-keepalive*** - посмотреть состояние соседа-VPC

***show vpc consistency-parameters global*** - посмотреть параметры которые не совпадают, из-за чего может не работать VPC





### Настройки хоста(сервера)





В этой лабораторной для настройки VPC необходимо использовать на хосте Port-channel. Поэтому L3-образ IOS был заменен на L2-образ IOS.





***channel-group 1 mode active*** - включаем порты в сторону LEAF в режим LACP.

***ip address 192.168.10.10 255.255.255.0*** - ip-адрес иммитирующий сервер настраиваем на агрегате Po.





```
!
interface Port-channel1
 description LACP to VPC
 no switchport
 ip address 192.168.10.10 255.255.255.0
!
interface Ethernet0/0
 description VXLAN_LF5
 no switchport
 no ip address
 duplex auto
 channel-group 1 mode active
!
interface Ethernet0/1
 description VXLAN_LF6
 no switchport
 no ip address
 duplex auto
 channel-group 1 mode active
!
```









### Настройка VPC




#### Настройка LF5



***feature vpc*** - включаем функционал VPC

***vrf context KEEP_VPC*** - нужно обязательно создать vrf для peer-keepalive

***vpc domain 1***

***peer-keepalive destination 10.5.6.2 source 10.5.6.1 vrf KEEP_VPC*** - настраиваем в режиме интерфейса где будет работать peer-keepalive. Адреса берем с интерфейсов между которыми будут ходить keepalive между коммутаторами. Он обязательно должен быть в vrf

***vrf member KEEP_VPC*** - эта команды вводится на интерфейсе, что включить его в vrf.

***vpc peer-link*** - настраиваем на линке где будет работать peer-link

***vpc 7*** - включаем на агрегате в сторону хостов



#### Настройка VPC



```
conf t

vrf context KEEP_VPC


feature vpc

vpc domain 1
  peer-keepalive destination 10.5.6.2 source 10.5.6.1 vrf KEEP_VPC

interface port-channel7
  vpc 7

interface port-channel99
  vpc peer-link
```





#### Настройка агрегированных портов, порта под peer-keepalive.





#### Под peer-link



```
interface port-channel99
  description VPC_PEER_LINK
  switchport mode trunk
  spanning-tree port type network
  vpc peer-link
```



```
interface Ethernet1/4
  description VPC_PEER_LINK
  switchport mode trunk
  channel-group 99 mode active

interface Ethernet1/5
  description VPC_PEER_LINK
  switchport mode trunk
  channel-group 99 mode active
```



#### Под peer-keepalive



```
interface Ethernet1/3
  description LF6_eth1/3_VPC_KEEPALIVE
  no switchport
  vrf member KEEP_VPC
  ip address 10.5.6.1/24
  no shutdown
```



#### В сторону хоста




```
interface port-channel7
  description VPC_to_NODE10_eth0/0
  switchport access vlan 10
  switchport trunk allowed vlan 10
  vpc 7
```



```
interface Ethernet1/7
  description NODE10_eth0/0_DOWNLINK
  switchport access vlan 10
  switchport trunk allowed vlan 10
  channel-group 7 mode active
```





#### Настройка Loopback1 к которому привязан NVE1 и добавление одинакового ip-secondary





```
interface loopback1
  description NVE LOOPBACK
  ip address 10.254.5.5/31
  ip address 10.254.99.99/31 secondary
```





#### Настройка Vlan10  с одинаковым ip и fabric-forwarding





```
conf t

fabric forwarding anycast-gateway-mac 0000.0000.6666


interface Vlan10
  no shutdown
  ip address 192.168.10.99/24
  fabric forwarding mode anycast-gateway


```




#### Смотрим состояние VPC и Port-channel


![тут скрин_2](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_07_VXLAN_VPC/screenshots/Screenshot_2.jpg)

![скрин3](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_07_VXLAN_VPC/screenshots/Screenshot_3.jpg)

![скрин 4](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_07_VXLAN_VPC/screenshots/Screenshot_4.jpg)




#### Настройка LF6



#### Настройки VPC



```
feature vpc

vrf context KEEP_VPC

vpc domain 1
  peer-keepalive destination 10.5.6.1 source 10.5.6.2 vrf KEEP_VPC

interface port-channel7
  vpc 7

interface port-channel99
  vpc peer-link
```





#### Настройка портов и агрегатов





```
interface Vlan10
  no shutdown
  ip address 192.168.10.99/24
  fabric forwarding mode anycast-gateway
```





```
interface port-channel7
  description VPC_to_NODE10_eth0/1
  switchport access vlan 10
  switchport trunk allowed vlan 10
  vpc 7

interface Ethernet1/7
  description NODE10_eth0/1
  switchport access vlan 10
  switchport trunk allowed vlan 10
  channel-group 7 mode active


```



```
interface port-channel99
  description VPC_PEER_LINK
  switchport mode trunk
  spanning-tree port type network
  vpc peer-link


interface Ethernet1/4
  description VPC_PEER_LINK
  switchport mode trunk
  channel-group 99 mode active

interface Ethernet1/5
  description VPC_PEER_LINK
  switchport mode trunk
  channel-group 99 mode active
```



```
interface Ethernet1/3
  description LF5_eth1/3_VPC_KEEPALIVE
  no switchport
  vrf member KEEP_VPC
  ip address 10.5.6.2/24
  no shutdown
```



```
interface loopback1
  description NVE LOOPBACK
  ip address 10.254.6.6/31
  ip address 10.254.99.99/31 secondary
```



#### Смотрим состояние VPC и Port-channel



![скрин_5](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_07_VXLAN_VPC/screenshots/Screenshot_5.jpg)


![скрин_6](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_07_VXLAN_VPC/screenshots/Screenshot_6.jpg)



![скрин_7](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_07_VXLAN_VPC/screenshots/Screenshot_7.jpg)





#### Проверка работы VPC



![скрин_9](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_07_VXLAN_VPC/screenshots/Screenshot_9.jpg)


![скрин_8](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_07_VXLAN_VPC/screenshots/Screenshot_8.jpg)





### Конец.