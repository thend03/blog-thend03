---
draft: false
date: 2023-03-14T19:26:34+08:00
title: "InvalidProviderToken"
description : "推送失败"
slug: "apns-push-failed"
authors: ["since"]
tags: ["push"]
categories: []
externalLink: ""
series: []
---

## 背景
推送业务需要，需要由后台将消息通过apns(苹果统一推送服务)下发到用户的手机上，推送系统通知给用户。
## 后台架构
Java服务端使用了一个开源的库，使用这个开源的库将消息推送给apns。使用的p8证书，token鉴权方式。

使用的ios推送库git地址如下:
https://github.com/jchambers/pushy
```
<dependency>
         <groupId>com.eatthepath</groupId>
         <artifactId>pushy</artifactId>
          <version>0.14.2</version>
</dependency>
```

apns client构建伪代码如下
```
//伪代码，实际需要p8证书文件转byte数组
byte[] certBytes = new byte[]{};
//伪代码，自己的p8证书的 teamId
String apnsTeamId = "teamId";
//伪代码, 自己的p8证书的keyId
String apnsKeyId = "keyId"

//event_loop_group自己按需构建
EventLoopGroup EVENT_LOOP_GROUP = new NioEventLoopGroup(8, ConsumerTreadFactoryUtil.createPushFactory("default", "all"));

ApnsClient apnsClient = new ApnsClientBuilder()
                .setApnsServer(apnsHost)
                .setSigningKey(ApnsSigningKey.loadFromInputStream(new ByteArrayInputStream(certBytes),
                   apnsTeamId, apnsKeyId))
                .setEventLoopGroup(EVENT_LOOP_GROUP).setConcurrentConnections(16)
                .build();
```

## 问题
测试环境运行了一段时间之后，再次推送ios消息之后，报错了，错误信息就是InvalidProviderToken。

先说结论，出现这个错误，就是teamId/keyId/bundleId/p8证书,这几个值对不上。
几个注意点
- 客户端生成regId的时候，使用的bundleId要和服务端推送使用的bundleId一致
- teamId和keyId不要填反了
- 证书要使用正确

我这边的问题是teamId/keyId/bundleId是正确的，就是证书选错了，我这边有2个证书，一个是Auth_xx.p8，一个是Auth_xx1.p8。我实际要用的是Auth_xx.p8，结果服务端上传的时候，用了Auth_xx1.p8证书。

其中Auth_xx.p8，xx就是证书的keyId.

搜索一番之后，发现我填的keyId和证书的文件名称对不上，修改之后就正常了。
![image.png](https://upload-images.jianshu.io/upload_images/14026161-fbd5d77310ffbecd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



## 参考
https://github.com/node-apn/node-apn/issues/477#issuecomment-263531121
https://github.com/jchambers/pushy/discussions/915
