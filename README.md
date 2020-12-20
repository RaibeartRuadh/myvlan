# Строим бонды и вланы

в Office1 в тестовой подсети появляется сервера с доп интерфесами и адресами
в internal сети testLAN
- testClient1 - 10.10.10.254
- testClient2 - 10.10.10.254
- testServer1- 10.10.10.1
- testServer2- 10.10.10.1

равести вланами
testClient1 <-> testServer1
testClient2 <-> testServer2

между centralRouter и inetRouter
"пробросить" 2 линка (общая inernal сеть) и объединить их в бонд
проверить работу c отключением интерфейсов

для сдачи - вагрант файл с требуемой конфигурацией
Разворачиваться конфигурация должна через ансибл


# Проверяем:

Поднимаем стенд:

    $ vagrant up

Подключаемся к центральному роутеру, повышаем привелегии до sudo и смотрим состояние наших агрегированных линков через демона teamd

    $ vagrant ssh centralRouter
    $ sudo -i
    $ teamdctl team0 state


    [root@centralRouter ~]# teamdctl team0 state
    setup:
      runner: loadbalance
    ports:
      eth1
        link watches:
          link summary: up
          instance[link_watch_0]:
            name: ethtool
            link: up
            down count: 0
      eth2
        link watches:
          link summary: up
          instance[link_watch_0]:
            name: ethtool
            link: up
            down count: 0

Выполняем трассировку выхода в интернет:

    [root@centralRouter ~]# tracepath -n google.com
     1?: [LOCALHOST]                                         pmtu 1500
     1:  192.168.255.1                                         0.496ms 
     1:  192.168.255.1                                         0.312ms 
     2:  no reply


Отключим один из линков:

    [root@centralRouter ~]# ip link set dev eth1 down
    [root@centralRouter ~]# teamdctl team0 state
    setup:
      runner: loadbalance
    ports:
      eth1
        link watches:
          link summary: up
          instance[link_watch_0]:
            name: ethtool
            link: up
            down count: 0
      eth2
        link watches:
          link summary: down
          instance[link_watch_0]:
            name: ethtool
            link: down
            down count: 1





    [vagrant@testClient1 ~]$ cat /etc/sysconfig/network-scripts/ifcfg-eth1.100 
    VLAN=yes
    TYPE=Vlan
    PHYSDEV=eth1
    VLAN_ID=100
    BOOTPROTO=none
    IPADDR=10.10.10.1
    PREFIX=24
    DEFROUTE=yes
    IPV4_FAILURE_FATAL=no
    IPV6INIT=no
    NAME=eth1.100
    DEVICE=eth1.100
    ONBOOT=yes
    [vagrant@testClient1 ~]$ 



    [vagrant@testClient2 ~]$ cat /etc/sysconfig/network-scripts/ifcfg-eth1.101
    VLAN=yes
    TYPE=Vlan
    PHYSDEV=eth1
    VLAN_ID=101
    BOOTPROTO=none
    IPADDR=10.10.10.1
    PREFIX=24
    DEFROUTE=yes
    IPV4_FAILURE_FATAL=no
    IPV6INIT=no
    NAME=eth1.101
    DEVICE=eth1.101
    ONBOOT=yes










