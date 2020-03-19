---
title: kubeadm源码分析
date: 2018-01-15
updated: 2018-01-15
tags:
    - kubernetes
---
# kubeadm源码分析
## kubeadm cmd入口
* kubeadm代码的主要位置为cmd/kubeadm中
* 使用pflag, cobra的库实现的命令行入口,具体可见[https://o-my-chenjian.com/2017/09/20/Using-Cobra-With-Golang/](https://o-my-chenjian.com/2017/09/20/Using-Cobra-With-Golang/ "Golang之使用Cobra")；[http://tonybai.com/2015/07/01/config-solutions-for-golang-app/](http://tonybai.com/2015/07/01/config-solutions-for-golang-app/)
* 在`cmd/kubeadm/app/cmd/cmd.go`文件中，定义了各个kubeadm命令的子函数
``` python
func NewKubeadmCommand(f cmdutil.Factory, in io.Reader, out, err io.Writer) *cobra.Command {
    cmds.AddCommand(NewCmdCompletion(out, ""))
    cmds.AddCommand(NewCmdInit(out)) //定义了kubeadm init子命令的入口函数
    cmds.AddCommand(NewCmdJoin(out)) //定义了kubeadm join子命令的入口函数
    cmds.AddCommand(NewCmdReset(out)) //定义了kubeadm reset子命令的入口函数
    cmds.AddCommand(NewCmdVersion(out))
    cmds.AddCommand(NewCmdToken(out, err))
```
* 然后执行cmd.Execute方法
<!-- more -->
## kubeadm init
* kubeadm init代码主要位置为 `cmd/kubeadm/app/cmd/init.go`文件中。
* 当程序执行kubeadm init时，命令行参数都保存在`MasterConfiguration`对象中，然后依次执行`Init`对象的`Validate,Run`方法，执行kubeadm init命令。
* 在`Validate`方法中对kubeadm init子命令的参数进行进一步的验证，这里需要注意的是，kubeadm init 命令行参数如果使用了`config`时，不能在传递其他参数
* 在`Run`方法中, 主要执行以下几个步骤:
    - 构建证书ssl的证书文件，生成证书时先判断`/etc/kubernetes/pki`目录下证书是否存在,如果存在则直接加载,若不存在则生成证书;主要生成以下几个证书`ca.crt`,`apiserver.crt`,`apiserver-kubelet-client.crt`,`sa.pub`等。
    - 生成kubeconfig文件, 使用apiserver的证书ca.crt生成admin，kubelet，scheduler，Controller的kubeconfig，其中admin和kubelet的用户即Organization分别为system:master, system:nodes
    - 生成静态pod的manifest文件, 根据cfg中的参数分别生成apiserver，scheduler，controller，proxy的静态podmanifest文件，使得kubelet启动个组件容器。
    - 使用admin的kubeconfig文件构建client对象，检查apiserver的是否启动成功，成功则继续，否则一直等待，直到500s后超时
    - 给master节点打上node标签
    - 为bootstrap token生成secret,该secret对象包含bootstrap的tokenId，tokenSecret及time，description，usages
    - 为bootstrap生成名为cluster-info的configmap对象，保持cluster信息
    - 创建必要serviceAccount，即kube-dns和kube-proxy
    - 创建RBAC rules
    - 通过deamonset创建kubeProxy组件
    - 通过deployment创建kubeDNS组件
## kubeadm join
* kubeadm join代码的主要位置为`cmd/kubeadm/app/cmd/join.go`文件中
* 当程序执行kubeadm join时，命令行参数都保存在对象NodeConfiguration中，然后一次执行`Join`对象的`Validate`,`Run`方法，执行kubeadm join命令
* 在`Validata`方法中，进一步验证kubeadm join的有效性，，这里需要注意的是，kubeadm join命令行参数如果使用了`config`时，不能在传递其他参数。 同时传递token参数等同于discovery-token和tls-bootstrap-token
* 在`Run`方法中，主要执行以下步骤：
    - 根据APIServers的地址，生产kubeconfig对象获取cluster-info的configmap对象，获取kubernetes集群的信息，其中kubeconfig的中仅仅指定了server地址，InsecureSkipTlsVerify为true。
    ![kubeadm_join_kubeconfig.png](https://i.loli.net/2020/03/17/QwUMzDgT8H7iZv6.png)
    - 根据bootstrap token基于jws（json web signature）验证cluster-info中的集群信息
    - 根据cluster中的ca，bootstrap token， tls-bootstrap-token-user的user，重新生产tls bootstrap的kubeconfig对象
    - 利用tls bootstrap的kubeconfig对象向APIServer申请该node的证书和私钥，并将kubeconfig保存至`/etc/kubernetes/kubelet.conf`文件中，同时current context对象修改为使用证书和私钥的context：kubelet-csr与kubeapi-server通信
* 至此kubeadm join执行完成，可以通过`kubectl get node`获取到该node信息
## kubeadm reset
* kubeadm reset子命令的代码位置为`cmd/kubeadm/app/cmd/reset.go`文件中
* 当程序执行`kubeadm reset`操作时，会执行函数`NewCmdReset`方法，并将参数保存到对象Reset中，并执行`Run`方法中，主要执行以下步骤：
    - 判断系统否是为system，是的话停止kubelet service
    - unmount 目录`/var/libe/kubelet`下的所有mount点
    - 判断docker服务是否运行，如果运行的话，停止容器名前缀为k8s_的容器
    - 删除`/var/lib/kubelet`,`/etc/cni/net.d`,`/var/lib/dockershim`及`/var/lib/etcd`目录下的全部文件
* kubeadm resest操作执行完成
## kubeadm token
* kubeadm token子命令的代码位于`cmd/kubeadm/cmd/token.go`文件中。
* kubeadm token子命令的入口在函数`NewCmdToken`中，在`NewCmdToken`函数中又定义了`create`,`delete`,`generate`和`list`子命令。
* 当程序执行`kubeadm token create`子命令时，主要执行以下步骤：
    - 根据传入的参数**kubeconfig**（默认值为**/etc/kubernetes/admin.conf**）生成client对象
    - 如果传入的token值为空，则随机生成bootstrap token，否则验证传入的token值格式是否正确。
    - 使用client创建或者更新类型为bootstrap.kubernetes.io/token的secret对象
* 当程序执行`kubeadm token delete`子命令时， 同理利用**kubeconfig**创建client对象，然后通过client对象删除k8s的secret对象。
* 当程序执行`kubeadm token list`子命令时，同理调用secret list命令获取bootstrap token
* 当程序执行`kubeadm token generate`子命令时，只生成随机的bootstrap token，但是不创建bootstarp的secret对象。
