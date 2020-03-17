---
title: 使用hexo，github pages部署博客
date: 2017-12-15
updated: 2018-03-13
tags:
  - hexo
  - github pages
---
使用github pages，hexo部署博客记录

<!-- more -->
# 环境安装
安装node.js和git

# 注册github

1. 注册github
2. 创建代码库
创建repo:username.github.io
3. 开启gh-pages功能
在setting点击 ` automatic page generator ` ,如果配置没有问题的话，那么大约15分钟后，`username.github.io`就可以访问了

# 安装hexo
## hexo初始安装
在cmd命令行中输入
```
npm install hexo-cli -g
npm install hexo
```

执行以下命令检查hexo是否安装完成
```
hexo -v
```

## 初始化hexo
在cmd命令行中输入
```
hexo init
npm install
```
之后npm将会自动安装你需要的组件，只需要等待npm操作即可。

## 基本的hexo命令
```
hexo g #生成html文件
hexo s #启动预览hexo server,通过浏览器访问http://localhost:4000访问
hexo d #部署html到远程github
```

## 配置hexo
可以在_config.yml文件中修改大部分的配置
具体可以见[hexo官网](https://hexo.io/zh-cn/docs/configuration.html)
常用配置如下：
>title: 青柠檬老爸的blog #网站主题
subtitle: openstacker   #网站子主题
description: 总以为可以慢慢长大。。。 #个人描述
author: Eric Qin        #作者
language: zh-Hans       #语言
timezone:
theme: next             #主题，推荐使用next
deploy:                 #远端git仓库
  type: git
  repo: git@github.com:qinweiwei/qinweiwei.github.io.git
  branch: master
avatar: /images/lemon.jpg #作者头像

主题的相关配置具体可见[next官网](http://theme-next.iissnan.com/)

## 添加新文章
1. 在hexo的source文件夹下，所有文章都会保存到`_post`文件夹下，只要在`_post`文件下新建*md*类型的文件即可。
2. 执行` hexo g `进行渲染
3. 在新建的文章头需要添加yml信息
>---
title: 使用hexo，github pages部署博客
tags:
  - hexo
  - github pages
---

## 注意事项:
1. 默认next没有分类和标签，需要执行
```
hexo new page tags
hexo new page categories
```

 1.	Next的主题配置文件中配置相应的menu和menu_icons
 2. 执行 hexo new page tags 或 hexo new page categories 创建标签页或分类页,会在source文件夹下生成tags或categories目录,目录下有index.md文件
 3. 编辑index.md文件
![1.png](https://i.loli.net/2020/03/17/jdDOCIb1voPrtc9.png)

2. 如何设置「阅读全文」？
在文章中使用 ` <!-- more --> ` 手动进行截断，Hexo 提供的方式

## 遇到的问题
1. 执行`hexo g`时报以下错误，是由于post目录下的md文件中可能存在一些与md冲突的字符，比如单一的`*`，`{{`，`}}`等，对于这类的字符最好通过引用的方式使用

```
INFO  Start processing
FATAL Something's wrong. Maybe you can find the solution here: http://hexo.io/docs/troubleshooting.html
Template render error: (unknown path) [Line 6, Column 8]
  unexpected token: /
    at Object.exports.prettifyError (D:\work\workspace\hexo\blog\node_modules\hexo\node_modules\nunjucks\src\lib.js:34:15)
    at Obj.extend.render (D:\work\workspace\hexo\blog\node_modules\hexo\node_modules\nunjucks\src\environment.js:469:27)
    at Obj.extend.renderString (D:\work\workspace\hexo\blog\node_modules\hexo\node_modules\nunjucks\src\environment.js:327:21)
    at D:\work\workspace\hexo\blog\node_modules\hexo\lib\extend\tag.js:66:9
    at Promise._execute (D:\work\workspace\hexo\blog\node_modules\hexo\node_modules\bluebird\js\release\debuggability.js:299:9)
    at Promise._resolveFromExecutor (D:\work\workspace\hexo\blog\node_modules\hexo\node_modules\bluebird\js\release\promise.js:481:18)
    at new Promise (D:\work\workspace\hexo\blog\node_modules\hexo\node_modules\bluebird\js\release\promise.js:77:14)
    at Tag.render (D:\work\workspace\hexo\blog\node_modules\hexo\lib\extend\tag.js:64:10)

```



