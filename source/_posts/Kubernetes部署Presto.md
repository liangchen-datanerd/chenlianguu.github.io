---
title: Kubernetes部署Presto
top: false
toc: true
mathjax: true
date: 2020-03-28 17:47:45
index_img: https://i.loli.net/2020/03/28/InC7YBLkgTVEAJ4.png
summary:
tags:
 - kubernetes
 - presto
 - rook
 - ceph
 - 部署
categories: kubernetes
---
## 介绍

> Presto

Presto是一个分布式SQL查询引擎，用于查询分布在一个或多个不同数据源中的大数据集。完整安装包括一个Coordinator和多个Worker。 由客户端提交查询，从Presto命令行CLI提交到Coordinator。 Coordinator进行解析，分析并执行查询计划，然后分发处理队列到Worker。

Presto是完全基于内存的分布式大数据查询引擎，所有查询和计算都在内存中执行。

Presto的输入是SQL语句；输出是具体的SQL执行结果。

Presto可以对接不同的数据源，例如MySQL、Hive等。

Presto可以对SQL的查询过程进行优化，包括SQL本身的执行计划优化，以及用分布式查询提高并发等。

Presto不是数据库，并不能处理在线事务。

> ceph 

Ceph是当前非常流行的开源分布式存储系统，具有高扩展性、高性能、高可靠性等优点，同时提供块存储服务(rbd)、对象存储服务(rgw)以及文件系统存储服务(cephfs)，Ceph在存储的时候充分利用存储节点的计算能力，在存储每一个数据时都会通过计算得出该数据的位置，尽量的分布均衡。

> rook

Rook是一个开放源码的云本机存储协调器，提供平台、框架和对各种存储解决方案的支持，以便与云本机环境进行本机集成。

Rook将存储软件转变为自我管理、自我扩展和自我修复的存储服务。它通过自动化部署、引导、配置、供应、扩展、升级、迁移、灾难恢复、监视和资源管理来实现这一点。Rook使用底层云本地容器管理、调度和协调平台提供的设施来执行其职责。

Rook利用扩展点深入集成到云本机环境中，为调度、生命周期管理、资源管理、安全、监控和用户体验提供无缝体验。

> 基于kubernetes部署presto

将presto数据持久化到ceph集群中，保证presto的数据高可用

## 软件版本

- Kubernetes v1.15
- Rook v1.0.2
- CentOS 7 

## 部署rook

rook的部署可以使用kubernetes的包管理工具[helm](https://helm.sh/)安装，由于rook及kubernetes版本众多以及涉及到相关源等问题，本文没有采用helm安装方式部署rook

```shell
$ git clone https://github.com/rook/rook.git 
$ cd rook 
$ git checkout v1.0.2
$ cd cluster/examples/kubernetes/ceph/
```

`cluster/examples/kubernetes/ceph/`目录包含部署rook相关文件

安装Rook Common Objects & Operator

```shell
kubectl create -f common.yaml
kubectl create -f operator.yaml
```

检查`rook-ceph-operator,rook-ceph-agent`, rook-discover`安装成功

```shell
$ kubectl get pods -n rook-ceph 
NAME                                   READY       STATUS        RESTARTS    AGE 
rook-ceph-agent-chj4l                  1/1         Running       0           84s 
rook-ceph-operator-548b56f995-v4mvp    1/1         Running       0           4m5s 
rook-discover-vkkvl                    1/1         Running       0    
```

## 部署Rook Ceph Cluster

```shell
kubectl apply -f cluster.yml
```

检查是否部署成功

```shell
kubectl get pods -n rook-ceph
NAME                                  READY   STATUS      RESTARTS   AGE
rook-ceph-agent-chl7b                 1/1     Running     0          4h14m
rook-ceph-agent-tbx4l                 1/1     Running     2          4h14m
rook-ceph-mgr-a-bf88cfdc4-wvhjb       1/1     Running     0          3h52m
rook-ceph-mon-a-df88f8cbf-dklvj       1/1     Running     0          3h53m
rook-ceph-operator-548b56f995-q47jb   1/1     Running     5          4h14m
rook-ceph-osd-0-655957b867-p8tf7      1/1     Running     0          3h51m
rook-ceph-osd-1-7578dc7874-f9q64      1/1     Running     0          3h50m
rook-ceph-osd-prepare-node-2-nvr7d    0/2     Completed   0          3h45m
rook-ceph-osd-prepare-node-3-2mm2p    0/2     Completed   0          3h45m
rook-discover-44789                   1/1     Running     3          4h14m
rook-discover-c4r7d                   1/1     Running     0          4h14m
```

## 测试ceph cluster

创建StorageClass以便ceph cluster动态创建PV、

```shell
kubectl create -f storageclass-test.yaml

# 查找Rook Operator
$ kubectl get pod -n rook-ceph --selector=app=rook-ceph-operator 
NAME                                  READY   STATUS    RESTARTS   AGE
rook-ceph-operator-548b56f995-q47jb   1/1     Running   5          4h23m

# 检查ceph cluster健康状态
kubectl exec -n rook-ceph -it rook-ceph-operator-548b56f995-q47jb -- ceph status
  cluster:
    id:     bed40efd-b3e8-4b18-b9a0-e8723f51e747
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum a (age 4h)
    mgr: a(active, since 3h)
    osd: 2 osds: 2 up (since 3h), 2 in (since 3h)

  data:
    pools:   1 pools, 100 pgs
    objects: 0 objects, 0 B
    usage:   13 GiB used, 22 GiB / 35 GiB avail
    pgs:     100 active+clean
    
# 查看ceph磁盘信息
kubectl exec -n rook-ceph -it rook-ceph-operator-548b56f995-q47jb -- ceph df
RAW STORAGE:
    CLASS     SIZE       AVAIL      USED       RAW USED     %RAW USED
    hdd       35 GiB     22 GiB     13 GiB       13 GiB         37.40
    TOTAL     35 GiB     22 GiB     13 GiB       13 GiB         37.40

POOLS:
    POOL            ID     STORED     OBJECTS     USED     %USED     MAX AVAIL
    replicapool      1        0 B           0      0 B         0        19 GiB
```

## 配置ceph dashborad

在cluster.yaml文件中默认已经启用了ceph dashboard，查看dashboard的service：

```shell
[root@node-1 presto-kubernetes]# kubectl get svc -n rook-ceph
NAME                                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
rook-ceph-mgr                            ClusterIP   10.1.73.175    <none>        9283/TCP            6h29m
rook-ceph-mgr-dashboard                  ClusterIP   10.1.228.105   <none>        8443/TCP            6h29m
rook-ceph-mgr-dashboard-external-https   NodePort    10.1.38.28     <none>        8443:30102/TCP      21m
rook-ceph-mon-a                          ClusterIP   10.1.130.153   <none>        6789/TCP,3300/TCP   6h31m
```

rook-ceph-mgr-dashboard监听的端口是8443，创建nodeport类型的service以便集群外部访问。

```shell
kubectl apply -f rook/cluster/examples/kubernetes/ceph/dashboard-external-https.yaml
```

查看一下nodeport暴露的端口，这里是30102端口：

```
[root@node-1 presto-kubernetes]# kubectl get svc -n rook-ceph
NAME                                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
rook-ceph-mgr                            ClusterIP   10.1.73.175    <none>        9283/TCP            6h29m
rook-ceph-mgr-dashboard                  ClusterIP   10.1.228.105   <none>        8443/TCP            6h29m
rook-ceph-mgr-dashboard-external-https   NodePort    10.1.38.28     <none>        8443:30102/TCP      21m
rook-ceph-mon-a                          ClusterIP   10.1.130.153   <none>        6789/TCP,3300/TCP   6h31m
```

获取Dashboard的登陆账号和密码

```shell
[root@node-1 ~]# MGR_POD=`kubectl get pod -n rook-ceph | grep mgr | awk '{print $1}'`
[root@node-1 ~]# kubectl -n rook-ceph logs $MGR_POD | grep password
debug 2020-02-25 23:54:04.280 7ffb08d89700  0 log_channel(audit) log [DBG] : from='client.4198 -' entity='client.admin' cmd=[{"username": "admin", "prefix": "dashboard set-login-credentials", "password": "Wo9nf64nDs", "target": ["mgr", ""], "format": "json"}]: dispatch
debug 2020-02-25 23:58:11.177 7f8606e0e700  0 log_channel(audit) log [DBG] : from='client.4373 -' entity='client.admin' cmd=[{"username": "admin", "prefix": "dashboard set-login-credentials", "password": "Wo9nf64nDs", "target": ["mgr", ""], "format": "json"}]: dispatch
```

找到username和password字段，我这里是admin，Wo9nf64nDs
打开浏览器输入任意一个Node的IP+nodeport端口，这里使用master节点 ip访问：https://localhost:30102/#/dashboard

## 部署presto

>  presto并不提供官方docker镜像，本文使用github开源仓库[dharmeshkakadia](https://github.com/dharmeshkakadia/presto-kubernetes)镜像

- clone仓库

  ```shell
  git clone https://github.com/dharmeshkakadia/presto-kubernetes/ && cd presto-kubernetes
  ```

- 启动Coordinator

  ```shell
  kubectl create -f coordinator-deployment.yaml 
  kubectl create -f presto-service.yaml
  ```

- 启动workers

  ```sh
  kubectl create -f worker-deployment.yaml
  ```

- 使用presto，找出外部地址

  ```shell
  [root@node-1 ~]# kubectl get svc presto
  NAME     TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
  presto   NodePort   10.1.27.143   <none>        8080:32151/TCP   27h
  ```

- 测试Presto客户端连接

  ```shell
  wget https://repo1.maven.org/maven2/com/facebook/presto/presto-cli/0.208/presto-cli-0.208-executable.jar
  
  chmod +x presto-cli-0.208-executable.jar 
  
  /presto-cli-0.208-executable.jar --server 192.168.112.240:30418 --catalog hive --schema default
  presto:default>
  ```

  

## 挂载ceph卷到presto

> 由于github开源版本，没有提供数据卷挂载，因此需要相关yaml文件进行修改，配置rook提供RBD服务

rook可以提供以下3类型的存储：
 Block: Create block storage to be consumed by a pod
 Object: Create an object store that is accessible inside or outside the Kubernetes cluster
 Shared File System: Create a file system to be shared across multiple pods
在提供（Provisioning）块存储之前，需要先创建StorageClass和存储池。K8S需要这两类资源，才能和Rook交互，进而分配持久卷（PV）。
在kubernetes集群里，要提供rbd块设备服务，需要有如下步骤：

1. 创建rbd-provisioner pod
2. 创建rbd对应的storageclass
3. 创建pvc，使用rbd对应的storageclass
4. 创建pod使用rbd pvc

通过rook创建Ceph Cluster之后，rook自身提供了rbd-provisioner服务，所以我们不需要再部署其provisioner。

```shell
# 配置storageclass
kubectl apply -f ~/cluster/examples/kubernetes/ceph/storageclass.yaml

# 检查storageclass
kubectl get storageclass
NAME              PROVISIONER          AGE
rook-ceph-block   ceph.rook.io/block   171m
```

> 修改preso-kubernetes文件夹中的worker-deployment.yaml

```shell
vim worker-deployment.yaml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: presto-data-claim
spec:
  storageClassName: rook-ceph-block
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---

apiVersion: apps/v1beta1
kind: Deployment
metadata:
  labels:
    presto: worker
  name: worker
spec:
  replicas: 1
  template:
    metadata:
      labels:
        presto: worker
    spec:
      containers:
      - env:
        - name: HTTP_SERVER_PORT
          value: "8080"
        - name: PRESTO_JVM_HEAP_SIZE
          value: "8"
        - name: PRESTO_MAX_MEMORY
          value: "10"
        - name: PRESTO_MAX_MEMORY_PER_NODE
          value: "1"
        - name : COORDINATOR
          value: "presto"
        image: johandry/presto
        livenessProbe:
          exec:
            command:
            - /etc/init.d/presto status | grep -q 'Running as'
          failureThreshold: 3
          periodSeconds: 300
          timeoutSeconds: 10
        name: worker
        ports:
        - containerPort: 8080
        volumeMounts:
            - name: presto-data-volume
              mountPath: /var/presto/data
      restartPolicy: Always
      volumes:
        - name: presto-data-volume
          persistentVolumeClaim:
            claimName: presto-data-claim
```

重新apply yaml文件

```shell
kubectl aplly -f worker-deployment.yaml
```

检查是否配置成功

```shell
[root@node-1 ~]# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                       STORAGECLASS      REASON   AGE
pvc-1ff7bb97-238e-4ebd-bfc3-fb9db1ae5656   10Gi       RWO            Delete           Bound    default/presto-data-claim   rook-ceph-block            100m
[root@node-1 ~]# kubectl get pvc
NAME                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
presto-data-claim   Bound    pvc-1ff7bb97-238e-4ebd-bfc3-fb9db1ae5656   10Gi       RWO            rook-ceph-block   100m
```

注意：这里的pv会自动创建，当提交了包含 StorageClass 字段的 PVC 之后，Kubernetes 就会根据这个 StorageClass 创建出对应的 PV，这是用到的是Dynamic Provisioning机制来动态创建pv，PV 支持 Static 静态请求，和动态创建两种方式。

```shell
# ceph集群端检查
[root@node-1 ~]# kubectl exec -n rook-ceph -it rook-ceph-operator-548b56f995-q47jb -- rbd info -p replicapool pvc-1ff7bb97-238e-4ebd-bfc3-fb9db1ae5656
rbd image 'pvc-1ff7bb97-238e-4ebd-bfc3-fb9db1ae5656':
        size 10 GiB in 2560 objects
        order 22 (4 MiB objects)
        snapshot_count: 0
        id: 1dd4352e100d
        block_name_prefix: rbd_data.1dd4352e100d
        format: 2
        features: layering
        op_features:
        flags:
        create_timestamp: Wed Feb 26 05:23:09 2020
        access_timestamp: Wed Feb 26 05:23:09 2020
        modify_timestamp: Wed Feb 26 05:23:09 2020
```

登陆pod检查rbd设备

```shell
[root@node-1 presto-kubernetes]# kubectl exec -it worker-7b66b5cb5-d2vzt bash
root@worker-7b66b5cb5-d2vzt:/usr/lib/presto-server-0.167-t.0.3/etc# df -h
Filesystem                                                                                        Size  Used Avail Use% Mounted on
/dev/mapper/docker-253:0-203738-8a1bc94ee31c6ca3e406a6c740b84cd8bd2c681c568dc8a4007f273208bbd9fd   10G  1.4G  8.7G  14% /
tmpfs                                                                                              64M     0   64M   0% /dev
tmpfs                                                                                             1.9G     0  1.9G   0% /sys/fs/cgroup
/dev/mapper/centos-root                                                                            18G  6.3G   12G  36% /etc/hosts
shm                                                                                                64M     0   64M   0% /dev/shm
/dev/rbd0                                                                                          10G   33M   10G   1% /var/presto/data
tmpfs                                                                                             1.9G   12K  1.9G   1% /run/secrets/kubernetes.io/serviceaccount
tmpfs                                                                                             1.9G     0  1.9G   0% /proc/acpi
tmpfs                                                                                             1.9G     0  1.9G   0% /proc/scsi
tmpfs                                                                                             1.9G     0  1.9G   0% /sys/firmware
```

## Trouble Shooting

- Docker拉取镜像源time out或者拉取不上，增加docker镜像源，把163，阿里，Azure的docker加速器最好都加上

  ```shell
  vim /etc/docker/daemin.json
  
  {
    "registry-mirrors": [
          "https://dockerhub.azk8s.cn",
          "https://b3sst9pc.mirror.aliyuncs.com",
          "https://hub-mirror.c.163.com"
  ]
  }
  
  systemctl daemon-reload
  systemctl restart docker
  ```

  

## Ref

- [Using the PostgreSQL Operator with Rook Ceph Storage](https://info.crunchydata.com/blog/crunchy-postgresql-operator-with-rook-ceph-storage)

- [presto-kubernetes](https://github.com/dharmeshkakadia/presto-kubernetes)

- [kubernetes部署rook+ceph存储系统](https://blog.csdn.net/networken/article/details/85772418)

- [kubernetes上部署rook-ceph存储系统](https://blog.csdn.net/ygqygq2/article/details/103014350)
- [在Kubernetes上部署Presto](https://blog.csdn.net/chenleiking/article/details/82493798)
- [Docker镜像加速器](https://docker_practice.gitee.io/install/mirror.html)