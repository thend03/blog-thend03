---
draft: false
date: 2024-06-26T09:10:25+08:00
title: "grep使用不当引起的惨案"
slug: "incorrect-use-of-grep" 
tags: ["linux","grep"]
categories: ["linux"]
authors: ["since"]
description: "grep使用不当引起的惨案"
disableShare: true # 底部不显示分享栏
---



# 背景

最近在跑jenkins任务，有一段脚本使用了如下的语句，统计文件中某一字符串出现的次数

```sh
def result = sh(script: "grep -c 'Uploading successfully' result.txt", returnStdout: true).trim()
```



这个命令返回文本中包含`Uploading successfully`字符串的数量。

看似很简单的一个grep命令，好像也没什么问题。

但是一直行就失败，失败的我很难受。



```shell
缺陷扫描异常:hudson.AbortException: script returned exit code 1
```



而且jenkins执行一次要一个小时，多次打日志定位错误，折磨，折磨，折磨！！！



# 原因

shell语句执行会有状态码，0代表执行成功，其他值代表执行失败，返回1代表执行失败，jenkins script会异常退出，导致无法执行正常逻辑



在grep这个场景下，no match return 1。按常理来说，grep没有匹配到值直接返回状态0, 结果0就好，不知道为啥状态会返回1



在命令行下使用示例复现一下，echo $?会打印上个shell命令的状态

```shell
#匹配不上
 ~/ echo aabcd| grep -c x
0
 ~/ echo $?
1

#匹配得上
 ~/ echo aabcd| grep -c a
1
 ~/ echo $?
0
```



可以看到，匹配得上的执行`echo $?`返回了0，即正常退出， 匹配不上的返回了1，即异常退出。



至于匹配的上的，但是count返回了1，可以看下下面的文档解释，按行匹配的，有1行匹配，count+1

![image-20240626101334003](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202406261013045.png)



# 解决方案

可以在grep命令之后追加 `||:`或者`||true`这2种方式，让grep命令正常返回。



问了下gpt，gpt更推进使用`||:`，所以使用`||:`实现在没有匹配行的情况下返回0。



以下是修改后的执行示例

```shell
#匹配不上
 ~/ echo aaabcd| grep -c x||:
0
 ~/ echo $?
0

#匹配得上
 ~/ echo aaabcd| grep -c a||:
1
 ~/ echo $?
0
```

修改后的命令不论是否有匹配，都是返回的0，即正常退出而不是异常退出

