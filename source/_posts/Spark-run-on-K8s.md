---
title: Spark run on K8s
top: false
toc: true
mathjax: true
date: 2020-01-09 12:40:24
index_img: https://i.loli.net/2020/01/09/yfu8QZc7i3sRdrU.png
summary: 
tags:
 - spark
 - kubernetes
 - 部署
 - 大数据
categories: kubernetes
---

## 部署
首先查看了下spark的官方文档，了解了spark怎么在k8s上面跑的，实际上不需要搭建spark集群，提交作业到k8s的api server即可。看似简单但是还是不知道怎么动手实践。于是youtube上面搜了下相关视频，按照视频很快就实践了一把spark run on k8s，具体步骤如下：

- step 1 到k8-master机器下载二进制spark最新二进制安装包，并解压

```shell
cd /opt
wget http://mirrors.tuna.tsinghua.edu.cn/apache/spark/spark-2.4.4/spark-2.4.4-bin-hadoop2.7.tgz
tar -zxvf http://mirrors.tuna.tsinghua.edu.cn/apache/spark/spark-2.4.4/spark-2.4.4-bin-hadoop2.7.tgz
```

- step 2 制作spark的docker镜像

```shell
cd spark-2.4.4-bin-hadoop2.7
./bin/docker-image-tool.sh -r chenlianguu -t v2.4.4 build  # 制作spark进行
./bin/docker-image-tool.sh -r chenlianguu -t v2.4.4 push  #将spark镜像推送到docker hub
docker images
[root@k8s-master opt]# docker images
REPOSITORY                                                        TAG                 IMAGE ID            CREATED             SIZE
chenliang/spark-r                                                 v2.4.4              6479a523e3f7        20 hours ago        759MB

```

- step 3 提交作业到k8s

在提交spark的作业的机器上，把api server的proxy打开

```shell
kubectl proxy
```

```shell
./bin/spark-submit \
--master k8s://http://127.0.0.1:8001 \
--name spark-pi \
--deploy-mode cluster \
--class org.apache.spark.examples.SparkPi \
--conf spark.executor.instances=3 \
--conf spark.kubernetes.container.image=chenlianguu/spark-r:v2.4.4 \
/opt/spark-2.4.4-bin-hadoop2.7/examples/jars/spark-examples_2.11-2.4.4.jar
```

## 运行说明
[![k8s-cluster-mode.png](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/scaled-840-0/k8s-cluster-mode.png)](http://52.83.79.244:6875/uploads/images/gallery/2019-12-Dec/k8s-cluster-mode.png)
集群模式下，通过spark-submit提交程序到k8s集群，具体一下步骤：  

- 通过k8s创建spark driver端的pod
- driver端在k8s其他节点创建executor端的pod并保持通信，executor具体执行代码，这里设计到权限问题，假如没有对应的权限创建pods，执行spark会报错，具体见trouble  shooting第二点
- 当程序跑完了，executor端pod会终止并清理，driver端的pod会保持complete状态并持久化log信息，最后会由k8s api server进行driver端pod的垃圾回收工作

## trouble shooting 
跑spark on k8s的pi example碰到的一些坑及解决方案
*  找不到jar问题
![1](https://i.loli.net/2020/01/09/xzqUmBIuCdP6HlW.jpg)
![2](https://i.loli.net/2020/01/09/U6uIFOrgcjQAkn4.jpg)
之前提交的脚本是这样的  

```shell
./bin/spark-submit \
--master k8s://http://127.0.0.1:8001 \
--name spark-pi \
--deploy-mode cluster \
--class org.apache.spark.examples.SparkPi \
--conf spark.executor.instances=3 \
--conf spark.kubernetes.container.image=chenlianguu/spark-r:v2.4.4 \
/opt/spark-2.4.4-bin-hadoop2.7/examples/jars/spark-examples_2.11-2.4.4.jar
```

其中jar包制定的地址是local，路径填写应该加上协议local:///opt/spark-2.4.4-bin-hadoop2.7/examples/jars/spark-examples_2.11-2.4.4.jar，加上之后还是报同样的错，google后发现jar包实际上是在jar包在docker镜像里面的地址，因此改为local:///opt/spark/examples/jars/spark-examples_2.11-2.4.4.jar   

*  jar问题解决了，又开始报错，报错信息如下  
![3](https://i.loli.net/2020/01/09/BgsviowCp5EzARG.jpg)
第一感觉就是权限问题，网上找到了解决方案，ref：[解决办法](https://github.com/GoogleCloudPlatform/continuous-deployment-on-kubernetes/issues/113)，按照这个思路，在k8s-master节点，输入   
```
kubectl create clusterrolebinding default --clusterrole cluster-admin --serviceaccount=default:default
```   


然后继续执行spark-submit脚本，可以顺利启动driver端，但是有2个executor端还是报错，说是资源不够，有一个executor端执行成功，整个job还是顺利的跑下来了，查看driver端日志。kubectl logs podsname 

![5](https://i.loli.net/2020/01/09/5bWGKh34DqVQfYU.jpg)
------



参考资料：     
[spark官方doc](https://spark.apache.org/docs/latest/running-on-kubernetes.html)   
[youtube动手视频](https://www.youtube.com/watch?v=l7UoE97Z24I&list=LL-3gJZTnF4DbSyb7KID7P7g&index=2&t=0s)