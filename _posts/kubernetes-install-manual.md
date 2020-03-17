---
title: 手动安装kubernetes
date: 2018-01-24
updated: 2018-03-05 14:39:00
tags:
    - kubernetes
---
# 手动安装kubernetes集群

## 环境准备
1. 准备4个centos7系统的虚机，搭建HA环境的kubernetes集群
    master: 172.168.0.25, 172.168.0.26, 172.168.0.27
    node: 172.168.0.28
2. 虚机之前做免秘钥认证，方便访问
    - ssh-keygen生成ssh的公钥和私钥
    - ssh-copy-id 拷贝公钥至各个节点
3. 修改各个节点的/etc/hosts文件
> 172.168.0.25 qinweiwei-1
> 172.168.0.26 qinweiwei-2
> 172.168.0.27 qinweiwei-3
> 172.168.0.28 qinweiwei-4
4. 安装ansible，方便直接在各个节点执行命令
    - yum install ansible
    - ansible all -m shell -a "command"
5. 安装docker,执行以下命令：ansible all -m command -a "yum install docker -y"
> CentOS-Base.repo文件使用`mirrorlist`,不要用`baseurl`
> mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=os&infra=$infra
#baseurl=http://mirror.centos.org/centos/$releasever/os/$basearch/
6. 关闭所有节点的SELinux,使用命令 setenforce 0
## 创建TLS证书和秘钥
<!-- more -->
kubernetes 系统的各组件需要使用 TLS 证书对通信进行加密, 下边依次介绍使用`cfssl`和`openssl`工具构建CA和其他证书
### 生成的 CA 证书和秘钥文件如下：
- ca-key.pem
- ca.pem
- kubernetes-key.pem
- kubernetes.pem
- kube-proxy.pem
- kube-proxy-key.pem
- admin.pem
- admin-key.pem
### 使用证书的组件如下：
- etcd：使用 ca.pem、kubernetes-key.pem、kubernetes.pem
- kube-apiserver：使用 ca.pem、kubernetes-key.pem、kubernetes.pem
- kubelet：使用 ca.pem
- kube-proxy：使用 ca.pem、kube-proxy-key.pem、kube-proxy.pem
- kubectl：使用 ca.pem、admin-key.pem、admin.pem
### 使用cfssl工具构建证书
#### 安装cfssl工具
1. go get -u github.com/cloudflare/cfssl/cmd/...
2. cp $GOPATH/bin/cfssl* /usr/bin/
#### 创建 CA (Certificate Authority)
创建 CA 配置文件

```
mkdir /root/ssl
cd /root/ssl
cfssl print-defaults config > config.json
cfssl print-defaults csr > csr.json
# 根据config.json文件的格式创建如下的ca-config.json文件
# 过期时间设置成了 87600h
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
EOF
```

字段说明:
- ca-config.json：可以定义多个 profiles，分别指定不同的过期时间、使用场景等参数；后续在签名证书时使用某个 profile
- signing：表示该证书可用于签名其它证书；生成的 ca.pem 证书中 CA=TRUE
- server auth：表示client可以用该 CA 对server提供的证书进行验证
- client auth：表示server可以用该CA对client提供的证书进行验证
创建 CA 证书签名请求
创建 ca-csr.json 文件，内容如下：

```
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

- "CN"：Common Name，kube-apiserver 从证书中提取该字段作为请求的用户名 (User Name)；浏览器使用该字段验证网站是否合法
- "O"：Organization，kube-apiserver 从证书中提取该字段作为请求用户所属的组 (Group)
生成 CA 证书和私钥:

```
$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
$ ls ca*
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem
```

#### 创建 kubernetes 证书
创建 kubernetes 证书签名请求

```
$ cat kubernetes-csr.json
{
    "CN": "kubernetes",
    "hosts": [
      "127.0.0.1",
      "172.20.0.112",
      "172.20.0.113",
      "172.20.0.114",
      "172.20.0.115",
      "10.254.0.1",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "BeiJing",
            "L": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
```

- 如果 hosts 字段不为空则需要指定授权使用该证书的 IP 或域名列表，由于该证书后续被 etcd 集群和 kubernetes master 集群使用，所以上面分别指定了 etcd 集群、kubernetes master 集群的主机 IP 和 kubernetes 服务的服务 IP（一般是 kue-apiserver 指定的 service-cluster-ip-range 网段的第一个IP，如 10.254.0.1。
生成 kubernetes 证书和私钥

```
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes
$ ls kubernetes*
kubernetes.csr  kubernetes-csr.json  kubernetes-key.pem  kubernetes.pem
```

或者直接在命令行上指定相关参数

```
$ echo '{"CN":"kubernetes","hosts":[""],"key":{"algo":"rsa","size":2048}}' | cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes -hostname="127.0.0.1,172.20.0.112,172.20.0.113,172.20.0.114,172.20.0.115,kubernetes,kubernetes.default" - | cfssljson -bare kubernetes
```

#### 创建 admin 证书
创建 admin 证书签名请求

```
$ cat admin-csr.json
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
```

- 后续 kube-apiserver 使用 RBAC 对客户端(如 kubelet、kube-proxy、Pod)请求进行授权
- kube-apiserver 预定义了一些 RBAC 使用的 RoleBindings，如 cluster-admin 将 Group system:masters 与 Role cluster-admin 绑定，该 Role 授予了调用kube-apiserver 的所有 API的权限
- OU 指定该证书的 Group 为 system:masters，kubelet 使用该证书访问 kube-apiserver 时 ，由于证书被 CA 签名，所以认证通过，同时由于证书用户组为经过预授权的 system:masters，所以被授予访问所有 API 的权限

生成 admin 证书和私钥

```
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
$ ls admin*
admin.csr  admin-csr.json  admin-key.pem  admin.pem
```
#### 创建 kube-proxy 证书
创建 kube-proxy 证书签名请求

```
$ cat kube-proxy-csr.json
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

- CN 指定该证书的 User 为 system:kube-proxy
- kube-apiserver 预定义的 RoleBinding cluster-admin 将User system:kube-proxy 与 Role system:node-proxier 绑定，该 Role 授予了调用 kube-apiserver Proxy 相关 API 的权限
生成 kube-proxy 客户端证书和私钥

```
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes  kube-proxy-csr.json | cfssljson -bare kube-proxy
$ ls kube-proxy*
kube-proxy.csr  kube-proxy-csr.json  kube-proxy-key.pem  kube-proxy.pem
```

#### 校验证书
以 kubernetes 证书为例
- cfssl-certinfo -cert kubernetes.pem

```
{
  "subject": {
    "common_name": "kubernetes",
    "country": "CN",
    "organization": "k8s",
    "organizational_unit": "System",
    "locality": "BeiJing",
    "province": "BeiJing",
    "names": [
      "CN",
      "BeiJing",
      "BeiJing",
      "k8s",
      "System",
      "kubernetes"
    ]
  },
  "issuer": {
    "common_name": "kubernetes",
    "country": "CN",
    "organization": "k8s",
    "organizational_unit": "System",
    "locality": "BeiJing",
    "province": "BeiJing",
    "names": [
      "CN",
      "BeiJing",
      "BeiJing",
      "k8s",
      "System",
      "kubernetes"
    ]
  },
  "serial_number": "342576193900195567380707167251794624995944458399",
  "sans": [
    "qinweiwei-1",
    "qinweiwei-2",
    "qinweiwei-3",
    "qinweiwei-4",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local",
    "127.0.0.1",
    "172.168.0.25",
    "172.168.0.26",
    "172.168.0.27",
    "172.168.0.28",
    "10.254.0.1"
  ],
  "not_before": "2018-01-28T12:38:00Z",
  "not_after": "2028-01-26T12:38:00Z",
  "sigalg": "SHA256WithRSA",
  "authority_key_id": "",
  "subject_key_id":
  ......
}
```

- 确认 Issuer 字段的内容和 ca-csr.json 一致
- 确认 Subject 字段的内容和 kubernetes-csr.json 一致
- 确认 X509v3 Subject Alternative Name 字段的内容和 kubernetes-csr.json 一致
- 确认 X509v3 Key Usage、Extended Key Usage 字段的内容和 ca-config.json 中 kubernetes profile 一致

#### 分发证书
将生成的证书和秘钥文件（后缀名为.pem）拷贝到所有机器的 /etc/kubernetes/ssl 目录下备用

```
$ sudo mkdir -p /etc/kubernetes/ssl
$ sudo cp *.pem /etc/kubernetes/ssl
```

### 使用openssl工具构建证书和秘钥
#### 创建CA
- openssl genrsa -out ca-key.pem 2048
- openssl req -x509 -new -nodes -key ca-key.pem -days 10000 -out ca.pem -subj "/CN=kubernetes"
#### 创建kubernetes证书
- cp /etc/pki/tls/openssl.cnf .
- vim openssl.cnf

```
[req]
req_extensions = v3_req # 这行默认注释关着的 把注释删掉# 下面配置是新增的
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = qinweiwei-1
DNS.2 = qinweiwei-2
DNS.3 = qinweiwei-3
DNS.4 = qinweiwei-4
DNS.5 = kubernetes
DNS.6 = kubernetes.default
DNS.7 = kubernetes.default.svc
DNS.8 = kubernetes.default.svc.cluster
DNS.9 = kubernetes.default.svc.cluster.local
IP.1 = 127.0.0.1
IP.2 = 172.168.0.25
IP.3 = 172.168.0.26
IP.4 = 172.168.0.27
IP.5 = 172.168.0.28
IP.6 = 10.254.0.1
```

- openssl genrsa -out kubernetes-key.pem 2048
- openssl req -new -key kubernetes-key.pem -out kubernetes.csr -subj "/C=CN/ST=beijing/L=beijing/O=k8s/OU=system/CN=kubernetes/" -config openssl.cnf
- openssl x509 -req -in kubernetes.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out kubernetes.pem -days 365 -extensions v3_req -extfile openssl.cnf
#### admin 证书
- cp /etc/pki/tls/openssl.cnf .
- vim openssl.cnf

```
[req]
req_extensions = v3_req # 这行默认注释关着的 把注释删掉# 下面配置是新增的
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
```

- openssl genrsa -out admin-key.pem 2048
- openssl req -new -key admin-key.pem -out admin.csr -subj "/C=CN/ST=beijing/L=beijing/O=system:masters/OU=system/CN=admin/" -config openssl.cnf
- openssl x509 -req -in admin.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out admin.pem -days 365 -extensions v3_req -extfile openssl.cnf
#### 创建kube-proxy的证书
同admin证书一样，只是-subj "/CN=system:kube-proxy"
#### 校验证书
以 kubernetes 证书为例
- openssl x509  -noout -text -in  kubernetes.pem

```
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            8b:21:12:8d:52:e2:0e:60
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=kubernetes
        Validity
            Not Before: Jan 28 14:23:58 2018 GMT
            Not After : Jan 28 14:23:58 2019 GMT
        Subject: C=CN, ST=beijing, L=beijing, O=k8s, OU=system, CN=kubernetes
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:a8:07:b3:42:ef:be:b3:15:92:35:4d:ab:5e:92:
                    c9:34:cb:97:49:41:11:b4:6f:87:0f:b6:8b:2a:e4:
                    c0:ac:b3:a3:f2:51:5e:0b:7f:fb:2f:85:d6:56:a2:
                    a5:65:91:79:9f:06:84:c4:4b:a7:2d:21:a4:e9:96:
                    07:87:31:80:4b:63:2d:c9:9b:88:aa:77:85:96:7d:
                    8f:b0:39:d0:ea:8c:ab:9c:af:17:5d:d9:68:65:0b:
                    c1:b3:27:f2:92:94:70:fe:b6:27:8c:26:95:6a:bc:
                    b2:80:22:c8:e9:1d:79:10:7f:07:88:c0:d8:8b:31:
                    15:99:7d:33:e0:ac:62:8c:04:b6:ce:e5:f8:aa:37:
                    53:20:e2:38:ef:bc:1f:4c:66:5c:47:99:c1:d7:8c:
                    03:44:5b:22:36:bc:c5:55:8c:5b:a9:4d:ab:3a:9f:
                    fa:09:3f:35:57:23:c7:05:e5:c7:c8:e8:23:5c:2f:
                    e2:12:f4:fc:65:79:a0:4f:56:91:47:6b:7b:0b:82:
                    56:ae:e3:c9:10:77:0b:ad:33:20:14:1f:b0:25:2e:
                    76:aa:e6:51:4c:8d:ad:3f:9b:73:ae:32:8b:f0:76:
                    3d:1d:48:02:76:23:e2:d2:27:69:94:9a:14:de:20:
                    c2:33:ed:21:41:16:ef:69:64:82:a5:f7:6d:56:48:
                    2c:71
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Basic Constraints:
                CA:FALSE
            X509v3 Key Usage:
                Digital Signature, Non Repudiation, Key Encipherment
            X509v3 Subject Alternative Name:
                DNS:qinweiwei-1, DNS:qinweiwei-2, DNS:qinweiwei-3, DNS:qinweiwei-4, DNS:kubernetes, DNS:kubernetes.default, DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.cluster, DNS:kubernetes.default.svc.cluster.local, IP Address:127.0.0.1, IP Address:172.168.0.25, IP Address:172.168.0.26, IP Address:172.168.0.27, IP Address:172.168.0.28, IP Address:10.254.0.1
                ......
```

#### 分发证书
将生成的证书和秘钥文件（后缀名为.pem）拷贝到所有机器的 /etc/kubernetes/ssl 目录下备用

```
$ sudo mkdir -p /etc/kubernetes/ssl
$ sudo cp *.pem /etc/kubernetes/ssl
```

## 搭建高可用etcd集群
### TLS 认证文件
需要为 etcd 集群创建加密通信的 TLS 证书，这里复用以前创建的 kubernetes 证书
> $ cp ca.pem kubernetes-key.pem kubernetes.pem /etc/kubernetes/ssl

- kubernetes 证书的 hosts 字段列表中包含上面三台机器的 IP，否则后续证书校验会失败

### 下载二进制文件
到 https://github.com/coreos/etcd/releases 页面下载最新版本的二进制文件

```
$ https://github.com/coreos/etcd/releases/download/v3.1.5/etcd-v3.1.5-linux-amd64.tar.gz
$ tar -xvf etcd-v3.1.5-linux-amd64.tar.gz
$ sudo mv etcd-v3.1.5-linux-amd64/etcd* /usr/bin
```

### 创建 etcd 的 systemd unit 文件
注意替换IP地址为你自己的etcd集群的主机IP。

```
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
ExecStart=/usr/bin/etcd \
  --name ${ETCD_NAME} \
  --cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  --peer-cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --peer-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --initial-advertise-peer-urls ${ETCD_INITIAL_ADVERTISE_PEER_URLS} \
  --listen-peer-urls ${ETCD_LISTEN_PEER_URLS} \
  --listen-client-urls ${ETCD_LISTEN_CLIENT_URLS},https://127.0.0.1:2379 \
  --advertise-client-urls ${ETCD_ADVERTISE_CLIENT_URLS} \
  --initial-cluster-token ${ETCD_INITIAL_CLUSTER_TOKEN} \
  --initial-cluster qinweiwei-1=https://172.168.0.25:2380,qinweiwei-2=https://172.168.0.27:2380,qinweiwei-3=https://172.168.0.28:2380 \
  --initial-cluster-state new \
  --data-dir=${ETCD_DATA_DIR}
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

- 指定 etcd 的工作目录为 /var/lib/etcd，数据目录为 /var/lib/etcd，需在启动服务前创建这两个目录，否则etcd进程启动失败。
- 为了保证通信安全，需要指定 etcd 的公私钥(cert-file和key-file)、Peers 通信的公私钥和 CA 证书(peer-cert-file、peer-key-file、peer-trusted-ca-file)、客户端的CA证书（trusted-ca-file）
- 创建 kubernetes.pem 证书时使用的 kubernetes-csr.json 文件的 hosts 字段包含所有 etcd 节点的IP，否则证书校验会出错
- --initial-cluster-state 值为 new 时，--name 的参数值必须位于 --initial-cluster 列表中

环境变量配置文件/etc/etcd/etcd.conf。

```
# [member]
ETCD_NAME=qinweiwei-1
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_LISTEN_PEER_URLS="https://172.168.0.25:2380"
ETCD_LISTEN_CLIENT_URLS="https://172.168.0.25:2379"

#[cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://172.168.0.25:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="https://172.168.0.25:2379"
```

其他两个etcd节点只要将上面的IP地址改成相应节点的IP地址即可

### 启动 etcd 服务

```
$ sudo mv etcd.service /etc/systemd/system/
$ sudo systemctl daemon-reload
$ sudo systemctl enable etcd
$ sudo systemctl start etcd
$ systemctl status etcd
```

###  验证服务
在任一 kubernetes master 机器上执行如下命令
> etcdctl --ca-file=/etc/kubernetes/ssl/ca.pem --cert-file=/etc/kubernetes/ssl/kubernetes.pem --key-file=/etc/kubernetes/ssl/kubernetes-key.pem --endpoints https://172.168.0.27:2379 cluster-health

结果最后一行为 cluster is healthy 时表示集群服务正常。

## 搭建kubernetes master节点
下载kube-apiserver，kube-scheduler，kube-controller-manage，kubelet，kube-proxy的二进制文件，并拷贝至PATH路径下

```
$ wget https://dl.k8s.io/v1.7.0/kubernetes-server-linux-amd64.tar.gz
$ tar -xzvf kubernetes-server-linux-amd64.tar.gz
$ cp -r server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl,kube-proxy,kubelet} /usr/bin/
```

### 配置和启动kube-apiserver
serivce配置文件/usr/lib/systemd/system/kube-apiserver.service内容：

```
[Unit]
Description=Kubernetes API Service
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target
After=etcd.service

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/apiserver
ExecStart=/usr/bin/kube-apiserver \
      $KUBE_LOGTOSTDERR \
      $KUBE_LOG_DIR \
      $KUBE_LOG_LEVEL \
      $KUBE_ETCD_SERVERS \
      $KUBE_API_ADDRESS \
      $KUBE_API_PORT \
      $KUBELET_PORT \
      $KUBE_ALLOW_PRIV \
      $KUBE_SERVICE_ADDRESSES \
      $KUBE_ADMISSION_CONTROL \
      $KUBE_API_ARGS
Restart=on-failure
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

/etc/kubernetes/config文件的内容为:

```
###
# kubernetes system config
#
# The following values are used to configure various aspects of all
# kubernetes services, including
#
#   kube-apiserver.service
#   kube-controller-manager.service
#   kube-scheduler.service
#   kubelet.service
#   kube-proxy.service
# logging to stderr means we get it in the systemd journal
KUBE_LOG_DIR="--log-dir=/var/log/kubernetes/"
KUBE_LOGTOSTDERR="--logtostderr=false"

# journal message level, 0 is debug
KUBE_LOG_LEVEL="--v=4"

# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow-privileged=true"

# How the controller-manager, scheduler, and proxy find the apiserver
#KUBE_MASTER="--master=http://sz-pg-oam-docker-test-001.tendcloud.com:8080"
KUBE_MASTER="--master=http://172.168.0.25:8080"
```

该配置文件同时被kube-apiserver、kube-controller-manager、kube-scheduler、kubelet、kube-proxy使用。

apiserver配置文件/etc/kubernetes/apiserver内容为:

```
###
## kubernetes system config
##
## The following values are used to configure the kube-apiserver
##
#
## The address on the local server to listen to.
#KUBE_API_ADDRESS="--insecure-bind-address=sz-pg-oam-docker-test-001.tendcloud.com"
KUBE_API_ADDRESS="--advertise-address=172.168.0.25 --bind-address=172.168.0.25 --insecure-bind-address=172.168.0.25"
#
## The port on the local server to listen on.
#KUBE_API_PORT="--port=8080"
#
## Port minions listen on
#KUBELET_PORT="--kubelet-port=10250"
#
## Comma separated list of nodes in the etcd cluster
KUBE_ETCD_SERVERS="--etcd-servers=https://172.168.0.25:2379,https://172.168.0.27:2379,https://172.168.0.28:2379"
#
## Address range to use for services
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"
#
## default admission control policies
KUBE_ADMISSION_CONTROL="--admission-control=ServiceAccount,NamespaceLifecycle,NamespaceExists,LimitRanger,ResourceQuota"
#
## Add your own!
KUBE_API_ARGS="--authorization-mode=RBAC --runtime-config=rbac.authorization.k8s.io/v1beta1 --kubelet-https=true --experimental-bootstrap-token-auth --token-auth-file=/etc/kubernetes/token.csv --service-node-port-range=30000-32767 --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem --client-ca-file=/etc/kubernetes/ssl/ca.pem --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem --etcd-cafile=/etc/kubernetes/ssl/ca.pem --etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem --etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem --enable-swagger-ui=true --apiserver-count=3 --audit-log-maxage=30 --audit-log-maxbackup=3 --audit-log-maxsize=100 --audit-log-path=/var/lib/audit.log --event-ttl=1h"
```

- --authorization-mode=RBAC 指定在安全端口使用 RBAC 授权模式，拒绝未通过授权的请求；
- kube-scheduler、kube-controller-manager 一般和 kube-apiserver 部署在同一台机器上，它们使用非安全端口和 kube-apiserver通信;
- kubelet、kube-proxy、kubectl 部署在其它 Node 节点上，如果通过安全端口访问 kube-apiserver，则必须先通过 TLS 证书认证，再通过 RBAC 授权；
- kube-proxy、kubectl 通过在使用的证书里指定相关的 User、Group 来达到通过 RBAC 授权的目的；
- 缺省情况下 kubernetes 对象保存在 etcd /registry 路径下，可以通过 --etcd-prefix 参数进行调整；
- runtime-config配置为rbac.authorization.k8s.io/v1beta1，表示运行时的apiVersion；
启动apiserver

```
$ systemctl daemon-reload
$ systemctl enable kube-apiserver
$ systemctl start kube-apiserver
$ systemctl status kube-apiserver
```

### 配置和启动 kube-controller-manager
创建 kube-controller-manager的serivce配置文件:

```
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/controller-manager
ExecStart=/usr/bin/kube-controller-manager \
      $KUBE_LOGTOSTDERR \
            $KUBE_LOG_DIR \
      $KUBE_LOG_LEVEL \
      $KUBE_MASTER \
      $KUBE_CONTROLLER_MANAGER_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

配置文件/etc/kubernetes/controller-manager

```
###
# The following values are used to configure the kubernetes controller-manager

# defaults from config and apiserver should be adequate

# Add your own!
KUBE_CONTROLLER_MANAGER_ARGS="--address=127.0.0.1 --service-cluster-ip-range=10.254.0.0/16 --cluster-name=kubernetes --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem --root-ca-file=/etc/kubernetes/ssl/ca.pem --leader-elect=true --use-service-account-credentials=true --controllers=*,bootstrapsigner,tokencleaner --allocate-node-cidrs=true --cluster-cidr=192.168.0.0/16"
```

- --service-cluster-ip-range 参数指定 Cluster 中 Service 的CIDR范围，该网络在各 Node 间必须路由不可达，必须和 kube-apiserver 中的参数一致；
- -cluster-signing-* 指定的证书和私钥文件用来签名为 TLS BootStrap 创建的证书和私钥；
- --root-ca-file 用来对 kube-apiserver 证书进行校验，指定该参数后，才会在Pod 容器的 ServiceAccount 中放置该 CA 证书文件；
- --address 值必须为 127.0.0.1，因为当前 kube-apiserver 期望 scheduler 和 controller-manager 在同一台机器

启动 kube-controller-manager

```
$ systemctl daemon-reload
$ systemctl enable kube-controller-manager
$ systemctl start kube-controller-manager
```

### 启动 kube-controller-manager

创建 kube-scheduler的serivce配置文件,文件路径/usr/lib/systemd/system/kube-scheduler.service

```
[Unit]
Description=Kubernetes Scheduler Plugin
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/scheduler
ExecStart=/usr/bin/kube-scheduler \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBE_MASTER \
            $KUBE_SCHEDULER_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

配置文件/etc/kubernetes/scheduler

```
###
# kubernetes scheduler config

# default config should be adequate

# Add your own!
KUBE_SCHEDULER_ARGS="--leader-elect=true --address=127.0.0.1"
```

启动 kube-scheduler

```
$ systemctl daemon-reload
$ systemctl enable kube-scheduler
$ systemctl start kube-scheduler
```

验证 master 节点功能

```
$ kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
etcd-2               Healthy   {"health": "true"}
```

## 安装kubectl命令行工具

创建 kubectl kubeconfig 文件

```
$ export KUBE_APISERVER="https://172.172.168.0.25:6443"
$ # 设置集群参数
$ kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER}
$ # 设置客户端认证参数
$ kubectl config set-credentials admin \
  --client-certificate=/etc/kubernetes/ssl/admin.pem \
  --embed-certs=true \
  --client-key=/etc/kubernetes/ssl/admin-key.pem
$ # 设置上下文参数
$ kubectl config set-context kubernetes \
  --cluster=kubernetes \
  --user=admin
$ # 设置默认上下文
$ kubectl config use-context kubernetes
```

- admin.pem 证书 OU 字段值为 system:masters，kube-apiserver 预定义的 RoleBinding cluster-admin 将 Group system:masters 与 Role cluster-admin 绑定，该 Role 授予了调用kube-apiserver 相关 API 的权限
- 生成的 kubeconfig 被保存到 ~/.kube/config 文件

## 创建 kubeconfig 文件

kubelet、kube-proxy 等 Node 机器上的进程与 Master 机器的 kube-apiserver 进程通信时需要认证和授权;
kubernetes 1.4 开始支持由 kube-apiserver 为客户端生成 TLS 证书的 TLS Bootstrapping 功能，这样就不需要为每个客户端生成证书了；该功能当前仅支持为 kubelet 生成证书；

### 创建 TLS Bootstrapping Token

Token auth file
Token可以是任意的包涵128 bit的字符串，可以使用安全的随机数发生器生成

```
export BOOTSTRAP_TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')
cat > token.csv <<EOF
${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF
```

BOOTSTRAP_TOKEN 将被写入到 kube-apiserver 使用的 token.csv 文件和 kubelet 使用的 bootstrap.kubeconfig 文件，如果后续重新生成了 BOOTSTRAP_TOKEN，则需要：
- 更新 token.csv 文件，分发到所有机器 (master 和 node）的 /etc/kubernetes/ 目录下，分发到node节点上非必需
- 重新生成 bootstrap.kubeconfig 文件，分发到所有 node 机器的 /etc/kubernetes/ 目录下
- 重启 kube-apiserver 和 kubelet 进程
- 重新 approve kubelet 的 csr 请求

```
$cp token.csv /etc/kubernetes/
```

### 创建 kubelet bootstrapping kubeconfig 文件

```
$ cd /etc/kubernetes
$ export KUBE_APISERVER="https://172.172.168.0.25:6443"
$ # 设置集群参数
$ kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=bootstrap.kubeconfig
$ # 设置客户端认证参数
$ kubectl config set-credentials kubelet-bootstrap \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=bootstrap.kubeconfig
$ # 设置上下文参数
$ kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig
$ # 设置默认上下文
$ kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
```

- --embed-certs 为 true 时表示将 certificate-authority 证书写入到生成的 bootstrap.kubeconfig 文件中
- 设置客户端认证参数时没有指定秘钥和证书，后续由 kube-apiserver 自动生成

### 创建 kube-proxy kubeconfig 文件

```
$ export KUBE_APISERVER="https://172.172.168.0.25:6443"
$ # 设置集群参数
$ kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kube-proxy.kubeconfig
$ # 设置客户端认证参数
$ kubectl config set-credentials kube-proxy \
  --client-certificate=/etc/kubernetes/ssl/kube-proxy.pem \
  --client-key=/etc/kubernetes/ssl/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
$ # 设置上下文参数
$ kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
$ # 设置默认上下文
$ kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

- 设置集群参数和客户端认证参数时 --embed-certs 都为 true，这会将 certificate-authority、client-certificate 和 client-key 指向的证书文件内容写入到生成的 kube-proxy.kubeconfig 文件中
- kube-proxy.pem 证书中 CN 为 system:kube-proxy，kube-apiserver 预定义的 RoleBinding cluster-admin 将User system:kube-proxy 与 Role system:node-proxier 绑定，该 Role 授予了调用 kube-apiserver Proxy 相关 API 的权限；

### 分发 kubeconfig 文件

将两个 kubeconfig 文件分发到所有 Node 机器的 /etc/kubernetes/ 目录

```
$ cp bootstrap.kubeconfig kube-proxy.kubeconfig /etc/kubernetes/
```

## 部署node节点
### 安装和配置 kubelet
kubelet 启动时向 kube-apiserver 发送 TLS bootstrapping 请求，需要先将 bootstrap token 文件中的 kubelet-bootstrap 用户赋予 system:node-bootstrapper cluster 角色(role)， 然后 kubelet 才能有权限创建认证请求(certificate signing requests)

```
kubectl create clusterrolebinding kubelet-bootstrap \
  --clusterrole=system:node-bootstrapper \
  --user=kubelet-bootstrap
```

- --user=kubelet-bootstrap 是在 /etc/kubernetes/token.csv 文件中指定的用户名，同时也写入了 /etc/kubernetes/bootstrap.kubeconfig 文件；
创建 kubelet 的service配置文件，文件位置/usr/lib/systemd/system/kubelet.service

```
[Unit]
Description=Kubernetes Kubelet Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/kubelet
ExecStart=/usr/bin/kubelet \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_DIR \
            $KUBE_LOG_LEVEL \
            $KUBELET_API_SERVER \
            $KUBELET_ADDRESS \
            $KUBELET_PORT \
            $KUBELET_HOSTNAME \
            $KUBE_ALLOW_PRIV \
            $KUBELET_POD_INFRA_CONTAINER \
            $KUBELET_ARGS
Restart=on-failure

[Install]
WantedBy=multi-user.target

```

kubelet的配置文件/etc/kubernetes/kubelet。其中的IP地址更改为你的每台node节点的IP地址
注意：/var/lib/kubelet需要手动创建，否则可能导致kubelet启动失败

```
###
## kubernetes kubelet (minion) config
#
## The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
KUBELET_ADDRESS="--address=172.168.0.25"
#
## The port for the info server to serve on
#KUBELET_PORT="--port=10250"
#
## You may leave this blank to use the actual hostname
KUBELET_HOSTNAME="--hostname-override=qinweiwei-1"
#
## location of the api-server
#KUBELET_API_SERVER="--api-servers=http://172.168.0.25:8080"
#
## pod infrastructure container
KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=gcr.io/google_containers/pause-amd64:3.0"
#
## Add your own!
KUBELET_ARGS="--cgroup-driver=systemd --cluster-dns=10.254.0.2 --experimental-bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig --kubeconfig=/etc/kubernetes/kubelet.kubeconfig --require-kubeconfig --cert-dir=/etc/kubernetes/ssl --cluster-domain=cluster.local.--serialize-image-pulls=false --network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin"
```

- --address 不能设置为 127.0.0.1，否则后续 Pods 访问 kubelet 的 API 接口时会失败，因为 Pods 访问的 127.0.0.1 指向自己而不是 kubelet
- 如果设置了 --hostname-override 选项，则 kube-proxy 也需要设置该选项，否则会出现找不到 Node 的情况
- --experimental-bootstrap-kubeconfig 指向 bootstrap kubeconfig 文件，kubelet 使用该文件中的用户名和 token 向 kube-apiserver 发送 TLS Bootstrapping 请求
- 管理员通过了 CSR 请求后，kubelet 自动在 --cert-dir 目录创建证书和私钥文件(kubelet-client.crt 和 kubelet-client.key)，然后写入 --kubeconfig 文件
- 建议在 --kubeconfig 配置文件中指定 kube-apiserver 地址，如果未指定 --api-servers 选项，则必须指定 --require-kubeconfig 选项后才从配置文件中读取 kube-apiserver 的地址，否则 kubelet 启动后将找不到 kube-apiserver (日志中提示未找到 API Server），kubectl get nodes 不会返回对应的 Node 信息
- —kubeconfig=/etc/kubernetes/kubelet.kubeconfig中指定的kubelet.kubeconfig文件在第一次启动kubelet之前并不存在，请看下文，当通过CSR请求后会自动生成kubelet.kubeconfig文件，如果你的节点上已经生成了~/.kube/config文件，你可以将该文件拷贝到该路径下，并重命名为kubelet.kubeconfig，所有node节点可以共用同一个kubelet.kubeconfig文件，这样新添加的节点就不需要再创建CSR请求就能自动添加到kubernetes集群中。同样，在任意能够访问到kubernetes集群的主机上使用kubectl —kubeconfig命令操作集群时，只要使用~/.kube/config文件就可以通过权限认证，因为这里面已经有认证信息并认为你是admin用户，对集群拥有所有权限。
- 使用calico网络时，需要配置cni相关参数

### 启动kublet

```
$ systemctl daemon-reload
$ systemctl enable kubelet
$ systemctl start kubelet
$ systemctl status kubelet
```

### 通过 kublet 的 TLS 证书请求

kubelet 首次启动时向 kube-apiserver 发送证书签名请求，必须通过后 kubernetes 系统才会将该 Node 加入到集群。

查看未授权的 CSR 请求

```
$ kubectl get csr
NAME        AGE       REQUESTOR           CONDITION
csr-2b308   4m        kubelet-bootstrap   Pending
$ kubectl get nodes
No resources found.
```

通过 CSR 请求

```
$ kubectl certificate approve csr-2b308
certificatesigningrequest "csr-2b308" approved
$ kubectl get nodes
NAME        STATUS    AGE       VERSION
10.64.3.7   Ready     49m       v1.7.3
```

自动生成了 kubelet kubeconfig 文件和公私钥

```
$ ls -l /etc/kubernetes/kubelet.kubeconfig
-rw------- 1 root root 2284 Apr  7 02:07 /etc/kubernetes/kubelet.kubeconfig
$ ls -l /etc/kubernetes/ssl/kubelet*
-rw-r--r-- 1 root root 1046 Apr  7 02:07 /etc/kubernetes/ssl/kubelet-client.crt
-rw------- 1 root root  227 Apr  7 02:04 /etc/kubernetes/ssl/kubelet-client.key
-rw-r--r-- 1 root root 1103 Apr  7 02:07 /etc/kubernetes/ssl/kubelet.crt
-rw------- 1 root root 1675 Apr  7 02:07 /etc/kubernetes/ssl/kubelet.key
```

注意：假如你更新kubernetes的证书，只要没有更新token.csv，当重启kubelet后，该node就会自动加入到kuberentes集群中，而不会重新发送certificaterequest，也不需要在master节点上执行kubectl certificate approve操作。前提是不要删除node节点上的/etc/kubernetes/ssl/kubelet*和/etc/kubernetes/kubelet.kubeconfig文件。否则kubelet启动时会提示找不到证书而失败。

### 配置 kube-proxy

创建 kube-proxy 的service配置文件

文件路径/usr/lib/systemd/system/kube-proxy.service

```
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/proxy
ExecStart=/usr/bin/kube-proxy \
      $KUBE_LOGTOSTDERR \
            $KUBE_LOG_DIR \
      $KUBE_LOG_LEVEL \
      $KUBE_MASTER \
      $KUBE_PROXY_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

```

kube-proxy配置文件/etc/kubernetes/proxy

```
###
# kubernetes proxy config

# default config should be adequate

# Add your own!
KUBE_PROXY_ARGS="--bind-address=172.168.0.25  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig --cluster-cidr=10.254.0.0/16 --proxy-mode=iptables"
```

- --hostname-override 参数值必须与 kubelet 的值一致，否则 kube-proxy 启动后会找不到该 Node，从而不会创建任何 iptables 规则；
- kube-proxy 根据 --cluster-cidr 判断集群内部和外部流量，指定 --cluster-cidr 或 --masquerade-all 选项后 kube-proxy 才会对访问 Service IP 的请求做 SNAT；
- --kubeconfig 指定的配置文件嵌入了 kube-apiserver 的地址、用户名、证书、秘钥等请求和认证信息；
预定义的 RoleBinding cluster-admin 将User system:kube-proxy 与 Role system:node-proxier 绑定，该 Role 授予了调用 kube-apiserver Proxy 相关 API 的权限；


### 启动 kube-proxy

```
$ systemctl daemon-reload
$ systemctl enable kube-proxy
$ systemctl start kube-proxy
$ systemctl status kube-proxy
```

## 部署calico

### 下载calico相关的docker镜像

```
docker pull quay.io/calico/cni:v1.10.0
docker pull quay.io/calico/kube-policy-controller:v0.7.0
docker pull quay.io/calico/node:v2.5.1
```

### 构建RBAC
> kubectl apply -f https://docs.projectcalico.org/v2.5/getting-started/kubernetes/installation/rbac.yaml

### 安装calico

- 下载calico.yaml文件

```
# Calico Version v2.5.1
# https://docs.projectcalico.org/v2.5/releases#v2.5.1
# This manifest includes the following component versions:
#   calico/node:v2.5.1
#   calico/cni:v1.10.0
#   calico/kube-policy-controller:v0.7.0

# This ConfigMap is used to configure a self-hosted Calico installation.
kind: ConfigMap
apiVersion: v1
metadata:
  name: calico-config
  namespace: kube-system
data:
  # Configure this with the location of your etcd cluster.
  etcd_endpoints: "http://127.0.0.1:2379"

  # Configure the Calico backend to use.
  calico_backend: "bird"

  # The CNI network configuration to install on each node.
  cni_network_config: |-
    {
        "name": "k8s-pod-network",
        "cniVersion": "0.1.0",
        "type": "calico",
        "etcd_endpoints": "__ETCD_ENDPOINTS__",
        "etcd_key_file": "__ETCD_KEY_FILE__",
        "etcd_cert_file": "__ETCD_CERT_FILE__",
        "etcd_ca_cert_file": "__ETCD_CA_CERT_FILE__",
        "log_level": "info",
        "mtu": 1500,
        "ipam": {
            "type": "calico-ipam"
        },
        "policy": {
            "type": "k8s",
            "k8s_api_root": "https://__KUBERNETES_SERVICE_HOST__:__KUBERNETES_SERVICE_PORT__",
            "k8s_auth_token": "__SERVICEACCOUNT_TOKEN__"
        },
        "kubernetes": {
            "kubeconfig": "__KUBECONFIG_FILEPATH__"
        }
    }

  # If you're using TLS enabled etcd uncomment the following.
  # You must also populate the Secret below with these files.
  etcd_ca: ""   # "/calico-secrets/etcd-ca"
  etcd_cert: "" # "/calico-secrets/etcd-cert"
  etcd_key: ""  # "/calico-secrets/etcd-key"

---

# The following contains k8s Secrets for use with a TLS enabled etcd cluster.
# For information on populating Secrets, see http://kubernetes.io/docs/user-guide/secrets/
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: calico-etcd-secrets
  namespace: kube-system
data:
  # Populate the following files with etcd TLS configuration if desired, but leave blank if
  # not using TLS for etcd.
  # This self-hosted install expects three files with the following names.  The values
  # should be base64 encoded strings of the entire contents of each file.
  # etcd-key: null
  # etcd-cert: null
  # etcd-ca: null

---

# This manifest installs the calico/node container, as well
# as the Calico CNI plugins and network config on
# each master and worker node in a Kubernetes cluster.
kind: DaemonSet
apiVersion: extensions/v1beta1
metadata:
  name: calico-node
  namespace: kube-system
  labels:
    k8s-app: calico-node
spec:
  selector:
    matchLabels:
      k8s-app: calico-node
  template:
    metadata:
      labels:
        k8s-app: calico-node
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
        scheduler.alpha.kubernetes.io/tolerations: |
          [{"key": "dedicated", "value": "master", "effect": "NoSchedule" },
           {"key":"CriticalAddonsOnly", "operator":"Exists"}]
    spec:
      hostNetwork: true
      serviceAccountName: calico-node
      containers:
        # Runs calico/node container on each Kubernetes node.  This
        # container programs network policy and routes on each
        # host.
        - name: calico-node
          image: quay.io/calico/node:v2.5.1
          env:
            # The location of the Calico etcd cluster.
            - name: ETCD_ENDPOINTS
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_endpoints
            # Choose the backend to use.
            - name: CALICO_NETWORKING_BACKEND
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: calico_backend
            # Cluster type to identify the deployment type
            - name: CLUSTER_TYPE
              value: "k8s,bgp"
            # Disable file logging so `kubectl logs` works.
            - name: CALICO_DISABLE_FILE_LOGGING
              value: "true"
            # Set Felix endpoint to host default action to ACCEPT.
            - name: FELIX_DEFAULTENDPOINTTOHOSTACTION
              value: "ACCEPT"
            # Configure the IP Pool from which Pod IPs will be chosen.
            - name: CALICO_IPV4POOL_CIDR
              value: "192.168.0.0/16"
            - name: CALICO_IPV4POOL_IPIP
              value: "always"
            # Disable IPv6 on Kubernetes.
            - name: FELIX_IPV6SUPPORT
              value: "false"
            # Set Felix logging to "info"
            - name: FELIX_LOGSEVERITYSCREEN
              value: "info"
            # Set MTU for tunnel device used if ipip is enabled
            - name: FELIX_IPINIPMTU
              value: "1440"
            # Location of the CA certificate for etcd.
            - name: ETCD_CA_CERT_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_ca
            # Location of the client key for etcd.
            - name: ETCD_KEY_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_key
            # Location of the client certificate for etcd.
            - name: ETCD_CERT_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_cert
            # Auto-detect the BGP IP address.
            - name: IP
              value: ""
            - name: FELIX_HEALTHENABLED
              value: "true"
          securityContext:
            privileged: true
          resources:
            requests:
              cpu: 250m
          livenessProbe:
            httpGet:
              path: /liveness
              port: 9099
            periodSeconds: 10
            initialDelaySeconds: 10
            failureThreshold: 6
          readinessProbe:
            httpGet:
              path: /readiness
              port: 9099
            periodSeconds: 10
          volumeMounts:
            - mountPath: /lib/modules
              name: lib-modules
              readOnly: true
            - mountPath: /var/run/calico
              name: var-run-calico
              readOnly: false
            - mountPath: /calico-secrets
              name: etcd-certs
        # This container installs the Calico CNI binaries
        # and CNI network config file on each node.
        - name: install-cni
          image: quay.io/calico/cni:v1.10.0
          command: ["/install-cni.sh"]
          env:
            # The location of the Calico etcd cluster.
            - name: ETCD_ENDPOINTS
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_endpoints
            # The CNI network config to install on each node.
            - name: CNI_NETWORK_CONFIG
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: cni_network_config
          volumeMounts:
            - mountPath: /host/opt/cni/bin
              name: cni-bin-dir
            - mountPath: /host/etc/cni/net.d
              name: cni-net-dir
            - mountPath: /calico-secrets
              name: etcd-certs
      volumes:
        # Used by calico/node.
        - name: lib-modules
          hostPath:
            path: /lib/modules
        - name: var-run-calico
          hostPath:
            path: /var/run/calico
        # Used to install CNI.
        - name: cni-bin-dir
          hostPath:
            path: /opt/cni/bin
        - name: cni-net-dir
          hostPath:
            path: /etc/cni/net.d
        # Mount in the etcd TLS secrets.
        - name: etcd-certs
          secret:
            secretName: calico-etcd-secrets

---

# This manifest deploys the Calico policy controller on Kubernetes.
# See https://github.com/projectcalico/k8s-policy
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: calico-policy-controller
  namespace: kube-system
  labels:
    k8s-app: calico-policy
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ''
    scheduler.alpha.kubernetes.io/tolerations: |
      [{"key": "dedicated", "value": "master", "effect": "NoSchedule" },
       {"key":"CriticalAddonsOnly", "operator":"Exists"}]
spec:
  # The policy controller can only have a single active instance.
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      name: calico-policy-controller
      namespace: kube-system
      labels:
        k8s-app: calico-policy
    spec:
      # The policy controller must run in the host network namespace so that
      # it isn't governed by policy that would prevent it from working.
      hostNetwork: true
      serviceAccountName: calico-policy-controller
      containers:
        - name: calico-policy-controller
          image: quay.io/calico/kube-policy-controller:v0.7.0
          env:
            # The location of the Calico etcd cluster.
            - name: ETCD_ENDPOINTS
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_endpoints
            # Location of the CA certificate for etcd.
            - name: ETCD_CA_CERT_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_ca
            # Location of the client key for etcd.
            - name: ETCD_KEY_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_key
            # Location of the client certificate for etcd.
            - name: ETCD_CERT_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_cert
            # The location of the Kubernetes API.  Use the default Kubernetes
            # service for API access.
            - name: K8S_API
              value: "https://kubernetes.default:443"
            # Since we're running in the host namespace and might not have KubeDNS
            # access, configure the container's /etc/hosts to resolve
            # kubernetes.default to the correct service clusterIP.
            - name: CONFIGURE_ETC_HOSTS
              value: "true"
          volumeMounts:
            # Mount in the etcd TLS secrets.
            - mountPath: /calico-secrets
              name: etcd-certs
      volumes:
        # Mount in the etcd TLS secrets.
        - name: etcd-certs
          secret:
            secretName: calico-etcd-secrets

---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: calico-policy-controller
  namespace: kube-system

---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: calico-node
  namespace: kube-system
```

- 修改`etcd_endpoints`字段为实际的etcd地址
- 对于tls的etcd需要修改以下配置

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: calico-config
  namespace: kube-system
data:
  etcd_endpoints: "https://172.168.0.25:2379"
  ......
  etcd_ca: "/calico-secrets/etcd-ca"
  etcd_cert: "/calico-secrets/etcd-cert"
  etcd_key: "/calico-secrets/etcd-key"

......

apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: calico-etcd-secrets
  namespace: kube-system
data:
  # 通过base64编码后的值
  etcd-key: ""
  etcd-cert: ""
  etcd-ca: ""
```
*注意*： 通过base64编码可以通过命令获取
> kubectl create secret generic helloworld-tls --from-file=/etc/kubernetes/ssl/kubernetes.pem --from-file=/etc/kubernetes/ssl/kubernetes-key.pem --from-file=/etc/kubernetes/ssl/ca.pem
> kubectl get secret helloworld-tls -o yaml
> 对于kubernetes宿主机为虚机时，使用calico ipip模式的通信时pod间网络可能不通，需要主机虚机的安全组规则是否允许ip协议为ipip的通过（ipv4.proto == 4）

## 安装addon组件
### 安装kube-dns
在kubernetes目录`kubernetes/cluster/addons/dns`下有kube-dns的yaml部署文件，只需要关注kubedns-controller.yaml.sed, kubedns-svc.yaml.sed, kubedns-sa.yaml, kubedns-cm.yaml四个文件。从`kubernetes/centos/deployAddons.sh`脚本可以看出, 修改yaml.sed文件中的DNS_DOMAIN和DNS_SERVER_IP为实际的值后，即可以直接部署kube-dns

```
  sed -e "s/\\\$DNS_DOMAIN/${DNS_DOMAIN}/g" "${KUBE_ROOT}/cluster/addons/dns/kubedns-controller.yaml.sed" > kubedns-controller.yaml
  sed -e "s/\\\$DNS_SERVER_IP/${DNS_SERVER_IP}/g" "${KUBE_ROOT}/cluster/addons/dns/kubedns-svc.yaml.sed" > kubedns-svc.yaml
  KUBEDNS=`eval "${KUBECTL} get services --namespace=kube-system | grep kube-dns | cat"`

  if [ ! "$KUBEDNS" ]; then
    # use kubectl to create kube-dns deployment and service
    ${KUBECTL} --namespace=kube-system create -f kubedns-sa.yaml
    ${KUBECTL} --namespace=kube-system create -f kubedns-cm.yaml
    ${KUBECTL} --namespace=kube-system create -f kubedns-controller.yaml
    ${KUBECTL} --namespace=kube-system create -f kubedns-svc.yaml

    echo "Kube-dns deployment and service is successfully deployed."

```

- 注意DNS_DOMAIN和DNS_SERVER_IP的值需要跟kubelet的配置项`--cluster-domain`和`--cluster-dns`保持一致。

### 安装dashboard
在`kubernetes/cluster/addons/dashboard`目录下有部署dashboard的yaml文件，直接部署即可，另外除了`dashboard-controller.yaml`和`dashboard-service.yaml`这两个文件，还应该增加rbac的yaml文件，否则dashboard启动时会报权限的错误

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard
  namespace: kube-system

---

kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: dashboard
subjects:
  - kind: ServiceAccount
    name: dashboard
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```
### 安装heapster

- 从git库上下载heapser
- 在`heapster/deploy/kube-config/influxdb`目录修改heapster.yaml和grafana.yaml文件service的类型为NodePort用于通过nodeport方式访问
- 执行`kubectl create -f .`
- 在`heapster/deploy/kube-config/rbac`目录下执行`kubectl create -f heapster-rbac.yaml`

### 安装ingress-nginx Controller 

直接按照官网文档上直接安装即可 https://github.com/kubernetes/ingress-nginx/tree/master/deploy

```
curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/namespace.yaml | kubectl apply -f -
curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/default-backend.yaml | kubectl apply -f -
curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/configmap.yaml | kubectl apply -f -
curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/tcp-services-configmap.yaml | kubectl apply -f -
curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/udp-services-configmap.yaml | kubectl apply -f -
curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/rbac.yaml | kubectl apply -f -
curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/with-rbac.yaml | kubectl apply -f -
使用NodePort方式访问
curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/baremetal/service-nodeport.yaml | kubectl apply -f -
```

创建ingress资源

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: guestbook
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/add-base-url: "true"
spec:
  rules:
  - host: k8s.test
    http:
      paths:
      - path: /aa
        backend:
          serviceName: frontend
          servicePort: 80
      - path: /nginx
        backend:
          serviceName: nginx
          servicePort: 80
```

有些情况，访问ingress的资源时报出404错误，需要在ingress的yaml中加上

```
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/add-base-url: "true"
```

具体可见https://github.com/kubernetes/ingress-nginx/blob/master/docs/examples/rewrite/README.md

