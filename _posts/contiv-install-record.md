---
title: contiv netplugin安装
date: 2017-12-16
updated: 2017-12-17
tags:
    - kubernetes
    - contiv
---
### 前置条件
1. 安装docker： yum install docker
> centos7上默认安装的docker版本为1.10，若要安装1.12需要修改Centos-Base.repo文件中[extras]中使用`baseurl`,注释掉`mirrorlist`
2.  生成kubernetes.repo文件
```
[kubernetes]
name=Kubernetes
baseurl=http://yum.kubernetes.io/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
```
3.  shadowsocks&privoxy搭建
<!-- more -->
* pip install shadowsocks
* 构建shadowsocks的配置文件

    {
      "server": "xxxx",
      "server_port": 443,
      "password": "******",
      "method": "aes-256-cfb",
      "remarks": "",
      "auth": false,
      "local_port": 1080
    }

* 启动shadowsocks
> sslocal -c shadow.json
* yum install privoxy
* 修改/etc/privoxy/config文件
> listen-address  10.18.74.200:8118
> forward-socks5t   /               127.0.0.1:1080 .
* systemctl start privoxy
* 配置yum代理
> export http_proxy=http://10.18.74.200:8118/
> export htts_proxy=http://10.18.74.200:8118/
* 配置docker的代理
```
1. 在/etc/systemd/system/目录下新建文件夹 docker.service.d
2. 在docker.service.d目录下新建文件http.conf
3. 在http.conf文件写入
    [Service]
    Environment="HTTP_PROXY=http://10.18.74.200:8118/" "HTTPS_PROXY=https://10.18.74.200:8118/"
4. 重启docker systemctl restart docker
```
4. 安装kubeadm
> yum install kubeadm
---
### 安装kubernetes和contiv netplugin
1.  在master节点上执行kubeadm init --kubernetes-version v1.6.2 --apiserver-advertise-address 10.18.74.200 --skip-preflight-checks=true
> ps: kubeadm会启动kubelet进程，但有时会启动失败，需要手动systemctl start kubelet
2. 下载contiv-1.1.1.tar并解压
3. 执行 sudo ./contiv-1.1.1/install/k8s/install.sh -n 10.18.74.200
4. 然后在slave节点上执行 kubeadm join --token 77fdb5.fa8af48e0d899b9e 10.18.74.203:6443 --skip-preflight-checks=true
5. 执行kubectl get nodes查看node
6. 访问https://10.18.74.200:10000通过UI访问contiv netplugin
