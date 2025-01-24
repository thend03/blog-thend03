---
draft: false
date: 2023-09-14T14:59:12+08:00
title: "heap-oom"
slug: "heap-oom" 
tags: ["oom","heap"]
authors: ["since"]
description: "heap oom"
---

## 问题

测试环境的某台机器运行一段时间后发生了oom，类型是heap oom,即堆内存溢出，用不了了，然后由于配置了`-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/logs/`，在发生oom后，会自动生成dump文件，所以可以拿着dump进行分析，进程先重启恢复使用。



## 前置信息

java进程的JVM参数如下,堆内存2g，其中老年代1g，新生代1g

```
-Xmx2048M -Xms2048M -Xmn1024M 
-XX:MaxMetaspaceSize=512M -XX:MetaspaceSize=512M 
-XX:+UseConcMarkSweepGC -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=70 -XX:+ExplicitGCInvokesConcurrentAndUnloadsClasses -XX:+CMSClassUnloadingEnabled -XX:+ParallelRefProcEnabled -XX:+CMSScavengeBeforeRemark 
-XX:+HeapDumpOnOutOfMemoryError -XX:ErrorFile=/data/logs/hs_err_pid%p.log -Xloggc:/dev/shm/gc.log -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=2 -XX:GCLogFileSize=16M 
-XX:HeapDumpPath=/data/logs/ -XX:+PrintGCDetails -XX:+PrintGCDateStamps 
-XX:+PrintClassHistogramBeforeFullGC -XX:+PrintClassHistogramAfterFullGC 
-XX:+PrintCommandLineFlags -XX:+PrintGCApplicationConcurrentTime 
-XX:+PrintGCApplicationStoppedTime -XX:+PrintTenuringDistribution -XX:+PrintHeapAtGC
```





## 排查过程

### MAT

使用mat分析dump文件

选择leak suspect，检测下内存泄露

![image-20230914171847827](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202309141718863.png)



### Overview

然后会给一个基本的报告，这个报告就比较明显了，dump文件1.9个g，其中1.8g是problem suspect1，而下面problem suspect1全是

**java.lang.Thread**对象，413个线程占了1.8g内存，平均每个线程占用了4m的内存，看线程名称的话，初步分析是rocketmq消费的时候出了问题



![image-20230914172320532](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202309141723560.png)



### histogram

直方图以类为维度，展示了类有多少个实例，shallow heap和retained heap的内存大小

shallow heap翻译过来是浅堆，就是类对象自身占用的内存，retained heap翻译过来就是深堆，就是类对象本身以及对象的成员变量占用的内存

看如下的直方图，可以看到2g的对，排名第一的java.lang.StackTraceElement这个类，对象数是29344740个，2900万个，然后shallow heap和retained heap大概是895m，占了一半的堆内存，问题可能就出现在这了。继续往下看

![image-20230915084604031](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202309150846059.png)

### dominator tree

下一步就是看一下支配树这一项了，会按照内存占用比例从高到低展示对象排序。

支配树内存占用排名前32位全都是java.lang.Thread,每个线程内存占用量按照55m算，55*32=1760m,大概占了堆的85%+，和overview给出的数据基本一致

这32个线程都是rocketmq的消费线程，那应该是消费出问题了，继续点开线程查看详情

![image-20230915085322575](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202309150853611.png)

展开第一个线程详情之后，可以看到具体的内存占用来源了，thread_local持有的map占用了所有的内存，层层展开之后，可以看到每个key只有0.01m内存占用，但是数量比较多，一共有3874个entry，折合下来38m，差不多都是org.mybatis.spring.MyBatisSystemException这种异常造成的了

![image-20230915090508790](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202309150905838.png)

复制下cause信息看下

```
Data too long for column 'name' at row 1
```

堆栈信息如下，其他的线程也是类似，那可以得到如下结论：mq消费的时候，由于mybatis执行更新发生异常，cat的context持有了异常信息，异常过多导致了oom

![image-20230915092353441](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202309150923502.png)

## 小结

本次的oom原因比较直观，基本上看下overview,histogram,dominator tree就可以得出结论了

修复也比较简单，要么限制下入库时name字段的长度，要么将name字段的长度调大一点就好了
