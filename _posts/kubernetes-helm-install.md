---
title: helm安装过程记录
date: 2018-03-06
updated: 2018-03-06 16:08:00
tags:
    - kubernetes
---

# helm安装过程记录
## helm安装

helm的安装比较简单，直接从`https://github.com/kubernetes/helm/releases`上下载helm可执行文件放到*PATH*目录下，然后执行以下命令，初始化helm client并安装helm server： Tiller到kubernetes集群中
> helm init

注意在执行`helm init`时helm client会访问默认的repo地址，如果helm所在环境不能访问默认的repo地址`https://kubernetes-charts.storage.googleapis.com`则tiller安装会失败，此时可以通过增加参数`--stable-repo-url`修改默认的repo地址
<!-- more -->
## Monocular的安装

- Helm和Tiller安装
- Nginx Ingress Controller安装
- 执行命令`helm repo add monocular https://kubernetes-helm.github.io/monocular`，增加monoculaer的repo
- 执行命令`helm install test/monocular --name monocular -f custom.yaml --wait`,其中custom.yaml文件内容如下：

```
ingress:
  hosts:
    - foo.bar.com
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/add-base-url: "true"
mongodb:
  persistence:
    enabled: false

api:
  config:
    repos:
      #- name: monocular
      #  url: https://kubernetes-helm.github.io/monocular
      #  source: https://github.com/kubernetes-helm/monocular/tree/master/charts
      - name: stable
        url: http://10.18.74.203:8000
        source: http://10.18.74.203:8000
```

helm UI monoculaer安装遇到的问题：
1. chart页面刷新不出来： 需要确认monoculaer api服务是否可以访问外网，访问chart时需要向chart的source地址下载ico文件
2. 执行deployment时报错，是因为repo 缺少source原因造成，具体可见https://www.bountysource.com/issues/47813516-error-if-repo-source-missing，repos需要增加source属性

```
Subscriber.js:227 Uncaught TypeError: Cannot read property 'match' of undefined
    at t.maintainerUrl (chart-details-info.component.ts:53)
    at e.detectChangesInternal (chart-details-info.component.ngfactory.ts:313)
    at e.t.detectChanges (view.js:425)
    at t.detectChangesInNestedViews (view_container.js:67)
    at e.detectChangesInternal (chart-details-info.component.ngfactory.ts:607)
    at e.t.detectChanges (view.js:425)
    at e.t.internalDetectChanges (view.js:410)
    at e.detectChangesInternal (chart-details.component.ngfactory.ts:433)
    at e.t.detectChanges (view.js:425)
    at t.detectChangesInNestedViews (view_container.js:67)
    at e.detectChangesInternal (chart-details.component.ngfactory.ts:551)
    at e.t.detectChanges (view.js:425)
    at e.t.internalDetectChanges (view.js:410)
    at e.detectChangesInternal (chart-details.component.ngfactory.ts:95)
    at e.t.detectChanges (view.js:425)
```

