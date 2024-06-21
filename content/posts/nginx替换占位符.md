---
draft: false
date: 2024-01-18T09:09:37+08:00
title: "nginx替换占位符"
slug: "nginx-replace-placeholder" 
tags: ["apitable","nginx","k8s","placeholder"]
authors: ["since"]
description: "nginx配置文件使用占位符,读取环境变量替换占位符"
---

## 背景

此篇是apitable落地的第一篇，实现了nginx的可配。



最近在预研一个开源项目，打算实现本地化部署，在测试环境的k8s集群中部署所有的服务

那么流量入口是用的nginx，里面有server和upstream配置，也需要在k8s中部署

由于upstream下游是写死的，我这边的需求是能在测试环境，动态调用不同的服务。

如果按照代码里当前的配置，想调用不同的服务，那只能修改upstream地址，重新打镜像。

所以我这边的想法是只打一个镜像，通过读取环境变量的方式，替换占位符。

那么经过一番搜索，整理，最终实现了环境变量替换占位符的目标。

nginx原生是不支持这种方式的，所以搜索的过程还有点艰难。

## 项目信息

是一个开源的多维表格项目，代码地址: [apitable](https://github.com/apitable/apitable)

nginx配置信息在gateway/conf.d目录下



## 替换详细步骤

以[ups-backend-server-conf](https://github.com/apitable/apitable/blob/develop/gateway/conf.d/upstream/ups-backend-server.conf)为例，这个是访问java后端进程的地址配置

![image-20240118093552647](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202401180935766.png)

在k8s中，如果存在backend-server这个service其实也还行，但是我的目的是可以方便的访问本地服务实现调试。

所以默认的这种配置不大适合我，需要修改为占位符的方式。



但是nginx有一个问题 ，就是 nginx约定好的$host/$remote等变量也会被这个命令替换掉，替换掉后基本上都会造成问题。

所以占位符替换首要是要解决这么一个问题。

经过搜索之后，可以先拿到当前环境所有已定义的环境变量，使用文本处理，得到所有想要替换的环境变量。

此时使用envsubst指定替换功能，就不会替换掉$host/$remote等信息了。



下面介绍下实现在k8s部署nginx，使用环境变量替换占位符的详细步骤。

### 目录结构

在原有的目录结构上，新增了build_nginx.sh、docker-entrypoint.sh、Dockerfile这几个新文件。

修改了upstream下所有的server配置，修改为占位符，启动之后从环境变量读取替换。

![image-20240118094800484](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202401180948512.png)



### 修改配置文件

还是以以[ups-backend-server-conf](https://github.com/apitable/apitable/blob/develop/gateway/conf.d/upstream/ups-backend-server.conf)为例，进行修改

修改之后的配置文件内容如下

```nginx
#upstream backend  {
#   server backend-server:8081;
#}


upstream backend  {
  server ${BACKEND_SERVER};
}
```

配置文件中将固定地址修改为配置项

### docker build脚本

我们的目标是在k8s中部署Nginx，首先是要打个nginx镜像出来。

使用如下的docker build脚本,在命令行执行`./build_nginx.sh test`打镜像，推到镜像仓库，版本号可以自定义

```shell
#!/bin/bash

version="$1"

docker build --platform=linux/amd64 -t "registry.self.com/apitable/nginx-server:$version" .

docker push "registry.self.com/apitable/nginx-server:$version"

echo "image url: registry.self.com/apitable/nginx-server:$version"
```

### Dockerfile

使用如下的dockerfile生成自己的nginx镜像，替换环境变量的操作在docker-entrypoint.sh里执行

```dockerfile
FROM --platform=linux/amd64 nginx:latest

# 安装所需组件
RUN apt-get update && apt-get install -y vim net-tools telnet iputils-ping dnsutils


RUN mkdir -p "/etc/nginx/templates"

#添加配置
COPY conf.d/default.conf /etc/nginx/conf.d/default.conf
COPY conf.d     /etc/nginx/templates

RUN for f in /etc/nginx/templates/*/*.conf; do mv "$f" "${f%.conf}.conf.template"; done


#添加替换占位符脚本
COPY docker-entrypoint.sh /usr/local/bin
RUN chmod +x /usr/local/bin/docker-entrypoint.sh


EXPOSE 80

ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]

CMD ["nginx", "-g", "daemon off;"]
```

### docker-entrypoint.sh替换占位符

如下的是docker-entrypoint.sh里的内容，执行替换自定义占位符，不会替换nginx自带的占位符。

注意最后一行，不添加最后一行会导致Dockerfile里的cmd无法执行。

```shell
#!/bin/bash

set -e

ME=$(basename $0)

# 打开文件描述符 3，将输出定向到标准错误
exec 3>&2

auto_envsubst() {
  local template_dir="${NGINX_ENVSUBST_TEMPLATE_DIR:-/etc/nginx/templates}"
  local suffix="${NGINX_ENVSUBST_TEMPLATE_SUFFIX:-.template}"
  local output_dir="${NGINX_ENVSUBST_OUTPUT_DIR:-/etc/nginx/conf.d}"

  local template defined_envs relative_path output_path subdir
  # 这里拿到了所有定义的环境变量
  # [root@iZwz9hmxhr8nh716gmser4Z tmp]# printf '${%s} ' $(env | cut -d= -f1)
  # ${XDG_SESSION_ID} ${HOSTNAME} ${TERM} ${SHELL} ${HISTSIZE} ${SSH_CLIENT} ${NNHOST} ${OLDPWD} ${SSH_TTY} ${NGINX_HOST} ${USER} ${LS_COLORS} ${MAIL} ${PATH} ${PWD} ${LANG} ${HISTCONTROL} ${SHLVL} ${HOME} ${LOGNAME} ${SSH_CONNECTION} ${LESSOPEN} ${XDG_RUNTIME_DIR} ${_}
  defined_envs=$(env | cut -d= -f1 | awk '{printf "${%s} ", $1}')
  echo "defined_envs: $defined_envs"
  [ -d "$template_dir" ] || return 0
  if [ ! -w "$output_dir" ]; then
    echo >&3 "$ME: ERROR: $template_dir exists, but $output_dir is not writable"
    return 0
  fi
  find "$template_dir" -follow -type f -name "*$suffix" -print | while read -r template; do
    relative_path="${template#$template_dir/}"
    output_path="$output_dir/${relative_path%$suffix}"
    subdir=$(dirname "$relative_path")

    # 添加以下输出语句确认找到的文件
    echo "Found template file: $template, output_dir: $output_dir, subdir: $subdir"

    # create a subdirectory where the template file exists
    mkdir -p "$output_dir/$subdir"
    echo >&3 "$ME: Running envsubst on $template to $output_path"
    # 这里指定只替换定义的环境变量，因此不会覆盖掉nginx约定好的$host...等变量
    # 相当于执行了
    # envsubst "${XDG_SESSION_ID} ${HOSTNAME} ${TERM} ${SHELL} ${HISTSIZE} ${SSH_CLIENT} ${NNHOST} ${OLDPWD} ${SSH_TTY} ${NGINX_HOST} ${USER} ${LS_COLORS} ${MAIL} ${PATH} ${PWD} ${LANG} ${HISTCONTROL} ${SHLVL} ${HOME} ${LOGNAME} ${SSH_CONNECTION} ${LESSOPEN} ${XDG_RUNTIME_DIR} ${_}" <"$template" >"$output_path"
    envsubst "$defined_envs" <"$template" >"$output_path"

    echo "输出替换后的实际文件: $output_path"
    cat "$output_path"
  done
}

auto_envsubst
echo "begin start nginx"

#添加这一行, 继续执行CMD,不然会卡住失败
exec "$@"

```



### k8s nginx deployment

那么经过上面的脚本，可以打包一个可以自定义环境变量的nginx镜像，下面需要在k8s上面部署nginx镜像

deployment配置如下,  在启动之后使用环境变量替换了占位符

`backend-server-service.apitable.svc.cluster.local:8081`这种是k8s service访问的形式，指向了backend-server的deployment

关于service的详细信息可以见文档: [K8S DNS](https://kubernetes.feisky.xyz/setup/addon-list/kube-dns#zhi-chi-de-dns-ge-shi)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-server
  namespace: apitable  #指定namespace
  component: apitable nginx server
  labels:
    app: nginx-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-server
  template:
    metadata:
      labels:
        app: nginx-server
    spec:
      containers:
      - name: nginx
        image: registry.self.com/apitable/nginx-server:latest #替换为自己镜像地址
        imagePullPolicy: Always
        ports:
        - containerPort: 80
        env:
        - name: BACKEND_SERVER 
          value: "backend-server-service.apitable.svc.cluster.local:8081" 
        - name: IMAGE_PROXY_SERVER
          value: "imageproxy-server-service.apitable.svc.cluster.local:8080"
        - name: MINIO_SERVER
          value: "minio-service.apitable.svc.cluster.local:9000"
        - name: ROOM_SERVER
          value: "room-server-service.apitable.svc.cluster.local:3333"
        - name: ROOM_SERVER_SOCKET
          value: "room-server-service.apitable.svc.cluster.local:3002"
        - name: ROOM_SERVER_SOCKET_ROOM
          value: "room-server-service.apitable.svc.cluster.local:3005"
        - name: ROOM_SERVER_DOCUMENT
          value: "room-server-service.apitable.svc.cluster.local:3006"
        - name: WEB_SERVER
          value: "web-server-service.apitable.svc.cluster.local:8080"
```



service如下,使用nodeport进行访问，nodeport相当于在宿主机上起了个端口供外部访问，不然集群地址外部是访问不通的

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-server-service
spec:
  type: NodePort
  selector:
    app: nginx-server
  ports:
    - name: nginx-server
      protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30017
```

在k8s本地化部署过程中，配置了如下的多种service，用于调试测试

![image-20240118100316215](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202401181003266.png)



### 最终效果

进入pod，查看/etc/nginx/conf.d/upstream/ups-backend-server.conf

![image-20240118101000268](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202401181010353.png)
