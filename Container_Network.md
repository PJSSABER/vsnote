#### Network base point
##### 隧道技术

##### vlan
  
##### vxlan  
    是对传统VLAN协议的一种扩展。VXLAN的特点是将L2的以太帧封装到UDP报文（即L2 over L4）中，并在L3网络中传输。在Linux中也设计为一个网络设备
    ```shell
        ip link add name type vxlan \
        id 42 \  # 通过id进行配对
        remote xxx
    ```
##### bridge  
    虚拟的网络设备，类似一个交换机. Linux开启ip forward(路由转发)后，赋予网桥IP地址(这会让包通过内核网络栈)，可以让网桥具有路由的功能
    ```shell
        # show bridges
        brctl show
    ```
##### veth-pair  
  虚拟网络设备，成对出现，会从任意一端进入，另一端传出


#### Container Network

##### Docker 7 model
1.  default bridge
    both level 2 && 3 network v-device, connecting two side of the bridge.给网桥设置IP地址后，就能像进行三层网络的转发
    ![](/images/image.png.png)
    in this mod, each container will use a Vinterface toconnect to default
    docker bridge.
    Also use DHCP to allocate ip to each container. containers connect to each other.
    ![](/images/2023-01-19-13-43-47.png)
    注意这里多个container在使用同一网桥的时候是在一个网段的
    网桥与container内部的虚拟网卡通过veth-pair设备连接
2.  user defined bridge
    不使用dockers的默认bridge， 而创建自己的Bridge.
3.  Host
    using host networkding directly
4.  Mac Vlan
    每个container会获取一个虚拟mac地址，与host的网关直连
    cons: 1.需要混雜模式（英語：promiscuous mode) 2. NO DHCP 需要自己指定IP
    ![](/images/2023-01-19-10-57-27.png)
5. IP Vlan l2
    与4相同， 但与host share一个mac地址，每个container会获取一个IP地址
6. IP valn l3
    利用3层网络，让host将数据转发给container 需要修改HOST上的routing table
    ![](/images/2023-01-19-10-58-17.png)

##### flannel 
![](/images/2023-01-20-11-34-43.png)
##### calic
同一局域网下可以使用纯的3层网络，不需要VXLAN进行拆解包，更加高效
![](/images/2023-01-20-11-45-48.png)
好像连网桥都不需要
不同局域网的时候如何处理：使用隧道设备
#### useful command

```shell
docker inspect
# show docker network
docker network ls

# show bridge link
bridge show

# create docker bridge
docker network create -d macvlan[...]

# show route rule
ip route [list table]

# show namespace
ip netns 

# exec in specific net namespace
ip netns exec namespace [your command]
```