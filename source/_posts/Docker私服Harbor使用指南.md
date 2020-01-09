---
title: Docker私服Harbor使用指南
top: false
toc: true
mathjax: true
date: 2020-01-09 11:05:59
index_img: https://i.loli.net/2020/01/09/LKEbkBZWwV7rRT1.jpg
summary:
tags:
 - harbor
 - docker
 - https
 - 证书
 - 部署
categories: 运维
---

## harbor搭建

官方提供两种方式安装harbor，离线及在线方式。本文档描述离线方式进行安装

查看docker-compose版本，若版本低进行升级安装，

[安装文档]: http://161.189.27.8:8090/whdev/whdata/wikis/Docker%E7%9B%B8%E5%85%B3%E6%9C%8D%E5%8A%A1%E7%9A%84%E6%90%AD%E5%BB%BA

下载官方安装包

```shell
cd /opt
wget https://storage.googleapis.com/harbor-releases/release-1.9.0/harbor-offline-installer-v1.9.2-rc1.tgz
# 解压
tar zxvf harbor-offline-installer-v1.9.2-rc1.tgz
# 查看
cd harbor
ls
[root@ip-172-31-23-16 harbor]# ll
total 623296
drwxr-xr-x 3 root root        20 Nov  5 05:50 common
-rw-r--r-- 1 root root      5369 Nov  5 06:07 docker-compose.yml
-rw-r--r-- 1 root root 638214056 Nov  1 03:14 harbor.v1.9.2.tar.gz
-rw-r--r-- 1 root root      5816 Nov  5 06:07 harbor.yml
-rwxr-xr-x 1 root root      5088 Nov  1 03:13 install.sh
-rw-r--r-- 1 root root     11347 Nov  1 03:13 LICENSE
-rwxr-xr-x 1 root root      1748 Nov  1 03:13 prepare
```

修改配置，主要修改两个文件

harbor.yml为系统配置文件，docker-compose.yml为docker相关配置文件

修改harbor.yml

```shell
vim harbor.yml
# 修改hostname
hostname: 52.83.79.244
# 修改映射端口
http:
  port: 8093
# 修改admin password（可选）
harbor_admin_password: 1qaz!QAZ
# 修改数据库相关参数
database:
  password: root
  max_idle_conns: 50
  max_open_conns: 100
# 修改数据持久化host目录
data_volume: /data/harbor
# 修改log目录
 location: /var/log/harbor
```

修改docker-compose.yml文件

```yaml
vim docker-compose.yml
# 添加ports映射
registry:
    image: goharbor/registry-photon:v2.7.1-patch-2819-2553-v1.9.2
    container_name: registry
    restart: always
    ports:
      - 5000:5000
```

启动harbor

```
./install.sh
```

安装之后查看

```shell
docker-compose ps
     Name                     Command                  State                 Ports
---------------------------------------------------------------------------------------------
harbor-core         /harbor/harbor_core              Up (healthy)
harbor-db           /docker-entrypoint.sh            Up (healthy)   5432/tcp
harbor-jobservice   /harbor/harbor_jobservice  ...   Up (healthy)
harbor-log          /bin/sh -c /usr/local/bin/ ...   Up (healthy)   127.0.0.1:1514->10514/tcp
harbor-portal       nginx -g daemon off;             Up (healthy)   8080/tcp
nginx               nginx -g daemon off;             Up (healthy)   0.0.0.0:8093->8080/tcp
redis               redis-server /etc/redis.conf     Up (healthy)   6379/tcp
registry            /entrypoint.sh /etc/regist ...   Restarting
registryctl         /harbor/start.sh                 Up (healthy)
```

登录harbor ，账号密码为harbor.yml所设置的密码

http://52.83.79.244:8093/harbor/projects

登录时候还不能直接利用docker push 到服务器，这是因为 docker1.3.2 版本开始默认 docker registry 使用的是 https，我们设置 Harbor 默认 http 方式，所以当执行用 docker login、pull、push 等命令操作非 https 的 docker regsitry 的时就会报错。解决办法：

```shell
vim /usr/lib/systemd/system/docker.service
# ExecStart 增加 --insecure-registry=52.83.79.244【配置文件中的hostname】
ExecStart=/usr/bin/dockerd $OPTIONS $DOCKER_STORAGE_OPTIONS $DOCKER_ADD_RUNTIMES --insecure-registry=52.83.79.244:8093
# 重启服务
systemctl daemon-reload
systemctl restart docker
```

harbor常用命令

```shell
# 需要进入harbor文件夹执行
cd /opt/harbor

docker-compose up -d               ###后台启动，如果容器不存在根据镜像自动创建
docker-compose down   -v           ###停止容器并删除容器
docker-compose start               ###启动容器，容器不存在就无法启动，不会自动创建镜像
docker-compose stop                ###停止容器
```

## 使用及配置

- docker 登录到harbor

  ```shell
  [root@ip-172-31-23-16 harbor]# docker login 52.83.79.244:8093
  Username: admin
  Password:
  WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
  Configure a credential helper to remove this warning. See
  https://docs.docker.com/engine/reference/commandline/login/#credentials-store
  
  Login Succeeded
  ```

登录harbor后创建一个公共仓库，命名为wuhan

将镜像发布到harbor

```shell
# 查看需要上传的镜像
docker images
# 为镜像上tag
docker tag jenkins/jenkins:lts 52.83.79.244:8093/wuhan/jenkins:lts
# 上传到wuhan库
docker push jenkins/jenkins:lts 52.83.79.244:8093/wuhan/jenkins:lts
# 若需要上传到library库
docker push jenkins/jenkins:lts 52.83.79.244:8093/library/jenkins:lts
```
## 配置https协议
```shell
# 准备工作
mkdir -p /data/harbor-cert
yum install -y openssl

------------
cd data/harbor-cert
openssl genrsa -des3 -out server.key 2048 
openssl req -new -key server.key -out server.csr 
cp server.key server.key.org openssl rsa -in server.key.org -out server.key 
openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt 
```
生成证书之后，修改harbor.yaml文件
具体配置如下
[![WX20191219-155232@2x.png](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/scaled-840-0/WX20191219-155232@2x.png)](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/WX20191219-155232@2x.png)


## 使用及配置

- docker 登录到harbor

  ```shell
  [root@ip-172-31-23-16 harbor]# docker login 52.83.79.244:8093
  Username: admin
  Password:
  WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
  Configure a credential helper to remove this warning. See
  https://docs.docker.com/engine/reference/commandline/login/#credentials-store
  
  Login Succeeded
  ```

登录harbor后创建一个公共仓库，命名为wuhan

将镜像发布到harbor

```shell
# 查看需要上传的镜像
docker images
# 为镜像上tag
docker tag jenkins/jenkins:lts 52.83.79.244:8093/wuhan/jenkins:lts
# 上传到wuhan库
docker push jenkins/jenkins:lts 52.83.79.244:8093/wuhan/jenkins:lts
# 若需要上传到library库
docker push 52.83.79.244:8093/library/jenkins:lts
```

## trouble shooting
- docker登录harbor报错   
  window直接修改docker设置，添加52.83.79.244:8093到docker insecure registry   
  linux修改方法：https://blog.csdn.net/u010397369/article/details/42422243