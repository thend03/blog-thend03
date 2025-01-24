---
draft: false
date: 2024-02-06T13:07:38+08:00
title: "笔记本安装manjaro+windows双系统"
slug: "dual-system-installation-manjaro-and-windows" 
tags: ["manajaro"]
categories: ["linux","manjaro"]
authors: ["since"]
description: "如何在笔记本电脑上安装双系统"
---

## 背景

手上有一台windows，想着再机器上安装一个双系统，刚好之前折腾过一次manjaro，这次安装双系统还是想尝试一下manjaro。

manjaro是基于arch的linux发行版，arch命令行操作比较多，manjaro在此之上，提供了友好的图形界面操作，更加适合新手。

选它一个是界面美观，另外几个比如ubuntu/centos，在服务端经常接触，所以桌面版就想尝尝鲜。

manjaro官方的桌面版分为3类: GNOME、KDE、XFCE。

下载地址如下: https://manjaro.org/download/

我选择的是manjaro kde，也就是下图的plasma desktop, 其他2个版本可以通过如上链接点击具体的版本了解

![image-20240206132600195](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402061326257.png)

## 制作启动盘

要安装manjaro的话，得先制作一个启动盘，需要一个U盘, 最好是16G以上的。然后在windows系统使用rufuse这款工具，选择dd模式进行写入

### 下载rufus

rufus是开源免费的USB启动盘制作工具，在官网和github下载即可，按照机器的配置下载对应的exe文件

官网下载地址:  https://rufus.ie/zh/

github下载地址: https://github.com/pbatard/rufus/releases/

### 下载manjaro镜像

进入manjaro下载页: https://manjaro.org/download/

选择想要安装的版本，我这里选择plasma desktop，即kde版本, 点击download, 选择image即可进行下载

![image-20240206133614533](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402061336556.png)

![image-20240206133654393](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402061336421.png)



### U盘格式化

在制作启动盘之前，先将U盘进行格式化，避免安装失败

### 制作启动盘

启动rufus，启动之后界面类似于如下这样，这个是从官网搂的图，写入模式需要选择以DD镜像模式写入

设备是指U盘，引导类型是你的iso镜像文件，其他的按照程序默认即可

状态为准备就绪，就代表启动盘做好了，不要再点开始了，那样会重新做盘，可能会产生未知的错误

![Rufus screenshot 1](https://rufus.ie/pics/screenshot1_zh_CN.png)



## 磁盘分区

做好启动盘之后，就得把机器的磁盘分一块出来用于安装manjaro。

这个涉及到windows磁盘管理。使用windows磁盘管理，整一个新加卷主分区出来，至少保留100G给manjaro。

具体新建磁盘分区可以自行搜索。不再赘述。



## BIOS设置

根据机器型号的不同，使用不同的快捷键，进入BIOS, 关闭secure boot

然后选择U盘镜像进行安装启动

## 安装过程

安装时语言选择英文，这样会避免home目录是中文的问题，可以安装好之后系统语言再设置为中文。

安装时选择分区的时候，切忌要选对硬盘，不然可能会有抹除windows系统盘的风险

然后按照安装指引进行安装即可，不清楚的可以选择默认，直至安装完成。



## 参考链接

[惠普战66安装win10+Manjaro双系统](https://www.jianshu.com/p/f7e97f7fb7c8)
