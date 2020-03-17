---
title: kubernetes集群中calico网络不通的问题定位过程记录
date: 2018-02-16 
updated: 2018-02-16 
tags:
    - kubernetes
    - calico
---
# kubernetes集群中calico网络不通的问题定位过程记录
手动搭建kubernetes集群，并基于Standard Hosted Install方式搭建calico网络，具体可见上一篇博客，搭建完成后创建pod，发现网络不通，以下为定位并解决该问题的过程记录如下：
1. 通过`kubectl get pod -o wide`查看pod的ip地址及所在node节点
<!-- more -->
2. 在所在node节点查看路由信息，未发现异常

```
[root@qinweiwei-2 ~]# ip r
default via 172.168.0.1 dev eth0  proto static  metric 100 
169.254.169.254 via 172.168.0.1 dev eth0  proto dhcp  metric 100 
172.17.0.0/16 dev docker0  proto kernel  scope link  src 172.17.0.1 
172.168.0.0/24 dev eth0  proto kernel  scope link  src 172.168.0.27  metric 100 
blackhole 192.168.135.0/26  proto bird 
192.168.135.3 dev cali12d4a061371  scope link 
192.168.143.64/26 via 172.168.0.28 dev tunl0  proto bird onlink 
192.168.252.128/26 via 172.168.0.25 dev tunl0  proto bird onlink

[root@qinweiwei-1 ~]# ip r
default via 172.168.0.1 dev eth0  proto static  metric 100 
169.254.169.254 via 172.168.0.1 dev eth0  proto dhcp  metric 100 
172.17.0.0/16 dev docker0  proto kernel  scope link  src 172.17.0.1 
172.168.0.0/24 dev eth0  proto kernel  scope link  src 172.168.0.25  metric 100 
192.168.135.0/26 via 172.168.0.27 dev tunl0  proto bird onlink 
192.168.143.64/26 via 172.168.0.28 dev tunl0  proto bird onlink 
blackhole 192.168.252.128/26  proto bird 
```

3. calico默认安装方式为ipip模式，在节点*qinweiwei-1*上ping pod的ip 192.168.135.3地址无法ping通，在节点*qinweiwei-2*上通过tcpdump命令抓包，未抓到任何包
> tcpdump -i any -nvv ip proto 4 

4. 因为环境是基于openstack的虚机搭建，于是查看虚机的secgroup规则时发现，由于secgroup验证，虚机之前ipip协议的包通过安全组，设置安全组后，发现*qinweiwei-2*上通过tcpdump抓包后抓到的icmp的包，但是仍然无法ping通

5. 查看route没有问题，怀疑是iptables的原因，执行`iptables -F`命令后，可以ping通了，可以确定是iptables的原因导致无法ping通

6. 依次执行iptables -nvxL INPUT ，iptables -F INPUT等类似的命令，最终确定是规则链`cali-pri-k8s_ns.default
`造成无法ping通

```
[root@qinweiwei-2 ~]# iptables --line -nvL cali-pri-k8s_ns.default
Chain cali-pri-k8s_ns.default (1 references)
num   pkts bytes target     prot opt in     out     source               destination         
1        0     0 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:yVw1jR_AsGFo9Rpe *
```

7. 查看资料该自定义的链与calico的profile有关，通过calioctl命令查询profile，发现profile为空

```
[root@qinweiwei-1 ~]# ETCD_ENDPOINTS=https://172.168.0.25:2379,https://172.168.0.27:2379,https://172.168.0.28:2379 ETCD_KEY_FILE=/etc/kubernetes/ssl/kubernetes-key.pem ETCD_CERT_FILE=/etc/kubernetes/ssl/kubernetes.pem ETCD_CA_CERT_FILE=/etc/kubernetes/ssl/ca.pem calicoctl get profile
NAME 

```

8. 手动创建名为k8s_ns.default的proflie后，网络可以ping通

```
[root@qinweiwei-1 ~]# cat profile.yaml 
apiVersion: v1
kind: profile
metadata:
  name: k8s_ns.default
  tags:
    - k8s_ns.default
spec:
  egress:
  - action: allow
    destination: {}
    source: {}
  ingress:
  - action: allow
    destination: {}
    source: {}
---
[root@qinweiwei-2 ~]# iptables --line -nvL cali-pri-k8s_ns.default
Chain cali-pri-k8s_ns.default (1 references)
num   pkts bytes target     prot opt in     out     source               destination         
1        0     0 MARK       all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:6MWuUqsVPzpSgE3L */ MARK or 0x1000000
2        0     0 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* cali:UGCdoOXoPRcONGv8 */ mark match 0x1000000/0x1000000
```

8. 可以发现造成网络不通的原因是kuberentes的ns的profile没有自动创建导致的，初步怀疑是calico-policy-controller的问题
9. 查看calio-policy-controller的log发现controller与kuberentes api交互抱出大量的JSONDecodeError，应该是controller获取namespaces，pod出错

```
2017-07-05 09:05:32,405 5 INFO Restarting watch on resource: Pod
2017-07-05 09:05:32,408 5 ERROR Unhandled exception killed NetworkPolicy manager
Traceback (most recent call last):
  File "<string>", line 324, in _manage_resource
  File "<string>", line 441, in _sync_resources
  File "site-packages/requests/models.py", line 866, in json
  File "site-packages/simplejson/__init__.py", line 516, in loads
  File "site-packages/simplejson/decoder.py", line 374, in decode
  File "site-packages/simplejson/decoder.py", line 404, in raw_decode
JSONDecodeError: Expecting value: line 1 column 1 (char 0)
2017-07-05 09:05:32,408 5 INFO Restarting watch on resource: NetworkPolicy
2017-07-05 09:05:33,303 5 INFO Syncing 'Namespace' objects
```

10. 通过命令ss命令，查看到calico-policy-controller与kuberentes的api连接已经建立

```
[root@qinweiwei-1 ~]# ss -nap | grep 16788
tcp    ESTAB      0      0      172.168.0.25:45133              10.254.0.1:443                 users:(("controller",pid=16788,fd=7))
tcp    ESTAB      0      0      172.168.0.25:57403              172.168.0.25:2379                users:(("controller",pid=16788,fd=8))
tcp    ESTAB      0      0      172.168.0.25:45135              10.254.0.1:443                 users:(("controller",pid=16788,fd=6))
tcp    ESTAB      0      0      172.168.0.25:57406              172.168.0.25:2379                users:(("controller",pid=16788,fd=9))
tcp    ESTAB      0      0      172.168.0.25:57356              172.168.0.25:2379                users:(("controller",pid=16788,fd=4))
tcp    ESTAB      0      0      172.168.0.25:45134              10.254.0.1:443                 users:(("controller",pid=16788,fd=5))
```

从而发现连接没有问题，应该是权限的问题

11. 最终发现是权限的问题，由于calico-policy-controller的serviceaccount设置错误，导致controller获取ns， pod等api失败，修改后问题解决。






