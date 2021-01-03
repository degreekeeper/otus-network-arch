# UNDERLAY. OSPF



### Топология



![тут скрин 1](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_02_UNDERLAY_OSPF/screenshots/Screenshot_1.jpg)





#### Цель: Настроить OSPF для Underlay сети

​                                

1. Настроить OSPF в Underlay сети, для IP связанности между всеми устройствами NXOS
2. План работы, адресное пространство, схема сети, настройки - зафиксированы в документации 



#### Общие концепции:

1. router-id выбирается по принципеу **10.255.0.xxx** - где **xxx** это цифра из hostname оборудования

2. Пытаемся максимально унифицировать конфиг. Для этого используем одинаковые номера портов под одинаковые включения (хосты, LAEF, SPINE, CORE). Используем типовые description. Используем типовые конфигурации протоколов.

3. На CORE-ROUTER будет настроен интерфейс ***loopback8888*** для имитации внешней сети с адресом 8.8.8.8/32. Данная сеть будет являться маршрутом по умолчанию, но анонсировать отдельно по OSPF не будет.

4. В сети полностью отсутствуют широковещательные сегменты L2. Поэтому чтобы избежать выборов DR/BDR все интерфейсы участвующие в работе OSPF будут настроены в режиме P2P с помощью команды ***ip ospf network point-to-point***.

5. Для уменьшения таблиц маршрутизации ЦОД будет разделен на AREA. 

   5.1 Backbone AREA 0 будет находиться между CORE-ROUTER и всеми интерфейсами SPINE, смотрящими в сторону CORE-ROUTER. В ней не будет никаких ограничений, будут приниматься и анонсироваться все маршруты, кроме анонсирования внешнего 8.8.8.8/32 нижестояющим роутерам.

   5.2 Каждый POD будет находиться в собственной AREA, т.е. в неё будут включены интерфейсы SPINE, смотрящие в сторону LEAF. И все интерфейсы на LEAF. Данные AREA будут настроены как ***totally stub***. C помощью команды
     ***area 0.0.0.1 stub no-summary*** на всех SPINE. И команды ***area 0.0.0.1 stub*** на всех LEAF. После чего на LEAF не будут доходить LSA 3, 4, 5, чем мы уменьшим таблицу маршрутизации. В таблице маршрутизации будет только маршрут по-умолчанию и внутренние сети AREA из LSA 1, 2.

6.  На LEAF все интерфейсы по умолчанию будут настроены в режиме ***passive*** c помощью команды ***passive-interface default***. Это сделано чтобы уменьшит размер конфига, т.к. большинство интерфейсов будет использоваться для включения хостов. На интерфейсах аплинках в сторону SPINE LSU пакеты будут разрешены с помощью команды ***no ip ospf passive-interface***.



#### Настройка CORE-ROUTER:







##### Настройки интефрейсов

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
 ip ospf network point-to-point
 ip ospf 1 area 0
!
interface Ethernet0/2
 description SP2_eth1/1_DOWNLINK
 ip address 10.0.2.1 255.255.255.0
 ip ospf network point-to-point
 ip ospf 1 area 0
!
interface Ethernet0/3
 description SP3_eth1/1_DOWNLINK
 ip address 10.0.3.1 255.255.255.0
 ip ospf network point-to-point
 ip ospf 1 area 0
!
```



##### Настройка OSPF и маршрута по умолчанию во внешние сети:



```
!
router ospf 1
 router-id 10.255.0.0
 default-information originate
!
ip forward-protocol nd
!
ip route 0.0.0.0 0.0.0.0 Loopback8888
!
```



##### Пример вывода таблицы маршрутизации на CORE-ROUTER



![тут скрин 5](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_02_UNDERLAY_OSPF/screenshots/Screenshot_5_core_route.jpg)



#### Настройка на SPINE. Для примера взяли SP1





##### Настройки на интерфейсах



```
interface Ethernet1/1
  description CORE_eth0/1_UPLINK
  no switchport
  ip address 10.0.1.2/24
  ip ospf network point-to-point
  ip router ospf 1 area 0.0.0.0
  no shutdown


interface Ethernet1/4
  description LF4_eth1/1_DOWNLINK
  no switchport
  ip address 10.1.4.1/24
  ip ospf network point-to-point
  ip router ospf 1 area 0.0.0.1
  no shutdown

interface Ethernet1/5
  description LF5_eth1/5_DOWNLINK
  no switchport
  ip address 10.1.5.1/24
  ip ospf network point-to-point
  ip router ospf 1 area 0.0.0.1
  no shutdown

interface Ethernet1/6
  description LF6_eth1/1_DOWNLINK
  no switchport
  ip address 10.1.6.1/24
  ip ospf network point-to-point
  ip router ospf 1 area 0.0.0.1
  no shutdown
```



##### Настройки OSPF



```
router ospf 1
  router-id 10.255.0.1
  area 0.0.0.1 stub no-summary
```





##### Пример таблицы маршрутизации на SP1



![тут скрин 8](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_02_UNDERLAY_OSPF/screenshots/Screenshot_8_spine_route.jpg)





#### Настройки на LEAF. Для примера берем LF4



##### Настройки на интерфейсах



```
interface Ethernet1/1
  description SP1_eth1/4_UPLINK
  no switchport
  ip address 10.1.4.2/24
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf 1 area 0.0.0.1
  no shutdown

interface Ethernet1/2
  description SP2_eth1/4_UPLINK
  no switchport
  ip address 10.2.4.2/24
  ip ospf network point-to-point
  no ip ospf passive-interface
  ip router ospf 1 area 0.0.0.1
  no shutdown

interface Ethernet1/7
  description NODE9_eth0/0_DOWNLINK
  no switchport
  ip address 10.4.9.1/24
  ip router ospf 1 area 0.0.0.1
  no shutdown
```



##### Настройки OSPF



```
router ospf 1
  router-id 10.255.0.4
  area 0.0.0.1 stub
  passive-interface default
```




##### Таблица маршрутизации на LF4 и LF7. 



Т.к. в POD2 у нас только один LEAF, на него по OSPF будет приходить только один маршрут по-умолчанию.



##### LF4



![тут скрин 6](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_02_UNDERLAY_OSPF/screenshots/Screenshot_6_leaf_route.jpg)



##### LF7


![тут скрин 7](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_02_UNDERLAY_OSPF/screenshots/Screenshot_7_leaf7_route_pod2.jpg)



#### Настройки на NODE(хостах). Для примера возьмем NODE9



```
!
interface Ethernet0/0
 description LF4_eth1/7_UPLINK
 ip address 10.4.9.2 255.255.255.0
!
ip route 0.0.0.0 0.0.0.0 Ethernet0/0 10.4.9.1
!
```



#### Проверим сетевую связанность между NODE, а так же до внешнего маршрута 8.8.8.8/32.



##### Проверка с NODE9



![тут скрин 2](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_02_UNDERLAY_OSPF/screenshots/Screenshot_2_pingnode9.jpg)



##### Проверка с NODE10



![тут скрин 3](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_02_UNDERLAY_OSPF/screenshots/Screenshot_3_ping_node10.jpg)



##### Проверка с NODE11


![тут скрин 4](https://github.com/degreekeeper/otus-network-arch/blob/main/LAB_02_UNDERLAY_OSPF/screenshots/Screenshot_4_ping_node11.jpg)







#### Конец.