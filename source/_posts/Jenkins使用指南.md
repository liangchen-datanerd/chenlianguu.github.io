---
title: Jenkins使用指南
top: false
toc: true
mathjax: true
date: 2020-01-09 13:07:22
index_img: https://i.loli.net/2020/01/09/aQSYzXWHexZGBLu.png
summary:
tags:
 - docker
 - jenkins
 - CI/CD
categories: CI/CD
---

## 简介 

Jenkins是一个广泛用于持续构建的可视化web工具，持续构建说得更直白点，就是各种项目的"自动化"编译、打包、分发部署。jenkins可以很好的支持各种语言（比如：java, c#, php等）的项目构建，也完全兼容ant、maven、gradle等多种第三方构建工具，同时跟svn、git能无缝集成，也支持直接与知名源代码托管网站，比如github、bitbucket直接集成。
简单点说，Jenkins其实就是大的框架集，可以整个任何你想整合的内容，实现公司的整个持续集成体系！

## 安装

基于docker安装：

```shell
docker run -it \
  --name jenkins \
  --restart always \
  --user root \
  -p 10002:8080 \
  -p 50000:50000 \
  -v /data/jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /opt/jdk1.8.0_25:/opt/jdk1.8.0_25 \
  -v /bin/docker:/bin/docker \
  -v /data/repository:/data/repository \
  -v $(which docker):/usr/bin/docker \
  jenkins/jenkins:lts
```

注意：需要使用jenkins/jenkins:lts镜像，jenkins:latest镜像官方已不提供支持，版本过低

其中将外部docker映射到了内部docker，这样在jenkins容器内部也可以使用docker命令了

注意启动之后会有个随机的密码：
例：
ff9c1128af2840d990798418bd3c92f2

如果采用以-it的形式启动，可以在命令窗口中看到。

[![password.png](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/scaled-840-0/password.png)](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/password.png)

当然也可以进入容器，在/var/jenkins_home/secrets/initialAdminPassword中找到。

注意！映射在容器中的/var/jenkins_home 目录到具有名字 jenkins-data 的volume。 如果这个卷不存在，那么这个 docker run 命令会自动为你创建卷。 如果您希望每次重新启动Jenkins（通过此 docker run … 命令）时保持Jenkins状态，则此选项是必需的，jenkins数据会在该卷进行持久化 。 否则，那么在每次重新启动后，Jenkins将有效地重置为新的实例。

（可选 /var/run/docker.sock 表示Docker守护程序通过其监听的基于Unix的套接字。 该映射允许 jenkinsci/blueocean 容器与Docker守护进程通信， 如果 jenkinsci/blueocean 容器需要实例化其他Docker容器，则该守护进程是必需的。 如果运行声明式管道，其语法包含agent部分用 docker.

进入 ip:10002 jenkins安装界面

[![install.png](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/scaled-840-0/install.png)](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/install.png)

安装对应插件

## Demo 

```
本Demo实现的场景是push到项目master分支，自动触发打包发布到docker镜像仓库harbor，然后在拉取镜像在docker中运行
```

项目Demo地址：http://161.189.27.8:8090/chenliang/jenkinsdemo

jenkins项目地址：http://52.83.79.244:10002/job/jenkins-demo

大家可以拉取项目Demo代码，修改代码push提交master，自动触发打包执行，在jenkins的console output查看执行信息

### 初始化

- step1 gitlab新建项目 比如jenkins-demo

- step2 

  jenkins添加gitlab账户及密码

  凭据--->系统--->全局凭据--->add credentials

  ![1572856908321](./assets\pass.png)

- step3  jenkins新建项目，选择maven项目---source code managment

  填写项目的地址，选择step生成的credentials，选择代码分支

[![maven.png](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/scaled-840-0/maven.png)](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/maven.png)

[![demo02.png](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/scaled-840-0/demo02.png)](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/demo02.png)

- step4  选择Build Trigger--->勾选gitlab触发选项--->点击generate生成scretkey并记住--->复制gitlab webhook的url

  url及secretkey在gitlab设置中需要用

[![setting.png](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/scaled-840-0/setting.png)](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/setting.png)

- step5 回到gitlab项目的setting的integration页面中，填写step4中的url及secretkey，取消勾选ssl

[![step5.png](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/scaled-840-0/step5.png)](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/step5.png)

- step6 build中填写构建maven命令及构建后需要执行的程序，本demo执行的是打包然后java命令执行jar包

[![build.png](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/scaled-840-0/build.png)](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/build.png)

  

### 测试

以gitlab项目jenkinsdemo为例：http://161.189.27.8:8090/chenliang/jenkinsdemo 当修改代码push到master中的时候会自动出发打包及执行jar包，在jenkins项目的console output中查看打包及执行日志

[![output.png](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/scaled-840-0/output.png)](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/output.png)

## trouble shooting

### 安装插件报错

[![ceverror01.png](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/scaled-840-0/ceverror01.png)](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/ceverror01.png)

更改jenkins源--->进入系统管理--->管理插件--->高级 

将

```
http://updates.jenkins-ci.org/update-center.json
```

更换为

```
http://mirror.esuni.jp/jenkins/updates/update-center.json
```

保存即可

[![demo01.png](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/scaled-840-0/demo01.png)](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/demo01.png)

### jenkins内部执行docker命令报错

docker内部执行docker报的错误信息：docker: error while loading shared libraries: libltdl.so.7: cannot open shared object file: No such file or directory

第一次使用的docker部署jenkins的时候，出现了两个问题：

1、因为用户权限问题挂载/home/jenkins/data到/var/jenkins_home挂载不了。后面通过修改data目录的所属用户可以解决，即在容器下查询用户id（1000）,然后把data改成同样的用户id

2、即便挂载docker命名和docker.sock,也修改了相应的权限，仍存在libltdl7没有权限读取。当然好像也不影响使用，只是在容器里面执行docker info的时候，会报无法读取libltdl.so.7的信息。

docker: error while loading shared libraries: /usr/lib/x86_64-linux-gnu/libltdl.so.7: cannot read file data: Error 21

 

于是查找资料在jenkins/jenkins基础上再建一个Jenkins镜像。

编辑Dockerfile

```shell
FROM jenkins/jenkins:lts

USER root
#清除了基础镜像设置的源，切换成阿里云的jessie源
RUN echo '' > /etc/apt/sources.list.d/jessie-backports.list \
  && echo "deb http://mirrors.aliyun.com/debian jessie main contrib non-free" > /etc/apt/sources.list \
  && echo "deb http://mirrors.aliyun.com/debian jessie-updates main contrib non-free" >> /etc/apt/sources.list \
  && echo "deb http://mirrors.aliyun.com/debian-security jessie/updates main contrib non-free" >> /etc/apt/sources.list
#更新源并安装缺少的包

RUN apt-get update && apt-get install -y libltdl7

ARG dockerGid=999

RUN echo "docker:x:${dockerGid}:jenkins" >> /etc/group USER jenkins

# 安装 docker-compose  --- 挂载宿主机上的就可以了
# RUN curl -L https://github.com/docker/compose/releases/download/1.17.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
# RUN chmod +x /usr/local/bin/docker-compose
```

build镜像

```shell
docker build . -t myjenkins:v1
```
