---
draft: false
date: 2024-01-19T10:37:12+08:00
title: "apitable从docker compose到k8s deployment的转变"
slug: "apitable-from-docker-compose-to-k8s-deployment" 
tags: ["apitable","docker-compose","k8s-deployment"]
authors: ["since"]
description: "实在docker-compose到k8s deployment的转变"
---

此篇是apitable系列文章的第2篇，如何实现从本地的docker-compose部署到k8s集群使用deployment部署

附: [apitable代码地址](https://github.com/apitable/apitable)



## 背景

apitable提供了makefile实现了本地环境部署，但是我们的目标是落地测试环境和生产环境。

为了实现可移植性和可扩展性，刚好我们有一个k8s集群，所以决定在k8s上进行部署



## 过程

Quick start里本地启动介绍了2步，make dataenv以及make run，就可以本地启动了。那么makefile会有一些比较复杂的逻辑。

其中make dataenv是使用docker-compose启动了一堆的依赖组件，MySQL、Redis、RabbitMQ、Minio是通过dataenv启动的。

那么由于它这个makefile比较复杂，docker-compose文件也比较多，makefile最终执行完启动, 环境变量隐藏在里面启动了

那么在这种情况下怎么做转换呢

![image-20240119110009668](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202401191100776.png)



### 生成docker-compose预览

在询问GPT之后，基于make dataenv的执行过程，给了我如下的一个命令

```
docker compose --env-file .env -f docker-compose.yaml -f docker-compose.dataenv.yaml config --resolve-image-digests
```

执行之后可以看到在控制台生成了一份填充好环境变量的yaml文件

![image-20240119110310462](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202401191103489.png)

yaml详细内容如下

```yaml
name: apitable
services:
  backend-server:
    depends_on:
      init-db:
        condition: service_completed_successfully
        required: true
    environment:
      API_PROXY: http://backend-server:8081
      ASSETS_BUCKET: assets
      ASSETS_URL: assets
      AWS_ACCESS_KEY: apitable
      AWS_ACCESS_SECRET: apitable@com
      AWS_ENDPOINT: http://minio:9000
      AWS_REGION: us-east-1
      BACKEND_BASE_URL: http://backend-server:8081/api/v1/
      BACKEND_INFO_URL: http://backend-server:8081/api/v1/client/info
      CALLBACK_DOMAIN: ""
      CORS_ORIGINS: '*'
      DATA_PATH: .
      DATABASE_TABLE_PREFIX: apitable_
      DATABUS_SERVER_BASE_URL: http://databus-server:8625
      DB_HOST: mysql
      DB_NAME: apitable
      DB_PASSWORD: apitable@com
      DB_PORT: "3306"
      DB_USERNAME: root
      DEFAULT_TIME_ZONE: Asia/Singapore
      ENV: apitable
      HUAWEICLOUD_OBS_ACCESS_KEY: apitable
      HUAWEICLOUD_OBS_ENDPOINT: obs.cn-south-1.myhuaweicloud.com
      HUAWEICLOUD_OBS_SECRET_KEY: apitable@com
      IMAGE_BACKEND_SERVER: apitable/backend-server:latest
      IMAGE_DATABUS_SERVER: apitable/databus-server:latest
      IMAGE_GATEWAY: apitable/openresty:latest
      IMAGE_IMAGEPROXY_SERVER: apitable/imageproxy-server:latest
      IMAGE_INIT_APPDATA: apitable/init-appdata:latest
      IMAGE_INIT_DB: apitable/init-db:latest
      IMAGE_MINIO: minio/minio:RELEASE.2023-01-25T00-19-54Z
      IMAGE_MYSQL: mysql:8.0.32
      IMAGE_PULL_POLICY: always
      IMAGE_RABBITMQ: rabbitmq:3.11.9-management
      IMAGE_REDIS: redis:7.0.8
      IMAGE_REGISTRY: docker.io
      IMAGE_ROOM_SERVER: apitable/room-server:latest
      IMAGE_WEB_SERVER: apitable/web-server:latest
      MAIL_ENABLED: "false"
      MAIL_FROM: ""
      MAIL_HOST: ""
      MAIL_PASSWORD: ""
      MAIL_PORT: ""
      MAIL_SSL_ENABLE: "true"
      MAIL_TYPE: smtp
      MAIL_USERNAME: ""
      MINIO_ACCESS_KEY: apitable
      MINIO_SECRET_KEY: apitable@com
      MYSQL_DATABASE: apitable
      MYSQL_HOST: mysql
      MYSQL_PASSWORD: apitable@com
      MYSQL_PORT: "3306"
      MYSQL_ROOT_PASSWORD: apitable@com
      MYSQL_USERNAME: root
      NEST_GRPC_ADDRESS: static://room-server:3334
      NGINX_HTTP_PORT: "80"
      NGINX_HTTPS_PORT: "443"
      OSS_BUCKET_NAME: assets
      OSS_CACHE_TYPE: minio
      OSS_CLIENT_TYPE: aws
      OSS_ENABLED: "true"
      OSS_HOST: assets
      OSS_TYPE: QNY1
      PUBLIC_URL: ""
      RABBITMQ_HOST: rabbitmq
      RABBITMQ_PASSWORD: apitable@com
      RABBITMQ_PORT: "5672"
      RABBITMQ_USERNAME: apitable
      RABBITMQ_VHOST: /
      REDIS_DB: "0"
      REDIS_HOST: redis
      REDIS_PASSWORD: apitable@com
      REDIS_PORT: "6379"
      ROOM_GRPC_URL: room-server:3334
      SERVER_DOMAIN: ""
      SKIP_REGISTER_VALIDATE: "true"
      SMS_ENABLED: "false"
      SOCKET_DOMAIN: http://room-server:3333/socket
      SOCKET_GRPC_URL: room-server:3007
      SOCKET_RECONNECTION_ATTEMPTS: "10"
      SOCKET_RECONNECTION_DELAY: "500"
      SOCKET_TIMEOUT: "5000"
      SOCKET_URL: http://room-server:3002
      TEMPLATE_PATH: ./static/web_build/index.html
      TEMPLATE_SPACE: spcNTxlv8Drra
      TIMEZONE: Asia/Singapore
      TZ: Asia/Singapore
      USE_CUSTOM_PUBLIC_FILES: "true"
      USE_NATIVE_MODULE: "0"
    expose:
      - "8081"
    healthcheck:
      test:
        - CMD-SHELL
        - curl -sS 'http://localhost:8081' || exit 1
      timeout: 5s
      interval: 5s
      retries: 60
      start_period: 30s
    image: docker.io/apitable/backend-server:latest@sha256:8f5ba8f8bdee76c7981d92e8e9309403e8312c0795e7cac71cea2f8763f15bee
    networks:
      apitable: null
    pull_policy: always
    restart: always
  databus-server:
    depends_on:
      mysql:
        condition: service_healthy
        required: true
    environment:
      API_PROXY: http://backend-server:8081
      ASSETS_BUCKET: assets
      ASSETS_URL: assets
      AWS_ACCESS_KEY: apitable
      AWS_ACCESS_SECRET: apitable@com
      AWS_ENDPOINT: http://minio:9000
      AWS_REGION: us-east-1
      BACKEND_BASE_URL: http://backend-server:8081/api/v1/
      BACKEND_INFO_URL: http://backend-server:8081/api/v1/client/info
      CALLBACK_DOMAIN: ""
      CORS_ORIGINS: '*'
      DATA_PATH: .
      DATABASE_TABLE_PREFIX: apitable_
      DATABUS_SERVER_BASE_URL: http://databus-server:8625
      DB_HOST: mysql
      DB_NAME: apitable
      DB_PASSWORD: apitable@com
      DB_PORT: "3306"
      DB_USERNAME: root
      ENV: apitable
      HUAWEICLOUD_OBS_ACCESS_KEY: apitable
      HUAWEICLOUD_OBS_ENDPOINT: obs.cn-south-1.myhuaweicloud.com
      HUAWEICLOUD_OBS_SECRET_KEY: apitable@com
      IMAGE_BACKEND_SERVER: apitable/backend-server:latest
      IMAGE_DATABUS_SERVER: apitable/databus-server:latest
      IMAGE_GATEWAY: apitable/openresty:latest
      IMAGE_IMAGEPROXY_SERVER: apitable/imageproxy-server:latest
      IMAGE_INIT_APPDATA: apitable/init-appdata:latest
      IMAGE_INIT_DB: apitable/init-db:latest
      IMAGE_MINIO: minio/minio:RELEASE.2023-01-25T00-19-54Z
      IMAGE_MYSQL: mysql:8.0.32
      IMAGE_PULL_POLICY: always
      IMAGE_RABBITMQ: rabbitmq:3.11.9-management
      IMAGE_REDIS: redis:7.0.8
      IMAGE_REGISTRY: docker.io
      IMAGE_ROOM_SERVER: apitable/room-server:latest
      IMAGE_WEB_SERVER: apitable/web-server:latest
      MAIL_ENABLED: "false"
      MAIL_FROM: ""
      MAIL_HOST: ""
      MAIL_PASSWORD: ""
      MAIL_PORT: ""
      MAIL_SSL_ENABLE: "true"
      MAIL_TYPE: smtp
      MAIL_USERNAME: ""
      MINIO_ACCESS_KEY: apitable
      MINIO_SECRET_KEY: apitable@com
      MYSQL_DATABASE: apitable
      MYSQL_HOST: mysql
      MYSQL_PASSWORD: apitable@com
      MYSQL_PORT: "3306"
      MYSQL_ROOT_PASSWORD: apitable@com
      MYSQL_USERNAME: root
      NEST_GRPC_ADDRESS: static://room-server:3334
      NGINX_HTTP_PORT: "80"
      NGINX_HTTPS_PORT: "443"
      OSS_BUCKET_NAME: assets
      OSS_CACHE_TYPE: minio
      OSS_CLIENT_TYPE: aws
      OSS_ENABLED: "true"
      OSS_HOST: assets
      OSS_TYPE: QNY1
      PUBLIC_URL: ""
      RABBITMQ_HOST: rabbitmq
      RABBITMQ_PASSWORD: apitable@com
      RABBITMQ_PORT: "5672"
      RABBITMQ_USERNAME: apitable
      RABBITMQ_VHOST: /
      REDIS_DB: "0"
      REDIS_HOST: redis
      REDIS_PASSWORD: apitable@com
      REDIS_PORT: "6379"
      ROOM_GRPC_URL: room-server:3334
      SERVER_DOMAIN: ""
      SKIP_REGISTER_VALIDATE: "true"
      SMS_ENABLED: "false"
      SOCKET_DOMAIN: http://room-server:3333/socket
      SOCKET_GRPC_URL: room-server:3007
      SOCKET_RECONNECTION_ATTEMPTS: "10"
      SOCKET_RECONNECTION_DELAY: "500"
      SOCKET_TIMEOUT: "5000"
      SOCKET_URL: http://room-server:3002
      TEMPLATE_PATH: ./static/web_build/index.html
      TEMPLATE_SPACE: spcNTxlv8Drra
      TIMEZONE: Asia/Singapore
      TZ: Asia/Singapore
      USE_CUSTOM_PUBLIC_FILES: "true"
      USE_NATIVE_MODULE: "0"
    expose:
      - "8625"
    image: docker.io/apitable/databus-server:latest@sha256:462fa8bea11df94642b80a58d683aaf9995d79061588843a9d6b7ee66a421600
    networks:
      apitable: null
    ports:
      - mode: ingress
        target: 8625
        published: "8625"
        protocol: tcp
    pull_policy: always
    restart: always
  gateway:
    depends_on:
      backend-server:
        condition: service_healthy
        required: true
      imageproxy-server:
        condition: service_started
        required: true
      init-appdata:
        condition: service_completed_successfully
        required: true
      room-server:
        condition: service_started
        required: true
      web-server:
        condition: service_started
        required: true
    environment:
      TZ: Asia/Singapore
    image: docker.io/apitable/openresty:latest@sha256:fea46da296a1f156cebf9d26e9b40c781690479fb7629dafb5f1411494b669a8
    networks:
      apitable: null
    ports:
      - mode: ingress
        target: 80
        published: "80"
        protocol: tcp
      - mode: ingress
        target: 443
        published: "443"
        protocol: tcp
    pull_policy: always
    restart: always
  imageproxy-server:
    environment:
      BASEURL: http://minio:9000
      TZ: Asia/Singapore
    expose:
      - "8080"
    image: docker.io/apitable/imageproxy-server:latest@sha256:09ce6fc6817c99247e40cfb52489f217481a867b45d308d9525e13d38aa1e340
    networks:
      apitable: null
    pull_policy: always
    restart: always
  init-appdata:
    depends_on:
      init-db:
        condition: service_completed_successfully
        required: true
      mysql:
        condition: service_healthy
        required: true
    environment:
      API_PROXY: http://backend-server:8081
      ASSETS_BUCKET: assets
      ASSETS_URL: assets
      AWS_ACCESS_KEY: apitable
      AWS_ACCESS_SECRET: apitable@com
      AWS_ENDPOINT: http://minio:9000
      AWS_REGION: us-east-1
      BACKEND_BASE_URL: http://backend-server:8081/api/v1/
      BACKEND_INFO_URL: http://backend-server:8081/api/v1/client/info
      CALLBACK_DOMAIN: ""
      CORS_ORIGINS: '*'
      DATA_PATH: .
      DATABASE_TABLE_PREFIX: apitable_
      DATABUS_SERVER_BASE_URL: http://databus-server:8625
      DB_HOST: mysql
      DB_NAME: apitable
      DB_PASSWORD: apitable@com
      DB_PORT: "3306"
      DB_USERNAME: root
      ENV: apitable
      HUAWEICLOUD_OBS_ACCESS_KEY: apitable
      HUAWEICLOUD_OBS_ENDPOINT: obs.cn-south-1.myhuaweicloud.com
      HUAWEICLOUD_OBS_SECRET_KEY: apitable@com
      IMAGE_BACKEND_SERVER: apitable/backend-server:latest
      IMAGE_DATABUS_SERVER: apitable/databus-server:latest
      IMAGE_GATEWAY: apitable/openresty:latest
      IMAGE_IMAGEPROXY_SERVER: apitable/imageproxy-server:latest
      IMAGE_INIT_APPDATA: apitable/init-appdata:latest
      IMAGE_INIT_DB: apitable/init-db:latest
      IMAGE_MINIO: minio/minio:RELEASE.2023-01-25T00-19-54Z
      IMAGE_MYSQL: mysql:8.0.32
      IMAGE_PULL_POLICY: always
      IMAGE_RABBITMQ: rabbitmq:3.11.9-management
      IMAGE_REDIS: redis:7.0.8
      IMAGE_REGISTRY: docker.io
      IMAGE_ROOM_SERVER: apitable/room-server:latest
      IMAGE_WEB_SERVER: apitable/web-server:latest
      MAIL_ENABLED: "false"
      MAIL_FROM: ""
      MAIL_HOST: ""
      MAIL_PASSWORD: ""
      MAIL_PORT: ""
      MAIL_SSL_ENABLE: "true"
      MAIL_TYPE: smtp
      MAIL_USERNAME: ""
      MINIO_ACCESS_KEY: apitable
      MINIO_SECRET_KEY: apitable@com
      MYSQL_DATABASE: apitable
      MYSQL_HOST: mysql
      MYSQL_PASSWORD: apitable@com
      MYSQL_PORT: "3306"
      MYSQL_ROOT_PASSWORD: apitable@com
      MYSQL_USERNAME: root
      NEST_GRPC_ADDRESS: static://room-server:3334
      NGINX_HTTP_PORT: "80"
      NGINX_HTTPS_PORT: "443"
      OSS_BUCKET_NAME: assets
      OSS_CACHE_TYPE: minio
      OSS_CLIENT_TYPE: aws
      OSS_ENABLED: "true"
      OSS_HOST: assets
      OSS_TYPE: QNY1
      PUBLIC_URL: ""
      RABBITMQ_HOST: rabbitmq
      RABBITMQ_PASSWORD: apitable@com
      RABBITMQ_PORT: "5672"
      RABBITMQ_USERNAME: apitable
      RABBITMQ_VHOST: /
      REDIS_DB: "0"
      REDIS_HOST: redis
      REDIS_PASSWORD: apitable@com
      REDIS_PORT: "6379"
      ROOM_GRPC_URL: room-server:3334
      SERVER_DOMAIN: ""
      SKIP_REGISTER_VALIDATE: "true"
      SMS_ENABLED: "false"
      SOCKET_DOMAIN: http://room-server:3333/socket
      SOCKET_GRPC_URL: room-server:3007
      SOCKET_RECONNECTION_ATTEMPTS: "10"
      SOCKET_RECONNECTION_DELAY: "500"
      SOCKET_TIMEOUT: "5000"
      SOCKET_URL: http://room-server:3002
      TEMPLATE_PATH: ./static/web_build/index.html
      TEMPLATE_SPACE: spcNTxlv8Drra
      TIMEZONE: Asia/Singapore
      USE_CUSTOM_PUBLIC_FILES: "true"
      USE_NATIVE_MODULE: "0"
    image: docker.io/apitable/init-appdata:latest@sha256:4fa2ed5d1a5a3e2f7bd449352ec3054747127aabe4f91c62fc13f4660b25558b
    networks:
      apitable: null
    pull_policy: always
  init-db:
    depends_on:
      mysql:
        condition: service_healthy
        required: true
    environment:
      ACTION: update
      DATABASE_TABLE_PREFIX: apitable_
      DB_HOST: mysql
      DB_NAME: apitable
      DB_PASSWORD: apitable@com
      DB_PORT: "3306"
      DB_USERNAME: root
      TZ: Asia/Singapore
    image: docker.io/apitable/init-db:latest@sha256:31cdae8220bdda327e773c2854ced625704203c043774b0992bb13dbb510ae6c
    networks:
      apitable: null
    pull_policy: always
  minio:
    command:
      - server
      - --console-address
      - :9001
      - /data
    container_name: minio
    environment:
      MINIO_ACCESS_KEY: apitable
      MINIO_ROOT_PASSWORD: apitable@com
      MINIO_ROOT_USER: apitable
      MINIO_SECRET_KEY: apitable@com
      TZ: Asia/Singapore
    expose:
      - "9000"
      - "9001"
    healthcheck:
      test:
        - CMD-SHELL
        - curl -sS 'http://localhost:9000' || exit 1
      timeout: 5s
      interval: 5s
      retries: 30
    image: docker.io/minio/minio:RELEASE.2023-01-25T00-19-54Z@sha256:6a125ee7860f387d4b1d9aface8be01413e2a9f803508d23b5e9dd64b833e7a7
    networks:
      apitable: null
    ports:
      - mode: ingress
        target: 9000
        published: "9000"
        protocol: tcp
      - mode: ingress
        target: 9001
        published: "9001"
        protocol: tcp
    pull_policy: always
    restart: always
    volumes:
      - type: bind
        source: /Users/since/zto/APITable/.data/minio/data
        target: /data
        bind:
          create_host_path: true
      - type: bind
        source: /Users/since/zto/APITable/.data/minio/config
        target: /root/.minio
        bind:
          create_host_path: true
  mysql:
    command:
      - --default-authentication-plugin=mysql_native_password
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_general_ci
      - --sql_mode=IGNORE_SPACE,NO_ENGINE_SUBSTITUTION
      - --lower_case_table_names=2
    container_name: mysql
    environment:
      MYSQL_DATABASE: apitable
      MYSQL_ROOT_PASSWORD: apitable@com
      TZ: Asia/Singapore
    expose:
      - "3306"
    healthcheck:
      test:
        - CMD-SHELL
        - mysql apitable -uroot -papitable@com -e 'SELECT 1;'
      timeout: 5s
      interval: 5s
      retries: 60
      start_period: 30s
    image: docker.io/library/mysql:8.0.32@sha256:f496c25da703053a6e0717f1d52092205775304ea57535cc9fcaa6f35867800b
    networks:
      apitable: null
    ports:
      - mode: ingress
        target: 3306
        published: "3306"
        protocol: tcp
    pull_policy: always
    restart: always
    volumes:
      - type: bind
        source: /Users/since/zto/APITable/.data/mysql
        target: /var/lib/mysql
        bind:
          create_host_path: true
  rabbitmq:
    container_name: rabbitmq
    environment:
      RABBITMQ_DEFAULT_PASS: apitable@com
      RABBITMQ_DEFAULT_USER: apitable
    expose:
      - "5671"
      - "5672"
      - "15672"
    image: docker.io/library/rabbitmq:3.11.9-management@sha256:742359d27f34c379585f46645a5b5a9a8e61877b7b3d8dc009b57701ab4b62c2
    networks:
      apitable: null
    ports:
      - mode: ingress
        target: 5671
        published: "5671"
        protocol: tcp
      - mode: ingress
        target: 5672
        published: "5672"
        protocol: tcp
      - mode: ingress
        target: 15672
        published: "15672"
        protocol: tcp
    pull_policy: always
    restart: always
    volumes:
      - type: bind
        source: /Users/since/zto/APITable/.data/rabbitmq
        target: /var/lib/rabbitmq
        bind:
          create_host_path: true
  redis:
    command:
      - redis-server
      - --appendonly
      - "yes"
      - --requirepass
      - apitable@com
    container_name: redis
    environment:
      TZ: Asia/Singapore
    expose:
      - "6379"
    image: docker.io/library/redis:7.0.8@sha256:6a59f1cbb8d28ac484176d52c473494859a512ddba3ea62a547258cf16c9b3ae
    networks:
      apitable: null
    ports:
      - mode: ingress
        target: 6379
        published: "6379"
        protocol: tcp
    pull_policy: always
    restart: always
    volumes:
      - type: bind
        source: /Users/since/zto/APITable/.data/redis
        target: /data
        bind:
          create_host_path: true
  room-server:
    depends_on:
      mysql:
        condition: service_healthy
        required: true
    environment:
      API_MAX_MODIFY_RECORD_COUNTS: "30"
      API_PROXY: http://backend-server:8081
      ASSETS_BUCKET: assets
      ASSETS_URL: assets
      AWS_ACCESS_KEY: apitable
      AWS_ACCESS_SECRET: apitable@com
      AWS_ENDPOINT: http://minio:9000
      AWS_REGION: us-east-1
      BACKEND_BASE_URL: http://backend-server:8081/api/v1/
      BACKEND_INFO_URL: http://backend-server:8081/api/v1/client/info
      CALLBACK_DOMAIN: ""
      CORS_ORIGINS: '*'
      DATA_PATH: .
      DATABASE_TABLE_PREFIX: apitable_
      DATABUS_SERVER_BASE_URL: http://databus-server:8625
      DB_HOST: mysql
      DB_NAME: apitable
      DB_PASSWORD: apitable@com
      DB_PORT: "3306"
      DB_USERNAME: root
      ENABLE_SOCKET: "true"
      ENV: apitable
      HUAWEICLOUD_OBS_ACCESS_KEY: apitable
      HUAWEICLOUD_OBS_ENDPOINT: obs.cn-south-1.myhuaweicloud.com
      HUAWEICLOUD_OBS_SECRET_KEY: apitable@com
      IMAGE_BACKEND_SERVER: apitable/backend-server:latest
      IMAGE_DATABUS_SERVER: apitable/databus-server:latest
      IMAGE_GATEWAY: apitable/openresty:latest
      IMAGE_IMAGEPROXY_SERVER: apitable/imageproxy-server:latest
      IMAGE_INIT_APPDATA: apitable/init-appdata:latest
      IMAGE_INIT_DB: apitable/init-db:latest
      IMAGE_MINIO: minio/minio:RELEASE.2023-01-25T00-19-54Z
      IMAGE_MYSQL: mysql:8.0.32
      IMAGE_PULL_POLICY: always
      IMAGE_RABBITMQ: rabbitmq:3.11.9-management
      IMAGE_REDIS: redis:7.0.8
      IMAGE_REGISTRY: docker.io
      IMAGE_ROOM_SERVER: apitable/room-server:latest
      IMAGE_WEB_SERVER: apitable/web-server:latest
      INSTANCE_MAX_MEMORY: 4096M
      MAIL_ENABLED: "false"
      MAIL_FROM: ""
      MAIL_HOST: ""
      MAIL_PASSWORD: ""
      MAIL_PORT: ""
      MAIL_SSL_ENABLE: "true"
      MAIL_TYPE: smtp
      MAIL_USERNAME: ""
      MINIO_ACCESS_KEY: apitable
      MINIO_SECRET_KEY: apitable@com
      MYSQL_DATABASE: apitable
      MYSQL_HOST: mysql
      MYSQL_PASSWORD: apitable@com
      MYSQL_PORT: "3306"
      MYSQL_ROOT_PASSWORD: apitable@com
      MYSQL_USERNAME: root
      NEST_GRPC_ADDRESS: static://room-server:3334
      NGINX_HTTP_PORT: "80"
      NGINX_HTTPS_PORT: "443"
      NODE_ENV: apitable
      NODE_OPTIONS: --max-old-space-size=2048 --max-http-header-size=80000
      OSS_BUCKET_NAME: assets
      OSS_CACHE_TYPE: minio
      OSS_CLIENT_TYPE: aws
      OSS_ENABLED: "true"
      OSS_HOST: assets
      OSS_TYPE: QNY1
      PUBLIC_URL: ""
      RABBITMQ_HOST: rabbitmq
      RABBITMQ_PASSWORD: apitable@com
      RABBITMQ_PORT: "5672"
      RABBITMQ_USERNAME: apitable
      RABBITMQ_VHOST: /
      REDIS_DB: "0"
      REDIS_HOST: redis
      REDIS_PASSWORD: apitable@com
      REDIS_PORT: "6379"
      ROOM_GRPC_URL: room-server:3334
      SERVER_DOMAIN: ""
      SKIP_REGISTER_VALIDATE: "true"
      SMS_ENABLED: "false"
      SOCKET_DOMAIN: http://room-server:3333/socket
      SOCKET_GRPC_URL: room-server:3007
      SOCKET_RECONNECTION_ATTEMPTS: "10"
      SOCKET_RECONNECTION_DELAY: "500"
      SOCKET_TIMEOUT: "5000"
      SOCKET_URL: http://room-server:3002
      TEMPLATE_PATH: ./static/web_build/index.html
      TEMPLATE_SPACE: spcNTxlv8Drra
      TIMEZONE: Asia/Singapore
      TZ: Asia/Singapore
      USE_CUSTOM_PUBLIC_FILES: "true"
      USE_NATIVE_MODULE: "0"
    expose:
      - "3333"
      - "3334"
      - "3001"
      - "3002"
      - "3006"
      - "3005"
      - "3007"
    image: docker.io/apitable/room-server:latest@sha256:7fe50c091022eab7d8066e293c8c8543e064cb39f71e91fe5d5291acd2e85a72
    networks:
      apitable: null
    pull_policy: always
    restart: always
  web-server:
    environment:
      API_PROXY: http://backend-server:8081
      ASSETS_BUCKET: assets
      ASSETS_URL: assets
      AWS_ACCESS_KEY: apitable
      AWS_ACCESS_SECRET: apitable@com
      AWS_ENDPOINT: http://minio:9000
      AWS_REGION: us-east-1
      BACKEND_BASE_URL: http://backend-server:8081/api/v1/
      BACKEND_INFO_URL: http://backend-server:8081/api/v1/client/info
      CALLBACK_DOMAIN: ""
      CORS_ORIGINS: '*'
      DATA_PATH: .
      DATABASE_TABLE_PREFIX: apitable_
      DATABUS_SERVER_BASE_URL: http://databus-server:8625
      DB_HOST: mysql
      DB_NAME: apitable
      DB_PASSWORD: apitable@com
      DB_PORT: "3306"
      DB_USERNAME: root
      ENV: apitable
      HUAWEICLOUD_OBS_ACCESS_KEY: apitable
      HUAWEICLOUD_OBS_ENDPOINT: obs.cn-south-1.myhuaweicloud.com
      HUAWEICLOUD_OBS_SECRET_KEY: apitable@com
      IMAGE_BACKEND_SERVER: apitable/backend-server:latest
      IMAGE_DATABUS_SERVER: apitable/databus-server:latest
      IMAGE_GATEWAY: apitable/openresty:latest
      IMAGE_IMAGEPROXY_SERVER: apitable/imageproxy-server:latest
      IMAGE_INIT_APPDATA: apitable/init-appdata:latest
      IMAGE_INIT_DB: apitable/init-db:latest
      IMAGE_MINIO: minio/minio:RELEASE.2023-01-25T00-19-54Z
      IMAGE_MYSQL: mysql:8.0.32
      IMAGE_PULL_POLICY: always
      IMAGE_RABBITMQ: rabbitmq:3.11.9-management
      IMAGE_REDIS: redis:7.0.8
      IMAGE_REGISTRY: docker.io
      IMAGE_ROOM_SERVER: apitable/room-server:latest
      IMAGE_WEB_SERVER: apitable/web-server:latest
      MAIL_ENABLED: "false"
      MAIL_FROM: ""
      MAIL_HOST: ""
      MAIL_PASSWORD: ""
      MAIL_PORT: ""
      MAIL_SSL_ENABLE: "true"
      MAIL_TYPE: smtp
      MAIL_USERNAME: ""
      MINIO_ACCESS_KEY: apitable
      MINIO_SECRET_KEY: apitable@com
      MYSQL_DATABASE: apitable
      MYSQL_HOST: mysql
      MYSQL_PASSWORD: apitable@com
      MYSQL_PORT: "3306"
      MYSQL_ROOT_PASSWORD: apitable@com
      MYSQL_USERNAME: root
      NEST_GRPC_ADDRESS: static://room-server:3334
      NGINX_HTTP_PORT: "80"
      NGINX_HTTPS_PORT: "443"
      OSS_BUCKET_NAME: assets
      OSS_CACHE_TYPE: minio
      OSS_CLIENT_TYPE: aws
      OSS_ENABLED: "true"
      OSS_HOST: assets
      OSS_TYPE: QNY1
      PUBLIC_URL: ""
      RABBITMQ_HOST: rabbitmq
      RABBITMQ_PASSWORD: apitable@com
      RABBITMQ_PORT: "5672"
      RABBITMQ_USERNAME: apitable
      RABBITMQ_VHOST: /
      REDIS_DB: "0"
      REDIS_HOST: redis
      REDIS_PASSWORD: apitable@com
      REDIS_PORT: "6379"
      ROOM_GRPC_URL: room-server:3334
      SERVER_DOMAIN: ""
      SKIP_REGISTER_VALIDATE: "true"
      SMS_ENABLED: "false"
      SOCKET_DOMAIN: http://room-server:3333/socket
      SOCKET_GRPC_URL: room-server:3007
      SOCKET_RECONNECTION_ATTEMPTS: "10"
      SOCKET_RECONNECTION_DELAY: "500"
      SOCKET_TIMEOUT: "5000"
      SOCKET_URL: http://room-server:3002
      TEMPLATE_PATH: ./static/web_build/index.html
      TEMPLATE_SPACE: spcNTxlv8Drra
      TIMEZONE: Asia/Singapore
      USE_CUSTOM_PUBLIC_FILES: "true"
      USE_NATIVE_MODULE: "0"
    expose:
      - "8080"
    image: docker.io/apitable/web-server:latest@sha256:bc4c11eb1faec60f293c3392aa69e85f6fbc87817c15358917335053dcfade8f
    networks:
      apitable: null
    pull_policy: always
    restart: always
networks:
  apitable:
    name: apitable_default
    driver: bridge
```



### 转换

那么有了yaml之后，就可以方便的执行转换了，因为我们已经拿到了启动各个pod需要的环境变量，虽然有些是冗余无效的配置，但是前

期我们的目标是能在k8s上部署，后续配置我们再优化

那么接下来把yaml发给GPT, GPT会生成一份简陋的配置，环境变量没贴完整

![image-20240119110703050](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202401191107116.png)





这里deployment的配置是残缺的

![image-20240119110746140](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202401191107188.png)





在经过多次询问之后，最终完整的将所有的environment转换成了env



![image-20240119111057765](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202401191110807.png)

![image-20240119111114914](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202401191111957.png)

![image-20240119111130864](https://cdn.jsdelivr.net/gh/thend03/mdPic/picGo/202401191111909.png)



最终生成了如下的yaml配置

```yaml
apiVersion: v1
kind: Service
metadata:
  name: your-service
spec:
  selector:
    app: your-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080  # 适当替换为你的容器端口

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: your-deployment
spec:
  replicas: 1  # 根据需要的副本数进行调整
  selector:
    matchLabels:
      app: your-app
  template:
    metadata:
      labels:
        app: your-app
    spec:
      containers:
      - name: your-container
        image: your-image
        ports:
        - containerPort: 8080  # 适当替换为你的容器端口
        env:
        - name: API_MAX_MODIFY_RECORD_COUNTS
          value: "30"
        - name: API_PROXY
          value: "http://backend-server:8081"
        - name: ASSETS_BUCKET
          value: "assets"
        - name: ASSETS_URL
          value: "assets"
        - name: AWS_ACCESS_KEY
          value: "apitable"
        - name: AWS_ACCESS_SECRET
          value: "apitable@com"
        - name: AWS_ENDPOINT
          value: "http://minio:9000"
        - name: AWS_REGION
          value: "us-east-1"
        - name: BACKEND_BASE_URL
          value: "http://backend-server:8081/api/v1/"
        - name: BACKEND_INFO_URL
          value: "http://backend-server:8081/api/v1/client/info"
        - name: CALLBACK_DOMAIN
          value: ""
        - name: CORS_ORIGINS
          value: '*'
        - name: DATA_PATH
          value: "."
        - name: DATABASE_TABLE_PREFIX
          value: "apitable_"
        - name: DATABUS_SERVER_BASE_URL
          value: "http://databus-server:8625"
        - name: DB_HOST
          value: "mysql"
        - name: DB_NAME
          value: "apitable"
        - name: DB_PASSWORD
          value: "apitable@com"
        - name: DB_PORT
          value: "3306"
        - name: DB_USERNAME
          value: "root"
        - name: ENABLE_SOCKET
          value: "true"
        - name: ENV
          value: "apitable"
        - name: HUAWEICLOUD_OBS_ACCESS_KEY
          value: "apitable"
        - name: HUAWEICLOUD_OBS_ENDPOINT
          value: "obs.cn-south-1.myhuaweicloud.com"
        - name: HUAWEICLOUD_OBS_SECRET_KEY
          value: "apitable@com"
        - name: IMAGE_BACKEND_SERVER
          value: "apitable/backend-server:latest"
        - name: IMAGE_DATABUS_SERVER
          value: "apitable/databus-server:latest"
        - name: IMAGE_GATEWAY
          value: "apitable/openresty:latest"
        - name: IMAGE_IMAGEPROXY_SERVER
          value: "apitable/imageproxy-server:latest"
        - name: IMAGE_INIT_APPDATA
          value: "apitable/init-appdata:latest"
        - name: IMAGE_INIT_DB
          value: "apitable/init-db:latest"
        - name: IMAGE_MINIO
          value: "minio/minio:RELEASE.2023-01-25T00-19-54Z"
        - name: IMAGE_MYSQL
          value: "mysql:8.0.32"
        - name: IMAGE_PULL_POLICY
          value: "always"
        - name: IMAGE_RABBITMQ
          value: "rabbitmq:3.11.9-management"
        - name: IMAGE_REDIS
          value: "redis:7.0.8"
        - name: IMAGE_REGISTRY
          value: "docker.io"
        - name: IMAGE_ROOM_SERVER
          value: "apitable/room-server:latest"
        - name: IMAGE_WEB_SERVER
          value: "apitable/web-server:latest"
        - name: INSTANCE_MAX_MEMORY
          value: "4096M"
        - name: MAIL_ENABLED
          value: "false"
        - name: MAIL_FROM
          value: ""
        - name: MAIL_HOST
          value: ""
        - name: MAIL_PASSWORD
          value: ""
        - name: MAIL_PORT
          value: ""
        - name: MAIL_SSL_ENABLE
          value: "true"
        - name: MAIL_TYPE
          value: "smtp"
        - name: MAIL_USERNAME
          value: ""
        - name: MINIO_ACCESS_KEY
          value: "apitable"
        - name: MINIO_SECRET_KEY
          value: "apitable@com"
        - name: MYSQL_DATABASE
          value: "apitable"
        - name: MYSQL_HOST
          value: "mysql"
        - name: MYSQL_PASSWORD
          value: "apitable@com"
        - name: MYSQL_PORT
          value: "3306"
        - name: MYSQL_ROOT_PASSWORD
          value: "apitable@com"
        - name: MYSQL_USERNAME
          value: "root"
        - name: NEST_GRPC_ADDRESS
          value: "static://room-server:3334"
        - name: NGINX_HTTP_PORT
          value: "80"
        - name: NGINX_HTTPS_PORT
          value: "443"
        - name: NODE_ENV
          value: "apitable"
        - name: NODE_OPTIONS
          value: "--max-old-space-size=2048 --max-http-header-size=80000"
        - name: OSS_BUCKET_NAME
          value: "assets"
        - name: OSS_CACHE_TYPE
          value: "minio"
        - name: OSS_CLIENT_TYPE
          value: "aws"
        - name: OSS_ENABLED
          value: "true"
        - name: OSS_HOST
          value: "assets"
        - name: OSS_TYPE
          value: "QNY1"
        - name: PUBLIC_URL
          value: ""
        - name: RABBITMQ_HOST
          value: "rabbitmq"
        - name: RABBITMQ_PASSWORD
          value: "apitable@com"
        - name: RABBITMQ_PORT
          value: "5672"
        - name: RABBITMQ_USERNAME
          value: "apitable"
        - name: RABBITMQ_VHOST
          value: "/"
        - name: REDIS_DB
          value: "0"
        - name: REDIS_HOST
          value: "redis"
        - name: REDIS_PASSWORD
          value: "apitable@com"
        - name: REDIS_PORT
          value: "6379"
        - name: ROOM_GRPC_URL
          value: "room-server:3334"
        - name: SERVER_DOMAIN
          value: ""
        - name: SKIP_REGISTER_VALIDATE
          value: "true"
        - name: SMS_ENABLED
          value: "false"
        - name: SOCKET_DOMAIN
          value: "http://room-server:3333/socket"
        - name: SOCKET_GRPC_URL
          value: "room-server:3007"
        - name: SOCKET_RECONNECTION_ATTEMPTS
          value: "10"
        - name: SOCKET_RECONNECTION_DELAY
          value: "500"
        - name: SOCKET_TIMEOUT
          value: "5000"
        - name: SOCKET_URL
          value: "http://room-server:3002"
        - name: TEMPLATE_PATH
          value: "./static/web_build/index.html"
        - name: TEMPLATE_SPACE
          value: "spcNTxlv8Drra"
        - name: TIMEZONE
          value: "Asia/Singapore"
        - name: TZ
          value: "Asia/Singapore"
        - name: USE_CUSTOM_PUBLIC_FILES
          value: "true"
        - name: USE_NATIVE_MODULE
          value: "0"

```



有了基本的架子之后，就可以在当前的模板上进行更新修改了，按需调整配置项，最终实现了make dataenv和make run涉及的所有组件

在k8s deployment上部署的目标。
