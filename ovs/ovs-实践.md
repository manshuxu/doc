## sdn (software defines network)
看了些相关的资料，这里记录一下自己对sdn的理解，能力有限，如有错误欢迎指正。

sdn软件定义网络，目的是想要利用软件来模拟网络设备，如交换机，路由器之类的。

为什么需要这么做? 一个主要原因是云计算的高速发展，给传统的数据中心带来了更加灵活和复杂的组网需求。

传统网络设备完成数据中心服务器的组网，在此之上，通过sdn来完成虚机和容器之间的连通

## ovs (openvSwitch)
官方的说法：Open vSwitch是一款高质量的多层虚拟交换机，以开源Apache 2许可证授权，非常适合在虚拟机环境中充当2层交换机。支持多种基于Linux的虚拟化技术，包括Xen / XenServer、KVM和VirtualBox。
支持Open Flow协议，所以可以很方便的通过编程实现大规模网络的自动化，被大量运用于SDN网络中。

![](https://github.com/manshuxu/doc/blob/master/ovs/ovs-framework.png)

架构和原理之类的文章很多，这里就不在一一阐述，本文以实践为主。

## 安装

```sh
# 安装docker
yum install -y docker-1.13.1

# 预安装
yum -y install wget openssl-devel gcc make python-devel openssl-devel kernel-devel graphviz kernel-debug-devel autoconf automake rpm-build redhat-rpm-config libtool python-twisted-core python-zope-interface PyQt4 desktop-file-utils libcap-ng-devel groff checkpolicy selinux-policy-devel

# 安装open vswitch:
yum install -y  openvswitch-2.8.2-1.el7.x86_64  # 此版本包含ovs-docker
systemctl start openvswitch.service
systemctl is-active openvswitch
systemctl enable openvswitch
```

### 实验拓扑

![](https://github.com/manshuxu/doc/blob/master/ovs/ovs.png)

此图符合本文所有实验

## ovs单机连通性

创建容器, 设置net=none可以防止docker0默认网桥影响连通性测试
```sh
docker run -itd --name con6 --net=none ubuntu:14.04 /bin/bash
docker run -itd --name con7 --net=none ubuntu:14.04 /bin/bash
docker run -itd --name con8 --net=none ubuntu:14.04 /bin/bash
```
创建网桥
```sh
ovs-vsctl add-br ovs0
```
使用ovs-docker给容器添加网卡，并挂到ovs0网桥上
```sh
ovs-docker add-port ovs0 eth0 con6 --ipaddress=192.168.1.2/24
ovs-docker add-port ovs0 eth0 con7 --ipaddress=192.168.1.3/24
ovs-docker add-port ovs0 eth0 con8 --ipaddress=192.168.1.4/24
```
查看网桥
```sh
[root@controller /]# ovs-vsctl show
21e4d4c5-cadd-4dac-b025-c20b8108ad09
    Bridge "ovs0"
        Port "b167e3dcf8db4_l"
            Interface "b167e3dcf8db4_l"
        Port "f1c0a9d0994d4_l"
            Interface "f1c0a9d0994d4_l"
        Port "121c6b2f221c4_l"
            Interface "121c6b2f221c4_l"
        Port "ovs0"
            Interface "ovs0"
                type: internal
    ovs_version: "2.8.2"
```
测试连通性
```sh
[root@controller /]# docker exec -it con8 sh
# ping 192.168.1.2      
PING 192.168.1.2 (192.168.1.2) 56(84) bytes of data.
64 bytes from 192.168.1.2: icmp_seq=1 ttl=64 time=0.886 ms
^C
--- 192.168.1.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.886/0.886/0.886/0.000 ms
# 
# ping 192.168.1.3  
PING 192.168.1.3 (192.168.1.3) 56(84) bytes of data.
64 bytes from 192.168.1.3: icmp_seq=1 ttl=64 time=0.712 ms
^C
--- 192.168.1.3 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.712/0.712/0.712/0.000 ms
# 
```

### 设置VLAN tag
查看网桥
```sh
[root@controller /]# ovs-vsctl show
21e4d4c5-cadd-4dac-b025-c20b8108ad09
    Bridge "ovs0"
        Port "b167e3dcf8db4_l"
            Interface "b167e3dcf8db4_l"
        Port "f1c0a9d0994d4_l"
            Interface "f1c0a9d0994d4_l"
        Port "121c6b2f221c4_l"
            Interface "121c6b2f221c4_l"
        Port "ovs0"
            Interface "ovs0"
                type: internal
    ovs_version: "2.8.2"
```

查看interface
```sh
[root@controller /]# ovs-vsctl list interface f1c0a9d0994d4_l
_uuid               : cf400e7c-d2d6-4e0a-ad02-663dd63d1751
admin_state         : up
duplex              : full
error               : []
external_ids        : {container_id="con6", container_iface="eth0"}
ifindex             : 239
ingress_policing_burst: 0
ingress_policing_rate: 0
lacp_current        : []
link_resets         : 1
link_speed          : 10000000000
link_state          : up
mac_in_use          : "96:91:0a:c9:02:d6"
mtu                 : 1500
mtu_request         : []
name                : "f1c0a9d0994d4_l"
ofport              : 3
other_config        : {}
statistics          : {collisions=0, rx_bytes=1328, rx_crc_err=0, rx_dropped=0, rx_errors=0, rx_frame_err=0, rx_over_err=0, rx_packets=18, tx_bytes=3032, tx_dropped=0, tx_errors=0, tx_packets=40}
status              : {driver_name=veth, driver_version="1.0", firmware_version=""}
type                : ""
```
设置vlan tag
```sh
ovs-vsctl set port   f1c0a9d0994d4_l tag=100  //con6
ovs-vsctl set port   b167e3dcf8db4_l tag=100  //con8
ovs-vsctl set port   121c6b2f221c4_l tag=200  //con7
```

测试连通性
```sh
[root@controller /]# docker exec -it con8 sh
# 
# ping 192.168.1.2 -c 3
PING 192.168.1.2 (192.168.1.2) 56(84) bytes of data.
64 bytes from 192.168.1.2: icmp_seq=1 ttl=64 time=0.413 ms
64 bytes from 192.168.1.2: icmp_seq=2 ttl=64 time=0.061 ms
64 bytes from 192.168.1.2: icmp_seq=3 ttl=64 time=0.057 ms
--- 192.168.1.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2044ms
rtt min/avg/max/mdev = 0.057/0.177/0.413/0.166 ms
# 
# ping 192.168.1.3 -c 3
PING 192.168.1.3 (192.168.1.3) 56(84) bytes of data.
From 192.168.1.4 icmp_seq=1 Destination Host Unreachable
From 192.168.1.4 icmp_seq=2 Destination Host Unreachable
--- 192.168.1.3 ping statistics ---
3 packets transmitted, 0 received, +3 errors, 100% packet loss, time 2068ms
pipe 3
# 
```

## 跨主机连通性
### 环境
##### host1 172.29.101.123  
```
网桥:  ovs0      

容器:    
con6  192.168.1.2   
con7  192.168.1.3   
con8  192.168.1.4   
```
创建方式依上

##### host2 172.29.101.82
```
网桥: ovs1

容器: con11
```
准备环境
```sh
创建网桥
ovs-vsctl add-br ovs1

创建容器
docker run -itd --name con11 --net=none ubuntu:14.04 /bin/bash

挂到ovs0网桥
ovs-docker add-port ovs1 eth0 con11 --ipaddress=192.168.1.6/24
```

查看网桥ovs1
```sh
[root@compute82 /]# ovs-vsctl show
380ce027-8edf-4844-8e89-a6b9c1adaff3
    Bridge "ovs1"
        Port "0384251973e64_l"
            Interface "0384251973e64_l"
        Port "ovs1"
            Interface "ovs1"
                type: internal
    ovs_version: "2.8.2"
```

### 设置vxlan
在host1上
```sh
[root@controller /]# ovs-vsctl add-port ovs0 vxlan1 -- set interface vxlan1 type=vxlan options:remote_ip=172.29.101.82 options:key=flow
[root@controller /]# 
[root@controller /]# ovs-vsctl show
21e4d4c5-cadd-4dac-b025-c20b8108ad09
    Bridge "ovs0"
        Port "b167e3dcf8db4_l"
            tag: 100
            Interface "b167e3dcf8db4_l"
        Port "f1c0a9d0994d4_l"
            tag: 100
            Interface "f1c0a9d0994d4_l"
        Port "121c6b2f221c4_l"
            tag: 200
            Interface "121c6b2f221c4_l"
        Port "ovs0"
            Interface "ovs0"
                type: internal
        Port "vxlan1"
            Interface "vxlan1"
                type: vxlan
                options: {key=flow, remote_ip="172.29.101.82"}
    ovs_version: "2.8.2"
```
在host2上
```sh
[root@compute82 /]# ovs-vsctl add-port ovs1 vxlan1 -- set interface vxlan1 type=vxlan options:remote_ip=172.29.101.123 options:key=flow
[root@compute82 /]# 
[root@compute82 /]# ovs-vsctl show
380ce027-8edf-4844-8e89-a6b9c1adaff3
    Bridge "ovs1"
        Port "0384251973e64_l"
            Interface "0384251973e64_l"
        Port "vxlan1"
            Interface "vxlan1"
                type: vxlan
                options: {key=flow, remote_ip="172.29.101.123"}
        Port "ovs1"
            Interface "ovs1"
                type: internal
    ovs_version: "2.8.2"
```


### 设置vlan tag 

```sh
ovs-vsctl set port 0384251973e64_l tag=100
```

### 连通性测试
```sh
[root@compute82 /]# docker exec -ti con11 bash
root@c82da61bf925:/# ping 192.168.1.2
PING 192.168.1.2 (192.168.1.2) 56(84) bytes of data.
64 bytes from 192.168.1.2: icmp_seq=1 ttl=64 time=0.161 ms
64 bytes from 192.168.1.2: icmp_seq=2 ttl=64 time=0.206 ms
^C
--- 192.168.1.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1000ms
root@c82da61bf925:/# 
root@c82da61bf925:/# ping 192.168.1.3
PING 192.168.1.3 (192.168.1.3) 56(84) bytes of data.
^C
--- 192.168.1.3 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2027ms
root@c82da61bf925:/# 
root@c82da61bf925:/# exit
```

### 结论
vxlan只能连通两台机器的ovs上同一个网段的容器，无法连通ovs上不同网段的容器。如果需要连通不同网段的容器，接下来我们尝试通过ovs的流表来解决这个问题。

# OpenFlow
### flow table
支持openflow的交换机中可能包含多个flow table。每个flow table包含多条规则，每条规则包含匹配条件和执行动作。flow table中的每条规则有优先级，优先级高的优先匹配，匹配到规则以后，执行action，如果匹配失败，按优先级高低，继续匹配下一条。如果都不匹配，每张表会有默认的动作，一般为drop或者转给下一张流表。

### 实践
#### 环境
 host1 172.29.101.123  
```sh
网桥:  ovs0      

容器:    
con6  192.168.1.2     tag=100
con7  192.168.1.3     tag=100
```
 host2 172.29.101.82
```sh
网桥: ovs1

容器:  
con9:  192.168.2.2    tag=100
con10：192.168.2.3    tag=100
con11: 192.168.1.5    tag=100
```
### 查看默认流表
在host1上查看默认流表
```sh
[root@controller msxu]# ovs-ofctl dump-flows ovs0
 cookie=0x0, duration=27858.050s, table=0, n_packets=5253660876, n_bytes=371729202788, priority=0 actions=NORMAL
```
在容器con6中ping con7，网络连通
```sh
[root@controller /]# docker exec -ti con6 bash
root@9ccc5c5664f9:/# 
root@9ccc5c5664f9:/# ping 192.168.1.3
PING 192.168.1.3 (192.168.1.3) 56(84) bytes of data.
64 bytes from 192.168.1.3: icmp_seq=1 ttl=64 time=0.613 ms
64 bytes from 192.168.1.3: icmp_seq=2 ttl=64 time=0.066 ms
--- 192.168.1.3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1058ms
rtt min/avg/max/mdev = 0.066/0.339/0.613/0.274 ms
root@9ccc5c5664f9:/# 
```
删除默认流表
```sh
[root@controller /]# ovs-ofctl del-flows ovs0
[root@controller /]# 
[root@controller /]# ovs-ofctl dump-flows ovs0
[root@controller /]# 
```
测试网络连通性，发现网络已经不通
```sh
[root@controller /]# docker exec -ti con6 bash
root@9ccc5c5664f9:/# 
root@9ccc5c5664f9:/# ping 192.168.1.3
PING 192.168.1.3 (192.168.1.3) 56(84) bytes of data.
^C
--- 192.168.1.3 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1025ms
root@9ccc5c5664f9:/# 
```

### 添加流表
如果要con6和con7能够通信，需要建立规则，让ovs转发对应的数据

查看con6和con7在ovs上的网络端口
```sh
[root@controller /]# ovs-vsctl show
21e4d4c5-cadd-4dac-b025-c20b8108ad09
    Bridge "ovs0"
        Port "f1c0a9d0994d4_l"
            tag: 100
            Interface "f1c0a9d0994d4_l"
        Port "121c6b2f221c4_l"
            tag: 100
            Interface "121c6b2f221c4_l"
        Port "ovs0"
            Interface "ovs0"
                type: internal
        Port "vxlan1"
            Interface "vxlan1"
                type: vxlan
                options: {key=flow, remote_ip="172.29.101.82"}
    ovs_version: "2.8.2"
[root@controller /]# ovs-vsctl list interface f1c0a9d0994d4_l |grep ofport
ofport              : 3
ofport_request      : []
[root@controller /]# 
[root@controller /]# ovs-vsctl list interface 121c6b2f221c4_l |grep ofport
ofport              : 4
ofport_request      : []
```
添加规则：
```sh
[root@controller /]#ovs-ofctl add-flow ovs0 "priority=1,in_port=3,actions=output:4"
[root@controller /]#ovs-ofctl add-flow ovs0 "priority=2,in_port=4,actions=output:3"
[root@controller /]# ovs-ofctl dump-flows ovs0
 cookie=0x0, duration=60.440s, table=0, n_packets=0, n_bytes=0, priority=1,in_port="f1c0a9d0994d4_l" actions=output:"121c6b2f221c4_l"
 cookie=0x0, duration=50.791s, table=0, n_packets=0, n_bytes=0, priority=1,in_port="121c6b2f221c4_l" actions=output:"f1c0a9d0994d4_l"
[root@controller /]#
```
测试连通性：con6和con7已通
```sh
[root@controller msxu]# docker exec -ti con6 bash
root@9ccc5c5664f9:/# ping 192.168.1.3
PING 192.168.1.3 (192.168.1.3) 56(84) bytes of data.
64 bytes from 192.168.1.3: icmp_seq=1 ttl=64 time=0.924 ms
64 bytes from 192.168.1.3: icmp_seq=2 ttl=64 time=0.058 ms
^C
--- 192.168.1.3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1057ms
rtt min/avg/max/mdev = 0.058/0.491/0.924/0.433 ms
root@9ccc5c5664f9:/# 
```
设置一条优先级高的规则：
```sh
[root@controller /]# ovs-ofctl add-flow ovs0 "priority=2,in_port=4,actions=drop"
[root@controller /]# 
[root@controller /]# docker exec -ti con6 bash
root@9ccc5c5664f9:/# 
root@9ccc5c5664f9:/# ping  192.168.1.3
PING 192.168.1.3 (192.168.1.3) 56(84) bytes of data.
^C
--- 192.168.1.3 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2087ms
root@9ccc5c5664f9:/# 
root@9ccc5c5664f9:/# 
```
流表中的规则是有优先级的，priority数值越大，优先级越高。流表中，优先级高的优先匹配，并执行匹配规则的actions。如果不匹配，继续匹配优先级低的下一条。

### 跨网段连通
在上一个vxlan的实践中，通过设置vxlan可以打通两个机器上的ovs，但我们提到两个机器ovs上的容器得在同一个网段上才能通信。

在ip为192.168.2.2的con9上ping另一台机上的con6 192.168.1.2
```sh
[root@compute82 /]# docker exec -ti con9 bash
root@b55602aad0ac:/# 
root@b55602aad0ac:/# ping 192.168.1.2
connect: Network is unreachable
root@b55602aad0ac:/# 
```
#### 添加流表规则：  
在host1上：
```sh
[root@controller /]# ovs-ofctl add-flow ovs0 "priority=4,in_port=6,actions=output:3"
[root@controller /]# 
[root@controller /]# ovs-ofctl add-flow ovs0 "priority=4,in_port=3,actions=output:6"
[root@controller /]# ovs-ofctl dump-flows ovs0
 cookie=0x0, duration=3228.737s, table=0, n_packets=7, n_bytes=490, priority=1,in_port="f1c0a9d0994d4_l" actions=output:"121c6b2f221c4_l"
 cookie=0x0, duration=3215.544s, table=0, n_packets=0, n_bytes=0, priority=1,in_port="121c6b2f221c4_l" actions=output:"f1c0a9d0994d4_l"
 cookie=0x0, duration=3168.297s, table=0, n_packets=9, n_bytes=546, priority=2,in_port="121c6b2f221c4_l" actions=drop
 cookie=0x0, duration=12.024s, table=0, n_packets=0, n_bytes=0, priority=4,in_port=vxlan1 actions=output:"f1c0a9d0994d4_l"
 cookie=0x0, duration=3.168s, table=0, n_packets=0, n_bytes=0, priority=4,in_port="f1c0a9d0994d4_l" actions=output:vxlan1

```
在host2上
```sh
[root@compute82 /]# ovs-ofctl add-flow ovs1 "priority=1,in_port=1,actions=output:6"
[root@compute82 /]# 
[root@compute82 /]# ovs-ofctl add-flow ovs1 "priority=1,in_port=6,actions=output:1"
[root@compute82 /]# ovs-ofctl dump-flows ovs1
 cookie=0x0, duration=1076.522s, table=0, n_packets=27, n_bytes=1134, priority=1,in_port="0384251973e64_l" actions=output:vxlan1
 cookie=0x0, duration=936.403s, table=0, n_packets=0, n_bytes=0, priority=1,in_port=vxlan1 actions=output:"0384251973e64_l"
 cookie=0x0, duration=70205.443s, table=0, n_packets=7325, n_bytes=740137, priority=0 actions=NORMAL

```
#### 测试连通性
在host2 con9上ping 192.168.1.2
```sh
[root@compute82 /]# docker exec -ti con9 bash
root@b55602aad0ac:/# 
root@b55602aad0ac:/# ping 192.168.1.2
connect: Network is unreachable
root@b55602aad0ac:/# 
```
发现网络并不通，查看发现路由规则有问题，添加默认路由规则，注意这里需要已privileged权限进入容器
```sh
[root@compute82 /]# docker exec --privileged -ti con9 bash
root@b55602aad0ac:/# 
root@b55602aad0ac:/# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
192.168.2.0     0.0.0.0         255.255.255.0   U     0      0        0 eth0
root@b55602aad0ac:/# route add default dev eth0
root@b55602aad0ac:/# 
root@b55602aad0ac:/# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         0.0.0.0         0.0.0.0         U     0      0        0 eth0
192.168.2.0     0.0.0.0         255.255.255.0   U     0      0        0 eth0
root@b55602aad0ac:/# 
```
在host1和host2的容器中都添加好路由规则后，测试连通性
```sh
[root@compute82 /]# docker exec --privileged -ti con9 bash
root@b55602aad0ac:/# 
root@b55602aad0ac:/# ping 192.168.1.2
PING 192.168.1.2 (192.168.1.2) 56(84) bytes of data.
64 bytes from 192.168.1.2: icmp_seq=1 ttl=64 time=1.16 ms
64 bytes from 192.168.1.2: icmp_seq=2 ttl=64 time=0.314 ms
^C
--- 192.168.1.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 0.314/0.739/1.165/0.426 ms
```
已成功通过ovs，vxlan打通两台机器上不同网段容器
