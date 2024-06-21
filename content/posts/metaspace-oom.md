---
draft: false
date: 2023-09-07T09:47:33+08:00
title: "metaspace-oom"
slug: "metaspace-oom" 
tags: ["jvm","oom"]
authors: ["since"]
description: "metaspace oom"
---

## 问题背景

生产环境有机器产生了oom，一开始怀疑是heap oom，因为之前配置了发生oom，自动进行dump，所以分析了dump文件之后，发现4g的heap，最终dump只有300多m，没有分析出有用的内容。

后来又一次oom之后，终于在容器控制台发现了错误根因，是Metaspace发生了oom。

所以着重分析下Metaspace为什么会oom。

## 前置知识

### 启动参数

先看下启动参数，看下JVM具体的参数配置

```
-Xmx4096M -Xms4096M -Xmn2048M 
-XX:MaxMetaspaceSize=512M -XX:MetaspaceSize=512M 
-XX:+UseConcMarkSweepGC -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=70 
-XX:+ExplicitGCInvokesConcurrentAndUnloadsClasses -XX:+CMSClassUnloadingEnabled 
-XX:+ParallelRefProcEnabled -XX:+CMSScavengeBeforeRemark 
-XX:+HeapDumpOnOutOfMemoryError -XX:ErrorFile=/data/logs/hs_err_pid%p.log -Xloggc:/dev/shm/gc.log 
-XX:HeapDumpPath=/data/logs/ 
-XX:+PrintGCDetails -XX:+PrintGCDateStamps 
-XX:+PrintClassHistogramBeforeFullGC -XX:+PrintClassHistogramAfterFullGC 
-XX:+PrintCommandLineFlags -XX:+PrintGCApplicationConcurrentTime 
-XX:+PrintGCApplicationStoppedTime -XX:+PrintTenuringDistribution 
-XX:+PrintHeapAtGC -Ddubbo.reference.check=false 
-XX:+UnlockDiagnosticVMOptions
```

这个是公司默认的jvm参数，可以看到，堆最大是4g，metaspace最大是512m，metaspace给的空间已经很大了，但还是发生了oom。

关于metaspace的大小的配置项，我们后面再提一下具体含义。



### Metaspace里有什么

分析Metaspace oom之前，先确认下metaspace里会有哪些东西。jdk1.8相比1.7，移除了perm gen，把class,method,constant pool等内容移到了metaspace，单独的一块区域，和机器内存直接相关，不受堆管控。



在底层，MetaSpace 主要由 Klass Metaspace 和 NoKlass Metaspace 两大部分组成。

Klass MetaSpace： 就是用来存 Klass 的，就是 Class 文件在 JVM 里的运行时数据结构，这部分默认放在 Compressed Class Pointer Space 中，是一块连续的内存区域，紧接着 Heap。

Compressed Class Pointer Space 不是必须有的，如果设置了 -XX:-UseCompressedClassPointers，或者 -Xmx 设置大于 32 G，就不会有这块内存，这种情况下 Klass 都会存在 NoKlass Metaspace 里。

NoKlass MetaSpace： 专门来存 Klass 相关的其他的内容，比如 Method，ConstantPool 等，可以由多块不连续的内存组成。虽然叫做 NoKlass Metaspace，但是也其实可以存 Klass 的内容，上面已经提到了对应场景。



MetaSpace 的对象为什么无法释放，我们看下面两点

- **MetaSpace 内存管理：** 类和其元数据的生命周期与其对应的类加载器相同，只要类的类加载器是存活的，在 Metaspace 中的类元数据也是存活的，不能被回收。每个加载器有单独的存储空间，通过 ClassLoaderMetaspace 来进行管理 SpaceManager* 的指针，相互隔离的。
- **MetaSpace 弹性伸缩：** 由于 MetaSpace 空间和 Heap 并不在一起，所以这块的空间可以不用设置或者单独设置，一般情况下避免 MetaSpace 耗尽 VM 内存都会设置一个 MaxMetaSpaceSize，在运行过程中，如果实际大小小于这个值，JVM 就会通过 `-XX:MinMetaspaceFreeRatio` 和 `-XX:MaxMetaspaceFreeRatio` 两个参数动态控制整个 MetaSpace 的大小，这个方法会在 CMSCollector 和 G1CollectorHeap 等几个收集器执行 GC 时调用。这个里面会根据 `used_after_gc`，`MinMetaspaceFreeRatio` 和 `MaxMetaspaceFreeRatio` 这三个值计算出来一个新的 `_capacity_until_GC` 值（水位线）。然后根据实际的 `_capacity_until_GC` 值使用 `MetaspaceGC::inc_capacity_until_GC()` 和 `MetaspaceGC::dec_capacity_until_GC()` 进行 expand 或 shrink。



为了避免弹性伸缩带来的额外 GC 消耗，一般要将 `-XX:MetaSpaceSize` 和 `-XX:MaxMetaSpaceSize` 两个值设置为固定的，但是这样也会导致在空间不够的时候无法扩容，然后频繁地触发 GC，最终 OOM。

所以关键原因就是 ClassLoader 不停地在内存中 load 了新的 Class ，一般这种问题都发生在动态类加载等情况上。



可能出现问题的点有如下几个

- groovy动态加载类
- 反射使用不当频繁创建类
-  Orika 的 classMap
- JSON 的 ASMSerializer

基本都集中在反射、Javasisit 字节码增强、CGLIB 动态代理、OSGi 自定义类加载器等的技术点上。



### 排查手段

有了上面的背景之后，我们排查方向分为2部分，一部分是看发版记录，最近有哪些改动，哪些地方可能会频繁创建类，二是从jvm层面排查。我们重点说一下jvm层面的排查方向。

***注意事项: 写这篇文章的时候，问题已经解决，所以部分数据不是出问题时采集的数据，是为了写文章使用对应的排查工具生成的数据，具体位置处会标明数据来源和时机***



### JVM排查

jdk自带的排查工具，我们可以用的有jps,jstat,jmap等

其中jmap看的是堆的内存大小和分配

jstat可以看gc相关信息.

gc.log里的gc日志也可以辅助我们排查

#### jmap

jmap -heap pid的信息如下，这个可以辅助我们看下堆内分代的内存使用情况，当然这个对我们的排查帮助不大，下面看一下命令效果

```
Attaching to process ID 5311, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.73-b02

using parallel threads in the new generation.
using thread-local object allocation.
Concurrent Mark-Sweep GC

Heap Configuration:
   MinHeapFreeRatio         = 40
   MaxHeapFreeRatio         = 70
   MaxHeapSize              = 4294967296 (4096.0MB)
   NewSize                  = 2147483648 (2048.0MB)
   MaxNewSize               = 2147483648 (2048.0MB)
   OldSize                  = 2147483648 (2048.0MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 536870912 (512.0MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 536870912 (512.0MB)
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
New Generation (Eden + 1 Survivor Space):
   capacity = 1932787712 (1843.25MB)
   used     = 735313536 (701.2496337890625MB)
   free     = 1197474176 (1142.0003662109375MB)
   38.04419551276617% used
Eden Space:
   capacity = 1718091776 (1638.5MB)
   used     = 733845856 (699.8499450683594MB)
   free     = 984245920 (938.6500549316406MB)
   42.71284376370823% used
From Space:
   capacity = 214695936 (204.75MB)
   used     = 1467680 (1.399688720703125MB)
   free     = 213228256 (203.35031127929688MB)
   0.683608654800061% used
To Space:
   capacity = 214695936 (204.75MB)
   used     = 0 (0.0MB)
   free     = 214695936 (204.75MB)
   0.0% used
concurrent mark-sweep generation:
   capacity = 2147483648 (2048.0MB)
   used     = 613751744 (585.3192749023438MB)
   free     = 1533731904 (1462.6807250976562MB)
   28.580042719841003% used

66527 interned Strings occupying 7048176 bytes.
```

还有一个命令是jmap -histo pid，这个可以看堆内存活的对象类型和实例数量

```
 num     #instances         #bytes  class name
----------------------------------------------
   1:       5478775      506824136  [C
   2:       1421313      315369496  [B
   3:       3791992       91007808  java.lang.String
   4:       2593077       82978464  java.util.HashMap$Node
   5:        730043       78351624  [I
   6:       1393624       67623928  [Ljava.lang.Object;
   7:        712992       62743296  java.lang.reflect.Method
   8:        317758       48979560  [Ljava.util.HashMap$Node;
   9:        178237       32795608  com.fasterxml.jackson.core.json.UTF8StreamJsonParser
  10:        513502       20540080  java.util.LinkedHashMap$Entry
  11:        639968       20478976  com.mysql.cj.conf.BooleanProperty
  12:        941240       18059848  [Ljava.lang.Class;
  13:        353510       16968480  java.util.HashMap
  14:        241178       13505968  sun.nio.cs.UTF_8$Encoder
  15:        539462       12947088  java.lang.StringBuilder
  16:        178448       12848256  com.fasterxml.jackson.databind.deser.DefaultDeserializationContext$Impl
  17:        218814       12253584  com.fasterxml.jackson.core.io.IOContext
```

#### jstat

使用jstat观察gc和metaspace的使用情况，jstat主要输出各个区域的使用量比例(***写这篇文章的时候问题已修复，所以metaspace使用占比比较平稳***)

   `jstat -gcutil 5311` 这个命令以1000ms为间隔，输出pid 5311的进程的gc情况，从这里可以看出metaspace的使用比例的变化

比如s0是survivo 0当前空间的已使用比例，s1的survivo 1当前空间的已使用比例，E是eden区的已使用比例，O是old gen的已使用比例

```

  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT   
  0.00   1.04  57.32  27.40  92.59  86.14 125511 1969.952    28    5.062 1975.014
  0.00   1.04  78.69  27.40  92.59  86.14 125511 1969.952    28    5.062 1975.014
  0.00   1.04  97.81  27.40  92.59  86.14 125511 1969.952    28    5.062 1975.014
  0.98   0.00  32.47  27.40  92.59  86.14 125512 1969.968    28    5.062 1975.031
  0.98   0.00  66.19  27.40  92.59  86.14 125512 1969.968    28    5.062 1975.031
  0.98   0.00  91.34  27.40  92.59  86.14 125512 1969.968    28    5.062 1975.031
  0.00   0.81  17.07  27.41  92.59  86.14 125513 1969.980    28    5.062 1975.042
  0.00   0.81  39.36  27.41  92.59  86.14 125513 1969.980    28    5.062 1975.042
  0.00   0.81  60.87  27.41  92.59  86.14 125513 1969.980    28    5.062 1975.042
  0.00   0.81  89.84  27.41  92.59  86.14 125513 1969.980    28    5.062 1975.042
  0.89   0.00   9.76  27.42  92.59  86.14 125514 1969.995    28    5.062 1975.058
  0.89   0.00  25.07  27.42  92.59  86.14 125514 1969.995    28    5.062 1975.058
  0.89   0.00  44.26  27.42  92.59  86.14 125514 1969.995    28    5.062 1975.058
```

#### gc.log

另一个就是从gc.log里查看metaspace的日志,下面一段日志是gc前和gc后各个区域的内存变化

可以观察下heap before gc和heap after gc2处的metaspace空间使用变化(***此处是问题解决后的gc.log,仅供参考***)

metaspace数据如下，可以看到距离512m还有足够的空间，注意此处是问题解决后的数据，仅供排查参考

- used 176869K， 已使用大概是172m
- capacity 189800K， 容量大概是185m
- committed 190848K，已提交大概是186m
- reserved 1216512K，保留大概是1188m

```
2023-09-12T09:03:08.449+0800: 1249459.698: Total time for which application threads were stopped: 0.0195137 seconds, Stopping threads took: 0.0001738 seconds
2023-09-12T09:03:11.392+0800: 1249462.641: Application time: 2.9429699 seconds
2023-09-12T09:03:11.396+0800: 1249462.646: Total time for which application threads were stopped: 0.0046629 seconds, Stopping threads took: 0.0002150 seconds
2023-09-12T09:03:11.397+0800: 1249462.646: Application time: 0.0000922 seconds
2023-09-12T09:03:11.400+0800: 1249462.649: Total time for which application threads were stopped: 0.0034414 seconds, Stopping threads took: 0.0001087 seconds
2023-09-12T09:03:14.304+0800: 1249465.553: Application time: 2.9035482 seconds
{Heap before GC invocations=130961 (full 14):
 par new generation   total 1887488K, used 1679309K [0x00000006c0000000, 0x0000000740000000, 0x0000000740000000)
  eden space 1677824K, 100% used [0x00000006c0000000, 0x0000000726680000, 0x0000000726680000)
  from space 209664K,   0% used [0x0000000733340000, 0x00000007334b34a0, 0x0000000740000000)
  to   space 209664K,   0% used [0x0000000726680000, 0x0000000726680000, 0x0000000733340000)
 concurrent mark-sweep generation total 2097152K, used 1209833K [0x0000000740000000, 0x00000007c0000000, 0x00000007c0000000)
 Metaspace       used 176869K, capacity 189800K, committed 190848K, reserved 1216512K
  class space    used 19973K, capacity 22145K, committed 23168K, reserved 1048576K
2023-09-12T09:03:14.307+0800: 1249465.557: [GC (Allocation Failure) 2023-09-12T09:03:14.307+0800: 1249465.557: [ParNew
Desired survivor size 107347968 bytes, new threshold 6 (max 6)
- age   1:     674280 bytes,     674280 total
- age   2:     177208 bytes,     851488 total
- age   3:     238904 bytes,    1090392 total
- age   4:     191384 bytes,    1281776 total
- age   5:      38080 bytes,    1319856 total
- age   6:      40232 bytes,    1360088 total
: 1679309K->1868K(1887488K), 0.0143320 secs] 2889142K->1211742K(3984640K), 0.0146560 secs] [Times: user=0.03 sys=0.01, real=0.01 secs] 
Heap after GC invocations=130962 (full 14):
 par new generation   total 1887488K, used 1868K [0x00000006c0000000, 0x0000000740000000, 0x0000000740000000)
  eden space 1677824K,   0% used [0x00000006c0000000, 0x00000006c0000000, 0x0000000726680000)
  from space 209664K,   0% used [0x0000000726680000, 0x0000000726853010, 0x0000000733340000)
  to   space 209664K,   0% used [0x0000000733340000, 0x0000000733340000, 0x0000000740000000)
 concurrent mark-sweep generation total 2097152K, used 1209874K [0x0000000740000000, 0x00000007c0000000, 0x00000007c0000000)
 Metaspace       used 176869K, capacity 189800K, committed 190848K, reserved 1216512K
  class space    used 19973K, capacity 22145K, committed 23168K, reserved 1048576K
}
```

#### jcmd

最后一个可用的jvm工具就是cmd，使用jcmd打印下class数量，看下哪些包下的类增长比较快

使用命令如下

```
jcmd <PID> GC.class_stats|awk '{print$13}'|sed  's/\(.*\)\.\(.*\)/\1/g'|sort |uniq -c|sort -nrk1
```

实际进程打印如下,可以看到最多的是sun.reflect包下的class最多

```
   2283 sun.reflect
    416 com.xx.xx.xx.xx.impl
    316 java.lang.invoke
    306 com.sun.proxy
    275 reactor.core.publisher
    268 java.util
    198 com.google.common.collect
    189 com.google.protobuf
    181 io.netty.channel
    174 java.lang
```



### dump分析

dump 快照之后通过 JProfiler 或 MAT 观察 Classes 的 Histogram (直方图) ，这个估计得深厚的功力才能看出来

![image-20230912134908936](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202309131533501.png)

### arthas

arthas也有很多的命令可以帮助排查，比如dashboard，可以看metaspace区的内存占用，vmtool可以查看具体的class

如下是dashborad命令，可以看到metaspace的使用量

![image-20230912135302354](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202309131533082.png)

可以使用vmtool，匹配指定包名的class 

```
vmtool --action getInstances --className sun.reflect.* --limit 100
```

![image-20230912135822213](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202309131533919.png)

可以使用transform，有类加载的时候就会触发

```
watch java.lang.instrument.ClassFileTransformer transform '{params,returnObj,throwExp}'  -n 5  -x 3
```

也可以跟踪下class init

```
stack sun.reflect.DelegatingClassLoader <init>  -n 5
```



## 实际排查过程

前面介绍了一些关于metaspace的知识，也介绍了一些排查方向，接下来介绍下实际排查过程

接到告警之后，查看进程，发现了metaspace oom的错误，于是开始往metaspace oom的方向开始排查

经过一通google/baidu之后，发现了如上介绍的一些方向和手段

首先是看了美团的文章[场景三：MetaSpace 区 OOM](https://tech.meituan.com/2020/11/12/java-9-cms-gc.html)

发现可以使用jcmd命令进行排查，看下哪个包的class比较多，执行以下命令，得到如下的结果

```
jcmd 5311 GC.class_stats|awk '{print$13}'|sed  's/\(.*\)\.\(.*\)/\1/g'|sort |uniq -c|sort -nrk1|head -n 10

 2294 sun.reflect
    416 com.xx.impl
    316 java.lang.invoke
    316 com.sun.proxy
    275 reactor.core.publisher
    268 java.util
    198 com.google.common.collect
    189 com.google.protobuf
    181 io.netty.channel
    174 java.lang
```

多次执行jcmd之后，发现sun.reflect包下的class比较多，且有增长，于是把目光放在了sun.reflect上。

以为是反射用的不当导致的，然后使用arthas，分析了dashboard以及使用了vmtool查看sun.reflect包的instance。

当时使用dashboard分析，metaspace上涨的非常的快，然后使用vmtool命令查看sun.reflect

```
vmtool --action getInstances --className sun.reflect.* --limit 100
```

执行结果类似下图

![image-20230912135822213](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202309131533382.png)

分析发现里面有mytabis，也有自己调用产生的反射，但是不足以导致metaspace oom



后来想到了加上如下2个参数,用于打印类加载、卸载日志

```
-XX:+TraceClassLoading -XX:+TraceClassUnloading
```

由于发布系统默认没打nohup日志，所以在这里走了点弯路，类加载、卸载信息没打到日志里，后来上机器手动用nohup java -jar xx.jar &之后，发现了端倪

可以看到启动之后，打了比较多的类似下面的这些日志，可以初步判断是com.googlecode.aviator.Expression使用不当造成的

```
[Loaded Script_1691559755244_2757/2021240514 from com.googlecode.aviator.Expression]
[Loaded Script_1691559755339_2758/6228193 from com.googlecode.aviator.Expression]
[Loaded Script_1691559755492_2759/419920034 from com.googlecode.aviator.Expression]
[Loaded Script_1691559756493_2760/1460748148 from com.googlecode.aviator.Expression]
[Loaded Script_1691559756500_2761/520805009 from com.googlecode.aviator.Expression]
[Loaded Script_1691559756506_2762/398664510 from com.googlecode.aviator.Expression]
[Loaded Script_1691559756573_2763/306461935 from com.googlecode.aviator.Expression]
[Loaded Script_1691559756577_2764/1116508142 from com.googlecode.aviator.Expression]
[Loaded Script_1691559756791_2765/870354750 from com.googlecode.aviator.Expression]
[Loaded Script_1691559756799_2766/1661127610 from com.googlecode.aviator.Expression]
[Loaded Script_1691559756812_2767/1883979577 from com.googlecode.aviator.Expression]
[Loaded Script_1691559757019_2768/1281538781 from com.googlecode.aviator.Expression]
[Loaded Script_1691559757332_2769/589742334 from com.googlecode.aviator.Expression]
[Loaded Script_1691559757338_2770/1560860126 from com.googlecode.aviator.Expression]
[Loaded Script_1691559757977_2771/213218340 from com.googlecode.aviator.Expression]
[Loaded Script_1691559758313_2772/505164301 from com.googlecode.aviator.Expression]
[Loaded Script_1691559758318_2773/601453331 from com.googlecode.aviator.Expression]
[Loaded Script_1691559759007_2774/1011764766 from com.googlecode.aviator.Expression]
[Loaded Script_1691559759014_2775/338373159 from com.googlecode.aviator.Expression]
[Loaded Script_1691559759146_2776/1714310047 from com.googlecode.aviator.Expression]
[Loaded Script_1691559759154_2777/1663245165 from com.googlecode.aviator.Expression]
[Loaded Script_1691559759227_2778/797537408 from com.googlecode.aviator.Expression]
[Loaded Script_1691559759234_2779/551097430 from com.googlecode.aviator.Expression]
[Loaded Script_1691559759553_2780/594411270 from com.googlecode.aviator.Expression]
[Loaded Script_1691559759559_2781/1720465741 from com.googlecode.aviator.Expression]
[Loaded Script_1691559759586_2782/215181464 from com.googlecode.aviator.Expression]
[Loaded Script_1691559759601_2783/2036721542 from com.googlecode.aviator.Expression]
[Loaded Script_1691559759797_2784/1723282218 from com.googlecode.aviator.Expression]
[Loaded Script_1691559759802_2785/953430028 from com.googlecode.aviator.Expression]
[Loaded Script_1691559759942_2786/847216521 from com.googlecode.aviator.Expression]
[Loaded Script_1691559759949_2787/1763495463 from com.googlecode.aviator.Expression]
[Loaded Script_1691559760350_2788/47562916 from com.googlecode.aviator.Expression]
```

跟着com.googlecode.aviator.Expression这个类，发现可能造成异常的代码如下

```
Expression expression = AviatorEvaluator.compile(ruleExpressProcessed);
```

这个还有一个重载方法,是否缓存

```
public static Expression compile(String expression, boolean cached) {
        return getInstance().compile(expression, cached);
    }
```

跟踪代码，最后会发现cached字段会影响classloader的获取，如果cached=true,则会使用已缓存的classloader，如果false每次都会new 一个新的classloader

```
com.googlecode.aviator.AviatorEvaluatorInstance#newCodeGenerator(java.lang.String, boolean)
```



### 问题修复

将cached设置为true,metaspace使用率就很平稳了，不会出现高频的类加载和卸载情况。



## 问题复现

本打算在本地复现下这个问题的，但是本地能gc掉，生产环境由于每次2-3天才会发生oom，所以没有发现oom的时候，metaspace里有哪些元素，可以看下本地验证过程和现象，帮助理解

### 测试代码

maven依赖

```
        <dependency>
            <groupId>com.googlecode.aviator</groupId>
            <artifactId>aviator</artifactId>
            <version>5.2.5</version>
        </dependency>
```

jvm参数

堆设置的比较大,metaspace只设置了128m

```
-Xmx4096M
-Xms4096M
-Xmn2048M
-XX:MaxMetaspaceSize=128M
-XX:MetaspaceSize=128M
-XX:+UseConcMarkSweepGC
-XX:+UseCMSInitiatingOccupancyOnly
-XX:CMSInitiatingOccupancyFraction=70
-XX:+ExplicitGCInvokesConcurrentAndUnloadsClasses
-XX:+CMSClassUnloadingEnabled
-XX:+ParallelRefProcEnabled
-XX:+CMSScavengeBeforeRemark
-XX:+HeapDumpOnOutOfMemoryError
-XX:ErrorFile=/data/logs/hs_err_pid%p.log
-Xloggc:/dev/shm/gc.log
-XX:HeapDumpPath=/data/logs/
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-XX:+PrintClassHistogramBeforeFullGC
-XX:+PrintClassHistogramAfterFullGC
-XX:+PrintCommandLineFlags
-XX:+PrintGCApplicationConcurrentTime
-XX:+PrintGCApplicationStoppedTime
-XX:+PrintTenuringDistribution
-XX:+PrintHeapAtGC
-Ddubbo.reference.check=false
-XX:+UnlockDiagnosticVMOptions
-XX:NativeMemoryTracking=detail
//class loading unloading
-XX:+TraceClassLoading
-XX:+TraceClassUnloading
```



使用多线程去并发访问

````
public static void main(String[] args) {
        ThreadFactory threadFactory = new ThreadFactoryBuilder().setNameFormat("test-pool-%d").build();

        ThreadPoolExecutor executor = new ThreadPoolExecutor(50, 50, 1, TimeUnit.SECONDS, new LinkedBlockingQueue<>(50), threadFactory);

        for (int i = 0; i < 32; i++) {
            executor.execute(() -> {
                while (true) {
                    try {
                        String rule = "field01+ field02";
                        Map<String, Object> map = new HashMap<>();
                        int random1 = new Random().nextInt();
                        int random2 = new Random().nextInt();
                        map.put("field01", random1);
                        map.put("field02", random2);
                        Expression compiledExp = AviatorEvaluator.compile(rule);
                        Object obj = compiledExp.execute(map);
                        System.out.println(obj);
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            });
        }


    }
````



控制台也输出了同样的日志

![image-20230912163302034](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202309131532929.png)

然后使用jconsole连到jvm上观察内存使用情况

首先是metaspace的使用情况，最高是到50m，然后gc掉，持续这一过程，说明metaspace使用量在上升，但是能gc的掉，而且不会触发阈值。

生产上发生oom，可能有流量因素，访问量比本地模拟的大。

![image-20230914102800888](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202309141028935.png)



在看下类的加载卸载情况，程序运行了1个半小时，加载和卸载数量差不多持平，大概6529万次，可以发现类是一直在加载，但是本地模拟的情况下，卸载的也快，生产上的具体细节由于网络隔离问题，无法模拟出，仅能根据加载数量看出类加载数量异常了，这块代码是有问题的

![image-20230914102639065](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202309141026106.png)



概览里可以看到已加载的类的数量，在一个区间范围内波动，一边加载一边卸载

![image-20230914130832590](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202309141308641.png)



### 修复验证

代码修改如下，将cached设置为true

```
Expression compiledExp = AviatorEvaluator.compile(rule,true);
```

概览数据如下，可以发现类只加载了2000多个就平稳了，不会加载新的class

![image-20230914132245192](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202309141322257.png)



metaspace使用量稳定在13m,不会波动

![image-20230914132321198](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202309141323256.png)

类在加载了2274个之后保持稳定，没有新增

![image-20230914132359797](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202309141323853.png)



根据修改之后的现象判断，造成问题的原因就是每次都会new classloader，导致类频繁加载卸载，叠加生产流量不可控，极端情况可能就会导致oom

## 总结

由于未设置metaspace区监控，具体的原因暂未发现，比如Metaspace里的class数量占比，分类等。

修复之后再上线，使用arthas观察metaspace使用量，发现metaspace区域内存使用量。

关于metaspace的可用资料，相比于堆内存的oom来说，比较少，所以排查下来也没有那么顺利，不过经过这次问题排查，对metaspace有了一定的了解，下次再排查问题时，可以有更好的切入方向了

## 参考资料

- [场景三：MetaSpace 区 OOM](https://tech.meituan.com/2020/11/12/java-9-cms-gc.html)
- [深入理解堆外内存metaspace](https://www.javadoop.com/post/metaspace)
