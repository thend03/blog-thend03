---
draft: false
date: 2024-02-06T15:36:27+08:00
title: "manjaro环境配置"
slug: "manjaro-env-config" 
tags: ["manjaro"]
categories: ["manjaro"]
authors: ["since"]
description: "manjaro环境配置"
---

## 前言

在[上一篇安装windows+manjaro中](https://blog.thend03.com/post/dual-system-installation-manjaro-and-windows/)，安装好了双系统，那么安装好之后，需要更新下manjaro系统，安装好日常使用的环境和应用程序，方便后续的使用。

本篇分为系统、应用程序、外观3个部分进行说明。

## 更新系统

在安装好系统之后，由于系统是国外的发行版，一些安装包、应用程序啥的，都是国外的源，没代理的话可能会下载慢，甚至下载不下来，所以首要是要对系统进行更新，更换国内的镜像源。



### 更换国内镜像源

打开kconsole终端，在命令行输入如下命令

```sh
sudo pacman-mirrors -i -c China -m rank
```

![image-20240206155454245](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402061554278.png)



执行命令之后会在命令行终端输出可用的列表，然后弹窗供选择，源选择一个就好，多了没啥用，还会降低速度

![image-20240206155642311](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402061556333.png)



### 添加archlinuxcn源

添加archlinuxcn的源，可以获得更多的包

编辑配置文件

```sh
sudo vi /etc/pacman.conf
```

在配置文件中添加如下配置

```sh
[archlinuxcn]
SigLevel = Optional TrustAll
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch
```



### 安装yay包管理工具

yay是aur仓库的包管理助手，使用yay不用像使用pacman一样，需要sudo，而且包比官方仓库的全，官方仓库没有的包可以使用yay尝试安装

打开命令行，输入如下命令安装yay

```sh
sudo pacman -S yay
```



### 安装snap包管理工具

有些包只有snap有，所以也安装下snap

```sh
sudo pacman -S snapd
```

然后执行如下命令

```sh
sudo systemctl enable --now snapd.socket
```

创建链接

```sh
sudo ln -s /var/lib/snapd/snap /snap
```

测试snap功能

```sh
$ sudo snap install hello-world
hello-world 6.3 from Canonical✓ installed
$ hello-world
Hello World!
```



### 更新系统

换好源，添加完源之后，执行以下命令更新系统

```sh
sudo pacman -Syyu
```



### 安装base-devel

安装基础构建包

```sh
sudo pacman -S base-devel
```



到这里系统层面的设置基本上差不多了，下面介绍一些常用应用程序

一些常用的命令如vim等需要自行安装，使用命令行安装即可，不再多介绍。

## 安装常用软件

### 科学上网

搞定了科学上网之后，下载包，访问google， github才会丝滑。所以先搞定科学上网

科学上网有3个选项: Qv2ray、clash for windows、v2raya

其中qv2ray停止运营了，clash删库跑路了，v2raya还在正常更新。

我之前用过qv2ray和clash，目前一些历史版本还能用。所以也介绍下这2个的安装使用

#### Qv2ray

qv2ray的项目解散后，连官网也不维护了，烟消云散，好歹github的仓库还在，还能下载安装包

2.7.0下载地址: https://github.com/Qv2ray/Qv2ray/releases/tag/v2.7.0

由于要安装在linux上，所以选择AppImage进行下载

![image-20240207095145127](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402070951175.png)

下载好之后，把AppImage拖到桌面，双击就能运行。由于这个版本比较老了，可能没对新的桌面框架做适配，所以桌面快捷方式会运行失败。

然后需要下载v2ray-core，最新的v2ray-core版本和qv2ray不兼容，所以要下载老版本的v2ray-core，才能运行。经过测试,4.45.2版本可以兼容，所以我们下载4.45.2版本的v2ray-core。

V2ray-core-4.45.2下载地址: https://github.com/v2fly/v2ray-core/releases/tag/v4.45.2

下载64位的linux包即可。

![image-20240207095932292](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402070959322.png)



下载好v2ray-core之后解压，打开qv2ray首选项-内核设置，设置v2ray-core路径，然后点击检查v2ray核心设置，检查通过v2ray-core配对就成功了。

![image-20240207100303182](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071003215.png)



![image-20240207100417665](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071004722.png)



接下来就是添加订阅链接，打开分组，添加新的组，或者修改默认分组，添加你的机场的订阅链接，添加之后更新订阅，如果更新失败，切换下订阅类型试试。

![image-20240207101011187](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071010224.png)



如果是ssr之类的协议，可以去github下载对应的插件，放到本地的~/.config/qv2ray/plugins目录下，然后重启qv2ray，点击插件按钮即可加载插件。

ssr插件下载地址:  https://github.com/Qv2ray/QvPlugin-SSR/releases/tag/v3.0.0

其他插件可在github仓库自行搜索: https://github.com/orgs/Qv2ray/repositories?type=all

最后需要去系统设置那手动设置下代理，让系统代理生效

![image-20240207134916818](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071349852.png)

其他配置项不在赘述

#### clash for windows

说是for windows，不过linux也能用。不过clash作者删库跑路了，所以github无法下载，yay也无法安装，只能找互联网残存版本安装。

下载地址: https://archive.org/download/clash_for_windows_pkg

下载linux 64位版本，解压之后就可以在命令行执行cfw命令启动clash

![image-20240207131743826](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071317865.png)



也可以在桌面新建快捷方式，以下是我的clash桌面快捷方式配置, clash解压路径位/opt/app/clash-for-windows，路径需自行修改

```sh
[Desktop Entry]
Name=Clash for Windows
Comment=Clash for Windows
Exec=/opt/app/clash-for-windows/cfw
Icon=/opt/app/clash-for-windows/logo.jpeg
Terminal=false
Type=Application
Categories=Developer;
```

![image-20240207132348592](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071323658.png)



首先拿到clash订阅链接，然后点击profiles, 粘贴订阅链接，点击download，就会下载订阅配置到本地

![image-20240207133005052](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071330084.png)

然后在proxy选择可用节点，策略选择rule(规则模式)

![image-20240207134625115](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071346148.png)



最后再打开系统设置，设置代理为clash端口，代理就倒腾完了

![image-20240207134916818](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071349852.png)



#### v2raya

上面的2个都已经停止维护和跑路了，v2raya还没跑路，且用且珍惜

首先要安装v2ray-core，然后使用命令行安装v2raya

```sh
yay -S v2raya
```

安装完成之后，可以使用命令行启动v2raya，端口是2017，注意看提示，root和非root启动有不同的启动方式

![image-20240207140831309](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071408350.png)



启动之后，浏览器访问localhost:2017，创建一个管理员用户

![image-20240207140931593](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071409634.png)



然后导入订阅

![image-20240207141020208](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071410287.png)



导入订阅之后，就可以选择节点了

![image-20240207141048498](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071410542.png)



#### 命令行代理

有些任务需要在终端下进行，所以也需要设置命令行代理。命令行代理有2种方式，一种是设置环境变量，另一种是使用工具，让工具走

代理。

#### 环境变量

临时的直接执行以下命令，终端窗口关闭，变量清除，端口看使用的是哪种代理方式

```
export https_proxy=http://127.0.0.1:7890 http_proxy=http://127.0.0.1:7890 all_proxy=socks5://127.0.0.1:7890
```



长期的就把配置想到配置文件里，manajaro默认用的是zsh，那么修改~/.zshrc文件, 添加如下2行命令

````
alias setproxy="export ALL_PROXY=socks5://127.0.0.1:7890; echo 'SET PROXY SUCCESS!!!'"

alias unsetproxy="unset ALL_PROXY; echo 'UNSET PROXY SUCCESS!!!'"
````



保存之后执行source命令`source ~/.zshrc`

下次想用代理的时候，就在命令行执行setproxy，不想用代理就执行unsetproxy

#### proxychains

另一个是使用proxychains，proxychains设置代理，然后执行命令都加上proxychains

执行命令安装proxychains

```sh
sudo pacman -S proxychains-ng
```



然后修改配置文件，设置代理, 最后一行将socks5代理设置为本地代理端口

```sh
sudo vim /etc/proxychains.conf
```

![image-20240207142856019](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071428065.png)



安装好之后，如果需要代理，可以在所有命令之前都加上proxychains，如

```shell
proxychains git clone xxx
```



### 输入法

在进行其他步骤之前首先是安装输入法，输入法我选择的是开源的rime框架，开源，安全，强大，可定制化程度高，但是也会带来不小的

学习和解决问题的成本。



本篇介绍我本人比较喜欢的一款基于rime的输入法，在manjaro可用，如需其他的比如搜狗linux输入法，本文不涉及。

选这款输入法主要是被它的颜值吸引，基于rime封装之后，解决了大部分的问题，当然在linux下的开源项目，总归会遇到各种各样千奇

百怪的bug。



既然选择了linux，就直面恐惧，直面问题吧。



这款输入法名字叫做四叶草输入法，好听颜值在我个人看来也很高

输入法开源地址如下: https://github.com/fkxxyz/rime-cloverpinyin

Linux安装教程地址如下: https://github.com/fkxxyz/rime-cloverpinyin/wiki/linux

本部分基于项目的wiki，补充下安装过程，解决一些wiki上遗留的问题



#### 安装fcitx5

在manjaro可以直接使用pacman安装fcitx5，fcitx5是目前最新的输入法框架，内核轻量，功能强大。

```sh
sudo pacman -S fcitx5 fcitx5-qt fcitx5-gtk fcitx5-configtool
```

然后需要修改配置，wiki上写着要将配置写入`~/.xprofile`，但是经过我的测试，光写到这个文件还不行，可能会导致无法展示这个输入法。

最好是在/etc/environment里也写一份同样的配置

配置项具体内容如下

```sh
export GTK_IM_MODULE=fcitx5
export QT_IM_MODULE=fcitx5
export XMODIFIERS="@im=fcitx5"

export LANG="zh_CN.UTF-8"
export LC_CTYPE="zh_CN.UTF-8"
```

改完配置之后，重新登录桌面使配置生效，当然重启电脑最好。

![image-20240206164217643](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402061642673.png)

如上图所示，fcitx5就启用了



#### 安装rime

执行以下命令安装rime

```sh
sudo pacman -S fcitx5-rime
```

然后右键托盘图标点开配置，将“中州韻”加入列表即可

点击配置，然后选择右下角的添加输入法，点击中州韵添加到分组即可

![image-20240206164242555](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402061642583.png)



![image-20240206164614889](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402061646916.png)



![image-20240206164656103](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402061646133.png)



#### 安装四叶草输入法

以上2步，安装好输入法框架，rime和fcitx5，就可以安装最终的输入法方案了，即四叶草输入法

wiki上介绍使用yay安装，但是我这边没有成功，无法展示四叶草输入方案

```sh
yay -S rime-cloverpinyin
```

所以我选择手动下载安装包到用户目录进行安装, 下载地址:  https://github.com/fkxxyz/rime-cloverpinyin/releases

将压缩包下载到~/.local/share/fcitx5/rime，然后解压，将文件夹里的文件复制到~/.local/share/fcitx5/rime目录下

![image-20240206171823495](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402061718523.png)



#### 新增配置文件

在~/.local/share/fcitx5/rime文件夹下新建default.custom.yaml ，内容为

```yaml
patch:
  "menu/page_size": 8
  schema_list:
    - schema: clover
```

其中 8 表示打字的时候输入面板的每一页的候选词数目，可以设置成 1~9 任意数字。

写好该文件之后，点击右下角托盘图标右键菜单，点“重新部署”，然后再点右键，在方案列表里面应该就有“ 🍀️四叶草拼音输入法”的选项了。

关于 default.custom.yaml 文件的更多解释，可以参考[官方文档定制指南](https://github.com/rime/home/wiki/CustomizationGuide)

这里也有作者的一些配置供参考: [四叶草输入法基本配置](https://www.fkxxyz.com/d/cloverpinyin/#%E5%9F%BA%E6%9C%AC%E9%85%8D%E7%BD%AE)



#### 使用四叶草输入法

写好该文件之后，点击右下角托盘图标右键菜单，点“重新部署”，然后再点右键，在方案列表里面应该就有🍀️四叶草拼音输入法的选项了。

![image-20240206172329385](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402061723413.png)

#### 美化输入法

fcitx5可以使用搜狗皮肤美化，具体的可以参考如下的教程

[fcitx5使用搜狗皮肤](https://www.fkxxyz.com/d/ssfconv/)

这里就说一点，转换之后得到的皮肤，是需要放到指定用户目录下的，不同的输入法框架用户目录不同

| 平台   | rime用户资料夹位置         |
| ------ | -------------------------- |
| ibus   | ~/.config/ibus/rime        |
| fcitx  | ~/.config/fcitx/rime       |
| fcitx5 | ~/.local/share/fcitx5/rime |



我这里使用的是fcitx5，但是皮肤最终的路径是在~/.local/share/fcitx5/themes，这个要注意下

![image-20240206170148217](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402061701245.png)

教程里介绍的fcitx的皮肤路径，ibus的需要自行搜索解决

其他关于用户资料夹的介绍，详见官方文档: [Rime 中的数据文件分布及作用](https://github.com/rime/home/wiki/RimeWithSchemata#rime-中的數據文件分佈及作用)

另外介绍2款做好的皮肤

https://github.com/ayamir/fcitx5-nord

https://github.com/ayamir/fcitx5-gruvbox



#### 输入法配置项修改

点击系统托盘-输入法-配置，或者从系统设置进入输入法配置页面，这里可以进行输入法的配置

![image-20240206173004837](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402061730867.png)



如果想修改输入法的字体和皮肤，可以选择经典用户界面设置，设置字体，字号，输入法皮肤等

![image-20240206173102804](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402061731837.png)



![image-20240206174352050](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402061743093.png)





中州韵的配置在如下位置修改，红框圈出来的选项，如果选中的话，会导致光标一直固定在最前面，很别扭

![image-20240207092006262](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402070920301.png)

![image-20240207093451576](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402070934619.png)



### 截图工具

下一个就是截图工具了，截图必不可少，推荐flameshot(火焰截图), 另一个我常用的截图工具snipaste，由于linux适配有问题，暂不推荐

执行以下命令安装flameshot

```sh
sudo pacman -S flameshot
```



安装好后可以在搜索栏搜索flameshot启动, 启动之后可以在系统托盘处点击火焰截图进行截图

![image-20240207143857294](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071438337.png)



上面使用起来会有点不方便，需要为火焰截图设置快捷键截图

打开系统设置，快捷键，如果搜不到火焰截图，就点击下方添加应用程序进行添加，添加好之后，设置截图快捷键，以后就可以使用快捷键截图了。

![image-20240207144342026](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071443075.png)



### 转换deb包

有些只有deb包，没有manjaro安装包，为了能在manjaro上安装，所以要安装deb转换工具

执行以下命令安装debtap

```sh
sudo pacman -S debtap
```

执行转换之前，需要有一次更新行为，这里可能会超时或失败，记得上代理

```sh
sudo debtap -u
```

转换deb包

```sh
sudo debtap xxx.deb
```

安装转换完成的包

```sh
sudo pacman -U xxx.pkg.tar.zst
```



### 安装ohmyzsh

manjaro的默认shell是zsh，ohmyzsh对zsh做了进一步的封装

执行以下命令安装ohmyzsh,为了防止网络不通，使用proxychains做代理, 实在下载不下来，只能手动下载install.sh，然后执行安装脚本

```sh
proxychains sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

然后下载如下的几个插件

```sh
# 自动补全
git clone https://github.com/zsh-users/zsh-autosuggestions $ZSH_CUSTOM/plugins/zsh-autosuggestions


git clone https://github.com/zsh-users/zsh-syntax-highlighting.git $ZSH_CUSTOM/plugins/zsh-syntax-highlighting

# zsh-vi-mod
git clone https://github.com/jeffreytse/zsh-vi-mode $ZSH_CUSTOM/plugins/zsh-vi-mode
```

下载好之后修改全新的~/.zshrc，历史的文件记得在安装的时候备份

```sh
plugins=(
    git
    zsh-syntax-highlighting
    zsh-autosuggestions
    zsh-vi-mode
)
```

如果想修改主题的话，修改~/.zshrc文件里的如下配置，默认主题是robbyrussell

```sh
ZSH_THEME="robbyrussell"
```

更多主题见如下地址: https://github.com/ohmyzsh/ohmyzsh/wiki/Themes

### 安装typora

typora是我个人比较喜欢的markdown编辑器

```sh
sudo pacman -S typora
```

### 安装chrome

执行命令安装chrome, 安装好之后可以在命令行执行`google-chrome-stable`启动，也可以搜索chrome进行启动

```sh
yay -S google-chrome
```

chrome桌面快捷方式, 在桌面新建文本文件，写入如下配置

```sh
[Desktop Entry]
BinaryPattern=chrome;
MimeType=
Name=chrome
Exec=google-chrome-stable
Icon=/opt/app/chrome-logo.png
Type=Application
Terminal=0
```



### 安装vscode

进入下载页，点击下载64位的tar包，由于不支持deb和rpm，所以手动下载tar包安装

下载地址: https://code.visualstudio.com/#alt-downloads

![image-20240207153414931](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071534976.png)





桌面快捷方式如下，新建文本文件，写入如下配置然后保存

```sh
[Desktop Entry]
Name=Visual Studio Code
Comment=Code Editing. Refined.
GenericName=Text Editor
Exec=/opt/app/vscode/bin/code --unity-launch %F
Icon=/opt/app/vscode/logo.png
Type=Application
StartupNotify=false
StartupWMClass=Code
Categories=TextEditor;Development;IDE;
MimeType=text/plain;inode/directory;application/x-visual-studio-code-workspace;
Actions=new-empty-window;
Keywords=vscode;
```



#### wifi

使用linux开wifi，manjaro和arch的文档太复杂，找了个开源工具，可以创建wifi热点

项目地址如下: https://github.com/lakinduakash/linux-wifi-hotspot

执行命令安装

```sh
yay -S linux-wifi-hotspot
```

安装之后可以在应用程序搜索wifi hotspot,即可启动

5Ghz可能会有bug，建议使用2.4G, SSID是你的wifi名称，password是wifi密码，其他默认即可

![sc4.png](https://github.com/lakinduakash/linux-wifi-hotspot/blob/master/docs/sc4.png?raw=true)

### 美化系统

kde的桌面ui排布和windows相似，本次美化的目标是把桌面美化成mac风格



#### 主题/应用程序/图标/窗口风格

先说主题，系统设置-外观-plasma视觉风格-获取新plasma视觉风格，搜索`MacBreeze Shadowless`，然后点击安装，安装完成之后点击使用此主题



![image-20240207155200413](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071552460.png)





![image-20240207155223499](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071719192.png)







然后设置应用程序风格，系统设置-外观-应用程序风格-配置GNOME/GTK应用程序风格-获取新GNOME/GTK应用程序风格，搜索`McMojave`， 选择喜欢的类型，安装然后使用。



![image-20240207155302144](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071553228.png)



![image-20240207155318126](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071553230.png)

![image-20240207155348176](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071553288.png)





修改图标，系统设置-外观-全局主题-图标, 获取新图标，搜索`Mojave CT icons`安装并使用

![image-20240207155505160](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071555273.png)

![image-20240207155535111](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071555229.png)



系统设置-外观-窗口装饰元素-获取新窗口装饰，搜索`McMojave Aurorae`,安装并使用



#### 顶栏设置

Manjaro kde的任务面板和windows类似，是在底部，那么为了仿Mac，所以要删除底栏，添加一个顶栏

点击面板然后右键，选择进入编辑模式，然后更多选项，选择移除面板，即可删除面板，删除底部面板即可，避免和latte-dock冲突

![image-20240207162936655](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071629711.png)



![image-20240207162423700](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071624795.png)





在桌面右键选择，添加面板-应用程序菜单栏。

![image-20240207163935357](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071639416.png)





下面进行顶栏的设置，点击添加部件，添加如下部件: 应用程序启动器，锁定/注销、系统托盘、数字时钟等

![image-20240207164023244](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071640365.png)





然后为顶栏添加2个间距用于使用时钟分隔左右，2个蓝色的条就是间距，全局菜单就是其他应用的设置按钮菜单

![image-20240207164306007](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071643073.png)





然后调整顶栏布局，将应用程序启动器放在顶栏的最左边，然后在它的右边放全局菜单，然后再右边放间隔，后面依次放数字时钟、间

隔、锁定/注销、系统托盘。这样应用程序启动器就会固定在最左边





最终效果如下

![image-20240207164336310](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071643417.png)



应用程序启动器的图标要改成mac，右键点击应用程序启动器，选择配置应用程序启动器

![image-20240207164540551](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071645649.png)







修改图标，选择本地图标即可



![image-20240207164735851](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071647944.png)



附一个LOGO

![icons8-mac-os](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071649679.svg)





#### 安装latte-dock

要想实现和mac一样的dock效果，需要安装latte-dock，安装好之后，应用程序搜索latte-dock启动即可

```sh
sudo pacman -S latte-dock
```





启动之后效果如下，其他设置可以自行更改

![image-20240207165217483](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071652577.png)





#### 安装nautilus

安装nautilus替换dolphin，文件管理会更加美观

```sh
sudo pacman -S nautilus
```





然后点击系统设置-应用程序-默认应用程序-文件管理器，选择文件即可

![image-20240207165612819](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202402071656914.png)





#### 更换壁纸

桌面右键-配置桌面和壁纸，选择喜欢的壁纸即可





## 小结

经过如上的配置，一个基本可用的仿mac系统的manjaro就基本好了，后面有其他的更新就再补充





## 参考链接

- manjaro终端美化: https://segmentfault.com/a/1190000022863791
- manjaro配置: https://zhuanlan.zhihu.com/p/114296129
- manjaro配置: https://zhuanlan.zhihu.com/p/460826583
- manjaro配置: https://zhuanlan.zhihu.com/p/656733028
- manjaro配置: https://blog.lyh543.cn/posts/2021-09-30-config-manjaro.html
- kde仿mac美化: https://blog.csdn.net/Luo_Jin/article/details/88776326#t6
- deb转换: https://www.redren.net/8146.html
- deb换源: https://www.cnblogs.com/bluestorm/p/16988840.html
- fcitx5介绍: https://wiki.archlinuxcn.org/wiki/Fcitx5
- Clash: https://help.hitun.io/zh/article/linux-clash-for-windows-1xajysh/
