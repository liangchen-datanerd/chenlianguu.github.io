---
title: Docker大数据环境搭建指南
top: false
toc: true
mathjax: true
date: 2020-03-03 13:14:49
index_img: https://miro.medium.com/max/1400/0*d1YEe9DrpwFomYys.png
summary:
tags:
 - docker
 - big data
 - 证书
 - 部署
categories: 运维
---

## 思路
将大数据平台资源docker化，主要基于以下考虑

- 开箱即用，不常驻后台，需要的时候启动集群即可，不用的时候关闭集群释放机器资源
- 避免了直接安装端口占用问题，大数据平台所需端口较多
- spark资源docker化，便于后期k8s进行资源管理调度

上述好处主要是基于测试环境下，基于生产环境的大数据平台化要考虑的点很多，例如后期扩容、运维、安全等因素，大数据平台整个是否docker化后期有待考证。

## 镜像制作方案

使用Docker来搭建hadoop,spark及mysql的集群，首先使用Dockerfile制作镜像，把相关的软件拷贝到约定好的目录 下，把配置文件在外面先配置好，再拷贝移动到hadoop,spark的配置目录，为了能使得mysql能从其它节点被访问到，要配置mysql的访问权限。

## 整体架构

一共3个节点，即启动3个容器。hadoop-master,hadoop-node1,hadoop-node2这三个容器里面安装hadoop和spark集群。
![architect](https://i.loli.net/2020/03/03/Ap2vwPqN9ijY84K.png)

## 集群部署

### 集群网络规划及子网配置

既然是做集群，网络的规划是少不了的,至于网络，可以通过Docker中的DockerNetworking的支持配置。首先设置网络，docker中设置 子网可以通过docker network create 方法，这里我们通过命令设置如下的子网。--subnet指定子网络的网段，并为这个子网命名一个名字叫spark

```shell
# 创建子网
docker network create --subnet=172.16.0.0/16 spark
# 查看网络
docker network ls
NETWORK ID          NAME                       DRIVER              SCOPE
fab2dd51d1cf        spark                      bridge              local
```

 接下来就在我们创建的子网落spark中规划集群中每个容器的ip地址。网络ip分配如下:

hadoop-master 172.16.0.2

hadoop-node1 172.16.0.3

hadoop-node2 172.16.0.4

### 软件版本

网络规划好了，首先Spark我们使用最新的2.4.4版本，Hadoop采用比较稳定的hadoop-2.7.3版本，scala采用scala-2.11.8，JDK采用jdk-8u101-linux-x64。

### SSH无密钥登录规则配置

注意这里不使用ssh-keygen -t rsa -P ''这种方式生成id_rsa.pub，然后集群节点互拷贝id_rsa.pub到authorized_keys文件这种方式，而 是通过在.ssh目录下配置ssh_conf文件的方式，ssh_conf中可以配置SSH的通信规则，例如以正则表达式的方式指定hostname为XXX的 机器之间实现互联互通，而不进行额外的密钥验证。为了编写这个正则表达式，我们5个节点的hostname都以hadoop-*的方式作为开 头，这就是采用这种命名规则的原因。下面来看下ssh_conf配置的内容:

```shell
Host localhost
	StrictHostKeyChecking no
Host 0.0.0.0 
	StrictHostKeyChecking no
Host hadoop-* 
	StrictHostKeyChecking no
```

注意上面的最后一行，Host hadoop-* 指定了它的严格的Host验证StrictHostKeyChecking 为no，这样既可以是这5个hostname以 hadoop-*开头的容器之间实现互联互通，而不需要二外的验证。

### 构建镜像

Dockerfile编写完成，接下来写一个build.sh脚本，内容如下:

```shell
 echo build hadoop images
 docker build -t="spark" . 
```

表示构建一个名叫spark的镜像，.表示Dockerfile的路径，因为在当前路径下，所有用.,若在其他地方则用绝对路径指定Dockerfile的路径 即可。

运行sh build.sh，就会开始制作镜像了。

## 集群运行

### 启动容器 start_container.sh

使用这个镜像可完成容器的启动，因为使用了基于DockerNetworking的网络机制，因此可以在启动容器的时候为容器在子网172.16.0.0/16 spark中分贝172.16.0.1 172.16.0.255以外的IP地址，容器内部容器的通信是基于hostname，因此 需要指定hostname，为了方便容器的管理，需要为启动的每个容器指定一个名字。为了方便外网访问，需要通过-p命令指定容器到宿主机的端口映射。还要为每个容器增加host列表。

```shell
# hadoop-master
docker run -itd --restart=always \
	--net spark \
	--ip 172.16.0.2 \
	--privileged \
	-p 18032:8032 \
	-p 28080:18080 \
	-p 29888:19888 \
	-p 17077:7077 \
	-p 51070:50070 \
	-p 18888:8888 \
	-p 19000:9000 \
	-p 11100:11000 \
	-p 51030:50030 \
	-p 18050:8050 \
	-p 18081:8081 \
	-p 18900:8900 \
	--name hadoop-master \
	--hostname hadoop-master \
	--add-host hadoop-node1:172.16.0.3 \
	--add-host hadoop-node2:172.16.0.4 \
	--add-host hadoop-mysql:172.16.0.6 \
	spark /usr/sbin/init
	
# hadoop-node1
docker run -itd --restart=always \
	--net spark \
	--ip 172.16.0.3 \
	--privileged \
	-p 18042:8042 \
	-p 51010:50010 \
	-p 51020:50020 \
	--name hadoop-node1 \
	--hostname hadoop-node1 \
	--add-host hadoop-master:172.16.0.2 \
	--add-host hadoop-node2:172.16.0.4 \
	spark /usr/sbin/init

# hadoop-node2
docker run -itd --restart=always \
	--net spark \
	--ip 172.16.0.4 \
	--privileged \
	-p 18043:8042 \
	-p 51011:50011 \
	-p 51021:50021 \
	--name hadoop-node2 \
	--hostname hadoop-node2 \
	--add-host hadoop-master:172.16.0.2 \
	--add-host hadoop-node1:172.16.0.3 \
	spark /usr/sbin/init

```



### 关闭集群 stop_container.sh

```shell
echo stop containers
docker stop hadoop-master
docker stop hadoop-node1
docker stop hadoop-node2
echo remove containers
docker rm hadoop-master
docker rm hadoop-node1
docker rm hadoop-node2

echo rm containers

docker ps
```

### 重启集群 restart_container.sh

```shell
echo stop containers
docker stop hadoop-master
docker stop hadoop-node1
docker stop hadoop-node2
echo restart containers
docker start hadoop-master
docker start hadoop-node1
docker start hadoop-node2
echo start sshd
docker exec -it hadoop-master systemctl start sshd
docker exec -it hadoop-node1 systemctl start sshd
docker exec -it hadoop-node2 systemctl start sshd
docker exec -it hadoop-master ~/restart-hadoop.sh
echo  containers started

docker ps
```

## Trouble Shooting

- docker里面执行systemctl报错

  解决方案：启动的时候用/usr/sbin/init
  
- docker登录harbor报错   
  window直接修改docker设置，添加ip:8093到docker insecure registry   
  linux修改方法：https://blog.csdn.net/u010397369/article/details/42422243  

## Demo

进入hadoop-master容器内部，执行spark-shell

```shell
root@node-2 docker-spark]# docker exec -it hadoop-master /bin/bash
[root@hadoop-master ~]# spark
spark-class   spark-shell   spark-sql     spark-submit  sparkR
[root@hadoop-master ~]# spark-shell
19/12/03 04:51:25 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
19/12/03 04:51:43 WARN util.Utils: spark.executor.instances less than spark.dynamicAllocation.minExecutors is invalid, ignoring its setting, please update your configs.
Spark context Web UI available at http://hadoop-master:4040
Spark context available as 'sc' (master = spark://hadoop-master:7077, app id = app-20191203045141-0000).
Spark session available as 'spark'.
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.4.4
      /_/

Using Scala version 2.11.12 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_101)
Type in expressions to have them evaluated.
Type :help for more information.

scala>
```



## 本地部署

- 想要在自己的笔记本环境使用

  首先笔记本环境下需要有docker环境

  - 拉取gitlab上面的项目：git clone git@161.189.27.8:chenliang/docker-spark.git
  - 拉去harbor上面的镜像：docker pull 52.83.79.244:8093/wuhan/spark:v1（也可以自己构建镜像，Dockerfile文件在gitlab项目里边）  
  	前提：机器docker环境登录harbor，账号密码：admin 1qaz!QAZ   
    docker login 52.83.79.244:8093  
    登录报错：Error response from daemon: Get https://52.83.79.244:8093/v2/: http: server gave HTTP response to HTTPS client
    解决：参见trouble shooting
  - 使用相关脚本执行启动、停止、重启集群

  

- 如何自己的spark程序如何在docker环境下执行  

  启动spark的集群之后，使用docker cp等命令将打好的jar包打进容器内，使用spark脚本执行