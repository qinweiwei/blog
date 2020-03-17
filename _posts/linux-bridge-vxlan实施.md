---
title: 基于linux bridge的vxlan测试
tags:
    - vxlan
---

## 基于linux bridge的vxlan测试

1.  创建linux brige
```
brctl addbr br-vx100
ip link set br-vx100 up
```
<!-- more -->

2.  创建vxlan interface
```
ip link add vxlan100 type vxlan id 100 remote 10.5.42.225 local 10.5.42.224 dstport 4789
ip link set vxlan100 up
```
3.  将vxlan interface加入到bridge中
```
brctl addif br-vx100 vxlan100
```
4.  创建namespace,veth对,将veth的一端加入bridge,另一端加入namespace中
```
ip link add veth100 type veth peer name veth101
ip netns add ns100
ip link set veth100 netns ns100
ip netns exec ns100 ip addr add 10.10.10.2/24 dev veth100
ip netns exec ns100 ip link set veth100 mtu 1450
ip netns exec ns100 ip link set veth100 up
ip link set veth101 mtu 1450
ip link set veth101 up
brctl addif br-vx100 veth101
```
5.  同理在另一台主机上执行以上操作,配置veth100的ip地址为10.10.10.1
6.  测试两台主机的namespace内的连通性
```
ip netns exec ns100 ping 10.10.10.1
ip netns exec ns100 ip neighbor
```
7.  通过brideg fdb show的命令查看vxlan的fdb
```
bridge fdb show
```
## 测试vxlan的性能
1.  安装iperf
```
yum install epel-releame
yum install -y iperf
```
2.  在一台机器的namespace内执行如下命令:
```
ip netns exec ns100 iperf -s
```
3.  在同一台机器的namespace内执行如下命令
```
ip netns exec ns100 iperf -c 10.10.10.2 -t 10 -i l -w
```