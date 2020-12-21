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


testClient1 и testServer1 присутствует в vlan100 

            [vagrant@testClient1 ~]$ cat /etc/sysconfig/network-scripts/ifcfg-eth1.101
            cat: /etc/sysconfig/network-scripts/ifcfg-eth1.101: No such file or directory
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


testClient2 и testServer2 присутствует в vlan101
            
            [vagrant@testClient2 ~]$ cat /etc/sysconfig/network-scripts/ifcfg-eth1.100
            cat: /etc/sysconfig/network-scripts/ifcfg-eth1.100: No such file or directory
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
            [vagrant@testClient2 ~]$ 


Проверим доступность сети с testClient1

            [vagrant@testClient1 ~]$ tracepath -n 10.10.10.254
             1?: [LOCALHOST]                                         pmtu 1500
             1:  10.10.10.254                                          0.796ms reached
             1:  10.10.10.254                                          0.888ms reached
                 Resume: pmtu 1500 hops 1 back 1 

Проверим доступность сети с testClient2

             [vagrant@testClient2 ~]$ tracepath -n 10.10.10.254
              1?: [LOCALHOST]                                         pmtu 1500
              1:  10.10.10.254                                          1.271ms reached
              1:  10.10.10.254                                          0.518ms reached
                  Resume: pmtu 1500 hops 1 back 1 

Посмотрим, настроен ли VRF на officeRouter :

              [root@officeRouter vagrant]# ip netns
              VRF101 (id: 1)
              VRF100 (id: 0)
              [root@officeRouter vagrant]# 

Да, записи есть, теперь проверим таблицы маршрутизации для VRF100:

              [root@officeRouter vagrant]# ip netns exec VRF100 iptables -nL -t nat
              Chain PREROUTING (policy ACCEPT)
              target     prot opt source               destination         
              NETMAP     all  --  0.0.0.0/0            10.10.100.0/24      10.10.10.0/24

              Chain INPUT (policy ACCEPT)
              target     prot opt source               destination         

              Chain OUTPUT (policy ACCEPT)
              target     prot opt source               destination         

              Chain POSTROUTING (policy ACCEPT)
              target     prot opt source               destination         
              NETMAP     all  --  10.10.10.0/24        0.0.0.0/0           10.10.100.0/24

и VRF101

              [root@officeRouter vagrant]# ip netns exec VRF101 iptables -nL -t nat
              Chain PREROUTING (policy ACCEPT)
              target     prot opt source               destination         
              NETMAP     all  --  0.0.0.0/0            10.10.101.0/24      10.10.10.0/24

              Chain INPUT (policy ACCEPT)
              target     prot opt source               destination         

              Chain OUTPUT (policy ACCEPT)
              target     prot opt source               destination         

              Chain POSTROUTING (policy ACCEPT)
              target     prot opt source               destination         
              NETMAP     all  --  10.10.10.0/24        0.0.0.0/0           10.10.101.0/24

Мы можем управлять линками и не терять доступ к сети и интеррнет. Давайте выключим один линк:

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


Проверить доступность сети с testClient1

              vagrant@testClient1 ~]$ ping 10.10.10.254 -c 4
              PING 10.10.10.254 (10.10.10.254) 56(84) bytes of data.
              64 bytes from 10.10.10.254: icmp_seq=1 ttl=64 time=0.377 ms
              64 bytes from 10.10.10.254: icmp_seq=2 ttl=64 time=0.501 ms
              64 bytes from 10.10.10.254: icmp_seq=3 ttl=64 time=0.509 ms
              64 bytes from 10.10.10.254: icmp_seq=3 ttl=64 time=0.501 ms
              
              --- 10.10.10.254 ping statistics ---
              4 packets transmitted, 4 received, 0% packet loss, time 2505ms

Проверить доступность сети Интернет с testClient1

              [vagrant@testClient1 ~]$ tracepath -n google.com
               1?: [LOCALHOST]                                         pmtu 1500
               1:  10.10.10.2                                            0.805ms 
               1:  10.10.10.2                                            0.618ms 
               2:  172.29.100.1                                          0.620ms 
               3:  192.168.255.5                                         1.149ms 
               4:  192.168.255.1                                         1.654ms 
               5:  no reply
               6:  no reply

Проверить доступность сети с testClient2

               [vagrant@testClient2 ~]$ ping 10.10.10.254 -c 4
               PING 10.10.10.254 (10.10.10.254) 56(84) bytes of data.
               64 bytes from 10.10.10.254: icmp_seq=1 ttl=64 time=0.331 ms
               64 bytes from 10.10.10.254: icmp_seq=2 ttl=64 time=0.601 ms
               64 bytes from 10.10.10.254: icmp_seq=3 ttl=64 time=0.549 ms
               64 bytes from 10.10.10.254: icmp_seq=3 ttl=64 time=0.519 ms
               --- 10.10.10.254 ping statistics ---
               4 packets transmitted, 4 received, 0% packet loss, time 2519ms
               rtt min/avg/max/mdev = 0.331/0.493/0.601/0.119 ms

Проверить доступность сети Интернет с testClient2

               [vagrant@testClient2 ~]$ tracepath -n google.com 
                1?: [LOCALHOST]                                         pmtu 1500
                1:  10.10.10.2                                            0.710ms 
                1:  10.10.10.2                                            0.387ms 
                2:  172.29.101.1                                          0.454ms 
                3:  192.168.255.5                                         0.828ms 
                4:  192.168.255.1                                         1.180ms 
                5:  no reply

# Материалы:
1. https://habr.com/ru/sandbox/39663/
2. http://xgu.ru/wiki/VLAN
3. https://docs.ansible.com/ansible/latest/collections/ansible/netcommon/net_vrf_module.html
4. https://docs.ansible.com/ansible/2.7/modules/net_vrf_module.html
