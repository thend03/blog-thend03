---
draft: false
date: 2023-10-19T14:40:56+08:00
title: "ssh配置"
slug: "ssh-config" 
tags: ["ssh"]
authors: ["since"]
description: "ssh配置"
---

ssh在了Iinux系统中有着比较重要的位置，想要远程连接服务器进行操作，ssh是不可或缺的步骤。

我有一台海外的vps，系统是ubuntu，由于使用的机场把22端口封了，导致ssh连接vps非常的不方便，连接一直报如下的错

```sh
 ~/ ssh root@107.173.87.238 -p 22
kex_exchange_identification: Connection closed by remote host
Connection closed by 107.173.87.238 port 22
 ~/
```

咨询过机场，机场方出于端口被滥用的目的，将22端口封禁了，这就会导致想使用ssh连接vps，或者使用github ssh，要先关闭代理，关了代理之后其他的网站又无法访问了，非常的麻烦。

所以本文以此为背景，了解下ssh的配置，达到更换端口，不受限制的目的。

另外为了使用简便出发，介绍下使用私钥登录，以及将登录配置写进配置文件，以简化连接。

## 背景知识

### 什么是ssh

简单说，SSH是一种网络协议，用于计算机之间的加密登录。使用ssh我们可以从本地机器连接到远程的服务器。

### ssh的简单使用

一般我们有2种方式进行ssh连接，一个是使用账号密码，另一个是使用公私钥进行连接

以账号密码的方式登录,

```sh
 ~/ ssh root@107.173.87.238 -p 22
root@107.173.87.238's password:
```

解释下如上命令

- ssh就是我们本文介绍的工具
- root是要登录的目标机器的用户
- 107.173.87.238是我们要登录的服务端的ip
- -p是指定使用连接的服务端的端口，默认是22

本次目标就是修改22端口，使用别的端口进行ssh连接

命令行输入命令回车之后，需要我们输入服务端机器的root账号的密码，输入成功之后，就会登录到107.173.87.238，可以使用root账号在这台机器进行操作了。

另一种是使用秘钥的方式进行登录，首先是生成公钥和私钥，私钥客户端机器自己留着，公钥上传到服务端机器

```sh
 ~/ ssh -i ~/.ssh/racknerd_107.173.87.238 -p 2222 root@107.173.87.238
```

如果私钥正确，输入如上命令回车即可登录远端机器

- -i是指定私钥文件
- `~/.ssh/racknerd_107.173.87.238`,是我用工具生成的私钥文件
- -p是指定连接的端口，默认是22，这里是我修改过的
- root@107.173.87.238是指以root用户登录远端机器

## 修改ssh连接端口

我们主要是要把默认的22端口改掉，换成其他的没有被禁用的端口，保证挂代理的同时，可以使用ssh访问远端服务器

### 基本概念

ssh客户端: 这个是我们要通过某台机器访问目标端的机器，假如我们想在自己的电脑上，使用命令行或者xshell/putty等工具连接服务器，那我们的命令行、xshell、putty等就是ssh客户端

ssh服务端: 我们要连接的目标机器

### 机器信息

本次服务端是购买的一台vps，系统是ununtu 22.04 LTS

```sh
root@racknerd-7c9c56:~# uname -a
Linux racknerd-7c9c56 5.15.0-76-generic #83-Ubuntu SMP Thu Jun 15 19:16:32 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux

root@racknerd-7c9c56:~# lsb_release -a
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 22.04.2 LTS
Release:	22.04
Codename:	jammy
```

客户端是我自己的电脑，命令行执行ssh命令

### 修改ssh连接端口

ubuntu系统的ssh配置在/ect/ssh下

![image-20231019153828910](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202310191538012.png)



![image-20231019154403289](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202310191544315.png)



我们如果想修改服务端端口的话，可以选择直接修改sshd_config的文件。

这里有一个注意点，我们可以放开22端口的注释，然后在port 22下面新增一个port 2222的配置，这样就可以通过22和2222进行ssh连接，如果2222配置错误，我们还可以留有余地，使用22端口进行连接。

这里解释下ssh_config和sshd_config的区别，ssh_config是ssh客户端的配置，sshd_config是服务端的配置。

由于我们这次是修改服务端的配置，所以我们需要修改sshd_config的配置

如果不想修改现有文件，导致错误的话，其实也可以在sshd_config.d/目录下新增xxx.conf文件，在里面新增配置，在这个目录添加配置文件可以生效是因为sshd_config文件里引入了sshd_config.d/*.conf文件

我是在目录下新建了一个my_sshd.conf文件

里面有如下配置项，可以看到配置了2个端口，22和2222，如果sshd_config文件里已经放开了22端口，则可以只添加2222

```sh
Port 22
Port 2222
```

然后将2222端口加入防火墙以放行端口

使用iptables则执行如下命令

```sh
iptables -A INPUT -p tcp --dport 2222 -j ACCEPT
```

如果使用firewall-cmd则执行如下命令

```sh
firewall-cmd --zone=public --add-port=2222/tcp --permanent
firewall-cmd --reload
```

如果是公有云平台，则在对应公有云平台的安全组进行设置

然后执行如下命令重启sshd，则服务端的配置就重启好了

```sh
sudo service sshd restart
```

执行命令查看ssh端口是否可以启动

```sh
netstat -nlp|grep 22
```

可以看到22和2222端口都以正常启动

![image-20231019161001703](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202310191610746.png)

接下来就可以使用ssh命令进行连接了，使用 `ssh -p 2222 root@107.173.87.238`进行账号密码连接，这样就可以避免22端口被禁导致的连接失败的问题了

### 详细步骤

这一步只讲步骤，不讲其他的额外知识

1.使用`ssh -p 22 root@107.173.87.238`连接到机器上，我们的任何修改都是要先连接到目标机器上，所以这次我们还是先使用22端口连接上去进行修改

2.在/etc/ssh/sshd_config.d/目录下新增my_sshd.conf文件, 然后使用vim编辑my_sshd.conf,将新的端口配置写入my_sshd.conf里

- 首先执行`cd /etc/ssh/sshd_config.d/`，进入要修改的目录
- 然后执行`touch my_sshd.conf`，新增配置文件
- 执行`vim my_sshd.conf` ，编辑文件，添加`Port 22`和`Port 2222`2个端口配置(分2行，详情见上方)，修改完之前:wq保存
- 最后检查/etc/ssh/sshd_config，看文件里的`PermitRootLogin yes`配置项是否打开，如果被注释掉，需要去掉#，值为no的话需要修改为yes，这样可以支持使用root用户登录

3.将2222端口添加到防火墙

- 如果使用iptables则执行如下命令`iptables -A INPUT -p tcp --dport 2222 -j ACCEPT`
- 如果是firewall-cmd则执行如下命令`firewall-cmd --zone=public --add-port=2222/tcp --permanent` 然后执行reload加载`firewall-cmd --reload`
- 如果是公有云则去对应厂商的网络安全策略那里修改

4.重启sshd服务,使用如下命令`sudo service sshd restart`

5.重启之后验证端口是否正常启动，使用`netstat -nlp|grep 22`查看端口占用，如果22和2222都已经启动，则sshd服务正常，否则需要检查原因

6.客户端重新连接，验证22和2222端口功能是否正常，分别使用`ssh -p 22 root@107.173.87.238`和`ssh -p 2222 root@107.173.87.238`，如果2个都能正常连接，则修改成功

## 使用公钥和私钥进行连接

在上一步中我们修改了ssh连接的端口，可以使用2222端口进行ssh连接，免去了代理封禁了22端口的困扰。但是每次进行连接的时候总要输一大串的连接命令，还要输入繁杂的密码，相信机器的密码都不会太简单，每次复制粘贴密码比较繁琐。

那么ssh除了账号密码的方式，还支持公私钥的方式，客户端拿着私钥去连接服务端，我们这一小节就介绍ssh使用秘钥进行连接的方式

### 生成公钥和私钥

 我们可以使用ssh-keygen命令生成秘钥

```sh
 ~/ ssh-keygen -t rsa -f ~/.ssh/test -b 4096
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in test
Your public key has been saved in test.pub
The key fingerprint is:
SHA256:yL155XDEOzFmSlHcOvUVTyOrduhjWtfakA9gJhVWBUg since@fengchuangdeMacBook-Pro.local
The key's randomart image is:
+---[RSA 4096]----+
|          oE+=o+o|
|          .+o =.+|
|          ..B+ .o|
|     . o ..==+  .|
|      o S.oB=o   |
|         o*=o.o  |
|        o .=.= . |
|         .+ o *  |
|         .   . o |
+----[SHA256]-----+
```

解释下这个命令，这个命令是用来生成ssh密钥对的

- -t用于指定使用的加密算法，支持如下几种加密算法: dsa 、 ecdsa 、 ecdsa-sk 、 ed25519 、 ed25519-sk 、 rsa
- -b是bits，用来指定生成的密钥对的密钥位数，上述命令是生成4096位密钥文件
- -f用于指定生成的秘钥文件名称和存储路径，不止定-f的话会默认生成~/.ssh/id_rsa和~/.ssh/id_rsa.pub
- -C用于添加评论，本次命令没有使用这个参数

使用上述命令就生成了一个可用的ssh秘钥对，其中`~/.ssh/test`文件是私钥，ssh客户端连接服务端的时候拿私钥进行连接，`~/.ssh/test.pub`是公钥文件，需要上传到服务端，客户端连接请求过来的时候，服务端拿着公钥判断权限用的，关于服务端公钥配置，待会再详细介绍。

### 服务端设置公钥

上一步生成公钥和私钥之后，私钥客户端留着自己连接的时候用，公钥需要上传到要连接的服务端，建立ssh连接的时候，服务端使用公钥进行鉴权，这一小节介绍服务端如何设置公钥文件

以下操作会指明各个文件和步骤是服务端操作还是客户端操作，请明确操作环境。

目前的大前提我们只配置了账号密码登录，配置了22和2222端口用于进行ssh连接。

首先我们在客户端有`~/.ssh/test.pub`这个公钥文件，我们需要把公钥信息上传到服务端去，可以看到公钥文件里是一些加密后的字符串，如何把公钥文件上传到服务端，有如下几种方式

![image-20231019224844800](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202310192248883.png)

***第一种是使用ssh-copy-id***

我们使用ssh-copy-id将test.pub公钥文件，从客户端机器上传到服务端

```shell
 ~/.ssh/ ssh-copy-id -i ./test.pub -p 2222 root@107.173.87.238
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "./test.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@107.173.87.238's password:

Number of key(s) added:        1

Now try logging into the machine, with:   "ssh -p '2222' 'root@107.173.87.238'"
and check to make sure that only the key(s) you wanted were added.
```

看上方的命令和执行结果，-i指定要上传的公钥文件，-p指定端口，其中需要我们输入root账号的密码，然后可以看到公钥就被成功安装了。

然后上服务器检查，可以发现~下原来没有.ssh文件夹，也没有~/.ssh/authorized_keys文件，ssh-copy-id为我们创建了目录和文件，且把公钥文件写入到authorized_keys文件里了。

如果没有/root/.ssh文件以及/root/.ssh/authorized_keys文件，需要手动创建目录和文件，再执行上述的ssh-copy-id命令。

然后在***服务端***执行以下2个命令,切记要复制这2个命令执行，不要随意修改

```shell
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

然后检查~/.ssh/authorized_keys的文件内容是否和test.pub一致，如果一致那就上传公钥成功了

然后就可以在客户端执行ssh命令进行连接了，尝试一下，可以看到已经可以正常连接了

```shell
 ~/.ssh/ ssh -i ~/.ssh/test -p 2222 root@107.173.87.238
root@racknerd-7c9c56:~#
```

***第二种是使用ssh命令执行上传和写入***

第二种我们使用ssh命令，读取test.pub，并执行命令写入服务端的~/.ssh/authorized_keys文件，再执行第2种之前，我们先删掉服务器上的~/.ssh目录，然后再执行命令上传

详细命令如下,执行之后输入密码，即可将本地的公钥文件写入服务端的~/.ssh/authorized_keys文件里

```shell
 ~/.ssh/ ssh -p 2222 root@107.173.87.238 'mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys' < ~/.ssh/test.pub

root@107.173.87.238's password:
```

这个命令里包含了多个步骤，依次解释下命令含义

- ssh -p 2222 root@107.173.87.238这个就是使用ssh连接服务端
- mkdir -p ~/.ssh是在服务端创建~/.ssh目录
- 'cat >> .ssh/authorized_keys' < ~/.ssh/id_rsa.pub的作用是，将本地的公钥文件~/.ssh/id_rsa.pub，重定向追加到远程文件authorized_keys的末尾，注意>>表示的是追加写入，不会覆盖原文件

然后在服务端执行以下2个命令,切记要复制这2个命令执行，不要随意修改

```shell
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

然后就可以在客户端执行ssh命令进行连接了，尝试一下，可以看到已经可以正常连接了

```shell
 ~/.ssh/ ssh -i ~/.ssh/test -p 2222 root@107.173.87.238
root@racknerd-7c9c56:~#
```

***第三种是使用scp命令，先将本地的公钥文件上传到服务端，然后在服务端执行cat命令将公钥文件内容写入authorized_keys文件***

第三种就是先将客户端本地的公钥文件上传到服务端，然后使用ssh连接到服务端，执行命令将公钥文件内容写入authorized_keys文件

照例先删除服务端的~/.ssh文件夹，具体命令如下

```shell
 ~/.ssh/ scp -P 2222 ~/.ssh/test.pub root@107.173.87.238:~/
root@107.173.87.238's password:
Permission denied, please try again.
root@107.173.87.238's password:
test.pub
```

解释下具体的含义，这行命令就是使用scp将客户端本地的~/.ssh/test.pub文件上传到服务端的~/目录下，注意P是大写，代表连接使用的端口

然后ssh连接到服务端，使用如下命令创建~/.ssh目录和~/.ssh/authorized_keys文件

```shell
mkdir ~/.ssh
touch ~/.ssh/authorized_keys
```

然后服务端执行如下命令授权，不要随意修改

```shell
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

然后执行cat命令，将已经上传到服务器的test.pub文件的内容写入~/.ssh/authorized_keys

```shell
cat ~/test.pub >> ~/.ssh/authorized_keys
```

然后就可以在客户端执行ssh命令进行连接了，尝试一下，可以看到已经可以正常连接了

```shell
 ~/.ssh/ ssh -i ~/.ssh/test -p 2222 root@107.173.87.238
root@racknerd-7c9c56:~#
```



接下来有些注意事项，如果使用ssh -i无法连接，需要检查下/etc/ssh/sshd_config这个文件，看看如下几个配置项是否被注释掉

如果有#就是被注释了，未生效，需要去掉#号放开，放开之后需要执行`service sshd restart`重启服务端ssh服务

```shell
　RSAAuthentication yes
　PubkeyAuthentication yes
　AuthorizedKeysFile .ssh/authorized_keys
```



综合看下来，还是使用ssh-copy-id比较简单省事，其他两种为我们展示了具体的为服务端添加公钥的过程

### 详细步骤

以ssh-copy-id为例，说明下详细步骤

1.执行ssh-copy-id命令将客户端公钥文件上传到服务端`ssh-copy-id -i ./test.pub -p 2222 root@107.173.87.238`

2.检查服务端是否存在~/.ssh目录，是否存在~/.ssh/authorized_keys文件，如果任一不存在，则执行`mkdir ~/.ssh &&touch ~/.ssh/authorized_keys `创建路径和文件之后，重复执行第一步 

3.执行命令为服务端公钥文件授权,需要执行`chmod 700 ~/.ssh`和`chmod 600 ~/.ssh/authorized_keys`

4.检查服务端~/.ssh/authorized_keys文件内容是否和客户端的公钥文件内容一致，一致则说明上传成功

5.客户端使用`ssh -i ~/.ssh/test -p 2222 root@107.173.87.238`尝试连接，如果成功则添加秘钥方式修改成功

6.如果客户端连接失败，检查下`/etc/ssh/sshd_config`文件的`RSAAuthentication`，`PubkeyAuthentication`，`AuthorizedKeysFile`等key是否放开，如果注释掉需要放开，执行``service sshd restart`重启sshd服务

## ssh使用config进行登录

上面2种登录方式，不管是使用使用用户密码，还是使用公私钥，都需要在命令行输入一大串命令，比较繁琐。

使用命令行进行ssh连接的时候，虽然有历史记录，但仍比较繁琐，需要频繁复制输入密码，有了私钥认证之后好一点，但还是比较繁琐。

本小节介绍如何使用配置文件的方式，将连接配置写到配置文件里，简化操作，提高使用ssh连接的效率

ssh有个配置文件，路径是`~/.ssh/config`，可以在这个文件里面配置ssh连接信息，另外一个配置文件地址是`/etc/ssh/ssh_config`，这个上一小节有介绍

我本地有如下3个配置，写在了`~/.ssh/config`文件里

```sh
Host github.com
  HostName ssh.github.com
  User git
  Port 443
Host vps-racknerd-id
  HostName 107.173.87.238
  User root
  Port 2222
  IdentityFile ~/.ssh/test
Host vps-racknerd-pwd
  HostName 107.173.87.238
  User root
  Port 2222
```

第一个是github的配置，由于22端口被封，每次git push非常的费劲，后面根据搜索，将端口换成了443,问题得以解决

第二个是vps的配置，使用了私钥文件鉴权的方式

第三个是vps的配置，使用了用户密码的鉴权方式

解释下配置项的含义

- Host配置的要连接的主机名，github的Host好像是固定的，github.com， *为默认值，匹配所有的Host的行为，我理解这里的Host可以相当于别名使用了，ssh 匹配Host，然后执行Host的逻辑
- HostName，这个是指定ssh连接的目标地址，可以是域名，也可以是ip
- User是指定ssh连接的用户
- Port是指定ssh连接的端口
- IdentityFile是指定的ssh连接的私钥文件地址

我本地连接vps,现在可以使用`vps-racknerd-id`和`vps-racknerd-id`进行连接了

使用vps-racknerd-pwd，还是使用用户密码的方式登录，仍需输入用户对应的密码，优点是少输很多东西

```sh
 /etc/ssh/ ssh vps-racknerd-pwd
root@107.173.87.238's password:
root@racknerd-7c9c56:~#
```

使用vps-racknerd-id，使用私钥方式登录，不用输入密码，即可直接连接，只需执行一步就可连接

```sh
 ~/.ssh/ ssh vps-racknerd-id
root@racknerd-7c9c56:~#
```



## 总结

本文介绍了3个功能: 修改ssh默认端口，使用私钥登录，使用配置进行登录

第一个功能实现了修改端口，可以避免代理禁用22端口导致的连接问题

第二个功能实现了使用私钥文件登录，不用再每次输入密码

第三个功能实现了将配置信息写到config文件里，简化了连接的命令



上面介绍的是命令行使用ssh连接的方式，如果有条件的话，可以使用其他的ssh客户端，比如xshell、putty、secureCRT、mobaXterm等工具，这些工具带了界面，可以在界面上进行操作，并且有秘钥文件管理和存储密码的功能





## 参考链接

[openssh config file examples for linux/unix](https://www.cyberciti.biz/faq/create-ssh-config-file-on-linux-unix/)

[SSH原理与运用-阮一峰](http://www.ruanyifeng.com/blog/2011/12/ssh_remote_login.html)

[设置SSH通过秘钥登录](https://www.runoob.com/w3cnote/set-ssh-login-key.html)

[SSH教程](https://wangdoc.com/ssh/)
