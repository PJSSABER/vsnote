# Network base point

## RPS
Receive Packet Steering (RPS) is a feature in Linux networking that allows the distribution of incoming network traffic across multiple CPUs.

When a network interface card (NIC) receives a packet, it typically interrupts the CPU to process the packet. If a single CPU receives too many interrupts, it can become a bottleneck and slow down network performance. RPS aims to solve this problem by distributing the processing of incoming packets across multiple CPUs, which can increase throughput and reduce latency.

With RPS enabled, the NIC queues the incoming packets in hardware, and then the kernel assigns each packet to a processing CPU based on a hashing algorithm that takes into account various packet fields, such as the source and destination IP addresses and ports. The CPU that receives the interrupt then processes the packet, which can include forwarding it to another network device or processing it locally.
## 隧道技术

## vlan
  
## vxlan  
    是对传统VLAN协议的一种扩展。VXLAN的特点是将L2的以太帧封装到UDP报文（即L2 over L4）中，并在L3网络中传输。在Linux中也设计为一个网络设备
    ```shell
        ip link add name type vxlan \
        id 42 \  # 通过id进行配对
        remote xxx
    ```
## bridge  
    虚拟的网络设备，类似一个交换机. Linux开启ip forward(路由转发)后，赋予网桥IP地址(这会让包通过内核网络栈)，可以让网桥具有路由的功能
    ```shell
        # show bridges
        brctl show
    ```
## veth-pair  
  虚拟网络设备，成对出现，会从任意一端进入，另一端传出

网桥模式

## container to container
1. 连接同一个网桥的container在同一个2层网络内，开启ip_forward即可通信

## contianer to outside
1. 网桥配置ip, 这样网桥就成为一个L2&L3虚拟网络设备(主机能ping通)
2. container内部设置网桥为默认网关
3. SNAT，使物理网卡转发的包的source-ip从网桥ip映射为物理网卡ip, 接受包时进行反映射

## outside to container
1. DNAT 暴露端口，将物理网卡ip转换为container-ip
在主机上开一个端口，将 目的ip+端口 映射为局域网ip+port

# Container Network

Docker 7 model
##  default bridge

- both level 2 && 3 network v-device, connecting two side of the bridge.
- 给网桥设置IP地址后，就能像进行三层网络的转发
    ![](/images/image.png.png)
- in this mod, each container will use a Vinterface to connect to default docker bridge.

- Also use DHCP to allocate ip to each container. containers connect to each other.
    ![](/images/2023-01-19-13-43-47.png)

- 注意这里多个container在使用同一网桥的时候是在一个网段的, 网桥与container内部的虚拟网卡通过veth-pair设备连接

```shell
root@r06u24-dcai-spr-cd:/home/raspadmin# ip addr show

5: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:6e:de:ac:f8 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 brd 172.18.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:6eff:fede:acf8/64 scope link
       valid_lft forever preferred_lft forever
...

78: veth1cea035@if77: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default
    link/ether f2:aa:cb:dd:bd:fd brd ff:ff:ff:ff:ff:ff link-netnsid 6
    inet6 fe80::f0aa:cbff:fedd:bdfd/64 scope link
       valid_lft forever preferred_lft forever

root@r06u24-dcai-spr-cd:/home/raspadmin# bridge link
78: veth1cea035@if77: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master docker0 state forwarding priority 32 cost 2
```
可以看到 bridge 有虚拟的veth以太设备（二层），又有ip DHCP(三层)

- container 间可以互相ping
- container 可以ping 外部, how? 网桥 NAT协议

```shell
# container 内部

/ # ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
77: eth0@if78: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    link/ether 02:42:ac:12:00:04 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.4/16 brd 172.18.255.255 scope global eth0
       valid_lft forever preferred_lft forever
/ # ip route
default via 172.18.0.1 dev eth0        # 默认的网关是 docker0 网桥
172.18.0.0/16 dev eth0 scope link  src 172.18.0.4

# 在host上抓取包
root@r06u24-dcai-spr-cd:/home/raspadmin# tcpdump -i docker0
15:16:06.421076 IP 172.18.0.4 > 10.166.33.27: ICMP echo request, id 18, seq 4, length 64
15:16:06.625748 IP 10.166.33.27 > 172.18.0.4: ICMP echo reply, id 18, seq 4, length 64
15:16:07.421277 IP 172.18.0.4 > 10.166.33.27: ICMP echo request, id 18, seq 5, length 64
15:16:07.471314 ARP, Request who-has cupcon03.amr.corp.intel.com tell 172.18.0.4, length 28
15:16:07.471412 ARP, Reply cupcon03.amr.corp.intel.com is-at 02:42:6e:de:ac:f8 (oui Unknown), length 28
15:16:07.625672 IP 10.166.33.27 > 172.18.0.4: ICMP echo reply, id 18, seq 5, length 64
15:16:07.727324 ARP, Request who-has 172.18.0.4 tell cupcon03.amr.corp.intel.com, length 28
...
```

- 要让外部访问可以进入docker，需要端口暴露 `docker run -p 80:80 busybox `

- 可以直接用hostname互相ping， 有hosts解析
##  user defined bridge

```shell
docker network create asgard

docker run -itd --rm --network newasgard busybox
```
不使用dockers的默认bridge， 而创建自己的Bridge.
各个子bridge 下的container下 互不相通

##  Host
using host networkding directly
不需要专门暴露端口
没有任何网络隔离

##  Mac Vlan
不需要专门暴露端口

要与外网相连 ip需要与网关在一个网段，如同一个真实网卡一样

bridge 模式

    每个container会获取一个虚拟mac地址，与host的网关直连
    ![](/images/2023-01-19-10-57-27.png)

```shell
 docker network create -d macvlan \   # -d 指定driver模式
 --subnet 172.17.28.0/24 \  # subnet ip address
 --gateway 172.17.28.1 \
 -o parent=ens259f0 \    # parameter specify NIC
 newasgard   # network name

 docker network create -d macvlan --subnet 172.17.28.0/24 --gateway 172.17.28.1 -o parent=ens259f0  newasgard

 docker run -itd --rm --network newasgard --ip 172.17.28.8 --name env1 busybox 

# container

/ # ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
79: eth0@if2: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
    link/ether 02:42:ac:11:1c:08 brd ff:ff:ff:ff:ff:ff
    inet 172.17.28.8/24 brd 172.17.28.255 scope global eth0
       valid_lft forever preferred_lft forever

```
cons: 1.需要混雜模式（英語：promiscuous mode) 2. NO DHCP 需要自己指定IP

## IP Vlan l2

与4相同， 但与host share一个mac地址
要与外网相连 ip需要与网关在一个网段
```shell
docker network create -d ipvlan \   # -d 指定driver模式
 --subnet 172.17.28.0/24 \  # subnet ip address
 --gateway 172.17.28.1 \
 -o parent=ens259f0 \    # parameter specify NIC
 newasgard   # network name

```
## IP Vlan l3
完全在三层不需要 ARP
利用3层网络，让host将数据转发给container 需要修改HOST上的routing table
    ![](/images/2023-01-19-10-58-17.png)

```shell
 docker network create -d ipvlan \   # -d 指定driver模式
 --subnet 192.168.94.0/24 --subnet 192.168.95.0/24 \  # subnet ip address
 -o parent=ens802 -o ipvlan_mode=l3 \    # parameter specify NIC
 newasgard   # network name

docker run -itd --rm --network newasgard --ip 192.168.94.8 --name env1 busybox 

docker run -itd --rm --network newasgard --ip 192.168.95.7 --name env2 busybox
```

Host网卡 （ -o parent 指定）将指定的网卡作为了一个三层路由器
如上设置时，

- env1, env2 都不能ping通外部环境如 google.com等，因为包会发给host-nic转发出去，但是外部的包却传不回来（找不到container ip地址）
- 如果需要ping通外部环境，需要设置 host的网关， 令 ipvlanl3 的网段的next hop也跳向 host
- env1 和 env2 可以互相ping，host网卡负责三层的转发

## flannel 
![](/images/2023-01-20-11-34-43.png)

## calic
同一局域网下可以使用纯的3层网络，不需要VXLAN进行拆解包，更加高效
![](/images/2023-01-20-11-45-48.png)
好像连网桥都不需要
不同局域网的时候如何处理：使用隧道设备

# useful command

```shell
docker inspect

# show docker network
docker network ls

# show bridge link
bridge show

bridge link

# create docker bridge
docker network create -d macvlan[...] -o [parameter] <network_name>

# show route rule
ip route [list table]
route
# show namespace
ip netns 

# exec in specific net namespace
ip netns exec namespace [your command]

# useful tracing tool
tcpdump

traceroute
```

enp137s0f4u2: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.239.83.142  netmask 255.255.255.0  broadcast 10.239.83.255
        inet6 fe80::74b9:deff:fe34:8018  prefixlen 64  scopeid 0x20<link>
        ether 76:b9:de:34:80:18  txqueuelen 1000  (Ethernet)
        RX packets 18603  bytes 1285787 (1.2 MiB)
        RX errors 0  dropped 42  overruns 0  frame 0
        TX packets 1227  bytes 212190 (207.2 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0


Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.239.83.1     0.0.0.0         UG    101    0        0 enp137s0f4u2
10.239.83.0     0.0.0.0         255.255.255.0   U     101    0        0 enp137s0f4u2
169.254.3.0     0.0.0.0         255.255.255.0   U     100    0        0 enp2s0f4u1u2c2




