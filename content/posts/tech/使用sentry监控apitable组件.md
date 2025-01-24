---
draft: false
date: 2024-01-19T14:37:22+08:00
title: "使用sentry监控apitable"
slug: "use-sentry-to-monitor-apitable" 
tags: ["apitable","sentry"]
authors: ["since"]
description: "使用sentry监控apitable"
---

## 背景

这篇是apitable的第3篇，如何解决子进程监控。

apitable代码地址: https://github.com/apitable/apitable

apitable里面是使用sentry进行监控上报的，那么需要自己部署一个sentry服务端



## 搭建sentry服务端

sentry部署的话分为3种方式，一种是使用docker-compose部署，一种是在k8s部署，还有一种是手动安装部署。

本文只介绍docker-compose和k8s部署，手动安装参考官方文档

### 使用docker-compose部署

Sentry-self-hosted代码地址: https://github.com/getsentry/self-hosted

下载之后执行./install.sh就可以自动安装sentry了。

其他配置可参考项目里的readme

这个本地安装会比较简单，但是测试环境和生产环境因为网络原因依赖老是下载不了，网络隔离实在是太费劲。

### 使用k8s部署

docker-compose可以搭建一个可用的sentry服务端，但是高可用无法保证

那么就考虑在k8s上搭建，有个项目做了一个相当于一键安装部署的东西，叫helm charts。

Sentry helm charts代码地址如下: https://github.com/sentry-kubernetes/charts

这个使用起来也比较简单

#### 命令行安装

首先是打开命令行，执行如下语句，关联sentry charts仓库

```shell
helm repo add sentry https://sentry-kubernetes.github.io/charts
```

然后在命令行执行install语句

```shell
helm install --namespace sentry --timeout 20m sentry sentry/sentry
```

然后看下命令行执行结果，运气好的话一次就能成功，运气好，会失败，我这边失败的原因是由于pod比较多，并且sentry会起很多个

pod，导致资源不足，sentry无法完整的启动。部署失败的时候需要注意下资源问题。

#### 页面安装

如果k8s集群带页面的话，也可以在页面进行安装

我这边使用的是rancher管理页面

先到应用市场，先添加仓库

![image-20240119150229542](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202401191502591.png)



添加好之后在列表就有一个新的仓库 sentry

![image-20240119150335661](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202401191503686.png)

然后到charts页签，搜索sentry, 选择sentry执行安装

![image-20240119150426943](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202401191504969.png)

右侧可以选择版本，选完版本之后点击install

![image-20240119150603882](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202401191506907.png)





![image-20240119151305011](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202401191513047.png)



这一步配置yaml，使用默认的即可，配置文件比较大，想改好需要高深的功力

![image-20240119153825685](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202401191538722.png)



超时时间设置的长一点，免得安装超时失败，600s可能会不够



![image-20240119154006296](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202401191540322.png)

然后点击右下角install，等待安装结束即可。



最终安装成功之后，在应用市场那显示有一个已安装app

![image-20240119154133431](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202401191541464.png)

最终部署一个sentry，部署了37个deployment，5个Job, 8个statefulsets ,一共起了62个pod，吃的资源还是比较多

9.x的版本吃的资源会少一点，大概起34个pod

![image-20240119154223771](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202401191542809.png)
