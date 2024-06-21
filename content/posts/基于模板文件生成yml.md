---
draft: false
date: 2023-07-11T22:45:17+08:00
title: "基于模板文件生成yml"
slug: "generate-yml-based-on-template" 
tags: ["shell","yml","yq"]
authors: ["since"]
description: "使用shell,基于配置文件模板生成yml文件"
---

## 背景

我这边使用gohangout消费kafka，写入clickhouse，每次来新的需求，需要根据不同的evn，不同的日志来源，配置不同的topic/consumer/broker/password/table/fields等，所以想使用shell更新模板文件，根据不同的条件生成不同的yml配置文件

## 生成过程

### 交互式输入

使用shell，我就想着使用交互式输入，每需要一个变量，就终端输入一个，然后赋值到变量里，使用sed去替换，基本上topic/consumer/password等都能实现，但是在host列表那出现了问题。

交互式输入的简单栗子

```
#!/bin/bash

# 提示用户输入
echo "请输入env: "
read env

if [ ! -n "$env" ] ;then
    echo "请输入环境参数，可选参数列表如下: dev,fat,pro,pro-eco-cluster,pro-new-cluster"
    exit 1
fi
envValue=$env
#模拟的sed命令，按照实际情况替换
sed -i .bak "s/env_template/$envValue/g" "abc.txt"
```



### 列表的问题

yml中列表大概如下所示

```
outputs:
  - Clickhouse:
      database: "default"
      hosts:
        - 192.168.10.11:9000
        - 192.168.10.12:9000
```

如果模板文件如下

```
outputs:
  - Clickhouse:
      database: "default"
      hosts:
        #要替换的占位符
        host_list_template
```

如果按照上面的方式，host_list="- 192.168.10.10\n- 192.168.10.11",将host_list使用sed替换掉模板文件里的host_list_template，则最终生成的文件格式如下

```
outputs:
  - Clickhouse:
      database: "default"
      hosts:
        #要替换的占位符
        - 192.168.10.10
- 192.168.10.11
```

实际只有第一行会保持缩进，下面所有行缩进就乱了，这样的话yml格式会报错，需要人工去format一下。

由于一开始其他的普通字符串我都是用的sed，所以陷入了思维误区，满脑子都是想着怎么生成一个能保持缩进的字符串，然后替换掉占位符，再经过我一上午和chatgpt反复拉扯，无意义的对话之后，终于还是没能生成保持缩进的字符串，chatgpt误我啊。

下午继续战斗的时候，发现了一个yml命令行工具y1，这个命令可以读取/修改yml文件

使用brew install yq，即可使用

使用yq可以给指定的key赋值，这样就避免自己手动去保持缩进格式了，遍历list，给key赋值即可解决，那么理论上上面所有的普通字符串都可以使用yq去赋值，而不用写占位符去sed替换了。

一个更新示例如下

模板文件file.yml

```
outputs:
  - Clickhouse:
      database: "default"
      hosts:
    user: "default"
    password: "a"
    table: "a"
    bulk_actions: 100000
    flush_interval: 8
    concurrent: 4
    fields: ['appname', 'className']

```

更新脚本test.sh

```
#!/bin/bash

# 定义多个列表元素
list1=("element1" "element2")
list2=("element3" "element4" "element5")
list3=("element6")

# 根据条件选择不同的列表元素
if [[ "$1" == "dev" ]]; then
    selected_list=("${list1[@]}")
elif [[ "$1" == "fat" ]]; then
    selected_list=("${list2[@]}")
else
    selected_list=("${list3[@]}")
fi


# 输出结果
echo "(${selected_list[*]})"


# 逐个添加列表元素
for element in "${selected_list[@]}"; do
  yq eval --inplace '.outputs[0].Clickhouse.hosts += ["'"$element"'"]' file.yml
done
```

最终生成的结果如下

传入fat

```
outputs:
  - Clickhouse:
      database: "default"
      hosts:
        - element3
        - element4
        - element5
    user: "default"
    password: "a"
    table: "a"
    bulk_actions: 100000
    flush_interval: 8
    concurrent: 4
    fields: ['appname', 'className']
```

## 总结

shell中修改yml，还得是包装好的工具省事，强烈推荐yq,配合文档和chatgpt熟悉用法
