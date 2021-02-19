# MULTICAST



### Топология



![тут скрин 0](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_05_MULTICAST/screenshots/Screenshot_0.jpg)



### Цель: Настроить PIM в сети.



1. Настроить PIM на всех устройствах (кроме коммутаторов доступа); *Для IP связанности между устройствами можно использовать любой протокол динамической маршрутизации;
2. План работы, адресное пространство, схема сети, настройки - зафиксируете в документации;



#### Технические решения:



1. За RP выбираем центральный маршрутизатор CORE-ZERO

2. Для ip связанности используем протокол BGP



#### Настройки на CORE-ZERO



```
!
ip multicast-routing
!
interface Ethernet0/1
 description SP1_eth1/1_DOWNLINK
 ip address 10.0.1.1 255.255.255.0
 ip pim sparse-mode
!
interface Ethernet0/2
 description SP2_eth1/1_DOWNLINK
 ip address 10.0.2.1 255.255.255.0
 ip pim sparse-mode
!
interface Ethernet0/3
 description SP3_eth1/1_DOWNLINK
 ip address 10.0.3.1 255.255.255.0
 ip pim sparse-mode
!
ip pim rp-address 10.255.0.0
```



На каждом из хостов NODE9,10,11 будет запущен статический igmp-join изображающий клиента на группы соотвественно: 


*239.9.9.9* 

*239.10.10.10* 

*239.11.11.11*



#### Настройки на NODE9



```
!
interface Ethernet0/0
 description LF4_eth1/7_UPLINK
 ip address 10.4.9.2 255.255.255.0
 ip igmp join-group 239.9.9.9
!
```


Наличие джойна на Last-hop мультикаст-маршрутизаторе LF4




![скрин1](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_05_MULTICAST/screenshots/Screenshot_1.jpg)



#### Настройки на NODE10



```
!
interface Ethernet0/0
 description LF5_eth1/7_UPLINK
 ip address 10.5.10.2 255.255.255.0
 ip igmp join-group 239.10.10.10
!
interface Ethernet0/1
 description LF6_eth1/7_UPLINK
 ip address 10.6.10.2 255.255.255.0
 ip igmp join-group 239.10.10.10
!
```



Наличие джойна на вышестояющем мультикаст-маршрутизаторе LF5 и LF6



![тут скрин 2 и 3](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_05_MULTICAST/screenshots/Screenshot_2.jpg)


![](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_05_MULTICAST/screenshots/Screenshot_3.jpg)





Настройки на NODE11



```
!
interface Ethernet0/0
 description LF7_eth1/7_UPLINK
 ip address 10.7.11.2 255.255.255.0
 ip igmp join-group 239.11.11.11
!
```



Наличие джойна на вышестояющем мультикаст-маршрутизаторе LF7


![скрин 4](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_05_MULTICAST/screenshots/Screenshot_4.jpg)



#### Иммитация источников мультикаста и проверка наличия групп на Last-hop роутерах и First-Hop роутерах



Сделаем пинг до мультикастовых групп 239.10.10.10 и 239.11.11.11 с NODE9 изобразив таким образом источник мультикаста


![скрин 4_1](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_05_MULTICAST/screenshots/Screenshot_4_1.jpg)



Видим что LF4 стал First-hop Router. У него появились два маршрута до групп 239.10.10.10 и 239.11.11.11 где источником является ip-адрес NODE9 - 10.4.9.2, а Incoming interface eth1/7.


![тут скрин 5](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_05_MULTICAST/screenshots/Screenshot_5.jpg)




На LF5 и LF6 мы так же видим эту пару группа-источник (10.4.9.2/32, 239.10.10.10/32). Данные роутеры теперь являются Last-hop Router. Теперь они получают мультикаст трафик и отдают его по джойну от NODE10 (*, 239.10.10.10/32)


![тут скрин 6 и 7](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_05_MULTICAST/screenshots/Screenshot_6.jpg)


![](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_05_MULTICAST/screenshots/Screenshot_7.jpg)




На LF7 видим маршурт до источника группы 239.11.11.11. Он будет являться Last-hop Router для этой мультикаст группы и отдавать трафик мультикаст на NODE11, которые его запрашивает по статик-джойну. Так же виден маршурт от First-hop Router LF4 (10.4.9.2/32, 239.11.11.11/32).


![тут скрин 8](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_05_MULTICAST/screenshots/Screenshot_8.jpg)




Изобразим источник мультикаст трафика для группы 239.9.9.9 на NODE11 с помощью пинга.


![тут скрин 9](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_05_MULTICAST/screenshots/Screenshot_9.jpg)



Аналогично видим, что LF7 для этой группы стал Last-Hop Router и получает пару источник-группа (10.7.11.2/32, 239.9.9.9/32) с интерфейса до NODE11 eth1/7 и адреса до него 10.7.11.2


![скрин 10](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_05_MULTICAST/screenshots/Screenshot_10.jpg)


а First-Hop Router стал LF4:


![тут скрин 11](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_05_MULTICAST/screenshots/Screenshot_11.jpg)

#### Пример настроек на SPINE



Настройки на SP1 (команда ip pim ssm range 232.0.0.0/8 появляется автоматически)




```
feature pim

ip pim rp-address 10.255.0.0 group-list 224.0.0.0/4
ip pim ssm range 232.0.0.0/8

interface Ethernet1/1
  description CORE_eth0/1_UPLINK
  no switchport
  ip address 10.0.1.2/24
  ip pim sparse-mode
  no shutdown

interface Ethernet1/2

interface Ethernet1/3

interface Ethernet1/4
  description LF4_eth1/1_DOWNLINK
  no switchport
  ip address 10.1.4.1/24
  ip pim sparse-mode
  no shutdown

interface Ethernet1/5
  description LF5_eth1/5_DOWNLINK
  no switchport
  ip address 10.1.5.1/24
  ip pim sparse-mode
  no shutdown

interface Ethernet1/6
  description LF6_eth1/1_DOWNLINK
  no switchport
  ip address 10.1.6.1/24
  ip pim sparse-mode
  no shutdown
```




Пример настроек на LEAF


Настройки на LF4 (команда ip pim ssm range 232.0.0.0/8 появляется автоматически)



```
feature pim

ip pim rp-address 10.255.0.0 group-list 224.0.0.0/4
ip pim ssm range 232.0.0.0/8

interface Ethernet1/1
  description SP1_eth1/4_UPLINK
  no switchport
  ip address 10.1.4.2/24
  ip pim sparse-mode
  no shutdown

interface Ethernet1/2
  description SP2_eth1/4_UPLINK
  no switchport
  ip address 10.2.4.2/24
  ip pim sparse-mode
  no shutdown

interface Ethernet1/3

interface Ethernet1/4

interface Ethernet1/5

interface Ethernet1/6

interface Ethernet1/7
  description NODE9_eth0/0_DOWNLINK
  no switchport
  ip address 10.4.9.1/24
  ip pim sparse-mode
  no shutdown
```




Таким образом все клиенты, запрашивающие мультикаст трафик, получают его при наличии источников.



### Конец.