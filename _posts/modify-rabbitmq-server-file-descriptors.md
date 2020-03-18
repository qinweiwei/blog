---
title: rabbitmq file descriptors调整
date: 2018-12-18
updated: 2018-12-18
tags:
  - ulimit
  - rabbitmq-server
---

# 记一次rabbitmq server的file descriptors调整记录
在生产环境的openstack集群中发现rabbitmq server的File descriptors和Socket descriptors的值只有4096，而且total_used和sockets_used也快达到上限，为了保证rabbitmq server正常工作，需要调整一下total_limit和sockets_limit的值

1. 通过ulimit -a 查看root用户的open files值为65535,
2. 通过 su - rabbitmq -s /bin/bash -c 'ulimit -a' 命令查看rabbitmq用户的open files值1024
3. 两者值都有些小，调整/etc/security/limit.conf 文件后，通过ulimit -a 命令发现open files值没有修改成功，退出从新登录后仍然不行
<!-- more -->

```
*       soft    core    unlimited
*       hard    core    unlimited
*       soft    nproc   unlimited
*       hard    nproc   unlimited
*       soft    nofile  1024000
*       hard    nofile  1024000

```

4. 怀疑跟pam文件有关，修改/etc/pam.d/login文件，增加`session    required     pam_limits.so`，修改/etc/ssh/sshd_config文件，修改UsePAM=yes后，并重启sshd服务，重新登录后ulimit -a查看open files值仍然不对
5. 怀疑是在某些文件中写死了ulimit -n 60000的命令，查看/etc/profile文件后，发现在该文件中存在命令 ulimit -n 60000，导致每次登录后open files都被重新修改为60000，去掉该命令后 open files值被修改为1024000
6. 执行systemctl restart rabbitmq-server后通过`rabbitmqctl status`命令查看total_limit和socket_limit值仍然为4096
7. 查找相关资料，systemd进程会忽略limit.conf文件，需要手动创建目录*/etc/systemd/system/rabbitmq-server.service.d/*，然后在该目录下新建文件*limit.conf*，在该文件中添加以下字段


```
[Service]
LimitNOFILE=300000

```

8. 执行以下命令后，查看rabbitmq server的total_limit和socket_limit值仍然为4096

```
systemctl daemon-reload
systemctl restart rabbitmq-server
```

9. 手动执行rabbitmq-server -detached后，查看total_limit和socket_limit值仍然为4096，所以应该跟systemd无关
10.  继续定位，发现rabbitmq-server命令后调用/usr/lib/rabbitmq/bin/rabbitmq-server脚本去启动rabbitmq，在/usr/lib/rabbitmq/bin/rabbitmq-server脚本中发现存在ulimit -n 4096命令，去掉后该问题得以解决。

```
start_rabbitmq_server() {
    ensure_thread_pool_size
    ulimit -n 4096
    check_start_params &&
    RABBITMQ_CONFIG_FILE=$RABBITMQ_CONFIG_FILE \
    exec ${ERL_DIR}erl \
    ...
```
