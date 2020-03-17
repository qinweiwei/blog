---
title: contiv netplugin源码分析2
tag:
    - contiv
    - kubernetes
date: 2017-12-17
updated: 2017-12-17
---
contiv netplugin源码分析2

# netplugin的启动过程

1. netplugin的进程的入口函数为`netplugin/netplugin/netd.go`文件中的main函数
2. 在main函数中，netplugin进程先定义了flag参数，主要包括以下参数：
3. *debug*，*syslog*，*json-log*，*host-lable*，*plugin-mode*，*config*，*vtep-ip*，*ctrl-ip*，*vlan-if*, *vxlan-port*,*cluster-store*
![s1.png](https://i.loli.net/2020/03/17/ZklUf6rtHhEy5JQ.png)
1[s2.png](https://i.loli.net/2020/03/17/6YFXczplMABEyZR.png)

1. 定义pluginConfig属性，包含drive和instance，其中drive包括network层的driver：ovs，State的driver，默认的是etcd，instance包括参数如下
![s3.png](https://i.loli.net/2020/03/17/27zNyv5bDlVeEMZ.png)
2. 创建agent对象，然后分别执行agent的ProcessCurrentState，PostInit以及HandleEvents方法，具体方法详见以下步骤。
![s4.png](https://i.loli.net/2020/03/17/fJnMti8SL7wWghT.png)
1.  在创建agent对象时，调用`NewAgent`方法生成Agent对象，在`NewAgent`方法中分别执行etcd/consul集群的初始化，netplugin对象的初始化，然后PluginMode，分别调用docker或者kubernetes插件的初始化，最后创建Agent对象返回，Agent对象包括两部分，netplugin对象和pluginConfig
![s5.png](https://i.loli.net/2020/03/17/ysVPvYBeSjnmFMk.png)
1.  在netplugin对象的初始化中，根据pluginConfig对象Driver属性分别构建network和state的driver
![s6.png](https://i.loli.net/2020/03/17/GyxmjYotn3lapbM.png)
2. 在kubernetes插件初始化的过程中，先构建kubernetes集群的客户端，然后启动cni server，监听unix socket:`/run/contiv/contiv-cni.sock`, 接收kubernetes发送的addPod和deletePod的请求。
3. 构建完agent对象后，执行agent的ProcessCurrentState()方法。对于etcd或者consul中的
