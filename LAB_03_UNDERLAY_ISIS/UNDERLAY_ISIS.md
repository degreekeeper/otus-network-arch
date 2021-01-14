# Underlay. ISIS.



### Топология



![тут скриншот 1](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_03_UNDERLAY_ISIS/screenshots/Screenshot_1.jpg)



#### Цель: Настроить IS-IS для Underlay сети

В этой самостоятельной работе мы ожидаем, что вы самостоятельно:

1. настроить IS-IS в Underlay сети, для IP связанности между всеми устройствами NXOS
2. План работы, адресное пространство, схема сети, настройки - зафиксированы в документации



#### Общие концепции:



1. ISO адрес будет выглядеть следующим образом: 49.000X.000Y.000Y.000Y.00
   где X - совпадает с номером зоны, Y - совпадает с цифрой хостнейма оборудования. Кроме AREA 0 - там Y будет равным 100, т.к. нельзя использовать адрес 49.0000.0000.0000.0000.00

2. Для уменьшения таблицы маршрутизации на Leaf внутри одно POD будет исключительно L1-взаимодействие. Со Spine на Leaf будут попадать только внутренние маршруты зоны и дефолтные маршруты от CORE-ZERO.

3. Между AREA100 и нижестоящими AREA будет настроено только L2-взаимодействие.

4. Маршрут по умолчанию с CORE-ZERO будет анонсировать по L2-взаимодействию через команду  ***default-information originate***.

5. Статический маршрут 8.8.8.8/32 анонсироваться не будет и используется для тестирования  внешнего взаимодействия за пределами ЦОД.

6. Для передачи database L1 в database L2 на SPINE будет использоваться команда 

   ```
     address-family ipv4 unicast
       distribute level-1 into level-2 all
   ```

   

7. На всех роутерах ISIS будет использоваться команда  ***metric-style transition*** для увеличения возможно значения метрики ISIS

8. Между LF5 и LF6 будет поднят ли под VPС так же на L1-уровне взаимодействия ISIS



#### Настройка CORE-ROUTER:



```
!
interface Loopback8888
 description DEFAULT
 ip address 8.8.8.8 255.255.255.255
!
interface Ethernet0/0
 no ip address
 shutdown
!
interface Ethernet0/1
 description SP1_eth1/1_DOWNLINK
 ip address 10.0.1.1 255.255.255.0
 ip router isis 1
 isis circuit-type level-2-only
!
interface Ethernet0/2
 description SP2_eth1/1_DOWNLINK
 ip address 10.0.2.1 255.255.255.0
 ip router isis 1
 isis circuit-type level-2-only
!
interface Ethernet0/3
 description SP3_eth1/1_DOWNLINK
 ip address 10.0.3.1 255.255.255.0
 ip router isis 1
 isis circuit-type level-2-only
!
router isis 1
 net 49.0000.0100.0100.0100.00
 metric-style transition
 log-adjacency-changes
 default-information originate
!
ip route 0.0.0.0 0.0.0.0 Loopback8888
!
```



#### Таблица соседств ISIS и  таблица маршрутизации ISIS:



![тут скрин 2](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_03_UNDERLAY_ISIS/screenshots/Screenshot_2.jpg)



### Настройка SPINE на примере SP1



#### Настройка интерфейсов



```
interface Ethernet1/1
  description CORE_eth0/1_UPLINK
  no switchport
  ip address 10.0.1.2/24
  isis circuit-type level-2
  ip router isis 1
  no shutdown


interface Ethernet1/4
  description LF4_eth1/1_DOWNLINK
  no switchport
  ip address 10.1.4.1/24
  isis network point-to-point
  isis circuit-type level-1
  ip router isis 1
  no shutdown

interface Ethernet1/5
  description LF5_eth1/5_DOWNLINK
  no switchport
  ip address 10.1.5.1/24
  isis network point-to-point
  isis circuit-type level-1
  ip router isis 1
  no shutdown

interface Ethernet1/6
  description LF6_eth1/1_DOWNLINK
  no switchport
  ip address 10.1.6.1/24
  isis network point-to-point
  isis circuit-type level-1
  ip router isis 1
  no shutdown
```



#### Настройка ISIS:



```
feature isis

router isis 1
  net 49.0001.0001.0001.0001.00
  metric-style transition
  log-adjacency-changes
  address-family ipv4 unicast
    distribute level-1 into level-2 all

interface Ethernet1/1
  isis circuit-type level-2
  ip router isis 1

interface Ethernet1/4
  isis network point-to-point
  isis circuit-type level-1
  ip router isis 1

interface Ethernet1/5
  isis network point-to-point
  isis circuit-type level-1
  ip router isis 1

interface Ethernet1/6
  isis network point-to-point
  isis circuit-type level-1
  ip router isis 1
```



#### Таблица соседств ISIS и таблица маршрутизации ISIS:



![тут скриншот 3](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_03_UNDERLAY_ISIS/screenshots/Screenshot_3.jpg)





### Настройки LEAF на примере LF5 



#### Настройка интерфейсов:



```
interface Ethernet1/1
  description SP1_eth1/5_UPLINK
  no switchport
  ip address 10.1.5.2/24
  isis network point-to-point
  isis circuit-type level-1
  ip router isis 1
  no shutdown

interface Ethernet1/2
  description SP2_eth1/5_UPLINK
  no switchport
  ip address 10.2.5.2/24
  isis network point-to-point
  isis circuit-type level-1
  ip router isis 1
  no shutdown

interface Ethernet1/3
  description LF6_eth1/3_WEST_EAST
  no switchport
  ip address 10.5.6.1/24
  isis network point-to-point
  isis circuit-type level-1
  ip router isis 1
  no shutdown

interface Ethernet1/7
  description NODE10_eth0/0_DOWNLINK
  no switchport
  ip address 10.5.10.1/24
  isis network point-to-point
  isis circuit-type level-1
  ip router isis 1
  no shutdown
```



#### Настройка ISIS:



```
feature isis

router isis 1
  net 49.0001.0005.0005.0005.00
  metric-style transition
  log-adjacency-changes

interface Ethernet1/1
  isis network point-to-point
  isis circuit-type level-1
  ip router isis 1

interface Ethernet1/2
  isis network point-to-point
  isis circuit-type level-1
  ip router isis 1

interface Ethernet1/3
  isis network point-to-point
  isis circuit-type level-1
  ip router isis 1

interface Ethernet1/7
  isis network point-to-point
  isis circuit-type level-1
  ip router isis 1
```



#### Таблица соседств ISIS и таблица маршрутизации ISIS:



![тут скриншот 4](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_03_UNDERLAY_ISIS/screenshots/Screenshot_4.jpg)





#### Проверка доступности NODE между друг другом и доступности внешнего адреса 8.8.8.8:



#### NODE9



![тут скрин 5](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_03_UNDERLAY_ISIS/screenshots/Screenshot_5.jpg)



#### NODE10



![тут скрин 6](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_03_UNDERLAY_ISIS/screenshots/Screenshot_6.jpg)





#### NODE11



![тут скрин 7](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_03_UNDERLAY_ISIS/screenshots/Screenshot_7.jpg)







### Конец.

